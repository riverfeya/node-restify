---
title: Quick Start
permalink: /docs/home/
redirect_from: /docs/
---

Настроить сервер быстро и легко. Вот простой эхо-сервер:

```js
var restify = require('restify');

function respond(req, res, next) {
  res.send('hello ' + req.params.name);
  next();
}

var server = restify.createServer();
server.get('/hello/:name', respond);
server.head('/hello/:name', respond);

server.listen(8080, function() {
  console.log('%s listening at %s', server.name, server.url);
});
```

Попробуйте выполнить это с помощью следующих команд curl, чтобы увидеть,
что Restify превратит это в
```sh
$ curl -is http://localhost:8080/hello/mark -H 'accept: text/plain'
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 10
Date: Mon, 31 Dec 2012 01:32:44 GMT
Connection: keep-alive

hello mark


$ curl -is http://localhost:8080/hello/mark
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 12
Date: Mon, 31 Dec 2012 01:33:33 GMT
Connection: keep-alive

"hello mark"


$ curl -is http://localhost:8080/hello/mark -X HEAD -H 'connection: close'
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 12
Date: Mon, 31 Dec 2012 01:42:07 GMT
Connection: close
```

ЗАметьте, что по-умолчанию, curl использует `Connection: keep-alive`. Чтобы
HEAD method возвращался правильно, вам нужно передать `Connection: close`.

Поскольку curl часто используется с REST APIs, Плагины Restify включают плагин для
обойти эту особенность в curl. Плагин проверяет, является ли пользовательский агент
is curl. Если это так, он устанавливает Connection header в "close" и удаляет
"Content-Length" header.

```js
server.pre(restify.plugins.pre.userAgentConnection());
```

## Цепи обработчика стиля Синатры

Как и многие другие Node.js based REST frameworks, restify использует 
стиль Синатры для определения маршрутов и обработчиков функций, 
которые обслуживают эти маршруты:

```js
server.get('/', function(req, res, next) {
  res.send('home')
  return next();
});

server.post('/foo',
  function(req, res, next) {
    req.someData = 'foo';
    return next();
  },
  function(req, res, next) {
    res.send(req.someData);
    return next();
  }
);
```

На сервере restify есть три различных цепочки обработчиков:

* `pre` - цепочка обработчиков, выполняемая до маршрутизации
* `use` - цепочка обработчиков выполняется после маршрутизации
* `{httpVerb}` - цепочка обработчиков, выполненная специально для маршрута

Все три цепочки обработчиков принимают одну функцию, несколько функций, 
или массив функций.


## Universal pre-handlers: server.pre()

Цепочка обработчика `pre` выполняется перед маршрутизацией. Это означает, что эти обработчики
будет выполняться для входящего запроса, даже если это маршрут, который вы не сделали регистр. 
Это может быть полезно для регистрации метрик или для очистки входящего запроса перед его маршрутизацией.

```js
// дедупликация косых черт в URL перед маршрутизацией
server.pre(restify.plugins.pre.dedupeSlashes());
```


## Universal handlers: server.use()

 `use` цепочки обработчиков выполняются после выбора маршрута для обслуживания
запроса. Обработчики функций, которые подключены через `use()` метод будет
запущен для всех роутов. Поскольку restify запускает обработчики в том порядке, в котором они
зарегистрированы, убедитесь, что все ваши `use()` вызовы происходят до определения любого
роута.

```js
server.use(function(req, res, next) {
    console.warn('run for all routes!');
    return next();
});
```


## Использование next()

По завершении каждой функции в цепочке обработчиков, Вы несете ответственность за
вызов `next()`. Вызов `next()` перейдет к следующей функции в цепочке.

В отличие от других REST фреймворков, вызов `res.send()` не активирует `next()`
автоматически. Во многих приложениях работа может продолжаться и после
`res.send()`, поэтому очистка ответа не является синонимом завершения
запроса.

В обычном случае, `next()` обычно не принимает никаких параметров. Если по
какой-то причине вы хотите прекратить обработку запроса, вы можете вызвать `next(false)`
прекратить обработку запроса:

```js
server.use([
  function(req, res, next) {
    if (someCondition) {
      res.send('done!');
      return next(false);
    }
    return next();
  },
  function(req, res, next) {
    // if someCondition is true, this handler is never executed
  }
]);
```

`next()` также принимает любой объект, для которого `instanceof Error` в true, который 
заставит restify отправлять этот объект Error как ответ клиенту. 
код состояния для ответа будет выведен из свойства `statusCode` объекта ошибки.
 Если `statusCode` не найден, то по умолчанию будет 500.
