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
"`application/vnd.uapi-group.cli-introspection`" and `commands` is a non-empty
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
1. **positional arguments**, that are identified by their position in relation to other arguments, and
2. **options**, that are specified by an explicit name
  and are primarily interpreted independently of their position.

A **verb** is a type of a positional argument that describes a command of its own.
Verbs are also known as subcommands.

Options are also known as "switches" and "optional arguments"

### Command objects

The command object is recursively defined as follows.

```json
{
  "type": "command",
  "id": "name1",
  "names" : ["name1", "name2", …],
  "versions": ["myversion"],
  "features": ["feature1", "feature2", …]
  "abstract": ["paragraph1", "paragraph2", …],
  "postscript": ["paragraph1", "paragraph2", …],
  "help": "short help",
  "arguments": [<option|argument|command>, …],
  "valueName": "COMMAND",
  "synopsis": [],
  "documentation": ["doc1", …],
  "project": "myproject",
  "isDeprecated": false
}
```

All keys except `type`, `id`, and `names` are optional
and are treated as empty or false when missing.

`type` is the fixed string "`command`" and signals that this is a command object,
describing either a top-level command or verb.

`id` is a non-empty string defining the internal handle for the command object.
The first element of the `names` array should be used.

`names` is a non-empty array of string names for that command.
The first element of that array is the primary name of that command.
Further names can be added as aliases,
e.g. for backward compatibility.

`versions` is a an array of strings describing the version of the program.
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

`arguments` is an array defining a commands's positional arguments and options
in the form described below.
It consists of *option objects*, *argument objects*, and *command  objects*.

`valueName` is a string shown to users to identify the argument,
e.g. in usage information.
It is only useful for subcommands or verbs inside a parent `arguments` array.
It defaults to `COMMAND`.

`synopsis` is an array of arrays of strings that,
defines synopsis or usage string of the command.
If not defined it can be dervied from the `arguments` array of the command.
It's use is for convenience so that clients do not need to parse the full arguments
array to construct the usage string,
or if there are multiple usage strings that should highlight different usages.

`documentation` is an array of string values describing URIs referencing
documentation for this command, see `man:uri(7)`for a description of valid URIs.
Preferably the URIs should be URLs starting with `https://` or `man:` as these
are widely supported in modern terminals.

