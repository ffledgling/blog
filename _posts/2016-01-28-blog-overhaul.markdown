---
layout: post
status: publish
published: True
title: Blog Overhaul
author:
  display_name: ffledgling
  login: ffledgling
  email: ffledgling@gmail.com
date: '2016-01-28 03:28:51 +0530'
categories:
- Technical
- taskcluster
tags:
- Jekyll
- WordPress
- Migrations
---

I'll keep this one short and sweet.

Over the past couple of months I've constantly been collecting material I'd like to blog about. New
ideas, learnings, processes, experiences and so forth.  Unfortunately I've been putting off doing so
because of the massive overhead that writing and publishing a post on WordPress involves.

I tend to do all/most of my writing in my text editor in markdown. To publish a post to WordPress,
I'd usually convert the post to some format acceptable to WordPress via `pandoc` and then spend the
next two-ish hours getting the formatting right and fixing the errors.

I realized this wasn't really a good use of my time **and** that I didn't really need WordPress in
the first place. As a result, I decided to move to Jekyll, which is a static website generator that
understands markdown natively.

The upside is that I no longer have to worry about upgrading/patching WordPress, running a MySQL
database, a PHP-FPM pool in addition to the Apache HTTP Server while trying to keep the memory
limited to the ~400MB I have available on my server.

The downside is that I lose a few things like, Atom/XSS feeds, Comments, automatic category and tag
pages. Most of these I think I can work around using a plugin or by putting in some more effort
tweaking the Liquid Templates, so hopefully in time, it shouldn't be too shabby.

I've put it out as is right now, broken, borked, incomplete; It's complete enough to let me publish
my writing though and that's a decent start.
