---
title: UAPI.3 Discoverable Disk Image
category: Concepts
layout: default
version: 1.0
SPDX-License-Identifier: CC-BY-4.0
weight: 3
aliases:
- /UAPI.3
- /3
---
# UAPI.3 Discoverable Disk Image (DDI)

| Version | Changes |
|---------|---------|
| 1.0     | Initial Release |

DDIs (Discoverable Disk Images) are self-describing file system images that follow the DPS ([Discoverable
Partitions Specification](discoverable_partitions_specification.md)), wrapped in a GPT partition table, that
may contain root (or `/usr/`) filesystems for bootable OS images, system extensions, configuration
extensions, portable services, containers and more, and shall be protected by signed `dm-verity` all combined
into one.  They are designed to be composable and stackable, and provide security by default.

## Image Format
The images use the GPT partition table verbatim, so it will not be redefined here. Each partition contains
a standard Linux filesystem (e.g.: `erofs`), so again this will not be redefined here.
The [DPS](discoverable_partitions_specification.md) defines the GUIDs to use and the format of the
`dm-verity` signature partition's JSON content.

It is recommended to use a sector size of 512 bytes or 4096 for DDIs. Software operating with DDIs should
automatically derive the sector size used for a DDI by looking for the `EFI PART` magic string at offsets 512
or 4096, as per GPT specification.

## Naming

DDIs should use `.raw` as file suffix. A secondary suffix may be used to clarify the specific usage class of
a DDI. For now the two secondary suffixes `.sysext.raw` and `.confext.raw` are defined (for system extension
DDIs and configuration extension DDIs, see [Extension
Images](https://uapi-group.org/specifications/specs/extension_image) for details).

The MIME type for DDIs is `application/vnd.efi.img`, [as per
IANA](https://www.iana.org/assignments/media-types/application/vnd.efi.img).

## Image Version
If the DDI is versioned, the version format described in the
[Version Format Specification](version_format_specification.md) must be used. The underscore character (`_`)
must be used to separate the version from the name of the image. For example: `foo_1.2.raw` denotes a `foo`
DDI with version `1.2`.
