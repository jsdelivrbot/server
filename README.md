# EpiBlog : API Server

This is the nodejs server of the EpiBlog project.  
This serveir is currently deployed on heroku : https://epiblog-api.herokuapp.com/  
The client projet is online in GitHub (Page) : https://nodejsepitech.github.io/epiblogClient/  
It allows the client of the same project to interact with a mysql database, in order to :
- create, modify, delete a user account
- log a user in / out
- create, modify, delete posts and comments

### Built with

- [express](http://expressjs.com/) : a nodejs web framework
- [ws](https://github.com/websockets/ws) : a nodejs WebSocket library
- [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken) : an implementation of JSON Web Tokens
- [joi](https://github.com/hapijs/joi) : a javascript objects validator
- [mysql](https://github.com/mysqljs/mysql) : a nodejs driver for mysql
- [eslint](https://eslint.org/) : a javascript linter
- [mocha](https://github.com/mochajs/mocha) && [chai](http://www.chaijs.com/) : nodejs libraries for testing

### Tests

Run the command `npm run test` to launch the tests.  



### Usage

Every **HTTP route** returns a json response, with the following format :
```ecmascript 6
const response = {
    message: '[An adequate message depending on the requested route and the response code]',
    data: {}|[] // optional
}
```
Some routes can be accessed without being authenticated, every other routes need to provide a token in the headers.
This token can be retrieved via the `POST /authenticate` route (see below).  
For every route requiring a token in the headers, the error response codes and associated reasons are the same as
described in the `GET /me` route (see below).  
If an error occurred internally, the response code is `500` with a fallback message, but the server should continue
running.  

The **WebSocket server** is used only for comments :
- to get all the comments of a specific post (user can be unauthenticated)
- to post a comment on a post (user has to be authenticated)
- to post a message on the global chat (user can be unauthenticated)  
The message should look like as follow :
```ecmascript 6
const request = {
    'x-request-id': '[An element that will be returned to the client]', // optional
    'x-method': /^(get|post)$/,
    'x-post-id': '[The post id for the comment]', // depends
    'x-username': '[A username]', // depends
    'x-authentication-token': '[The user jwt]' // depends
    'body': '[The content of the comment]' // depends
}
WebSocket.send(JSON.stringify(request));
```
Server side, the message will be parsed with the `JSON.parse` method.  
Any error on the server (like a bad json message stringified, a bad request, a database server) will send a stringified
json message as follow :
```ecmascript 6
const response = {
    'status': /^400|403|500$/,
    'message': '[An explicit message]'
};
WebSocketServer.send(JSON.stringify(response));
```
More details below.



#### `GET /`

- Code `200` (success)



#### `GET /me`  
Aim : to get the authenticated user's account information  
Header : `x-authentication-token`
- Code `403` (unauthorized) :
    - the value of `x-authentication-token` is invalid
    - user is not found in database
    - user has not logged in
    - user's account has been deleted
- Code `400` (bad request) :
    - `x-authentication-token` is not provided in headers
- Code `200` (success)  
    response.data contains user's account information : `{id: [user's id], firstname: [user's firstname], ...}`



#### `POST /authenticate` 
Aim : to authenticate a user and get a json web token    
Post data : `username` and `password`
- Code `400` :
    - `username` is not provided
    - `password` is not provided
    - `username` not found in database
    - user's account has been deleted
    - `password` does not match the one in database
- Code `200`  
    response.data contains a json web token : `{token: [user's jwt]}`



#### `GET /logout`  
Aim : to log out the authenticated user  
Header : `x-authentication-token`
- Code `200`



#### `POST /user`  
Aim : to create a new user
Post data : `username`, `firstname`, `lastname`, `email`, `password`, `passwordConfirmation` and `avatar`
- Code `400` :
    - `username` is not provided, does not only contain letters and numbers, or is not at least 4 and at most 30
    characters long
    - `firstname` is not provided
    - `lastname` is not provided
    - `email` is not provided, or is not email formatted
    - `password` is not provided, or is not containing at least 1 digit, 1 uppercase character, 1 lowercase
    character, 1 special character (space forbidden), and is not between 8 and 15 characters
    - `passwordConfirmation` is not provided, or is not the same as `password`
    - `username` already exists in database
    - `email` already exists in database
- Code `200`  
    response.data contains the id of the newly created user : `{id: [user's id]}`



#### `GET /user/[:id]`  
Aim : to get a user's account information  
Route parameter : `id` (the user id)  
Header : `x-authentication-token`
- Code `403` :
    - the `id` is not the same compared to the token user's id and the token user is not administrator
- Code `400` :
    - the `id` was not found in the database
- Code `200` (success)  
    response.data contains user's account information : `{id: [user's id], firstname: [user's firstname], ...}`



#### `PATCH /user/[:id]`  
Aim : to update a user's account  
Route parameter : `id` (the user id)  
Header : `x-authentication-token`
Post data : `field` and `value`
- Code `403` :
    - the `id` is not the same compared to the token user's id and the token user is not administrator
- Code `400` :
    - the `id` was not found in the database
    - `field` is not provided, or is a forbidden one (like `id`, `email`, `is_administrator`, `created_at` or
     `password`)
    - `value` is not provided



#### `DELETE /user/[:id]`  
Aim : to delete a user's account  
Route parameter : `id` (the user id)  
Header : `x-authentication-token`
- Code `403` :
    - the `id` is not the same compared to the token user's id and the token user is not administrator
- Code `400` :
    - the `id` was not found in the database



#### `POST /post`  
Aim : to create a new post  
Header : `x-authentication-token`
Post data : `title`, `description` and `content`
- Code `400` :
    - `title` is not provided, or is not at least 8 and at most 32 characters long
    - `descrition` (while optionnal) is not at least 8 and at most 128 characters long
    - `content` is not provided
- Code `200`  
    response.data contains the id of the newly created post : `{id: [post's id]}`



#### `GET /posts`
Aim : to get all existing posts  
- Code `200`  
    response.data contains an array of all post's data : `[{id: [post 1 id], ...}, {id: [post 2 id], ...}]`



#### `GET /post/[:id]`  
Aim : to get a post  
Route parameter : `id` (the post id)  
- Code `400` :
    - the `id` was not found in the database
- Code `200`
    response.data contains the post's information : `{id: [post's id], title: [postt's title], ...}`



#### `PATCH /post/[:id]`  
Aim : to update a post  
Route parameter : `id` (the user id)  
Header : `x-authentication-token`
Post data : `field` and `value`
- Code `400` :
    - the `creator_id` is not the same compared to the token user's id and the token user is not administrator
    - the `id` was not found in the database
    - `field` is not provided, or is a forbidden one (like `id`, `creator_id` or `created_at`)
    - `value` is not provided



#### `DELETE /post/[:id]`  
Aim : to delete a post  
Route parameter : `id` (the post id)  
Header : `x-authentication-token`
- Code `400` :
    - the `creator_id` is not the same compared to the token user's id and the token user is not administrator
    - the `id` was not found in the database



#### Get all the comments of a specific post  
The request will respect the following format :
```ecmascript 6
const request = {
    'x-request-id': '[An optionnal arbitrary id]', //optional
    'x-method': 'get',
    'x-post-id': '[The post id for the comment]',
}
```
The server will return the following response when it succeeds :
```ecmascript 6
const response = {
    'x-request-id': request.id, // optional
    'status': 200,
    'message': '[An explicit message]',
    'data': [{
        'username': '[The username of the comment\'s author]',
        'avatar': '[null, or the comment\'s author avatar]',
        'content': '[The comment content]',
        'created_at': '[The comment timestamp of its creation date]'
    }, {
        // same object for an other comment
//  }, ...
    }]
}
```



#### Post a comment on a specific post  
The request will respect the following format :
```ecmascript 6
const request = {
    'x-request-id': '[An optionnal arbitrary id]', //optional
    'x-method': 'post',
    'x-post-id': '[The post id for the comment]',
    'x-authentication-token': '[A valid jwt]',
    'body': '[The comment content]'
}
```
The value of the json web token can be retrieved via the HTTP route `POST /authenticate`.  
The server will return the following response when it succeeds :
```ecmascript 6
const response = {
    'x-request-id': request.id, // optional
    'status': 200,
    'message': '[An explicit message]'
}
```



#### Post a message on the global chat  
The request will respect the following format :
```ecmascript 6
const request = {
    'x-method': 'post',
    'x-username': '[A username]', // optional
    'x-authentication-token': '[A valid jwt]', // mandatory, if no 'x-username'
    'body': '[The message content]'
}
```
The server will send the following message to every client when it succeeds :
```ecmascript 6
const response = {
    'avatar': '[null, or the user avatar from the jwt]', // optional
    'username': '[The username provided in x-username, or the user username from the jwt]',
    'content': '[The message content]'
}
```



### Authors

Olivier Drouin ([olivier.drouin@epitech.eu](mailto:olivier.drouin@epitech.eu))

### License
MIT
