---
title: Prune Juice 0.7 Released
---

For a description of `prune-juice`, see previous post on [Pruning unused Haskell dependencies]({% post_url 2021-03-08-pruning-unused-haskell-dependencies %}).

Since releasing [prune-juice](https://hackage.haskell.org/package/prune-juice), I have received a number of requests
asking to apply the unused dependencies directly to the cabal files. It ended up being a lot harder than I expected
to implement, and I'm proud to say that it's finally supported!

For those of you who just want the goods, you can run it with `prune-juice --apply`, and with `prune-juice --apply
--no-verify` to skip confirmation.

If you have any bug reports or feature requests, please create an issue [here](
https://github.com/dfithian/prune-juice/issues).

If you'd like to read about implementing the new features, there is an in-depth post
[here]({% post_url 2021-11-12-applying-file-changes-with-fix-and-gadts %}).
