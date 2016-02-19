# ![pageres](media/promo.png)

[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/geekuillaume/ChatUp?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

ChatUp is a highly performant, scalable and extensible chat platform based on websockets.

It uses [NodeJS](https://nodejs.org/), [Redis](http://redis.io/) and [Nginx](http://nginx.org/) with the [PushStream Module](https://github.com/wandenberg/nginx-push-stream-module).

ChatUp is:

- **SCALABLE** on multiple servers
- **SECURE** with JSON Web Token Authentication to integrate in an existing system
- **EXTENSIBLE** via server-side and client-side plugins
- **FAULT-TOLERANT**
- **MODULARIZED** in multiple micro-services
- **USED AT LARGE SCALE** and created for [Streamup](https://streamup.com/)

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Test it with Docker](#test-it-with-docker)
- [Plugins](#plugins)
- [How to Install](#how-to-install)
- [How to use](#how-to-use)
  - [Dispatcher](#dispatcher)
  - [Chat Worker](#chat-worker)
  - [Redis](#redis)
  - [Client lib](#client-lib)
- [How does it works](#how-does-it-works)
- [FAQ](#faq)
  - [How to use SSL](#how-to-use-ssl)
  - [How to transmit user information](#how-to-transmit-user-information)
  - [How to ban users](#how-to-ban-users)
  - [How to get messages history of a room](#how-to-get-messages-history-of-a-room)
  - [How to use additional channels](#how-to-use-additional-channels)
  - [How to post a message from the API endpoint](#how-to-post-a-message-from-the-api-endpoint)
- [License](#license)
- [History](#history)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Test it with Docker

ChatUp is available on Docker. You can use this image to test it quickly or to deploy it easily in production.

To do so, execute those three commands:

```sh
docker run --name chatup-redis -d redis # Create a redis server
docker run --link chatup-redis:redis -e CHATUP_REDISHOST=redis --name chatup-dispatcher -d geekuillaume/chatup dispatcher # Start the dispatcher service in charge of the load-balacing
docker run --link chatup-redis:redis -e CHATUP_REDISHOST=redis --name chatup-worker -d geekuillaume/chatup worker --use-container-ip # Start the worker service in charge of all the communication
```

You can then get the dispatcher IP with `docker inspect --format '{{ .NetworkSettings.IPAddress }}' chatup-dispatcher` and use the example [client page](http://rawgit.com/geekuillaume/ChatUp/master/examples/client/index.html) indicating this IP to test it.

You can spawn multiple workers they will be load-balanced without having to do anything else. If you want to add multiple dispatchers, you need to put a load-balancer in front of them.

### IMPORTANT

By default, the chat uses example key for the JWT verfication. To test it use the awesome [jwt.io](http://jwt.io/) website with the `RS256` algorithm and the [public](examples/JWTKeyExample.pub) and [private](examples/JWTPrivateExample.key) example keys. In production, create a new Docker image from `geekuillaue/chatup` and include your key.

Here is a basic JWT and it's payload:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYW1lIjoiVGVzdCAxIiwiX3B1YmxpYyI6eyJuYW1lIjoiVGVzdCAxIn19.1uNu_T7xKtozXgqwoY31ouDo13H-RJ_q-yfqWau2Im-3PXxEcnn_hFuSJii_XJQKpVz1bVJG4vV9o67Wi0vI1B9A2WGHA2Wud9zWHj0UiL-jWhPd_EypMlVhr6AVe6YeP_IeguUAqD6u9tjOQhPrmIQ9zw327Pm9CHpGD_JZAgeHmVNaz67f-4nrRNZkGWrVrPXe2TKaiSz9gAIfMdae0ySY14QMStWHR-80YLwq2lpRmAWamxf6BCZ8f6HMv6k-0QcFb-n8j0wtOrKVxICQvSBhdyHQCTrGqKuRsLBd3eLBAMPlhmWKDyNYsCnvA9A73bYNPMN3w_FOy3jzv6LpBA
```

```json
{
  "name": "Test 1",
  "_public": {
    "name": "Test 1"
  }
}
```

## Plugins

- [Blacklist](https://github.com/geekuillaume/chatup-blacklist): blocks messages containing specific strings, administrable via an API on the dispatchers
- [Slack](https://github.com/geekuillaume/chatup-slack): sends a message to a Slack channel when a message containing a specific string is sent
- [Rate Limiter](https://github.com/geekuillaume/chatup-ratelimiter): limits the number of messages a user can send to avoid spamming

## How to Install

On Ubuntu (tested on 14.04):
```sh
curl 'https://rawgit.com/geekuillaume/ChatUp/master/examples/install.sh' | bash
```

You can look at the [install.sh](examples/install.sh) script but basically, it installs Git, NodeJS, Nginx, clone this repo and install its NodeJS dependancies.

For other plateforms, you can port the [install.sh](examples/install.sh) script very easily.

## How to use

There is three type of installations, each of them can be used simultaneously on the same machine of scaled to N machines.

For fault-tolerancy, you need to have at least 2 of each.

### Dispatcher

The dispatcher handles the initial request of the client and redirects him to the best ChatWorker. It's also used for statistics.

To start a dispatcher, simply start the [corresponding example script](examples/dispatcher/index.js) with:

```bash
node examples/dispatcher/index.js
```

You then need to load-balance the clients requests to all the dispatchers and point the client to the correct URL.

### Chat Worker

The Chat Worker is responsible for all the messages in the chat rooms. It's composed of two elements, the NodeJS service that receives the messages from the clients and Redis, and Nginx (with PushStream) that broadcast these messages to the connected clients.

You need to have these two elements running on the same machine.

To start the ChatWorker Nginx service:
```bash
node examples/chatWorker/index.js
```

To start Nginx:
```bash
sudo /usr/local/nginx/sbin/nginx -c $PWD/examples/chatWorker/nginx.conf
```
(You need to indicate the absolute path of the nginx [configuration file](examples/chatWorker/nginx.conf))

### Redis

Redis is used for messaging between all the other components. You simply have to [install it](http://redis.io/download#installation) and point the dispatchers and ChatWorkers to it.

### Client lib

The [client lib](client/lib.js) is separated in two classes: `ChatUpProtocol` and `ChatUp`. The first handles all the messages and fault-tolerancy and the second is an integration with HTML and CSS.

The client lib is packaged with all its dependancies thanks to Browserify and exposes these two classes to the `window.ChatUp` object (so use `window.ChatUp.ChatUpProtocol` or `window.ChatUp.ChatUp`).

You can also see the [example page](examples/client/index.html) to test your installation.

You can modify use the ChatUp class as a starting point for your implementation.

## How does it works

When the client FOO want to join the channel BAR:

- the client sends a request to the dispatchers asking to join channel BAR
- the dispatchers replies with the hostname of an available worker
- the client initiates the SUBSCRIBER websocket with the worker and receives the last X messages

If the client is connected and want to send message:

- the client initiates the PUBLISHER websocket with the worker and sends its JWT
- the worker verifies it and if correct accept the connection
- the client can send a message to the PUBLISHER websocket and they will be broadcasted to every worker and every clients in the specific channel

## FAQ

### How to use SSL

SSL is supported in ChatUp but you need to attach a valid domain to your ChatWorkers as certificates cannot be attached to an IP.

To activate SSL, see the corresponding examples files.

### How to transmit user information

User information are stored inside the JSON Web Token. This token must be signed by you authentication server and passed to the client.

Inside this token, you must include a `_public` object that will be broadcast to all the rooms clients on each message. It will also be available in the dispatcher stats.

### How to ban users

You can ban a user on a channel by making a POST request to one of the dispatcher on the endpoint `/ban`.

The body of this POST request needs to be a JSON Web Token containing an array of the users to ban, channel on which to ban them and an optionnal ban expire (in seconds). Here an example of a JSON web token content:

```json
[{
  "name": "test1",
  "channel": "TestRoom",
  "expire": 30
}, {
  "name": "test2",
  "channel": "TestRoom2"
}]
```

Using the example Public/Private key pair with this example makes the following JWT:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.W3sibmFtZSI6InRlc3QxIiwiY2hhbm5lbCI6IlRlc3RSb29tIiwiZXhwaXJlIjozMH0seyJuYW1lIjoidGVzdDIiLCJjaGFubmVsIjoiVGVzdFJvb20yIn1d.nQ3S48j7J4x3NgNAhi2Qz36MebQMOxc5rHrMcm0D3bERRei2kyTVYmvcLLLJSeSyCX2KzQiV9iMnYgk4JKSEfR52tw4UUXa-7jbgmhhcDpIwo4hiIWgsZokKdo3uRX_UX8jI4ii64Tc8aq-kZiEut4WfxGjuVLlHqj-u77ileKugzhDn7bh-m0PhvdJyZGmCMCcXLnKF-TX2w-XJ_5ZvET5Ki2FH_55-W-WCrX8kPA9pSg5WLrdCunqh6p4zNFMXBxRqV3q1u3TSq4DJQkQRACAKZRqhoJz3KsYNzlxfAfhkt0OsJCwoUAOlcg95xmmSJoxwFJTCojo2lK5YTLD3yg
```

The `name` is matched against the `name` field of the `_public` sent in the client JSON web token.

### How to get messages history of a room

You can configure the Chat Worker to keep in cache (on Redis) the last N messages (look at the example file for the Chat Worker). These messages are available on an API endpoint from the Dispatcher. You can access these messages by requesting the `/messages/:channelName` route.

Remarque: These messages are only available from the API endpoints, it's different from the messages in cache served when a new user connects. Thoses messages are stored directly in Nginx and are automatically fetched on connection start by the Client Lib.

### How to use additional channels

You can configure a client to listen on additional channels to be able to send him specific notifications. To do that, just add the `additionalChannels` property in the client configuration (look at the example for a concrete use).

These channels can be used to send a message to a specific user or to broadcast a message to all your users.

### How to post a message from the API endpoint

You can use the `POST /post` API endpoint to send multiple messages to specific channels. This request body must be a JWT containing something like this:

```json
[{
  "channel": "notifications",
  "msg": "This is a test",
  "user": {
    "name": "Server"
  }
}, {
  "channel": "notifications2",
  "msg": "This is a second test",
  "user": {
    "name": "Server 2"
  }
}]
```

The message will be broadcasted to all Chat Workers and then to all clients that are subscribed to the channel.

You can also replace the `msg` field by the `ev` field in a message. This message will then be caught by the `onEv` handler in the client lib. When you use the `ev` field, you can also include the `data` field who can be anything and will be passed to each client.

## License

ChatUp is licensed on GNU GENERAL PUBLIC LICENSE.

See the [LICENSE](LICENSE) file for more information.

## History

ChatUp has been developed for Streamup by myself, Guillaume Besson. After some talks, we decided to publish it freely for other to use.

It's currently used on [Streamup](https://streamup.com) under high load and going strong.
