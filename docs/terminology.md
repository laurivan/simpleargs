<!-- GENERATED TOC -->
<!-- TOC START -->
## Table of Contents

  - [Terminology](#terminology)
<!-- TOC END -->

## Terminology
Before diving into the syntax of simpleargs a few words about terminology.
The example below demonstrates the three elements of a command invocation:
_environment variable assignments_, _command name_, and _arguments_ (options and parameters)
```
                                                             (positional)
                                     options                  parameters
                           --------------------------    --------------------
> log_level=INFO mycommand -v --user alex --sort=time -- input.txt output.txt
  -------------- --------- --------------------------------------------------
    environment   command                       arguments
    variable      name
    assignment
```

* **Environment variable assignments**  
  These augment the environment seen by the command.
  This syntax might not be particularly widely known probably because it is mentioned only in passing
  in [Bash reference manual](https://tiswww.case.edu/php/chet/bash/bashref.html#Command-Execution-Environment).
  > The environment for any simple command or function may be augmented temporarily by prefixing it with parameter assignments,
  as described in Shell Parameters.
  These assignment statements affect only the environment seen by that command.

  In the above command log level is set using this syntax: `log_level=INFO`.
  It has the same effect as `export log_level=INFO` before running the command except
  it affects only the environment where the command is run (not the current shell).
* **Command name or script name**  
  Usually the first word in a command invocation since environment variable assignments are rarely used.

* **Arguments**  
  The tokens after the command name. They can be further divided into two types: _options_ and _parameters_.
  * **Options or flags or switches**  
    The tokens prefixed with `-` or `--`.
    The terms are used pretty much interchangeably.
    In this documentation _option_ is sometimes used to describe the functionality as a whole
    while _flag_ denotes the actual string that is typed onto the command line. For example,

    > The verbose _option_ can be turned on by using either the _short flag_ `-v` or the _long flag_ `--verbose`.

    There are three types of options:
    1. _No argument_ options like `-v` that are either on or off.
       In some cases the flag can be given multiple times.
       For example, `ps` uses `-w` for wide output and `-w -w` (or `-ww` for short) for unlimited width output.
    2. _Required argument_ options like `--user` require a value as the next word on the command line.
    3. _Optional argument_ options like `--sort` may or may not have a value.
       The syntax with equals sign has to be used.
       Otherwise `--sort time` would be ambigious: is `time` the value of sort option or a positional parameter?

  * **(Positional) parameters**  
    Often define the input and/or output (files) of the command.
    Parameters can be mandatory (`cp`) or optional (`ls`).

There is also the _end of options_ `--` argument that is used to signal the end of options:
any following arguments are considered non-option arguments i.e. parameters.
Its usage is often optional but sometimes it is needed to remove ambiguity.
for example when a positional parameter might start with a hyphen `-`.
Consider using `grep` with a pattern that starts with a hyphen:
```
$ echo "high-risk" | grep -- "-r"
high-risk
```
