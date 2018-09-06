# csurf-expire

[![NPM Version][npm-image]][npm-url]
[![NPM Downloads][downloads-image]][downloads-url]

### This is a fork of the excellent [csurf library](https://github.com/expressjs/csurf) extending its functionality by preventing expired cookies from being allowed to send requests.

>The only change is that when using the cookie option and setting the `maxAge` paratmeter, any cookie that has exceeded the `maxAge` parameter will be rejected and a 403 error will be generated. This should not be a problem for normal users, as a new cookie and CSRF token is generated on each request. If you wish to set a `maxAge` on the cookie without rejecting expired cookies, then you can set the expires parameter on the cookie instead. This does not apply when using session middleware.

Node.js [CSRF](https://en.wikipedia.org/wiki/Cross-site_request_forgery) protection middleware.



Requires either a session middleware or [cookie-parser](https://www.npmjs.com/package/cookie-parser) to be initialized first.

  * If you are setting the ["cookie" option](#cookie) to a non-`false` value,
    then you must use [cookie-parser](https://www.npmjs.com/package/cookie-parser)
    before this module.
  * Otherwise, you must use a session middleware before this module. For example:
    - [express-session](https://www.npmjs.com/package/express-session)
    - [cookie-session](https://www.npmjs.com/package/cookie-session)

If you have questions on how this module is implemented, please read
[Understanding CSRF](https://github.com/pillarjs/understanding-csrf).

## Installation

This is a [Node.js](https://nodejs.org/en/) module available through the
[npm registry](https://www.npmjs.com/). Installation is done using the
[`npm install` command](https://docs.npmjs.com/getting-started/installing-npm-packages-locally):

```sh
$ npm install csurf-expire
```

## API

<!-- eslint-disable no-unused-vars -->

```js
var csurf = require('csurf-expire')
```

### csurf([options])

Create a middleware for CSRF token creation and validation. This middleware
adds a `req.csrfToken()` function to make a token which should be added to
requests which mutate state, within a hidden form field, query-string etc.
This token is validated against the visitor's session or csrf cookie.

#### Options

The `csurf` function takes an optional `options` object that may contain
any of the following keys:

##### cookie

Determines if the token secret for the user should be stored in a cookie
or in `req.session`. Defaults to `false`.

When set to `true` (or an object of options for the cookie), then the module
changes behavior and no longer uses `req.session`. This means you _are no
longer required to use a session middleware_. Instead, you do need to use the
[cookie-parser](https://www.npmjs.com/package/cookie-parser) middleware in
your app before this middleware.

When set to an object, cookie storage of the secret is enabled and the
object contains options for this functionality (when set to `true`, the
defaults for the options are used). The options may contain any of the
following keys:

  - `key` - the name of the cookie to use to store the token secret
    (defaults to `'_csrf'`).
  - `path` - the path of the cookie (defaults to `'/'`).
  - **`maxAge` - the maximum age the cookie can be before it expires in seconds. Setting this parameter will cause a 403 response if any cookie and csfr token is sent beyond the maxAge window**
  - any other [res.cookie](http://expressjs.com/4x/api.html#res.cookie)
    option can be set.

##### ignoreMethods

An array of the methods for which CSRF token checking will disabled.
Defaults to `['GET', 'HEAD', 'OPTIONS']`.

##### sessionKey

Determines what property ("key") on `req` the session object is located.
Defaults to `'session'` (i.e. looks at `req.session`). The CSRF secret
from this library is stored and read as `req[sessionKey].csrfSecret`.

If the ["cookie" option](#cookie) is not `false`, then this option does
nothing.

##### value

Provide a function that the middleware will invoke to read the token from
the request for validation. The function is called as `value(req)` and is
expected to return the token as a string.

The default value is a function that reads the token from the following
locations, in order:

  - `req.body._csrf` - typically generated by the `body-parser` module.
  - `req.query._csrf` - a built-in from Express.js to read from the URL
    query string.
  - `req.headers['csrf-token']` - the `CSRF-Token` HTTP request header.
  - `req.headers['xsrf-token']` - the `XSRF-Token` HTTP request header.
  - `req.headers['x-csrf-token']` - the `X-CSRF-Token` HTTP request header.
  - `req.headers['x-xsrf-token']` - the `X-XSRF-Token` HTTP request header.

## Example

### Simple express example

The following is an example of some server-side code that generates a form
that requires a CSRF token to post back.

```js
var cookieParser = require('cookie-parser')
var csrf = require('csurf-expire')
var bodyParser = require('body-parser')
var express = require('express')

// setup route middlewares
var csrfProtection = csrf({ cookie: { maxAge: 60 * 60 * 8 } })
var parseForm = bodyParser.urlencoded({ extended: false })

// create express app
var app = express()

// parse cookies
// we need this because "cookie" is true in csrfProtection
app.use(cookieParser())

app.get('/form', csrfProtection, function (req, res) {
  // pass the csrfToken to the view
  res.render('send', { csrfToken: req.csrfToken() })
})

app.post('/process', parseForm, csrfProtection, function (req, res) {
  res.send('data is being processed')
})
```

Inside the view (depending on your template language; handlebars-style
is demonstrated here), set the `csrfToken` value as the value of a hidden
input field named `_csrf`:

```html
<form action="/process" method="POST">
  <input type="hidden" name="_csrf" value="{{csrfToken}}">
  
  Favorite color: <input type="text" name="favoriteColor">
  <button type="submit">Submit</button>
</form>
```

#### Using AJAX

When accessing protected routes via ajax both the csrf token will need to be
passed in the request. Typically this is done using a request header, as adding
a request header can typically be done at a central location easily without
payload modification.

The CSRF token is obtained from the `req.csrfToken()` call on the server-side.
This token needs to be exposed to the client-side, typically by including it in
the initial page content. One possibility is to store it in an HTML `<meta>` tag,
where value can then be retrieved at the time of the request by JavaScript.

The following can be included in your view (handlebar example below), where the
`csrfToken` value came from `req.csrfToken()`:

```html
<meta name="csrf-token" content="{{csrfToken}}">
```

The following is an example of using the
[Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) to post
to the `/process` route with the CSRF token from the `<meta>` tag on the page:

<!-- eslint-env browser -->

```js
// Read the CSRF token from the <meta> tag
var token = document.querySelector('meta[name="csrf-token"]').getAttribute('content')

// Make a request using the Fetch API
fetch('/process', {
  credentials: 'same-origin', // <-- includes cookies in the request
  headers: {
    'CSRF-Token': token // <-- is the csrf token as a header
  },
  method: 'POST',
  body: {
    favoriteColor: 'blue'
  }
})
```

### Ignoring Routes

**Note** CSRF checks should only be disabled for requests that you expect to
come from outside of your website. Do not disable CSRF checks for requests
that you expect to only come from your website. An existing session, even if
it belongs to an authenticated user, is not enough to protect against CSRF
attacks.

The following is an example of how to order your routes so that certain endpoints
do not check for a valid CSRF token.

```js
var cookieParser = require('cookie-parser')
var csrf = require('csurf-expire')
var bodyParser = require('body-parser')
var express = require('express')

// create express app
var app = express()

// create api router
var api = createApiRouter()

// mount api before csrf is appended to the app stack
app.use('/api', api)

// now add csrf and other middlewares, after the "/api" was mounted
app.use(bodyParser.urlencoded({ extended: false }))
app.use(cookieParser())
app.use(csrf({ cookie: { maxAge: 60 * 60 * 8 } }))

app.get('/form', function (req, res) {
  // pass the csrfToken to the view
  res.render('send', { csrfToken: req.csrfToken() })
})

app.post('/process', function (req, res) {
  res.send('csrf was required to get here')
})

function createApiRouter () {
  var router = new express.Router()

  router.post('/getProfile', function (req, res) {
    res.send('no csrf to get here')
  })

  return router
}
```

### Custom error handling

When the CSRF token validation fails, an error is thrown that has
`err.code === 'EBADCSRFTOKEN'`. This can be used to display custom
error messages.

```js
var bodyParser = require('body-parser')
var cookieParser = require('cookie-parser')
var csrf = require('csurf-expire')
var express = require('express')

var app = express()
app.use(bodyParser.urlencoded({ extended: false }))
app.use(cookieParser())
app.use(csrf({ cookie: true }))

// error handler
app.use(function (err, req, res, next) {
  if (err.code !== 'EBADCSRFTOKEN') return next(err)

  // handle CSRF token errors here
  res.status(403)
  res.send('form tampered with')
})
```

## License

[MIT](LICENSE)

[npm-image]: https://img.shields.io/npm/v/csurf-expire.svg
[npm-url]: https://npmjs.org/package/csurf-expire
[downloads-image]: https://img.shields.io/npm/dm/csurf-expire.svg
[downloads-url]: https://npmjs.org/package/csurf-expire
