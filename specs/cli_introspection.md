---
title: UAPI.X CLI-Introspection Specification
category: Interfaces
layout: default
version: 0.0
SPDX-License-Identifier: CC-BY-4.0
weight: 1
aliases:
- /UAPI.18
- /18
---

# UAPI.X The CLI-Introspection Specification

| Version | Changes |
|---------|---------|
| 0.0     | WIP     |

This document defines a JSON schema defining a machine-readable equivalent of
what programs commonly output when they are called with the `--help` or an
equivalent option.

This output can be used,
among other things,
to check whether a program has a given option or to automatically generate
command-line completion for different shells.

## Target Audience

The target audience for this specification are:

* People writing runnable programs that have command-line interfaces,
* authors of tab-completion helpers, and
* people interested in automatic testing of documentation and self-documenting
  interfaces.

## The Schema

The schema is defined as

```json
{
  "mediaType" : "application/vnd.uapi-group.cli-introspection",
  "commands": [<command object>, …]
}
```

where `mediaType` is the fixed string
`application/vnd.uapi-group.cli-introspection-0` and `commands` is a non-empty
JSON array of *command objects*.

Further additions to this specification will only be additive and non-backward
compatible changes may only be introduced by defining a new `mediaType`.

The `commands` array defines the actual CLI and a single binary can define
arbitrarily many CLIs,
but must define at least one.
Defining multiple CLIs is relevant for multicall binaries that behave
differently depending on with what name they are called,
e.g the basename of `argv[0]`,
`program_invocation_short_name` in the GNU system
context,
or in the context of scripting languages the name under which they are imported

### Nomenclature

A *CLI* is the textual interface that a program presents to a user.
It consists of the program being called under a name followed by units that are
separated by whitespace,
these units are called *arguments*.

Arguments may be single words or multiple words that may be escaped in some
way in the case that they contain whitespace themselves,
that is not meant to separate them from other arguments.

Arguments describe the inputs to a program on the CLI.
There are two types of arguments *positional arguments* and *optional arguments*.

Positional arguments have a fixed position in relation to each other and to
optional arguments.

A special kind of positional argument is one that describes a command of its own.
These will be called *verbs*,
but are also known as subcommands.

Optional arguments are also called options,
which they will be called here for conciseness,
as well as switches.

### Command objects

The command object is recursively defined as follows.

```json
{
  "type": "command"
  "names" : ["name1", "name2", …],
  "version": ["myversion"],
  "features": ["feature1", "feature2", …]
  "abstract": ["paragraph1", "paragraph2", …],
  "postscript": ["paragraph1", "paragraph2", …],
  "help": "short help"
  "arguments": [<option|argument|command>, …],
  "documentation": ["doc1", …],
  "project": "myproject",
  "isDeprecated": false
}
```

All keys except `type` and `names` are optional and are treated as empty string
or empty array when missing.

`type` is the fixed string `command` and signals that this is a command object,
describing either a top-level command or verb.

`names` is a non-empty array of string names for that command.
The first element of that array is the primary name of that command.
Further names can be added as aliases,
e.g. for backward compatibility.

`version` is a an array describing the version of the program.
The strings in this array should be compatible with the UAPI.10 Version Format
Specification-compatible.

`features` defines an array of strings that define builtin-in features of this
program.

`abstract` and `postscript` define arrays of strings that are shown respectively
before and after the description of options in the program's help output.
Each element of the array represents a paragraph of text.
Both are meant for standalone help output, e.g. for the top-level help output of
a program or specific help output of a verb in cases where they have their own
help output.

`help` defines the help text of the command. This is useful for single-line
usage information as well as for short descriptions of verbs in the list of
options.

`arguments` defines a CLI's positional and optional arguments and are described
below.
It consists of *option objects*, *argument objects*, and *command  objects*.
The depth of this object,
since command objects in this array may themselves define command objects in
their `arguments`,
may not exceed 15.

`documentation` is an array of string values describing URIs referencing
documentation for this command,
see `man:uri(7)`for a description of valid URIs.
Preferably the URIs should be URLs starting with `https://` as these are widely
supported in modern terminals.

`project` is a string describing what this command belongs to.
This may the package that installed it or the project that produced it.

`isDeprecated` is a boolean describing whether the command has been

### Argument objects

Argument objects describe positional arguments that are not verbs.

```json
{
  "type": "argument"
  "name": "filename",
  "value_name": "FILE",
  "help": "Show this help",
  "sections": ["Arguments"],
  "values": [<value object>],
  "isDeprecated": false
}
```

All keys except `type`, `name` and `argument` are optional and are treated as
empty when missing.

`type` is the fixed string `argument` and signals that this is an argument object,
describing a positional argument.

`names` is a non-empty string defining the name of the argument.

`argument` defines the argument type, which is one of the strings:
-`required_argument`, or
-`optional_argument`.
An argument object whose `argument` value is `required_argument` must be
passed an argument,
while an argument of whose `argument` value is `optional_argument` may be omitted.

