<!-- GENERATED TOC -->
<!-- TOC START -->
## Table of Contents

  - [Installation](#installation)
    - [Requirements](#requirements)
    - [Installing](#installing)
<!-- TOC END -->

## Installation
### Requirements
* **bash** - version 4.3 or newer
* **getopt** - GNU _enhanced_ getopt from `util-linux` package is used by simpleargs under the hood
* **standard utilities** - commands such as `grep`, `sed`, `printf`, etc.

The above requirements should be fulfilled by any modern Linux distribution.

<details>
  <summary>macOS/OS X users</summary>

Using simpleargs on OSX is possible but one needs to install a couple of dependencies that come out of the box on Linux.
First of all, install the newest `bash` with [Homebrew](https://brew.sh/) and optionally make it your default shell.
See for example [this StackExchange answer](https://apple.stackexchange.com/a/292760) for instructions.

Second, install a set of GNU utilities using `brew` command:
```
brew install coreutils gnu-sed gnu-getopt gettext
```
You might want to install GNU `grep` and `find` as well
although they are not required by simpleargs.
```
brew install grep findutils
```
On top of that the GNU versions of the binaries such as `sed` and `getopt` need to be taken into use by adding them into your `PATH`.
[This StackOverflow post](https://stackoverflow.com/questions/57972341/how-to-install-and-use-gnu-ls-on-macos)
shows how to do it.
For example, my `~/.bashrc` contains the following lines:
```
export PATH="/usr/local/opt/coreutils/libexec/gnubin:${PATH}"
export MANPATH="/usr/local/opt/coreutils/libexec/gnuman:${MANPATH}
export PATH="/usr/local/opt/gnu-sed/libexec/gnubin:${PATH}"
export MANPATH="/usr/local/opt/gnu-sed/libexec/gnuman:${MANPATH}"
export PATH="/usr/local/opt/gnu-getopt/bin:${PATH}"
export MANPATH="/usr/local/opt/gnu-getopt/bin:${MANPATH}"

# GNU grep and find
export PATH="/usr/local/opt/grep/libexec/gnubin:${PATH}"
export MANPATH="/usr/local/opt/grep/libexec/gnuman:${MANPATH}"
export PATH="/usr/local/opt/findutils/libexec/gnubin:${PATH}"
export MANPATH="/usr/local/opt/findutils/libexec/gnuman:${MANPATH}"
```

</details>

### Installing
Download the latest simpleargs bundle in your home directory and install
simpleargs by issuing the following commands as an ordinary user:
```
# simpleargs bundle downloaded into ~/simpleargs-bundle
. ~/simpleargs-bundle
sa-install-local ~/simpleargs-bundle

# For system wide installation, run as root:
# sa-install-global ~/simpleargs-bundle
```
Below you can see an example of the expected output.
```
Installing simpleargs...
Installation complete
         User home directory: /home/eles
                      Bundle: /home/eles/.simpleargs.d/simpleargs-bundle
  Utility functions + config: /home/eles/.simpleargs.d/simpleargs.rc
          Completion library: /home/eles/.simpleargs.d/simpleargs-completion
          Configuration file: /home/eles/.simpleargs.d/simpleargs.conf
       Bootstrap appended to: /home/eles/.bashrc
```

As indicated by the output the installation performed two steps
* added the library files under `~/.simpleargs.d`
* appended bootstrap code to `~/.bashrc`

That's it! Open a new shell (or run `. ~/.bashrc`) to source the bootstrap code placed in your `.bashrc` and run
```
sa-requirements
```
to test whether your system has all the required dependencies.
If yes, you're ready to write your first `simpleargs` based script.
