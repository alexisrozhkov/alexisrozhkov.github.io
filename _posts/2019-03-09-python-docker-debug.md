---
layout: post
image: images/zoltan-tasi-1384972.jpg  # Photo by Zoltan Tasi on Unsplash
title: "Debugging a running python script in a docker container"
excerpt: ""
tags: 
  - python
  - docker 
  - debugging
---

Recently I've experienced a seemingly random freeze in my python script, which was running inside of a docker container. It looked like a rare issue, so waiting for it to happen again was undesirable, and I wanted to extract as much information from this event as possible. Container with a hung script has been deployed to a remote server and wasn't ready for any type of convenient debugging, which made getting any insight into what had happened seem to be a tough task. Logs didn't help either - nothing suspicious was posted around the time of the event. The only useful bit of information I've been able to get was CPU utilization reported via `docker stats` for a corresponding container, which was ~100%. So far it looked like a deadlock, but it wasn't a particularly useful insight. 

After a bit of research, I've found few solutions which seemed interesting (registering a signal handler to dump thread backtraces or interrupt the program for a pdb session), but involved modifying and restarting the script, which I didn't consider at the time. There also was an option of using gdb for getting a backtrace of a running python script, which seemed to be a step in right direction, but unfortunately didn't provide enough information to narrow down the issue.

Finally, I've found something that seemed like an answer - a way to get a python backtrace for a running script. It was described [in DebuggingWithGdb section of python wiki](https://wiki.python.org/moin/DebuggingWithGdb) and involved installing gdb and python debugging extensions. The only gotcha in my case was - the script was running inside of a container. At first, it didn't seem like a huge deal, so I've attached to a container with a hung script, installed the needed tools and tried to attach to the faulty process:
```
alexey@laptop:~$ docker exec -it CONTAINER_ID /bin/bash
root@CONTAINER_ID:/usr/src/app# apt-get install gdb python3.6-dbg
...
root@CONTAINER_ID:/usr/src/app# gdb python PROCESS_ID
...
Reading symbols from python...Reading symbols from /usr/lib/debug/.build-id/72/cdaf061ffbc62f05117e8c5825ac1e700838b1.debug...done.
done.
Attaching to program: /usr/bin/python, process PROCESS_ID
Could not attach to process.  If your uid matches the uid of the target
process, check the setting of /proc/sys/kernel/yama/ptrace_scope, or try
again as the root user.  For more details, see /etc/sysctl.d/10-ptrace.conf
ptrace: Operation not permitted.
/usr/src/app/PROCESS_ID: No such file or directory.
```
Unfortunately, I wasn't able to attach to the process from within the container without changing something. There was another option - performing the same thing, but from the host machine. Let's see if that works:
```
alexey@laptop:~$ sudo apt install gdb python3.6-dbg
...
alexey@laptop:~$ sudo gdb python HOST_PROCESS_ID
...
Reading symbols from python...(no debugging symbols found)...done.
Attaching to program: /usr/bin/python, process HOST_PROCESS_ID
[New LWP 9729]
[New LWP 9730]
[New LWP 9731]
[New LWP 9738]
[New LWP 9739]
[New LWP 9740]
[New LWP 9752]
[New LWP 9807]
[New LWP 9808]
[New LWP 9809]
[New LWP 9810]
Cannot access memory at address 0x24748b3024446350
Cannot access memory at address 0x24748b3024446348
Cannot access memory at address 0x24748b3024446350
Cannot access memory at address 0x24748b3024446350
Cannot access memory at address 0x24748b3024446348

warning: Target and debugger are in different PID namespaces; thread lists and other data are likely unreliable.  Connect to gdbserver inside the container.
0x00007f59d0fc4827 in ?? ()
(gdb) py-bt
Undefined command: "py-bt".  Try "help".
```
Still no luck! But it might be actually due to a difference of python interpreter versions on the host system and inside the container. I didn't actually verify this, but the fact that debugging symbols were found in the first case and not in the second seemed to agree with this hypothesis. Problem is - there's a very particular version of python inside of the container, and I don't really want to have it alongside with other ones on the host machine. Instead, I decided to try something else:
- run a new container on the same machine from the same image and jump directly into the bash session
- when running the container, make sure it:
    - will see the processes on the host system (including the one we want to debug in another container) (`--pid=host`)
    - runs in a privileged mode (to allow fiddling with the process) (`--privileged`)

```
alexey@laptop:~$ docker run -it --rm --privileged --pid=host IMAGE_NAME /bin/bash
root@DEBUGGER_CONTAINER_ID:/usr/src/app# apt install gdb python3.6-dbg
...
root@DEBUGGER_CONTAINER_ID:/usr/src/app# gdb python HOST_PROCESS_ID
...
Reading symbols from python...Reading symbols from /usr/lib/debug/.build-id/72/cdaf061ffbc62f05117e8c5825ac1e700838b1.debug...done.
done.
Attaching to program: /usr/bin/python, process HOST_PROCESS_ID
[New LWP 9729]
[New LWP 9730]
[New LWP 9731]
[New LWP 9738]
[New LWP 9739]
[New LWP 9740]
[New LWP 9752]
[New LWP 9807]
[New LWP 9808]
[New LWP 9809]
[New LWP 9810]
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

warning: Target and debugger are in different PID namespaces; thread lists and other data are likely unreliable
0x00007f59d0fc4827 in futex_abstimed_wait_cancelable (private=0, abstime=0x0, 
    expected=0, futex_word=0x7f591c000c10)
    at ../sysdeps/unix/sysv/linux/futex-internal.h:205
205    ../sysdeps/unix/sysv/linux/futex-internal.h: No such file or directory.
(gdb) py-bt
Traceback (most recent call first):
  ...
```
Nice! Now the py-* commands are recognized, and python backtrace is shown properly. We can proceed to showing backtraces of all the threads (shorthand `t a a COMMAND` comes in handy) and intensely staring at the output.

In my case, it resulted in spotting fork-related lines, which was an immediate red flag, since the script was multi-threaded, and mixing these 2 is a big no-no (at least in python world). What made it interesting was the fact that the script maintainers were not aware of this call, because it was buried deep in a 3rd-party python package.

At this point it was still speculation, so I've run the container on a different machine, and tried to reproduce the hang by increasing the number of concurrent threads and repeatedly triggering the code path that performs forking. It turned out to be reasonably easy to cause the hang, so then I proceeded to fix it - getting rid of the fork call (and a "faulty" package altogether). After performing these changes I haven't been able to trigger the hang using the same approach as before - I guess that's a happy ending :)

PS. I found explanations in this article very helpful: [Stabbing yourself with a fork() in a multiprocessing.Pool full of sharks](https://codewithoutrules.com/2018/09/04/python-multiprocessing/). Unfortunately, in my case setting the start method to `spawn` didn't help, and instead of looking deeper into why this happened, I went with a simpler approach of getting rid of the multiprocessing call at all.


