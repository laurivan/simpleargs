<!-- GENERATED TOC -->
<!-- TOC START -->
## Table of Contents

  - [Options and Parameters](#options-and-parameters)
    - [Defining script options](#defining-script-options)
      - [No argument option](#no-argument-option)
      - [Required argument option](#required-argument-option)
      - [Optional argument option](#optional-argument-option)
    - [Defining script parameters](#defining-script-parameters)
      - [Normal parameters](#normal-parameters)
      - [Optional parameters](#optional-parameters)
      - [Varargs parameters](#varargs-parameters)
      - [Optional varargs parameters](#optional-varargs-parameters)
<!-- TOC END -->

## Options and Parameters
The following sections describe how to define options and parameters as well as
how to use directives to tweak their behaviour, add validation, documentation etc.
There are a lot of examples for two reasons.
They are usually the simplest way to illustrate the functionality.
Additionally, they give the reader ideas of the possibilities of that particular feature.

[Getting Started](getting-started.md) section already covered the basic structure of a simpleargs based script:
<!--@eval filter-simpleargs-script -r 1 -r 'e/^# ...$/' -r // $SIMPLEARGS_DOC_SCRIPTS/reference-guide-preamble/myscript -->
```
#!/usr/bin/env bash

# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" -q -v/--verbose -p/--print/--print-values
sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
# ----------------------------------------------------------------------------
echo "q: '${q}'"
echo "v: '${v}', verbose: '${verbose}'"
echo "p: '${p}', print: '${print}', print_values: '${print_values}'"
```

Since the scripts differ only in the user part and the call to `sa_parse`
the examples are shortened to include only the relevant parts.
Hence, the above example is presented as seen below.
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/reference-guide-preamble/myscript -->
```
sa_parse "$0" -q -v/--verbose -p/--print/--print-values
# ...
echo "q: '${q}'"
echo "v: '${v}', verbose: '${verbose}'"
echo "p: '${p}', print: '${print}', print_values: '${print_values}'"
```

### Defining script options
#### No argument option
To define an option that takes no arguments one simply adds a slash `/` separated list of flags:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/reference-guide-preamble/myscript -->
```
sa_parse "$0" -q -v/--verbose -p/--print/--print-values
# ...
echo "q: '${q}'"
echo "v: '${v}', verbose: '${verbose}'"
echo "p: '${p}', print: '${print}', print_values: '${print_values}'"
```

The above example defines three options.
Each option has one or more (equivalent) flags which correspond to _option variables_.
The variable names are simply the flag names with other than [alphanumeric](https://en.wikipedia.org/wiki/Alphanumeric)
characters replaced with an underscore `_`.
Leading hyphen(s) are not replaced but simply removed.

Hence, the variable names for the options are:

* `q` for `-q`
* `v` and `verbose` for `-v/--verbose`
* `p` and `print` and `print_values` for `-p/--print/--print-values`

The variable is assigned a string value `true` or `false`
depending on whether the flag is present in the invocation or not respectively.

<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/reference-guide-preamble -c 'myscript -q --print' -->
```
$ myscript -q --print
q: 'true'
v: 'false', verbose: 'false'
p: 'true', print: 'true', print_values: 'true'
```
Note that all the variables corresponding to a specific option are assigned
the same value even if only one of the flags is specified in the invocation.
That is, in the example above variables `p`, `print`, and `print_values` are all assigned value `true`.

#### Required argument option
An option with a required argument is defined similarly to a _no argument option_:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/required-argument-option/myscript -->
```
sa_parse "$0" -u/--user arg
# ...
echo "u: '${u}'"
echo "user: '${user}'"
```
The only difference is the keyword `arg` that comes right after the flag token and
signals that the option takes a mandatory value.
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/required-argument-option -c 'myscript -u jack' -c 'myscript -u' -->
```
$ myscript -u jack
u: 'jack'
user: 'jack'
$ myscript -u
ERROR: Processing arguments failed: option requires an argument -- u
Usage: myscript [OPTION]...
```

Deriving the option's variable name from the flag definition is handy in most cases.
The syntax is concise and results in readable code when a long flag is defined for the option.
However, sometimes this behaviour is not wanted.
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/required-argument-option/cumbersome/myscript -->
```
sa_parse "$0" -u arg -r arg
# ...
echo "Connecting to ${r} with '${u}'..."
```
If the script uses only short flags it is difficult to write self-documenting code.
This is easily fixed by replacing the keyword `arg` with an _implicit variable name_:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/required-argument-option/cumbersome-fixed/myscript -->
```
sa_parse "$0" -u user -r remote_host
# ...
echo "Connecting to ${remote_host} with '${user}'..."
echo "u: '${u}', r: '${r}'" # These variables are no longer used
```
Note that declaring the implicit variable name results in the option value being assigned to only that variable.
That is, no variable names are derived from the flag anymore.
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/required-argument-option/cumbersome-fixed -c 'myscript -u jack -r example.abc.com' -->
```
$ myscript -u jack -r example.abc.com
Connecting to example.abc.com with 'jack'...
u: '', r: ''
```

#### Optional argument option
This option type is relatively rarely seen.
`sed` man page provides an example:
```
  -i[SUFFIX], --in-place[=SUFFIX]
      edit files in place (makes backup if SUFFIX supplied)
```

One can replace all occurrences of `foo` with `bar` in the input file with
```sh
sed -i 's/foo/bar/g' data.txt
```
but a mistake in the replacement command may result in loss of data since
the output overwrites the input file.
However, providing _an optional value_ `.bkp` instructs sed to make a backup of the input file:
```sh
sed -i.bkp 's/foo/bar/g' data.txt # Creates data.txt.bkp
# OR
sed --in-place=.bkp 's/foo/bar/g' data.txt # Creates data.txt.bkp
```

To achieve similar behaviour with simpleargs one uses `@optionalvalue` directive:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/optional-argument-option/myscript -->
```
sa_parse "$0" -i/--in-place arg @optionalvalue
# ...
echo "in_place: '${in_place}'"
```
The directive `@optionalvalue` is something new.
Directives are covered in more detail in the following sections.
For now it is sufficient to know that the directive alters the option
by making it take an _optional_ value.
This means that invocations with and without the option value are acceptable
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/optional-argument-option -c 'myscript --in-place' -c 'myscript --in-place=".bkp"' -->
```
$ myscript --in-place
in_place: ''
$ myscript --in-place=".bkp"
in_place: '.bkp'
```

A sharp reader might ask what happens if the script is invoked with no arguments at all.
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/optional-argument-option -c 'myscript' -->
```
$ myscript
in_place: ''
```
How can the script differentiate between the three invocations:

1. `myscript -i` (option given with no value)
2. `myscript -i""` (option given with an empty value)
3. `myscript` (option not given at all)

After all the option variable `${in_place}` expands to an empty string in all three cases.
There is no way to distinguish between scenarios 1 and 2 since in both cases
the argument that the script sees is `-i`. An empty string is assigned to the option variable.

However, in scenario 3 the option variable is not set to an empty string — it is left unset.
One can [distinguish between an unset variable and a variable with an empty value](https://stackoverflow.com/a/5406887).
However, the problem is more easily tackled by defining _a default value_ for the option using [@default](directives.md#default) directive.

### Defining script parameters
In addition to options many scripts accept parameters.
For example, input and output files are normally given as _positional_ parameters to scripts:
<!--@eval filter-simpleargs-script -r 1 $SIMPLEARGS_DOC_SCRIPTS/defining-script-parameters/normal-script/foo2bar -->
```
#!/usr/bin/env bash

# Invocation: foo2bar input.txt output.txt
input_file="$1"
output_file="$2"
sed 's/foo/bar/g' "${input_file}" > "${output_file}"
```

Simpleargs lets user define _named parameters_. The example above could be written like
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/defining-script-parameters/simpleargs-script/foo2bar -->
```
sa_parse "$0" "<input file>" "<output file>"
# ...
sed 's/foo/bar/g' "${input_file}" > "${output_file}"
```
The arguments to `sa_parse` define two mandatory parameters.
Variable names are derived from the definitions similarly to options:
the angle brackets (`<` and `>`) are removed and characters that cannot be present in shell variable names are replaced with underscores `_`.

This particular functionality in simpleargs brings up the distinction between _positional and named parameters_.
From now on by _parameters_ one means _named parameters_.
When talking about _positional parameters_ it will be explicitly stated or clear from the context.

Simpleargs supports normal (required) and optional parameters.
Both can be either _single_ or _varargs_ parameters.
This gives a total of four types of _named parameters_ that are described next.

#### Normal parameters
You already saw how (normal) parameters are defined:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/normal-parameters/foo2bar -->
```
sa_parse "$0" --verbose "<input file>" "<output file>"
# ...
${verbose} && echo "Filtering ${input_file} --> ${output_file}"
sed 's/foo/bar/g' "${input_file}" > "${output_file}"
```
As illustrated above parameter definitions are written after all option definitions.
The syntax consists of a parameter name enclosed in angle brackets.
The parameter name can include spaces but (as with options)
the corresponding variable name is derived by replacing non-alphanumeric characters with underscores.
Note that the definition needs to be quoted with either double `"` or single quotes `'`
to prevent the shell from interpreting the brackets as redirection operators.

Since the example defines _required_ parameters invoking the script like `foo2bar input.txt` will result in an error
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/foo2bar --path $SIMPLEARGS_DOC_SCRIPTS/normal-parameters -c "cat input.txt" -c 'foo2bar input.txt' -c 'foo2bar --verbose input.txt output.txt' -c 'cat output.txt' -->
```
$ cat input.txt
This foo might become bar.
$ foo2bar input.txt
ERROR: Missing required parameter <output file>
Usage: foo2bar [OPTION]... <input file> <output file>
$ foo2bar --verbose input.txt output.txt
Filtering input.txt --> output.txt
$ cat output.txt
This bar might become bar.
```

#### Optional parameters
Optional parameters are surprisingly — _optional_.
Modifying the previous example we could derive a script that prints its output to stdout if the output file parameter is omitted:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/optional-parameters/foo2bar -->
```
sa_parse "$0" "<input file>" "[<output file>]"
# ...
if [ -n "${output_file}" ]
then
  sed 's/foo/bar/g' "${input_file}" > "${output_file}"
else
  sed 's/foo/bar/g' "${input_file}"
fi
```
The syntax adheres to the common convention of surrounding optional tokens with square brackets `[]`.
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/foo2bar --path $SIMPLEARGS_DOC_SCRIPTS/optional-parameters -c "cat input.txt" -c 'foo2bar input.txt' -->
```
$ cat input.txt
This foo might become bar.
$ foo2bar input.txt
This bar might become bar.
```

#### Varargs parameters
What if the script is supposed to produce a single output file but take possibly multiple input files.
This would normally be implemented somewhat like
<!--@eval filter-simpleargs-script -r 1 $SIMPLEARGS_DOC_SCRIPTS/varargs-parameters/normal-script/foo2bar -->
```
#!/usr/bin/env bash
# Invocation: foo2bar output.txt input1.txt input2.txt ...
output_file="$1"; shift
> "${output_file}"
for input_file in "$@"
do
    echo "--- File: ${input_file} ---" >> "${output_file}"
    sed 's/foo/bar/g' "${input_file}" >> "${output_file}"
done
```

[Varargs](https://en.wikipedia.org/wiki/Variadic_function) (short for _variable-length arguments_)
is a common idiom in many programming languages:
a function can take zero or more arguments the number of which is not known beforehand.
The above functionality can be achieved using simpleargs with a _varargs parameter_:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/varargs-parameters/simpleargs-script/foo2bar -->
```
sa_parse "$0" "<output file>" "<input files>..."
#...
> "${output_file}"
for input_file in "${input_files[@]}"
do
    echo "--- File: ${input_file} ---" >> "${output_file}"
    sed 's/foo/bar/g' "${input_file}" >> "${output_file}"
done
```
The syntax aims to be self-describing:
a normal parameter definition `"<input files>"` is suffixed with three dots indicating
that there can be *one or more* output files.
That's right, with simpleargs a varargs parameter takes at least one value.
Providing no values is an error (see [the next section](#optional-varargs-parameters) for "parameters with zero or more values").

The values (input files) are now accessed through an array (`${input_files[0]`, `${input_files[1]`, ...).
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/foo2bar --path $SIMPLEARGS_DOC_SCRIPTS/varargs-parameters/simpleargs-script -c "cat input.txt" -c 'cat input-two.txt' -c 'foo2bar out.txt' -c 'foo2bar out.txt input.txt input-two.txt' -c 'cat out.txt' -->
```
$ cat input.txt
This foo might become bar.
$ cat input-two.txt
Something might happen to this foo as well.
$ foo2bar out.txt
ERROR: Missing required parameter <input files>...
Usage: foo2bar [OPTION]... <output file> <input files>...
$ foo2bar out.txt input.txt input-two.txt
$ cat out.txt
--- File: input.txt ---
This bar might become bar.
--- File: input-two.txt ---
Something might happen to this bar as well.
```

#### Optional varargs parameters
What if the script in the previous section should support stdin as its input?
That is, if no input files are specified the script should take its input from stdin.
As stated in the previous section invoking the script above with
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/foo2bar --path $SIMPLEARGS_DOC_SCRIPTS/varargs-parameters/simpleargs-script -c 'foo2bar out.txt' -->
```
$ foo2bar out.txt
ERROR: Missing required parameter <input files>...
Usage: foo2bar [OPTION]... <output file> <input files>...
```
would result in an error since a varargs parameter takes *one* or more values.

To overcome this one can use an _optional varargs parameter_:
<!--@eval filter-simpleargs-script --example $SIMPLEARGS_DOC_SCRIPTS/optional-varargs-parameters/foo2bar -->
```
sa_parse "$0" "<output file>" "[<input files>]..."
#...
> "${output_file}"
if [ ${#input_files[*]} -eq 0 ]
then
    # Read input from stdin
    sed 's/foo/bar/g' >> "${output_file}"
else
    for input_file in "${input_files[@]}"
    do
        echo "--- File: ${input_file} ---" >> "${output_file}"
        sed 's/foo/bar/g' "${input}" >> "${output_file}"
    done
fi
```
The syntax is analogous with the normal varargs parameter:
an optional varargs parameter can be defined by appending three dots
to an optional parameter definition `"[<input file>]..."`.
The if-statement checks whether there are any input files and if not executes
`sed` with no input files which makes it read its input from stdin.
<!--@eval shell-session-simulator --exec-dir $SIMPLEARGS_DOC_RESOURCES/foo2bar --path $SIMPLEARGS_DOC_SCRIPTS/optional-varargs-parameters -c 'echo "This foo is about to change." | foo2bar filtered.txt' -c 'cat filtered.txt' -->
```
$ echo "This foo is about to change." | foo2bar filtered.txt
$ cat filtered.txt
This bar is about to change.
```
