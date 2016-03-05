---
layout: post
title:  "Dissecting Linux Performance Bottlenecks: CPU"
date:   "2016-02-21 19:56:49"
categories: "linux, performance"
---

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

###### To be continued ...
