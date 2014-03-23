HTTP.methods [![Build Status](https://travis-ci.org/CollectionFS/Meteor-http-methods.png?branch=master)](https://travis-ci.org/CollectionFS/Meteor-http-methods)
============

This package add the abillity to add `HTTP` server methods to your project. It's a server-side package only *- no client simulations added.*

##Usage
HTTP.methods can be added
```js
  HTTP.methods({
    'list': function() {
      return '<b>Default content type is text/html</b>';
    }
  });
```

##Methods scope
The methods scope contains different kinds of inputs. We can also get user details if logged in.


* `this.userId` The user whos id and token was used to run this method, if set/found
* `this.method` - `GET`, `POST`, `PUT`, `DELETE`
* `this.query` - query params `?token=1&id=2` -> { token: 1, id: 2 }
* `this.params` - Set params /foo/:name/test/:id -> { name: '', id: '' }
* `this.userAgent` - Get the useragent string set in request header
* `this.setUserId(id)` - Option for setting the `this.userId`
* `this.isSimulation` - Allways false on the server
* `this.unblock` - Not implemented
* `this.setContentType('text/html')` - Set the content type in header, defaults to text/html
* `this.addHeader('Content-Disposition', 'attachment; filename="name.ext"')`
* `this.setStatusCode(200)` - Set the status code in response header
* `createReadStream` - If a request then get the read stream
* `createWriteStream` - If you want to stream data to the client
* `Error` - When streaming we have to be able to send error and close connection

##Passing data via header
From the client:
```js
  HTTP.post('list', {
    data: { foo: 'bar' }
  }, function(err, result) {
    console.log('Content: ' + result.content + ' === "Hello"');
  });
```

HTTP Server method:
```js
  HTTP.methods({
    'list': function(data) {
      if (data.foo === 'bar') {
        /* data i passed via the header is parsed by EJSON.parse if
        not able then it returns the raw data instead */
      }
      return 'Hello';
    }
  });
```

##Parametres
The method name or url can be used to pass `params` values to the method.

Client
```js
  HTTP.post('/items/12/emails/5', function(err, result) {
    console.log('Got back: ' + result.content);
  });
```

Server
```js
  HTTP.methods({
    '/items/:itemId/emails/:emailId': function() {
      // this.param.itemId === '12'
      // this.param.emailId === '5'
    }
  });
```

##Extended usage
The `HTTP.methods` normally takes a function, but it can be set to an object for fingrained handling.

Example:
```js
  HTTP.methods({
    '/hello': {
      get: function(data) {},
      // post: function(data) {},
      // put: function(data) {},
      // delete: function(data) {}
    }
  });
```
*In this example the mounted http rest point will only support the `get` method*

Example:
```js
  HTTP.methods({
    '/hello': {
      method: function(data) {},
    }
  });
```
*In this example all methods `get`, `put`, `post`, `delete` will use the same function - This would be equal to setting the function directly on the http mount point*

##Authentication
The client needs the `access_token` to login in HTTP methods. *One could create a HTTP login/logout method for allowing pure external access*

Client
```js
  HTTP.post('/hello', {
    params: {
      token: Accounts && Accounts._storedLoginToken()
    }
  }, function(err, result) {
    console.log('Got back: ' + result.content);
  });
```

Server
```js
  'hello': function(data) {
    if (this.userId) {
      var user = Meteor.users.findOne({ _id: this.userId });
      return 'Hello ' + (user && user.username || user && user.emails[0].address || 'user');
    } else {
      this.setStatusCode(401); // Unauthorized
    }
  }
```

##Using custom authentication
It's possible to make your own function to set the userId - not using the builtin token pattern.
```js
  // My auth will return the userId
  var myAuth = function() {
    // Read the token from '/hello?token=5'
    var userToken = self.query.token;
    // Check the userToken before adding it to the db query
    // Set the this.userId
    if (userToken) {
      var user = Meteor.users.findOne({ 'services.resume.loginTokens.token': userToken });

      // Set the userId in the scope
      return user && user._id;
    }  
  };

  HTTP.methods({
    '/hello': {
      auth: myAuth,
      method: function(data) {
        // this.userId is set by myAuth
        if (this.userId) { /**/ } else { /**/ }
      }
    }
  });
```
*The above resembles the builtin auth handler*


##Login and logout (TODO)
These operations are not currently supported for off Meteor use - Theres some security considerations.

`basic-auth` is broadly supported, but:
* password should not be sent in clear text - hash with base64?
* should be used on https connections
* Its difficult / impossible to logout a user?

`token` the current `access_token` seems to be a better solution. Better control and options to logout users. But calling the initial `login` method still requires:
* hashing of password
* use https
