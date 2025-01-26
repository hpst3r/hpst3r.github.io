---
title: "Snippet: branch and merge a detached head with main"
date: 2025-01-25T10:12:59-00:00
draft: false
---

I keep doing this. But I keep doing it infrequently enough that I need to look up the solution every darn time. So I'm putting it here.

I think VSCode's source control integration does this somehow when I make changes in a submodule and main project, then commit the main project changes first, though I have no idea why this is the case.

The following is probably the easiest way to fix up a detached head and get back to work:

```txt
$ git status
HEAD detached from 590729e
nothing to commit, working tree clean

# create the 'detached' branch
$ git branch detached

# update the working tree to match main/master branch
$ git checkout master

# merge the detached commit with the checked out branch
$ git merge detached
```

You could also stash the work you've done and commit it on top of the master branch, but I overwrote a couple hours of changes while attempting to do that and decided to figure something else out instead.