<!-- GENERATED TOC -->
<!-- TOC START -->
## Table of Contents

  - [Command Completion](#command-completion)
    - [Advanced command completion](#advanced-command-completion)
    - [Option flag completion](#option-flag-completion)
      - [Optional value options](#optional-value-options)
    - [Valid values completion](#valid-values-completion)
    - [Filename filtering](#filename-filtering)
<!-- TOC END -->

## Command Completion
Scripts using simpleargs get command completion (aka autocompletion) for free.
When simpleargs based script is executed for the first time
the option and parameter definitions are parsed and the result is cached.
If the script is modified the definitions are parsed again
when the script is executed next time.
The cached option and parameter definitions are used for command completion.
<!-- TODO: Link to "Internals" section (not yet written) -->
<!-- TODO: Create a new top level section "Internals and Development" -->

In the previous sections we have shown how one can
* restrict the valid values of an option or parameter using [@validvalues](directives.md#validvalues),
  [@validvaluescommand](directives.md#validvaluescommand) or [@validvaluesfile](directives.md#validvaluesfile) directives
* validate script's input values using [validation directives](validation.md#adding-validation-directives)
  such as [@@glob](validation.md#glob) or [@@grep](validation.md#grep)

Using these directives makes your script more robust, but they serve also another purpose — command completion.
Let's say your script supports `--security` option with three possible values:
<!--@evalsh filter-simpleargs-script -r 1  $SIMPLEARGS_DOC_SCRIPTS/command-completion/example -->
```sh
#!/usr/bin/env bash

# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" --security arg @validvalues=low,medimum,high
sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
# ----------------------------------------------------------------------------
echo "Security level: ${security}"
```

Copy-paste the script into a file named `example`,
place it in your `PATH`, and make it executable.
To get the parsing result placed in the cache the script needs to be executed.
It's handy to use `-h/--help` option which ensures that the rest of the script
is not run.
To get the command completion routines taken into use
one can open a new terminal or run `sa-refresh-completion` in the current one.
```
$ example -h
...
$ sa-refresh-completion
```
The refreshing needs to be done only once
when the script has been created (and executed).
Now one can use `tab` key to get values completed automatically.
```
$ example -<tab><tab>
--help      --security  -h
$ example --s<tab>
$ example --security m<tab>
$ example --security medium
```
Remember that if you modify the script you need to execute it
before the changes take effect in the completion routines.
Executing the script with `-h` flag was one way of refreshing the cache.
Another one is to set `sa_parse_only` variable to `true`
for the execution of the script
to only get the options and parameter definitions parsed
(and thus the cache refreshed).
```sh
$ emacs example               # Modify the script
$ sa_parse_only=true example  # Execute the script; stop after parsing option definitions
```

### Advanced command completion
The types of autocompletion provided by simpleargs can be divided into
* option flag completion
* valid values completion
* filename filtering

The functionality is best described with an example.
The options and their semantics are arbitrary and tailored only for illustrative purposes.
<!--@evalsh filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/command-completion/myscript -->
```sh
sa_parse "$0" \
         -a -b -c \
         -v/--verbose \
         -p port \
         -s/--security low,medium,high@default \
         --compass-point north,south,west,east \
         -o/--order none@default,name,type,size @optionalvalue \
         \
         --signal arg @validvaluescommand="compgen -A signal" \
         --user-or-group arg \
         @validvaluescommand="compgen -A user; compgen -A group" \
         --write-log log_file @@!exists \
         --custom-config @@file @@glob "*.rc" \
         "<operation>" @validvalues=scan,prune,upload \
         "<input file>" @@glob "*.c"

```

### Option flag completion
You are probably already familiar with _option flag completion_.
Try typing in your terminal
```
grep --
```
and hit `tab` key twice.
The long flags accepted by grep are displayed.
Simpleargs provides the same functionality for your scripts:
```
$ myscript -<tab><tab>
--compass-point  --security       -a               -p
--custom-config  --signal         -b               -s
--help           --user-or-group  -c               -v
--order          --verbose        -h
--order=         --write-log      -o
$ myscript --c<tab><tab>
--compass-point  --custom-config
$ myscript --co<tab>
$ myscript --compass-point
```

The reader is recommended to read
[the POSIX recommendations](https://www.gnu.org/software/libc/manual/html_node/Argument-Syntax.html) for command line arguments.
TL;DR; Since simpleargs uses `getopt` internally simpleargs based scripts accept command line arguments in a similar manner.
For example, options can be combined:
```
$ myscript -a -b -c
# ...is equivalent to
$ myscript -abc
```
The option value can be given...
```
# ...separately
$ myscript -p 8080
# ...or as one token
$ myscript -p8080
```
You can even write
```
$ myscript -abcp8080
```

or

```
$ myscript -abcsmedium
```
and command completion will help you all the way.
```
$ myscript -abcs<tab> <1>
$ myscript -abcs me<tab> <2>
$ myscript -abcs medium
```

* **<1>** Pressing `tab` will append a space. This signals that a valid option was detected.
* **<2>** Since the previous option was `-s` (`--security`) the value is completed as expected.

Furthermore, you can even utilize the more rarely used ways of specifying options and have your values still completed correctly:
```
$ myscript -abcsme<tab>
$ myscript -abcsmedium
# ...which is probably not that readable but still equivalent to
$ myscript -a -b -c -s medium
```

#### Optional value options
A few words about [options with optional values](options-and-parameters.md#optional-argument-option).
In our example `-o/--order` is of that kind.
```
$ myscript -o
# ...is the same as
$ myscript --order

# Similarly
$ myscript -oname
# ...is the same as
$ myscript --order=name

# However
$ myscript -o name
# ...or
$ myscript --order name
```
The last example might be deceptive.
Argument `name` is not the value of the option (`-o` or `--order`) but a separate token altogether.
That is the reason for the autocompletion behaviour:
```
$ myscript -o<tab>
$ myscript -o <1>
$ myscript -ona<tab> <2>
$ myscript -oname
```
* **<1>** No space is appended after `-o` since the user might want to...
* **<2>** ...provide a value for the option

Similarly
```
$ myscript --or<tab>
$ myscript --order<tab<tab> <1>
--order   --order=          <2>
$ myscript --order=n<tab>   <3>
$ myscript --order=name
```
* **<1>** `--order` is completed but no space is appended since...
* **<2>** ...the user might want to provide a value for the option in which case...
* **<3>** ...`=` needs to be typed before the value.

### Valid values completion
Similarly to options, defining valid values for a parameter allows its values
to be autocompleted.
Our example script defines valid values for the first _parameter_:
<!--@evalsh filter-simpleargs-script -r '/scan,prune/' -r 'e//' $SIMPLEARGS_DOC_SCRIPTS/command-completion/myscript -->
```sh
         "<operation>" @validvalues=scan,prune,upload \
```
This allows using autocompletion:
```
$ myscript -a --verbose -p 8080 <tab><tab>
prune   scan    upload
$ myscript -a --verbose -p 8080 u<tab>
$ myscript -a --verbose -p 8080 upload
```

### Filename filtering
The last parameter of the example script is has no valid values defined:
```sh
"<input file>" @@glob "*.c"
```
In this case simpleargs falls back to default file completion of Bash.
However, since there's additional validation rules the completion can
be a bit more clever:
```
$ ls -l
total 0
-rw-r--r-- 1 lval wheel  0 Feb 22 21:33 config.c
-rw-r--r-- 1 lval wheel  0 Feb 22 21:33 config.h
drwxr-xr-x 2 lval wheel 64 Feb 22 21:33 data
-rw-r--r-- 1 lval wheel  0 Feb 22 21:33 main.c
-rw-r--r-- 1 lval wheel  0 Feb 22 21:33 mock.c
-rw-r--r-- 1 lval wheel  0 Feb 22 21:33 mock.h
-rw-r--r-- 1 lval wheel  0 Feb 22 21:33 resource.c
-rw-r--r-- 1 lval wheel  0 Feb 22 21:33 resource.h
drwxr-xr-x 2 lval wheel 64 Feb 22 21:33 test
drwxr-xr-x 2 lval wheel 64 Feb 22 21:33 util
$ myscript scan <tab><tab>
config.c    data/       main.c      mock.c      resource.c  test/       util/
$ myscript scan mo<tab>
$ myscript scan mock.c
```
Because `<input file>` needs to have `.c` suffix
the completion won't suggest header (`.h`) files.
However, it will suggest directories even though they don't have `.c` suffix
because files within the directories might.