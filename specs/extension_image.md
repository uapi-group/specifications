---
title: Extension Images
category: Concepts
layout: default
version: 1
SPDX-License-Identifier: CC-BY-4.0
---
# Extension Images
Extension Images are DDIs ([Discoverable Disk Images](discoverable_disk_image.md)) that are
built to extend a base system via an overlay. A base system or a root DDI can be extended by several extension
DDIs via, usually, a read-only OverlayFS. The defining characteristic of an Extension Image is that it contains
an `extension-release.<IMAGE>` file that identifies itself and the base system or root DDI it applies to,
and must not contain an `os-release` file.

## Image Format
Extensions are DDIs ([Discoverable Disk Images](discoverable_disk_image.md)), so the file format will not be
redefined here.

## Extension Types
There are two types of extension images, sysext (System Extension) and confext (Configuration Extension).
They are differentiated by the directory hierarchies they contain.

### sysext (System Extension)
sysext images extend `/usr/` (OS vendor tree) and/or `/opt/` (third-party vendor tree). They must contain a
`/usr/lib/extension-release.d/extension-release.<IMAGE>` file to identify them.

### confext (Configuration Extension)
confext images extend `/etc`. They must contain a `/etc/extension-release.d/extension-release.<IMAGE>` file
to identify them.

## Image Content
Extension Images should be additive, and not override content present in the base image or other DDIs,
but this will not be enforced.

## Identification
An Extension Image must contain a `extension-release.<IMAGE>` file, where `<IMAGE>` must either match the
name of the sysext minus the suffix, or alternatively `extension-release.<IMAGE>` must be tagged with a
`user.extension-release.strict` xattr set to the string `"0"` in order to be valid. This is to make it
obvious to users that a sysext is used for its purpose.
The format of `extension-release.<IMAGE>` is the same as the
[`os-release` file](https://www.freedesktop.org/software/systemd/man/os-release.html), and it is a
newline-separated list of environment-like shell-compatible variable assignments. New fields
`SYSEXT_LEVEL=` and `CONFEXT_LEVEL=` have been introduced to allow an implementation to match a sysext or
a confext with the base image upon which it is layered: if the field is present, it must match between the
layers or the Extension Image must be ignored, while if it is not present, but `VERSION_ID=` is, then the
latter must match instead.
In addition, the `ID=` field must be present and match the base image's, or be set to the special value
`_any`, in case the Extension Image can be used on any Linux distribution.

### Fields in extension-release — Matching with the base system or DDI
The following fields are used in order to match with the base system or DDI.
#### `SYSEXT_LEVEL=` `CONFEXT_LEVEL=`
A lower-case string (mostly numeric, no spaces or other characters outside of 0–9, a–z, ".", "_" and
"-") identifying the operating system extensions support level, to indicate which extension images are
supported.

Examples: `"SYSEXT_LEVEL=2"`, `"CONFEXT_LEVEL=15.14"`.

If not present, and if `VERSION_ID=` is present instead, then this will be checked instead.

#### `VERSION_ID=` `ID=` `ARCHITECTURE=`
`VERSION_ID=` and `ID=` are used to match the Extension Image with the root DDI, and `ARCHITECTURE=` is used
to match with the host's CPU architecture, as defined in the
[`os-release` specification](https://www.freedesktop.org/software/systemd/man/os-release.html).
`ID=` and `ARCHITECTURE=` also support specifying the `_any` wildcard, which allows the matching mechanism
to be bypassed.

### Fields in extension-release — Identifying the Extension Image
The identification fields defined in the
[`os-release` specification](https://www.freedesktop.org/software/systemd/man/os-release.html)
can be used to also identify the sysext itself, by prefixing them with `SYSEXT_`. For example,
`SYSEXT_ID=myext` `SYSEXT_VERSION_ID=0.1` denotes a 'myext' sysext of version '0.1'.
There are also extension-specific fields that do not apply to 'os-release', `SYSEXT_SCOPE=`,
`CONFEXT_SCOPE=` and `ARCHITECTURE=`.

#### `SYSEXT_SCOPE=` `CONFEXT_SCOPE=`
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
|loongarch64|
|native|
|any|