`help` is a string that defines the help text of the argument.

`sections` is an array of string that defines sections in which this option should
be shown.
This is only for display-purposes.

`values` is an array of *value objects* describing the values this option may
take.
An empty array describes an argument that is a single arbitrary word with no
further documented semantic.
Value objects are described in a section below.

`isDeprecated` is a boolean describing whether this option has been deprecated.

### Option Objects

Option objects describe optional arguments.

```json
{
  "type": "option"
  "names": ["-h","--help"],
  "argument": "no_argument",
  "help": "Show this help",
  "sections": [""],
  "values": [<value object>],
  "isDeprecated": false
}
```

All keys except `type`, `names` and `argument` are optional and are treated as
empty when missing.

`type` is the fixed string `option` and signals that this is an option object,
describing an optional argument.

`names` is a non-empty array of strings defining the name of an option.
Names starting with dashes define options and names not starting with dashes
define positional arguments.
An option object may define only options or arguments,
but not both for the same object,
i.e. an option object may not have names both with and without dashes.
Options prefixed with a single dash (`-`) are called short options and options
prefixed with two dashes (`--`) are called long options.
Short options are usually followed by a single character,
whereas long options can be a longer string.
Short options may be followed by multiple characters,
but this is discouraged.

`argument` defines the argument type, which is one of the strings:
-`no_argument`,
-`required_argument`, or
-`optional_argument`.
An argument object whose `argument` value is `no_argument` cannot be passed an
argument,
an argument object whose `argument` value is `required_argument` must be
passed an argument,
and an argument of whose `argument` value is `optional_argument` may be omitted.

`help` is a string that defines the help text of the option.

`sections` is an array of string that defines sections in which this option should
be shown.
This is only for display-purposes.

`values` is an array of *value objects* describing the values this option may
take.
An empty array describes an argument that is a single arbitrary word with no
further documented semantic.
Value objects are described in a section below.

`isDeprecated` is a boolean describing whether this option has been deprecated.

### Value objects

Value objects describe a value that is passed as an argument,
either to a positional argument or an optional argument.

```json
{
  "type": "value",
  "value": "myname",
  "help": "help text",
  "default": true,
  "isDeprecated": false
}
```

All keys except `type` and one of `value`, `category`, `dynamic` or `missing`
are optional and are treated as empty string or false when missing.

`type` is the fixed string `value` and signals that this is a value object,
describing an argument value.

`value` is a string describing a possible static value.

`category` is a string describing a category of values:
- `path`, any filesystem path,
- `file`, a filesystem path to a regular file,
- `dir`, a filesystem path to a directory,
- `pid`, a PID,
- `uid`, a numeric UID,
- `gid`, a numeric GID,
- `username`, a user's name,
- `groupname`, a group's name,
- `hostname`, a hosts' name, and
- `unit`, a systemd unit's name.

`dynamic` is a non-empty string describing a commands,
that can be called to generate multiple values.
This is relevant for the dynamic generation of completion candidates during
command line completion.
The current input for the argument,
if any,
otherwise an empty string will be passed as first and only argument.
The standard output stream of the command defines the values,
one per line.

`missing` is a boolean signalling that some values are missing from the
description.

`help` is a string describing the help text that should be shown for the value.

`default` is a boolean describing whether this value is the default value for
the option this is a value for.

If multiple of `value`, `dynamic` and `missing` are defined,
`missing` has the highest precedence, followed by `value` and `dynamic`.

## Example

This is an example for a description for `systemd-id128`, with the help output

```console
$ systemd-id128 --help
> systemd-id128 [OPTION…] COMMAND …

Generate and print 128-bit identifiers.

Commands:
  new                  Generate a new ID
  machine-id           Print the ID of current machine
  boot-id              Print the ID of current boot
  invocation-id        Print the ID of current invocation
  var-partition-uuid   Print the UUID for the /var/ partition
  show [NAME|UUID]     Print one or more UUIDs
  help                 Show this help

Options:
  -h --help            Show this help
     --version         Show package version
     --no-pager        Do not start a pager
     --no-legend       Do not show headers and footers
     --json=FORMAT     Output inspection data in JSON (takes one of pretty, short, off)
  -j                   Equivalent to --json=pretty (on TTY) or --json=short (otherwise)
  -p --pretty          Generate samples of program code
  -P --value           Only print the value
  -a --app-specific=ID Generate app-specific IDs
  -u --uuid            Output in UUID format

See the systemd-id128.1 man page for details.
```

The resulting CLI introspection JSON would be

