---
title: Version Format Specification
layout: default
version: 1.0
SPDX-License-Identifier: CC-BY-4.0
---

# Version Format Specification

| Version | Changes |
|---------|---------|
| 1.0     | Initial Release |

This specification defines the format of version strings and their ordering.
Various documents that are part of the [uapi-group specification suite](https://uapi-group.org/specifications),
as well as other projects like [systemd](https://systemd.io),
rely on this specification to have
a sort order for strings that include version components,
and use it for various purposes,
such as choosing the default boot entry in the [Boot Loader Specification](boot_loader_specification.md).

## Version Format
The version string is a sequence of zero or more characters.

The following characters have special meaning:
- ASCII digits (`0-9`) form numerical components.
- ASCII letters (`a-z`, `A-Z`) form alphabetical components.
- Dot (`.`) separates parts of a component.
- Minus (`-`) separates major parts of the version string.
- Tilde (`~`) starts a suffix that always sorts lower.
- Caret (`^`) starts a suffix that always sorts higher.

Other characters are treated as separators.
This includes plus (`+`) and underscore (`_`) and other printable or non-printable characters.
The underscore MAY be used.
The plus SHOULD NOT be used, to avoid confusion with SEMVER which attaches a special meaning to it.
Other characters MUST NOT be used in a version string.

Note that in some contexts (for example [the DDI specification](discoverable_disk_image.md) and DEB
package file names), the underscore is used as a separator and cannot be used freely in the version
string.

## Version Comparison

The following method should be used to compare version strings. The algorithm
is based on rpm's `rpmvercmp()`, but not identical.

Both strings are compared from the beginning until the end, or until the
strings are found to compare as different. In a loop:
1. Any characters which are outside of the set of listed above (`a-z`, `A-Z`, `0-9`, `-`, `.`, `~`, `^`)
   are skipped in both strings. In particular, this means that non-ASCII characters
   that are Unicode digits or letters are skipped too.
2. If the remaining part of one of strings starts with `~`:
   if other remaining part does not start with `~`,
   the string with `~` compares lower. Otherwise, both tilde characters are skipped.
3. If one of the strings has ended: if the other string hasn't, the string that
   has remaining characters compares higher. Otherwise, the strings compare
   equal.
4. If the remaining part of one of strings starts with `-`:
   if the other remaining part does not start with `-`,
   the string with `-` compares lower. Otherwise, both minus characters are skipped.
5. If the remaining part of one of strings starts with `^`:
   if the other remaining part does not start with `^`,
   the string with `^` compares lower. Otherwise, both caret characters are skipped.
6. If the remaining part of one of strings starts with `.`:
   if the other remaining part does not start with `.`,
   the string with `.` compares lower. Otherwise, both dot characters are skipped.
7. If either of the remaining parts starts with a digit: numerical prefixes are
   compared numerically. Any leading zeroes are skipped.
   The numerical prefixes (until the first non-digit character) are evaluated as numbers.
   If one of the prefixes is empty, it evaluates as 0.
   If the numbers are different, the string with the bigger number compares higher.
   Otherwise, the comparison continues at the following characters at point 1.
8. Leading alphabetical prefixes are compared alphabetically.
   The substrings are compared letter-by-letter.
   If both letters are the same, the comparison continues with the next letter.
   All capital letters compare lower than lower-case letters (`B < a`).
   When the end of one substring has been reached (a non-letter character or the end
   of the whole string), if the other substring has remaining letters, it compares higher.
   Otherwise, the comparison continues at the following characters at point 1.

## Comparison with Other Specifications
Other specifications exist to mandate version formats:

- [RPM Packaging Guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/Versioning/)
- [Debian Policy](https://www.debian.org/doc/debian-policy/ch-controlfields.html#version)
- [Semantic Versioning](https://semver.org/)

All of these, including the present document, share some commonalities but are also
incompatible in some ways, as they all evolved in different environments. The main
differences are as follows.

- to separate components DEB uses `_`, RPM uses `-` with positional logic (it assumes different meaning in different positions), and SemVer does not specify anything as it is concerned only with the version part of the string
- to identify a pre-release suffix RPM and DEB use `~` and SemVer uses `-`
- to identify a rebuild suffix DEB uses `+`, SemVer uses `.`, and RPM increases the `release` part of the version
- to identify an epoch prefix DEB and RPM use `:`, and SemVer does not specify anything

## Examples
Examples (with '' meaning the empty string):

* `11 == 11`
* `systemd-123 == systemd-123`
* `bar-123 < foo-123`
* `123a > 123`
* `123.a > 123`
* `123.a < 123.b`
* `123a > 123.a`
* `11α == 11β`
* `B < a`
* '' < `0`
* `0.` > `0`
* `0.0` > `0`
* `0` > `~`
* '' > `~`
* `1_` == `1`
* `_1` == `1`
* `1_` < `1.2`
* `1_2_3` > `1.3.3`
* `1+` == `1`
* `+1` == `1`
* `1+` < `1.2`
* `1+2+3` > `1.3.3`

Note how in the `1_2_3` > `1.3.3` and `1+2+3` > `1.3.3` cases, the underscore and plus characters act as
separators between components, so we first compare `1` with `1.3.3` as numerical version strings, and
`1` < `1.3.3`. The remainder of the first string is not used in the comparison.

* `122.1` < `123~rc1-1` < `123` < `123-a` < `123-a.1` < `123-1` < `123-1.1` < `123^post1` < `123.a-1` < `123.1-1` < `123a-1` < `124-1`

In the above example each entry compares smaller than every entry to its right and equal only to itself,
conversely each entry compares larger to every entry to its left and compares unequal to all except itself.

## Notes
[systemd-analyze](https://www.freedesktop.org/software/systemd/man/systemd-analyze.html)
implements this version comparison algorithm as
```
systemd-analyze compare-versions <version-a> <version-b>
```

