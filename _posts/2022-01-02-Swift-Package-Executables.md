---
title:  "Running Swift Package Manager executables from the command line"
permalink: swift-package-executables
---

### Swift Package Manager and Executable Targets
[Swift Package Manager](https://www.swift.org/package-manager/) is a great way to build command line utilities[^1]. It allows for simple scripting to be combined with a strong type system, modularity, and an ecosystem of packages like [Swift Argument Parser](https://github.com/apple/swift-argument-parser). Unlike other languages like python, bash, or even standalone Swift scripts, Swift Package Manager executables do require compilation before running. Swift does provide a handy `swift run` command which allows for building and execution of a swift package executable all in one command, but produces some extra output from building that may be undesirable. `swift run` also requires the working directory to contain the relevant `Package.swift` file or for that directory to be explicitly passed in via the `--package-path` option.

There are a few ways to approach building and running your executable: manually building and running the executable, compiling the executable once and storing it somewhere in your path, and writing a wrapper script that builds and runs your executable on demand.

Let's take a look at the various approaches.

### Manually build and run the executable

This is conceptually the simplest approach. We'll invoke `swift run` to incrementally build and run the command. We can optionally run the release configuration of the tool for performance.

```sh
swift run --package-path /path/to/package -c release ExecutableTargetName [arguments]
```

Executing this works great but the output retains some output from the implicit `swift build` happening which may interfere with commands that expect their stdout to be parsed.

```
[0/0] Build complete!
Hello, world!
```

Note: You can omit the `--package-path` option if your working directory is the root of the package.

* Pro
    * Simplest approach, no extra deployment steps beyond having access to the source
    * No cruft left on your machine that may outlive the need for the swift package
* Con
    * Lots of memorization of paths and complex commands to write
    * Multiple steps make it hard to chain with other commands
    * Changes your working directory and requires manually changing back to resume context
    * Some overhead of checking if any changes require the executable to be recompiled since the last run
    * Swift build output pollutes STDOUT

### Compile the executable once
Compile the executable once and copy it somewhere in your path (e.g. `/usr/local/bin`). Note that we can take this opportunity to rename the executable to something more idiomatic for the command line.

```shell
swift build -c release --package-path "/path/to/package"
cp -f /path/to/package/.build/release/ExecutableTargetName /usr/local/bin/executable-target-name
```

Calling executable-target-name now no longer invokes a swift build command, so our output is clean of any extra printing from `swift build`.

```
Hello, world!
```

* Pro
    * Command is available on path making it easy to execute
    * No need to memorize paths, change directories, or even keep the source of the package after building and copying the built binary
    * Easy to chain with other commands
    * No Swift build output on STDOUT
* Con
    * Any changes to the package or executable source itself isn't picked up until manually rebuilt and the old binary is replaced
    * Binary lives on your path until you explicitly delete it

### Wrapper script
Put a custom wrapper script in your path (e.g. `/usr/local/bin`) that builds and runs the swift package. The wrapper script can be named something more idiomatic for the command line.

```shell
#! /bin/zsh

# Define the path to the package
PACKAGE_PATH="/path/to/package"

# Execute the build
# Execute in a subshell to avoid success messages from polluting our command's STDOUT
BUILD_STDOUT=$(swift build --configuration release --package-path "$PACKAGE_PATH" --product "ExecutableTargetName")
BUILD_RESULT=$?

if [[ $BUILD_RESULT -eq 0 ]]; then
    # The build succeeded. Let's now run the script, skipping the build since we've just built
    swift run --configuration release --package-path "$PACKAGE_PATH" --skip-build "ExecutableTargetName" "$@"
    
    # Forward the script run's exit status
    RUN_RESULT=$?
    exit $RUN_RESULT
else
    # If we failed, print out the build failure STDOUT and forward the build's exist status
    echo $BUILD_STDOUT
    exit $BUILD_RESULT
fi
```

Because of our use of the subshell, successful builds and runs of the script do not show any output from the `swift build` command. Build failures and anything on STDERR coming from the build command will be printed as expected.

```
Hello, world!
```

* Pro
    * Command is available on path making it easy to execute
    * No need to memorize paths or manually change directories
    * Changes to the package or executable source are immediately available
    * No Swift build output on STDOUT
* Con
    * Wrapper script lives somewhere in your path until you explicitly delete it
    * Stops working if the source of the swift package moves or is deleted
    * Some overhead of checking if any changes require the executable to be recompiled since the last run

### Which approach works best?

For my purposes, the wrapper script approach has the right balance of pros and cons. Most code I write lives in a single large repo with tools living as local packages in a subdirectory. There is also an `init.sh` file that lives in the root of this repo. Sourcing that file sets up some handy conveniences like adding the repository's scripts folder to the path for easy access to all the command line tools.

With this type of repository structure, we can take things even further and reduce or outright eliminate many cons of the wrapper script approach. Since our wrapper script and swift package are both part of the same repo, we can make changes to the two in lock-step to avoid getting out of sync. And when an executable is obsolete and removed from the repo, the wrapper script can be removed from path simultaneously.

### Monorepo tweaks

Some minor tweaks to reduce the cons of the wrapper script approach when operating in a monorepo follow. First we add an init file that developers will source in their shell when working in this repo.

`init.sh`:
```shell
#! /bin/sh

# Get the directory containing this script, from https://unix.stackexchange.com/questions/76505
export MY_REPO_ROOT=$(exec 2>/dev/null;cd -- $(dirname "$0"); unset PWD; /usr/bin/pwd || /bin/pwd || pwd)

# Add the tools/wrappers directory to our path for easy execution
export PATH=$PATH:"$MY_REPO_ROOT/tools/wrappers"
```

Next we update our wrapper to use generic paths based on the repo root variable defined in `init.sh`.

Wrapper script for a monorepo in `tools/wrappers/example-executable`:
```shell
#! /bin/zsh

# Define the path to the package
PACKAGE_PATH="$MY_REPO_ROOT/tools/swift/ExampleExecutable"

# Execute the build
# Execute in a subshell to avoid success messages from polluting our command's STDOUT
BUILD_STDOUT=$(swift build --configuration release --package-path "$PACKAGE_PATH" --product "ExampleExecutableTarget")
BUILD_RESULT=$?

if [[ $BUILD_RESULT -eq 0 ]]; then
    # The build succeeded. Let's now run the script, skipping the build since we've just built
    swift run --configuration release --package-path "$PACKAGE_PATH" --skip-build "ExampleExecutableTarget" "$@"
    
    # Forward the script run's exit status
    RUN_RESULT=$?
    exit $RUN_RESULT
else
    # If we failed, print out the build failure STDOUT and forward the build's exist status
    echo $BUILD_STDOUT
    exit $BUILD_RESULT
fi
```

### A more generic approach

If you just have a couple of swift package executables having a few of these individual wrapper scripts isn't too bad. If you have a larger suite of tools, you may want to invest in a generic runner script to reduce duplication between wrapper scripts.

Generic wrapper for SPM execution in `tools/wrappers/spm-runner`
```shell
#! /bin/zsh

# A simple runner for Swift Package Manager executables in this repo's tools/swift directory.
# Call this with the first argument being the package name and the second argument being the executable product name.
# All further arguments will be forwarded to the executable

# Example: spm-runner ExampleExecutable ExampleExecutableTarget [arguments-for-executable]

PACKAGE_NAME="$1"
EXECUTABLE_PRODUCT_NAME="$2"

# Define the path to the package
PACKAGE_PATH="$MY_REPO_ROOT/tools/swift/$PACKAGE_NAME"

# Execute the build
# Execute in a subshell to avoid success messages from polluting our command's STDOUT
BUILD_STDOUT=$(swift build --configuration release --package-path "$PACKAGE_PATH" --product "$EXECUTABLE_PRODUCT_NAME")
BUILD_RESULT=$?

if [[ $BUILD_RESULT -eq 0 ]]; then
    # The build succeeded so run the script, skipping the build since we've just built, and forwarding on arguments meant for the executable
    swift run --configuration release --package-path "$PACKAGE_PATH" --skip-build "$EXECUTABLE_PRODUCT_NAME" "${@:3}"
    
    # Forward the script run's exit status
    RUN_RESULT=$?
    exit $RUN_RESULT
else
    # If we failed, print out the build failure STDOUT and forward the build's exist status
    echo $BUILD_STDOUT
    exit $BUILD_RESULT
fi
```

Command-specific wrapper in `tools/wrappers/executable-example`
```shell
#! /bin/zsh

"$MY_REPO_ROOT/tools/wrappers/spm-runner" ExampleExecutable ExampleExecutableTarget "$@"
exit $?
```

With this simple runner and wrapper, we can now build and run Swift Package Manager based executables directly from command line, ensuring you're always invoking the latest version of the executable. For a sample repo that implements the monorepo-style wrappers, see [this example repository](https://github.com/markavitale/spm-executable-wrapper-example). Thanks for reading!

---

[^1]: [Building a command line tool using the Swift Package Manager - Swift by Sundell](https://www.swiftbysundell.com/articles/building-a-command-line-tool-using-the-swift-package-manager/)
