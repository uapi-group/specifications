---
title: UAPI.6 Configuration Files Specification
layout: posts
version: 1.0
---

# UAPI.6 Configuration Files Specification

| Version | Changes |
|---------|---------|
| 1.0     | Initial Release |

## Introduction

Various specifications attempt to define configuration files and file formats. This
specification establishes where these files should be looked for, in which
order, and how precedence, masking, extensions and overrides work.

The purpose of the rules defined here is to allow OS vendors to implement the
hermetic-usr pattern, where all vendor files are shipped in the vendor tree itself
(`/usr/`), including configuration files with system defaults, while allowing local
or vendor overrides without modifying the original files, for easier management. This is
especially beneficial for image-based deployments, where the vendor tree is read-only.

These rules are derived from existing real-world usage from the [systemd
project](https://github.com/systemd/systemd) and the [libeconf
project](https://github.com/openSUSE/libeconf). It is highly recommended to use libeconf,
which is readily available and maintained in all major distributions, rather than
reimplementing this specification locally.

This specification is agnostic toward the actual format and content of the configuration
files, the precise path used under each top-level hierarchy, and also toward the
filenames and extensions, with the exception of the `.d/` notation for drop-ins
directories. It is strongly encouraged to enforce a specific suffix for the configuration
files, in order to disambiguate (e.g.: with backup files), but the choice of which suffix
to use is left to each implementation.

## Storage Directories and Overrides
In order to allow shipping system defaults owned by the OS vendor, while at the same
time letting local users or admins override those defaults, `/usr/` and `/etc/` are both
supported for storage of configuration files, with the latter having higher priority.
The precise location under `/usr/` is left open for the implementation to decide - it
could be hard-coded to `/usr/lib/` or it could be left to each application to pick from
various options, such as `/usr/share/` or `/usr/etc/`.
Programs must work correctly if no configuration files are found in `/etc/`.
Optionally, `/run/` is also supported for ephemeral overrides.

For example, `/usr/lib/foo/bar.conf` provides the default configuration file.
If `/run/foo/bar.conf` is present and supported, it would take precedence over
`/usr/lib/foo/bar.conf`.
Finally, a user can create `/etc/foo/bar.conf` which would take precedence and
completely override both.

## Masking
As a special override case, it must be possible to mask files across different
locations by creating a symlink to `/dev/null` or an empty file.

For example, an empty `/etc/foo/bar.conf` means that `/usr/lib/foo/bar.conf` is
masked and thus not parsed.

## Drop-ins
All configuration paths must support drop-ins,
except for configuration file formats where automatic combining of multiple files is not feasible,
for example scripts or structured documents.
Supporting drop-ins means that in addition to parsing a full configuration file,
an implementation also parses the drop-in files in the drop-in directories associated with it.

Drop-ins always have higher precedence than the configuration file they refer to.
Drop-ins are sorted in the lexicographic order using the file name without the path,
regardless of the hierarchy under which they are stored.
The drop-ins that are later in this order have higher precedence.

Considering the following files are present on the filesystem, this would be the order in which the
files are parsed. Note, that files with the same name override each other. The configuration in
`bar.conf` has the lowest priority, and is read before `a.conf` and `b.conf`. `b.conf` has the
highest priority:

  - ~~`/usr/lib/foo/bar.conf`~~ (overridden by `/etc/foo/bar.conf`)
  - `/etc/foo/bar.conf`
  - ~~`/usr/lib/foo/bar.conf.d/a.conf`~~ (overridden by `/etc/foo/bar.conf.d/a.conf`)
  - `/etc/foo/bar.conf.d/a.conf`
  - `/usr/lib/foo/bar.conf.d/b.conf`

If a config file is masked, drop-ins must still be parsed, unless they are masked
themselves.

For example, even if `/usr/lib/foo/bar.conf` is masked by an empty `/etc/foo/bar.conf`,
`/usr/lib/foo/bar.conf.d/a.conf` must still be parsed and applied, unless there is also
an empty `/etc/foo/bar.conf.d/a.conf`, in which case the drop-in is masked too.

Drop-ins are not recursive, so a drop-in cannot have a directory of drop-ins.

For example, `/etc/foo/bar.conf.d/a.conf` cannot be overridden by
`/etc/foo/bar.conf.d/a.conf.d/b.conf`, and the latter must be ignored if it exists.

### Drop-ins without Main Configuration File
Optionally, schemes with only drop-ins, without a 'main' configuration file, should also
be supported by implementations. In such schemes many drop-ins are loaded from a common
directory in each hierarchy.

For example, `/usr/lib/foo.d/a.conf`, `/usr/lib/foo.d/b.conf` and `/etc/foo.d/c.conf`
are all loaded and parsed in this scheme, in this order.
