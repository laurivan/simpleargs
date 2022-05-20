<!-- GENERATED TOC -->
<!-- TOC START -->
## Table of Contents

  - [Getting Started](#getting-started)
    - [Example Script](#example-script)
    - [Adding Functionality](#adding-functionality)
    - [More basics](#more-basics)
    - [Beyond the basics](#beyond-the-basics)
    - [Advanced Features](#advanced-features)
<!-- TOC END -->

## Getting Started
### Example Script
Instead of creating the traditional `Hello World` program we shall develop a script that
is more versatile in its greetings.
Our script should support options and parameters to allow invocations like
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/10-first-definitions -c 'greet --greeting "Hey" "Alex"' -->
```
$ greet --greeting "Hey" "Alex"
Hey, Alex!
```

First, create a new text file named `greet`.
Give it execution permission and place it in a directory that is in your `PATH`.
For example,
```
$ touch greet; chmod u+x greet; mv greet ~/bin
```
Insert the following code into the newly created file:
<!--@evalsh cat $SIMPLEARGS_DOC_SCRIPTS/getting-started/10-first-definitions/greet -->
```sh
#!/usr/bin/env bash

# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }     #1
sa_parse "$0" --greeting arg "<person>"                                        #2
sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"  #3
# ----------------------------------------------------------------------------
echo "${greeting}, ${person}!"                                                 #4
```
The file now consists of
* [shebang line](https://en.wikipedia.org/wiki/Shebang_(Unix)) (the first line)
* _simpleargs header_ (the lines enclosed by and including the comment lines)
* _user code_ section (echo statement)

Referring to the numbering at the end of the lines:
1. Sources the library (or prints an error message and terminates the script in case of a failure).
   The value of `${SIMPLEARGS}` is set in `~/.bashrc` (where it was added by the installation).
2. Parses the option and parameter definitions. The first argument to `sa_parse` is always `"$0"`.
3. Finalizes the parsing and does the actual processing of the arguments given to the script.
4. The user code comes after the header. In this case prints a parameterized greeting.

The call to `sa_parse` is the only part of the simpleargs header that changes from script to script.
Notice the resemblance between the script invocation and call to `sa_parse`:
```sh
   greet      --greeting "Hey"   "Alex"
sa_parse "$0" --greeting  arg  "<person>"
```
<!--
The option value and the positional parameter have been replaced with _keyword_ `arg` and _placeholder_ `"<person>"`.
This is the basic pattern of developing `simpleargs` based scripts:
an example of the desired script invocation is transformed into generic form and inserted into the simpleargs header.
-->

When the script is invoked simpleargs processes the arguments and assigns them to variables.
The variable names are derived from the definitions given to `sa_parse`:
- `--greeting` -> `${greeting}`
- `"<person>"` -> `${person}`

Consequently, the user part of the script is reduced into a single `echo` statement printing out a string containing the variables.
With very little effort we have implemented a compact shell script that accepts command line options.
There is even some rudimentary error handling:
<!--@evalsh shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/10-first-definitions -c 'greet --greeting "Hello" "World"' -c 'greet --greeting "Good morning" "Jeff"' -c 'greet --greeting "Good evening"' -->
```sh
$ greet --greeting "Hello" "World"
Hello, World!
$ greet --greeting "Good morning" "Jeff"
Good morning, Jeff!
$ greet --greeting "Good evening"
ERROR: Missing required parameter <person>
Usage: greet [OPTION]... <person>
```

> **_TIP_**
You can use a helper utility for creating script templates.
This saves you the trouble of copy-pasting from this web page or from another script.
Try it! In your terminal run `sa-create-script -h` to display the usage instructions and then `sa-create-script myscript` to create a script named `myscript`.

### Adding Functionality
The script is still far away from perfect. What if someone forgets to specify the greeting?
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/10-first-definitions -c 'greet "Jeff"' -->
```
$ greet "Jeff"
, Jeff!
```
Oops! There should be a default greeting if none is specified in the invocation.
This can be achieved by complementing the definition of `--greeting` option.
For this purpose simpleargs recognizes a set of [_directives_](directives.md)  that start with `@` character.
To add a default value for an option one uses [@default](directives.md#default) directive:
<!--@eval filter-simpleargs-script -r /sa_parse/ -r e// $SIMPLEARGS_DOC_SCRIPTS/getting-started/15-adding-default-greeting/greet -->
```
sa_parse "$0" --greeting arg @default="Hello" "<person>"
```
Now we can omit the greeting.
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/15-adding-default-greeting -c 'greet "Jeff"' -->
```
$ greet "Jeff"
Hello, Jeff!
```

To validate the inputs of the script there are [validation directives](validation.md)
that start with **two** `@` characters.
For example, to ensure that a valid name is given to the script
we can use an [extended regular expression](validation.md#egrep):
<!--@evalsh filter-simpleargs-script -r "/<person>/" -r e// --trim-lines $SIMPLEARGS_DOC_SCRIPTS/getting-started/30-adding-validation/greet -->
```sh
"<person>" @@egrep "^[A-Z][a-z]+$"
```
Now `person` parameter accepts only values with a single uppercase letter followed by one or more lowercase letters.
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/30-adding-validation -c 'greet --greeting "Hi" "jo"' -c 'greet --greeting "Hi" "Jo"' -->
```
$ greet --greeting "Hi" "jo"
ERROR: <person>: 'jo' does not match extended regular expression '^[A-Z][a-z]+$'
Usage: greet [OPTION]... <person>
$ greet --greeting "Hi" "Jo"
Hi, Jo!
```

### More basics
How about options that do not take a value?
Or using varying number of positional parameters?
Examine the updated version of the script below.
<!--@evalsh filter-simpleargs-script -r 1 $SIMPLEARGS_DOC_SCRIPTS/getting-started/40-small-talk/greet -->
```sh
#!/usr/bin/env bash

# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" \
         -g/--greeting arg @default="Hello" \
         --small-talk \
         "[<person>]..." @default=World @@egrep "^[A-Z][a-z]+$" @varname=people
sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
# ----------------------------------------------------------------------------
for person in "${people[@]}"
do
    echo "${greeting}, ${person}!"
done
${small_talk} && echo "Such nice weather, isn't it?"
```
Let's go through the changes one by one:

* To make the option and parameter definitions more readable they are divided onto multiple lines.
  A backslash character `\ ` is used to make the newline treated as [a line continuation](https://www.gnu.org/software/bash/manual/bashref.html#Escape-Character).
  This does not change the parsing in any way.
  Directives affect the preceding option or parameter up until the next option or parameter definition.

* The greeting can now be specified with either the short flag `-g` or the long flag `--greeting`.
  The value of the option is assigned always to both variables: `${g}` and `${greeting}`.
  <!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/40-small-talk -c 'greet -g "Hey" "Jim"' -c 'greet --greeting "Hey" "Jim"' -->
  ```
  $ greet -g "Hey" "Jim"
  Hey, Jim!
  $ greet --greeting "Hey" "Jim"
  Hey, Jim!
  ```

* There is a new option `--small-talk` which is not followed by `arg` keyword.  This implies a [_no-value option_](options-and-parameters.md#no-argument-option).
  The corresponding variable is `${small_talk}` (dashes in the flag name are replaced with underscores: `-` -> `_`).
  It is assigned the value `true` or `false` depending on whether the option is given on the command line.
  The variable is used to control whether to print a statement about the weather.
  <!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/40-small-talk -c 'greet "Jim"' -c 'greet --small-talk "Jim"' -->
  ```
  $ greet "Jim"
  Hello, Jim!
  $ greet --small-talk "Jim"
  Hello, Jim!
  Such nice weather, isn't it?
  ```

* To allow greeting multiple people with a single invocation the definition for the positional parameter has been changed into `"[<person>]..."`.
  The square brackets denote an _optional parameter_.
  The three dots mean that there can be more than one parameter value.
  This implies that the command line parameters should be assigned into an array rather than an ordinary variable.

  Finally, [@varname directive](directives.md#varname) is used to specify the variable name (of the array) explicitly.
  Otherwise, the array variable name would have been `person` which can now be used as the loop variable in the user code when printing the greetings to each person.

  If there are no positional parameters in the script invocation
  the array will contain only one value `World` (the default).
  Validation of the values is done for each parameter separately.

  <!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/40-small-talk -c 'greet' -c 'greet "Jim" "Jack" "Jeff"' -c 'greet "Jim" "mo" "Tom"' -->
  ```
  $ greet
  Hello, World!
  $ greet "Jim" "Jack" "Jeff"
  Hello, Jim!
  Hello, Jack!
  Hello, Jeff!
  $ greet "Jim" "mo" "Tom"
  ERROR: [<person>]...: 'mo' does not match extended regular expression '^[A-Z][a-
  z]+$'
  Usage: greet [OPTION]... [<person>]...
  ```

### Beyond the basics
Let's add two more pieces of functionality:
an option for specifying what kind of weather it is and another one for specifying one or more people that accompany us.
<!--@eval filter-simpleargs-script -r 1 $SIMPLEARGS_DOC_SCRIPTS/getting-started/50-weather-type-and-friends/greet -->
```
#!/usr/bin/env bash

# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" \
         -g/--greeting arg @default="Hello" \
         --small-talk \
         --weather sunny,cloudy,rainy,foggy,mysterious@default \
         -f/--my-friend arg @@egrep "^[A-Z][a-z]+$" @varname=my_friends \
         @multivalue: \
         "[<person>]..." @default=World @@egrep "^[A-Z][a-z]+$" @varname=people
sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
# ----------------------------------------------------------------------------

for person in "${people[@]}"
do
    echo "${greeting}, ${person}!"
done
case ${#my_friends[@]} in
    0) echo "Today I'm utterly alone.";;
    1) echo "Let me introduce my friend, ${my_friends[0]}." ;;
    2) echo "Here are my friends, ${my_friends[0]} and ${my_friends[1]}." ;;
    *) echo "Here are my ${#my_friends[@]} friends." ;;
esac

${small_talk} && echo "Such ${weather} weather, isn't it?"
```
The script gets more versatile but grows in length by only a couple of lines.

* The first addition is for specifying the weather type.
  The token (i.e. word) after `--weather` is something new.
  A comma separated list after a flag name specifies the values that are valid for the option.
  Furthermore, string `@default` appended to one of them denotes the default value (`mysterious`).

  What if the value itself contains a comma `,` or `@`?
  The same behaviour can be achieved by using directives:
  ```
  --weather arg @validvalues%=sunny%cloudy%rainy%foggy%mysterious @default=mysterious
  ```
  [@validvalues](directives.md#validvalues) is used for specifying the valid values
  and [@default](directives.md#default) the default value for the option.
  Percent `%` sign is used as the delimiter for the valid values.

  The following sections will cover all the directives and their syntax in more detail.
  For now it is enough to know that directives consist of _name_ (e.g. _validvalues_),
  optionally a _value_ (after the equals sign) and sometimes a one-character modifier
  right before the equals sign.

  As usual the value of the option is assigned to `${weather}` which is used on
  the last line of the script to customize the small talk statement.
  <!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/50-weather-type-and-friends -c 'greet --small-talk --weather cloudy Jim' -c 'greet --small-talk --weather stormy Jim' -->
  ```
  $ greet --small-talk --weather cloudy Jim
  Hello, Jim!
  Today I'm utterly alone.
  Such cloudy weather, isn't it?
  $ greet --small-talk --weather stormy Jim
  ERROR: --weather: invalid value 'stormy'
  Valid values are: 'sunny', 'cloudy', 'rainy', 'foggy' and 'mysterious'.
  Usage: greet [OPTION]... [<person>]...
  ```

* The other addition is `-f/--my-friend` option.
  You are already familiar with most of the tokens affecting this option.
  Keyword `arg` implies that this option takes a value.
  Validation directive [@@egrep](validation.md#egrep) makes the option accept only values that
  consist of an uppercase letter followed by one or more lowercase letters.
  `@varname` sets an explicit variable name for the option.

  However, on its own line [@multivalue:](directives.md#multivalue) is something new.
  This directive means that the option can take multiple values.
  In other words, the flag can specified more than in a single invocation.
  Colon `:` has been set as a delimiter character.
  Since the flag has short (`-f`) and long (`--my-friend`) versions the option can be used in various ways.

  The following invocations are equivalent:
  ```
  greet -f Jim -f Jack -f John
  greet --my-friend Jim --my-friend Jack -f John
  greet -f Jim:Jack:John
  greet --my-friend Jim:Jack -f John
  ```

  All of them result in the same printout:
  <!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/50-weather-type-and-friends -c 'greet -f Jim:Jack:John' -->
  ```
  $ greet -f Jim:Jack:John
  Hello, World!
  Here are my 3 friends.
  ```

  As can be inferred from the source code the names of the friends are assigned to
  an array named `my_friends` (specified with [@varname](directives.md#varname) directive).
  Depending on the number of friends a different statement is printed.
  <!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/50-weather-type-and-friends -c 'greet -f Alex:Jenny' -c 'greet' -->
  ```
  $ greet -f Alex:Jenny
  Hello, World!
  Here are my friends, Alex and Jenny.
  $ greet
  Hello, World!
  Today I'm utterly alone.
  ```

### Advanced Features
When the script is run there are two steps taken by simpleargs.
First the option and parameter definitions are parsed (by `sa_parse`).
Secondly the script arguments are processed (by `sa_process`).

The script arguments vary from invocation to invocation but since
the option and parameter definitions rarely change the result of the parsing is applicable for reuse.
Hence, simpleargs caches the parsing result under `~/.simpleargs.d/cached` directory.
Modifying the script will make its timestamp newer than the saved parsing result which invalidates the cache:
the next invocation parses the definitions and refreshes the cache.

Using the cache speeds up the invocations since the definitions need to be parsed only after the script has been modified.
An additional benefit is the possibility to utilize the cached data for command completion.

To try out the feature first run the following commands:
```
greet
sa-refresh-completion
```
Running `greet` ensures that the cache is refreshed.
The second command updates Bash completion routines for the script.
Then type in the script name and a dash, and then use `tab` key to trigger the auto-completion:
```
$ greet -<tab><tab>
-f            --greeting    --help        --small-talk
-g            -h            --my-friend   --weather
$ greet --s<tab>
$ greet --small-talk --w<tab>
$ greet --small-talk --weather <tab><tab>
cloudy      foggy       mysterious  rainy       sunny
$ greet --small-talk --weather s<tab>
```
And finally hit enter:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/50-weather-type-and-friends -c 'greet --small-talk --weather sunny' -->
```
$ greet --small-talk --weather sunny
Hello, World!
Today I'm utterly alone.
Such sunny weather, isn't it?
```

With simpleargs it is also straightforward to add documentation to your scripts.
A short and a long description of the script are specified as variables before the simpleargs header.
For documenting options and parameters one uses [@doc](directives.md#doc) directive.
<!--@evalsh filter-simpleargs-script -r 1 -r '/# -+$/' $SIMPLEARGS_DOC_SCRIPTS/getting-started/90-final/greet -->
```sh
#!/usr/bin/env bash

sa_short_description="greet people"
sa_long_description=(
"Greet people with customizable greeting and optional small talk. \
Additionally, you can introduce your friends to the people you are greeting" )

# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }

sa_parse "$0" \
         -g/--greeting arg @@egrep "^[A-Z][a-z]+( [a-z]+)?$" @default=Hello \
         @doc="Greeting to print (default: '@{d}')." \
         @doc="Greeting can be a single word starting with a capital \
letter (e.g. 'Hey') or consist of two words (e.g. 'Good morning')." \
         --small-talk \
         @doc="In addition to greeting people exercise some small talk \
 about the weather. See also --weather." \
         --weather sunny,cloudy,rainy,foggy,mysterious@default \
         @doc="Weather type to be used in small talk (enabled using \
--small-talk). Allowed weather types: @{v} (default: '@{d}')." \
         -f/--my-friend arg @@egrep "^[A-Z][a-z]+$" @varname=my_friends \
         @multivalue: \
         @doc="One or more friends in your entourage." \
         "[<person>]..." @default=World @@egrep "^[A-Z][a-z]+$" \
         @varname=people \
         @doc="One or more people to greet. By default greet only '@{d}'."

sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
# ----------------------------------------------------------------------------
```

Each `@doc` directive corresponds to one paragraph on script's man page.
Placeholders `@{v}` and `@{d}` are expanded to entry's valid values and default value respectively.
Running `greet -h` or `greet --help` prints the usage instructions:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/90-final -c 'greet -h' -->
```
$ greet -h
Usage: greet [OPTION]... [<person>]...
Greet people with customizable greeting and optional small talk. Additionally,
you can introduce your friends to the people you are greeting

Parameters:
  [<person>]...                 One or more people to greet. By default greet
                                only 'World'.

Options:
  -h, --help                    Print usage instructions and exit.
  -g ARG, --greeting ARG        Greeting to print (default: 'Hello').
  --small-talk                  In addition to greeting people exercise
                                some small talk  about the weather. See also
                                --weather.
  --weather ARG                 Weather type to be used in small talk (enabled
                                using --small-talk). Allowed weather types:
                                'sunny', 'cloudy', 'rainy', 'foggy', and
                                'mysterious' (default: 'mysterious').
  -f ARG1:ARG2:..., --my-friend ARG1:ARG2:...
                                One or more friends in your entourage.
```

Man page can be displayed using `man greet`:
<!--@evalonly shell-session-simulator --no-command-print --path $SIMPLEARGS_DOC_SCRIPTS/getting-started/90-final -c 'sa_parse_only=true greet -h' -->
<!--@eval shell-session-simulator --no-command-print -c 'MANWIDTH=80 man greet | col -bx' -->
```
GREET(1)                         User Scripts                         GREET(1)



NAME
       greet - greet people

SYNOPSIS
       greet [OPTION]... [<person>]...

DESCRIPTION
       Greet  people with customizable greeting and optional small talk. Addi-
       tionally, you can introduce your friends to the people you are greeting

OPTIONS
       -h, --help
              Print usage instructions and exit.


       -g ARG, --greeting ARG
              Greeting to print (default: 'Hello').

              Greeting  can  be  a  single word starting with a capital letter
              (e.g. 'Hey') or consist of two words (e.g. 'Good morning').


       --small-talk
              In addition to greeting people exercise some small  talk   about
              the weather. See also --weather.


       --weather ARG
              Weather  type  to  be used in small talk (enabled using --small-
              talk).  Allowed  weather  types:  'sunny',  'cloudy',   'rainy',
              'foggy', and 'mysterious' (default: 'mysterious').


       -f ARG1:ARG2:..., --my-friend ARG1:ARG2:...
              One or more friends in your entourage.




greet                                                                 GREET(1)
```

<!-- TODO: continue here: add "where from here" -->
