---
layout: post
title:  "Dissecting `tail -f`: linux inotify"
date:   2016-02-17 23:06:25
categories: "linux, C, system call"
---

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
