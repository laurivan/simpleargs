<!-- GENERATED TOC -->
<!-- TOC START -->
## Table of Contents

  - [Automatic Documentation](#automatic-documentation)
    - [Displaying usage instructions with `--help`](#displaying-usage-instructions-with-help)
    - [Automatic man page generation](#automatic-man-page-generation)
<!-- TOC END -->

## Automatic Documentation
The majority of shell scripts are written quickly —
the documentation even more quickly or omitted altogether.
This is certainly partly because most scripts are relatively short-lived and written ad-hoc.
However, at least part of the problem is the lack of documentation mechanisms that are _easy and quick_ to use.

Sure, there are [man pages](https://en.wikipedia.org/wiki/Man_page)
but learning [troff](https://en.wikipedia.org/wiki/Troff) and
creating additional files for one's scripts is not going to cut it in most cases.
That is why simpleargs provides an easy and simple way to add documentation to shell scripts.
The documentation is embedded in the script file itself where it stays close to the source
and is thus more likely to be updated when the script itself is updated.

An example is the best way to illustrate the functionality:
<!--@evalsh filter-simpleargs-script -r 1 $SIMPLEARGS_DOC_SCRIPTS/automatic-documentation/base/myssh -->
```sh
#!/usr/bin/env bash

sa_short_description="uses SSH to connect to a remote server"
sa_long_description=(
"Uses SSH to connect to a remote host with the specified port \
and user. This script is only a wrapper to ssh command."
"This is the second paragraph of the script description. \
The paragraphs are defined as the items of an array \
'sa_long_description'."
)

# -------------------------------- simpleargs --------------------------------
. "${SIMPLEARGS}" || { echo "Error loading '${SIMPLEARGS}'" >&2; exit 1; }
sa_parse "$0" \
         --security-level low,medium,high@default \
         @doc="Security level to use. Allowed values: @{v} \
(default: @{d})." \
         -p/--port port @default=22 \
         @doc="The SSH port of the remote host. Default: @{d}." \
         -u/--user user @default='${USER}' \
         @doc="\
The user used to connect to the remote host. \
By default uses the current user." \
         @doc="More paragraphs can be added using additional \
@doc directives. Only the first paragraph is shown in the \
usage printout (all of them are shown on the man page)." \
         "<remote host>" \
         @doc="The remote host to connect to."

sa_end_parse $?; sa_process "$@"; sa_end_process $?; eval "set -- ${sa_args}"
# ----------------------------------------------------------------------------

echo "sshing: ${user}@${remote_host} (port: ${port}"
# ...
```

A short and a long description of the script are specified as variables before the simpleargs header.
For documenting options and parameters one uses `@doc` directive.
Each _@doc_ directive corresponds to one paragraph on script's man page.
Placeholders _@\{v\}_ and  _@\{d\}_ are expanded to the valid values and the default value of an entry respectively.

### Displaying usage instructions with `--help`
Running the script with `-h` or `--help` displays the script's usage instructions:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/automatic-documentation/base -c 'myssh --help' -->
```
$ myssh --help
Usage: myssh [OPTION]... <remote host>
Uses SSH to connect to a remote host with the specified port and user. This
script is only a wrapper to ssh command.

Parameters:
  <remote host>                 The remote host to connect to.

Options:
  -h, --help                    Print usage instructions and exit.
  --security-level ARG          Security level to use. Allowed values: 'low',
                                'medium', and 'high' (default: high).
  -p ARG, --port ARG            The SSH port of the remote host. Default: 22.
  -u ARG, --user ARG            The user used to connect to the remote host. By
                                default uses the current user.
```

The `-h/--help` option is something that isn't present in our script example.
See [this FAQ](faq-and-tips.md#adding-h-does-not-work-or-how-do-i-disable-the-h-help-option)
for more information about how it is implemented.
In short, technically enabling the help flag is equivalent to adding the following option definition:
```sh
-h/--help @doc="Print usage instructions and exit." \
          @afterprocessing="sa_display_usage; exit 0"
```
That is, the usage instructions are printed by simpleargs built-in function `sa_display_usage`.

The usage instructions can also be displayed when the script is used incorrectly.
This is done by modifying the value of `sa_process_failure_action` variable:
* **none**  
  Only print the error message (e.g. "unrecognized option").
* **display-synopsis**  
  In addition to the error message display the script synopsis. This is the default value.
* **display-usage**  
  In addition to the error message display the usage instructions (which includes the synopsis).

So, by default mistyping a command will look like this:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/automatic-documentation/print-synopsis -c 'myssh --prot 45 example.com' -->
```
$ myssh --prot 45 example.com
ERROR: Processing arguments failed: unrecognized option `--prot'
Usage: myssh [OPTION]... <remote host>
```

If one wants only the error message to be printed
one can set the variable before simpleargs header...
<!--@evalsh filter-simpleargs-script -r 1 -r /sa_process_failure_action/ $SIMPLEARGS_DOC_SCRIPTS/automatic-documentation/no-synopsis-on-failure/myssh -->
```sh
#!/usr/bin/env bash

sa_process_failure_action=none
```
...and get more concise output:
<!--@eval shell-session-simulator --path $SIMPLEARGS_DOC_SCRIPTS/automatic-documentation/no-synopsis-on-failure -c 'myssh --prot 45 example.com' -->
```
$ myssh --prot 45 example.com
ERROR: Processing arguments failed: unrecognized option `--prot'
```


### Automatic man page generation
Also, man page is generated for the script when
* it is run for the first time
* it is run after the script has been modified

Opening the man page works as with other commands: `man myssh`.
<!--@evalonly shell-session-simulator --no-command-print --path $SIMPLEARGS_DOC_SCRIPTS/automatic-documentation/base -c 'sa_parse_only=true myssh' -->
<!--@evalonly MANPATH=~/.simpleargs.d/bin/man MANWIDTH=80 man myssh | col -bx > /tmp/myssh-manpage.txt -->
<!--@eval cat /tmp/myssh-manpage.txt -->
```
MYSSH(1)                         User Scripts                         MYSSH(1)



NAME
       myssh - uses SSH to connect to a remote server

SYNOPSIS
       myssh [OPTION]... <remote host>

DESCRIPTION
       Uses  SSH to connect to a remote host with the specified port and user.
       This script is only a wrapper to ssh command.

       This is the second paragraph of the script description. The  paragraphs
       are defined as the items of an array 'sa_long_description'.

OPTIONS
       -h, --help
              Print usage instructions and exit.


       --security-level ARG
              Security  level  to  use.  Allowed  values: 'low', 'medium', and
              'high' (default: high).


       -p ARG, --port ARG
              The SSH port of the remote host. Default: 22.


       -u ARG, --user ARG
              The user used to connect to the remote host. By default uses the
              current user.

              More  paragraphs  can be added using additional @doc directives.
              Only the first paragraph is shown in the usage printout (all  of
              them are shown on the man page).




myssh                                                                 MYSSH(1)
```

**NOTE**  
The generated man pages are stored in `~/.simpleargs.d/bin/man`.
They are available in `MANPATH` because `~/.simpleargs.d/bin` is added to `PATH` by `simpleargs.rc`
(see `man manpath`, `man 5 manpath`, and `manpath -d` for more details).
