---
layout: post
status: publish
published: True
title: "Writing a Custom Importer in Python 🐍"
author:
  display_name: ffledgling
  login: ffledgling
  email: ffledgling@gmail.com
date: '2018-09-12 03:14:17 AM +0530'
comments: true
categories:
- Technical
tags:
- python
- python-modules
- python-imports
- python-internals
---

While looking into the `import` statement in Python (2!) and how the internal Python import machinery works, I stumbled across mention of interesting functionality buried in `help('import')`. Here's what I found:

> If the module is not found in the cache, then "sys.meta_path" is searched (the specification for "sys.meta_path" can be found in **PEP 302**). The object is a list of *finder* objects which are queried in order as to whether they know how to load the module by calling their "find_module()" method with the name of the module. [...] If a finder can find the module it returns a *loader* (discussed later) or returns "None".

This probably doesn't make any sense at the moment, I had to go back and re-read everything slowly at least 3 times before I could figure out what was going. So I'm going to do you folks a solid and provide some context to what's going on.

First, a quick refresher (most python developers can probably skip this).
Here's how imports work in python (syntactically):

```python
# Import the module
import spam

# From the module spam, import bar into current namespace
from spam import bar

# The same thing really, just rename bar as, well, you get the idea
from spam import bar as b_b_b_barz

# From the module spam, import everything specified in __all__ list
# into the current namespace. If __all__ is not defined, import
# everything not starting with `_` instead.
from spam import *
```

Here's the difference between a module and a package in Python, as described officially (`help('import')`):

> To help organize modules and provide a hierarchy in naming, Python has a concept of packages. A package can contain other packages and modules while modules cannot contain other modules or packages. From a file system perspective, packages are directories and modules are files.

We'll often end up using module to mean both *package* and *module* as a blanket term, but this distinction is important to keep in mind because file style "modules" need to be handled differently from directory style "modules".

Now, what was I looking for? Ah, yes, I was trying to figure out how Python finds and then subsequently loads a module or a package into my current namespace when I type `import spam` . What really happens when I `import` something into my REPL[1] for example?

The answer to the question it turns out, is a lot of things. The overview is:

1. Python checks if the module is already imported, and if so, just creates a new reference to the existing module object or does nothing.
2. If it doesn't, it begins to search for it. The typical search involves looking through the directories in `sys.path` and checking for specific filenames that are derived from the name of the module you're trying to import.
3. Once found, python will load it, create a module object and then create a reference to it, usually with the module name itself or the name we asked for (i.e `spam` if we do `import spam` or `foo` if we do `import spam as foo`)

Most Python programmers who've worked with modules in any capacity understand this flow to some degree. Nothing surprising going on there.

Now back to where we started. See where the `help` says *"If the module is not found in the cache, then [...]".* Python has a nifty trick up its sleeve between **Step 1** and **Step 2**. The trick is, you can create your *own* custom Python module finder/loader. What would something like that do? Well, it'd allow you to write code to find and load a module the way *you* wanted to, *from* where you wanted to. Why would you want to do that, you ask? Glad you did, let's check out what the **PEP 302** mentioned in the original quote has to say about it:

> The only way to customize the import mechanism is currently to override the built-in **import** function. However, overriding **import** has many problems.
> *[...]*
> Extending the import mechanism is needed when you want to load modules that are stored in a non-standard way. Examples include modules that are bundled together in an archive; byte code that is not stored in a pyc formatted file; modules that are loaded from a database over a network.

Huh. Load modules **from a database over a network**. *Interesting.* Might be fun writing one that can load modules stored in say... S3 buckets or Github then? Why yes, yes it might be.

Before we dive headfirst into the deep-end however, I figured I'd start with a simple custom finder that I'm calling `tmpfinder`. What it does is pretty simple, it looks for and loads modules from `/tmp/modules` , a path that's not in the default list of paths (`sys.path`) that Python searches for modules in.

Before someone points it out, yes, this is pretty straightforward do even *without* writing an entire custom finder and loader. You can do it a couple of different ways, the simplest among them being:

- Add `/tmp/modules/` to `$PYTHONPATH`, then let python automatically find modules and use it's built-in (default) file importer to import the module correctly.
- Alternatively, one can use `.pth` files (called site configuration files) to use the [site-specific configuration hooks](https://docs.python.org/2.7/library/site.html), which then automatically adds any paths listed in `*.pth` files in the `site-packages` directory to the module search path and then the builtin importer picks up as usual.

Writing the `tmpfinder` module is a good place to start because:

1. We have sane behavior to compare it with.
2. It's simple enough functionality wise to let me focus on learning [The Importer Protocol](https://www.python.org/dev/peps/pep-0302/#id27) [2], the biggest piece of writing a custom importer. No worrying about the details of how the S3 protocol or Github API is implemented for now.
3. It's low effort, if I failed to grok the documentation or PEP or ran into some inscrutable bug or Pythonism, I wouldn't have wasted time writing a couple of hundred lines of code.

Now, on to the good part, the importer protocol itself and the code (I hear you impatient ones sighing heavily and muttering *finally* in the back, don't think I don't hear you).

The importer protocol itself is very well documented in [PEP-302](https://www.python.org/dev/peps/pep-0302/), but I will cover the gist of the protocol, without attempting to be comprehensive and worrying about the nitty-gritty details. Think of it as an ELI5... if 5yos were python developers... interested in the internals of how importing in python works... yeah.

The importer protocol, in essence consists of two interfaces: The **finder** and the **loader**.

As the name implies, the finder object is responsible for checking if it can find a given module. If it finds the module, it returns a loader object for that particular model, if not, it returns `None`.

The loader, as the you might've guessed, is responsible for actually loading the module and returning the newly created `module` object to the caller. It is also responsible ensuring a few other things happen before and during the loading phase, such as adding the module to `sys.modules` and ensuring attributes like `.__file__` exist on the module object and are instantiated correctly.

The finder object needs to be added the `sys.meta_path` list. Once that's done, the import system automatically invokes a `.find_module` on the finder object added to the meta path and provides it the name of the module to be imported, as well as the package path it's a part of (so relative imports work, if you so wish). If the finder returns a loader object (instead of None), it's used to actually load the object, by invoking the `.load_module` method.

Let's look at what the `tmpfinder` actually looks like[3], I recommend reading through it and paying attention to the inline comments, they explain the Importer Protocol in slightly greater detail inline.
```python
import sys
import types # Used to instantiate a Module object, unavailable as a builtin
import os.path

# os.path.join is used frequently, so we shorten our invocation
from os.path import join as ospj

class TmpFinder(object):
    """ Class to find and load modules from `/tmp/modules/` """

    def __init__(self):
        self.greeting = 'Hello from TmpFinder'
        self.tmp_prefix = ospj('/tmp', 'modules')

    def find_module(self, fullname, path=None):
        # Adding print statements, so that (1) we know everytime find_module is
        # invoked during the import process and (2) we see what the name and
        # path of the module passed to us look like.
        print self.greeting + ':find_module'
        print (fullname, path)

        # NOTE: For simplicity and brevity, we assume the module we're being
        # given is a top-level file module, not a directory based package or
        # sub-package. We also ignore `path`, doing so will break subpackage
        # imports and relative imports. Adding this functionality is easy
        # enough, but it detracts from the example, so we leave it out.
        location = ospj(self.tmp_prefix, *(fullname.split('.')))
        print 'Looking for module in: {}'.format(location)

        if os.path.exists(location):
            # We're returning the loader object here, which in this case, just
            # happens to be the same as the finder object
            return self

        # Default return for a function is None of course, so we do nothing
        # special when we don't find the module

    def load_module(self, fullname):
        # Helpful print statement tells us when loader is used
        print self.greeting + 'load_module'

        # If the module already exists in `sys.modules` we *must* use that
        # module, it's a mandatory part of the importer protcol
        if fullname in sys.modules:
            # Do nothing, just return None. This likely breaks the idempotency
            # of import statements, but again, in the interest of being brief,
            # we skip this part.
            return

        location = ospj(self.tmp_prefix, *(fullname.split('.')))
        print location

        try:
            # The importer protocol requires the loader create a new module
            # object, set certain attributes on it, then add it to
            # `sys.modules` before executing the code inside the module (which
            # is when the "module" actually gets code inside it)

            m = types.ModuleType(fullname, 'This is the doc string for the module')
            m.__file__ = '<tmp {}>'.format(location)
            m.__name__ = fullname
            m.__loader__ = self
            sys.modules[fullname] = m

            # Attempt to open the file, and exec the code therein within the
            # newly created module's namespace
            with open(location, 'r') as f:
                exec f in sys.modules[fullname].__dict__

            # Return our newly create module
            return m

        except Exception as e:
            # If it fails, we need to reset sys.modules to it's old state. This
            # is good practice in general, but also a mandatory part of the
            # spec, likely to keep the import statement idempotent and free of
            # side-effects across imports.

            # Delete the entry we might've created; use LBYL to avoid nested
            # exception handling
            if sys.modules.get(fullname):
                del sys.modules[fullname]
            raise e
```

Now, how do we actually try it out? Simple enough, first let's create a python module to import, here's what it looks like:

```python
In [7]: %cat /tmp/modules/spam
#!/usr/bin/env python

def foo():
    print "Yay, we are spam!"
```

Just a run-of-the-mill `foo()` function inside the `spam` module.

Fire up your interpreter in the same directory as `tmpfinder.py` and try importing spam now:

```python
In [1]: import spam
-------------------------------------------------
ImportError     Traceback (most recent call last)
<ipython-input-1-bdb680daeb9f> in <module>()
----> 1 import spam

ImportError: No module named spam
```

Makes sense, we've not really done anything special to tell Python where the spam module is. Let's do that using what we've learnt from PEP-302:

```python
In [2]: import sys

In [3]: import tmpfinder

In [4]: sys.meta_path.append(tmpfinder.TmpFinder())
```
```
In [5]: import spam
Hello from TmpFinder:find_module
('spam', None)
Looking for module in: /tmp/modules/spam
Hello from TmpFinderload_module
/tmp/modules/spam

In [6]: spam.foo()
Yay, we are spam!
```

It works! But what did we do here? We create a `TmpFinder` object, which knows how to look for and load files in `/tmp/modules/` , we then add this to the list of meta path objects.

The objects are iterated over and asked if they know how to find and load a module every time an `import` statement for said module is issued. See how our debug print statements kick in and leave a trail of breadcrumbs for us to follow? `find_module` is invoked, followed by `load_module` just as specified in the importer protocol. None of this is surprising or special of course, if you've read PEP-302 or followed the earlier discussion, Python is just doing what it says on the tin. The point of this exercise is just to make the protocol more tangible. For completeness sake, let's see what happens when we *don't* find a module:

```
In [7]: import notspam
Hello from TmpFinder:find_module
('notspam', None)
Looking for module in: /tmp/modules/notspam
-------------------------------------------------
ImportError     Traceback (most recent call last)
<ipython-input-7-3cb35e0ee4ca> in <module>()
----> 1 import notspam

ImportError: No module named notspam
```

The `find_module` on your finder object is still called, but since it isn't found, Python moves on and looks for it in more standard locations and formats, such as files and directories under `sys.path` . When it doesn't find a module named `notspam`, it raises the `ImportError` exception appropriately.

Annndddd that's how you write a simple custom importer for Python. Since this blog post is long enough as-is. I'll likely add the S3 bucket importer or Github importer as a second part to this post.

## References:

[1] The REPL, short for Read-Eval-Print-Loop is another name for Python's Interpreter in this context, the idea of a REPL is a general one however. Any interpreted language that provides a "Shell" that "Read"s input from the user, "Eval"uates it and "Print"s the output to standard output, over and over again ("Loop"ing), can be called a REPL.

[2] The Importer Protocol is formally defined in PEP-302, it dictates the interface as well as the expected side-effects of such a custom importer. It really is the crux of what we're trying to do here.

[3] I feel it important to point out that PEP-302 actually has a full fledged template for how to write a finder/loader/importer object under the ["Specification part 1: The Importer Protocol"](https://www.python.org/dev/peps/pep-0302/#specification-part-1-the-importer-protocol) section. Something to look into if you ever end up needing to write a production grade importer.