`project` is a string describing what this command belongs to.
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
  "isDeprecated": false,
  "repeatable": 0,
}
```

All keys except `type` and `id` are optional and are treated as empty, false, or
zero when missing.

`type` is the fixed string `"argument"` and signals that this is an argument object.

`id` is a non-empty string defining the internal handle for the argument object.

`valueName` is a string shown to users to identify the argument,
e.g. in usage information.
It defaults to `ARGUMENT`.

`help` is a string that defines the help text of the argument.

`sections` is an array of strings that defines sections in which this option should
be shown when the help output is split into sections.
It is intended mostly for display purposes.

`values` is an array of *value objects* describing the values this option may take.
An empty array describes an argument that is a single arbitrary word with no
further documented semantic.
Value objects are described in a section below.

`isRequired` specifies whether this positional argument
must be present in the command line.

`isDeprecated` is a boolean describing whether this argument has been deprecated.

`repeatable` is an integer describing how many times after a first use the
option can be repated.
A value of 0 indictaes that the argument can only be used once.
Use -1 for arbitrary amount of repetitions.

### Option Objects

Option objects describe options.

```json
{
  "type": "option"
  "id": "help",
  "names": ["-h","--help"],
  "value": "required|optional|no",
  "help": "Show this help",
  "valueName": "OPTION",
  "sections": [""],
  "values": [<value object>],
  "isDeprecated": false,
  "reapeatable": 0,
  "conflicts": "",
  "requires": "",
}
```

All keys except `type`, `id`, and `names` are optional
and are treated as empty, false, zero, or "no" (for "value") when missing.

`type` is the fixed string `"option"` and signals that this is an option object.

`id` is a non-empty string defining the internal handle for the option object.

`names` is a non-empty array of strings defining the name of an option.
This field must be present and at least one name must be specified.
The names in the array **must** all start with a dash.

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
-"`yes`",
-"`no`", or
-"`optional`".
When "`yes`", the option must be followed by a value.
When "`optional`", the option may be followed by a value.
// TODO: describe how to the presence or not of a value is figured out.
When "`no`", the option takes no value.

`help` is a string that defines the help text of the option.

`valueName` is a string shown to users to identify the option,
e.g. in usage information.
It defaults to `OPTION`.
// TODO: describe allowed chacters, more relaxed than an option name.

`sections` is an array of strings that defines sections
in which this option should be shown.
This is primarily intended for display purposes.

`values` is an array of *value objects* describing the values this option may take.
An empty array describes an argument that is a single arbitrary word with no
further documented semantic.
Value objects are described in a section below.

`isDeprecated` is a boolean describing whether this option has been deprecated.

`repeatable` is an integer describing how many times after a first use the
option can be repated.
A value of 0 indictaes that the option can only be used once.
Use -1 for arbitrary amount of repetitions.

`conflicts` is a string describing a glob of `id`s that conflict with this option.
An empty string means that this option does not conflict with anything.

`conflicts` is a string describing a glob of `id`s that are required together with this option.
An empty string means that this option does not requite any other option.

### Value objects

Value objects describe values passed as positional arguments or with an option.

```json
{
  "type": "value",
  "value": "myname",
  "valueName": "MYVAL",
  "help": "help text",
  "isDefault": true,
  "isDeprecated": false
}
```

All keys except `type` and one of `value`, `category`, `dynamic` or `missing`
are optional and are treated as empty string, or false when missing.

`type` is the fixed string `"value"` and signals that this is a value object.

`value` is a string describing a possible static value.

`valueName` is a string shown to users to identify the value,
e.g. in usage information.

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
- `unit`, a systemd unit's name,
- `uuid`, a UUID,
- `any`, any string.
The empty string is to be handled as `any` and leaves the completion open to the shell.

`dynamic` is a string describing a command,
that can be called to generate multiple values.
This is relevant for the dynamic generation of completion candidates during
command line completion.
The current input for the argument,
if any,
otherwise an empty string will be passed as first and only argument.
The standard output stream of the command defines the values,
one per line.
An empty string for this key means that values cannot be generated dynamically
for this object.

`missing` is a boolean signaling that some values are missing from the description.

`help` is a string describing the help text that should be shown for the value.

`isDefault` is a boolean describing whether this value is the default value for
the option this is a value for.

If multiple of `value`, `dynamic` and `missing` are defined,
`missing` has the highest precedence, followed by `value` and `dynamic`.

### References

Reference objects describe repetitions of previous options and arguments in
the arguments array.

```json
{
  "type": "reference",
  "id": "endref"
  "match": "opt.*"
}
```

The keys `type`, `id` and `match` are non-optional.

`type` is the fixed string `"reference"` and signals that this is an reference object.

`id` is a non-empty string defining the internal handle for the option object.

`match` is a string describing a glob of `id` fields for other objects
in the same argument array with smaller index than the object that is referencing them,
that should be repeated.

### Separators

Separators are objects that describe that parsing of options should end.
Commonly the string `--` is used to signify this in command line interfaces.

```json
{
  "type": "separator",
  "id": "sepopt",
  "valueName": "--",
  "continueWith": ""
}
```

The keys `type` and `id` are non-optional.
`valueName` defaults to `"--"`
and `cotinueWith` defaults to the empty string.

`type` is the fixed string `"separator"` and signals that this is an separator object.

`id` is a non-empty string defining the internal handle for the option object.

`valueName` is a string that represents the separator.
It defaults to `"--"`,
which is a commonly used representation for this.

`continueWith` is a string that if set to a non-empty value describes
a different program whose command line introspection should be used for subsequent arguments.

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

## Well-known option for command line introspection

A command supporting this specification should use `--introspect-cli` to output
its introspection information.

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
        "systemd-id128"
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
          "value": "required",
          "valueName": "FORMAT",
          "sections": [
            "Options"
          ],
          "help": "Output inspection data in JSON (takes one of pretty, short, off)",
          "values": [
            {
              "type": "value",
              "value": "short",
              "valueName": "FORMAT",
              "help": "the shortest possible output without any redundant whitespace or line breaks"
            },
            {
              "type": "value",
              "value": "pretty",
              "valueName": "FORMAT",
              "help": "a pretty version of the same"
            },
            {
              "type": "value",
              "value": "off",
              "valueName": "FORMAT",
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
          "argument": "required",
          "sections": [
            "Options"
          ],
          "help": "Generate app-specific IDs",
          "values": [
              {
                  "type": "value",
                  "category": "uuid"
                  "valueName": "ID",
                  "help": "An application-specific UUID"
              }
          ]
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
              "id": "show.arg",
              "argument": "optional"
              "values": [
                  {
                      "type": "value",
                      "id": "show.arg.name",
                      "category": "any",
                      "valueName": "NAME",
                      "help": "A DPS name"
                  },
                  {
                      "type": "value",
                      "id": "show.arg.uuid",
                      "category": "any",
                      "valueName": "UUID",
                      "help": "A DPS partition type UUID"
                  }
              ]
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
