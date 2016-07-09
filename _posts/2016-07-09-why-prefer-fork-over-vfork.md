---
layout: post
title:  "When to prefer fork() over vfork()"
date:   2016-07-09 15:34:20
categories: "linux, C, system call"
---

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
