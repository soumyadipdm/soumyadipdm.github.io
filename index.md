## Flask vs. Tornado vs. Klein vs. Sanic - simple performance comparision of Python web frameworks
*date:   2017-09-18 00:36:33*
*categories: python,webframeworks,devops,tools,api,flask,tornado,klein,sanic*

I have been developing a high-performance RESTful API that'd have to serve potentially more than 10k clients with many hundreds to thousands concurrent connections per second, doing IO-heavy work. I will definitely post more on this at a later point. With the criteria in mind, I set out to do some small perf test on the following frameworks:

1. [Flask](http://flask.pocoo.org)
2. [Tornado](http://www.tornadoweb.org/en/stable)
3. [Klein](https://github.com/twisted/klein)
4. [Sanic](https://github.com/channelcat/sanic)

I inteded to just run "Hello world!" on all of the above frameworks, and run test. Nothing fancy, just to give myself an idea, which one I should go with.

Sample apps:
------------

**Flask App:**

{% highlight python %}
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'

    if __name__ == "__main__":
        app.run(host="0.0.0.0", port=8888)
{% endhighlight %}

**Tornado App**

{% highlight python %}
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
{% endhighlight %}

**Klein App:**

{% highlight python %}
from klein import run, route

@route("/")
def home(request):
    return "Hello World!"

run("0.0.0.0", 8888)
{% endhighlight %}


**Sanic App**

{% highlight python %}
from sanic import Sanic
from sanic.response import json

app = Sanic()

@app.route("/")
async def test(request):
    return json({"hello": "world"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8888)
{% endhighlight %}

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

## Make your infrastructure searchable using Elasticsearch - Mitra
*date:   2017-07-16 01:58:33*
*categories: "python,elasticsearch,devops,tools*


As a weekend project and to play-around with Elasticsearch, I started writing [a tool](https://github.com/soumyadipdm/mitra), which to some extent borrows ideas from [osquery](https://osquery.io). This tool is to index all important information about your servers/OS/app and make them searchable from a centralized place. It uses [Elasticsearch](https://www.elastic.co) to index the documents.


Use-cases:
----------

 1. How many servers with 4.4 Linux kernel available in my infrastructure?
 2. How many servers do not have zsh installed?
 3. How many hosts with /etc/hosts size > 4k?
 4. What is the vm.swappiness setting across my infrastructure?

 ..and so on.
 
 Enjoy full power of [Lucene Search](https://www.elastic.co/guide/en/elasticsearch/reference/5.x/query-dsl-query-string-query.html#query-string-syntax) to get any data about your infrastructure
 
 Example search query:
  - Which hosts have ext4 filesystems

  `filename:"/etc/fstab" AND content:ext4`

How does it work:
-----------------

The tool has two mode: indexer and searcher

**Indexer:** It's a service that runs on all machines. Config file specifies a frequency at which this service is going to read files (specified in the config file, you can add as many as you want). You can specify a maximum size of file; if any file greater than that size, it's ignored with an error log.

A new index is created per day. So you can search historical content/stat of the files. This is important if you're troubleshooting an issue and would like to historical content of files like `/etc/sysctl.conf`or `/proc/meminfo`

Document ID is determined from sha256 hash of host name and name of the file being indexed. This has an advantage while searching, for an example, if you want to see the content of `/etc/hosts` of `xyz`machine, you can directly search the file using it's id obtained from sha256 hash.

**Searcher:** (WIP) I am working to make a lib and a cli for searching, for now I am using Kibana.

Config file:
------------
```
log:
  file: /tmp/runner.log  # log file location
  maxsize: 10  # maximum log file size in MB
  level: info  # log level

indexer:
  prefix: mitra_  # index prefix
  es_host: ubuntu02  # elasticsearch node
  es_port: 9200  # elasticsearch port
  maxsize: 10  # maximum file size, -1 stands for no limit

files:  # list of files to index into elasticsearch
  - /etc/hosts
  - /etc/resolv.conf
  - /etc/sysctl.conf
  - /etc/security/limits.conf
  - /etc/ntp.conf
  - /etc/fstab
  - /tmp/pkgs
  - /proc/meminfo
  - /proc/version

frequency: 300  # how frequently the indexer runner should run, in seconds
```

Installation:
-----------------
```
python setup.py build
python setup.py install
```

How to run:
-----------------
**Indexer**:
```
python bin/runner.py -c config/runner.yaml
```

**Note**: You can also use [Kibana](https://www.elastic.co/products/kibana) to search through the indices


## "A curious case of page cache"
*date:   2016-09-13 12:13:33*
*categories: "linux, high load, performance, page cache*

The other day I was called in for troubleshooting high load issue in a cluster. The cluster basically runs Java apps, and few nodes in the cluster consistently see high load. The load stays high (with load average hopping to 100+) for 5-6 hours, comes down for few hours and goes up once again. The issue was happening on random nodes in the cluster but at least 2 nodes were seeing this issue more frequently.


I got called in without much of context on the app side, the app owners started pressing for a kernel upgrade as they thought the issue did not happen on the nodes with latest kernel. This node has 2.6.32-504.el6.x86_64 and we are running the same kernel on at least 40k other hosts. If this is a kernel related issue, we would know. So I was not obviously convinced that the issue had anything to do with kernel.

As with any ongoing load issue, I would like start with `sar -q 1` and compare that with historical data (I usually pick up a time/day when the node was acting "normal" per app owners). This showed me there were at least 50-60 times increase in load.

Ok, so is the load CPU bound or IO bound? We need to know this to isolate the issue. If we know that issue is CPU bound, we can further drill down to compare User vs. Sys i.e User space code execution vs. Kernel space code execution time. Here I saw the IO wait was higher compared to what was considered normal. I executed `sar -u 1` to capture it while the issue was still ongoing, and compared that with historical data.

```
11:30:01 AM     CPU     %user     %nice   %system   %iowait    %steal     %idle
11:40:01 AM     all     26.88      0.31      2.51      0.44      0.00     69.86
11:50:01 AM     all     26.88      0.00      2.46      0.35      0.00     70.32
12:00:01 PM     all     26.82      0.00      2.45      0.30      0.00     70.43
12:10:01 PM     all     26.97      0.00      2.52      0.31      0.00     70.20
12:20:08 PM     all     16.82      0.00      4.14     31.58      0.00     47.46
12:30:06 PM     all      7.87      0.00      4.60     58.77      0.00     28.75
12:40:07 PM     all      8.39      0.00      4.66     59.33      0.00     27.62
12:50:07 PM     all      7.99      0.02      4.27     58.78      0.00     28.93
01:00:08 PM     all      8.95      0.26      4.73     53.93      0.00     32.13
01:10:07 PM     all      9.99      0.00      5.12     53.32      0.00     31.57
01:20:06 PM     all     21.38      0.00     10.72     28.25      0.00     39.65
01:30:05 PM     all     25.74      0.00     12.60     18.96      0.00     42.70
01:40:07 PM     all     18.33      0.00      9.94     32.05      0.00     39.68
01:50:05 PM     all     16.33      0.25      9.93     31.79      0.00     41.70
02:00:08 PM     all     22.76      0.08     11.92     24.37      0.00     40.88
02:10:05 PM     all     25.48      0.00     12.66     17.20      0.00     44.65
02:20:06 PM     all     26.67      0.00     12.80     16.88      0.00     43.65
02:30:04 PM     all     23.09      0.00     11.17     24.96      0.00     40.78
02:40:06 PM     all     23.67      0.00     11.82     24.96      0.00     39.55
02:50:03 PM     all     17.94      0.00      8.54     38.30      0.00     35.22
03:00:05 PM     all     20.31      0.14     10.02     31.82      0.00     37.71
03:10:03 PM     all     24.16      0.00     11.66     22.23      0.00     41.95
03:20:04 PM     all     27.74      0.00     13.53     14.90      0.00     43.82

03:20:04 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
03:30:05 PM     all     27.06      0.00     13.07     17.31      0.00     42.57
03:40:08 PM     all     21.53      0.07     10.34     30.79      0.00     37.26
03:50:02 PM     all     23.12      0.19     11.41     25.25      0.00     40.03
04:00:03 PM     all     24.16      0.00     11.74     21.80      0.00     42.30
04:10:03 PM     all     26.74      0.00     13.11     15.33      0.00     44.83
04:20:05 PM     all     26.98      0.00     12.88     17.25      0.00     42.89
04:30:07 PM     all     22.76      0.00     10.85     26.44      0.00     39.95
04:40:07 PM     all     23.28      0.00     11.27     27.57      0.00     37.87
04:50:01 PM     all     57.62      0.36      7.78     11.03      0.00     23.21
```

Hmm, so IO wait is obviously high. IO could be incurred by two things:

1. Application r/w

2. Kernel swapper thread finding dirty pages and passing that to flusher daemon to flush out to disks

In this case, I do see high System i.e kernel space code execution time too. Let's find out if `kswapd` is scanning for LRU pages, as that would explain higher sys% along with iowait. I executed `sar -B 1` to see this:

```
12:00:01 AM  pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff
02:30:01 AM   3152.86   2060.66   8426.85      0.30   8362.06      0.00      0.00      0.00      0.00
02:40:01 AM    626.94   2019.02   9078.88      0.18   9388.26      0.00      0.00      0.00      0.00
02:50:02 AM    806.48   2012.10   7301.27      0.17   5968.87      0.00      0.00      0.00      0.00
03:00:07 AM 694031.81   2097.44   7752.61    633.88 212117.33 375196.37      0.00 179629.27     47.88
03:10:07 AM 730073.18   1982.34   8704.16    697.53 206740.23 217153.60      0.00 182609.96     84.09
03:20:06 AM 733912.09   2179.16   7276.29    633.97 236314.30 213713.31      0.00 183655.17     85.94
03:30:04 AM 710894.55   1981.43   8260.88    637.41 198254.41 209061.44      0.00 177763.16     85.03
03:40:02 AM 723950.31   2185.91   7663.98    618.98 236609.63 209428.12      0.00 181126.74     86.49
```

`pgscank/s` is definitely high, scanning about 200MB/Second, but along with that I see high `majflt/s`, major fault is directly related to IO. So that explains high IO wait. Page cache is obviously getting thrashed by looking at `pgsteal/s` about 700mb worth of data is flushed out of page cache per second.

Let's see disk stats:

```
12:00:01 AM       DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm     %util
02:50:02 AM       sdb     33.45      0.00   1734.51     51.86      3.04     90.82      6.06     20.28
02:50:02 AM       sda     33.67      0.17   1734.51     51.51      3.15     93.63      6.11     20.56
02:50:02 AM     vgca0    293.07   1612.80   2279.47     13.28      0.09      0.16      0.16      4.60
03:00:07 AM       sdb    327.01  12111.43   1123.54     40.47     15.30     46.79      2.50     81.71
03:00:07 AM       sda    299.14  12397.52   1123.55     45.20     18.79     62.78      2.90     86.70
03:00:07 AM     vgca0   7415.11 1363553.94   3058.92    184.30     27.61      0.09      0.09     68.24
03:10:07 AM       sdb    335.46  12174.72    895.29     38.96     14.77     44.03      2.71     90.93
03:10:07 AM       sda    338.76  17134.08    895.27     53.22     20.57     60.72      2.96    100.34
03:10:07 AM     vgca0   7773.22 1430843.15   3055.96    184.47     30.33      0.09      0.09     70.23
03:20:06 AM       sdb    308.15  11905.36   1239.54     42.66     15.77     51.05      3.00     92.36
03:20:06 AM       sda    313.41  16458.11   1241.90     56.48     20.81     66.36      3.22    100.83
03:20:06 AM     vgca0   7765.77 1439454.04   3103.13    185.76     25.00      0.11      0.11     82.71
03:30:04 AM       sdb    331.49  14812.77    917.29     47.45     18.76     56.69      3.01     99.76
03:30:04 AM       sda    317.66  15146.45    914.91     50.56     24.17     76.09      3.17    100.70
03:30:04 AM     vgca0   7587.73 1391829.19   3036.32    183.83     29.94      0.09      0.09     65.09
03:40:02 AM       sdb    298.64  14346.38   1312.39     52.43     20.15     67.46      3.26     97.24
03:40:02 AM       sda    299.21  18617.07   1312.42     66.61     22.09     73.83      3.37    100.83
03:40:02 AM     vgca0   7634.43 1414939.57   3047.17    185.74     24.40      0.11      0.11     81.87
03:50:01 AM       sdb    227.84   8064.13   1475.39     41.87      8.35     36.65      3.38     77.05
03:50:01 AM       sda    281.24  17770.32   1475.39     68.43     13.91     49.46      3.46     97.38
```

We are definitely thrashing sda/sdb which are in software RAID1, serving OS and app logs. `vgca0` is Virident SSD card, serving application disk IO.

So what is happening? There's clearly demand for memory. I did not see swapping, plus we set `vm.swappiness = 1` on these clusters. So only in extreme situations we will see anonymous pages getting swapped out. Why are we thrashing page cache?

Memory is logically devided into two categories: Anonymous and Page cache. Anonymous memory is process address space minus `mmap()`-ed files, i.e it consists of stack, heap, data segment of the process. Page cache is used by VFS layer to read or write from or to disks. As disk access is way too slow, linux uses RAM to store disks blocks. Dirty pages are written to disk by per block device flusher daemon at an interval configured in `vm.dirty_writeback_centisecs` sysctl.

I wanted to know memory pressure is caused by what? Was it due to high demand for anonymous memory that can be caused by large heap or memory leak in app. Or, was it due to high insane amount of read/write that constantly asks for more page cache. Only these two condition can lead to huge amount of page cache eviction.

```
06:40:01 PM kbmemfree kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit
08:10:01 PM   2481764  63356060     96.23    203920  18315912  51221396     44.09
08:20:01 PM   1315452  64522372     98.00    207456  19469400  51209200     44.08
08:30:06 PM  21518484  44319340     67.32      4836     53060  51215004     44.09
08:40:11 PM  21445420  44392404     67.43      3048    130236  50824864     43.75
08:50:06 PM  21441128  44396696     67.43      3180    115856  50775136     43.71
09:00:08 PM  21357052  44480772     67.56      3008    176504  51126968     44.01
```

Fortunately, on these hosts we keep historical `/proc/meminfo` and `/proc/zoneinfo`, along with `/proc/buddyinfo`, these files can give me huge insights into Linux memory subsystem. From, `/proc/meminfo` I can easily figure out anonymous vs page cache increase/decrease. `/proc/zoneinfo` can tell me if number of free pages are hitting min/low water marks which would in turn explain why kswapd is woken up and subsequent page cache eviction. `/proc/buddyinfo` can point to memory fragmentation i.e shortage of higher order contiguous pages.


When the issue was on-going, I issued `grep -A 4 zone /proc/zoneinfo` per second in a loop for about 2 minutes, I did not see hitting the water marks:

```
Node 0, zone      DMA
  pages free     3933
        min      5
        low      6
        high     7
--
Node 0, zone    DMA32
  pages free     204673
        min      644
        low      805
        high     966
--
Node 0, zone   Normal
  pages free     2436172
        min      10586
        low      13232
        high     15879
--
Node 1, zone   Normal
  pages free     2732324
        min      11291
        low      14113
        high     16936
```

Just to brush up, kswapd is woken in following conditions:

A. If free <= min, kswapd starts scanning LRU pages and flushes dirty pages in async mode

B. If free <= low, kswapd does its job in synchronous mode pausing the process that needs memory be it anonymous or page cache

C. Once kswapd is awake, it will not stop until ensuring there's free >= high

In this case, I do see kswapd running, pages being scanned, but I do not see any of the watermark getting hit. This clearly says that we are seeing the aftermath/effect of the problem.

From historical `/proc/meminfo`, I see that there's about 15mb anonymous memory increment/drop. Confirming that the demand comes from page cache itself. I did not see any page allocation failure messages in `dmesg` which confirmed that there's not higher order page allocation request that are not being satisfied, causing kswapd to wake up.

I did see more than 14GB of cache drop, as well as decrease in Active(file) i.e `mmap()`-ed files.

```
Just before issue started:
-------------------------

2016-09-10 10:53:32.92709       MemTotal:       65837824 kB
2016-09-10 10:53:32.92709       MemFree:          270568 kB
2016-09-10 10:53:32.92709       Buffers:          212804 kB
2016-09-10 10:53:32.92709       Cached:         13413548 kB
2016-09-10 10:53:32.92709       SwapCached:            0 kB
2016-09-10 10:53:32.92709       Active:         44635360 kB
2016-09-10 10:53:32.92709       Inactive:       12715552 kB
2016-09-10 10:53:32.92709       Active(anon):   40444952 kB
2016-09-10 10:53:32.92709       Inactive(anon):  3284244 kB
2016-09-10 10:53:32.92709       Active(file):    4190408 kB
2016-09-10 10:53:32.92709       Inactive(file):  9431308 kB
2016-09-10 10:53:32.92709       Unevictable:           0 kB
2016-09-10 10:53:32.92709       Mlocked:               0 kB
2016-09-10 10:53:32.92709       SwapTotal:      50331644 kB
2016-09-10 10:53:32.92709       SwapFree:       50331644 kB
2016-09-10 10:53:32.92709       Dirty:              6036 kB
2016-09-10 10:53:32.92709       Writeback:           472 kB
2016-09-10 10:53:32.92709       AnonPages:      43725308 kB
2016-09-10 10:53:32.92709       Mapped:          3077196 kB
2016-09-10 10:53:32.92709       Shmem:              4292 kB
2016-09-10 10:53:32.92709       Slab:            5346076 kB
2016-09-10 10:53:32.92709       SReclaimable:     610752 kB
2016-09-10 10:53:32.92709       SUnreclaim:      4735324 kB
2016-09-10 10:53:32.92709       KernelStack:       71320 kB
2016-09-10 10:53:32.92709       PageTables:      1851204 kB
2016-09-10 10:53:32.92709       NFS_Unstable:          0 kB
2016-09-10 10:53:32.92709       Bounce:                0 kB
2016-09-10 10:53:32.92709       WritebackTmp:          0 kB
2016-09-10 10:53:32.92709       CommitLimit:    109585684 kB
2016-09-10 10:53:32.92709       Committed_AS:   79295800 kB
2016-09-10 10:53:32.92709       VmallocTotal:   34359738367 kB
2016-09-10 10:53:32.92709       VmallocUsed:      667000 kB
2016-09-10 10:53:32.92709       VmallocChunk:   34323607652 kB
2016-09-10 10:53:32.92709       HardwareCorrupted:     0 kB
2016-09-10 10:53:32.92709       AnonHugePages:         0 kB
2016-09-10 10:53:32.92709       HugePages_Total:       0
2016-09-10 10:53:32.92709       HugePages_Free:        0
2016-09-10 10:53:32.92709       HugePages_Rsvd:        0
2016-09-10 10:53:32.92709       HugePages_Surp:        0
2016-09-10 10:53:32.92709       Hugepagesize:       2048 kB
2016-09-10 10:53:32.92709       DirectMap4k:        5152 kB
2016-09-10 10:53:32.92709       DirectMap2M:     1957888 kB
2016-09-10 10:53:32.92709       DirectMap1G:    65011712 kB

Within few seconds after issue started:
--------------------------------------

2016-09-10 10:53:41.45840       MemTotal:       65837824 kB
2016-09-10 10:53:41.45840       MemFree:        14086836 kB
2016-09-10 10:53:41.45840       Buffers:            2372 kB
2016-09-10 10:53:41.45840       Cached:           192460 kB
2016-09-10 10:53:41.45840       SwapCached:            0 kB
2016-09-10 10:53:41.45840       Active:         40455816 kB
2016-09-10 10:53:41.45840       Inactive:        3469072 kB
2016-09-10 10:53:41.45840       Active(anon):   40450852 kB
2016-09-10 10:53:41.45840       Inactive(anon):  3284244 kB
2016-09-10 10:53:41.45840       Active(file):       4964 kB
2016-09-10 10:53:41.45840       Inactive(file):   184828 kB
2016-09-10 10:53:41.45840       Unevictable:           0 kB
2016-09-10 10:53:41.45840       Mlocked:               0 kB
2016-09-10 10:53:41.45840       SwapTotal:      50331644 kB
2016-09-10 10:53:41.45840       SwapFree:       50331644 kB
2016-09-10 10:53:41.45840       Dirty:              2444 kB
2016-09-10 10:53:41.45840       Writeback:            24 kB
2016-09-10 10:53:41.45840       AnonPages:      43730872 kB
2016-09-10 10:53:41.45840       Mapped:             7212 kB
2016-09-10 10:53:41.45840       Shmem:              4292 kB
2016-09-10 10:53:41.45840       Slab:            4981736 kB
2016-09-10 10:53:41.45840       SReclaimable:     224684 kB
2016-09-10 10:53:41.45840       SUnreclaim:      4757052 kB
2016-09-10 10:53:41.45840       KernelStack:       71488 kB
2016-09-10 10:53:41.45840       PageTables:      1851552 kB
2016-09-10 10:53:41.45840       NFS_Unstable:          0 kB
2016-09-10 10:53:41.45840       Bounce:                0 kB
2016-09-10 10:53:41.45840       WritebackTmp:          0 kB
2016-09-10 10:53:41.45840       CommitLimit:    109585684 kB
2016-09-10 10:53:41.45840       Committed_AS:   79298060 kB
2016-09-10 10:53:41.45840       VmallocTotal:   34359738367 kB
2016-09-10 10:53:41.45840       VmallocUsed:      667000 kB
2016-09-10 10:53:41.45840       VmallocChunk:   34323607652 kB
2016-09-10 10:53:41.45840       HardwareCorrupted:     0 kB
2016-09-10 10:53:41.45840       AnonHugePages:         0 kB
2016-09-10 10:53:41.45840       HugePages_Total:       0
2016-09-10 10:53:41.45840       HugePages_Free:        0
2016-09-10 10:53:41.45840       HugePages_Rsvd:        0
2016-09-10 10:53:41.45840       HugePages_Surp:        0
2016-09-10 10:53:41.45840       Hugepagesize:       2048 kB
2016-09-10 10:53:41.45840       DirectMap4k:        5152 kB
2016-09-10 10:53:41.45840       DirectMap2M:     1957888 kB
2016-09-10 10:53:41.45840       DirectMap1G:    65011712 kB

When the issue started:
-----------------------

2016-09-10 10:53:35.01732       MemTotal:       65837824 kB
2016-09-10 10:53:35.01732       MemFree:         2177640 kB
2016-09-10 10:53:35.01732       Buffers:          191812 kB
2016-09-10 10:53:35.01732       Cached:         11538272 kB
2016-09-10 10:53:35.01732       SwapCached:            0 kB
2016-09-10 10:53:35.01732       Active:         44593004 kB
2016-09-10 10:53:35.01732       Inactive:       10865308 kB
2016-09-10 10:53:35.01732       Active(anon):   40447192 kB
2016-09-10 10:53:35.01732       Inactive(anon):  3284244 kB
2016-09-10 10:53:35.01732       Active(file):    4145812 kB
2016-09-10 10:53:35.01732       Inactive(file):  7581064 kB
2016-09-10 10:53:35.01732       Unevictable:           0 kB
2016-09-10 10:53:35.01732       Mlocked:               0 kB
2016-09-10 10:53:35.01732       SwapTotal:      50331644 kB
2016-09-10 10:53:35.01732       SwapFree:       50331644 kB
2016-09-10 10:53:35.01732       Dirty:              4348 kB
2016-09-10 10:53:35.01732       Writeback:          3408 kB
2016-09-10 10:53:35.01732       AnonPages:      43726936 kB
2016-09-10 10:53:35.01732       Mapped:          3078376 kB
2016-09-10 10:53:35.01732       Shmem:              4292 kB
2016-09-10 10:53:35.01732       Slab:            5332632 kB
2016-09-10 10:53:35.01732       SReclaimable:     596220 kB
2016-09-10 10:53:35.01732       SUnreclaim:      4736412 kB
2016-09-10 10:53:35.01732       KernelStack:       71328 kB
2016-09-10 10:53:35.01732       PageTables:      1851200 kB
2016-09-10 10:53:35.01732       NFS_Unstable:          0 kB
2016-09-10 10:53:35.01732       Bounce:                0 kB
2016-09-10 10:53:35.01732       WritebackTmp:          0 kB
2016-09-10 10:53:35.01732       CommitLimit:    109585684 kB
2016-09-10 10:53:35.01732       Committed_AS:   79295668 kB
2016-09-10 10:53:35.01732       VmallocTotal:   34359738367 kB
2016-09-10 10:53:35.01732       VmallocUsed:      667000 kB
2016-09-10 10:53:35.01732       VmallocChunk:   34323607652 kB
2016-09-10 10:53:35.01732       HardwareCorrupted:     0 kB
2016-09-10 10:53:35.01732       AnonHugePages:         0 kB
2016-09-10 10:53:35.01732       HugePages_Total:       0
2016-09-10 10:53:35.01732       HugePages_Free:        0
2016-09-10 10:53:35.01732       HugePages_Rsvd:        0
2016-09-10 10:53:35.01732       HugePages_Surp:        0
2016-09-10 10:53:35.01732       Hugepagesize:       2048 kB
2016-09-10 10:53:35.01732       DirectMap4k:        5152 kB
2016-09-10 10:53:35.01732       DirectMap2M:     1957888 kB
2016-09-10 10:53:35.01732       DirectMap1G:    65011712 kB

While issue is ongoing:
-----------------------

2016-09-10 13:15:15.72978	MemTotal:       65837824 kB
2016-09-10 13:15:15.72978	MemFree:        14385856 kB
2016-09-10 13:15:15.72978	Buffers:            2760 kB
2016-09-10 13:15:15.72978	Cached:           106128 kB
2016-09-10 13:15:15.72978	SwapCached:            0 kB
2016-09-10 13:15:15.72978	Active:         40289400 kB
2016-09-10 13:15:15.72978	Inactive:        3380348 kB
2016-09-10 13:15:15.72978	Active(anon):   40280948 kB
2016-09-10 13:15:15.72978	Inactive(anon):  3284192 kB
2016-09-10 13:15:15.72978	Active(file):       8452 kB
2016-09-10 13:15:15.72978	Inactive(file):    96156 kB
2016-09-10 13:15:15.72978	Unevictable:           0 kB
2016-09-10 13:15:15.72978	Mlocked:               0 kB
2016-09-10 13:15:15.72978	SwapTotal:      50331644 kB
2016-09-10 13:15:15.72978	SwapFree:       50331644 kB
2016-09-10 13:15:15.72978	Dirty:              3372 kB
2016-09-10 13:15:15.72978	Writeback:           544 kB
2016-09-10 13:15:15.72978	AnonPages:      43560988 kB
2016-09-10 13:15:15.72978	Mapped:            21236 kB
2016-09-10 13:15:15.72978	Shmem:              4344 kB
2016-09-10 13:15:15.72978	Slab:            4912168 kB
2016-09-10 13:15:15.72978	SReclaimable:     147260 kB
2016-09-10 13:15:15.72978	SUnreclaim:      4764908 kB
2016-09-10 13:15:15.72978	KernelStack:       72216 kB
2016-09-10 13:15:15.72978	PageTables:      1858144 kB
2016-09-10 13:15:15.72978	NFS_Unstable:          0 kB
2016-09-10 13:15:15.72978	Bounce:                0 kB
2016-09-10 13:15:15.72978	WritebackTmp:          0 kB
2016-09-10 13:15:15.72978	CommitLimit:    109585684 kB
2016-09-10 13:15:15.72978	Committed_AS:   79433268 kB
2016-09-10 13:15:15.72978	VmallocTotal:   34359738367 kB
2016-09-10 13:15:15.72978	VmallocUsed:      667000 kB
2016-09-10 13:15:15.72978	VmallocChunk:   34323607652 kB
2016-09-10 13:15:15.72978	HardwareCorrupted:     0 kB
2016-09-10 13:15:15.72978	AnonHugePages:         0 kB
2016-09-10 13:15:15.72978	HugePages_Total:       0
2016-09-10 13:15:15.72978	HugePages_Free:        0
2016-09-10 13:15:15.72978	HugePages_Rsvd:        0
2016-09-10 13:15:15.72978	HugePages_Surp:        0
2016-09-10 13:15:15.72978	Hugepagesize:       2048 kB
2016-09-10 13:15:15.72978	DirectMap4k:        5152 kB
2016-09-10 13:15:15.72978	DirectMap2M:     1957888 kB
2016-09-10 13:15:15.72978	DirectMap1G:    65011712 kB
```

I have a habbit of pausing for a moment to review what I have gathered so far and what conclusion I have derived as of now, while still investigating. Keeps me focused and better organized.

So let's see what I understood from all of this data: There's a huge amount of IO that asks for more pages from page cache (probably it's reading or writing lots of unique blocks), as page cache demand increases, kswapd is woken up to free LRU pages in the beginning. The demand has to continue so much that, kswapd has to touch even the active pages. This aggressiveness comes from `vm.swappiness = 1` where we absolutely preferring page cache eviction over swapping.

Now, while flusher daemon is writting dirty pages to disk, processes are having to read directly from disk (because kswapd freed those active page cache pages). This incurs lots of pressure on spinning disks. kswapd and flusher daemon frequently ends being in `D` state. This further affects overall system load.

So the questions remain to be answered are:

1. What kind of IO is triggering page cache eviction? Disk IO? Is it read or write?

2. If it's disk IO, which disk is being accessed?

Once again I jumped into the historical sar data. Got to a time when the issue was just starting up, I saw huge reads in /dev/md3 (with /dev/sda and /dev/sdb as backing device). Could that be the cause or effect?

Remember while troubleshooting (especially performance related), it's easy to confuse cause and effect. But there's a drastic difference between them and your success is solely dependent upon whether you're able to differentiate between the two.

With that in my mind, I decided to look at other disk i.e SSD card. Till now, I just "assumed" that there could not be any issue with SSD, because I did not see wait time more than 1ms, which is good.

Any assumption is dangerous! yes, it was proven right once again. Now that I started focusing on SSD, I see utilizing constantly hitting 100%, even though wait time is normal. Moreover, the time when SSD util% goes up is about 10 second ahead of kswapd waking up and dropping page cache.

```
12:00:01 AM       DEV       tps  rd_sec/s  wr_sec/s  avgrq-sz  avgqu-sz     await     svctm   %util
08:50:06 PM       sdb    341.93  13827.24    684.79     42.44     33.09     95.33      2.94    100.47
08:50:06 PM       sda    337.96  15133.46    680.46     46.79     35.69    105.43      2.99    101.01
08:50:06 PM       md2      0.00      0.00      0.00      0.00      0.00      0.00      0.00    0.00
08:50:06 PM       md0    251.12  12683.38     22.70     50.60      0.00      0.00      0.00    0.00
08:50:06 PM       md1     65.21   5574.27     31.54     85.97      0.00      0.00      0.00    0.00
08:50:06 PM       md3    543.21  10702.46    619.42     20.84      0.00      0.00      0.00    0.00
08:50:06 PM     vgca0   5277.88 972442.71   2142.69    184.65     19.39      0.10      0.10    51.40
09:00:04 PM       sdb    314.37  12900.40    664.04     43.15     28.97     92.15      3.20    100.45
09:00:04 PM       sda    321.55  14513.24    664.46     47.20     36.19    112.61      3.14    101.03
09:00:04 PM       md2      0.00      0.00      0.00      0.00      0.00      0.00      0.00    0.00
09:00:04 PM       md0    232.87  11197.39     24.17     48.19      0.00      0.00      0.00     0.00
09:00:04 PM       md1     68.28   5153.17      7.47     75.58      0.00      0.00      0.00      0.00
09:00:04 PM       md3    639.13  11063.15    622.82     18.28      0.00      0.00      0.00      0.00
09:00:04 PM     vgca0   5488.65 1013505.28   2271.56    185.07     20.34      0.09      0.09     51.53
09:10:07 PM       sdb    341.20  15758.06    645.46     48.08     35.81    104.48      2.95    100.79
09:10:07 PM       sda    320.22  16648.49    652.67     54.03     37.27    116.41      3.15    101.00
09:10:07 PM       md2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
09:10:07 PM       md0    249.07  11011.35     23.18     44.30      0.00      0.00      0.00      0.00
09:10:07 PM       md1     60.25   5100.21      9.25     84.80      0.00      0.00      0.00      0.00
09:10:07 PM       md3    669.98  16296.21    610.18     25.23      0.00      0.00      0.00      0.00
09:10:07 PM     vgca0   5432.36 992804.00   2397.85    183.20     19.89      0.09      0.09     51.20
09:20:02 PM       sdb    192.75  10170.07   1564.15     60.88     16.72     87.53      3.40     65.55
09:20:02 PM       sda    225.89  19398.22   1557.61     92.77     19.13     84.68      3.23     72.96
09:20:02 PM       md2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
09:20:02 PM       md0    144.04   6125.15     22.19     42.68      0.00      0.00      0.00      0.00
09:20:02 PM       md1     54.23   3544.47     31.86     65.95      0.00      0.00      0.00      0.00
09:20:02 PM       md3    487.57  19896.40   1493.94     43.87      0.00      0.00      0.00      0.00
09:20:02 PM     vgca0   5484.36 929067.86   4087.08    170.15     19.38      0.08      0.08     43.80
09:30:01 PM       sdb     34.23     82.20   1865.28     56.89      3.30     96.54      6.10     20.89
09:30:01 PM       sda     90.07   8808.71   1865.28    118.50      3.78     41.98      4.24     38.20
09:30:01 PM       md2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
09:30:01 PM       md0     10.62    127.16     37.27     15.49      0.00      0.00      0.00      0.00
09:30:01 PM       md1     19.04    228.22    122.64     18.42      0.00      0.00      0.00      0.00
09:30:01 PM       md3    267.24   8535.46   1695.42     38.28      0.00      0.00      0.00      0.00
09:30:01 PM     vgca0    267.29   1538.20   2073.91     13.51      0.07      0.14      0.14      3.67
09:40:01 PM       sdb     31.30      1.07   1847.10     59.04      2.93     93.65      5.98     18.71
09:40:01 PM       sda     31.53      3.11   1847.10     58.67      3.08     97.75      6.01     18.94
09:40:01 PM       md2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
09:40:01 PM       md0      4.27      2.78     32.67      8.31      0.00      0.00      0.00      0.00
09:40:01 PM       md1     13.29      0.03    106.30      8.00      0.00      0.00      0.00      0.00
09:40:01 PM       md3    212.50      1.36   1698.89      8.00      0.00      0.00      0.00      0.00
09:40:01 PM     vgca0    279.33   1800.55   2161.58     14.18      0.09      0.15      0.15      4.31
09:50:01 PM       sdb     32.55      0.57   1851.13     56.89      3.24     99.46      6.13     19.95
09:50:01 PM       sda     32.98      1.40   1851.13     56.16      3.29     99.82      6.01     19.81
09:50:01 PM       md2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
09:50:01 PM       md0      4.46      0.73     34.90      7.99      0.00      0.00      0.00      0.00
09:50:01 PM       md1     14.70      0.99    116.60      8.00      0.00      0.00      0.00      0.00
09:50:01 PM       md3    211.21      0.07   1689.65      8.00      0.00      0.00      0.00      0.00
09:50:01 PM     vgca0    277.86   2174.51   2134.87     15.51      0.08      0.14      0.14      3.82
10:00:01 PM       sdb     35.99     70.40   2117.01     60.78      3.49     97.04      5.82     20.95
10:00:01 PM       sda    104.11   8918.24   2117.02    106.00      4.31     41.43      2.82     29.31
10:00:01 PM       md2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:00:01 PM       md0      5.78      1.37     45.47      8.10      0.00      0.00      0.00      0.00
10:00:01 PM       md1     16.50      0.00    131.96      8.00      0.00      0.00      0.00      0.00
10:00:01 PM       md3    325.08   8987.16   1929.13     33.58      0.00      0.00      0.00      0.00
10:00:01 PM     vgca0    269.02   1181.11   2104.41     12.21      0.10      0.18      0.18      4.82
10:10:02 PM       sdb     34.08      0.37   1903.82     55.87      3.10     90.95      5.99     20.40
10:10:02 PM       sda     34.24      1.55   1903.79     55.65      3.32     96.90      6.23     21.34
10:10:02 PM       md2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:10:02 PM       md0      5.52      1.04     43.60      8.08      0.00      0.00      0.00      0.00
10:10:02 PM       md1     19.31      0.05    154.42      8.00      0.00      0.00      0.00      0.00
10:10:02 PM       md3    212.04      0.72   1696.18      8.00      0.00      0.00      0.00      0.00
10:10:02 PM     vgca0    259.92    998.14   2039.20     11.69      0.09      0.15      0.15      4.00
10:20:01 PM       sdb     32.94      2.82   1834.64     55.79      3.18     96.50      5.94     19.57
10:20:01 PM       sda     33.61     61.41   1834.66     56.41      3.16     93.97      5.99     20.15
10:20:01 PM       md2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:20:01 PM       md0      4.53      1.59     35.28      8.15      0.00      0.00      0.00      0.00
10:20:01 PM       md1      9.50      0.00     76.03      8.00      0.00      0.00      0.00      0.00
10:20:01 PM       md3    214.78     62.51   1712.60      8.26      0.00      0.00      0.00      0.00
10:20:01 PM     vgca0    256.06   1633.22   1982.75     14.12      0.06      0.12      0.12      3.10
10:30:01 PM       sdb     33.01      0.03   1855.51     56.22      3.14     95.17      5.94     19.60
10:30:01 PM       sda     33.08      2.56   1855.51     56.16      3.24     97.90      6.05     20.02
10:30:01 PM       md2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
10:30:01 PM       md0      5.30      2.40     42.18      8.41      0.00      0.00      0.00      0.00
10:30:01 PM       md1     13.13      0.01    105.03      8.00      0.00      0.00      0.00      0.00
10:30:01 PM       md3    212.39      0.05   1699.04      8.00      0.00      0.00      0.00      0.00
10:30:01 PM     vgca0    253.72   1529.33   1968.26     13.79      0.09      0.16      0.16      4.13
```

So finally here's what is happening:

1. App submits enormous read requests from SSD card continuously

2. Althogh SSD is able to respond in < 1ms for the above read requests, these reads saturate page cache

3. As pressure on page cache increases, kswapd is awaken: "The Kraken is out of the sea"

4. kswapd asks flusher daemons to write out dirty pages to disk, while putting LRU pages in the free list

5. Step 1 continues, step 3 continues. kswapd now starts to put Active pages from page cache into the free list (due to aggressive `vm.swappiness = 1`)

6. Processes start to encounter Major faults, incurring disk IO

7. As flusher daemon keeps the disk busy, major faults add in to the problem, creating death for spinning disks

8. We see high IO wait, lots of processes in `D` state


I explained the whole thing to the app owners, with data to back it up. They checked and found there's been unusual read requests submitted to the app from particular set of clients and this has been going on for 3 months, the issue too started happening across cluster at the same time.

Take aways:
===========

* If you have shared a resource, you have a bottleneck. Page cache is shared for all of disk IO from *all* disks in a node

* SSD utilization being around 100% continuously could lead to a problem even though SSD is fast enough to satisfy IO requests within a milisecond

* Fast disk like SSD has the potential to saturate page cache quickly, starving other disk IOps

Happy ending!!



## "Gethostbyname vs. Getaddrinfo"
*date:   2016-09-09 19:50:33*
*categories: "system programming, linux, dns*

It's been a while since my last post, about 2 months. The other I faced some strange issue with something as simple as ``getent hosts `hostname` ``. We used this to determine IPv4 addr of a host and used that for setting SRC IP for an application. I know this is not at all ideal way of doing it, this is how it has been for a long time with the legacy app.

Issue was that ``getent hosts `hostname` `` started taking really long time (about 20 seconds or more) to resolve to IP, whereas ``getent ahosts `hostname` `` was getting me answer just within 1 second. I wanted to know why. After lots of `strace`-ing, `gdb`-ing and looking at [GNU getent source code](https://fossies.org/dox/glibc-2.24/getent_8c_source.html), I found that `getent hosts` uses `gethostbyname`, whereas `getent ahosts` uses `getaddrinfo`.

So what's the difference? Both tried querying DNS using `nscd` socket, upon timing out, it reads `/etc/resolv.conf` and queries each of them for each "key" until it has gotten an answer (could be positive or negetive).

So how are these "keys" formed? Basically if you query something like "123.abc.xyz.com", and if `/etc/resolv.conf` has `domain abc.xyz.com`, `gethostbyname` is actually querying for below entries:

```
A 123.abc.xyz.com
AAAA 123.abc.xyz.com
AAAA 123.abc.xyz.com.abc.xyz.com
AAAA 123.abc.xyz.com.xyz.com
```

Crazy right?

I am not sure why only AAAA record and not A record for the extra stupid queries. If gets NXDOMAIN, it issues same query to another DNS server.

On the other hand, `getaddrinfo` handles the query smartly. It just queries only one of the DNS (first `nameserver` entry in /etc/resolv.conf) server, one for A record and another for AAAA record.

With a combination of `strace` and `tcpdump` I was able to see why it was taking so long for ``getent hosts `hostname` `` to finish. Now I know why it's advised to use getaddrinfo.



## Go, here I come
*date:   2016-07-23 01:22:20*
*categories: programming, go*

I have trying to learn [Go](http://golang.org/) for a while, but for one or the other reason could not start it properly. The other day I thought it's the right time to start learning Go, I have an idea in my mind (more of that will come later) and Go would a perfect fit to deal with it.

So I grabbed ["The Go Programming Language"](http://www.amazon.in/Go-Programming-Language-Alan-Donovan/dp/9332569711) book and started reading. It's fun of course to learn Go, I know C and I can see the similarities (not many, but look and feel are quite similar).

I have "The C Programming Language", "Python Cookbook", and now "The Go Programming Language", all side by side. It's going to be fun!



## When to prefer fork() over vfork()
*date:   2016-07-09 15:34:20*
*categories: linux, C, system call*

It's been a very long time since my last post. And all this time even though I was real busy in personal and professional fronts, my fingers etched to write something down on this page.

It's supposed to be my journal, which I can go through at a later point to remind myself of the things interested me at some point.

Anyways, the other day I was going through the source code of famous "nix" program `crontab` in [Fedora repository](https://git.fedorahosted.org/cgit/cronie.git/tree/). It's a small program but has lots of surprises hidden in it. I liked how considerations on clock delay caused by process schedular etc. were accounted for.

While going through the code, I saw comments on preferring `fork()` over `vfork()`. Everyone knows that `vfork()` is efficient in terms of speed and memory consumption as it's "copy-on-write", the parent address space is copied "on-demand". This makes `vfork()` an ideal candidate for executing external programs by means of executing `execve()` immediately after forking off of parent process.

Now there could be a situation, when you need to do some sort of lengthy work before you actually `execve()` the external program. During this time, the parent process will be paused. So if in a case where you cannot immediately run `execve()` and parent process needs to go on doing it's own work without waiting for child to run `execve()`, it makes sense to prefer `fork()` over `vfork()`.

Here's the comment in [do_command.c](https://git.fedorahosted.org/cgit/cronie.git/tree/src/do_command.c#n55) that caught my eye and made me appreciate the amount of thinking/considerations even a small program like crontab had to have.

{% highlight c linenos %}
	Debug(DPROC, ("[%ld] do_command(%s, (%s,%ld,%ld))\n",
			(long) pid, e->cmd, u->name,
			(long) e->pwd->pw_uid, (long) e->pwd->pw_gid));

		/* fork to become asynchronous -- parent process is done immediately,
		 * and continues to run the normal cron code, which means return to
		 * tick().  the child and grandchild don't leave this function, alive.
		 *
		 * vfork() is unsuitable, since we have much to do, and the parent
		 * needs to be able to run off and fork other processes.
		 */
		switch (fork()) {
	case -1:
		log_it("CRON", pid, "CAN'T FORK", "do_command", errno);
		break;
	case 0:
		/* child process */
		acquire_daemonlock(1);
		/* Set up the Red Hat security context for both mail/minder and job processes:
		 */
		if (cron_set_job_security_context(e, u, &jobenv) != 0) {
			_exit(ERROR_EXIT);
		}
		ev = child_process(e, jobenv);
#ifdef WITH_PAM
		cron_close_pam();
#endif
		env_free(jobenv);
		Debug(DPROC, ("[%ld] child process done, exiting\n", (long) getpid()));
		_exit(ev);
		break;
	default:
		/* parent process */
		break;
	}
	Debug(DPROC, ("[%ld] main process returning to work\n", (long) pid));
{% endhighlight %}



## Dissecting Linux Performance Bottlenecks: CPU
*date:   "2016-02-21 19:56:49"*
*categories: "linux, performance*

This is the first of my "Linux Performance" series. In this part, I am going to be focusing on CPU and Linux process scheduler as performance bottlenecks, how to detect it etc. I will later write a series focusing on tuning resources etc.

uptime(1):
-------

While discovering the entire system, `uptime` can come in-handy. It gives a glimps of whether a system has been busy in the last 15 minutes and if it's going better or worse. It's an average of __total number of runnable (running + waiting to run) and uninterruptable (i.e waiting for IO etc) processes divided by the number of cpu (this includes CPU hardware threads)__

{% highlight shell %}
[unixguy@app02 ~]$ uptime
 20:19:29 up 52 min,  1 user,  load average: 0.34, 0.09, 0.07
{% endhighlight %}

In this case, I see that system was sitting almost idle, but in the last minute some processes ran. But keep in mind that these processes could have been CPU bound or IO bound or both. `uptime` does not give any clue to that.

pidstat(1):
--------

If I want to know what's using cpu, I run `pidstat` to see what's going on, `-I` option devides the %CPU count by the total number of CPU threads. A CPU saturation can be detected if the %CPU goes >= 100%

{% highlight shell %}
[unixguy@app02 ~]$ pidstat -I 1
Linux 3.10.0-229.el7.x86_64 (app02.local.net)   02/21/2016      _x86_64_       (2 CPU)

08:36:08 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:09 PM 10000      5540  200.00    0.00    0.00  100.50     1  sysbench

08:36:09 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:10 PM 10000      5565    0.00    1.00    0.00    0.51     0  pidstat

08:36:10 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command

08:36:11 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:12 PM 10000      5565    0.00    1.00    0.00    0.50     0  pidstat

08:36:12 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:13 PM 10000      5567   86.14    0.00    0.00   43.94     0  sysbench

08:36:13 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:14 PM 10000      5567  201.00    1.00    0.00  103.06     0  sysbench

08:36:14 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:15 PM 10000      5565    0.00    0.99    0.00    0.51     0  pidstat
08:36:15 PM 10000      5567  198.02    0.00    0.00  102.04     0  sysbench

08:36:15 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:16 PM 10000      5565    1.00    0.00    0.00    0.52     0  pidstat
08:36:16 PM 10000      5567  200.00    1.00    0.00  104.15     0  sysbench

08:36:16 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:17 PM 10000      5567  199.01    0.00    0.00  100.00     0  sysbench

08:36:17 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:18 PM 10000      5567  198.02    0.99    0.00  103.08     0  sysbench

08:36:18 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:19 PM 10000      5565    0.00    0.99    0.00    0.50     0  pidstat
08:36:19 PM 10000      5567  199.01    0.00    0.00   99.50     0  sysbench

08:36:19 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:20 PM 10000      5567  199.01    0.00    0.00  101.52     0  sysbench

08:36:20 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:21 PM 10000      5567  200.00    0.00    0.00  102.04     0  sysbench

08:36:21 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:22 PM 10000      5565    1.00    0.00    0.00    0.53     0  pidstat
08:36:22 PM 10000      5567  200.00    2.00    0.00  106.32     0  sysbench

08:36:22 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command

08:36:23 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:36:24 PM 10000      5565    0.00    1.00    0.00    0.50     0  pidstat
^C

Average:      UID       PID    %usr %system  %guest    %CPU   CPU  Command
Average:    10000      5565    0.12    0.31    0.00    0.22     -  pidstat
{% endhighlight %}

As it shows, in this example sysbench was running and at times consumed >100% cpu. That's a saturation point because I have only two CPU threads.

{% highlight shell %}
[unixguy@app02 ~]$ grep -c processor /proc/cpuinfo
2
{% endhighlight %}

Other important fields are %usr and %system. High %usr can be caused by application doing calculation, producing hashes etc. High %system can be caused by application calling too many syscalls or syscalls taking high cpu time. A %usr to %system ratio of 20/80 could mean that the process is IO bound, and could turn out to be a performance issue depending upon your environment. But more on that later.

_Quick Tip_: If you want to know what a particular application is doing at the moment, you can take a look at `/proc/XXX/wchan`, `/proc/XXX/status`, `/proc/XXX/syscall` files, where `XXX` stands for the PID of the process. For a process which is completely running in userspace, you would not see any syscall entry, `wchan` will be `0`, and `status` file would say state as running. To get more insight of such a process, you may need to attach GDB to it (if it's an ELF binary process) or may invoke other debuggers (if the process is of interpretive language like Python, Ruby, JAVA etc.)

mpstat(1):
-------

For an SMP (Symmetric Multiprocessing) system, it is worth verifying if the application is able to take advantage of multiple CPU cores and hardware threads. Suppose if the `pidstat` output above showed me 50% CPU time and ~0% system time, it would mean that the process is using only 1 of 2 cpu cores available (also watch out for CPU column, which shows the processor number, as reported by /proc/cpuinfo, the process was running on during the smapling time). If the process is reported to be "slower than expected", I would investigate more if the process is able to use more than one core at all. This is where mpstat comes in handy.

{% highlight shell %}
[unixguy@app02 ~]$ mpstat -P ALL 1
Linux 3.10.0-229.el7.x86_64 (app02.local.net)   02/21/2016      _x86_64_       (2 CPU)

09:13:14 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:15 PM  all   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
09:13:15 PM    0  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
09:13:15 PM    1    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00

09:13:15 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:16 PM  all   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
09:13:16 PM    0   24.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   76.00
09:13:16 PM    1   76.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   24.00

09:13:16 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:17 PM  all   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
09:13:17 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
09:13:17 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00

09:13:17 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:18 PM  all   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
09:13:18 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
09:13:18 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00

09:13:18 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:19 PM  all   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
09:13:19 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
09:13:19 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00

09:13:19 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:20 PM  all   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
09:13:20 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
09:13:20 PM    1   99.01    0.00    0.00    0.00    0.00    0.99    0.00    0.00    0.00    0.00

09:13:20 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:21 PM  all   49.75    0.00    0.00    0.00    0.00    0.50    0.00    0.00    0.00   49.75
09:13:21 PM    0    0.00    0.00    0.00    0.00    0.00    1.00    0.00    0.00    0.00   99.00
09:13:21 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00

09:13:21 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:22 PM  all   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
09:13:22 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
09:13:22 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00

09:13:22 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:23 PM  all   50.25    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   49.75
09:13:23 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
09:13:23 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00

09:13:23 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:24 PM  all   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
09:13:24 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
09:13:24 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00

09:13:24 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:25 PM  all   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
09:13:25 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
09:13:25 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00

09:13:25 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:26 PM  all   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
09:13:26 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
09:13:26 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00

09:13:26 PM  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
09:13:27 PM  all   50.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00   50.00
09:13:27 PM    0    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00  100.00
09:13:27 PM    1  100.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00    0.00
^C

Average:     CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest  %gnice   %idle
Average:     all   50.00    0.00    0.00    0.00    0.00    0.04    0.00    0.00    0.00   49.96
Average:       0    9.54    0.00    0.00    0.00    0.00    0.08    0.00    0.00    0.00   90.38
Average:       1   90.32    0.00    0.00    0.00    0.00    0.08    0.00    0.00    0.00    9.60
{% endhighlight %}

In this example, we can see that utilization of one of the cores is always almost 100%. The other core is almost idle. This could indicate that the appkication is probably not multithreaded or unable to use multiple cores. Before diving into application troubleshooting, I would check for any hardware issue. Depending upon the hardware vendor (or if it's a virtual machine you may need to check VM settings in KVM or ESXi), I would take a look at the faults reported by service processor either via `ipmitool` or via the service processor console. I have seen power and temperature issues causing CPU to disable certain cores [refer Intel SpeedStep technology](https://en.wikipedia.org/wiki/SpeedStep)

Processor Affinity and taskset(1):
----------------------------------

If it's found that the process only running on a single core, I would verify if the processor affinity of that process is set properly. `taskset` command gets and sets processor affinity of a process. Man pages of `taskset(1)` and `sched_getaffinity(2)/sched_setaffinity(2)` have good detail on the performance impact on process. Sometimes a process might want to run on a single CPU to leverage already warm L1/L2 caches.

{% highlight shell %}
[unixguy@app02 4927]$ taskset -cp 4927
pid 4927's current affinity list: 0,1
{% endhighlight %}

In the above example, PID 4927 is set to run on all available processors (two in my case).

Let's look at things from the process scheduler's perspective and see if it all looks good.

Scheduler Statistics:
---------------------

The first thing I will look is the `sched` file in proc a specific pid, it contains the scheduler statistics. The most important parts for troubleshooting are `nr_voluntary_switches` and `nr_involuntary_switches`.

`nr_voluntary_switches` stands for number of context switches due to current thread blocking i.e waiting for IO etc.

`nr_involuntary_switches` stands for number of times the kernel process scheduler has to kick the process out of running state, probably because it exhauseted its allocated cput time slice.

Now, high number of `nr_voluntary_switches` may indicate that the process is doing lots of systemcalls, we can further confirm this by looking at `pidstat` data for this process.

But, high number of (how high? you need to compare stats on a different machine with same kernel version and similar workload) `nr_involuntary_switches` may always mean that that the task is not getting enough time to run on cpu i.e probably due to too many tasks in the runnable queue, or simply because the priority of the process needs to be higher.


{% highlight shell %}
[unixguy@app02 4927]$ cat sched
bash (4927, #threads: 1)
-------------------------------------------------------------------
se.exec_start                                :      15186760.306364
se.vruntime                                  :       2768833.698349
se.sum_exec_runtime                          :           109.640759
nr_switches                                  :                 1026
nr_voluntary_switches                        :                 1025
nr_involuntary_switches                      :                    1
se.load.weight                               :                 1024
policy                                       :                    0
prio                                         :                  120
clock-delta                                  :                   59
mm->numa_scan_seq                            :                    0
numa_migrations, 0
numa_faults_memory, 0, 0, 1, 0, -1
numa_faults_memory, 1, 0, 0, 0, -1
{% endhighlight %}

## Dissecting `tail -f`: linux inotify
*date:   2016-02-17 23:06:25*
*categories: linux, C, system call*

We all know about the little programs that make up GNU coreutils, I feel we always take them for granted. They contain some extraordinary code, at times complex too.

The other day I was in need for a utility that would basically work like `tail -f` to watch important syslogs etc. So I started going through the code of `tail`, which can be found here: [http://git.savannah.gnu.org/cgit/coreutils.git/tree/src/tail.c](http://git.savannah.gnu.org/cgit/coreutils.git/tree/src/tail.c)

I knew about `inotify(7)` and many people attempting to write Python libraries based on it, but did not expect that `tail` would have already implemented it.

A bit about inotify:
--------------------

Before exploring the code of `tail`, let's first talk about `inotify(7)` and why is it so awesome. `inotify` stands for Inode Notifier, as far as I can see it was first introduced in 2005, primarily written by Robert Love. I have his book "Linux System Programming" and love every bit of it. `inotify` itself is not a system call, rather it's a collection of system calls like:  inotify_init(2), inotify_add_watch(2), inotify_rm_watch(2) etc. Using thesesystem calls, one can "monitor" inode information changes. Inode here could be of a file or for a directory. It's fully event based. So a good implementation of it does not suffer from high CPU usage, unlike that of loops calling `fstat(2)` to check for `mtime`. With `inotify(7)` one can just wait for events like `IN_MODIFY`, which is what I was looking for. When I get confused between `poll(2)`/`epoll(7)`/`select(2)` and `inotify(7)`, I try to remind myself that `poll` and its friends can be applied on any kind of IO entity like stream, socket etc. with an event trigger condition that may or may not be related to IO the entity directly, but `inotify` is designed for filesystems objects, with inode events as triggers.

*How can inotify be used?*

Here's what the man page says:

```
       To  determine  what  events have occurred, an application read(2)s from
       the inotify file descriptor.  If no events have so far occurred,  then,
       assuming  a blocking file descriptor, read(2) will block until at least
       one event occurs (unless interrupted by a signal,  in  which  case  the
       call fails with the error EINTR; see signal(7)).
```

A curious case of `tail -f`:
----------------------------

`tail` infact implements both inotify and non-inotify i.e fstat-mtime-loop based implementation used for `-f` option.

Code for the fstat version: [tail_forever](http://git.savannah.gnu.org/cgit/coreutils.git/tree/src/tail.c#n1098)

Code for the inotify version: [tail_forever_inotify](http://git.savannah.gnu.org/cgit/coreutils.git/tree/src/tail.c#n1386)

Let's go step by step:

1. Initializes a hash table for a key-value mapping for name of the file and its inotify watch descriptor

{% highlight c linenos %}
  wd_to_name = hash_initialize (n_files, NULL, wd_hasher, wd_comparator, NULL);
{% endhighlight %}

2. Descriptor mask is initialized with IN_MODIFY i.e inode modify event, as well as for other events like attribute change, file deletion or file movement (useful if the file being watched is deleted during this time)

{% highlight c linenos %}
  uint32_t inotify_wd_mask = IN_MODIFY;
  /* TODO: Perhaps monitor these events in Follow_descriptor mode also,
     to tag reported file names with "deleted", "moved" etc.  */
  if (follow_mode == Follow_name)
    inotify_wd_mask |= (IN_ATTRIB | IN_DELETE_SELF | IN_MOVE_SELF);
{% endhighlight %}

3. Starts inotiy watcher for every file specified in the command line

4. If `-F` option is specified, i.e retry if the file(s) are deleted during the watch, it starts to monitor parent directories of the files. This is done to get an event when the files reappear in the directory again

{% highlight c linenos %}
          if (follow_mode == Follow_name)
            {
              size_t dirlen = dir_len (f[i].name);
              char prev = f[i].name[dirlen];
              f[i].basename_start = last_component (f[i].name) - f[i].name;

              f[i].name[dirlen] = '\0';

               /* It's fine to add the same directory more than once.
                  In that case the same watch descriptor is returned.  */
              f[i].parent_wd = inotify_add_watch (wd, dirlen ? f[i].name : ".",
                                                  (IN_CREATE | IN_DELETE
                                                   | IN_MOVED_TO | IN_ATTRIB));

              f[i].name[dirlen] = prev;

              if (f[i].parent_wd < 0)
                {
                  if (errno != ENOSPC) /* suppress confusing error.  */
                    error (0, errno, _("cannot watch parent directory of %s"),
                           quoteaf (f[i].name));
                  else
                    error (0, 0, _("inotify resources exhausted"));
                  found_unwatchable_dir = true;
                  /* We revert to polling below.  Note invalid uses
                     of the inotify API will still be diagnosed.  */
                  break;
                }
            }
          f[i].wd = inotify_add_watch (wd, f[i].name, inotify_wd_mask);
{% endhighlight %}

5. Function returns True if for some reason like watcher could not be added etc, so that `tail` can revert to normal mtime based polling. It returns False if there are no watchable file found

{% highlight c linenos %}
  if (no_inotify_resources || found_unwatchable_dir
      || (follow_mode == Follow_descriptor && tailed_but_unwatchable))
    {
      hash_free (wd_to_name);

      errno = 0;
      return true;
    }
  if (follow_mode == Follow_descriptor && !found_watchable_file)
    return false;
{% endhighlight %}

6. It checks once again to see if files have reappeared since last checked. If so, they are added to the watch list and checks for new data in file (check_fspec() function)

7. Starts the big `while` loop

8. If `-p PID` option was specified i.e follow until PID dies, it checks if there's any change in file using `select(2)` every 1 second or number of seconds specified via `-s` option

{% highlight c linenos %}
      if (pid)
        {
          if (writer_is_dead)
            exit (EXIT_SUCCESS);

          writer_is_dead = (kill (pid, 0) != 0 && errno != EPERM);

          struct timeval delay; /* how long to wait for file changes.  */
          if (writer_is_dead)
            delay.tv_sec = delay.tv_usec = 0;
          else
            {
              delay.tv_sec = (time_t) sleep_interval;
              delay.tv_usec = 1000000 * (sleep_interval - delay.tv_sec);
            }

           fd_set rfd;
           FD_ZERO (&rfd);
           FD_SET (wd, &rfd);

           int file_change = select (wd + 1, &rfd, NULL, NULL, &delay);

           if (file_change == 0)
             continue;
           else if (file_change == -1)
             error (EXIT_FAILURE, errno, _("error monitoring inotify event"));
        }
{% endhighlight %}

9. Here comes inotify. safe_read() function tries to read the inotify file descriptior to see if the aforementioned events have occured. It blocks on that read, until an event has occured.

{% highlight c linenos %}
      if (len <= evbuf_off)
        {
          len = safe_read (wd, evbuf, evlen);
          evbuf_off = 0;

          /* For kernels prior to 2.6.21, read returns 0 when the buffer
             is too small.  */
          if ((len == 0 || (len == SAFE_READ_ERROR && errno == EINVAL))
              && max_realloc--)
            {
              len = 0;
              evlen *= 2;
              evbuf = xrealloc (evbuf, evlen);
              continue;
            }

          if (len == 0 || len == SAFE_READ_ERROR)
            error (EXIT_FAILURE, errno, _("error reading inotify event"));
        }
{% endhighlight %}

10. It checks for various events like attribute change, file deletion etc, if such event has occured for a file, removes it from the watcher list

{% highlight c linenos %}
      if (ev->mask & (IN_ATTRIB | IN_DELETE | IN_DELETE_SELF | IN_MOVE_SELF))
        {
          /* Note for IN_MOVE_SELF (the file we're watching has
             been clobbered via a rename) we leave the watch
             in place since it may still be part of the set
             of watched names.  */
          if (ev->mask & IN_DELETE_SELF)
            {
              inotify_rm_watch (wd, fspec->wd);
              hash_delete (wd_to_name, fspec);
            }

          /* Note we get IN_ATTRIB for unlink() as st_nlink decrements.
             The usual path is a close() done in recheck() triggers
             an IN_DELETE_SELF event as the inode is removed.
             However sometimes open() will succeed as even though
             st_nlink is decremented, the dentry (cache) is not updated.
             Thus we depend on the IN_DELETE event on the directory
             to trigger processing for the removed file.  */

          recheck (fspec, false);

          continue;
        }
{% endhighlight %}


11. Calls check_fspec() to check for new data in the event of IN_MODIFY. If there's new data, dump_remainder() is called to print new data i.e it seeks the pointer to the end of last read byte and reads till EOF, thus avoiding duplication. [dump_remainder](http://git.savannah.gnu.org/cgit/coreutils.git/tree/src/tail.c#n395)



## Hello Github pages!
*date:   2016-02-15 00:06:25*
*categories: general update*

So I have been trying to move to github pages for a while, mostly because I wanted to blog in the comfort of commandline and markdown.

I do have a Wordpress site [jotdownux.wordpress.com](http://jotdownux.wordpress.com) and I have been blogging there for more than a year now. But, at this point I am not sure if I will continue posting there.

I will try to write stuff every day, if not once a week at least. Let's see how things turn out.
