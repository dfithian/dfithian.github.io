---
title: Haskell Executable Sizes
---

This post is an experiment with reducing Haskell executable sizes. Inspired by
[https://dixonary.co.uk/blog/haskell/small](https://dixonary.co.uk/blog/haskell/small).

Reading the above post motivated me to investigate its techniques on a real world example. At work, there are some
complex services with many transitive dependencies. Since Haskell executables are statically compiled by default, all
the transitive dependencies are included in the output.

```bash
$ ls -latrh big-binary
-rwxr-xr-x  1 dan  staff   103M Feb 25 16:31 big-binary
```

This 103-megabyte binary is big enough that it takes a non-trivial amount of time to transfer over the wire if it needed
to be downloaded or uploaded. It could be deployed to a Raspberry Pi or to a phone, but it's unclear if the code could
be compiled on lower-end devices.

Haskell's large executables motivate two major questions: why are they so large, and how could reducing their size
change the ecosystem?

## Is it all fluff?

### Using an executable packer

Compiling the static version takes about 17 minutes. Running [upx](https://upx.github.io/) on the static binary is
pretty quick, and outputs a binary 5 times smaller than the original. I also ran `upx` in `-1` (fast) and `--best` mode;
fast mode took the same amount of time as normal, and best mode took much longer with a negligible gain in terms of
space. I ran the executable after this, to make sure it still worked correctly.

```bash
$ stack clean
$ time stack build
stack build  2654.23s user 756.79s system 328% cpu 17:19.48 total
$ time upx -o big-binary-base big-binary
upx -o big-binary-base big-binary  11.10s user 0.13s system 99% cpu 11.258 total
$ ls -latrh big-binary-base
-rwxr-xr-x  1 dan  staff    21M Feb 25 16:31 big-binary-base
```

#### UPX caveats

An executable run with `upx` unpacks itself at startup. [Running multiple instances of the same `upx`-packed executable
results in wasted
memory](https://stackoverflow.com/questions/353634/are-there-any-downsides-to-using-upx-to-compress-a-windows-executable/355581).

### Dynamic linking

Running GHC using the dynamic linker should show how much of the code is part of the actual package, and not a
dependency.

```bash
$ stack clean
$ time stack build --ghc-options -dynamic
stack build --ghc-options -dynamic  2324.91s user 599.80s system 354% cpu 13:45.71 total
$ ls -latrh big-binary-dynamic
-rwxr-xr-x  1 dan  staff    52K Feb 25 21:15 big-binary-dynamic
```

52 kilobytes! That's 2000 times smaller than the static binary (smaller than one percent!). Compiling took 14 minutes,
so there's a small gain from using the `-dynamic` flag in terms of compile time, but nothing big. Running `upx` deflates
it even further (running in fast and best mode again had negligble results).

```bash
$ time upx -o big-binary-dynamic-base big-binary-dynamic
upx -o big-binary-dynamic-base big-binary-dynamic  0.00s user 0.00s system 70% cpu 0.011 total
$ ls -latrh big-binary-dynamic-base
-rwxr-xr-x  1 dan  staff   8.0K Feb 25 21:15 big-binary-dynamic-base
```

We end with an 8-kilobyte dynamic binary, about 6 times smaller than the original dynamic binary.

### Results

#### Time

| static | dynamic | time difference
| --- | --- | --- |
| 17 min | 14 min | 3 min faster |

#### Size

| | static | dynamic | size difference
| --- | --- | --- | --- |
| **without upx** | 103 MB | 52 KB | 2000x smaller |
| **with upx** | 21 MB | 8 KB | 2500x smaller |
| **size difference** | 5x smaller | 6x smaller |

## Potential consequences

`big-binary` has over 200 direct dependencies in its cabal file. Most are open source, some are proprietary. At a mature
company, it's easy to imagine many complicated executables interacting with each other. Deploying executables alone or
within some virtualized container environment results in shipping many redundant bytes.

Compiling dynamically would only help in cases where multiple executables were deployed to the same server (or within
the same container). Given modern operational architecture design, it's unlikely a SaaS company would deploy multiple
services to the same server, but it could help in situations where script executables are bundled together.

Packing Haskell executables using `upx` seems to be a more general solution to the size problem, and it can be performed
on both static and dynamic outputs. The results speak for themselves, as `upx` was able to compress both the static and
dynamic example 5-6 times smaller than the original.

This leads to questions which are more pertinent to Haskell developers in general. Is this extra cruft affecting compile
times somehow? Could it be that the compiler spends a lot of extra time managing unnecessary bytes that could be
stripped out? I'm not sure how to answer these questions, but having gone through this exercise I'm really interested to
find out if there are could be any tangible consequences.
