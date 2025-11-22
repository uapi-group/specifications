---
title: UAPI.4 Extension Images
category: Concepts
layout: default
version: 1.0
SPDX-License-Identifier: CC-BY-4.0
---
# Extension Images

| Version | Changes |
|---------|---------|
| 1.0     | Initial Release |

Extension Images are DDIs ([Discoverable Disk Images](discoverable_disk_image.md)) that are
built to extend a base system via an overlay. A base system or a root DDI can be extended by several extension
DDIs via, usually, a read-only OverlayFS. The defining characteristic of an Extension Image is that it contains
an `extension-release.<IMAGE>` file that identifies itself and the base system or root DDI it applies to,
and must not contain an `os-release` file.

## Ordering
The default order in which extensions are applied is based on lexicographic sorting as
[defined in the Version Format Specification](version_format_specification.md), with images sorting as
older being placed lower in the overlay. Implementations may allow a different order to be explicitly
specified instead.

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

<h2 id="image-content">Image Content</h2>
Extension Images should be additive, and not override content present in the base image or other DDIs.
However, there currently is no safe and efficient way to detect collisions and to enforce content uniqueness
across the stack of images.
In a future version of this specification options for enforcing uniqueness may be provided.

## Base Directory Immutability / Mutability
By default, applying ("merging") an Extension Image on a mutable filesystem renders the underlying base
directory (`/etc/`, `/usr/`, `/opt/`) immutable.
By implication, merging a confext on a mutable filesystem will result in `/etc/` becoming read-only, and
merging a sysext might render `/usr/` and/or `/opt/` read-only, depending on the sysext's contents.

This affects base directories actually contained in extensions merged to a root filesystem:
for instance, if a sysext extends `/usr/` but not `/opt/`, `/opt/` will remain mutable if
it's on a mutable filesystem.
However, immutability affects the _full_ base directory.
Merging an extension on a mutable filesystem that ships a single custom path e.g. below
`/etc/appconfig-extra/some/sub/path/` will still render the entirety of `/etc/` immutable.
The base directory or directories remain immutable for as long as extensions are merged.
Mutability, if present before the merge, is regained only after all extensions overlaying a base directory
have been un-merged.

### Optional Mutability
While overlaid base directories are immutable by default, implementations may provide options
for mutability.
Retaining mutability may for instance be useful for compatibility with general purpose applications,
enabling users to operate a "mixed mode" with both system and configuration extensions
and traditional applications.
Mixed mode may also be integrated by distributions to facilitate a smooth transition from
traditional package management to a purely image-based composition of the root file system.
Lastly, mutability mode may be used on an originally immutable filesystem to allow and to capture
temporary changes.

### Extension Overlay [Im]Mutability Modes
System and configuration extensions may operate in one of three modes.

1. *Immutable mode* - The overlaid base directory is immutable.
    This is the default.
2. *Mutable mode* - Writes are directed to an _upperdir_ specified by the user or operator
    (see "Mutability Mode Configuration" below).
    This _upperdir_ will contain all changes made to the overlaid base directory.
    1. _Upperdir_ may be specified to _be_ the base directory: may be used to retain mutability
       of the base directory after extensions have been merged.
    2. Alternatively, _upperdir_ may be an entirely separate directory: modifications will be captured
       but the base directory will remain unchanged, retaining its state from before the extension was merged.
3. *Ephemeral Mode* - Similar to mutable mode (2.) above but writes are only stored temporarily while
    extensions are merged, and discarded as soon as extensions are un-merged.
    Location of temporary storage is implementation-specific.
    Useful for e.g. development and for one-shot validation operations.

### Mutability Mode Configuration
Immutable mode is the default.
If none of the configurations outlined below were specified then extension overlays operate
in immutable mode and base directories are read-only.
Implementation of any of the below mutable configurations is optional.
_If_ mutable modes are supported by an implementation, configuration option 1. below _must_ be supported
for compatibility across implementations.

Mutability modes may be configured in the following ways:
1. By creating qualified paths or soft-links below `/var/lib/extensions.mutable/`.
   See "Qualified Paths Definition" below for details.
   This is the most portable option across different implementations.
   If an implementation supports base directory mutability then this mode _must_ be supported.
2. By setting a respective option in an implementation's configuration file.
   This option is implementation-specific.
   Implementations may choose to support a single option, multiple options for
   system and configuration extensions, and/or multiple options per base directory.
   Using this option should selectively override any qualified path definitions from 1.
3. By passing a command line parameter upon extension merge or refresh.
   This option is implementation-specific.
   Implementations may choose to support a single option and/or multiple options per base directory.
   Using this option should selectively override any configurations from 1. and 2.

#### Qualified Paths Definition

Mutability Mode 1 enables mutability by creating paths or soft-links below
`/var/lib/extensions.mutable/`.
Qualified paths are:
* `/var/lib/extensions.mutable/etc/` - directory or soft-link to a directory to store writes to
  `/etc/`. This is for configuration extensions.
* `/var/lib/extensions.mutable/usr/` - directory or soft-link to a directory to store writes to
  `/usr/`. This is for system extensions.
* `/var/lib/extensions.mutable/opt/` - directory or soft-link to a directory to store writes to
  `/opt/`. This is for system extensions.

Each base directory is treated separately.
The existence and the type of each qualified path determines the mutability mode used.
The following mutability modes are supported:
* _Path does not exist_ - immutable mode.
* Path is a _directory, subvolume, or mount point_ - the path at
  `/var/lib/extensions.mutable/<basedir>/` is used to store writes to `/<basedir>/`.
  * A tmpfs mount at the qualified path may be used for a custom ephemeral mode.
    In this case, clean-up of the tmpfs is left to the user and/or is implementation-specific.
* Path is a _soft link_ - the soft link is followed and writes are stored at the link's destination.
  * If the destination is the base directory - i.e. `/var/lib/extensions.mutable/<basedir>/`
    points to `/<basedir>/` - then the stacking order changes and `<basedir>` becomes _upperdir_.
    Writes are directed to the base directory and files and paths present in the base directory
    override files and paths in extensions if present.
  * The soft link may point to a tmpfs destination for custom ephemeral mode.
    In this case, clean-up of the tmpfs is left to the user and/or is implementation-specific.
  * If the destination does not exist, immutable mode is used.

## File Suffix
Since extensions images are DDIs, they should carry the `.raw` suffix. In order to make discerning system
extensions and configuration extensions easy it is recommended to use the `.sysext.raw` suffix for system
extensions, and `.confext.raw` for configuration extensions.

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

#### `VERSION_ID=`, `ID=`, `ARCHITECTURE=`
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

#### `SYSEXT_SCOPE=`, `CONFEXT_SCOPE=`
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
|alpha|
|arc|
|arc-be|
|arm|
|arm-be|
|arm64|
|arm64-be|
|cris|
|ia64|
|loongarch64|
|m68k|
|mips|
|mips-le|
|mips64|
|mips64-le|
|parisc|
|parisc64|
|ppc|
|ppc-le|
|ppc64|
|ppc64-le|
|riscv32|
|riscv64|
|s390|
|s390x|
|sh|
|sh64|
|sparc64|
|sparc|
|tilegx|
|native|
|any|
