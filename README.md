# jsan-body-parser [![Build Status](https://travis-ci.org/mastilver/jsan-body-parser.svg?branch=master)](https://travis-ci.org/mastilver/jsan-body-parser)

Node.js body parsing middleware for [jsan](https://github.com/kolodny/jsan).

Parse incoming request bodies in a middleware before your handlers, available
under the `req.body` property.

## Installation

```sh
$ npm install jsan-body-parser
```

## API

```js
var jsanBodyParser = require('jsan-body-parser')
```


### jsanBodyParser(options)

Returns middleware that parses `jsan`. This parser accepts any Unicode
encoding of the body and supports automatic inflation of `gzip` and `deflate`
encodings.

A new `body` object containing the parsed data is populated on the `request`
object after the middleware (i.e. `req.body`).

#### Options

The `jsan` function takes an option `options` object that may contain any of
the following keys:

##### inflate

When set to `true`, then deflated (compressed) bodies will be inflated; when
`false`, deflated bodies are rejected. Defaults to `true`.

##### limit

Controls the maximum request body size. If this is a number, then the value
specifies the number of bytes; if it is a string, the value is passed to the
[bytes](https://www.npmjs.com/package/bytes) library for parsing. Defaults
to `'100kb'`.

##### reviver

The `reviver` option is passed directly to `JSON.parse` as the second
argument. You can find more information on this argument
[in the MDN documentation about JSON.parse](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse#Example.3A_Using_the_reviver_parameter).

##### strict

When set to `true`, will only accept arrays and objects; when `false` will
accept anything `JSON.parse` accepts. Defaults to `true`.

##### type

The `type` option is used to determine what media type the middleware will
parse. This option can be a function or a string. If a string, `type` option
is passed directly to the [type-is](https://www.npmjs.org/package/type-is#readme)
library and this can be an extension name (like `jsan`), a mime type (like
`application/jsan`), or a mime type with a wildcard (like `*/*` or `*/jsan`).
If a function, the `type` option is called as `fn(req)` and the request is
parsed if it returns a truthy value. Defaults to `application/jsan`.

##### verify

The `verify` option, if supplied, is called as `verify(req, res, buf, encoding)`,
where `buf` is a `Buffer` of the raw request body and `encoding` is the
encoding of the request. The parsing can be aborted by throwing an error.

## Errors

The middleware provided by this module create errors depending on the error
condition during parsing. The errors will typically have a `status` property
that contains the suggested HTTP response code and a `body` property containing
the read body, if available.

The following are the common errors emitted, though any error can come through
for various reasons.

### content encoding unsupported

This error will occur when the request had a `Content-Encoding` header that
contained an encoding but the "inflation" option was set to `false`. The
`status` property is set to `415`.

### request aborted

This error will occur when the request is aborted by the client before reading
the body has finished. The `received` property will be set to the number of
bytes received before the request was aborted and the `expected` property is
set to the number of expected bytes. The `status` property is set to `400`.

### request entity too large

This error will occur when the request body's size is larger than the "limit"
option. The `limit` property will be set to the byte limit and the `length`
property will be set to the request body's length. The `status` property is
set to `413`.

### request size did not match content length

This error will occur when the request's length did not match the length from
the `Content-Length` header. This typically occurs when the request is malformed,
typically when the `Content-Length` header was calculated based on characters
instead of bytes. The `status` property is set to `400`.

### stream encoding should not be set

This error will occur when something called the `req.setEncoding` method prior
to this middleware. This module operates directly on bytes only and you cannot
call `req.setEncoding` when using this module. The `status` property is set to
`500`.

### unsupported charset "BOGUS"

This error will occur when the request had a charset parameter in the
`Content-Type` header, but the `iconv-lite` module does not support it OR the
parser does not support it. The charset is contained in the message as well
as in the `charset` property. The `status` property is set to `415`.

### unsupported content encoding "bogus"

This error will occur when the request had a `Content-Encoding` header that
contained an unsupported encoding. The encoding is contained in the message
as well as in the `encoding` property. The `status` property is set to `415`.

## Examples

### Express/Connect top-level generic

This example demonstrates adding a generic JSON and URL-encoded parser as a
top-level middleware, which will parse the bodies of all incoming requests.
This is the simplest setup.

```js
var express = require('express')
var bodyParser = require('body-parser')
var jsanBodyParser = require('jsan-body-parser')

var app = express()

// parse application/x-www-form-urlencoded
app.use(bodyParser.urlencoded({ extended: false }))

// parse application/json
app.use(bodyParser.json())

// parse application/jsan
app.use(jsanBodyParser())

app.use(function (req, res) {
  res.setHeader('Content-Type', 'text/plain')
  res.write('you posted:\n')
  res.end(JSON.stringify(req.body, null, 2))
})
```

### Express route-specific

This example demonstrates adding body parsers specifically to the routes that
need them. In general, this is the most recommended way to use body-parser with
Express.

```js
var express = require('express')
var bodyParser = require('body-parser')
var jsanBodyParser = require('jsan-body-parser')

var app = express()

// create application/jsan parser
var jsanParser = jsanBodyParser()

// create application/x-www-form-urlencoded parser
var urlencodedParser = bodyParser.urlencoded({ extended: false })

// POST /login gets urlencoded bodies
app.post('/login', urlencodedParser, function (req, res) {
  if (!req.body) return res.sendStatus(400)
  res.send('welcome, ' + req.body.username)
})

// POST /api/users gets jsan bodies
app.post('/api/users', jsanParser, function (req, res) {
  if (!req.body) return res.sendStatus(400)
  // create user in req.body
})
```

### Change accepted type for parsers

All the parsers accept a `type` option which allows you to change the
`Content-Type` that the middleware will parse.

```js
// drop-in replacement of app.use(bodyParser.json())
app.use(jsanBodyParser({ type: 'application/json' }))
```

## License

[MIT](LICENSE)
