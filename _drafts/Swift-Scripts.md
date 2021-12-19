---
title:  "Executing Swift Package Manager executables from the command line"
permalink: swift-executables-from-command-line
---

### The goal: On demand compilation and execution of Swift Package Manager executable targets
Swift Package Manager is a great way to build command line utilities. It allows for simple scripting to be combined with a strong type system, modularity, and an ecosystem of packages like Swift Argument Parser. Unlike other languages like python, bash, or even standalone Swift scripts, Swift Package Manager executables do require compilation before running, which in turn requires 

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