---
layout: post
status: publish
published: True
title: Transplanting Subdirectories with Git
author:
  display_name: ffledgling
  login: ffledgling
  email: ffledgling@gmail.com
date: '2016-07-28 02:30:11 +0530'
comments: true
categories:
- Technical
tags:
- Git
- Linux
---

As someone who works in Developer Operations, I'm somtimes presented with interesting requests.  A
couple of months back I was asked to *"help move a directory from one git repository to another"*. It
was an odd one, but should be simple enough.\* 

The request explained in more detail was as follows:

`A` is a git repository that has subdirectories. The subdirectory of interest is called `P`.

```
A
├── 1
├── 2
├── 3
└── P
    ├── X
    └── Y
```

The subdirectory `P` needs to be moved into a new repository called `B`. This is what `B` is supposed to look like:

Before:

```
B
├── 4
├── 5
└── 6
```

After:

```
B
├── 4
├── 5
├── 6
└── P
    ├── X
    └── Y

```

To do this, I used the following git commands:

```
# Get into repository A.
cd A

# Generate a list of all patches that touched any files in the subdirectory P
git log --pretty=email --patch-with-stat --reverse --full-index --binary P/ > transplant.patches

# Go to Repository B
cd ../B

# Apply those patches
git am ../A/transplant.patches
```

Works like a charm. All the history is there, `git blame` works correctly on files. All shiny. 


***Except*** when a new request comes in a couple of months later to do another transplant. This time
the request is to transplant something from repository `B` into a third repository `C`.

Easy peasy, been there, done that. *Cough* right *cough*.

This time around I need to move a subdirectory `Q/X` from `B` into `C`. I fire up my terminal and
perform the operations same as before.

Current state of B:

```
B
├── 4
├── 5
├── 6
└── Q
    ├── X
    └── Y
```

C, before the transplant:

```
C
```

... *yep*, it's an empty repository with an init commit, it doesn't change much from the transplant perspective though.

After:

```
C
└── Q
    └── X
```

I call in the developer who'd requested the transplant and ask him to check if the repository looks
like he'd expect. He fires up a terminal, runs a `git blame`. Says the blame doesn't look right. I
ask him why. 

*"It doesn't have the history from before the transplant".  
"What transplant?".  
"The one you did last time...".  
"Huh? The one last time? That wasn't this path... was it?".  
"We moved it...".*

So for clarity sake, I'm going to show the states again.

After my first (original) transplant:

```
B
├── 4
├── 5
├── 6
└── P
    ├── X
    └── Y
```

After developers' move:

```
B
├── 4
├── 5
├── 6
└── Q
    ├── X
    └── Y
```

As you can see the `P` subdirectory was renamed to `Q`.

Now, the problem with moving a directory in git is that you can no longer do the nifty trick
where you extract all the commits that affect a particular subdirectory correctly. Once moved, all
commits after the move show up against the subdirectory path, but not the ones before. `git blame`
still works because git is able to trace changes across moves for single files. You can run `git log
--follow path/to/file` to see for yourself. This doesn't work for subdirectories.

So the relevant history in B looks something like this:

```
# <abrev SHA> <commit message>
56789 5th Commit; touches Q/X
45678 4th Commit; Moves P to Q
34567 3rd Commit; touches P/Y
23456 2nd Commit; touches P/X
12345 1st Commit; touches P
```

Ideally when transplanting `Q/X` into `C` the history should look something like this:

```
# <abrev SHA> <commit message>
56789 5th Commit; touches Q/X
45678 4th Commit; Moves P to Q
23456 2nd Commit; touches P/X
12345 1st Commit; touches P
```

Notice that the 3rd commit is gone because it created `P/Y` and we're not transplanting `P/Y`, just
`P/X`.

But when you run the patch generation command earlier in the post with `Q/X` as the specified path,
it'll simply give you patches corresponding to two commits.

```
# <abrev SHA> <commit message>
56789 1st Commit; touches Q/X
45678 4th Commit; Moves P to Q
```

An even more interesting observation is that the commit with SHA `45678` is the commit that
performed the original move. A move commit in git is interesting, because git doesn't actually treat
move as a first class operation, instead it treats a move as a delete operation followed by a
create/add operation. When generating a patch series with `git log <options> path/to/subdir/` the
"move" commit won't show information about the deletion of the files at the old path (`P/X` in our
case), simply information about creation of files at the new path (`Q/X` in our case). This is because
you asked git to give you the patches corresponding to the commits that that modified the new path.
The deletion of the old path doesn't modify the new path in anyway, the "creation part" of the
"move" commit is technically the commit that created the path for the first time.

The final solution to the problem was based off of this understanding. What I essentially needed
was:

- The history of the old path, before the move, i.e `P/X`
- The history of the new path, after the move, i.e `Q/X`
- The move commit, that:
    - Had the diffs deleting files at the old path, i.e `P/X`
    - Had diffs creating the files at the new path, i/e `Q/X`

To generate these, the commands I ran are below, with explanatory comments inline.

```
# Checkout the master, just to be sure, we're on the right branch.
git checkout master

# Look at the git log for the new path, figure out where the move happened.
# It'll be the last commit in the log.
git log Q/X

# Let's say the commit was 85ec27
#Look at the commit to confirm it has the additions and deletions
git show --summary 85ec27

# Checkout to the commit just before the move commit
git checkout 85ec27^1

# Generate the patchset for the old path
git log --pretty=email --patch-with-stat --reverse --full-index --binary P/X > transplant.beforemove.patches

# Checkout master
git checkout master

# Generate the patchset for the new path
git log --pretty=email --patch-with-stat --reverse --full-index --binary Q/X > transplant.aftermove.patches


# Generate the move patchset
git checkout 85ec27

# Generate the patch
git format-patch HEAD~1

# Open up the format-patch, edit it to remove all the diffs
# corresponding to files that it touches that are *not* in P/X or Q/X
vim 0001-my-move-commit.patch

# We now have all three of our required patches, let's apply them in order.
# Go into the new repo
cd C/

# Apply the beforemove.patches
git am ../B/transplant.beforemove.patches

# Check that the files in the old path are created
ls
# Should show P/X
tree

# Apply the bridge patch
# This part is tricky, if you modified the bridge patch wrong, it'll fail to apply.
git am ../B/0001-my-move-commit.patch

# Check to see if the files from the old path have been moved to the new path.
ls 
# Should show Q/X
tree

# Apply the aftermove.patches
git am ../B/transplant.aftermove.modified.patches

# Check to see the files are all there
ls
# Should show Q/X
tree

# Run a git blame, check the history to see everything is in order
git blame Q/X/somefile
git log Q/X/someotherfile

```

That's basically the gist of it. It's a fairly niche case and I didn't find a better way to do it
online or on stackoverflow or even when I asked on IRC, so I figured I'd document it online, for my
sake and possibly some other poor soul's.


<br />
<hr />


*\* Doing it is simple, doing it **right** on the other hand...  
I realized during the second transplant that the correct way is to probably do a 
`git filter-branch --subdirectory-filter` followed by a **git subtree merge***.


References:

1. <http://stackoverflow.com/q/1365541>
2. <https://www.kernel.org/pub/software/scm/git/docs/git-log.html>
3. <https://git-scm.com/book/en/v1/Git-Tools-Subtree-Merging>
