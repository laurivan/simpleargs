<!-- GENERATED TOC -->
<!-- TOC START -->
## Table of Contents

  - [Validation directives](#validation-directives)
    - [Resolving validation directive](#resolving-validation-directive)
    - [Using validation directives](#using-validation-directives)
    - [Combining validations with @validvalues](#combining-validations-with-validvalues)
    - [Standard (library) validation directives](#standard-library-validation-directives)
      - [@@grep](#grep)
      - [@@egrep](#egrep)
      - [@@glob](#glob)
      - [@@exists](#exists)
      - [@@file](#file)
      - [@@dir](#dir)
      - [@@notempty](#notempty)
      - [@@readable](#readable)
      - [@@writable](#writable)
      - [@@executable](#executable)
      - [@@modified](#modified)
      - [@@filetype](#filetype)
      - [@@int](#int)
      - [@@float](#float)
    - [Writing a custom validation directive](#writing-a-custom-validation-directive)
      - [Validation command interface](#validation-command-interface)
      - [Example validation directive](#example-validation-directive)
      - [Customizing the error message](#customizing-the-error-message)
<!-- TOC END -->

## Validation directives
There is an inherent trade-off in validating the input of shell scripts.
On one hand one would like to catch as much of the erroneous input as possible.
On the other hand one would like to avoid cluttering the script with dozens of lines of validation code.

With simpleargs one can add declarative validation to shell scripts
which has many advantages:
* intuitive and quick to write
* doesn't bloat the script
* handles the usual use cases with ease
* is extensible and flexible for more complex validations

Consider the following snippet of code which validates the two input parameters of an imaginary script:
* `$1`: should be a non-empty string specifying an input file (that should exist)
* `$2`: should be a non-empty string specifying an output file (that shouldn't exist already)

```sh
input_file="$1"
if [ -z "${input_file}" ]; then
    echo "ERROR: No input file provided"
    exit 1
fi
if [ ! -e "${input_file}" ]; then
    echo "ERROR: No such input file '${input_file}'"
    exit 1
fi

output_file="$2"
if [ -z "${output_file}" ]; then
    echo "ERROR: No output file provided"
    exit 1
fi
if [ -e "${output_file}" ]; then
    echo "ERROR: Output file already exists: '${output_file}'"
    exit 1
fi
```
This is a very typical use case in shell scripts.
The above example is over a dozen lines long.
With simpleargs one achieves the same validation with a couple of _validation directives_.
<!--@evalsh filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/preamble/myscript -->
```sh
sa_parse "$0" "<input file>" @@file "<output file>" @@!exists
# ...
sed 's/foo/bar/' "${input_file}" > "${output_file}"
```
Output from the script is shown below.
<!--@evalonly rm -f $SIMPLEARGS_DOC_RESOURCES/validation-preamble/output.txt -->
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/validation-preamble --path $SIMPLEARGS_DOC_SCRIPTS/validation/preamble -c 'ls' -c 'cat input.txt' -c 'myscript input.txt' -c 'myscript input.txt old-output.txt' -c 'myscript inptu.txt output.txt' -c 'myscript input.txt output.txt' -c 'cat output.txt' -->
```
$ ls
input.txt
old-output.txt
$ cat input.txt
This line contains 'foo'.
$ myscript input.txt
ERROR: Missing required parameter <output file>
Usage: myscript [OPTION]... <input file> <output file>
$ myscript input.txt old-output.txt
ERROR: <output file>: File exists: 'old-output.txt'
Usage: myscript [OPTION]... <input file> <output file>
$ myscript inptu.txt output.txt
ERROR: <input file>: No such (ordinary) file: 'inptu.txt'
Usage: myscript [OPTION]... <input file> <output file>
$ myscript input.txt output.txt
$ cat output.txt
This line contains 'bar'.
```
<!--@evalonly rm -f $SIMPLEARGS_DOC_RESOURCES/validation-preamble/output.txt -->

### Resolving validation directive
The syntax for adding validation is similar to [directives](directives.md).
Instead of one `@` character _validation directives_ start with two: for example `@@file`.
Using a validation directive like `@@xyz` makes simpleargs search for a corresponding command
that implements the validation.
The command is searched in the following order:

1. A shell function (provided by simpleargs library) which is prefixed with `sa_validate_`.
So, using `@@xyz` would check if `sa_validate_xyz` command exists.
2. If not found a command with the exact same name as in the validation directive.
So, using `@@xyz` would check if command (file or shell function) `xyz` exists.

The set of standard validation directives are described in the following sections.
Additionally, it is demonstrated how to write your own command to add a custom validation directive.

### Using validation directives
Validation directives are added after an option or a parameter whose value is to be validated.
The semantics of multiple validation directives is _AND_.
That is, the value has to pass all validations to be considered valid.
<!-- TODO: add section about using @validvalues and other validations together:
     "The relationship between @validvalues and @@ validation directives -->

The syntax of a validation directive is `@@[!]directivename [arg]...`.
* The simplest validation directive consists of nothing more than two `@` characters and the directive name:
  `@@exists` checks that the value (file or directory) exists.
* Using the optional `!` before the directive name negates the validation:
  `@@!exists` checks that the value is not an existing file (or directory).
* Some directives take one or more arguments.
  For example `@@grep '^a.*z$'` checks that the first character of the value is _a_ and the last one is _z_.
* Some directives can be used with or without arguments.
  Checking that a value is an integer can be done with `@@int`.
  A valid integer range can be specified by providing an argument:
  `@@int 10..` checks that the value is an integer greater than or equal to 10.
  `@@int 100..200` checks that the value is between 100 and 200 (inclusive).

A few examples:
```sh
# Check that the value is a valid unprivileged port number (availabe to non-root users)
-p/--port arg @@int 1024..65535
```
```sh
# Check that the value is an existing directory with a name that starts with "tmp"
# but does NOT end with a dash '-'
--tmp-dir arg @@dir @@glob 'tmp*' @@!glob '*-'
```

### Combining validations with @validvalues
The three directives, [@validvalues](directives.md#validvalues),
[@validvaluesfile](directives.md#validvaluesfile), and [@validvaluescommand](directives.md#validvaluescommand)
are also used to check values passed to options and parameters.
They are implemented as "ordinary" directives (as opposed to validation directives) because they simply define a set of valid values
as opposed to having some sort of validation logic.

If an entry with validation directives is complemented with e.g. `@validvalues`
the values are taken as additional values that are accepted by that entry.
To clarify, consider an option that specifies something that can take an integer value between 0 and 100.
Additionally, there are a couple of fixed, "alias" values.
For example, the script could internally have the interpretation: `low=35`, `medium=50`, `high=95`
```sh
  --level arg @validvalues=low,medium,high @@int 0..100
```
The option accepts values `low`, `medium`, `high`, `0`, `1` `2`, ..., `100`.
The added benefit is the script's ability to autocomplete the alias values
that are possibly used frequently.
```
$ myscript --level m<tab>
$ myscript --level medium
```

To specify a couple of autocompleted values but otherwise keep the values unrestricted one can use
a dummy validation like `@glob "*"` which allows any value.
```sh
# Allows any password to be used but also provides autocompletion for clever
# chaps that don't bother wasting time on inventing strong passwords.
--password arg @validvalues=secret,passw0rd,qwerty @glob "*"
```

<details>
  <summary>Example - autocompleted favorites</summary>

This behaviour can even be used to allow a script to autocomplete the values
that have been used in the past invocations.
```sh
#!/usr/bin/env bash

clear_favorites_and_exit() {
  > ${HOME}/.favorite_operations
  exit 0
}

# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }

sa_parse "$0" \
         --clear-favorites @afterprocessing='clear_favorites_and_exit' \
         "<operation>" @validvaluesfile='${HOME}/.favorite_operations' \
         @@grep '^[a-z][a-z]*$'
sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
# ----------------------------------------------------------------------------

echo "${operation}" >> ${HOME}/.favorite_operations
# Execute ${operation}
```

If the operation given as input is valid (consists of one or more lowercase letters)
it is added to a favorites file.
The favorites file is used as the source that provides the autocompleted values.
The `--clear-favorites` option can be used to reset the favorites by truncating the favorites file
to zero size.
The option is using [@afterprocessing](directives.md#afterprocessing) directive to allow the script
to be invoked with simple
```
$ myscript --clear-favorites
```
This invocation would not work if the clearing of favorites was done in the user part of the script:
the script would complain that it's missing a required parameter `operation`.
</details>


### Standard (library) validation directives
Simpleargs comes with a set of built-in validation directives.
They are implemented as shell functions with `sa_validate_` prefix.
This way for example `@@grep` will not try to invoke grep directly but the corresponding validation function `sa_validate_grep`.
This allows for example [the customisation of the error messages](#customizing-the-error-message).

Regular expression and glob validations:

* [@@grep](#grep)
* [@@egrep](#egrep)
* [@@glob](#glob)

Bash [test](http://wiki.bash-hackers.org/commands/classictest) based validations:

* [@@exists](#exists)
* [@@file](#file)
* [@@dir](#dir)
* [@@notempty](#notempty)
* [@@readable](#readable)
* [@@writable](#writable)
* [@@executable](#executable)
* [@@modified](#modified)

Other:

* [@@filetype](#filetype)
* [@@int](#int)
* [@@float](#float)

#### @@grep
**Description**  
Matches the value against a grep regular expression.

**Arguments**  
1. grep _basic_ regular expression (see `man grep` for details)

**Notes**  
Executes `grep -q -e "${pattern}" <<< "${value}"`

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/grep/myscript -->
```
sa_parse "$0" --name arg @@grep '^[A-Z][a-z][a-z]*$'
# ...
echo "Hello, ${name}!"
```
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/grep -c 'myscript --name james' -c 'myscript --name J' -c 'myscript --name James' -->
```
$ myscript --name james
ERROR: --name: 'james' does not match regular expression '^[A-Z][a-z][a-z]*$'
Usage: myscript [OPTION]...
$ myscript --name J
ERROR: --name: 'J' does not match regular expression '^[A-Z][a-z][a-z]*$'
Usage: myscript [OPTION]...
$ myscript --name James
Hello, James!
```

</details>

#### @@egrep
**Description**  
Matches the value against a grep _extended_ regular expression.

**Arguments**  
1. grep _extended_ regular expression (see `man grep` for details)

**Notes**  
Executes `grep -qE -e "${pattern}" <<< "${value}"`

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/egrep/myscript -->
```
sa_parse "$0" --j-name arg @@egrep '^J[a-z]+'
# ...
echo "Hello, ${j_name}!"
```
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/egrep -c 'myscript --j-name Patty' -c 'myscript --j-name J' -c 'myscript --j-name Jerry' -->
```
$ myscript --j-name Patty
ERROR: --j-name: 'Patty' does not match extended regular expression '^J[a-z]+'
Usage: myscript [OPTION]...
$ myscript --j-name J
ERROR: --j-name: 'J' does not match extended regular expression '^J[a-z]+'
Usage: myscript [OPTION]...
$ myscript --j-name Jerry
Hello, Jerry!
```

</details>

#### @@glob
**Description**  
Matches the value against a Bash extended glob.
Since the primary purpose of this directive is to check filenames
the validation is applied to the [basename](https://en.wikipedia.org/wiki/Basename) of the value i.e.
the part of the value after the last slash `/`.
Hence, input `/tmp/dir/data-3.txt` will match glob `data*.txt` (since `data-3.txt` matches the glob).

**Arguments**  
1. Bash extended glob (see [Bash Pattern Matching](https://tiswww.case.edu/php/chet/bash/bashref.html#Pattern-Matching))

**Notes**  
Executes `[[ "${file_basename}" = ${extended_glob} ]]`
For consistency the evaluation is done with `LC_COLLATE=C` which affects how character ranges are interpreted.
With many locales `[a-z]` is taken as `[aAbBcC...xXyYz]`].
Using `LC_COLLATE=C` makes `[a-z]` the set of lowercase letters.
It is also possible to use the predefined character classes such as `[:lower:]`.
See [Bash Pattern Matching](https://tiswww.case.edu/php/chet/bash/bashref.html#Pattern-Matching) and
[this Stack Exchange question](https://unix.stackexchange.com/questions/227070/why-does-a-z-match-lowercase-letters-in-bash)
for more information.

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/glob/myscript -->
```
sa_parse "$0" \
         "<text file>" @@glob "*.txt" \
         "<image file>" @@glob "*.@(png|jpg|gif|tiff)" \
         "<uppercase start>" @@glob "[A-Z]*" \
         "<no z end>" @@!glob "*z"
# ...
echo "All parameters OK"
```
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/glob -c 'myscript data.txt image.png Data.bin end.with.x' -c 'myscript /tmp/txt/data images/flower.jpeg dat4012 archive.tar.gz' -->
```
$ myscript data.txt image.png Data.bin end.with.x
All parameters OK
$ myscript /tmp/txt/data images/flower.jpeg dat4012 archive.tar.gz
ERROR: <text file>: 'data' does not match glob '*.txt'
ERROR: <image file>: 'flower.jpeg' does not match glob '*.@(png|jpg|gif|tiff)'
ERROR: <uppercase start>: 'dat4012' does not match glob '[A-Z]*'
ERROR: <no z end>: 'archive.tar.gz' matches glob '*z'
Usage: myscript [OPTION]... <text file> <image file> <uppercase start>
                <no z end>
```

</details>

#### @@exists
**Description**  
Checks that a file exists using Bash test operator: `[ -e "${value}" ]` (see `help test` for details)

**Arguments**  
No arguments.

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/exists/myscript -->
```
sa_parse "$0" "<input file>" @@exists "<output file>" @@!exists
```
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/file-tests --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/exists -c 'ls -l' -c 'myscript raedable.txt output.txt' -c 'myscript readable.txt empty.txt' -c 'myscript readable.txt output.txt' -->
```
$ ls -l
total 12
drwxr-xr-x 3 lval staff 96 Jan  1  1970 directory
-rw-r--r-- 1 lval staff  0 Jan  1  1970 empty.txt
-rwxr-xr-x 1 lval staff 39 Jan  1  1970 executable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 readable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 writable.txt
$ myscript raedable.txt output.txt
ERROR: <input file>: No such file: 'raedable.txt'
Usage: myscript [OPTION]... <input file> <output file>
$ myscript readable.txt empty.txt
ERROR: <output file>: File exists: 'empty.txt'
Usage: myscript [OPTION]... <input file> <output file>
$ myscript readable.txt output.txt
```

</details>

#### @@file
**Description**  
Checks that a _regular_ file exists using Bash test operator: `[ -f "${value}" ]` (see `help test` for details)

**Arguments**  
No arguments.

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/file/myscript -->
```
sa_parse "$0" "<input file>" @@file
```
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/file-tests --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/file -c 'ls -l' -c 'myscript non-existing.txt' -c 'myscript directory' -c 'myscript readable.txt' -->
```
$ ls -l
total 12
drwxr-xr-x 3 lval staff 96 Jan  1  1970 directory
-rw-r--r-- 1 lval staff  0 Jan  1  1970 empty.txt
-rwxr-xr-x 1 lval staff 39 Jan  1  1970 executable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 readable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 writable.txt
$ myscript non-existing.txt
ERROR: <input file>: No such (ordinary) file: 'non-existing.txt'
Usage: myscript [OPTION]... <input file>
$ myscript directory
ERROR: <input file>: No such (ordinary) file: 'directory'
Usage: myscript [OPTION]... <input file>
$ myscript readable.txt
```

</details>

#### @@dir
**Description**  
Checks that a directory exists using Bash test operator: `[ -d "${value}" ]` (see `help test` for details)

**Arguments**  
No arguments.

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/dir/myscript -->
```
sa_parse "$0" "<data dir>" @@dir
```
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/file-tests --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/dir -c 'ls -l' -c 'myscript non-existing.txt' -c 'myscript readable.txt' -c 'myscript directory' -->
```
$ ls -l
total 12
drwxr-xr-x 3 lval staff 96 Jan  1  1970 directory
-rw-r--r-- 1 lval staff  0 Jan  1  1970 empty.txt
-rwxr-xr-x 1 lval staff 39 Jan  1  1970 executable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 readable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 writable.txt
$ myscript non-existing.txt
ERROR: <data dir>: No such directory: 'non-existing.txt'
Usage: myscript [OPTION]... <data dir>
$ myscript readable.txt
ERROR: <data dir>: No such directory: 'readable.txt'
Usage: myscript [OPTION]... <data dir>
$ myscript directory
```

</details>

#### @@notempty
**Description**  
Checks that a file exists and is not empty using Bash test operator: `[ -s "${value}" ]` (see `help test` for details).
Note that you cannot check for existing and non-empty _directories_ with this.

**Arguments**  
No arguments.

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/notempty/myscript -->
```
sa_parse "$0" "<input file>" @@notempty
```
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/file-tests --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/notempty -c 'ls -l' -c 'myscript non-existing.txt' -c 'myscript empty.txt' -c 'myscript readable.txt' -->
```
$ ls -l
total 12
drwxr-xr-x 3 lval staff 96 Jan  1  1970 directory
-rw-r--r-- 1 lval staff  0 Jan  1  1970 empty.txt
-rwxr-xr-x 1 lval staff 39 Jan  1  1970 executable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 readable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 writable.txt
$ myscript non-existing.txt
ERROR: <input file>: No such file or not empty: 'non-existing.txt'
Usage: myscript [OPTION]... <input file>
$ myscript empty.txt
ERROR: <input file>: No such file or not empty: 'empty.txt'
Usage: myscript [OPTION]... <input file>
$ myscript readable.txt
```

</details>

#### @@readable
**Description**  
Checks that a file (or directory) is readable using Bash test operator: `[ -r "${value}" ]` (see `help test` for details)

**Arguments**  
No arguments.

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/readable/myscript -->
```
sa_parse "$0" "<input file>" @@readable
```
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/file-tests --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/readable -c 'ls -l' -c 'myscript non-existing.txt' -c 'myscript readable.txt' -->
```
$ ls -l
total 12
drwxr-xr-x 3 lval staff 96 Jan  1  1970 directory
-rw-r--r-- 1 lval staff  0 Jan  1  1970 empty.txt
-rwxr-xr-x 1 lval staff 39 Jan  1  1970 executable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 readable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 writable.txt
$ myscript non-existing.txt
ERROR: <input file>: No such file or not readable: 'non-existing.txt'
Usage: myscript [OPTION]... <input file>
$ myscript readable.txt
```

</details>

#### @@writable
**Description**  
Checks that a file (or directory) is writable using Bash test operator: `[ -w "${value}" ]` (see `help test` for details)

**Arguments**  
No arguments.

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/writable/myscript -->
```
sa_parse "$0" "<output file>" @@writable
```
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/file-tests --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/writable -c 'ls -l' -c 'myscript non-existing.txt' -c 'myscript readable.txt' -c 'myscript writable.txt' -->
```
$ ls -l
total 12
drwxr-xr-x 3 lval staff 96 Jan  1  1970 directory
-rw-r--r-- 1 lval staff  0 Jan  1  1970 empty.txt
-rwxr-xr-x 1 lval staff 39 Jan  1  1970 executable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 readable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 writable.txt
$ myscript non-existing.txt
ERROR: <output file>: No such file or not writable: 'non-existing.txt'
Usage: myscript [OPTION]... <output file>
$ myscript readable.txt
$ myscript writable.txt
```

</details>

#### @@executable
**Description**  
Checks that a file is executable using Bash test operator: `[ -x "${value}" ]` (see `help test` for details)

**Arguments**  
No arguments.

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/executable/myscript -->
```
sa_parse "$0" "<executable file>" @@executable
```
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/file-tests --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/executable -c 'ls -l' -c 'myscript non-existing.txt' -c 'myscript readable.txt' -c 'myscript executable.txt' -->
```
$ ls -l
total 12
drwxr-xr-x 3 lval staff 96 Jan  1  1970 directory
-rw-r--r-- 1 lval staff  0 Jan  1  1970 empty.txt
-rwxr-xr-x 1 lval staff 39 Jan  1  1970 executable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 readable.txt
-rw-r--r-- 1 lval staff  9 Jan  1  1970 writable.txt
$ myscript non-existing.txt
ERROR: <executable file>: No such file or not executable: 'non-existing.txt'
Usage: myscript [OPTION]... <executable file>
$ myscript readable.txt
ERROR: <executable file>: No such file or not executable: 'readable.txt'
Usage: myscript [OPTION]... <executable file>
$ myscript executable.txt
```

</details>

#### @@modified
**Description**  
Checks that a file has been modified since it was last read using Bash test operator: `[ -N "${value}" ]` (see `help test` for details)

**Arguments**  
No arguments.

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/modified/myscript -->
```
sa_parse "$0" "<input file>" @@modified
```
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/file-tests --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/modified -c 'ls -l writable.txt' -c 'touch -a writable.txt' -c 'myscript writable.txt' -c 'touch -m writable.txt' -c 'myscript writable.txt' -->
```
$ ls -l writable.txt
-rw-r--r-- 1 lval staff 9 Jan  1  1970 writable.txt
$ touch -a writable.txt
$ myscript writable.txt
ERROR: <input file>: File not modified since last read: 'writable.txt'
Usage: myscript [OPTION]... <input file>
$ touch -m writable.txt
$ myscript writable.txt
```

</details>

<!--@evalonly touch -d '1 January 1970' $SIMPLEARGS_DOC_RESOURCES/file-tests/* -->

<!-- Not sure if @@filetype should be part of the library.
     It is possibly cumbersome to use.
     It adds a dependency to 'file' command.
     It might not be used very often.

#### @@filetype
**Description**  
Checks the type of file using `file` command.
One can use `@@grep` and `@@glob` to validate that an input file is of correct type.
However, relying on the filename i.e. checking that it ends with `.jpg` is inherently error-prone.
Not all JPEG files have `.jpg` suffix (some might have `.JPG` or `.jpeg`).
`@@filetype` uses `file` command to extract and utilize the actual type of file.

* Checking that an input file is a JPEG file: `@@filetype "JPEG image data"`.
* Checking that an input file is one of the image types: `@@filetype "(JPEG|PNG|GIF) image data"`
  Note that using this directive comes with certain pitfalls. // TODO: fix indentation
  For example, checking for text files is not as simple as one would first think.
* `@@filetype "ASCII text"` would miss other character encodings (most notably UTF-8).
* `@@filetype "text"` would cover other text files as well but miss empty files.
  Depending on the scenario a script might need to accept an empty file as a `text file`.
* `@@filetype "text|empty"` would probably work in most scenarios.

**Arguments**  
1. Bash extended regular expression
   (see [Bash Pattern Matching](https://tiswww.case.edu/php/chet/bash/bashref.html#Pattern-Matching))
   to be matched against output of `file` command.

**Notes**  
Executes `[[ "$(file --brief --dereference "${value}" 2>/dev/null)" =~ ${extended_regex} ]]`

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/filetype/scriptedit -->
```
sa_parse "$0" "<shell script>" @@filetype "ASCII text executable|empty"
```
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/filetype --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/filetype -c "file *" -c 'scriptedit text.txt' -c 'scriptedit data.xml' -c 'scriptedit conv.sh' -c 'scriptedit empty-file' -->
```
$ file *
conv.sh:    Bourne-Again shell script text executable, ASCII text
data.xml:   XML 1.0 document text, ASCII text
empty-file: empty
text.txt:   ASCII text
$ scriptedit text.txt
ERROR: <shell script>: 'text.txt' is not of type 'ASCII text executable|empty' (
but 'ASCII text')
Usage: scriptedit [OPTION]... <shell script>
$ scriptedit data.xml
ERROR: <shell script>: 'data.xml' is not of type 'ASCII text executable|empty' (
but 'XML 1.0 document text, ASCII text')
Usage: scriptedit [OPTION]... <shell script>
$ scriptedit conv.sh
ERROR: <shell script>: 'conv.sh' is not of type 'ASCII text executable|empty' (b
ut 'Bourne-Again shell script text executable, ASCII text')
Usage: scriptedit [OPTION]... <shell script>
$ scriptedit empty-file
```

</details>

-->

#### @@int
**Description**  
Checks that a value is a valid integer and optionally that the integer is in the given range.
Bash integer equality test (`[ "${value}" -eq "${value}" ]`) is used to check the validity of the value.
Hence, examples of valid integers: 2, +5, -8, 0, -0, +0.

**Arguments**  
1. (Optional) integer range (inclusive) given as `[min]..[max]`.
Minimum or maximum value can be omitted from the range.
Omitting both bounds is the same as omitting the range altogether.
Examples of integer ranges: `1..5`, `0..`, `-8..+3`, `..1000`.
See the example script below for more details.

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/int/myscript -->
```
sa_parse "$0" \
         --int arg @@int @multivalue, \
         --non-negative-int arg @@int 0.. @multivalue, \
         --int-under-ten arg @@int ..9 @multivalue, \
         --positive-int-under-ten arg @@int 1..9 @multivalue, \
         --percent-value arg @@int 0..100 @multivalue, \
         --negative-two-digit arg "@@int -99..-10" @multivalue,
```
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/int -c 'myscript --int 12,-8,0,ab,-9' -c 'myscript --non-negative 1,+4,-3,84' -c 'myscript --int-under-ten 2,4,5,-4' -c 'myscript --positive-int-under-ten 2,4,5,-4' -c 'myscript --percent-value 12,0,100,45,101' -c 'myscript --negative-two-digit -23,-5,-19,-99' -->
```
$ myscript --int 12,-8,0,ab,-9
ERROR: --int: Invalid integer value: 'ab'
Usage: myscript [OPTION]...
$ myscript --non-negative 1,+4,-3,84
ERROR: --non-negative-int: Value '-3' less than 0
Usage: myscript [OPTION]...
$ myscript --int-under-ten 2,4,5,-4
$ myscript --positive-int-under-ten 2,4,5,-4
ERROR: --positive-int-under-ten: Value '-4' not in range: 1..9
Usage: myscript [OPTION]...
$ myscript --percent-value 12,0,100,45,101
ERROR: --percent-value: Value '101' not in range: 0..100
Usage: myscript [OPTION]...
$ myscript --negative-two-digit -23,-5,-19,-99
ERROR: --negative-two-digit: Value '-5' not in range: -99..-10
Usage: myscript [OPTION]...
```

</details>

**WARNING**  
Range with a negative minimum value cannot be specified as `--int arg @@int -12..`
since it is misinterpreted as a flag definition (starts with a dash).
Solution: use quotes to create a single word: `--int arg "@@int -12.."`

#### @@float
**Description**  
Checks that a value is a valid floating point number and optionally that the value is in the given range.
An extended (grep) regular expression `^[-+]?[0-9]*\.?[0-9]+([eE][-+]?[0-9]+)?$` is used to verify a valid floating point number.
Hence, examples of valid floats: `0`, `+1`, `-12.3`, `2E8`, `-119e-2`, `+3E-2` (scientific notation is accepted as well).

**Arguments**  
1. (Optional) float range (inclusive) given as `[min]..[max]`.
Minimum or maximum value can be omitted from the range.
Omitting both bounds is the same as omitting the range altogether.
Examples of integer ranges: `0..10E1`, `-1.8..12.3`, `-8e2..`, `12E-2..`.
See the example script below for more details.

<details>
  <summary>Example</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/validation/directives/float/myscript -->
```
sa_parse "$0" \
         --float arg @@float @multivalue, \
         --non-negative-float arg @@float 0.0.. @multivalue, \
         --float-not-over-million arg @@float ..1E6 @multivalue, \
         --positive arg @@float 0.. @@!float ..0 @multivalue,
```
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/validation/directives/float -c 'myscript --float 2.53e-1,-0,13,18.5A3' -c 'myscript --non-negative-float 0,0.0,1E-6,18' -c 'myscript --float-not-over-million 999999,1e6,' -c 'myscript --positive 12,0.15,0,-5' -->
```
$ myscript --float 2.53e-1,-0,13,18.5A3
ERROR: --float: Invalid float value: '18.5A3'
Usage: myscript [OPTION]...
$ myscript --non-negative-float 0,0.0,1E-6,18
$ myscript --float-not-over-million 999999,1e6,
$ myscript --positive 12,0.15,0,-5
ERROR: --positive: Value '-5' less than 0
ERROR: --positive: Value '0' should not be less than or equal to 0
ERROR: --positive: Value '-5' should not be less than or equal to 0
Usage: myscript [OPTION]...
```

</details>

### Writing a custom validation directive
As stated earlier each validation directive is backed by a command.
Hence, one can create a custom validation directive by simply creating a script or a shell function
that implements the interface described below.

#### Validation command interface
* The command takes at least one parameter, namely the value to be validated.
For example the library validation command `sa_validate_exists` (corresponding to `@@exists`) takes only one parameter.
When a script with definition
```sh
sa_parse "$0" "<input>" @@exists
```
is invoked with `myscript input.txt` the validation framework will execute
```
sa_validate_exists 'input.txt'
```
If there's no command `sa_validate_exists` the library tries to execute command named `exists`.

* If the command takes other parameters they come before the value to be validated.
That is, when a script with definition
```sh
sa_parse "$0" "<input>" @@glob "*.txt"
```
is invoked using `myscript input.txt` the library will execute
```
sa_validate_glob '*.txt' 'input.txt'
```
This implies that a validation command that has optional parameters needs to know
how to handle invocations with different number of parameters.

* The return value should be `0` if the value passes the validation, non-zero otherwise.
  However, note the special exit values described below.
* The command should not print anything to stdout or stderr (unless the function is used incorrectly).
* The command should not have any side effects since it may be run multiple times and also as part of command completion routines.
  For example, local variables should be used (in shell functions) in order not to litter the shell environment.
* The command should preferably run fairly quickly and have no external dependencies e.g. other commands
  or shell functions or network access that might not be available when the script is run.
* If the command is used incorrectly (for example wrong number of parameters) it should return exit status `${SA_INCORRECT_USE}`
  (possibly after printing an error message).

By default a validation failure generates a generic error message. For example,
```
$ myscript input.txt
ERROR: <input file>: value 'input.txt' failed validation for 'my-validation-command'
```
The error message can be customized by making the validation command conform to additional requirements:

* If either environment variable `sa_gen_normal_error_msg` or `sa_gen_negated_error_msg` is `true`
assign a custom error message to variable `sa_error_msg` and return with status `${SA_GENERATED_ERROR_MSG_STATUS}`.

The aspects of validation command interface described above can be seen in
`sa_validate_exists` function below (part of the simpleargs library).
The function corresponds to using `@@exists` validation directive.
Note the custom error message generation and especially how an error message is assigned
for the negated case i.e. when one has used `!@@exists` (file should not exist).
```sh
sa_validate_exists() {
    if [ $# -ne 1 ]
    then
        echo "${FUNCNAME}: incorrect number ($#) of parameters (should be 1)" >&2
        return "${SA_INCORRECT_USE}"
    fi

    local filename="$1"
    if [ "${sa_gen_normal_error_msg}" = "true" ]; then
      sa_error_msg="No such file: '${filename}'"
      return "${SA_GENERATED_ERROR_MSG_STATUS}"
    fi
    if [ "${sa_gen_negated_error_msg}" = "true" ]; then
      sa_error_msg="File exists: '${filename}'"
      return "${SA_GENERATED_ERROR_MSG_STATUS}"
    fi
    [ -e "$1" ]
}
```

#### Example validation directive
Let's demonstrate this with an example.
To introduce a validation directive `@@capitalized` that checks that a value starts
(or in the negated case does not start) with a capital letter
one can define a function before the simpleargs preamble:
<!--@eval filter-simpleargs-script -r 1 $SIMPLEARGS_DOC_SCRIPTS/validation/custom/simple-capitalized/myscript -->
```
#!/usr/bin/env bash

capitalized() {
    if [ "$#" -ne 1 ]
    then
        echo "${FUNCNAME}: incorrect number ($#) of parameters (should be 1)" >&2
        return "${SA_INCORRECT_USE}"
    fi
    local value="$1"
    grep -q -e "^[A-Z]" <<< "${value}"
}

# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" \
         --capitalized arg @@capitalized \
         --non-capitalized arg '@@!capitalized'

sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
# ----------------------------------------------------------------------------
echo "    Capitalized: ${capitalized}"
echo "Non-capitalized: ${non_capitalized}"
```
The validation command can be implemented as a shell function, a shell script or even a compiled binary.
Embedding a shell function inside the script file (as in the example above) has the advantage
of not introducing a dependency to the script.
However, the shell function cannot be reused in other scripts and it's not available for command completion routines.
A separate shell script or binary has its pros and cons the other way around.

In the example above one can see how the validation directive is used
in its normal form (`@@capitalized`) and negated form (`@@!capitalized`).
The error messages state simply that the validation failed for the given directive:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/validation/custom/simple-capitalized -c 'myscript --capitalized jo' -c 'myscript --non-capitalized Tim' -c 'myscript --capitalized Jo --non-capitalized tim' -->
```
$ myscript --capitalized jo
ERROR: --capitalized: value 'jo' failed validation for 'capitalized'
Usage: myscript [OPTION]...
$ myscript --non-capitalized Tim
ERROR: --non-capitalized: value 'Tim' failed validation for (not) 'capitalized'
Usage: myscript [OPTION]...
$ myscript --capitalized Jo --non-capitalized tim
    Capitalized: Jo
Non-capitalized: tim
```

#### Customizing the error message
To provide the user with a more helpful error message the validation command should check whether
it is invoked with a special environment variable set to `true`:

* Environment variable `${sa_gen_normal_error_message} = "true"` signals
  that validation has failed for the normal form
  of the directive (e.g. `@@capitalized`).
  That is, the value was _not_ capitalized.
  Hence, the command should set variable _sa_error_msg_ to a meaningful error message.

* Environment variable `${sa_gen_negated_error_msg} = "true"` signals
  that validation has failed for the _negated_ form
  of the directive (e.g. `@@!capitalized`).
  That is, the value was capitalized _but it should **not** be_.
  Again, `sa_error_msg` should be set accordingly (stating that the value should not start with a capital letter).
  <!--@eval filter-simpleargs-script -r 1 $SIMPLEARGS_DOC_SCRIPTS/validation/custom/custom-message-capitalized/myscript -->
  ```
  #!/usr/bin/env bash

  capitalized() {
      if [ "$#" -ne 1 ]
      then
          echo "${FUNCNAME}: incorrect number ($#) of parameters (should be 1)" >&2
          return "${SA_INCORRECT_USE}"
      fi
      local value="$1"

      if [ "${sa_gen_normal_error_msg}" = "true" ]
      then
         sa_error_msg="Value should start with a capital letter: '${value}'"
         return "${SA_GENERATED_ERROR_MSG_STATUS}"
      fi
      if [ "${sa_gen_negated_error_msg}" = "true" ]
      then
          sa_error_msg="Value should not start with a capital letter: '${value}'"
          return "${SA_GENERATED_ERROR_MSG_STATUS}"
      fi

      grep -q -e "^[A-Z]" <<< "${value}"
  }

  # -------------------------------- simpleargs --------------------------------
  . "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
  sa_parse "$0" \
           --capitalized arg @@capitalized \
           --non-capitalized arg '@@!capitalized'

  sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
  # ----------------------------------------------------------------------------
  echo "    Capitalized: ${capitalized}"
  echo "Non-capitalized: ${non_capitalized}"
  ```

Now the user gets more meaningful guidance on how to fix their invocation:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/validation/custom/custom-message-capitalized -c 'myscript --capitalized jo' -c 'myscript --non-capitalized Tim' -c 'myscript --capitalized Jo --non-capitalized tim' -->
```
$ myscript --capitalized jo
ERROR: --capitalized: Value should start with a capital letter: 'jo'
Usage: myscript [OPTION]...
$ myscript --non-capitalized Tim
ERROR: --non-capitalized: Value should not start with a capital letter: 'Tim'
Usage: myscript [OPTION]...
$ myscript --capitalized Jo --non-capitalized tim
    Capitalized: Jo
Non-capitalized: tim
```