Таким образом, фрагмент ниже отправит сериализованную ошибку клиенту с http
500:

```js
server.use(function(req, res, next) {
  return next(new Error('boom!'));
});
```

А это пошлет 404, поскольку конструктор `NotFoundError` предоставляет
значение 404 для `statusCode`:

```js
server.use(function(req, res, next) {
  return next(new NotFoundError('not here!'));
});
```

Вызов `res.send()` с объектом Error дает похожие результаты, с этим
сниппетом отправка http 500 с сериализованной ошибкой клиента:

```js
server.use(function(req, res, next) {
  res.send(new Error('boom!'));
  return next();
});
```

Разница между ними заключается в том, что `next()` с объектом Error 
позволяет использовать серверные [event emitter](/components/server.md#errors). 
Это позволяет вам обрабатывать все вхождения типа ошибки с использованием общего обработчика. 
См. раздел [errorhandling](#error handling) .


Наконец, вы можете вызвать `next.ifError(err)` с объектом Error to cause restify
to throw, bringing down the process. Это может быть полезно, если Error 
всплыла но не может быть обработана, требуя от вас убить процесс.


## Роутинг

restify роутинг, в 'basic' режиме, в значительной степени идентична express/sinatra,
в этом HTTP глаголы используются с параметризованным ресурсом для определения какая
цепочка обработчиков запущена. Значения, связанные с именованными заполнителями:
доступными в `req.params`. Эти значения будут декодированы до того, как будут переданы вам.

```js
function send(req, res, next) {
  res.send('hello ' + req.params.name);
  return next();
}

server.post('/hello', function create(req, res, next) {
  res.send(201, Math.random().toString(36).substr(3, 8));
  return next();
});
server.put('/hello', send);
server.get('/hello/:name', send);
server.head('/hello/:name', send);
server.del('hello/:name', function rm(req, res, next) {
  res.send(204);
  return next();
});
```

Вы также можете пройти в [RegExp](https://developer.mozilla.org/en/JavaScript/Reference/Global_Objects/RegExp)
object and access the capture group with `req.params` (which will not
be interpreted in any way):

```js
server.get(/^\/([a-zA-Z0-9_\.~-]+)\/(.*)/, function(req, res, next) {
  console.log(req.params[0]);
  console.log(req.params[1]);
  res.send(200);
  return next();
});
```

Здесь любой запрос вроде:

```sh
$ curl localhost:8080/foo/my/cats/name/is/gandalf
```

Приведет к `req.params[0]` beявляющийсяing `foo` и `req.params[1]` являющийся
`my/cats/name/is/gandalf`.


Routes может быть указан любым из следующих http verbs - `del`, `get`,
`head`, `opts`, `post`, `put`, и `patch`.


```js
server.get(
    '/foo/:id',
    function(req, res, next) {
        console.log('Authenticate');
        return next();
    },
    function(req, res, next) {
        res.send(200);
        return next();
    }
);
```

### Hypermedia

Если параметризованный маршрут был определен строкой (не регулярное выражение), вы можете
рендерить его из других мест на сервере. This is useful to have HTTP
responses that link to other resources, without having to hardcode URLs
throughout the codebase. Both path and query strings parameters get URL encoded
appropriately.


```js
server.get({name: 'city', path: '/cities/:slug'}, /* ... */);

// in another route
res.send({
  country: 'Australia',
  // render a URL by specifying the route name and parameters
  capital: server.router.render('city', {slug: 'canberra'}, {details: true})
});
```

Which returns:

```js
{
  "country": "Australia",
  "capital": "/cities/canberra?details=true"
}
```

### Versioned Routes

Most REST APIs tend to need versioning, and restify ships with support
for [semver](http://semver.org/) versioning in an `Accept-Version`
header, the same way you specify NPM version dependencies:

```js
var restify = require('restify');

var server = restify.createServer();

function sendV1(req, res, next) {
  res.send('hello: ' + req.params.name);
  return next();
}

function sendV2(req, res, next) {
  res.send({ hello: req.params.name });
  return next();
}

server.get('/hello/:name', restify.plugins.conditionalHandler([
  { version: '1.1.3', handler: sendV1 },
  { version: '2.0.0', handler: sendV2 }
]));

server.listen(8080);
```

Try hitting with:

```sh
$ curl -s localhost:8080/hello/mark
{"hello":"mark"}
$ curl -s -H 'accept-version: ~1' localhost:8080/hello/mark
"hello: mark"
$ curl -s -H 'accept-version: ~2' localhost:8080/hello/mark
{"hello":"mark"}
$ curl -s -H 'accept-version: ~3' localhost:8080/hello/mark | json
{
  "code": "InvalidVersion",
  "message": "~3 is not supported by GET /hello/mark"
}
```

In the first case, we didn't specify an `Accept-Version` header at all, so
restify treats that like sending a `*`. Much as not sending an `Accept` header
means the client gets the server's choice. Restify will choose this highest
matching route. In the second case, we explicitly asked for for V1, which got
us response a response from the version 1 handler function, but then we asked
for V2 and got back JSON. Finally, we asked for a version that doesn't exist
and got an error.

You can default the versions on routes by passing in a version field at server
creation time.  Lastly, you can support multiple versions in the API by using
an array:

```js
server.get('/hello/:name', restify.plugins.conditionalHandler([
  { version: ['2.0.0', '2.1.0', '2.2.0'], handler: sendV2 }
]));
```

In this case you may need to know more information such as what the original
requested version string was, and what the matching version from the routes
supported version array was. Two methods make this info available:

```js
server.get('/version/test', restify.plugins.conditionalHandler([
  {
    version: ['2.0.0', '2.1.0', '2.2.0'],
    handler: function (req, res, next) {
      res.send(200, {
        requestedVersion: req.version(),
        matchedVersion: req.matchedVersion()
      });
      return next();
    }
  }
]));
```

Hitting this route will respond as below:

```sh
$ curl -s -H 'accept-version: <2.2.0' localhost:8080/version/test | json
{
  "requestedVersion": "<2.2.0",
  "matchedVersion": "2.1.0"
}
```


## Upgrade Requests

Incoming HTTP requests that contain a `Connection: Upgrade` header are treated
somewhat differently by the node HTTP server.  If you want restify to push
Upgrade requests through the regular routing chain, you need to enable
`handleUpgrades` when creating the server.

To determine if a request is eligible for Upgrade, check for the existence of
`res.claimUpgrade()`.  This method will return an object with two properties:
the `socket` of the underlying connection, and the first received data `Buffer`
as `head` (may be zero-length).

Once `res.claimUpgrade()` is called, `res` itself is marked unusable for
further HTTP responses; any later attempt to `send()` or `end()`, etc, will
throw an `Error`.  Likewise if `res` has already been used to send at least
part of a response to the client, `res.claimUpgrade()` will throw an `Error`.
Upgrades and regular HTTP Response behaviour are mutually exclusive on any
particular connection.

Using the Upgrade mechanism, you can use a library like
[watershed](https://github.com/jclulow/node-watershed) to negotiate WebSockets
connections.  For example:

```js
var ws = new Watershed();
server.get('/websocket/attach', function upgradeRoute(req, res, next) {
  if (!res.claimUpgrade) {
    next(new Error('Connection Must Upgrade For WebSockets'));
    return;
  }

  var upgrade = res.claimUpgrade();
  var shed = ws.accept(req, upgrade.socket, upgrade.head);
  shed.on('text', function(msg) {
    console.log('Received message from websocket client: ' + msg);
  });
  shed.send('hello there!');

  next(false);
});
```

## Responses' Content Negotiation And Formatting

If you're using `res.send()` restify will determine the content-type to respond
with by, from highest priority to lowest priority:

1. using the value of `res.contentType` if present
1. otherwise, using the value of the `Content-Type` response header if set
1. otherwise, using `application/json` if the body is an object that is not a
   Buffer instance
1. otherwise, negotiating the content-type by matching available formatters with
   the request's `accept` header

If a content-type can't be determined, then restify will respond with an error.

If a content-type can be negotiated, restify then determines what formatter to
use to format the response's content.

If no formatter matching the content-type can be found, restify will by default
override the response's content-type to `'application/octet-stream'` and then
error if no formatter is found for that content-type.

This default behavior can be changed by passing `strictFormatters: false`
(default is false) when creating the restify server instance. In that case, if
no formatter is found for the negotiated content-type, the response is flushed
without applying any formatter.

Note in the examples above we've not defined any formatters, so we've been
leveraging the fact that restify ships with `application/json`, `text/plain` and
`application/octet-stream` formatters. You can add additional formatters to
restify by passing in a hash of content-type -> parser at server creation time:

```js
var server = restify.createServer({
  formatters: {
    'application/foo': function formatFoo(req, res, body) {
      if (body instanceof Error)
        return body.stack;

      if (Buffer.isBuffer(body))
        return body.toString('base64');

      return util.inspect(body);
    }
  }
});
```

For example, attempting to send a content-type that does not have a defined
formatter:

```js
server.get('/foo', function(req, res, next) {
  res.setHeader('content-type', 'text/css');
  res.send('hi');
  return next();
});
```

Will result in a response with a content-type of `application/octet-stream`:

```sh
$ curl -i localhost:3000/
HTTP/1.1 200 OK
Content-Type: application/octet-stream
Content-Length: 2
Date: Thu, 02 Jun 2016 06:50:54 GMT
Connection: keep-alive
```

However, if the server instance is created with `strictFormatters:false`:

```js
var server = restify.createServer({
  strictFormatters: false
});
```

The response would has a content-type of `text/css` even though no `'text/css'`
formatter is present:

```sh
$ curl -i localhost:3000/
HTTP/1.1 200 OK
Content-Type: text/css
Content-Length: 2
Date: Thu, 02 Jun 2016 06:50:54 GMT
Connection: keep-alive
```

As previously noted, restify ships with built-in formatters for json, text,
and binary. When you override or append to this, the "priority" might change;
to ensure that the priority is set to what you want, you should set a `q-value`
on your formatter definitions, which will ensure sorting happens the way you
want:

```js
restify.createServer({
  formatters: {
    'application/foo; q=0.9': function formatFoo(req, res, body) {
      if (body instanceof Error)
        return body.stack;

      if (Buffer.isBuffer(body))
        return body.toString('base64');

      return util.inspect(body);
    }
  }
});
```

Restify ships with the following default formatters, which can be overridden
when passing a formatters options to `createServer()`:

* application/javascript
* application/json
* text/plain
* application/octet-stream

The restify response object retains has all the "raw" methods of a node
[ServerResponse](http://nodejs.org/docs/latest/api/http.html#http.ServerResponse)
 on it as well.

```js
var body = 'hello world';
res.writeHead(200, {
  'Content-Length': Buffer.byteLength(body),
  'Content-Type': 'text/plain'
});
res.write(body);
res.end();
```

## Error handling

It is common to want to handle an error conditions the same way. As an example,
you may want to serve a 500 page on all `InternalServerErrors`. In this case,
you can add a listener for the `InternalServer` error event that is always
fired when this Error is encountered by restify as part of a `next(error)`
statement. This gives you a way to handle all errors of the same class
identically across the server. You can also use a generic `restifyError` event
which will catch errors of all types.

An example of sending a 404:

```js
server.get('/hello/:foo', function(req, res, next) {
  // resource not found error
  var err = new restify.errors.NotFoundError('oh noes!');
  return next(err);
});

server.on('NotFound', function (req, res, err, cb) {
  // do not call res.send! you are now in an error context and are outside
  // of the normal next chain. you can log or do metrics here, and invoke
  // the callback when you're done. restify will automtically render the
  // NotFoundError depending on the content-type header you have set in your
  // response.
  return cb();
});
```

For customizing the error being sent back to the client:

```js
server.get('/hello/:name', function(req, res, next) {
  // some internal unrecoverable error
  var err = new restify.errors.InternalServerError('oh noes!');
  return next(err);
});

server.on('InternalServer', function (req, res, err, cb) {
  // by default, restify will usually render the Error object as plaintext or
  // JSON depending on content negotiation. the default text formatter and JSON
  // formatter are pretty simple, they just call toString() and toJSON() on the
  // object being passed to res.send, which in this case, is the error object.
  // so to customize what it sent back to the client when this error occurs,
  // you would implement as follows:

  // for any response that is text/plain
  err.toString = function toString() {
    return 'an internal server error occurred!';
  };
  // for any response that is application/json
  err.toJSON = function toJSON() {
    return {
      message: 'an internal server error occurred!',
      code: 'boom!'
    }
  };

  return cb();
});

server.on('restifyError', function (req, res, err, cb) {
  // this listener will fire after both events above!
  // `err` here is the same as the error that was passed to the above
  // error handlers.
  return cb();
});
```

Here is another example of `InternalServerError`, but this time with a custom
formatter:

```js
const errs = require('restify-errors');

const server = restify.createServer({
  formatters: {
    'text/html': function(req, res, body) {
      if (body instanceof Error) {
        // body here is an instance of InternalServerError
        return '<html><body>' + body.message + '</body></html>';
      }
    }
  }
});

server.get('/', function(req, res, next) {
  res.header('content-type', 'text/html');
  return next(new errs.InternalServerError('oh noes!'));
});
```


### restify-errors

A module called restify-errors exposes a suite of error constructors for many
common http and REST related errors. These constructors can be used in
conjunction with the `next(err)` pattern to easily leverage the server's event
emitter. The full list of constructors can be viewed over at the
[restify-errors](https://github.com/restify/errors) repository. Here are some
examples:


```js
var errs = require('restify-errors');

server.get('/', function(req, res, next) {
  return next(new errs.ConflictError("I just don't like you"));
});
```

```sh
$ curl -is localhost:3000
HTTP/1.1 409 Conflict
Content-Type: application/json
Content-Length: 53
Date: Fri, 03 Jun 2016 20:29:45 GMT
Connection: keep-alive

{"code":"Conflict","message":"I just don't like you"}
```

When using restify-errors, you can also directly call `res.send(err)`, and
restify will automatically serialize your error for you:

```js
var errs = require('restify-errors');

server.get('/', function(req, res, next) {
  res.send(new errs.GoneError('gone girl'));
  return next();
});
```

```sh
$ curl -is localhost:8080/
HTTP/1.1 410 Gone
Content-Type: application/json
Content-Length: 37
Date: Fri, 03 Jun 2016 20:17:48 GMT
Connection: keep-alive

{"code":"Gone","message":"gone girl"}
```

This automatic serialization happens because the JSON formatter will call
`JSON.stringify()` on the Error object, and all restify-errors have a `toJSON`
method defined. Compare this to a standard Error object which does not have
`toJSON` defined:

```js
server.get('/sendErr', function(req, res, next) {
  res.send(new Error('where is my msg?'));
  return next();
});

server.get('/nextErr', function(req, res, next) {
  return next(new Error('where is my msg?'));
});
```

```sh
$ curl -is localhost:8080/sendErr
HTTP/1.1 410 Gone
Content-Type: application/json
Content-Length: 37
Date: Fri, 03 Jun 2016 20:17:48 GMT
Connection: keep-alive

{}

$ curl -is localhost:8080/nextErr
HTTP/1.1 410 Gone
Content-Type: application/json
Content-Length: 37
Date: Fri, 03 Jun 2016 20:17:48 GMT
Connection: keep-alive

{}
```

If you want to use custom errors, make sure you have `toJSON` defined, or use
restify-error's `makeConstructor()` method to automatically create Errors that
are supported with with `toJSON`.


#### HttpError

restify-errors provides constructors that inherit from either HttpError or
RestError. All HttpErrors have a numeric http `statusCode` and `body`
properties. The statusCode will automatically set the HTTP response status
code, and the body attribute by default will be the message.

All status codes between 400 and 5xx are automatically converted into
an HttpError with the name being 'PascalCase' and spaces removed.  For
the complete list, take a look at the
[node source](https://github.com/nodejs/node/blob/master/lib/_http_server.js#L17).

From that code above `418: I'm a teapot` would be `ImATeapotError`, as
an example.


#### RestError

A common problem with REST APIs and HTTP is that they often end
up needing to overload 400 and 409 to mean a bunch of different
things.  There's no real standard on what to do in these cases, but in
general you want machines to be able to (safely) parse these things
out, and so restify defines a convention of a `RestError`.  A
`RestError` is a subclass of one of the particular `HttpError` types,
and additionally sets the body attribute to be a JS object with the
attributes `code` and `message`.  For example, here's a built-in RestError:


```js
var errs = require('restify-errors');
var server = restify.createServer();

server.get('/hello/:name', function(req, res, next) {
  return next(new errs.InvalidArgumentError("I just don't like you"));
});

$ curl -is localhost:8080/hello/mark | json
HTTP/1.1 409 Conflict
Content-Type: application/json
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET
Access-Control-Allow-Headers: Accept, Accept-Version, Content-Length, Content-MD5, Content-Type, Date, Api-Version
Access-Control-Expose-Headers: Api-Version, Request-Id, Response-Time
Connection: close
Content-Length: 60
Content-MD5: MpEcO5EQFUZ2MNeUB2VaZg==
Date: Tue, 03 Jan 2012 00:50:21 GMT
Server: restify
Request-Id: bda456dd-2fe4-478d-809c-7d159d58d579
Response-Time: 3

{
  "code": "InvalidArgument",
  "message": "I just don't like you"
}
```

The built-in HttpErrors are:

* BadRequestError (400 Bad Request)
* UnauthorizedError (401 Unauthorized)
* PaymentRequiredError (402 Payment Required)
* ForbiddenError (403 Forbidden)
* NotFoundError (404 Not Found)
* MethodNotAllowedError (405 Method Not Allowed)
* NotAcceptableError (406 Not Acceptable)
* ProxyAuthenticationRequiredError (407 Proxy Authentication Required)
* RequestTimeoutError (408 Request Time-out)
* ConflictError (409 Conflict)
* GoneError (410 Gone)
* LengthRequiredError (411 Length Required)
* PreconditionFailedError (412 Precondition Failed)
* RequestEntityTooLargeError (413 Request Entity Too Large)
* RequesturiTooLargeError (414 Request-URI Too Large)
* UnsupportedMediaTypeError (415 Unsupported Media Type)
* RequestedRangeNotSatisfiableError (416 Requested Range Not Satisfiable)
* ExpectationFailedError (417 Expectation Failed)
* ImATeapotError (418 I'm a teapot)
* UnprocessableEntityError (422 Unprocessable Entity)
* LockedError (423 Locked)
* FailedDependencyError (424 Failed Dependency)
* UnorderedCollectionError (425 Unordered Collection)
* UpgradeRequiredError (426 Upgrade Required)
* PreconditionRequiredError (428 Precondition Required)
* TooManyRequestsError (429 Too Many Requests)
* RequestHeaderFieldsTooLargeError (431 Request Header Fields Too Large)
* InternalServerError (500 Internal Server Error)
* NotImplementedError (501 Not Implemented)
* BadGatewayError (502 Bad Gateway)
* ServiceUnavailableError (503 Service Unavailable)
* GatewayTimeoutError (504 Gateway Time-out)
* HttpVersionNotSupportedError (505 HTTP Version Not Supported)
* VariantAlsoNegotiatesError (506 Variant Also Negotiates)
* InsufficientStorageError (507 Insufficient Storage)
* BandwidthLimitExceededError (509 Bandwidth Limit Exceeded)
* NotExtendedError (510 Not Extended)
* NetworkAuthenticationRequiredError (511 Network Authentication Required)
* BadDigestError (400 Bad Request)
* BadMethodError (405 Method Not Allowed)
* InternalError (500 Internal Server Error)
* InvalidArgumentError (409 Conflict)
* InvalidContentError (400 Bad Request)
* InvalidCredentialsError (401 Unauthorized)
* InvalidHeaderError (400 Bad Request)
* InvalidVersionError (400 Bad Request)
* MissingParameterError (409 Conflict)
* NotAuthorizedError (403 Forbidden)
* RequestExpiredError (400 Bad Request)
* RequestThrottledError (429 Too Many Requests)
* ResourceNotFoundError (404 Not Found)
* WrongAcceptError (406 Not Acceptable)

And the built in RestErrors are:

* 400 BadDigestError
* 405 BadMethodError
* 500 InternalError
* 409 InvalidArgumentError
* 400 InvalidContentError
* 401 InvalidCredentialsError
* 400 InvalidHeaderError
* 400 InvalidVersionError
* 409 MissingParameterError
* 403 NotAuthorizedError
* 412 PreconditionFailedError
* 400 RequestExpiredError
* 429 RequestThrottledError
* 404 ResourceNotFoundError
* 406 WrongAcceptError

You can also create your own subclasses using the `makeConstructor` method:

```js
var errs = require('restify-errors');
var restify = require('restify');

errs.makeConstructor('ZombieApocalypseError');

var myErr = new errs.ZombieApocalypseError('zomg!');
```

The constructor takes `message`, `statusCode`, `restCode`, and `context`
options. Please check out the restify-errors repo for more information.


## Socket.IO

To use [socket.io](http://socket.io/) with restify, just treat your restify
server as if it were a "raw" node server:

```js
var server = restify.createServer();
var io = socketio.listen(server.server);

server.get('/', function indexHTML(req, res, next) {
    fs.readFile(__dirname + '/index.html', function (err, data) {
        if (err) {
            next(err);
            return;
        }

        res.setHeader('Content-Type', 'text/html');
        res.writeHead(200);
        res.end(data);
        next();
    });
});


io.sockets.on('connection', function (socket) {
    socket.emit('news', { hello: 'world' });
    socket.on('my other event', function (data) {
            console.log(data);
    });
});

server.listen(8080, function () {
    console.log('socket.io server listening at %s', server.url);
});
```
