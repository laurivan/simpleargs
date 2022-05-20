<!-- GENERATED TOC -->
<!-- TOC START -->
## Table of Contents

  - [FAQ and tips](#faq-and-tips)
    - [Adding -h does not work OR how do I disable the -h/--help option?](#adding-h-does-not-work-or-how-do-i-disable-the-h-help-option)
    - [Integer or float range with a negative minimum does not work.](#integer-or-float-range-with-a-negative-minimum-does-not-work)
    - [Do I have to use named parameters? Can I use ordinary parameters `$1`, `$2`, ...?](#do-i-have-to-use-named-parameters-can-i-use-ordinary-parameters-1-2-)
    - [What are the restrictions on parameter order?](#what-are-the-restrictions-on-parameter-order)
  - [Installed files](#installed-files)
<!-- TOC END -->

## FAQ and tips
### Adding -h does not work OR how do I disable the -h/--help option?
If you try to define your own option `-h` (or `--help`) you will get an error like
```
ERROR: -h: Option already defined: '-h'
ERROR: To disable the default help option (-h/--help) use 'sa_default_tokens=()'
```
To fix this you can use some other flag than `-h` for your option, or you can disable the default
help option like the error message suggests by adding
```sh
sa_default_tokens=()
```
to your script (before simpleargs preamble).

So, what's happening here?
In addition to the normal option and parameter definitions (arguments to `sa_parse`)
there is a special variable called `sa_default_tokens` which
* is normally undefined. This results in the library assigning it to its default value
  which is an array
  ```
  ( @expand=help )
  ```
  (we'll get to that syntax later)
* can be assigned by the script (overriding the default value) to an array that contains one or more option definitions:
  ```
  sa_default_tokens=( --port arg @@int @doc="Server port" )
  ```
  The definitions are processed _before_ the definitions given as arguments to `sa_parse`.

The use case for this mechanism is more evident if one considers writing dozens of scripts and wants all of them
to have a set of common options.
One might want all of one's scripts to have `--verbose` option.
This would be done by creating a source file `default-options.sh` with the following content
```sh
sa_default_tokens=( --verbose @doc="Enable additional output" )
```
and sourcing it in the beginning of the scripts
```
. default-options.sh
```

On the other hand, one might have a set of options that one uses frequently but not necessary all of them in every script
and one would want to avoid repeating the definitions in every file.
This can be solved by creating a source file e.g. `common-options.sh` with so called [expand tokens](directives.md#expand)
```sh
# sa_expand_token_<name>=( <definition>... )
sa_expand_token_server_port=( -p/--port arg @doc="Server port" )
sa_expand_token_server_password=( --password arg @doc="Server password" )
```

After sourcing the file one can use use `@expand` directive: `@expand=server_port` and `@expand=server_password`:
```sh
. common-options.sh
# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" @expand=server_port @expand=server_password
# is equivalent with
sa_parse "$0" \
         -p/--port arg @doc="Server port" \
         --password arg @doc="Server password"
```
As the name suggests the tokens are _expanded_ to the contents of the corresponding arrays.
This is an advanced feature with some caveats.
* changing the definitions in `common-options.sh` might break scripts using it
* the expand tokens might not be "compatible" with each other.
  In the above example the `--password` option cannot have the alternative short flag `-p` because it would clash with `-p/--port`.

Now you can understand why (or how) every simpleargs based script has `-h` and `--help` flags defined by default.
The default value for `sa_default_tokens` is `( @expand=help )` (as stated in the beginning of the answer),
and simpleargs defines two standard _expand tokens_
```
sa_expand_token_help=( "-h/--help" "@doc=Print usage instructions and exit."
                       "@afterprocessing=sa_display_usage; exit 0" )
sa_expand_token_version=( "--version" "@doc=Display version information."
                          "@afterprocessing=sa_display_version; exit 0" )
```

To include an option for showing the version information of your script try creating and running a new script.
```sh
sa-create-script verbose versiontest
./versiontest --version
```
Notice that the script specifies additional variables `sa_script_long_name`, `sa_script_version`, and `sa_script_build`.
Additionally, the default value for `sa_default_tokens` is overridden to include an expand token
that adds `--version` flag to the script.
```sh
sa_default_tokens=( @expand=help @expand=version )
```

### Integer or float range with a negative minimum does not work.
The definition
```sh
--integer @@int -10..0
```
gives an error message
```sh
ERROR: Invalid flag definition: '-10..0'
```
This is because the range definition `-10..0` starts with a dash.
This makes simpleargs interpret it as an option definition.
One can work around this by using quotes to create a single word:
```sh
--integer "@@int -10..0"
```

### Do I have to use named parameters? Can I use ordinary parameters `$1`, `$2`, ...?
Yes, you can use positional parameters (`$1`, `$2`, `$3`, ...) if that's all you need. Just define your script as
```sh
sa_parse "$0" --my-option --my-value-option value
```
Invocation like `myscript --my-option a.txt b.txt` will work as you'd expect:
* `$1 == a.txt`
* `$2 == b.txt`.

You can also mix named parameters and ordinary positional parameters:
```sh
sa_parse "$0" "<output file>"
```
`myscript output.txt a.txt b.txt` will assign "output.txt" to variable `output_file`
and the rest of the parameters remain in `$1` and `$2`.

However, if you use either type (normal or optional) of varargs parameters
there will (and can) be no positional parameters. That is, defining your script as
```sh
sa_parse "$0" "<input file>..."
```
will place all the script arguments in an array variable `input_file` —
there are no arguments left to be assigned into `$1`, `$2`, ...

### What are the restrictions on parameter order?
This question might arise when facing an error message such as
<!--@eval shell-session-simulator --no-command-print --path $SIMPLEARGS_DOC_SCRIPTS/faq/invalid-parameter-order -c 'myscript' -->
```
ERROR: Invalid token '<output>' (param) after token of type 'optionalparam'
```
The nature of optional and varargs parameters enforces some restrictions on the order of parameter definitions.
The restrictions ensure that the parameters given on command line can be interpreted unambiguously.

1. A varargs parameter can only be the last parameter.
   This implies that there can be no more than one varargs parameter.
2. After an optional parameter (normal or varargs) there can only be other optional parameters.

For example, the first restriction prevents defining your parameters like
```
sa_parse "$0" "<input file>..." "<operation>..."
```
Invoking the script with `myscript data1.txt data2.txt sort filter print` would probably be intended to
sort, filter, and print the contents of files `data1.txt` and `data2.txt` but ultimately that cannot be inferred.

The second restriction prevents for example the following scenario:
```
sa_parse "$0" "[<output>]" "<input>..."
```
What would it mean to use invocation such as `myscript a.txt b.txt`?
Is `a.txt` the optional output or one of the inputs?

One could argue that there is no ambiguity in
```sh
sa_parse "$0" "[<output>]" "<input>"
```
A single argument means that only the input is specified — invocation with two arguments specifies the output as well.
However, this kind of parameter definition is not accepted by simpleargs.

There are two rationales.
First, the same functionality (one required and one optional parameter) can be achieved by changing the order of the parameters.
Secondly, the restrictions allow more advanced features such as
[context sensitive command completion](command-completion.md).

## Installed files
Simpleargs can be installed globally (available to all users) or locally (per user). The files and their locations are described below for both installation types.

Simpleargs installation directory (denoted as `${SA}`) is

* `/usr/lib/simpleargs` for the global installation
* `${HOME}/.simpleargs.d` for local installations

The latter directory is also used in both installation types as the cache directory for user scripts.

The files deployed by the installation:

| File                    | Description |
| ----------------------- | ------------------------------------------- |
| `simpleargs-bundle`     | simpleargs library (sourced by user scripts) |
| `simpleargs-completion` | simpleargs autocompletion library (used by interactive shells) |
| `simpleargs.rc`         | simpleargs bootstrap file: contains bootstrapping code for simpleargs as well as utilities for working with simpleargs e.g. a function for creating simpleargs based scripts |

Additionally, there are configuration files that tweak simpleargs behaviour (e.g. log level).
They are sourced internally by simpleargs library if they exist.

| File                                    | Description |
| --------------------------------------- | ----------- |
| `/usr/lib/simpleargs/simpleargs.conf`   | Global configuration file (only in a global installation) |
| `${HOME}/.simpleargs.d/simpleargs.conf` | Local configuration file |

To enable the interactive functionality of simpleargs its bootstrap file is sourced
from `/etc/bash.bashrc` (global installation) or from `${HOME}/.bashrc` (local installation).
