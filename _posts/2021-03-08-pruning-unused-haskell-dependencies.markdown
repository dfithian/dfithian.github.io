---
title: Pruning Unused Haskell Dependencies
---

I've never been able to find a good Haskell library for pruning unused dependencies. When I Google "remove unused
haskell dependencies", there are a few results but none exactly what I need:

* [fix-import](https://hackage.haskell.org/package/fix-imports) - import linter
* [packunused](https://hackage.haskell.org/package/packunused) - abandoned since 2014
* [weeder](https://hackage.haskell.org/package/weeder) - dead code analysis

So I wrote a tool to do it.

[https://hackage.haskell.org/package/prune-juice](https://hackage.haskell.org/package/prune-juice)

## Example

Say I add a dependency that isn't used in my project. In this case, we'll use `lens`, because that's one of my favorite
libraries.

```bash
$ prune-juice
Some unused base dependencies for package prune-juice
  lens
```

If I move that dependency to an executable, I'll also get an error just for that executable. I can also target specific
packages within my project, in case `stack.yaml` is very big.

```bash
$ prune-juice --package prune-juice
Some unused dependencies for executable prune-juice in package prune-juice
  lens
```

## How does it work?

`prune-juice` uses `hpack` to parse the project `package.yaml` files, and `ghc-pkg` to load the `exposed-modules` fields
of all the direct dependencies of the packages. It parses imports of each source file, compares against the exposed
modules, and errors if any dependency listed in `package.yaml` is never imported by a source file in that package.

## Performance

Edit: A friend helped out with the performance problems. It should be fast now!

~~Because it calls `ghc-pkg` once per dependency and stores the results in a large in-memory map, performance is quite
slow in large projects. However, for a small project it's pretty fast.~~

```bash
$ time prune-juice
prune-juice  0.35s user 0.07s system 95% cpu 0.442 total
```

## Compatibility

Update 2021-03-11: Works with Cabal now!

~~Currently, the tool is only compatible with Stack and Hpack, but there's no reason it can't work with Cabal, Nix, or
any other build tool in the future.~~
