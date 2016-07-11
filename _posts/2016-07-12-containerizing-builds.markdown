---
layout: post
status: publish
published: True
title: Containerizing Builds
author:
  display_name: ffledgling
  login: ffledgling
  email: ffledgling@gmail.com
date: '2016-07-12 03:18:11 +0530'
comments: true
categories:
- Technical
tags:
- Kubernetes
- Docker
- Linux
---

I've recently had the opportunity to play around with Kubernetes, Docker and Jenkins.
As part of the effort, I wanted to containerize my Jenkins builds. You'd think this was a fairly
straight forward thing to do. Usually it'd go something like this:

- Figure out your environment requirements.
- Write a Dockerfile, create a container.
- Run build in container.
- Profit.

I'd done steps #2 through #4 before. How hard could #1 be?

Very hard apparently. The build environment I was working with was artisan, that is it was hand
crafted by many people over time. This meant there was no documentation about what files, configs,
packages, binaries and external services the build needed. I tried to figure out what the
requirements and dependencies are/were by asking people, but the knowledge I pieced together
appeared to be incomplete at best and would've been possibly incorrect in the worst case.

I then tried a different approach. I started out with as much of the environment as I could possibly
need to get a build running, and I'd try to subsequently slim it down. This was a good approach
initially, because it helped get a Proof-of-Concept containerized build out the door quickly.

It wasn't ideal though. A proper containerization process will force you do explicitly document your
dependencies. The packages you need. The binaries. Exact configs. Ports, Network access and so on.
This is good, it forces you to be very explicit about your environment. Mounting 30+ Gigabytes of
folder from the host file system into the container was less than ideal long term and also
circumvented the need to document things.

A better approach was needed.

I figured the best way to figure out the correct dependencies would be to ask the source of truth,
the build itself.  The best way to do this is to use `strace`. You can treat the build itself as a
black-box and simply monitor the system calls the build makes. By filtering the system calls I was
able to identify the different kinds of dependencies.

Here's an example of how this would go:

```
# Jenkins creates a shell script with all commands to run in /tmp/ 
# as part of build process every time it runs a build. You can see
# in the logs for your build. /tmp/build_script.sh is a copy of one
# such script from a build.

# trace the build, including all child processes
$ strace -f -o /tmp/strace.out /tmp/build_script.sh

# Figure out what files the build reads, look at the open() syscall.
$ grep 'open' /tmp/strace.out

# Machine/script friendly output, list of files:
$ cut -d' ' -f2- /tmp/strace.out | grep '^open(' | sed 's/open("\(.*\)".*/\1/'

# Figure binaries used directly, look at the execve syscall().
# The entire exec* family of functions, call execve underneath.
$ grep 'execve' /tmp/strace.out

# Figure out what machines it contacts over the network
$ grep htons /tmp/strace.out
```

You can tailor the filter on the strace output to your exact needs. Different system calls will tell
you different things. Want to know if the program checks for the existence of a file, but doesn't
read it? Do the difference operation on the sets of files you get from `stat()` and `read()`. Want
to know which domain names it tries to access? Filter for `getaddrinfo()`. And so on.


I've been using Linux for a fair few years now, but I'd never used `strace` except for the few times I
read a few comments about it. It became a mainstay of my debugging toolkit after [this PyCon talk by
Julia Evans](https://youtu.be/5v6o-VsLAew). It's worth a watch.


Until next time.

