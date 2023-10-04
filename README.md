---
title: UAPI Group Specifications
BookToC: false
---

# UAPI Group Specifications

* [Boot Loader Specification](specs/boot_loader_specification.md):
  Defines a set  of file formats and naming conventions to allow distribution independent boot loader menus supportable by multiple bootloaders.
* [Configuration Files Specification](specs/configuration_files_specification.md):
  Standardises default locations and environment variables for locating common files or base directories.
  This is derived from, and extends, the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html),
  to allow for separation between vendor and admin configuration files, drop-in files, and masking.
* [Discoverable Partitions Specification](specs/discoverable_partitions_specification.md):
  Discusses GUID UUIDs for auto-discovery of partition semantics and mount points.
* [Discoverable Disk Image](specs/discoverable_disk_image.md):
  Describes the Discoverable Disk Image format for self-describing system images.
* [Extension Image](specs/extension_image.md):
  Describes the use of Discoverable Disk Images to create extensions to a base image.
* [Unified Kernel Image](specs/unified_kernel_image.md):
  Describes the use of UEFI PE binaries to provide a Unified Kernel Image containing the kernel, initrd, command line, and other components.
* [Version Format Specification](specs/version_format_specification.md):
  Defines semantics of version strings used in the other specifications listed here.
* [Linux TPM PCR Registry](specs/linux_tpm_pcr_registry.md):
  An informative list of how TPM PCRs are used on a Linux system.

## Work in Progress

See [open PRs on github](https://github.com/uapi-group/specifications/pulls?q=is%3Apr+is%3Aopen+sort%3Aupdated-desc).

## Glossary

This section clarifies on terms and abbreviations used in specs and other documents.

## General terms and abbreviations
- *MOK* – Machine Owner Key (shim)
- *PCR* – TPM Platform Configuration Registers
- *TPM* – Trusted Platform Module (security chip)
- *SBAT* – UEFI Secure Boot Advanced Targeting

## Terms and abbreviations specific to UAPI group specifications
- [*DDI*](specs/discoverable_disk_image.md) - Discoverable Disk Image
- [*DPS*](specs/discoverable_partitions_specification.md) - Discovery Partition Specification
- [*UKI*](specs/unified_kernel_image.md) - Unified Kernel Images (sd-stub + kernel + initrd + more)
- [*BLS*](specs/boot_loader_specification.md) - Boot Loader Specification
- [*sysext*](specs/extension_image.md) – System Extension Image
  (type of DDI that is overlayed on top of `/usr/` and `/opt/` via overlayfs and can extend the underlying OS vendor resources in a composable, immutable fashion)
- [*confext*](specs/extension_image.md) – Configuration Extension Image
  (type of DDI that is overlayed on top of `/etc/` via overlayfs and can extend the underlying OS' configuration in a composable, immutable fashion)

## Participate

Please use the [specifications issue tracker](https://github.com/uapi-group/specifications/issues) to engage with the project.
