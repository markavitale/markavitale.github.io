---
title:  "Executing Swift Package Manager executables from the command line"
permalink: swift-executables-from-command-line
---

### Swift Package Manager and Executable Targets
Swift Package Manager is a great way to build command line utilities. It allows for simple scripting to be combined with a strong type system, modularity, and an ecosystem of packages like Swift Argument Parser. Unlike other languages like python, bash, or even standalone Swift scripts, Swift Package Manager executables do require compilation before running. Swift does provide a handy `swift run` command which allows for building and execution of a swift package executable all in one command, but requires your terminal's working directory to be the root of the swift package being run.

There are a few ways to approach building and running your executable:
1. Manually change directories to the root of the package (`cd /path/to/package.swift`) and then build/run the command from there (`swift run ExecutableTargetName [arguments]`)
    * Pro
        * Simplest approach, no extra deployment steps beyond having access to the source
        * No cruft left on your machine that may outlive the need for the swift package
    * Con
        * Lots of memorization of paths and complex commands to write
        * Multiple steps make it hard to chain with other commands
        * Changes your working directory and requires manually changing back to resume context
1. Compile the executable once and copy it somewhere in your path (e.g. `/usr/local/bin`)
    * Pro
        * Command is available on path, 
        * 
    * Con
        * Any changes to the package or executable source itself isn't picked up until manually rebuilt and the old binary is replaced
1. Write a wrapper script to build and run the swift package
    * Pro
        * 
    * Con
        * 

For my purposes, the wrapper script approach meets 

### Swift Package Manager Executables
- `cd ~/Code/tools/swift`
- `mkdir MyCommandLineTool`
- `cd MyCommandLineTool`
- `swift package init --type executable`

### `swift run` to build and execute
- cd into folder
- Use `swift run (target) ` 
- cd back to 

### Tying it all together
- make a bash script in /usr/local/bin
- Use `swift run --configuration release`

### More generic
- A generic swift-run command that takes the name of the package and optionally the executable within it