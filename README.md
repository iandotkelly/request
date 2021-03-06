# Request — Simplified HTTP client

[![NPM](https://nodei.co/npm/request.png?downloads=true&downloadRank=true&stars=true)](https://nodei.co/npm/request/)

## Super simple to use

Request is designed to be the simplest way possible to make http calls. It supports HTTPS and follows redirects by default.

```javascript
var request = require('request');
request('http://www.google.com', function (error, response, body) {
  if (!error && response.statusCode == 200) {
    console.log(body) // Print the google web page.
  }
})
```

## Streaming

You can stream any response to a file stream.

```javascript
request('http://google.com/doodle.png').pipe(fs.createWriteStream('doodle.png'))
```

You can also stream a file to a PUT or POST request. This method will also check the file extension against a mapping of file extensions to content-types (in this case `application/json`) and use the proper `content-type` in the PUT request (if the headers don’t already provide one).

```javascript
fs.createReadStream('file.json').pipe(request.put('http://mysite.com/obj.json'))
```

Request can also `pipe` to itself. When doing so, `content-type` and `content-length` are preserved in the PUT headers.

```javascript
request.get('http://google.com/img.png').pipe(request.put('http://mysite.com/img.png'))
```

Request emits a "response" event when a response is received. The `response` argument will be an instance of [http.IncomingMessage](http://nodejs.org/api/http.html#http_http_incomingmessage).

```javascript
request
  .get('http://google.com/img.png')
  .on('response', function(response) {
    console.log(response.statusCode) // 200
    console.log(response.headers['content-type']) // 'image/png'
  })
  .pipe(request.put('http://mysite.com/img.png'))
```

Now let’s get fancy.

```javascript
http.createServer(function (req, resp) {
  if (req.url === '/doodle.png') {
    if (req.method === 'PUT') {
      req.pipe(request.put('http://mysite.com/doodle.png'))
    } else if (req.method === 'GET' || req.method === 'HEAD') {
      request.get('http://mysite.com/doodle.png').pipe(resp)
    }
  }
})
```

You can also `pipe()` from `http.ServerRequest` instances, as well as to `http.ServerResponse` instances. The HTTP method, headers, and entity-body data will be sent. Which means that, if you don't really care about security, you can do:

```javascript
http.createServer(function (req, resp) {
  if (req.url === '/doodle.png') {
    var x = request('http://mysite.com/doodle.png')
    req.pipe(x)
    x.pipe(resp)
  }
})
```

And since `pipe()` returns the destination stream in ≥ Node 0.5.x you can do one line proxying. :)

```javascript
req.pipe(request('http://mysite.com/doodle.png')).pipe(resp)
```

Also, none of this new functionality conflicts with requests previous features, it just expands them.

```javascript
var r = request.defaults({'proxy':'http://localproxy.com'})

http.createServer(function (req, resp) {
  if (req.url === '/doodle.png') {
    r.get('http://google.com/doodle.png').pipe(resp)
  }
})
```

You can still use intermediate proxies, the requests will still follow HTTP forwards, etc.

## Proxies

If you specify a `proxy` option, then the request (and any subsequent
redirects) will be sent via a connection to the proxy server.

If your endpoint is an `https` url, and you are using a proxy, then
request will send a `CONNECT` request to the proxy server *first*, and
then use the supplied connection to connect to the endpoint.

That is, first it will make a request like:

```
HTTP/1.1 CONNECT endpoint-server.com:80
Host: proxy-server.com
User-Agent: whatever user agent you specify
```

and then the proxy server make a TCP connection to `endpoint-server`
on port `80`, and return a response that looks like:

```
HTTP/1.1 200 OK
```

At this point, the connection is left open, and the client is
communicating directly with the `endpoint-server.com` machine.

See [the wikipedia page on HTTP Tunneling](http://en.wikipedia.org/wiki/HTTP_tunnel)
for more information.

By default, when proxying `http` traffic, request will simply make a
standard proxied `http` request.  This is done by making the `url`
section of the initial line of the request a fully qualified url to
the endpoint.

For example, it will make a single request that looks like:

```
HTTP/1.1 GET http://endpoint-server.com/some-url
Host: proxy-server.com
Other-Headers: all go here

request body or whatever
```

Because a pure "http over http" tunnel offers no additional security
or other features, it is generally simpler to go with a
straightforward HTTP proxy in this case.  However, if you would like
to force a tunneling proxy, you may set the `tunnel` option to `true`.

If you are using a tunneling proxy, you may set the
`proxyHeaderWhiteList` to share certain headers with the proxy.

By default, this set is:

```
accept
accept-charset
accept-encoding
accept-language
accept-ranges
cache-control
content-encoding
content-language
content-length
content-location
content-md5
content-range
content-type
connection
date
expect
max-forwards
pragma
proxy-authorization
referer
te
transfer-encoding
user-agent
via
```

Note that, when using a tunneling proxy, the `proxy-authorization`
header is *never* sent to the endpoint server, but only to the proxy
server.  All other headers are sent as-is over the established
connection.

### Controlling proxy behaviour using environment variables

The following environment variables are respected by `request`:

 * `HTTP_PROXY` / `http_proxy`
 * `HTTPS_PROXY` / `https_proxy`
 * `NO_PROXY` / `no_proxy`

When `HTTP_PROXY` / `http_proxy` are set, they will be used to proxy non-SSL requests that do not have an explicit `proxy` configuration option present. Similarly, `HTTPS_PROXY` / `https_proxy` will be respected for SSL requests that do not have an explicit `proxy` configuration option. It is valid to define a proxy in one of the environment variables, but then override it for a specific request, using the `proxy` configuration option. Furthermore, the `proxy` configuration option can be explicitly set to false / null to opt out of proxying altogether for that request.

`request` is also aware of the `NO_PROXY`/`no_proxy` environment variables. These variables provide a granular way to opt out of proxying, on a per-host basis. It should contain a comma separated list of hosts to opt out of proxying. It is also possible to opt of proxying when a particular destination port is used. Finally, the variable may be set to `*` to opt out of the implicit proxy configuration of the other environment variables.

Here's some examples of valid `no_proxy` values:

 * `google.com` - don't proxy HTTP/HTTPS requests to Google.
 * `google.com:443` - don't proxy HTTPS requests to Google, but *do* proxy HTTP requests to Google.
 * `google.com:443, yahoo.com:80` - don't proxy HTTPS requests to Google, and don't proxy HTTP requests to Yahoo!
 * `*` - ignore `https_proxy`/`http_proxy` environment variables altogether.

## UNIX Socket 

`request` supports the `unix://` protocol for all requests. The path is assumed to be absolute to the root of the host file system. 

HTTP paths are extracted from the supplied URL by testing each level of the full URL against net.connect for a socket response.

Thus the following request will GET `/httppath` from the HTTP server listening on `/tmp/unix.socket`

```javascript
request.get('unix://tmp/unix.socket/httppath')
```

## Forms

`request` supports `application/x-www-form-urlencoded` and `multipart/form-data` form uploads. For `multipart/related` refer to the `multipart` API.

URL-encoded forms are simple.

```javascript
request.post('http://service.com/upload', {form:{key:'value'}})
// or
request.post('http://service.com/upload').form({key:'value'})
```

For `multipart/form-data` we use the [form-data](https://github.com/felixge/node-form-data) library by [@felixge](https://github.com/felixge). For the most basic case, you can pass your upload form data via the `formData` option.


```javascript
var formData = {
  my_field: 'my_value',
  my_buffer: new Buffer([1, 2, 3]),
  my_file: fs.createReadStream(__dirname + '/unicycle.jpg'),
  remote_file: request(remoteFile)
};
request.post({url:'http://service.com/upload', formData: formData}, function optionalCallback(err, httpResponse, body) {
  if (err) {
    return console.error('upload failed:', err);
  }
  console.log('Upload successful!  Server responded with:', body);
});
```

For more advanced cases (like appending form data options) you'll need access to the form itself.

```javascript
var r = request.post('http://service.com/upload', function optionalCallback(err, httpResponse, body) {
  if (err) {
    return console.error('upload failed:', err);
  }
  console.log('Upload successful!  Server responded with:', body);
})

// Just like always, `r` is a writable stream, and can be used as such (you have until nextTick to pipe it, etc.)
// Alternatively, you can provide a callback (that's what this example does — see `optionalCallback` above).
var form = r.form();
form.append('my_field', 'my_value');
form.append('my_buffer', new Buffer([1, 2, 3]));
form.append('my_buffer', fs.createReadStream(__dirname + '/unicycle.jpg'), {filename: 'unicycle.jpg'});
```
See the [form-data](https://github.com/felixge/node-form-data) README for more information & examples.

Some variations in different HTTP implementations require a newline/CRLF before, after, or both before and after the boundary of a `multipart/form-data` request. This has been observed in the .NET WebAPI version 4.0. You can turn on a boundary preambleCRLF or postamble by passing them as `true` to your request options.

```javascript
  request(
    { method: 'PUT'
    , preambleCRLF: true
    , postambleCRLF: true
    , uri: 'http://service.com/upload'
    , multipart:
      [ { 'content-type': 'application/json'
        ,  body: JSON.stringify({foo: 'bar', _attachments: {'message.txt': {follows: true, length: 18, 'content_type': 'text/plain' }}})
        }
      , { body: 'I am an attachment' }
      ]
    }
  , function (error, response, body) {
      if (err) {
        return console.error('upload failed:', err);
      }
      console.log('Upload successful!  Server responded with:', body);
    }
  )
```


## HTTP Authentication

```javascript
request.get('http://some.server.com/').auth('username', 'password', false);
// or
request.get('http://some.server.com/', {
  'auth': {
    'user': 'username',
    'pass': 'password',
    'sendImmediately': false
  }
});
// or
request.get('http://some.server.com/').auth(null, null, true, 'bearerToken');
// or
request.get('http://some.server.com/', {
  'auth': {
    'bearer': 'bearerToken'
  }
});
```

If passed as an option, `auth` should be a hash containing values `user` || `username`, `pass` || `password`, and `sendImmediately` (optional).  The method form takes parameters `auth(username, password, sendImmediately)`.

`sendImmediately` defaults to `true`, which causes a basic authentication header to be sent.  If `sendImmediately` is `false`, then `request` will retry with a proper authentication header after receiving a `401` response from the server (which must contain a `WWW-Authenticate` header indicating the required authentication method).

Note that you can also use for basic authentication a trick using the URL itself, as specified in [RFC 1738](http://www.ietf.org/rfc/rfc1738.txt). 
Simply pass the `user:password` before the host with an `@` sign.

```javascript
var username = 'username',
    password = 'password',
    url = 'http://' + username + ':' + password + '@some.server.com';

request({url: url}, function (error, response, body) {
   // Do more stuff with 'body' here
});
```

Digest authentication is supported, but it only works with `sendImmediately` set to `false`; otherwise `request` will send basic authentication on the initial request, which will probably cause the request to fail.

Bearer authentication is supported, and is activated when the `bearer` value is available. The value may be either a `String` or a `Function` returning a `String`. Using a function to supply the bearer token is particularly useful if used in conjuction with `defaults` to allow a single function to supply the last known token at the time or sending a request or to compute one on the fly.

## OAuth Signing

```javascript
// Twitter OAuth
var qs = require('querystring')
  , oauth =
    { callback: 'http://mysite.com/callback/'
    , consumer_key: CONSUMER_KEY
    , consumer_secret: CONSUMER_SECRET
    }
  , url = 'https://api.twitter.com/oauth/request_token'
  ;
request.post({url:url, oauth:oauth}, function (e, r, body) {
  // Ideally, you would take the body in the response
  // and construct a URL that a user clicks on (like a sign in button).
  // The verifier is only available in the response after a user has
  // verified with twitter that they are authorizing your app.
  var access_token = qs.parse(body)
    , oauth =
      { consumer_key: CONSUMER_KEY
      , consumer_secret: CONSUMER_SECRET
      , token: access_token.oauth_token
      , verifier: access_token.oauth_verifier
      }
    , url = 'https://api.twitter.com/oauth/access_token'
    ;
  request.post({url:url, oauth:oauth}, function (e, r, body) {
    var perm_token = qs.parse(body)
      , oauth =
        { consumer_key: CONSUMER_KEY
        , consumer_secret: CONSUMER_SECRET
        , token: perm_token.oauth_token
        , token_secret: perm_token.oauth_token_secret
        }
      , url = 'https://api.twitter.com/1.1/users/show.json?'
      , params =
        { screen_name: perm_token.screen_name
        , user_id: perm_token.user_id
        }
      ;
    url += qs.stringify(params)
    request.get({url:url, oauth:oauth, json:true}, function (e, r, user) {
      console.log(user)
    })
  })
})
```

## Custom HTTP Headers

HTTP Headers, such as `User-Agent`, can be set in the `options` object.
In the example below, we call the github API to find out the number
of stars and forks for the request repository. This requires a
custom `User-Agent` header as well as https.

```javascript
var request = require('request');

var options = {
	url: 'https://api.github.com/repos/mikeal/request',
	headers: {
		'User-Agent': 'request'
	}
};

function callback(error, response, body) {
	if (!error && response.statusCode == 200) {
		var info = JSON.parse(body);
		console.log(info.stargazers_count + " Stars");
		console.log(info.forks_count + " Forks");
	}
}

request(options, callback);
```

## request(options, callback)

The first argument can be either a `url` or an `options` object. The only required option is `uri`; all others are optional.

* `uri` || `url` - fully qualified uri or a parsed url object from `url.parse()`
* `qs` - object containing querystring values to be appended to the `uri`
* `useQuerystring` - If true, use `querystring` to stringify and parse
  querystrings, otherwise use `qs` (default: `false`).  Set this option to
  `true` if you need arrays to be serialized as `foo=bar&foo=baz` instead of the
  default `foo[0]=bar&foo[1]=baz`.
* `method` - http method (default: `"GET"`)
* `headers` - http headers (default: `{}`)
* `body` - entity body for PATCH, POST and PUT requests. Must be a `Buffer` or `String`.
* `form` - when passed an object or a querystring, this sets `body` to a querystring representation of value, and adds `Content-type: application/x-www-form-urlencoded; charset=utf-8` header. When passed no options, a `FormData` instance is returned (and is piped to request).
* `auth` - A hash containing values `user` || `username`, `pass` || `password`, and `sendImmediately` (optional).  See documentation above.
* `json` - sets `body` but to JSON representation of value and adds `Content-type: application/json` header.  Additionally, parses the response body as JSON.
* `multipart` - (experimental) array of objects which contains their own headers and `body` attribute. Sends `multipart/related` request. See example below.
* `preambleCRLF` - append a newline/CRLF before the boundary of your `multipart/form-data` request.
* `postambleCRLF` - append a newline/CRLF at the end of the boundary of your `multipart/form-data` request.
* `followRedirect` - follow HTTP 3xx responses as redirects (default: `true`). This property can also be implemented as function which gets `response` object as a single argument and should return `true` if redirects should continue or `false` otherwise.
* `followAllRedirects` - follow non-GET HTTP 3xx responses as redirects (default: `false`)
* `maxRedirects` - the maximum number of redirects to follow (default: `10`)
* `encoding` - Encoding to be used on `setEncoding` of response data. If `null`, the `body` is returned as a `Buffer`. Anything else **(including the default value of `undefined`)** will be passed as the [encoding](http://nodejs.org/api/buffer.html#buffer_buffer) parameter to `toString()` (meaning this is effectively `utf8` by default).
* `pool` - An object describing which agents to use for the request. If this option is omitted the request will use the global agent (as long as [your options allow for it](request.js#L747)). Otherwise, request will search the pool for your custom agent. If no custom agent is found, a new agent will be created and added to the pool.
  * A `maxSockets` property can also be provided on the `pool` object to set the max number of sockets for all agents created (ex: `pool: {maxSockets: Infinity}`).
* `timeout` - Integer containing the number of milliseconds to wait for a request to respond before aborting the request
* `proxy` - An HTTP proxy to be used. Supports proxy Auth with Basic Auth, identical to support for the `url` parameter (by embedding the auth info in the `uri`)
* `oauth` - Options for OAuth HMAC-SHA1 signing. See documentation above.
* `hawk` - Options for [Hawk signing](https://github.com/hueniverse/hawk). The `credentials` key must contain the necessary signing info, [see hawk docs for details](https://github.com/hueniverse/hawk#usage-example).
* `strictSSL` - If `true`, requires SSL certificates be valid. **Note:** to use your own certificate authority, you need to specify an agent that was created with that CA as an option.
* `jar` - If `true` and `tough-cookie` is installed, remember cookies for future use (or define your custom cookie jar; see examples section)
* `aws` - `object` containing AWS signing information. Should have the properties `key`, `secret`. Also requires the property `bucket`, unless you’re specifying your `bucket` as part of the path, or the request doesn’t use a bucket (i.e. GET Services)
* `httpSignature` - Options for the [HTTP Signature Scheme](https://github.com/joyent/node-http-signature/blob/master/http_signing.md) using [Joyent's library](https://github.com/joyent/node-http-signature). The `keyId` and `key` properties must be specified. See the docs for other options.
* `localAddress` - Local interface to bind for network connections.
* `gzip` - If `true`, add an `Accept-Encoding` header to request compressed content encodings from the server (if not already present) and decode supported content encodings in the response.  **Note:** Automatic decoding of the response content is performed on the body data returned through `request` (both through the `request` stream and passed to the callback function) but is not performed on the `response` stream (available from the `response` event) which is the unmodified `http.IncomingMessage` object which may contain compressed data. See example below.
* `tunnel` - If `true`, then *always* use a tunneling proxy.  If
  `false` (default), then tunneling will only be used if the
  destination is `https`, or if a previous request in the redirect
  chain used a tunneling proxy.
* `proxyHeaderWhiteList` - A whitelist of headers to send to a
  tunneling proxy.


The callback argument gets 3 arguments: 

1. An `error` when applicable (usually from [`http.ClientRequest`](http://nodejs.org/api/http.html#http_class_http_clientrequest) object)
2. An [`http.IncomingMessage`](http://nodejs.org/api/http.html#http_http_incomingmessage) object
3. The third is the `response` body (`String` or `Buffer`, or JSON object if the `json` option is supplied)

## Convenience methods

There are also shorthand methods for different HTTP METHODs and some other conveniences.

### request.defaults(options)

This method returns a wrapper around the normal request API that defaults to whatever options you pass in to it.

**Note:** You can call `.defaults()` on the wrapper that is returned from `request.defaults` to add/override defaults that were previously defaulted. 

For example:
```javascript
//requests using baseRequest() will set the 'x-token' header
var baseRequest = request.defaults({
  headers: {x-token: 'my-token'}
})

//requests using specialRequest() will include the 'x-token' header set in
//baseRequest and will also include the 'special' header
var specialRequest = baseRequest.defaults({
  headers: {special: 'special value'}
})
```

### request.put

Same as `request()`, but defaults to `method: "PUT"`.

```javascript
request.put(url)
```

### request.patch

Same as `request()`, but defaults to `method: "PATCH"`.

```javascript
request.patch(url)
```

### request.post

Same as `request()`, but defaults to `method: "POST"`.

```javascript
request.post(url)
```

### request.head

Same as request() but defaults to `method: "HEAD"`.

```javascript
request.head(url)
```

### request.del

Same as `request()`, but defaults to `method: "DELETE"`.

```javascript
request.del(url)
```

### request.get

Same as `request()` (for uniformity).

```javascript
request.get(url)
```
### request.cookie

Function that creates a new cookie.

```javascript
request.cookie('cookie_string_here')
```
### request.jar

Function that creates a new cookie jar.

```javascript
request.jar()
```


## Examples:

```javascript
  var request = require('request')
    , rand = Math.floor(Math.random()*100000000).toString()
    ;
  request(
    { method: 'PUT'
    , uri: 'http://mikeal.iriscouch.com/testjs/' + rand
    , multipart:
      [ { 'content-type': 'application/json'
        ,  body: JSON.stringify({foo: 'bar', _attachments: {'message.txt': {follows: true, length: 18, 'content_type': 'text/plain' }}})
        }
      , { body: 'I am an attachment' }
      ]
    }
  , function (error, response, body) {
      if(response.statusCode == 201){
        console.log('document saved as: http://mikeal.iriscouch.com/testjs/'+ rand)
      } else {
        console.log('error: '+ response.statusCode)
        console.log(body)
      }
    }
  )
```

For backwards-compatibility, response compression is not supported by default.
To accept gzip-compressed responses, set the `gzip` option to `true`.  Note
that the body data passed through `request` is automatically decompressed
while the response object is unmodified and will contain compressed data if
the server sent a compressed response.

```javascript
  var request = require('request')
  request(
    { method: 'GET'
    , uri: 'http://www.google.com'
    , gzip: true
    }
  , function (error, response, body) {
      // body is the decompressed response body
      console.log('server encoded the data as: ' + (response.headers['content-encoding'] || 'identity'))
      console.log('the decoded data is: ' + body)
    }
  ).on('data', function(data) {
    // decompressed data as it is received
    console.log('decoded chunk: ' + data)
  })
  .on('response', function(response) {
    // unmodified http.IncomingMessage object
    response.on('data', function(data) {
      // compressed data as it is received
      console.log('received ' + data.length + ' bytes of compressed data')
    })
  })
```

Cookies are disabled by default (else, they would be used in subsequent requests). To enable cookies, set `jar` to `true` (either in `defaults` or `options`) and install `tough-cookie`.

```javascript
var request = request.defaults({jar: true})
request('http://www.google.com', function () {
  request('http://images.google.com')
})
```

To use a custom cookie jar (instead of `request`’s global cookie jar), set `jar` to an instance of `request.jar()` (either in `defaults` or `options`)

```javascript
var j = request.jar()
var request = request.defaults({jar:j})
request('http://www.google.com', function () {
  request('http://images.google.com')
})
```

OR

```javascript
// `npm install --save tough-cookie` before this works
var j = request.jar();
var cookie = request.cookie('key1=value1');
var url = 'http://www.google.com';
j.setCookie(cookie, url);
request({url: url, jar: j}, function () {
  request('http://images.google.com')
})
```

To inspect your cookie jar after a request

```javascript
var j = request.jar() 
request({url: 'http://www.google.com', jar: j}, function () {
  var cookie_string = j.getCookieString(uri); // "key1=value1; key2=value2; ..."
  var cookies = j.getCookies(uri); 
  // [{key: 'key1', value: 'value1', domain: "www.google.com", ...}, ...]
})
```

## Debugging

There are at least three ways to debug the operation of `request`:

1. Launch the node process like `NODE_DEBUG=request node script.js`
   (`lib,request,otherlib` works too).

2. Set `require('request').debug = true` at any time (this does the same thing
   as #1).

3. Use the [request-debug module](https://github.com/nylen/request-debug) to
   view request and response headers and bodies.
