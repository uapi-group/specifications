---
title: sysext - System Extension
category: Concepts
layout: default
version: 1
SPDX-License-Identifier: CC-BY-4.0
---
# sysext (System Extension)
sysexts (System Extensions) are DDIs ([Discoverable Disk Images](../discoverable_disk_image)) that are
built to extend a base system via an overlay. A root (or `/usr/`) DDI can be extended by several sysext
DDIs via, usually, a read-only OverlayFS.

## Image Format
sysexts are DDIs ([Discoverable Disk Images](../discoverable_disk_image)), so the file format will not be
redefined here.

## Content
sysext images extend `/usr/` (OS vendor tree) and/or `/opt/` (third-party vendor tree). They should be
additive, and not override content present in the base image or other sysext images, but this will not be
enforced.

### Identification
A sysext must contain a `/usr/lib/extension-release.d/extension-release.<IMAGE>` file, where `<IMAGE>`
must either match the name of the sysext minus the suffix, or alternatively `extension-release.<IMAGE>`
must be tagged with a `user.extension-release.strict` xattr set to the string `"0"` in order to be valid.
This is to make it obvious to users that a sysext is used for its purpose.
The format of `extension-release.<IMAGE>` is the same as the
[`os-release` file](https://www.freedesktop.org/software/systemd/man/os-release.html), and it is a
newline-separated list of environment-like shell-compatible variable assignments. A new field
`SYSEXT_LEVEL=` has been introduced to allow an implementation to match a sysext with the base image upon
which it is layered: if `SYSEXT_LEVEL=` is present, it must match between the layers or the systext must
be ignored, while if it is not present, but `VERSION_ID=` is, then the latter must match instead.
In addition, the `ID=` field must be present and match the base image's, or be set to the special value
`_any`, in case the sysext can be used on any Linux distribution.

### Fields in extension-
#### `SYSEXT_LEVEL=`
A lower-case string (mostly numeric, no spaces or other characters outside of 0–9, a–z, ".", "_" and
"-") identifying the operating system extensions support level, to indicate which extension images are
supported.

Examples: `"SYSEXT_LEVEL=2"`, `"SYSEXT_LEVEL=15.14"`.

If not present, and if `VERSION_ID=` is present instead, then this will be checked instead.

#### `SYSEXT_SCOPE=`
Takes a space-separated list of one or more of the strings `"system"`, `"initrd"` and `"portable"`. This field
is optional and indicates what environments the system extension is applicable to: i.e. to regular systems,
to initrds, or to [portable service images](https://systemd.io/PORTABLE_SERVICES/). If unspecified,
`"SYSEXT_SCOPE=system portable"` is implied, i.e. any system extension without this field is applicable to
regular systems and to portable service environments, but not to initrd environments.

#### `ARCHITECTURE=`
A string that specifies which CPU architecture the userspace binaries require. This field is optional
and should only be used when just single architecture is supported. It may provide redundant
information when used in a GPT partition with a GUID type that already encodes the architecture. If this
is not the case, the architecture should be specified in e.g., an extension image, to prevent an
incompatible host from loading it.

Valid values:

|Architecture|
|------------|
|x86|
|x86-64|
|ppc|
|ppc-le|
|ppc64|
|ppc64-le|
|ia64|
|parisc|
|parisc64|
|s390|
|s390x|
|sparc|
|sparc64|
|mips|
|mips-le|
|mips64|
|mips64-le|
|alpha|
|arm|
|arm-be|
|arm64|
|arm64-be|
|sh|
|sh64|
|m68k|
|tilegx|
|cris|
|arc|
|arc-be|
|native|
|any|
