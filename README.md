# rtc-switchboard

This is an rtc.io signalling server (counterpart to
[rtc-signaller](https://github.com/rtc-io/rtc-signaller)) that makes use of
the excellent realtime abstraction library,
[primus](https://github.com/primus/primus). It has been designed and built
primarily as a _reference implementation_ for a signalling server and is
not designed to be deployed at scale.


[![NPM](https://nodei.co/npm/rtc-switchboard.png)](https://nodei.co/npm/rtc-switchboard/)

[![Build Status](https://img.shields.io/travis/rtc-io/rtc-switchboard.svg?branch=master)](https://travis-ci.org/rtc-io/rtc-switchboard) [![unstable](https://img.shields.io/badge/stability-unstable-yellowgreen.svg)](https://github.com/dominictarr/stability#unstable) 

## Try it out

If you would like to our test signalling server (no uptime guaranteed) then
you can use [rtc-quickconnect](https://github.com/rtc-io/rtc-quickconnect)
and take it for a spin:

```js
var quickconnect = require('rtc-quickconnect');

quickconnect('//switchboard.rtc.io/', { room: 'switchboard-test' })
  .createDataChannel('test')
  .once('channel:opened:test', function(peerId, dc) {
    dc.onmessage = function(evt) {
      console.log('received data: ', evt.data);
    };

    dc.send('hello');
  });

```

Other examples are available in the [guidebook](http://guidebook.rtc.io)

## Usage: Standalone

If you wish to use `rtc-switchboard` on it's own to test signalling,
then you can simply clone this repository, install dependencies and start
the server:

```
git clone https://github.com/rtc-io/rtc-switchboard.git
cd rtc-switchboard
npm install
node server.js
```

If you wish to run the server on a specific port, then set the `NODE_PORT`
environment variable prior to execution:

```
NODE_PORT=8997 node server.js
```

## Usage: API

To create an application using primus signalling, see the following
examples:

### Pure Node HTTP

```js
var server = require('http').createServer();
var switchboard = require('./')(server, { servelib: true });
var port = parseInt(process.env.NODE_PORT || process.env.PORT || process.argv[2], 10) || 3000;
var replify = require('replify');

server.on('request', function(req, res) {
  if (req.url === '/') {
    res.writeHead(302, {
      'Location': 'https://github.com/rtc-io/rtc-switchboard'
    });
    res.end('switchboard available from: https://github.com/rtc-io/rtc-switchboard');
  }
});

// start the server
server.listen(port, function(err) {
  if (err) {
    return console.log('Encountered error starting server: ', err);
  }

  console.log('server running at http://localhost:' + port + '/');
});

// add the repl
replify({
  name: 'switchboard',
  app: switchboard,
  contexts: {
    server: server
  }
});

switchboard.on('room:create', function(room) {
  console.log('room ' + room + ' created, now have ' + switchboard.rooms.length + ' active rooms');
});

switchboard.on('room:destroy', function(room) {
  console.log('room ' + room + ' destroyed, ' + switchboard.rooms.length + ' active rooms remain');

  if (typeof gc == 'function') {
    console.log('gc');
    gc();
  }
});


```

### Using Express

```js
var express = require('express');
var app = express();
var server = require('http').Server(app);
var port = process.env.PORT || 3000;

// create the switchboard
var switchboard = require('rtc-switchboard')(server);

// we need to expose the primus library
app.get('/rtc.io/primus.js', switchboard.library());

server.listen(port, function(err) {
  if (err) {
    return;
  }

  console.log('server listening on port: ' + port);
});
```

## Including the Primus Client

The `rtc-switchboard` makes use of the slick WebSockets abstraction library
[Primus](https://github.com/primus/primus). To work with the server, you
will need to include the `primus.js` library in your application prior to
attempting a websocket connection.

If you are working with a local standalone server, the following script
tag will likely do the job:

```html
<script src="http://localhost:3000/rtc.io/primus.js"></script>
```

__NOTE:__ A specific call to include primus is not required if you are
working with particular rtc.io library (such as
[rtc-glue](https://github.com/rtc-io/rtc-glue)), as they will ensure the
primus library has been included prior to running their internal code.

## Usage: Docker

If you are interested in deploying an instance of `rtc-switchboard` using
[docker](https://www.docker.com/) then the following is a great place to
start:

<https://github.com/synctree/docker-rtc-switchboard>

## Writing Custom Command Handlers

When you initialize the switchboard, you are able to provide custom handlers
for specific commands that you want handled by the switchboard. Imagine
for instance, that we want our switchboard to do something clever when a
client sends an `/img` command.

We would create our server to include the custom `img` command handler:

```js
var server = require('http').createServer();
var Primus = require('primus');

// create the signaller, providing our own primus instance (using engine.io)
var switchboard = require('rtc-switchboard')(server, {
  servelib: true,
  handlers: {
    img: require('./handlers/img')
  }
});

// start the server
server.listen(3000);
```

And then we would write a small module for the handler:

```js
module.exports = function(mgr, spark, data, payload) {
  console.log('received an img command with payload: ', payload);
};
```

## Logging and Analytics using the `data` event

Every message that flows through the switchboard (whether handled or not)
can be logged through tapping into the `data` event.  The example below
demonstrates how this can be done with a node logging module like
[bunyan](https://github.com/trentm/node-bunyan):

```js
var express = require('express');
var app = express();
var server = require('http').Server(app);
var port = process.env.PORT || 3000;
var bunyan = require('bunyan');
var log = bunyan.createLogger({ name: 'rtc-switchboard' });

// create the switchboard
var switchboard = require('rtc-switchboard')(server);

// we need to expose the primus library
app.get('/rtc.io/primus.js', switchboard.library());

server.listen(port, function(err) {
  if (err) {
    return;
  }

  console.log('server running at: http://localhost:' + port + '/');
});

switchboard.on('data', function(data, peerId, spark) {
  log.info({ peer: peerId }, 'received: ' + data);
});
```

As can be seen in the example above, the handlers of the `data` event can
expect to receive three arguments to the handler function, as per the code
snippet below:

```js
switchboard.on('data', function(data, peerId, spark) {
});
```

The `data` is the raw data of that has been sent from the client, the
`peerId` is the id of the peer sending the data (this will be `undefined` if
it is a message received prior to an `/announce` command).  Finally we have
the raw primus `spark` that can be examined for additional information.

## Reference

### switchboard(server, opts?)

Create the switchboard which uses primus under the hood. By default calling
this function will create a new `Primus` instance and use the
pure [websockets adapter](https://github.com/primus/primus#websockets).

## Custom Message Handlers

The socket server is configured to handle some specific rtc.io signaller
messages.  The handlers are stored in the `handlers/` folder of the
repository and have details outlined below.

### announce handler

Will handle `/announce` messages and associate the peer id assigned by
the client-side signaller with the socket on the server side.

This will allow routing of messages to the correct receipient when
`/to` messages are received.

### ping handler

Will handle `/ping` messages

## License(s)

### Apache 2.0

Copyright 2014 National ICT Australia Limited (NICTA)

   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
