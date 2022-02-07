# Code analysis

## Things wrong with the given code:

1. The code inside User.findOneAndUpdate is being executed after the response is sent, so if something unexpected happens, there is no way to warn the client.

2. Errors are not being handled in User.findOneAndUpdate

3. Because Shop.findById is being executed after the response is sent, the res.status(500).send() will raise the error "Cannot set headers after they are sent to the client"

4. Superagent errors are not being handled

## Refactor

 1. Use of `let` / `const`
 2. Use of `async` / `await`
 3. Use variables for readability
 4. It seems to me that there was a `=== -1` missing in `shop.invitations.indexOf()`
 5. It seems to me that all the two `.indexOf()` are to avoid duplicates. This can be achieved using mongoose [addToSet](https://mongoosejs.com/docs/api/array.html#mongoosearray_MongooseArray-addToSet)
 6. I added some comments, they are always useful

```js
exports.inviteUser = async function(req, res, next) {

  const invitationBody = req.body;
  const shopId = req.params.shopId;
  const authUrl = "https://url.to.auth.system.com/invitation";
  
  try {
  
    // Get the invitation response
    const invitationResponse = await superagent.post(authUrl).send(invitationBody);

    // Return error 400 if user is already invited
    if (invitationResponse.status === 200) {
      res.status(400).json({
        error: true,
        message: 'User already invited to this shop'
      });
      return;
    }

    if (invitationResponse.status === 201) {
      const authId = invitationResponse.body.authId
      const invitationId = invitationResponse.body.invitationId

      // update or create user
      const createdUser = await User.findOneAndUpdate({ authId }, { authId }, { upsert: true, new: true })

      // find shop
      const shop = await Shop.findById(shopId)
      if (!shop) res.status(500).send({ message: 'No shop found' })
      
      // update shop
      shop.invitations.addToSet(invitationId)
      shop.users.addToSet(createdUser)
      await shop.save()
    }

    res.json(invitationResponse);

  } catch (err) {
    res.status(500).send(err)
  }

  return;

};
```