```json
{
  "mediaType": "application/vnd.io.systemd.cli-introspection-0",
  "commands": [
    {
      "type": "command",
      "names": [
        "systemd-128"
      ],
      "version": [
        "262",
        "262~devel"
      ],
      "features": [
        "PAM",
        "+AUDIT",
        "-SELINUX",
        "+APPARMOR",
        "-IMA",
        "+IPE",
        "+SMACK",
        "+SECCOMP",
        "+GCRYPT",
        "+GNUTLS",
        "+OPENSSL",
        "+ACL",
        "+BLKID",
        "+CURL",
        "+ELFUTILS",
        "+FIDO2",
        "+IDN2",
        "+KMOD",
        "+LIBCRYPTSETUP",
        "+LIBCRYPTSETUP",
        "PLUGINS",
        "+LIBFDISK",
        "+PCRE2",
        "+PWQUALITY",
        "+P11KIT",
        "+QRENCODE",
        "+TPM2",
        "+BZIP2",
        "+LZ4",
        "+XZ",
        "+ZLIB",
        "+ZSTD",
        "+BPF",
        "FRAMEWORK",
        "+BTF",
        "+XKBCOMMON",
        "+UTMP",
        "+LIBARCHIVE"
      ],
      "documentation": [
        "https://www.freedesktop.org/software/systemd/man/latest/systemd-id128.html",
        "man:systemd-id128(1)"
      ],
      "abstract": [
        "Generate and print 128-bit identifiers."
      ],
      "postscript": [
        "See the systemd-id128(1) man page for details."
      ],
      "arguments": [
        {
          "type": "option",
          "names": [
            "-h",
            "--help"
          ],
          "argument": "no_argument",
          "sections": [
            "Options"
          ],
          "help": "Show this help"
        },
        {
          "type": "option",
          "names": [
            "--version"
          ],
          "argument": "no_argument",
          "sections": [
            "Options"
          ],
          "help": "Show package version"
        },
        {
          "type": "option",
          "names": [
            "--no-pager"
          ],
          "argument": "no_argument",
          "sections": [
            "Options"
          ],
          "help": "Do not start a pager"
        },
        {
          "type": "option",
          "names": [
            "--no-legend"
          ],
          "argument": "no_argument",
          "sections": [
            "Options"
          ],
          "help": "Do not show headers and footers"
        },
        {
          "type": "option",
          "names": [
            "--json"
          ],
          "argument": "required_argument",
          "value_name": "FORMAT",
          "sections": [
            "Options"
          ],
          "help": "Output inspection data in JSON (takes one of pretty, short, off)",
          "values": [
            {
              "type": "value",
              "value": "short",
              "help": "the shortest possible output without any redundant whitespace or line breaks"
            },
            {
              "type": "value",
              "value": "pretty",
              "help": "a pretty version of the same"
            },
            {
              "type": "value",
              "value": "off",
              "help": "no JSON output",
              "default": true
            }
          ]
        },
        {
          "type": "option",
          "names": [
            "-j"
          ],
          "argument": "no_argument",
          "sections": [
            "Options"
          ],
          "help": "Equivalent to --json=pretty (on TTY) or --json=short (otherwise)"
        },
        {
          "type": "option",
          "names": [
            "-p",
            "--pretty"
          ],
          "argument": "no_argument",
          "sections": [
            "Options"
          ],
          "help": "Generate samples of program code"
        },
        {
          "type": "option",
          "names": [
            "-P",
            "--value"
          ],
          "argument": "no_argument",
          "sections": [
            "Options"
          ],
          "help": "Only print the value"
        },
        {
          "type": "option",
          "names": [
            "-a",
            "--app-specific"
          ],
          "value_name": "ID",
          "argument": "required_argument",
          "sections": [
            "Options"
          ],
          "help": "Generate app-specific IDs"
        },
        {
          "type": "option",
          "names": [
            "-u",
            "--uuid"
          ],
          "sections": [
            "Options"
          ],
          "help": "Output in UUID format"
        },
        {
          "type": "command",
          "name": [
            "new"
          ],
          "help": "Generate a new ID"
        },
        {
          "type": "command",
          "name": [
            "machine-id"
          ],
          "value_name": "COMMAND",
          "sections": [
            "Commands"
          ],
          "help": "Print the ID of current machine"
        },
        {
          "type": "command",
          "name": [
            "boot-id"
          ],
          "value_name": "COMMAND",
          "sections": [
            "Commands"
          ],
          "help": "Print the ID of current boot"
        },
        {
          "type": "command",
          "name": [
            "invocation-id"
          ],
          "value_name": "COMMAND",
          "sections": [
            "Commands"
          ],
          "help": "Print the ID of current invocation"
        },
        {
          "type": "command",
          "name": [
            "var-partition-uuid"
          ],
          "value_name": "COMMAND",
          "sections": [
            "Commands"
          ],
          "help": "Print the UUID for the /var/ partition"
        },
        {
          "type": "command",
          "name": [
            "show"
          ],
          "value_name": "COMMAND",
          "sections": [
            "Commands"
          ],
          "arguments": [
            {
              "type": "argument",
              "name": "name_or_uuid",
              "argument": "optional_argument"
              "value_name": "NAME|UUID"
            }
          ],
          "help": "Print one or more UUIDs"
        },
        {
          "type": "command",
          "name": [
            "help"
          ],
          "value_name": "COMMAND",
          "sections": [
            "Commands"
          ],
          "help": "Show this help"
        }
      ]
    }
  ]
}
```
