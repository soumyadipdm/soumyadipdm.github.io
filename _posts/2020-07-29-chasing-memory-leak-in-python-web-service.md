---
layout: post
title:  "Chasing Memory Leak In a Python Web Service - Story Of Struggle"
date:   2020-07-29 13:00:53
categories: "python,api,sanic,gunicorn,couchbase"
---

TL;DR - I wish there were mature tools to inpsect a running Python process heap. At least something like [jmap/jhat](https://manpages.debian.org/unstable/openjdk-8-jdk-headless/jmap.1.en.html) in Java.

This is a story of how I chased a memory leak in one of the Python 3.7 web services that I developed. Before I go down too deep in to the issue, I would like to set some context.

The service in question acts sort-of like a dumb proxy for the Couchbase buckets. Number of clients could be >= 100k or more. QPS can go up arbitrarily high (this is an expected behavior of the clients). The proxy serves as a protection to Couchbase tier minimizing number of connections to the Couchbase tier. Here, the number of connections to backend Couchbase tier is equal to the number of workers in the service.
* The workers are [Sanic Gunicorn workers](https://sanic.readthedocs.io/en/latest/sanic/deploying.html#running-via-gunicorn) that expose a GET only REST interface to the clients.
* [couchbase](https://docs.couchbase.com/sdk-api/couchbase-python-client-2.5.4/api/couchbase.html) library is used which internally uses libcouchbase2-core v2.10.4. During my initial benchmarking for qps (which I did 2 year back), Sanic+asyncio Couchbase bucket was winner being able to give us more than 35k QPS/node (with just 4 gunicorn workers). However, it must be noted that asyncio Couchbase bucket is an [experimental feature](https://docs.couchbase.com/python-sdk/2.5/async-programming.html#asyncio-python-3-5).
* Nginx is used in front of gunicorn to terminate SSL.

Anyways, during the year of the service going in to the production, we started getting alerts of memory utilization from the instances. I did some initial investigation and thought it would be worth to switch from `ujson` to `json` in Python's stdlib. This thought came after I saw [many posts][1] [2] claiming `ujson` using more memory than `json`.


However, after a month or two I deployed that change, I started noticing same symptoms (95% of RAM consumed by the workers) across all of the instances of the service. Memory consumption continues to grow over a month or more until OS starts to swap out pages. This time I wanted to dig deeper and fix the issue parmanently, if possible.


First, I wanted to see what's in the heap of the Gunicorn worker processes. I started collecting `gc` module data for object counts and tried to spot an increase. Initially I used [guppy](https://github.com/zhuyifei1999/guppy3) to inspect Python objects in the heap. But even after running it for few weeks, I was not able to conclusively say which kind of object was the cause, because there was no growth in object count. Here, as it can be seen, a Gunicorn worker with 37GB RSS, shows about 58MB usage by the Python objects as per guppy. So something was really missing.

```
By referrers: Partition of a set of 533203 objects. Total size = 59814702 bytes.
 Index  Count   %     Size   % Cumulative  % Referrers by Kind (class / dict of class)
     0 132180  25  9964875  17   9964875  17 types.CodeType
     1 111687  21  6654157  11  16619032  28 tuple
     2  52747  10  6075871  10  22694903  38 function
     3  28752   5  4044377   7  26739280  45 dict of type
     4   9230   2  3161449   5  29900729  50 function, tuple
     5  14893   3  3127448   5  33028177  55 type
     6  33545   6  2288877   4  35317054  59 dict (no owner)
     7   9902   2  2178976   4  37496030  63 dict of module
     8  36605   7  1721503   3  39217533  66 zipfile.ZipInfo
     9   1539   0  1322644   2  40540177  68 dict of module, tuple
```

Here's `/proc/<pid>/smap` of the same process:
```
00717000-016c7000 rw-p 00000000 00:00 0                                  [heap]
Size:              16064 kB
Rss:               15300 kB
Pss:               13093 kB
Shared_Clean:        240 kB
Shared_Dirty:       2116 kB
Private_Clean:         0 kB
Private_Dirty:     12944 kB
Referenced:        15276 kB
Anonymous:         15300 kB
AnonHugePages:         0 kB
Swap:                764 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Locked:                0 kB
ProtectionKey:         0
VmFlags: rd wr mr mw me ac sd 
016c7000-e488f000 rw-p 00000000 00:00 0                                  [heap]
Size:            3720992 kB
Rss:             3720864 kB
Pss:             3720864 kB
Shared_Clean:          0 kB
Shared_Dirty:          0 kB
Private_Clean:         0 kB
Private_Dirty:   3720864 kB
Referenced:      3657128 kB
Anonymous:       3720864 kB
AnonHugePages:         0 kB
Swap:                  0 kB
KernelPageSize:        4 kB
MMUPageSize:           4 kB
Locked:                0 kB
ProtectionKey:         0
VmFlags: rd wr mr mw me ac sd
```

I tried to take coredump of the two heaps using `gcore` but `gdb` crashed:

```
(gdb) dump memory heap1.xxx 0x016c7000 0xe488f000
../../gdb/exec.c:646: internal-error: failed internal consistency check
A problem internal to GDB has been detected,
further debugging may prove unreliable.
Quit this debugging session? (y or n) n

../../gdb/exec.c:646: internal-error: failed internal consistency check
A problem internal to GDB has been detected,
further debugging may prove unreliable.
Create a core file of GDB? (y or n) n
(gdb) dump memory heap.xxx 0x00717000 0x016c7000
```

So I waited for few more days for another instance to go crazy on memory. This time I could capture the two heaps. Interesting fact is that first heap never grows. The second heap is the one that grows and correlates to RSS utilization.

After inspecting the heap dump with `strings` I could see many Couchbase keys lying around, even when clients are not definitely querying for the keys. Immediately another service came in my mind that queries this service every 60s. It could be possible that Python is not getting enough time to GC these objects. But there was no indication of object increase from `guppy` output as well as the size of the values of the keys were only in few hundred MBs, not in GBs. So there was a big mismatch.

Another thing that poked me all the time was that why this second heap is invisible to Python `gc` module i.e why don't I get to see objects consuming GBs of memory per `gc` module.

My suspicion was now towards the C library and especially the experimental asyncio support in the Couchbase library. I started exploring other strings in the heap dump and came across:
```
2082202 Unknown:Unknown
2082193 couchbase.context_info
```
As can be seen, `Unknown:Unknown` appeared `2082202` times in the heap dump and `couchbase.context_info` appeared `2082193` times. When the worker was spawned the same counts were:
```
49791 couchbase.context_info
49788 Unknown:Unknown
```

Ah ha...there's a correlation!


`couchbase.context_info` is  defined [here](https://github.com/couchbase/couchbase-python-client/blob/dc48cf64aacf07ed5af6b0faee7fe82000533a65/src/ext.c#L2203). Regardless of how much deeper I wanted to dig, my priority was to get the service healthy. So I enabled [max_requests and max_requests_jitter](https://docs.gunicorn.org/en/stable/settings.html#max-requests) to Gunicorn config. Setting them to high numbers like 200k gives me no growth in memory consumption (as the workers are killed and spawned) with no errors in responses.

What I want to learn more:
  * Can this be solved by upgrading to `Couchbase` 3.x lib. This is not currently feasible task.
  * What other tools are available to actually poke around the heap of a Python process (as opposed to using `gc` module for object counts).

P.S: I tried to use `pythongc` and `pythonstats` from [BCC](https://github.com/iovisor/bcc/tree/master/tools) tools but they failed since we were on a 3.x Linux kernel. I would like to give these a try once we are in 5.x kernel.

[1]: https://github.com/explosion/srsly/issues/4
[2]: https://github.com/ultrajson/ultrajson/issues/333
