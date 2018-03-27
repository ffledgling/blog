---
layout: post
status: publish
published: True
title: Building irssi with XMPP
author:
  display_name: ffledgling
  login: ffledgling
  email: ffledgling@gmail.com
date: '2016-09-09 03:54:55 PM +0530'
comments: true
categories:
- Technical
tags:
- irssi
- xmpp
---

If you're coming from the world of IRC, you're probably familiar with, or at least have heard of
irssi. It's a command line IRC client. You've also most likely used xmpp, maybe you just don't know
it. If you've used Facebook's messenger, Yahoo's Messenger or Google Talk/Hangouts, you've used
xmpp, since that's the underlying protocol they all use to some degree.

I've been a big fan of irssi and there just haven't been any command line xmpp clients that have
come close to the irssi experience. The best I got was finch, which is a TUI (terminal), but it's
fairly involved and doesn't really give you all the speed that comes with the CLI.

So I decided to get xmpp running with irssi. The environment that I work in doesn't give me root, so
things were compiled by hand, you can probably do better if you have a package manager.

Prereq packages (that I know of):
- perl
- gcc
- glib2
- glib2-devel
- ncurses-devel ncurses ncurses-lib
- make
- autoconf
- automake
- gtk-doc (optional)
- libtool (error messages when you skip this are a little confusing)
- gnutls (optional, but not really, you probably want to connect to your server over SSL anyway)


Most sane desktop machines will have these installed by default, if you don't, well you gotta build
them by hand as well :)

Download irssi source then do:

Configure:
./configure --prefix=$HOME/manuallyinstalled/ --with-perl=yes --with-proxy --with-socks

Now you want to build the [xmpp extension](https://github.com/cdidier/irssi-xmpp), but that seems to
need loudmouth. Let's build that then.

wget https://github.com/mcabber/loudmouth/archive/1.5.3.tar.gz

Then we want to build it.
./autogen.sh # -n if you want to skip the docs
./configure --prefix=$HOME/manuallyinstalled
make
make install

Now we try building irssi-xmpp, but first we gotta download it.
wget https://github.com/cdidier/irssi-xmpp/archive/v0.53.tar.gz

Then we build it.
To build it, we need to modify the `config.mk` file, to set `PREFIX = $HOME/manuallyinstalled`

Then we build it like so:
```
PKG_CONFIG_PATH=$HOME/manuallyinstalled/lib/pkgconfig/ make
make install
```

You need to give it the PKG_CONFIG_PATH because otherwise it can't find the `.pc` file for
loudmouth, which the make step needs to figure out flags and includes for loudmouth.

Then you need to run it.
Running it is fairly simple, but not as simple as just typing `./irssi` and being on your way.

If you just do `./irssi` and try to load the XMPP module like so:
```
/load xmpp
```

You'll get an error that looks like this:

```
19:28 -!- Irssi: Error loading module xmpp/core: libloudmouth-1.so.0: cannot open shared object file: No such file or directory
19:28 -!- Irssi: Error loading module xmpp/fe: /home/fedora/manuallyinstalled/lib/irssi/modules/libfe_xmpp.so: undefined symbol: xmpp_subscription
19:28 -!- Irssi: Error loading module xmpp/core: libloudmouth-1.so.0: cannot open shared object file: No such file or directory
```

You need to tell it where to find the loudmouth library you installed. To do that, run it like so:
```
LD_LIBRARY_PATH=$HOME/manuallyinstalled/lib/ irssi
```

Then do:
```
/load xmpp
```

and you'll see something like this:
```
19:32 -!- Irssi: Loaded module xmpp/core
19:32 -!- Irssi: Loaded module xmpp/text
19:32 -!- Irssi: Loaded module xmpp/fe
```

Then connect to your XMPP server of choice with:
```
XMPPCONNECT [-ssl] [-host <host>] [-port <port>] <jid>[/<resource>] <password>
```

After which you can do `/roster` to see if you can load the roster from the server. If every thing
is working correctly, it'll print out the list of users.

And that's it! Easy as pie.
