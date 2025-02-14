---
title: Client API
slug: /client-api/
sidebar_label: API
sidebar_position: 1
---

## IO

Exposed as the `io` namespace in the standalone build, or the result of calling `require('socket.io-client')`.

```html
<script src="/socket.io/socket.io.js"></script>
<script>
  const socket = io('http://localhost');
</script>
```

```js
const io = require('socket.io-client');
// or with import syntax
import io from 'socket.io-client';
```

### io.protocol

  * _(Number)_

The protocol revision number (currently: 4).

The protocol defines the format of the packets exchanged between the client and the server. Both the client and the server must use the same revision in order to understand each other.

You can find more information [here](https://github.com/socketio/socket.io-protocol).

### io([url][, options])

  - `url` _(String)_ (defaults to `window.location.host`)
  - `options` _(Object)_
    - `forceNew` _(Boolean)_ whether to reuse an existing connection
  - **Returns** `Socket`

Creates a new `Manager` for the given URL, and attempts to reuse an existing `Manager` for subsequent calls, unless the `multiplex` option is passed with `false`. Passing this option is the equivalent of passing `'force new connection': true` or `forceNew: true`.

A new `Socket` instance is returned for the namespace specified by the pathname in the URL, defaulting to `/`. For example, if the `url` is `http://localhost/users`, a transport connection will be established to `http://localhost` and a Socket.IO connection will be established to `/users`.

Query parameters can also be provided, either with the `query` option or directly in the url (example: `http://localhost/users?token=abc`).

```js
const io = require("socket.io-client");

const socket = io("ws://example.com/my-namespace", {
  reconnectionDelayMax: 10000,
  query: {
    auth: "123"
  }
});
```

is the short version of:

```js
const { Manager } = require("socket.io-client");

const manager = new Manager("ws://example.com", {
  reconnectionDelayMax: 10000
});

const socket = manager.socket("/my-namespace", {
  query: {
    auth: "123"
  }
});
```

See [new Manager(url[, options])](#new-Manager-url-options) for the list of available `options`.

### Initialization examples

#### With multiplexing

By default, a single connection is used when connecting to different namespaces (to minimize resources):

```js
const socket = io();
const adminSocket = io('/admin');
// a single connection will be established
```

That behaviour can be disabled with the `forceNew` option:

```js
const socket = io();
const adminSocket = io('/admin', { forceNew: true });
// will create two distinct connections
```

Note: reusing the same namespace will also create two connections

```js
const socket = io();
const socket2 = io();
// will also create two distinct connections
```

#### With custom `path`

```js
const socket = io('http://localhost', {
  path: '/myownpath'
});

// server-side
const io = require('socket.io')({
  path: '/myownpath'
});
```

The request URLs will look like: `localhost/myownpath/?EIO=3&transport=polling&sid=<id>`

```js
const socket = io('http://localhost/admin', {
  path: '/mypath'
});
```

Here, the socket connects to the `admin` namespace, with the custom path `mypath`.

The request URLs will look like: `localhost/mypath/?EIO=3&transport=polling&sid=<id>` (the namespace is sent as part of the payload).

#### With query parameters

```js
const socket = io('http://localhost?token=abc');

// server-side
const io = require('socket.io')();

// middleware
io.use((socket, next) => {
  let token = socket.handshake.query.token;
  if (isValid(token)) {
    return next();
  }
  return next(new Error('authentication error'));
});

// then
io.on('connection', (socket) => {
  let token = socket.handshake.query.token;
  // ...
});
```

#### With query option

```js
const socket = io({
  query: {
    token: 'cde'
  }
});
```

The query content can also be updated on reconnection:

```js
socket.on('reconnect_attempt', () => {
  socket.io.opts.query = {
    token: 'fgh'
  }
});
```

#### With `extraHeaders`

This only works if `polling` transport is enabled (which is the default). Custom headers will not be appended when using `websocket` as the transport. This happens because the WebSocket handshake does not honor custom headers. (For background see the [WebSocket protocol RFC](https://tools.ietf.org/html/rfc6455#section-4))

```js
const socket = io({
  transportOptions: {
    polling: {
      extraHeaders: {
        'x-clientid': 'abc'
      }
    }
  }
});

// server-side
const io = require('socket.io')();

// middleware
io.use((socket, next) => {
  let clientId = socket.handshake.headers['x-clientid'];
  if (isValid(clientId)) {
    return next();
  }
  return next(new Error('authentication error'));
});
```

#### With `websocket` transport only

By default, a long-polling connection is established first, then upgraded to "better" transports (like WebSocket). If you like to live dangerously, this part can be skipped:

```js
const socket = io({
  transports: ['websocket']
});

// on reconnection, reset the transports option, as the Websocket
// connection may have failed (caused by proxy, firewall, browser, ...)
socket.on('reconnect_attempt', () => {
  socket.io.opts.transports = ['polling', 'websocket'];
});
```

#### With a custom parser

The default [parser](https://github.com/socketio/socket.io-parser) promotes compatibility (support for `Blob`, `File`, binary check) at the expense of performance. A custom parser can be provided to match the needs of your application. Please see the example [here](https://github.com/socketio/socket.io/tree/master/examples/custom-parsers).

```js
const parser = require('socket.io-msgpack-parser'); // or require('socket.io-json-parser')
const socket = io({
  parser: parser
});

// the server-side must have the same parser, to be able to communicate
const io = require('socket.io')({
  parser: parser
});
```

#### With a self-signed certificate

```js
// server-side
const fs = require('fs');
const server = require('https').createServer({
  key: fs.readFileSync('server-key.pem'),
  cert: fs.readFileSync('server-cert.pem')
});
const io = require('socket.io')(server);
server.listen(3000);

// client-side
const socket = io({
  // option 1
  ca: fs.readFileSync('server-cert.pem'),

  // option 2. WARNING: it leaves you vulnerable to MITM attacks!
  rejectUnauthorized: false
});

```

## Manager

The `Manager` *manages* the Engine.IO [client](https://github.com/socketio/engine.io-client/) instance, which is the low-level engine that establishes the connection to the server (by using transports like WebSocket or HTTP long-polling).

The `Manager` handles the reconnection logic.

A single `Manager` can be used by several [Sockets](#Socket). You can find more information about this multiplexing feature [here](/docs/v2/namespaces/).

Please note that, in most cases, you won't use the Manager directly but use the [Socket](#Socket) instance instead.

### new Manager(url[, options])

  - `url` _(String)_
  - `options` _(Object)_
  - **Returns** `Manager`

Available options:

Option | Default value | Description
------ | ------------- | -----------
`path` | `/socket.io` | name of the path that is captured on the server side
`reconnection` | `true` | whether to reconnect automatically
`reconnectionAttempts` | `Infinity` | number of reconnection attempts before giving up
`reconnectionDelay` | `1000` | how long to initially wait before attempting a new reconnection. Affected by +/- `randomizationFactor`, for example the default initial delay will be between 500 to 1500ms.
`reconnectionDelayMax` | `5000` | maximum amount of time to wait between reconnections. Each attempt increases the reconnection delay by 2x along with a randomization factor.
`randomizationFactor` | `0.5` | 0 <= randomizationFactor <= 1
`timeout` | `20000` | connection timeout before a `connect_error` and `connect_timeout` events are emitted
`autoConnect` | `true` | by setting this false, you have to call `manager.open` whenever you decide it's appropriate
`query` | `{}` | additional query parameters that are sent when connecting a namespace (then found in `socket.handshake.query` object on the server-side)
`parser` | - | the parser to use. Defaults to an instance of the `Parser` that ships with socket.io. See [socket.io-parser](https://github.com/socketio/socket.io-parser).

Available options for the underlying Engine.IO client:

Option | Default value | Description
------ | ------------- | -----------
`upgrade` | `true` | whether the client should try to upgrade the transport from long-polling to something better.
`forceJSONP` | `false` | forces JSONP for polling transport.
`jsonp` | `true` | determines whether to use JSONP when necessary for polling. If disabled (by settings to false) an error will be emitted (saying "No transports available") if no other transports are available. If another transport is available for opening a connection (e.g. WebSocket) that transport will be used instead.
`forceBase64` | `false` | forces base 64 encoding for polling transport even when XHR2 responseType is available and WebSocket even if the used standard supports binary.
`enablesXDR` | `false` | enables XDomainRequest for IE8 to avoid loading bar flashing with click sound. default to `false` because XDomainRequest has a flaw of not sending cookie. |
`timestampRequests` | - | whether to add the timestamp with each transport request. Note: polling requests are always stamped unless this option is explicitly set to `false`
`timestampParam` | `t` | the timestamp parameter
`policyPort` | `843` | port the policy server listens on
`transports` | `['polling', 'websocket']` | a list of transports to try (in order). `Engine` always attempts to connect directly with the first one, provided the feature detection test for it passes.
`transportOptions` | `{}` | hash of options, indexed by transport name, overriding the common options for the given transport
`rememberUpgrade` | `false` | If true and if the previous websocket connection to the server succeeded, the connection attempt will bypass the normal upgrade process and will initially try websocket. A connection attempt following a transport error will use the normal upgrade process. It is recommended you turn this on only when using SSL/TLS connections, or if you know that your network does not block websockets.
`onlyBinaryUpgrades` | `false` | whether transport upgrades should be restricted to transports supporting binary data
`requestTimeout` | `0` | timeout for xhr-polling requests in milliseconds (`0`) (*only for polling transport*)
`protocols` | - | a list of subprotocols (see [MDN reference](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_servers#Subprotocols)) (*only for websocket transport*)

Node.js-only options for the underlying Engine.IO client:

Option | Default value | Description
------ | ------------- | -----------
`agent` | `false ` | the `http.Agent` to use
`pfx` | - | Certificate, Private key and CA certificates to use for SSL.
`key` | - | Private key to use for SSL.
`passphrase` | - | A string of passphrase for the private key or pfx.
`cert` | - | Public x509 certificate to use.
`ca` | - | An authority certificate or array of authority certificates to check the remote host against.
`ciphers` | - | A string describing the ciphers to use or exclude. Consult the [cipher format list](http://www.openssl.org/docs/apps/ciphers.html#CIPHER_LIST_FORMAT) for details on the format.
`rejectUnauthorized` | `false` | If true, the server certificate is verified against the list of supplied CAs. An 'error' event is emitted if verification fails. Verification happens at the connection level, before the HTTP request is sent.
`perMessageDeflate` | `true` | parameters of the WebSocket permessage-deflate extension (see [ws module](https://github.com/einaros/ws) api docs). Set to `false` to disable.
`extraHeaders` | `{}` | Headers that will be passed for each request to the server (via xhr-polling and via websockets). These values then can be used during handshake or for special proxies.
`forceNode` | `false` | Uses NodeJS implementation for websockets - even if there is a native Browser-Websocket available, which is preferred by default over the NodeJS implementation. (This is useful when using hybrid platforms like nw.js or electron)
`localAddress` | - | the local IP address to connect to


### manager.reconnection([value])

  - `value` _(Boolean)_
  - **Returns** `Manager|Boolean`

Sets the `reconnection` option, or returns it if no parameters are passed.

### manager.reconnectionAttempts([value])

  - `value` _(Number)_
  - **Returns** `Manager|Number`

Sets the `reconnectionAttempts` option, or returns it if no parameters are passed.

### manager.reconnectionDelay([value])

  - `value` _(Number)_
  - **Returns** `Manager|Number`

Sets the `reconnectionDelay` option, or returns it if no parameters are passed.

### manager.reconnectionDelayMax([value])

  - `value` _(Number)_
  - **Returns** `Manager|Number`

Sets the `reconnectionDelayMax` option, or returns it if no parameters are passed.

### manager.timeout([value])

  - `value` _(Number)_
  - **Returns** `Manager|Number`

Sets the `timeout` option, or returns it if no parameters are passed.

### manager.open([callback])

  - `callback` _(Function)_
  - **Returns** `Manager`

If the manager was initiated with `autoConnect` to `false`, launch a new connection attempt.

The `callback` argument is optional and will be called once the attempt fails/succeeds.

### manager.connect([callback])

Synonym of [manager.open([callback])](#manageropencallback).

### manager.socket(nsp, options)

  - `nsp` _(String)_
  - `options` _(Object)_
  - **Returns** `Socket`

Creates a new `Socket` for the given namespace.

### Event: 'connect_error'

  - `error` _(Object)_ error object

Fired upon a connection error.

### Event: 'connect_timeout'

Fired upon a connection timeout.

### Event: 'reconnect'

  - `attempt` _(Number)_ reconnection attempt number

Fired upon a successful reconnection.

### Event: 'reconnect_attempt'

  - `attempt` _(Number)_ reconnection attempt number

Fired upon an attempt to reconnect.

### Event: 'reconnecting'

  - `attempt` _(Number)_ reconnection attempt number

Fired upon an attempt to reconnect.

### Event: 'reconnect_error'

  - `error` _(Object)_ error object

Fired upon a reconnection attempt error.

### Event: 'reconnect_failed'

Fired when couldn't reconnect within `reconnectionAttempts`.

### Event: 'ping'

Fired when a ping packet is written out to the server.

### Event: 'pong'

  - `ms` _(Number)_ number of ms elapsed since `ping` packet (i.e.: latency).

Fired when a pong is received from the server.

## Socket

A `Socket` is the fundamental class for interacting with the server. A `Socket` belongs to a certain [Namespace](/docs/v2/namespaces) (by default `/`) and uses an underlying [Manager](#Manager) to communicate.

A `Socket` is basically an [EventEmitter](https://nodejs.org/api/events.html#events_class_eventemitter) which sends events to — and receive events from — the server over the network.

```js
socket.emit('hello', { a: 'b', c: [] });

socket.on('hey', (...args) => {
  // ...
});
```

### socket.id

  - _(String)_

A unique identifier for the socket session. Set after the `connect` event is triggered, and updated after the `reconnect` event.

```js
const socket = io('http://localhost');

console.log(socket.id); // undefined

socket.on('connect', () => {
  console.log(socket.id); // 'G5p5...'
});
```

### socket.connected

  - _(Boolean)_

Whether or not the socket is connected to the server.

```js
const socket = io('http://localhost');

socket.on('connect', () => {
  console.log(socket.connected); // true
});
```

### socket.disconnected

  - _(Boolean)_

Whether or not the socket is disconnected from the server.

```js
const socket = io('http://localhost');

socket.on('connect', () => {
  console.log(socket.disconnected); // false
});
```

### socket.open()

  - **Returns** `Socket`

Manually opens the socket.

```js
const socket = io({
  autoConnect: false
});

// ...
socket.open();
```

It can also be used to manually reconnect:

```js
socket.on('disconnect', () => {
  socket.open();
});
```

### socket.connect()

Synonym of [socket.open()](#socketopen).

### socket.send([...args][, ack])

  - `args`
  - `ack` _(Function)_
  - **Returns** `Socket`

Sends a `message` event. See [socket.emit(eventName[, ...args][, ack])](#socketemiteventname-args-ack).

### socket.emit(eventName[, ...args][, ack])

  - `eventName` _(String)_
  - `args`
  - `ack` _(Function)_
  - **Returns** `Socket`

Emits an event to the socket identified by the string name. Any other parameters can be included. All serializable datastructures are supported, including `Buffer`.

```js
socket.emit('hello', 'world');
socket.emit('with-binary', 1, '2', { 3: '4', 5: Buffer.from([6, 7, 8]) });
```

The `ack` argument is optional and will be called with the server answer.

```js
socket.emit('ferret', 'tobi', (data) => {
  console.log(data); // data will be 'woot'
});

// server:
//  io.on('connection', (socket) => {
//    socket.on('ferret', (name, fn) => {
//      fn('woot');
//    });
//  });
```

### socket.on(eventName, callback)

  - `eventName` _(String)_
  - `callback` _(Function)_
  - **Returns** `Socket`

Register a new handler for the given event.

```js
socket.on('news', (data) => {
  console.log(data);
});

// with multiple arguments
socket.on('news', (arg1, arg2, arg3, arg4) => {
  // ...
});
// with callback
socket.on('news', (cb) => {
  cb(0);
});
```

The socket actually inherits every method of the [Emitter](https://github.com/component/emitter) class, like `hasListeners`, `once` or `off` (to remove an event listener).

### socket.compress(value)

  - `value` _(Boolean)_
  - **Returns** `Socket`

Sets a modifier for a subsequent event emission that the event data will only be _compressed_ if the value is `true`. Defaults to `true` when you don't call the method.

```js
socket.compress(false).emit('an event', { some: 'data' });
```

### socket.binary(value)

Specifies whether the emitted data contains binary. Increases performance when specified. Can be `true` or `false`.

```js
socket.binary(false).emit('an event', { some: 'data' });
```

### socket.close()

  - **Returns** `Socket`

Disconnects the socket manually.

### socket.disconnect()

Synonym of [socket.close()](#socketclose).

### Events

The `Socket` instance emits all events sent by its underlying [Manager](#Manager), which are related to the state of the connection to the server.

It also emits events related to the state of the connection to the [Namespace](/docs/v2/namespaces/):

- `connect`,
- `disconnect`
- `error`.

### Event: 'connect'

Fired upon connection to the Namespace (including a successful reconnection).

```js
socket.on('connect', () => {
  // ...
});

// note: you should register event handlers outside of connect,
// so they are not registered again on reconnection
socket.on('myevent', () => {
  // ...
});
```

### Event: 'disconnect'

  - `reason` _(String)_

Fired upon disconnection. The list of possible disconnection reasons:

Reason | Description
------ | -----------
`io server disconnect` | The server has forcefully disconnected the socket with [socket.disconnect()](/docs/v2/server-api/#socket-disconnect-close)
`io client disconnect` | The socket was manually disconnected using [socket.disconnect()](/docs/v2/client-api/#socket-disconnect)
`ping timeout` | The server did not respond in the `pingTimeout` range
`transport close` | The connection was closed (example: the user has lost connection, or the network was changed from WiFi to 4G)
`transport error` | The connection has encountered an error (example: the server was killed during a HTTP long-polling cycle)

In all cases but the first (disconnection by the server), the client will wait for a small random delay and then reconnect.

```js
socket.on('disconnect', (reason) => {
  if (reason === 'io server disconnect') {
    // the disconnection was initiated by the server, you need to reconnect manually
    socket.connect();
  }
  // else the socket will automatically try to reconnect
});
```

### Event: 'error'

  - `error` _(Object)_ error object

Fired when an error occurs.

```js
socket.on('error', (error) => {
  // ...
});
```

### Event: 'connect_error'

  - `error` _(Object)_ error object

Fired upon a connection error.

```js
socket.on('connect_error', (error) => {
  // ...
});
```

### Event: 'connect_timeout'

Fired upon a connection timeout.

```js
socket.on('connect_timeout', (timeout) => {
  // ...
});
```

### Event: 'reconnect'

  - `attempt` _(Number)_ reconnection attempt number

Fired upon a successful reconnection.

```js
socket.on('reconnect', (attemptNumber) => {
  // ...
});
```

### Event: 'reconnect_attempt'

  - `attempt` _(Number)_ reconnection attempt number

Fired upon an attempt to reconnect.

```js
socket.on('reconnect_attempt', (attemptNumber) => {
  // ...
});
```

### Event: 'reconnecting'

  - `attempt` _(Number)_ reconnection attempt number

Fired upon an attempt to reconnect.

```js
socket.on('reconnecting', (attemptNumber) => {
  // ...
});
```

### Event: 'reconnect_error'

  - `error` _(Object)_ error object

Fired upon a reconnection attempt error.

```js
socket.on('reconnect_error', (error) => {
  // ...
});
```

### Event: 'reconnect_failed'

Fired when the client couldn't reconnect within `reconnectionAttempts`.

```js
socket.on('reconnect_failed', () => {
  // ...
});
```

### Event: 'ping'

Fired when a ping is sent to the server.

```js
socket.on('ping', () => {
  // ...
});
```

### Event: 'pong'

  - `ms` _(Number)_ number of ms elapsed since `ping` packet (i.e.: latency).

Fired when a pong is received from the server.

```js
socket.on('pong', (latency) => {
  // ...
});
```
