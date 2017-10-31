---
layout: post
title:  "Exploring D-bus for async application programming"
date:   2017-10-31 18:44:53
categories: "dbus,python,api,sanic"
---

While I have been waiting for my first child to be born, I wanted to learn a few things about [D-bus](https://www.freedesktop.org/wiki/Software/dbus/). D-bus or Desktop Bus has been around for a really long time, it's used widely in SystemD, gnome, KDE. Think of D-bus as Kafka/ZeroMQ for native desktop apps. It's not a kernel feature (I think there's BUS-1 for that), rather a userspace broker & lib. There's `dbus-daemon` that acts as a broker, you can connect to it using Domain Sockets or TCP/IP sockets, choice is yours (usually it'd be abstracted away from the application programmer by the lower level language specific libs).

Wikipedia has a really nice page describing the architecture: [click here](https://en.wikipedia.org/wiki/D-Bus)

```
  P1        P2        P3        P4        P5
  |         |         |         |         |
  v         v         v         v         v
-----------------Dbus Message Bus-------------------
  ^         ^         ^         ^         ^
  |         |         |         |         |
  P6        P7        P8        P9        P10
```

I wanted to explore more on how we can use D-bus for a typical REST API service with following setup:

```
Client -> Nginx -> Python Sanic Workers -> Python light weight backend service
```

D-bus would fit in between `Python Sanic Workers -> D-bus broker -> Python backend service`

D-bus has 2 buses: *SystemBus* and *SessionBus*. SystemBus can be used for system wide communication where as SessionBus is only limited to apps running within a specific user session.

I created a proof-of-concept app, it's available [here](https://github.com/soumyadipdm/dbus_app_server)

The app has 2 components:

1. DB Server: a simple dictionary being served over D-bus, using `pydbus` python module
2. App: Python Sanic app serving the api routes

This is how the app works:

1. Client requests `GET /api/v1/key1`
2. Sanic worker routes the request and calls `get_key` async function
3. `Connector.query` submits a query to D-bus asynchronously in `net.local.MyDBService` bus present in SystemBus invoking `net.local.MyDBService.get` method
4. When `Connector.query` returns with the result, `get_key` serializes and responds

Let's take a look at how it works in the db server app:

1. DB server registers `net.local.MyDBService` bus in SystemBus. You need to add a config xml in `/etc/dbus-1/system.d/` allowing the user (that'd be running the db app) to register bus and send/receive message to it.
2. `GLib` event loop runs, waiting for messages to come in
3. `MyDBService` is a class that defines the "retrospect xml" definig the message schema for this d-bus. `pydbus` allows you do that via the doc string or using `dbus` attribute
4. When a `net.local.MyDBService.get` method is called, `MyDBService.get` method is invoked to get a result
5. The result is then passed via D-bus
6. Any process listening to `net.local.MyDBService` messages, would get notified and they can do whatever with the result

`pydbus` lists a bunch of useful examples [here](https://github.com/LEW21/pydbus). It's exciting to see how much information you can get from SystemD using d-bus, without having to execute commands. Being a system engineer, it sparks ideas of monitoring daemon, health-check daemons that could be event based. However, `pydbus` itself requires a lot of work to make use of Python 3 `asyncio async/await`.

I haven't got a chance to do some performance benchmarking yet, though I am especially interested in how much better/worse it performs compared to a domain socket or TCP/UDP socket.

Will I use D-bus for IPC within the same host? Maybe, if the python libs for D-bus don't suck too much.
