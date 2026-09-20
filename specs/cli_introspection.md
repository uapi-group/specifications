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
`application/vnd.uapi-group.cli-introspection` and `commands` is a non-empty
JSON array of *command objects*.

Further additions to this specification will only be additive and non-backward
compatible changes may only be introduced by defining a new `mediaType`.

The `commands` array defines the actual command line interface.
A single binary can define arbitrarily many commands,
but must define at least one.
Defining multiple commands in a single program is relevant for multicall programs
that behave differently depending on the name with which name they are called,
e.g the basename of `argv[0]`,
`program_invocation_short_name` in the GNU system context,
or the name under which they are imported in case of scripting languages.

### Nomenclature

A **command** is the interface the program presents to a user
when called under a specific name.

The items in the argument array
(`argv` in [execve(2)](https://man7.org/linux/man-pages/man2/execve.2.html))
after the program name are called **arguments**.

The arguments may contain whitespace, quotes, and other characters that are special to the shell.
How the command typed by the user is split by the shell,
or more generally how the argument array is constructed in other cases,
is outside of the scope of this specification,
which only deals with the argument array as it is received by the program.

This specification divides the arguments into two categories:
- **positional arguments** are identified by their position in relation to other arugments,
- **options** are specified with an explicit name
  and are primarily interpreted independently of their position.

A **verb** is a type of a positional argument that describes a command of its own.
Verbs are also known as subcommands.

Options are also known as "switches".

### Command objects

The command object is recursively defined as follows.

```json
{
  "type": "command",
  "id": "name1",
  "names" : ["name1", "name2", …],
  "version": ["myversion"],
  "features": ["feature1", "feature2", …]
  "abstract": ["paragraph1", "paragraph2", …],
  "postscript": ["paragraph1", "paragraph2", …],
  "help": "short help",
  "arguments": [<option|argument|command>, …],
  "valueName": "",
  "synopsis": [],
  "documentation": ["doc1", …],
  "project": "myproject",
  "isDeprecated": false
}
```

All keys except `type`, `id`, and `names` are optional
and are treated as empty or false when missing.

`type` is the fixed string `command` and signals that this is a command object,
describing either a top-level command or verb.

`id` is a non-empty string defining the internal handle for the command object.
The first element of the `names` array should be used.

`names` is a non-empty array of string names for that command.
The first element of that array is the primary name of that command.
Further names can be added as aliases,
e.g. for backward compatibility.

`version` is a an array of strings describing the version of the program.
The strings in this array should be compatible with the
[UAPI.10 Version Format Specification](https://uapi-group.org/specifications/specs/version_format_specification/).

`features` defines an array of strings that define
builtin-in features of this program.

`abstract` and `postscript` define arrays of strings that are shown respectively
before and after the description of options in the program's help output.
Each element of the array represents a paragraph of text
as a single unbroken line.
Breaking lines for display purposes should be done during display.
Both are meant for standalone help output,
e.g. for the top-level help output of a program
or specific help output of a verb in cases where they have their own help output.

`help` defines the help text of the command.
It is useful for single-line usage information
as well as for short descriptions of verbs in the list of options.

`arguments` array defines a commands's positional arguments and options
in the form described below.
It consists of *option objects*, *argument objects*, and *command  objects*.

`valueName` is a string shown to users to identify the argument,
e.g. in usage information.
It is only useful for subcommands or verbs inside a parent `arguments` array.

`synopsis` is an array that,
if defined,
must be equal to the synopsis or usage string derived from the command and the `arguments` array.
It's use is for convenience so that clients do not need to parse the full arguments
array to construct the usage string.

`documentation` is an array of string values describing URIs referencing
documentation for this command, see `man:uri(7)`for a description of valid URIs.
Preferably the URIs should be URLs starting with `https://` or `man:` as these
are widely supported in modern terminals.

`project` is a string describing what this program belongs to.
This may the package that installed it or the project that produced it.

`isDeprecated` is a boolean describing whether the command has been deprecated
and its use should be avoided.

#### `help` vs. `abstract` and `postscript`

Both `help` as well as `abstract` and `postscript` define output that is
considered help or usage information.

They differ in their type due to their intended usage:
`help` is a single string, `abstract` and `postscript` are arrays of strings.

`help` is meant for short, single line help, for overviews, e.g. in command listings,
whereas `abstract` and `postscript` are meant for longer texts in full help output.

`Command` objects should define at least either `help` or `abstract`,
but can use all three for different display purposes,
decided by the consumer.

In the example below the top-level command defines only `abstract` and `postscript`,
while the subcommands only define `help`,
but subcommands could define `abstract` and `postscript` for usage with their own
full help output,
separate from their parent command,
and the top-level command could define `help`,
e.g. for single line usage information.

### Argument objects

Argument objects describe positional arguments that are not verbs.

```json
{
  "type": "argument"
  "id": "filename",
  "valueName": "FILE",
  "help": "filename to operate on",
  "sections": ["Arguments"],
  "values": [<value object>],
  "isRequired": false,
  "isDeprecated": false
}
```

All keys except `type` and `id` are optional and are treated as empty or false
when missing.

`type` is the fixed string `argument` and signals that this is an argument object.

`id` is a non-empty string defining the internal handle for the argument object.

`valueName` is a string shown to users to identify the argument,
e.g. in usage information.

`help` is a string that defines the help text of the argument.

`sections` is an array of strings that defines sections in which this option should
be shown when the help output is split into sections.
It is intended mostly for display purposes.

`values` is an array of *value objects* describing the values this option may take.
An empty array describes an argument that is a single arbitrary word with no
further documented semantic.
Value objects are described in a section below.

`isRequires` specifies whether this positional argument
must be present in the command line.

`isDeprecated` is a boolean describing whether this argument has been deprecated.

### Option Objects

Option objects describe options.

```json
{
  "type": "option"
  "id": "help",
  "names": ["-h","--help"],
  "value": "required|optional|no",
  "help": "Show this help",
  "valueName": "",
  "sections": [""],
  "values": [<value object>],
  "isDeprecated": false
}
```

All keys except `type`, `id`, `names`, and `argument` are optional
and are treated as empty or false when missing.

`type` is the fixed string `option` and signals that this is an option object,
describing an optional argument. This field must be present.

`id` is a non-empty string defining the internal handle for the option object.

`names` is a non-empty array of strings defining the name of an option.
This field must be present and at least one name must be specified.
The names in the array **must** all either start with a dash,
in which case the option object describes an option,
**or** they **must** all *not* start with a dash,
in which case the option object describes a positional argument.

When an option is specified by a name that starts with a single dash (`-`),
it is called a "short option",
and when it is specified by a name that starts with a double dash (`--`),
it is called a "long option".
Short option names are usually just a single character after the dash,
whereas long option names **should** be a longer string.
Multi-character short option names are discouraged.

Commands that have single-character short options
usually allow multiple short options to be specified together after a single dash.
The last of those options **may** take a value,
but the earlier ones **may not**,
since it would be impossible to distinguish the value from the other options.

`value` defines whether the option takes a value, and may be one of the strings:
-`no`,
-`required`, or
-`optional`.
When `required`, the option must be followed by a value.
When `optional`, the option may be followed by a value.
// TODO: describe how to the presence or not of a value is figured out.
When `no`, the option takes no value.

`help` is a string that defines the help text of the option.

`valueName` is a string shown to users to identify the argument of an option,
e.g. in usage information.
// TODO: describe allowed chacters, more relaxed than an option name.

`sections` is an array of strings that defines sections
in which this option should be shown.
This is primarily intended for display purposes.

`values` is an array of *value objects* describing the values this option may take.
An empty array describes an argument that is a single arbitrary word with no
further documented semantic.
Value objects are described in a section below.

`isDeprecated` is a boolean describing whether this option has been deprecated.

### Value objects

Value objects describe values passed as positional arguments or with an option.

```json
{
  "type": "value",
  "value": "myname",
  "help": "help text",
  "isDefault": true,
  "isDeprecated": false
}
```

All keys except `type` and one of `value`, `category`, `dynamic` or `missing`
are optional and are treated as empty string or false when missing.

`type` is the fixed string `value` and signals that this is a value object.

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

`missing` is a boolean signaling that some values are missing from the description.

`help` is a string describing the help text that should be shown for the value.

`isDefault` is a boolean describing whether this value is the default value for
the option this is a value for.

If multiple of `value`, `dynamic` and `missing` are defined,
`missing` has the highest precedence, followed by `value` and `dynamic`.

## Limits

The maximum allowed level of nesting in the JSON structure is 31.
A `command` object may contain other `command` objects in its `arguments` array,
so those may be nested at most 15 times.
Programs producing **must not** emit JSON structures that exceed this limit,
and consumers **should** ignore or refuse to use such outputs.

## Extensions

Vendors may add additional attributes to the objects described in this specification,
but they ust be prefixed with `x-vendorname.`,
e.g. a vendor `foo` wanting to add an attribute `bar` would use the attribute name `x-foo.bar`.

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

The resulting program introspection JSON would be:
```json
{
  "mediaType": "application/vnd.io.systemd.cli-introspection",
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
        "…"
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
      "synopsis": [
          "systemd-id128",
          "[OPTIONS...]",
          "COMMAND"
      ],
      "arguments": [
        {
          "type": "option",
          "names": [
            "-h",
            "--help"
          ],
          "argument": "no",
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
          "argument": "no",
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
          "argument": "no",
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
          "argument": "no",
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
          "argument": "required",
          "valueName": "FORMAT",
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
              "isDefault": true
            }
          ]
        },
        {
          "type": "option",
          "names": [
            "-j"
          ],
          "argument": "no",
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
          "argument": "no",
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
          "argument": "no",
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
          "valueName": "ID",
          "argument": "required",
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
          "valueName": "COMMAND",
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
          "valueName": "COMMAND",
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
          "valueName": "COMMAND",
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
          "valueName": "COMMAND",
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
          "valueName": "COMMAND",
          "sections": [
            "Commands"
          ],
          "arguments": [
            {
              "type": "argument",
              "name": "name_or_uuid",
              "argument": "optional"
              "valueName": "NAME|UUID"
            }
          ],
          "help": "Print one or more UUIDs"
        },
        {
          "type": "command",
          "name": [
            "help"
          ],
          "valueName": "COMMAND",
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

The `features` array has been shortened for length, since the internal are not
of interest here, so `"…"` is meant as valid JSON placeholder above
and not as an implementation guideline.
