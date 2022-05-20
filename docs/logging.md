<!-- GENERATED TOC -->
<!-- TOC START -->
## Table of Contents

  - [Logging](#logging)
    - [Logging functions `log`](#logging-functions-log)
    - [Logging functions `log_vars`](#logging-functions-log_vars)
    - [Configuration parameters](#configuration-parameters)
      - [log_level](#log_level)
      - [log_date_format](#log_date_format)
      - [log_enable_colors](#log_enable_colors)
      - [log_display_level](#log_display_level)
      - [log_align_level](#log_align_level)
      - [log_dest](#log_dest)
      - [log_file](#log_file)
    - [Example](#example)
<!-- TOC END -->

## Logging
Simpleargs library includes _rudimentary_ logging capabilities.
They were originally meant for internal debugging purposes.
However, the functions are available also for the user of the library.
The logging framework consists of logging functions and a set of configuration parameters that control the logging.

### Logging functions `log`
Log statements are output using `log` function.
Two forms of invocation are supported:
* `log <level> <message>`  
  The first form takes strictly two arguments: log level and message.
  Log level is one of _ERROR_, _WARN_, _INFO_, _CONFIG_, _FINE_, _FINER_ and _FINEST_.
  The message is printed with a newline automatically appended.
  More specifically the message is printed by executing `printf "%s\n" "${message}"`
  Hence, the following log statements...
  <!--@evalsh filter-simpleargs-script -r '/^log /' $SIMPLEARGS_DOC_SCRIPTS/logging/simple-log -->
  ```sh
  log INFO "Hello world!"
  log INFO "Escape codes (\t\n) are printed as is!"
  log INFO "%% <- two '%' signs"
  log INFO "Format specifiers (%d, %s %.2f) are not interpreted."
  name=Jim
  log INFO "Include variables as usual if you like, ${name}!"
  ```
  ...print
    <!--@eval shell-session-simulator --no-command-print --path $SIMPLEARGS_DOC_SCRIPTS/logging -c 'simple-log' -->
    ```
    INFO: Hello world!
    INFO: Escape codes (\t\n) are printed as is!
    INFO: %% <- two '%' signs
    INFO: Format specifiers (%d, %s %.2f) are not interpreted.
    INFO: Include variables as usual if you like, Jim!
    ```
* `log <level> <format string> <argument>...`  
  This form interprets the second parameter as
  [printf format string](http://wiki.bash-hackers.org/commands/builtin/printf#format_strings).
  The subsequent arguments correspond to the format specifiers in the format string.
  More specifically the message is printed by executing
  ```sh
  printf "${format_string}" "${arg1}" "${arg2}" ...
  ```
  Hence, the following log statements...
  <!--@evalsh filter-simpleargs-script -r '/^# An explicit newline/' $SIMPLEARGS_DOC_SCRIPTS/logging/complex-log -->
  ```sh
  # An explicit newline '\n' is needed
  log INFO "Hello %s!\n" "world"
  # '\t' escape code used to indent the message
  log INFO "\t%s is %d years old.\n" Jeff 52

  # Give a dummy argument to force "format string invocation" thus
  # omitting the newline (which is printed by the subsequent echo).
  log INFO "Processing something..." ""
  if log_enabled INFO
  then
      do-some-processing && echo "DONE" || echo "FAILED"
  fi
  percent=70
  log INFO "%d%% discount!\n" "${percent}"
  ```
  ...print
  <!--@eval shell-session-simulator --no-command-print --path $SIMPLEARGS_DOC_SCRIPTS/logging -c 'complex-log' -->
  ```
  INFO: Hello world!
  INFO: 	Jeff is 52 years old.
  INFO: Processing something...DONE
  INFO: 70% discount!
  ```

### Logging functions `log_vars`

For debugging there is `log_vars` function which can be used to print the values of variables neatly aligned.
The function can be used with both ordinary variables and arrays and associative arrays as well.
The first argument to `log_vars` is the log level.
The rest of the arguments can be any of the following:

* variable names
* a single (quoted) space to print an empty line
* string that starts with '#' to print a labeled separator line

For example,
<!--@evalsh filter-simpleargs-script -r 'e/^log_level=/' $SIMPLEARGS_DOC_SCRIPTS/logging/logging-variables -->
```sh
myvar="my value"
othervar="second value"
log_vars INFO myvar othervar

port=80
protocol=http
names=( John Rachel Kyle )
declare -A wheels=( [car]=4 [bike]=2 [unicycle]=1 )

log_vars INFO "#Connection config" port protocol " " names wheels
```
prints
<!--@eval shell-session-simulator --no-command-print --path $SIMPLEARGS_DOC_SCRIPTS/logging -c 'logging-variables' -->
```
INFO:    myvar: 'my value'
INFO: othervar: 'second value'
INFO:  Connection config
INFO:               port: '80'
INFO:           protocol: 'http'
INFO: 
INFO: # names[@]
INFO:   [0]: 'John'
INFO:   [1]: 'Rachel'
INFO:   [2]: 'Kyle'
INFO: # wheels[@]
INFO:       [bike]: '2'
INFO:   [unicycle]: '1'
INFO:        [car]: '4'
```

### Configuration parameters
The output of the logging statements can be controlled by modifying a set of configuration parameters.

#### log_level
Sets the level that is used to decide which log messages are output.

* Possible values: _ALL_, _ERROR_, _WARN_, _INFO_, _CONFIG_, _FINE_, _FINER_, _FINEST_ and _OFF_
* Default value: _WARN_

#### log_date_format
Controls the format of the timestamp printed alongside a log message.
The value is passed directly to `date` command (see [`man date`](http://man7.org/linux/man-pages/man1/date.1.html) for more information).

* Example values:
  * `+%S.%3N` (seconds and milliseconds)
  * `+%H.%M:%S` (hours, minutes and seconds)
  * an empty string (no timestamp is printed)
  * special keyword `time` (results in using date format `+%H:%M:%S.%3N`)
* Default value: an empty string (no timestamp is printed)

#### log_enable_colors
Controls whether log output is colored.
The coloring is done only if the log output destination is terminal.
That is, ANSI color escape codes are not used if writing to a file even if the value of the parameter is `true`.

* Possible values: `true` or `false`
* Default value: `true`

Message colors:
* ERROR: red
* WARN: yellow
* INFO, CONFIG, FINE: white
* FINER, FINEST: gray

#### log_display_level
Controls whether the log level is printed alongside a log message.
* Possible values: `true` or `false`
* Default value: `true`

#### log_align_level
Controls whether the log messages are aligned neatly when log levels are printed (see <<log_display_level>> TODO).
That is, if enabled the length of the longest log level name is used as the field width when printing the log level.
With abundant logging this might be considered more readable and/or aesthetically more pleasing.

* Possible values: `true` or `false`
* Default value: `false`

#### log_dest
Sets the log output destination.
The value is a file descriptor number.

* Possible values:
  * `1` for stdout
  * `2` for stderr
  * custom file descriptor number (must be opened by the user)
* Default value: `1` (stdout)

Note that setting `log_file` parameter overrides this value.

#### log_file
Setting this parameter makes the log output end up in a file.
The value is a path for the file where the log output should be written.
Non-existing file is created and an existing file overwritten.

Note that this parameter takes precedence over `log_dest`.

### Example
Using the logging framework is fully optional.
However, instead of using `echo` it is handy and practically no more effort to do something like below:
<!--@evalsh filter-simpleargs-script -r 1 $SIMPLEARGS_DOC_SCRIPTS/logging/log-example -->
```sh
#!/usr/bin/env bash

log_align_level=true
log_date_format=time
# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" \
         --log-level log_level_tmp @default=INFO \
         @validvalues=OFF,ERROR,WARN,INFO,CONFIG,FINE,FINER,FINEST,ALL \
         @doc="Logging level, default @{d} (values: @{v})"
sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
# ----------------------------------------------------------------------------
log_level=${log_level_tmp}

log ERROR error
log WARN warn
log INFO info
log CONFIG config
log FINE fine
log FINER finer
log FINEST finest
```
Note that since the logging framework is used also internally by simpleargs
`log_level` is set only after simpleargs is finished.
The initial value of `log_level` is `WARN` which normally results in no output produced by simpleargs
unless there's an error.
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/logging -c 'log-example -h' -c 'log-example --log-level INFO' -->
```
$ log-example -h
Usage: log-example [OPTION]...

Options:
  -h, --help                    Print usage instructions and exit.
  --log-level ARG               Logging level, default INFO (values: 'OFF',
                                'ERROR', 'WARN', 'INFO', 'CONFIG', 'FINE',
                                'FINER', 'FINEST', and 'ALL')
$ log-example --log-level INFO
20:32:02.332  ERROR: error
20:32:02.335   WARN: warn
20:32:02.339   INFO: info
```
