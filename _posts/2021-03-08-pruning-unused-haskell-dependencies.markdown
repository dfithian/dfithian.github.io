---
title: Pruning Unused Haskell Dependencies
---

I've never been able to find a good Haskell library for pruning unused dependencies. When I Google "remove unused
haskell dependencies", there are a few results but none exactly what I need:

* [fix-import](https://hackage.haskell.org/package/fix-imports) - import linter
* [packunused](https://hackage.haskell.org/package/packunused) - abandoned since 2014
* [weeder](https://hackage.haskell.org/package/weeder) - dead code analysis

So I wrote a tool to do it.

[https://hackage.haskell.org/package/prune-juice-0.3](https://hackage.haskell.org/package/prune-juice-0.3)

## How does it work?

`prune-juice` uses `hpack` to parse the project `package.yaml` files, and `ghc-pkg` to load the `exposed-modules` fields
of all the direct dependencies of the packages. It parses imports of each source file, compares against the exposed
modules, and errors if any dependency listed in `package.yaml` is never imported by a source file in that package.

## Performance

Because it calls `ghc-pkg` once per dependency and stores the results in a large in-memory map, performance is quite
slow in large projects. However, for a small project it's pretty fast.

```bash
prune-juice $ time prune-juice
prune-juice  2.42s user 0.86s system 85% cpu 3.832 total
```

## Compatibility

Currently, the tool is only compatible with Stack and Hpack, but there's no reason it can't work with Cabal, Nix, or
any other build tool in the future.
