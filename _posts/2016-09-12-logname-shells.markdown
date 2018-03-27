---
layout: post
status: publish
published: True
title: build yells, bash shells behind container doors
author:
  display_name: ffledgling
  login: ffledgling
  email: ffledgling@gmail.com
date: '2016-09-09 03:54:55 PM +0530'
comments: true
categories:
- Technical
tags:
- docker
- containers
- bash
- linux
---

Ran into a situation today where a colleague saw build's tests were failing when they were run on the Jenkins
slave farm that ran containers, but worked just fine on the Jenkins slave farm that used physical
machines.

Odd.

Check the strace. Nothing useful in there.
Check the test traceback. Seems to fail because a C++ String had a NULL byte in it.

Very Odd.

After poring over the source, my colleague found that the string being created was attempting to use
the environment variable, `LOGNAME` to create a path string. Now why the test tries to do this is
something that I won't go into, it ideally could've used something a lot more reliable such as
`getuid()`, avoiding the problem entirely.

Now to the interesting bit, why would `$LOGNAME` be defined on the physical machines, but not in the
containers?

The answer, is... it doesn't have anything to do with the container or the physical machine itself,
but how they are being used.

In the physical machine farm, the way jenkins slaves are run is that [master launches slave agent
via ssh](https://wiki.jenkins-ci.org/display/JENKINS/Distributed+builds#Distributedbuilds-Havemasterlaunchslaveagentviassh),
basically the master SSH's into the physical machine and fires up the slave agent with something
like `java -jar slave.jar`. The important thing to note is that the master SSH's into the physical
machine, we'll come back to why this is important.

In the container farm, the way jenkins slaves are run is that [slaves are launched
headlessly](https://wiki.jenkins-ci.org/display/JENKINS/Distributed+builds#Distributedbuilds-Launchslaveagentheadlessly),
what this means is that when the container is launched, the first executable that the container runs
is essentially the `slave.jar` itself or a wrapper script that calls `slave.jar` after some
bootstrapping. This `slave.jar` executable then dials home to actually connect to the Jenkins
master, which it knows how to find using certain pre-set environment variables. Note there was no
SSH here.

Now what's different between the first and the second case? Well the difference lies in the kind
of shell that the slave agent is spawned from. There are different modes a shell (bash in our
particular case), can run in. They are `interactive` vs. `non-interactive` and `login` vs
`non-login`. Any combination of the four possible combinations of [login,non-login] and
[interactive,non-interactive] is possible, although some combinations are a lot more common than
others.

One of the important aspects of these modes are:

- Login shells will read and load the `profile` family of files, such as `/etc/profile` and
  `~/.profile`.
- Interactive shells will read and load the `rc` family of files, such as `~/.bashrc`.

When you normally SSH into a machine for example, you will get an `interactive login` shell.
When you try to execute a command over SSH, with something like `ssh user@host '/usr/bin/binary'`,
you get a non-interactive non-login shell, *BUT* SSH sets some additional environment variables of
its own accord. When you start a binary with a script outside a shell (as the init process for
example), it seems to start it in a non-interactive non-login shell, which means `/etc/profile` is
not read.

Practically what this means is, the physical machines, with their `slave.jar` started via SSH, have
the `$LOGNAME` variable set, because SSH sets that always. This is inherited by the `slave.jar`.

For the container machines, this variable is not set. Because it is usually set in `/etc/profile`,
with something like `LOGNAME=$USER` where `$USER` is set by bash for login shells and since the
container runs the `slave.jar` in a non-interactive, non-login shell that is *not* spawned by SSH,
it doesn't source `/etc/profile` and doesn't set `$LOGNAME` on its own.

Not having LOGNAME set caused crypitc build failures.

Moral of the story: Don't rely on environment variables being set in compiled binaries unless you
explictly state that you need or use them. In this case, the right way would've been simply to use
the `getuid()` function instead.
