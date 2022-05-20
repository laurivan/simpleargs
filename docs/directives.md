<!-- GENERATED TOC -->
<!-- TOC START -->
## Table of Contents

  - [Using @ directives](#using-directives)
  - [List of directives](#list-of-directives)
    - [@default](#default)
    - [@optionalvalue](#optionalvalue)
    - [@varname](#varname)
    - [@onvalue and @offvalue](#onvalue-and-offvalue)
    - [@allowrepeat](#allowrepeat)
    - [@doc](#doc)
    - [@allowempty](#allowempty)
    - [@multivalue (list)](#multivalue-list)
    - [@validvalues](#validvalues)
    - [@validvaluesfile](#validvaluesfile)
    - [@validvaluescommand](#validvaluescommand)
    - [@afterprocessing](#afterprocessing)
    - [@required](#required)
    - [@expand](#expand)
<!-- TOC END -->

## Using @ directives
A lot of the functionality seen so far can be achieved with some effort using plain getopt(s).

In an earlier section about [optional argument options](options-and-parameters.md#optional-argument-option)
it was shown how to specify that the argument of an option is _optional_:
<!--@eval filter-simpleargs-script -r '/^sa_parse/' -r e// $SIMPLEARGS_DOC_SCRIPTS/optional-argument-option/myscript -->
```
sa_parse "$0" -i/--in-place arg @optionalvalue
```
In simpleargs attaching additional information to an option or a parameter is done using _directives_ aka _annotations_.
A directive affects the option or parameter that _precedes_ the directive.
In other words, the directives of an option or parameter follow that particular option or parameter.

The syntax of a directive is `@<name>[<modifier>][=<value>]`.
* The simplest directive consists of nothing more than `@` character and the directive name: `@optionalvalue`
* Some directives have a value which is separated from the directive name with `=` character: `@default=high`

Some directives have an optional modifier which is exactly one character long.
The modifier affects how the directive is interpreted.
The semantics of the modifier depend on the directive.
At the moment a modifier is allowed for `@validvalues` and `@multivalue`.
* For `@validvalue` the modifier specifies a separator character (which is comma `,` by default).
  So, the three directives below are equivalent.
  ```sh
  @validvalues=a,b,c
  @validvalues,=a,b,c
  @validvalues:=a:b:c
  ```
  They specify that the valid values of an option or a parameter are `a`, `b`, and `c`.
* `@multivalue` without a modifier specifies that an option (that takes a value) can have multiple values.
  In other words, the option can be specified many times on the command line: `myscript -n one -n two`
  If a modifier, for example comma, is used `@multivalue,` the values can be written _additionally_
  as a comma separated list: `myscript -e one -e two -e three,four,five

See [@validvalues](#validvalues) and [@multivalue](#multivalue) for more information.

Note that sometimes it is required to quote the value to prevent shell expansions or word splitting.
For example, `@doc` practically always needs quotes:
```sh
--user arg @doc="User name for the connection."
```
Otherwise, the value for the directive would be `user` and the rest of the words
would be interpreted as subsequent tokens (raising an error).
The examples in the following sections provide more insights into when the quotes are needed
and when it matters whether single or double quotes are used.

> **_NOTE_**  
Many directives, for example `@doc`, can be applied to both options and parameters.
To simplify things the text adopts a new term:
(command line) _entry_ which means either _option or parameter_.
Hence, instructing the user to `apply a directive to an entry` is to be read `apply a directive to an option or a parameter`.

## List of directives
### @default
`@default` is used to provide an entry a default value.
It can be used with
* [required argument options](options-and-parameters.md#required-argument-options)
* [optional argument options](options-and-parameters.md#optional-argument-options)
* [optional parameters](options-and-parameters.md#optional-parameters)
* [varargs optional parameters](options-and-parameters.md#varargs-optional-parameters)

The syntax is `@default=<value>`.
If the value contains whitespace it needs to be quoted: `@default="example value"`
If the script invocation doesn't specify an option or an optional parameter
the default value is assigned to the corresponding variable.

<details>
  <summary>Example</summary>

<!--@evalsh filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/default/myscript -->
```sh
sa_parse "$0" --user arg @default=root \
         "[<output>]" @default=stdout \
         "[<input file>]..." @default=-
# ...
echo "user: '${user}'"
echo "output: '${output}'"
echo "inputs:"
for i in "${input_file[@]}"
do
    echo "  - input: '${i}'"
done
```
When invoked without specifying options or parameters the default values are used.
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/default -c 'myscript' -c 'myscript --user john file:/tmp/test.txt a.txt b.txt c.txt' -->
```
$ myscript
user: 'root'
output: 'stdout'
inputs:
  - input: '-'
$ myscript --user john file:/tmp/test.txt a.txt b.txt c.txt
user: 'john'
output: 'file:/tmp/test.txt'
inputs:
  - input: 'a.txt'
  - input: 'b.txt'
  - input: 'c.txt'
```
Note that since the input files of the script are defined as
an optional varargs parameter the default value is assigned into the first index of an array.

At the moment only a single value can be set as the default of an optional _varargs_ parameter.
For multiple default values one has to do something like
```sh
if [ ${#input_file[*]} -eq 0 ]; then
  echo "No input files provided: using the defaults"
  input_file=( this.txt that.txt )
fi
```

</details>

To specify a dynamic default value one can either
* use an existing environment variable
  ```sh
  --log-dir arg @default='${HOME}'
  ```
* or export a new environment variable before the simpleargs header
  ```sh
  export DEFAULT_LOG_DIR=$(get_log_dir)
  ...
  --log-dir arg @default='${DEFAULT_LOG_DIR}'
  ```

This works because the default is processed using `envsubst` which expands any _exported_
environment variables in the value.
Also, the value for `@default` directive is enclosed in single quotes to prevent
the variable from being expanded "too early".
Since the (default) value is cached upon the first execution of the script
the unexpanded string needs to be stored - not what is expanded on the first execution.

Bear in mind that one cannot use Bash special variables like `$$` (PID of current shell)
or `~` (user's home directory) directly.
This can be worked around by creating a new environment variable.
```sh
export CURRENT_SHELL_PID=$$
...
--process-name arg @default='${CURRENT_SHELL_PID}'
```

### @optionalvalue
`@optionalvalue` specifies that an option takes
[an _optional_ argument](options-and-parameters#optional-argument-option).
In other words, the option can be specified with or without a value:

<details>
  <summary>Example</summary>
  <div style="border-style:solid;border-color:red">

The following script has an option for specifying whether to sort script output:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/optionalvalue/myscript -->
```
sa_parse "$0" -s/--sort sort_criteria @optionalvalue @default=none
# ...
if [ "${sort_criteria}" = "none" ]; then
    echo "no sorting"
elif [ "${sort_criteria}" = "" ] ||
     [ "${sort_criteria}" = "alphabetical" ]; then
    echo "sorting alphabetically"
elif [ "${sort_criteria}" = "time" ]; then
    echo "sorting by time"
fi
```
`@optionalvalue` is normally useful to use together with [@default](#default).
In the above example there is no sorting by default.
When `--sort` is used without a value the sorting criteria is alphabetical.
This construct is handy if the majority of invocations that require sorting use the same sorting criteria.
Example invocations with explanations are below
```sh
$ myscript                     # 1. no sorting (the default)
$ myscript --sort              # 2. sort alphabetically, ${sort_criteria} == ""
$ myscript --sort=""           # 3. sort alphabetically (equivalent with the above)
$ myscript --sort=alphabetical # 4. sort alphabetically (more tedious to write)
$ myscript --sort=time         # 5. sort by time
$ myscript -stime              # 6. sort by time (uses short flag)
$ myscript --sort time         # 7. WRONG 'time' is the first positional parameter
$ myscript -s time             # 8. WRONG 'time' is the first positional parameter
```

Without `@optionalvalue` directive the second invocation would be erroneous
(option missing its argument).
Note also the syntax when using the short flag (6th line): the flag and the value have no space(s) between them.
When using the long flag the equals sign `=` is mandatory.

The last two invocations illustrate the pitfall of having space between the flag and its (optional) value.
Is the value after `-s/--sort` the option's argument or a positional parameter?
The invocations are syntactically correct but they are actually sorting alphabetically
since the option is considered to have no value and `time` to be the first positional parameter.

> **_NOTE_**  
The `--<flag>=<value>` syntax is allowed also with required value options
in addition to the more common `--<flag> <value>`.

  </div>
</details>

### @varname
`@varname` is used to explicitly specify the variable name which holds the option's or parameter's value.
```sh
-u/--user arg @varname=database_user
```
Sections about [options](options-and-parameters.md#defining-script-options) and
[parameters](options-and-parameters.md#defining-script-parameters)
already covered how variable names are derived from the option and parameter definitions:
```
-n/--no-cache     # --> ${n} and ${no_cache} (derived from flags)
-u/user username  # --> ${username} (implicit variable name)
-u/user arg       # --> ${u} and ${user} ('arg' is a keyword)
"<output file>"   # --> ${ouput_file} (derived from definition)
"<input>..."      # --> ${input[0]}, ${input[1]}, ... (array)
```
This default behavior can be overridden with `@varname` in which case variable names
derived from the entry definitions are not used.

<details>
  <summary>Example</summary>

In some scenarios it would be useful to specify the variable name of an entry (option or parameter) explicitly.
For example, one would like to specify a _no argument_ option with only a short flag `-e` that turns on password encryption:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/varname/problematic/myscript -->
```
sa_parse "$0" -e
# ...
if ${e}
then
  # ...encrypt password"
fi
```
This works but one letter variable names will reduce the readability of the script. One cannot write the definition as
```
-e encrypt_password
```
since this would define a required argument option.
The solution is to specify the variable name explicitly:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/varname/solved/myscript -->
```
sa_parse "$0" -e @varname=encrypt_password
if ${encrypt_password} # Note: "${e}" is left unassigned
then
  # ...encrypt password"
fi
```

</details>

<details>
  <summary>Example</summary>

Consider a script with multiple input files.
<!--@evalsh filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/varname/input-files -->
```sh
sa_parse "$0" "<input file>..."
# ...
echo "inputs:"
for i in "${input_file[@]}"
do
    echo "  ${i}"
done
```
Using the parameter `<input file>...` in the script is a bit counterintuitive.
That is, in the for-loop one might want to use `input_file` as the loop variable
but the variable name is already reserved for the array.
One could define the parameter as `<input files>...`.
However, later we will see that the string in the parameter definition `input file`
is used in the generated documentation where the singular form is preferred.

The solution is to use `@varname`.
This allows using a more descriptive variable name in the loop.
<!--@evalsh filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/varname/input-files-improved -->
```sh
sa_parse "$0" "<input file>..." @varname=input_files
# ...
echo "inputs:"
for input_file in "${input_files[@]}"
do
    echo "  ${input_file}"
done
```

Note that using `@varname` overrides all previous variable name definitions. That is, writing
```sh
sa_parse "$0" -q/--quiet/--silent @varname=mouth_shut
```
makes the option value (`true` or `false`) available only as `${mouth_shut}`.
Nothing is assigned to `q`, `quiet` and `silent`.
This way one can prevent conflicts with variables used elsewhere in the script.
For example, using `i` as a loop counter variable would conflict with flag defined as `-i`.
Using
```sh
-i @varname=incasesensitive
```
removes the conflict.

</details>

### @onvalue and @offvalue
`@onvalue` (`@offvalue`) defines the value that is assigned to the option variable
when the flag is present (is not present) in the invocation.

By default [_no argument option_](options-and-parameters.md#no-argument-option) variables are assigned
a string value `true` if the corresponding flag is present, and `false` otherwise.
In other words, by default an option variable's _on-value_ is `true` and _off-value_ is `false`.
The on-value and off-value can be overridden using `@onvalue` and `@offvalue` directives.

For example, `@onvalue=on` changes the option variable's on-value from the default `true` to `on` — no surprises here.
However, defining the on-value will also set the off-value to an empty string — and vice versa.
The table below describes the different combinations.

| Option Definition              | $\{v\} on-value | $\{v\} off-value |
| ------------------------------ | --------------- | -----------------|
| `-v`                           | "true"          | "false"          |
| `-v @onvalue=yes`              | "yes"           | ""               |
| `-v @offvalue=no`              | ""              | "no"             |
| `-v @onvalue=yes @offvalue=no` | "yes            | "no"             |

<details>
  <summary>Example - combinations</summary>

<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/on-value-and-off-value/example/myscript -->
```
sa_parse "$0" --verbose \
         --encrypt @onvalue=yes \
         --sign @offvalue=no \
         --verify @onvalue=positive @offvalue=negative
# ...
echo "verbose: '${verbose}'"
echo "encrypt: '${encrypt}'"
echo "   sign: '${sign}'"
echo " verify: '${verify}'"
```
Running the script with and without the flags present:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/on-value-and-off-value/example -c 'myscript' -c 'myscript --verbose --encrypt --sign --verify' -->
```
$ myscript
verbose: 'false'
encrypt: ''
   sign: 'no'
 verify: 'negative'
$ myscript --verbose --encrypt --sign --verify
verbose: 'true'
encrypt: 'yes'
   sign: ''
 verify: 'positive'
```

</details>

<details>
  <summary>Example - rationale and use cases 1</summary>

Consider a rudimentray `curl` wrapper script that
allows curl to be invoked with `--verbose` option by defining a flag with the same name:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/on-value-and-off-value/curl-wrapper/myscript -->
```
sa_parse "$0" --verbose "<url>"
# ...
if ${verbose}
then
  curl --verbose "${url}"
else
  curl "${url}"
fi
```
One could shrink the `if-else` statement into one line by utilizing [command substitution](https://www.gnu.org/software/bash/manual/html_node/Command-Substitution.html):
```sh
curl $(${verbose} && echo "--verbose") "${url}"
```
The command substitution of the example expands to either `--verbose` or an empty string
(which is removed by [word splitting](https://www.gnu.org/software/bash/manual/html_node/Word-Splitting.html#Word-Splitting)).
However, this gets ugly very quickly if more options are added in similar manner.
Using `@onvalue` the script can be written compactly as
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/on-value-and-off-value/curl-wrapper-compact/myscript -->
```
sa_parse "$0" --verbose @onvalue=--verbose "<url>"
# ...
curl ${verbose} "${url}"
```
Invoking the script with `--verbose` will make `${verbose}` expand to the string `--verbose`
which is passed as an option to `curl`.
Invoking the script without the verbose flag results in `${verbose}` expanding to an empty string.
Note that `${verbose}` variable is _not_ enclosed in quotes.
This way word splitting can remove it and thus prevent an empty string being passed to `curl` as its first argument (raises an error).

</details>

<details>
  <summary>Example - rationale and use cases 2</summary>

Consider another script that by default uses HTTP for something but can use HTTPS if required.
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/on-value-and-off-value/https/myscript -->
```
sa_parse "$0" --use-https @varname=protocol @onvalue=https @offvalue=http
# ...
echo "Using ${protocol}"
```
Running the script with and without the flag:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/on-value-and-off-value/https -c 'myscript' -c 'myscript --use-https' -->
```
$ myscript
Using http
$ myscript --use-https
Using https
```
The option's variable name is set to `protocol` to make it more self-documenting (see [@varname](#varname)).
Additionally, the on-value and off-value are defined.
When the flag `--use-https` is used `${protocol}` is set to the on-value `https`;
when the flag is omitted `${protocol}` expands to the off-value `http`.

As presented, using `@onvalue` and `@offvalue` directives can save writing unnecessary if-else statements
and manual variable assignments.
In addition to writing more compact scripts one is also producing more readable code.

>**TIP**  
  Instead of comparing the default on-value and off-value as strings consider the more compact alternatives:
  ```sh
  if [ "${verbose}" = "true" ]; then
      echo "some verbose output"
  fi
  # The above can be replaced with...
  if ${verbose}; then
      echo "some verbose output"
  fi
  # ...or with the even more compact...
  ${verbose} && echo "some verbose output"
  ```

</details>

### @allowrepeat
`@allowrepeat` specifies that a [no argument option](options-and-parameters#no-argument-options)
can be repeated many times.
Using `@allowrepeat` makes available an additional option variable that contains 
the number of times the option flag was given in the script invocation.
The variable name is formed from the option variable by appending `_count`.
If the variable name for the option is `xyz` the count variable will be named `xyz_count`

See also [@multivalue](#multivalue-list) directive applicable to [required argument options](#required-argument-option).

<details>
  <summary>Example - verbose and more verbose</summary>

Normally no argument options are used to signal that some functionality is to be turned on or off.
However, sometimes it is significant how many times a flag is given.
Some scripts use `-v` for verbose output and `-v -v` (or `-vv` for short) for even more verbose output.

This can be implemented using _@allowrepeat_:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/allowrepeat/myscript -->
```
sa_parse "$0" -v/--verbose @allowrepeat
# ...
case "${v_count}" in
    0) log_level=ERROR;;
    1) log_level=INFO;;
    *) log_level=DEBUG;;
esac
echo "-v flag given: ${v}(${verbose_count}): log level set to ${log_level}"
```
Running the script:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/allowrepeat -c 'myscript' -c 'myscript -v' -c 'myscript -vv' -c 'myscript -vv -v -v --verbose' -->
```
$ myscript
-v flag given: false(0): log level set to ERROR
$ myscript -v
-v flag given: true(1): log level set to INFO
$ myscript -vv
-v flag given: true(2): log level set to DEBUG
$ myscript -vv -v -v --verbose
-v flag given: true(5): log level set to DEBUG
```

In the example above there are two option variables: `v` and `verbose`.
Hence, the additional variables are `v_count` and `verbose_count`.

Note that as `v` and `verbose` are assigned the same value ("true" or "false")
also `v_count` and `verbose_count` always hold the same number.
That is, how many times the flag (both `-v` and `--verbose` in total) was given on command line.

</details>

### @doc
`@doc` adds documentation to an entry.
The documentation is shown on the generated man page and usage printout.
The directive can be specified multiple times for an entry
resulting in multiparagraph documentation on the man page.
The usage printout always includes only the first paragraph.

The directive value can contain special placeholder:
* `@{d}` is replaced with the default value of the entry
* `@{v}` is replaced with the valid values for the entry

See [Automatic Documentation](automatic-documentation.md) for more information and examples.

<details>
  <summary>Example</summary>

<!--@evalsh filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/doc/myscript -->
```sh
sa_parse "$0" \
         --env arg @validvalues=local,dev,prod @default=local \
         @doc="Target environment (default: '@{d}'). Possible values: @{v}"
```
produces the following usage printout.
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/doc -c 'myscript -h' -->
```
$ myscript -h
Usage: myscript [OPTION]...

Options:
  -h, --help                    Print usage instructions and exit.
  --env ARG                     Target environment (default: 'local'). Possible
                                values: 'local', 'dev', and 'prod'
```

</details>

### @allowempty
`@allowempty` specifies that an option or parameter can have an empty value.
By default an empty option or parameter value raises an error:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/required-argument-option -c 'myscript --user ""' -->
```
$ myscript --user ""
ERROR: -u/--user: value is empty (use @allowempty to allow empty values)
Usage: myscript [OPTION]...
```
This is the default behaviour because passing an empty option or parameter value is almost always an error.
Specifying an empty value on command line manually is probably not that common.
However, consider the case of another script calling a simpleargs based script (myscript):
```sh
# ...
user="$(get-user)"
myscript --user "${user}"
# ...
```
What if `get-user` fails and `user` variable is left empty?
Since `myscript` raises an error by default the problem is caught early with a clear error message.
Otherwise, the eventual failure happens only later and might be more difficult to debug.

<details>
  <summary>Example</summary>

In some cases an empty value is a valid one.
As the error message in the example above suggests `@allowempty` can be used to signal that.
Consider the following script that capitalizes a (possibly empty) set of letters in its input.
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/allowempty/capitalize -->
```
sa_parse "$0" -c/--capitalized-letters letters @allowempty \
              "<input string>"
# ...
echo "${input_string}" | tr "${letters,,}" "${letters^^}"
```
The set of letters to be capitalized are given as an option value.
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/allowempty -c 'capitalize -c ae "abcde-abcde"' -c 'capitalize -c "" "abcde-abcde"' -->
```
$ capitalize -c ae "abcde-abcde"
AbcdE-AbcdE
$ capitalize -c "" "abcde-abcde"
abcde-abcde
```
Without `@allowempty` directive the second invocation of `capitalize` would fail.

</details>

### @multivalue (list)
`@multivalue` makes a [_required argument option_](options-and-parameters.md#required-argument-option)
accept multiple values.
The syntax is `@multivalue[<separator>]`.
Multiple values can be provided by specifying the option multiple times.
If the optional separator character, for example comma, is used (`@multivalue,`)
the values can be written _additionally_ as a comma separated list.

See also [@allowrepeat](#allowrepeat) directive applicable to
[no argument options](options-and-parameters.md#no-argument-option).

<details>
  <summary>Example - audio filters</summary>

Consider a script that is used to apply audio filters to mp3 files:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/multivalue/filter-audio/filter -->
```
sa_parse "$0" -f/--filter arg @multivalue @varname=filters \
              "<mp3 file>..." @varname=mp3s
# ...
echo "Filters:"
for filter in "${filters[@]}"; do
    echo " - ${filter}"
done
echo "MP3 files:"
for mp3 in "${mp3s[@]}"; do
    echo " - ${mp3}"
done
```
Running the script:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/multivalue/filter-audio -c 'filter -f echo -f lowpass wagner.mp3 mozart.mp3' -->
```
$ filter -f echo -f lowpass wagner.mp3 mozart.mp3
Filters:
 - echo
 - lowpass
MP3 files:
 - wagner.mp3
 - mozart.mp3
```

The functionality is straightforward:
the flag for filters is given two times and hence two values are assigned
as the items of `filters` array (specified by `@varname=filters`).

To avoid repeating the flag one can add a separator character to the directive.
Below colon `:` is used.
<!--@evalsh filter-simpleargs-script -r /sa_parse/ -r e/sa_end_parse/ $SIMPLEARGS_DOC_SCRIPTS/directives/multivalue/filter-audio-colon-separated/filter -->
```sh
sa_parse "$0" -f/--filter arg @multivalue: @varname=filters \
              "<mp3 file>..." @varname=mp3s
```

Be sure to use a character that cannot be present in the option values.
The recommended separator character is colon `:`.
It rarely occurs in the values and plays well with the command completion feature of simpleargs
(see [Command Completion](#command-completion) for details).
Other good candidates are comma `,`, percent `%`, and plus `+`, but technically you can use other characters as well.

Now you can call the script a bit more compactly and even mix the two ways of specifying filters:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/multivalue/filter-audio-colon-separated -c 'filter -f echo -f lowpass:noise-remove strauss.mp3' -->
```
$ filter -f echo -f lowpass:noise-remove strauss.mp3
Filters:
 - echo
 - lowpass
 - noise-remove
MP3 files:
 - strauss.mp3
```

</details>

### @validvalues
`@validvalues` specifies the valid values for an option or a parameter.
The syntax is
```
@validvalues[<sep>]=<item1><sep><item2><sep>...
```
where `<sep>` is a one character long separator that separates the items of the directive value.
The default separator is comma `,`.
Hence, the following definitions are equivalent.
```sh
--color arg @validvalues=red,green,blue
--color arg @validvalues,=red,green,blue
--color arg @validvalues+=red+green+blue
--color red,green,blue
```
The last definition is a shortcut where the token after the flag definition (`--color`) is interpreted as
a comma separated list of valid values.

There's also a shortcut for defining the default value.
Instead of writing `@default=green` one can append `@default` to the value.
In other words, one can specify `green` as the default with any of the following definitions:
```sh
--color arg @validvalues=red,green@default,blue
--color arg @validvalues,=red,green@default,blue
--color arg @validvalues+=red+green@default+blue
--color red,green@default,blue
```

<details>
  <summary>Example - audio compressor</summary>

A script for compressing audio files has an option for setting the compression factor.
The possible values are statically defined: "low", "medium", and "high".
In general, the valid values could be specifed with a [regular expression](validation.md#grep).
However, in many cases it is the simplest to enumerate the values.
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/validvalues/compress-audio/compressaudio -->
```
sa_parse "$0" -c/--compression-factor arg @validvalues=low,medium,high \
         "<input file>"
# ...
echo "Compressing '${input_file}' with ${c} compression..."
```
Running the script:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/validvalues/compress-audio -c 'compressaudio -c loe pachelbell.wav' -c 'compressaudio -c low pachelbell.wav' -->
```
$ compressaudio -c loe pachelbell.wav
ERROR: -c/--compression-factor: invalid value 'loe'
Valid values are: 'low', 'medium' and 'high'.
Usage: compressaudio [OPTION]... <input file>
$ compressaudio -c low pachelbell.wav
Compressing 'pachelbell.wav' with low compression...
```

</details>

<details>
  <summary>Example - commas in valid values</summary>

If the valid values contain commas themselves one needs to specify an explicit separator character.
```sh
--coordinates arg @validvalues:=0,0:1,4:8,12
```
In the above example the possible coordinate values are `0,0`, `1,4`, and `8,12`.

</details>

<details>
  <summary>Example - parameter values</summary>

One can use `@validvalues` directive also with parameters.
Some commands use a style where the first argument after the command is, an _operation_ or _command_.
Consider `hg`, the command line tool of Mercurial version control for instance.
```
NAME
      hg - Mercurial source code management system

SYNOPSIS
      hg command [option]... [argument]...
```
The first argument for `hg` is  `add`, `remove`, `push`, `pull`, `commit`, etc.
That is, the value of `command` is one from a finite set of actions.
One can mimic this using simpleargs:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/validvalues/command-as-first-parameter/my-hg -->
```
sa_parse "$0" "<command>" @validvalues=add,remove,push,pull
# ...
echo "Executing command '${command}'"
```
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/validvalues/command-as-first-parameter -c 'my-hg remoev' -c 'my-hg remove' -->
```
$ my-hg remoev
ERROR: <command>: invalid value 'remoev'
Valid values are: 'add', 'remove', 'push' and 'pull'.
Usage: my-hg [OPTION]... <command>
$ my-hg remove
Executing command 'remove'
```

So what is the benefit of using `@validvalues` over using a regular expression?
1. The definition is often easier and more natural to write and read.
2. The error message upon an invalid value is more informative (for example, valid values are displayed)
3. The shell can perform autocompletion for the value.
```
$ my-hg [tab][tab]
add    pull   push   remove
$ my-hg p[tab]
$ my-hg pu[tab][tab]
pull  push
$ my-hg pul[tab]
$ my-hg pull
```
See the section about [command completion](#command-completion) for more information.
</details>

See also [how to combine this directive with other validation](validation.md#combining-validations-with-validvalues).

### @validvaluesfile
`@validvaluesfile` specifies a path to a file that contains the valid values of an option or a parameter one per line.
The syntax is
```
@validvaluesfile=<path to file>
```

<details>
  <summary>Example - elements</summary>

Consider a script that prints the element chosen by the user.
<!--@evalsh filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/validvaluesfile/myscript -->
```sh
sa_parse "$0" -e/--element arg @validvaluesfile=/tmp/elements.txt
# ...
echo "Your favourite chemical element: ${element}"
```
The values are validated similarly to when using `@validvalues` directive.
<!--@evalonly cp $SIMPLEARGS_DOC_RESOURCES/elements.txt /tmp/elements.txt -->
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/validvaluesfile -c 'myscript -e Fire' -c 'myscript -e Iron' -->
```
$ myscript -e Fire
ERROR: -e/--element: invalid value 'Fire'
Valid values are: 'Actinium', 'Aluminium', 'Americium', 'Antimony', 'Argon',...
Usage: myscript [OPTION]...
$ myscript -e Iron
Your favourite chemical element: Iron
```
The file pointed to by the directive contains the valid values one per line:
<!--@eval shell-session-simulator -c 'head -n 5 /tmp/elements.txt' -->
```
$ head -n 5 /tmp/elements.txt
Actinium
Aluminium
Americium
Antimony
Argon
```

</details>

Using `@validvaluesfile` instead of `@validvalues` allows modifying the valid values between script invocations.
A cron job might update the file periodically to keep the set of valid values up-to-date.
Or maybe the valid values are defined per user:
```sh
sa_parse "$0" --favorite-food arg @validvaluesfile='${HOME}/.favorite-foods'
```
Using single quotes prevents the shell from expanding the environment variable "too early".
<!-- TODO: add write sections about caching and quoting and link them here -->
Note portability considerations: the file should be available in every environment
where the script is intended to be run.

### @validvaluescommand
`@validvaluescommand` defines an expression that (when evaluated) prints the valid values for an option or a parameter.
The values should be printed one per line and the expression should have `0` as return value.
The syntax is
```
@validvaluescommand=<expression>
```
Under the hood `eval` is used to evaluate the expression.
Therefore the expression is not restricted to simple commands.
Pipelines, shell functions, or even more complex constructs such as loops can be used as well.
However, note portability considerations:
the commands should be available in all environments where the script is intended to be used.
<!-- TODO: note about the command being used also when autocompleting -->

<details>
  <summary>Example</summary>

Consider a script that executes an action in your company's CI (Continuous Integration) pipeline.
The allowed actions are `analyse`, `compile`, `test`, and `deploy`.
However, after one unfortunate Friday when a production deployment went haywire
the company created a rule that the product shouldn't be deployed to production on Fridays.
<!--@evalsh filter-simpleargs-script -r 1 $SIMPLEARGS_DOC_SCRIPTS/directives/validvaluescommand/ci-execute -->
```sh
#!/usr/bin/env bash

# From man page of date:
# %w     day of week (0..6); 0 is Sunday
ci_actions='
  printf "analyse\ncompile\ntest\n"
  [ $(date +%w) -ne 5 ] && printf "deploy\n"
  true
'
# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" "<action>" @validvaluescommand="${ci_actions}"
sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
# ----------------------------------------------------------------------------
echo "Executing action: ${action}"
```
When evaluated `${ci_actions}` will print `analyse`, `compile`, and `test` on their own lines.
Additionally, if it's not Friday, `deploy` will be printed as well.
`true` will make the return value of the whole thing `0` also on Fridays.
Now, if someone tries to make a deployment on Friday the validation will intervene and save everyone's weekend:
```
$ ci-execute deploy
ERROR: <action>: invalid value 'deploy'
Valid values are: 'analyse', 'compile' and 'test'.
Usage: ci-execute [OPTION]... <action>
```

</details>

**WARNING**  
Note that there are some security considerations related to using `eval`.
See [this Bash FAQ entry](http://mywiki.wooledge.org/BashFAQ/048) for a detailed explanation.

### @afterprocessing
`@afterprocessing` defines an expression to be evaluated using `eval`
right after the corresponding option has been processed (and before anything else is processed).
The syntax is
```sh
@afterprocessing=<expression>
```
This directive is mainly for internal (or advanced) use.

<details>
  <summary>Example</summary>

The script below defines two flags with `@afterprocessing` directive added to each one.
<!--@evalsh filter-simpleargs-script -r 1 $SIMPLEARGS_DOC_SCRIPTS/directives/afterprocessing/myscript -->
```sh
#!/usr/bin/env bash

# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" \
         --host arg @afterprocessing='echo "--host: |${host}:${port}|"' \
         --port arg @afterprocessing='echo "--port: |${host}:${port}|"'
sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
# ----------------------------------------------------------------------------
echo "Done"
```
When the script is run with both flags enabled the expressions are evaluated
during the argument processing in `sa_process`.
The flags are handled in the order they are specified in the invocation
which results in the following output.
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/afterprocessing -c 'myscript' -c 'myscript --host example.com --port 8080' -c 'myscript --port 8080 --host example.com' -->
```
$ myscript
Done
$ myscript --host example.com --port 8080
--host: |example.com:|
--port: |example.com:8080|
Done
$ myscript --port 8080 --host example.com
--port: |:8080|
--host: |example.com:8080|
Done
```

</details>

<details>
  <summary>Example - implementing --help</summary>

`@afterprocessing` allows executing custom code after processing an option.
There's normally no need for this, but it is required in implementing the `--help` (or `--version`) flag:
```sh
-h/--help @doc="Print usage instructions and exit." \
          @afterprocessing="sa_display_usage; exit 0"
```
`@afterprocessing` directive above makes sure that calling `myscript --help` will display the usage instructions
and exit immediately after the `--help` flag is processed.
(See [this question](faq-and-tips.md#adding-h-does-not-work-or-how-do-i-disable-the-h-help-option) for more information about how `--help` is implemented.)
The reason the usage instructions cannot be displayed in the user part of the script
i.e. after the argument processing is fully done is intricate.
Consider a naive approach to implement the help flag.
<!--@evalsh filter-simpleargs-script --rule '/-- simpleargs --/' $SIMPLEARGS_DOC_SCRIPTS/directives/afterprocessing/naivehelp -->
```sh
# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" \
         -h/--help @doc="Print usage instructions and exit." \
         "<input>"
sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
# ----------------------------------------------------------------------------

if ${help}
then
    sa_display_usage
    exit 0
fi
# process ${input}
```
trying to display the usage instructions would fail because `sa_process` would signal a failure in
validating the arguments which would make `sa_end_process` exit the script before reaching
the printing of the usage instructions.
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/afterprocessing -c 'naivehelp --help' -->
```
$ naivehelp --help
ERROR: Missing required parameter <input>
Usage: naivehelp [OPTION]... <input>
```

</details>

### @required
`@required` marks _an option_ as a required one i.e. it must be given with every invocation.

<details>
  <summary>Example - user required</summary>

Usually the required arguments to a script are defined as parameters,
but if it makes sense to have mandatory options, `@required` directive should be used.
<!--@evalsh filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/directives/required/myscript -->
```sh
sa_parse "$0" -u/--user arg @required \
         "<input>"
# ...
echo "Processing ${input} (user: ${user})"
```
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/directives/required -c 'myscript input.txt' -c 'myscript -u jill input.txt' -->
```
$ myscript input.txt
ERROR: required option -u/--user not given
Usage: myscript [OPTION]... <input>
$ myscript -u jill input.txt
Processing input.txt (user: jill)
```

</details>

Note that this directive is only to be used with _options_, not _parameters_.
See [here](options-and-parameters.md#defining-script-parameters)
for how to define normal (required) and optional parameters.
Also, it goes without saying that it doesn't make sense to use `@required` and `@default` together.

### @expand
`@expand` is a special directive that is replaced with (i.e. expanded to) other tokens.
The syntax is
```
@expand=<token name>
```
At runtime `@expand=xyz` is replaced with the elements of an array named `sa_expand_token_xyz`.
The token name (in this case `xyz`) should consist of letters and underscore (`[A-Za-z_]`).

<details>
  <summary>Example - verbose</summary>

Defining an "expansion array" and using `@expand`
<!--@evalsh filter-simpleargs-script -r /sa_expand_token/ -r e/sa_end_parse/ $SIMPLEARGS_DOC_SCRIPTS/directives/expand/myscript-with-expand-token -->
```sh
sa_expand_token_verbose=( --verbose @doc="Makes script output more verbose." )
# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" @expand=verbose \

```
is equivalent with
<!--@evalsh filter-simpleargs-script -r '/^# -/' -r /sa_parse/ $SIMPLEARGS_DOC_SCRIPTS/directives/expand/myscript-normal -->
```sh
# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" --verbose @doc="Makes script output more verbose." \
```

</details>

There are (at least) two use cases for `@expand` directive:
1. A script with many options or parameters that have for example the same validation rules.
   One can avoid repetition:
   ```sh
   sa_expand_token_email=( @@egrep='...very long regex that matches valid email address...' )
   ...
   sa_parse "$0" \
            --cc arg @expand=email \
            --bcc arg @expand=email
            "<from>" @expand=email \
            "<to>" @expand=email \
   ```
2. One wants to use the same definitions in many scripts.
   `--verbose` option of the example above might be something that many scripts have in common.
   In this case the expansion array definition `sa_expand_token_verbose=(...)`
   should be placed in its own file e.g. `common-options.sh`.
   The scripts would source `common-options.sh` and use `@expand=verbose` in the argument definitions.

Note that the latter use case has a slight disadvantage.
The script is now dependent on `common-options.sh`.
If the script is given to someone else one needs to remember to provide `common-options.sh` as well.

See [this FAQ entry](faq-and-tips.md#adding-h-does-not-work-or-how-do-i-disable-the-h-help-option)
to learn how `@expand` is used to provide the `-h/--help` option to scripts by default.
