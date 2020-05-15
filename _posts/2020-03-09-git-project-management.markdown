---
title: Git Project Management
---

`hit` is a new tool for managing multiple git repositories as a project
[https://github.com/dfithian/hit](https://github.com/dfithian/hit). Drop a config in `~/.hitconfig` and start treating
groups of repositories like a single project - get info about status, diffs, create branches, rebase... you name it!

It works by using [Turtle](http://hackage.haskell.org/package/turtle) and `cd`s to each directory in the config to
execute the command you provide. **Every command with `git` will also work with `hit`.**

As always, I'd love feedback from anyone who uses it! The easiest place to find me is on fpchat (@dfithian).

Here's the link again: [https://github.com/dfithian/hit](https://github.com/dfithian/hit).
