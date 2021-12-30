---
title:  "Executing Swift Package Manager executables from the command line"
permalink: swift-executables-from-command-line
---

### Swift Package Manager and Executable Targets
Swift Package Manager is a great way to build command line utilities. It allows for simple scripting to be combined with a strong type system, modularity, and an ecosystem of packages like Swift Argument Parser. Unlike other languages like python, bash, or even standalone Swift scripts, Swift Package Manager executables do require compilation before running. Swift does provide a handy `swift run` command which allows for building and execution of a swift package executable all in one command, but requires your terminal's working directory to be the root of the swift package being run.

There are a few ways to approach building and running your executable: manually building and running the executable, compiling the executable once and storing it somewhere in your path, and writing a wrapper script that builds and runs your executable on demand.

Let's take a look at the various approaches.

### Manually build and run the executable

This is conceptually the simplest approach. When running the script, we'll first navigate to the directory that contains the Package.swift we want to execute.

```shell
cd /path/to/package
```

 Then we'll invoke `swift run` to incrementally build and run the command.
     
```sh
swift run ExecutableTargetName [arguments]
```

* Pro
    * Simplest approach, no extra deployment steps beyond having access to the source
    * No cruft left on your machine that may outlive the need for the swift package
* Con
    * Lots of memorization of paths and complex commands to write
    * Multiple steps make it hard to chain with other commands
    * Changes your working directory and requires manually changing back to resume context
    * Some overhead of checking if any changes require the executable to be recompiled since the last run

### Compile the executable once
Compile the executable once and copy it somewhere in your path (e.g. `/usr/local/bin`)

**TODO: Write the code to do this**

* Pro
    * Command is available on path making it easy to execute
    * No need to memorize paths, change directories, or even keep the source of the package after building and copying the built binary
    * Easy to chain with other commands
* Con
    * Any changes to the package or executable source itself isn't picked up until manually rebuilt and the old binary is replaced
    * Binary lives on your path until you explicitly delete it

### Wrapper script
Put a custom wrapper script to navigate to the package directory, build and run the swift package, and return to original working directory somewhere in your path (e.g. `/usr/local/bin`)

**TODO: Write the code to do this**

* Pro
    * Command is available on path making it easy to execute
    * No need to memorize paths or manually change directories
    * Changes to the package or executable source are immediately available
* Con
    * Wrapper script lives somewhere in your path until you explicitly delete it
    * Stops working if the source of the swift package moves or is deleted
    * Some overhead of checking if any changes require the executable to be recompiled since the last run

### Which approach works best?

For my purposes, the wrapper script approach has the right balance of pros and cons. Most code I write lives in a single large repo with tools living as local packages in a subdirectory. There is also an `init.sh` file that lives in the root of this repo. Sourcing that file sets up some handy conveniences like adding the repository's scripts folder to the path for easy access to all the command line tools.

With this type of repository structure, we can take things even further and reduce or outright eliminate many cons of the wrapper script approach. Since our wrapper script and swift package are both part of the same repo, we can make changes to the two in lock-step to avoid getting out of sync. And when an executable is obsolete and removed from the repo, the wrapper script can be removed simultaneously.

### A more generic approach

**TODO: Write the code to do this**

- A generic swift-run command that takes the name of the package and optionally the executable within it