---
layout: post
title:  "Flask vs. Tornado vs. Klein vs. Sanic - simple performance comparision of Python web frameworks
date:   2017-09-18 00:36:33
categories: "python,webframeworks,devops,tools,api,flask,tornado,klein,sanic"
---

I have been developing a high-performance RESTful API that'd have to serve potentially more than 10k clients with many hundreds to thousands concurrent connections per second, doing IO-heavy work. I will definitely post more on this at a later point. With the criteria in mind, I set out to do some small perf test on the following frameworks:

1. [Flask](http://flask.pocoo.org)
2. [Tornado](http://www.tornadoweb.org/en/stable)
3. [Klein](https://github.com/twisted/klein)
4. [Sanic](https://github.com/channelcat/sanic)

I inteded to just run "Hello world!" on all of the above frameworks, and run test. Nothing fancy, just to give myself an idea, which one I should go with.

Sample apps:
------------

**Flask App:**

```
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'

    if __name__ == "__main__":
        app.run(host="0.0.0.0", port=8888)
```

**Tornado App**

```
import tornado.httpserver
import tornado.ioloop
import tornado.options
import tornado.web
import logging

from tornado.options import define, options

define("port", default=8888, help="run on the given port", type=int)


class MainHandler(tornado.web.RequestHandler):
    def get(self):
        self.write("Hello, world!")


def main():
    tornado.options.parse_command_line()
    application = tornado.web.Application([
        (r"/", MainHandler),
    ])
    http_server = tornado.httpserver.HTTPServer(application)
    http_server.listen(options.port, address="0.0.0.0")
    tornado.ioloop.IOLoop.instance().start()


if __name__ == "__main__":
    main()
```

**Klein App:**

```
from klein import run, route

@route("/")
def home(request):
    return "Hello World!"

run("0.0.0.0", 8888)
```


**Sanic App**

```
from sanic import Sanic
from sanic.response import json

app = Sanic()

@app.route("/")
async def test(request):
    return json({"hello": "world"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8888)
```

Test Tool:
----------

I wanted basic http test (I was running "hello, world", what more can I expect!), so I downloaded and compiled [wrk](https://github.com/wg/wrk).

I ran wrk with following options:

```
~$ ./wrk -c 200 -t 2 -d 2m --latency http://192.168.56.10:8888
```

```
-c: concurrent connections 200
-t: 2 threads, I was on a 2 core virtual machine
-d: duration 2 minute
--latency: show me latency percentiles
```


The Result:
-----------

**Flask:**

```
Running 2m test @ http://192.168.56.10:8888
  2 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   103.07ms   52.76ms   1.76s    98.16%
    Req/Sec   622.83    160.41   828.00     90.20%
  Latency Distribution
     50%   96.45ms
     75%   98.76ms
     90%  103.40ms
     99%  289.35ms
  41559 requests in 2.00m, 6.66MB read
  Socket errors: connect 0, read 56, write 0, timeout 1197
Requests/sec:    346.15
Transfer/sec:     56.79KB
```

**Tornado:**

```
Running 2m test @ http://192.168.56.10:8888
  2 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   114.38ms    4.60ms 260.94ms   92.70%
    Req/Sec     0.88k    83.60     1.01k    59.22%
  Latency Distribution
     50%  113.87ms
     75%  115.56ms
     90%  117.53ms
     99%  128.10ms
  209244 requests in 2.00m, 41.51MB read
Requests/sec:   1743.52
Transfer/sec:    354.15KB
```

**Klein:**

```
Running 2m test @ http://192.168.56.10:8888
  2 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   121.04ms   12.69ms 399.29ms   87.58%
    Req/Sec   822.95    244.25     1.46k    82.08%
  Latency Distribution
     50%  119.76ms
     75%  125.64ms
     90%  132.10ms
     99%  155.28ms
  196434 requests in 2.00m, 29.60MB read
Requests/sec:   1635.87
Transfer/sec:    252.41KB
```

**Sanic:**

```
Running 2m test @ http://192.168.56.10:8888
  2 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    40.47ms   11.46ms 106.55ms   67.99%
    Req/Sec     2.47k   381.30     3.67k    61.68%
  Latency Distribution
     50%   38.91ms
     75%   46.42ms
     90%   56.29ms
     99%   72.22ms
  589355 requests in 2.00m, 71.94MB read
Requests/sec:   4907.53
Transfer/sec:    613.44KB
```

Conclusion:
-----------

Sanic is the clear winner with little less than 5k RPS and 56ms latency at 90th percentile. It draws power from async IO in python 3.5 or above. I ran `sanic` in python 3.5 virtual env, rest in python 2.7. Comparing `Tornado` and `Klein`, the formar gives slightly better performance. `Flask` is the slowest of all, obviously, because it's not async framework. When I ran `Tornado` & `Klein` in python 3.5, I got slightly better RPS, about 100 RPS more.

So looks like I will use `sanic` and `python 3.6` to build my high-performance API server, and thus my venture into async programming starts. As promised, I will definitely make the project available on my github page.
