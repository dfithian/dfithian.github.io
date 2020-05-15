---
title: GHC profiling (including Docker)
---

Recently, I tried to profile some memory and resource usage. As I often find with profiling, I struggled a little bit to
get set up. This post outlines profiling a GHC binary, including special considerations for Docker.

## Resources on profiling

[https://mpickering.github.io/ghc-docs/build-html/users_guide/profiling.html](https://mpickering.github.io/ghc-docs/build-html/users_guide/profiling.html)

## Compilation

Compile your program as `stack build --profile`.

* Due to a bug in profiling options with Template Haskell, you may also need to use `--ghc-options
  -fexternal-interpreter`. I noticed this with a bug similar to the one reported here:
  [https://github.com/commercialhaskell/stack/issues/4275](https://github.com/commercialhaskell/stack/issues/4275).

## Running

Run your program as `stack exec --profile $program -- +RTS -p -hm -RTS`

* `-p` will produce a time profiling report written to `$program.prof`
    * See
      [https://mpickering.github.io/ghc-docs/build-html/users_guide/profiling.html#time-and-allocation-profiling](https://mpickering.github.io/ghc-docs/build-html/users_guide/profiling.html#time-and-allocation-profiling)
      for other options
* `-hm` will produce a heap profile indexed by module (hence the `m`)
    * See [https://mpickering.github.io/ghc-docs/build-html/users_guide/profiling.html#profiling-memory-usage](https://mpickering.github.io/ghc-docs/build-html/users_guide/profiling.html#profiling-memory-usage) for other options

## Viewing profile data

You can view `.prof` data directly with a text editor. However, you will need the `hp2ps` utility (available with any
installation of `ghc` and/or `ghc-prof`) to view a graph of your program's memory usage: `hp2ps -c $program.hp` You can
open the graph called `$program.ps` in Preview on a Mac, or you can Google for PostScript rendering applications.

## Creating a Docker image

Your Docker container may need the following packages, which can be installed via `apt-get`: `ghc`, `ghc-prof`.

In addition, due to the nature of `.prof` files, which are not written while the program is running, you will need to
create a special entrypoint script. Special thanks to Rick Owens (`@rickowens`) for the skeleton script.

```bash
#!/bin/bash
# runs $program - to get profiling information, `ps ax | grep $program` and `kill -2 $pid`
while true
do
    # to run locally, change this line to: stack exec --profile $program -- +RTS -p -hm -RTS
    $program +RTS -p -hm -RTS
    sleep 3
    mv $program.hp $program.hp.keep
    mv $program.prof $program.prof.keep
done
```

Now, change your `Dockerfile` `ENTRYPOINT` to be this script. You will need to `COPY` this script to your container.

## Running a Docker image and extracting data

Once your Docker image is created and deployed, in order to collect profiling data you can run the following commands.

```bash
docker exec -it $container /bin/bash
ps ax | grep $program # get the pid
kill -2 $pid # -2 aka SIGINT aka CTRL-C
exit
docker cp $container:$path/$program.hp.keep $program.hp
docker cp $container:$path/$program.prof.keep $program.prof
tar -cvzf $program.tar.gz $program.hp $program.prof
```

Once the data is copied, extract it and refer to [viewing profile data](#viewing-profile-data).
