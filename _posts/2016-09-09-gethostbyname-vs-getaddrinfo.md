---
layout: post
title:  "Gethostbyname vs. Getaddrinfo"
date:   2016-09-09 19:50:33
categories: "system programming, linux, dns"
---

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
