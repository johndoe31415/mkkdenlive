# mkkdenlive
mkkdenlive is a build script tooling that downloads and installs the video
editing software kdenlive plus many of its dependencies. It has an easily
extensible data source that supplies all dependiencies in JSON format (plus the
way to build them), supports resuming the build chain at any point (e.g., when
playing around with settings of a particular package, you don't want to
re-build all dependencies that came before it) and supports patching single
files in single packages after their sources have been update (e.g., when you
want a specific patch to always make it into your custom kdenlive build).

## Why not use build-kdenlive.sh?
The melt framework already supplies a script that does pretty much this:
[build-kdenlive.sh](https://github.com/mltframework/mlt-scripts) -- however, I
still decedided to roll my own. Here's why:

   - I found the output of build-kdenlive.sh overwhelming and confusing. In
     particular, when a specific build step failed, I found it difficult to
     discern what the root cause was because all the logging is pasted into a single
     terminal output.
   - When build steps failed, I couldn't figure out how to resume the build
     (maybe with a changed version or changed build parameters) at that exact
     package instead of re-building all parent dependencies anew.
   - Later versions just wouldn't build anymore, but fail. While the root cause
     was super simple to find out, I still couldn't fix it. Nobody was able to help
     [when reporting the bug either](https://forum.kde.org/viewtopic.php?f=269&t=151446)
   - I wanted to customize the build process to include, for example, liboil,
     which isn't included in newer Ubuntu installations anymore.  For this I
     found messing around with the shell-based build script extremely difficult.
     It's not very well structured and doesn't invite tinkering with it.

## How is mkkdenlive different?
It addreses the aforementioned points:

   - The stdout output is lean (can be turned up using --verbose, of course),
     but individual build step logs go into one file each in the logs/
     directory. At the head of each log is the exact command that was run and the
     directory it was run in together with all environment variables to enable you
     to reproduce exactly what mkkdenlive did (in order to debug or tinker with
     settings). This is how the logs/ directory looks like after a kdenlive build:

     ```
     -rw------- 1 joe joe 416K   17.03.2018 18:47:16 2018_03_17_18_46_52_067_mlt_make.log
     -rw------- 1 joe joe  15K   17.03.2018 18:47:17 2018_03_17_18_47_16_068_mlt_make_install.log
     -rw------- 1 joe joe  12K   17.03.2018 18:47:17 2018_03_17_18_47_17_069_kdenlive_git_clean.log
     -rw------- 1 joe joe  893   17.03.2018 18:47:17 2018_03_17_18_47_17_070_kdenlive_git_reset.log
     -rw------- 1 joe joe  824   17.03.2018 18:47:17 2018_03_17_18_47_17_071_kdenlive_git_pull.log
     -rw------- 1 joe joe 6,6K   17.03.2018 18:47:20 2018_03_17_18_47_17_072_kdenlive_cmake.log
     -rw------- 1 joe joe 302K   17.03.2018 18:48:19 2018_03_17_18_47_20_073_kdenlive_make.log
     -rw------- 1 joe joe  32K   17.03.2018 18:48:21 2018_03_17_18_48_19_074_kdenlive_make_install.log
     ```

     And this is the header of each file, showing you how to exactly reproduce
     the particular build step:

     ```
     2018-03-17 18:47:20: make -j12

     CFLAGS="-I/home/joe/Videos/build/bin/include"
     LD_LIBRARY_PATH="/home/joe/Videos/build/bin/lib"
     PATH="/home/joe/Videos/build/bin/bin:/usr/joebin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/games:/usr/games:/home/joe/bin"
     PKG_CONFIG_PATH="/home/joe/Videos/build/bin/lib/pkgconfig"
     ========================================================================================================================
     CFLAGS="-I/home/joe/Videos/build/bin/include" LD_LIBRARY_PATH="/home/joe/Videos/build/bin/lib" PATH="/home/joe/Videos/build/bin/bin:/usr/joebin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/games:/usr/games:/home/joe/bin" PKG_CONFIG_PATH="/home/joe/Videos/build/bin/lib/pkgconfig" make -j12
     ```

   - When a build step fails, it's super simple to resume. Just do
     `./mkkdenlive -r` and it'll re-read the status file and resume with that
     last package. Or you want to build the whole dependency chain starting with a
     particular package? Simply do `./mkkdenlive -s FFmpeg` and it'll start building
     with FFmpeg and everything that follows.  Interested in only a single package
     in the chain? Sure enough: `./mkkdenlive -o mlt` will _only_ build melt.
   - The JSON-file is super simple to understand and customize according to
     your needs.

## What other options are there?
Take a look at the help page:
```
$ ./mkkdenlive --help
usage: mkkdenlive [-h] [-c filename] [-b filename] [-l path]
                  [-r | -s pkgname | -o pkgname | --postinstall-only]
                  [--force-postinstall] [-j jobcnt] [-t path] [--debug-build]
                  [-v]

optional arguments:
  -h, --help            show this help message and exit
  -c filename, --config filename
                        JSON build configuration to use. Defaults to
                        configuration.json.
  -b filename, --buildstat-filename filename
                        Filename to write build status into in order to resume
                        failed builds. Defaults to buildstats.json.
  -l path, --log-directory path
                        Log path. Defaults to logs.
  -r, --resume          Resume a previously failed build at the last package
                        that failed.
  -s pkgname, --start-with pkgname
                        Start with a particular package name in the build
                        configuration and run all builds afterwards.
  -o pkgname, --build-only pkgname
                        Only build a single package.
  --postinstall-only    Do not build anything, just run the postinstallation.
  --force-postinstall   By default, the postinstallation process will create
                        the appropriate symlinks for kdenlive only if they're
                        not present already. When present, mkkdenlive will
                        only check their correctness and warn if they're
                        wrong. When specifying this option, data inside
                        ~/.local/share may be overwritten because mkkdenlive
                        will forcefully remove the previous contents before
                        creating the symlinks. Use with caution.
  -j jobcnt, --parallel-build-jobs jobcnt
                        Build n jobs concurrently. Defaults to 12.
  -t path, --target path
                        Base directory in which kdenlive and dependencies are
                        downloaded and installed in. Defaults to ~/kdenlive-
                        build.
  --debug-build         Enable debugging mode (debug symbols, maybe less
                        optimization) for built components.
  -v, --verbose         Increase debugging verbosity level.
```

## How do I patch something?
Simply place the file(s) you want to replace in the patches/${pkgname}/
subdirectory and they'll be overwritten. One example is currently already in
there to make swfdec build.

## What are build dependencies?
To run mkkdenlive, you first need Python3, git and wget:

```
# apt-get install python3 git wget
```

To actually build kdenlive, this is required on a Ubuntu Artful installation:

```
# apt-get install build-essential cmake extra-cmake-modules googletest
  libgtest-dev nasm automake rsync libtool libglib2.0-dev libpangocairo-1.0-0
  libpango1.0-dev alsa libasound2-dev xutils-dev libegl1-mesa-dev libfftw3-dev
  libsdl2-dev libtheora-dev kio-dev libkf5notifications-dev
  libkf5notifyconfig-dev libkf5filemetadata-dev libkf5newstuff-dev
  qtbase5-dev-tools qtdeclarative5-dev libqt5svg5-dev libkf5bookmarks-dev
  libkf5kio-dev libkf5crash-dev libkf5doctools-dev breeze breeze-icon-theme
  libsamplerate0-dev libsox-dev libkf5declarative-dev libkf5declarative-dev
  meson qtquickcontrols2-5-dev libv4l-dev
```

Note that likely the concrete package names have already changed in a couple of
months, so take this only as a reference, not as a definitive list. If you do
fix the list, please don't forget to send a PR.

## What versions are built?
It depends, the details can be found in `configuration.json`. Most of the git
repos are taken from its source except for melt and kdenlive, where specific
tags are checked out to work around bugs that were present on master at the
time that I tried to build it. If you update any of the dependencies to newer
versions, also please feel free to send a PR.

## Why does this not work on my Mac?
Because I don't own a Mac, in all likelihood never will own a Mac and I don't
intend ever to build kdenlive for a Mac. If you look at the workarounds that
are necessary to get kdenlive to build on a Mac, you could check out
build-kdenlive.sh and fix the build system from there. But you're on your own,
I'm sorry.

## Your package sucks and you suck.
Send PRs, not hate :-)

## What's the license of mkkdenlive?
GNU GPL-3.
