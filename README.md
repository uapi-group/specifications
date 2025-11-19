---
title: UAPI Group Specifications
BookToC: false
---

# UAPI Group Specifications

* [Boot Loader Specification](specs/boot_loader_specification.md):
  Defines a set  of file formats and naming conventions to allow distribution independent boot loader menus supportable by multiple bootloaders.
  ([canonical online location](https://uapi-group.org/specifications/specs/boot_loader_specification/))
* [Configuration Files Specification](specs/configuration_files_specification.md):
  Standardises default locations and environment variables for locating common files or base directories.
  This is derived from, and extends, the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir/latest/),
  to allow for separation between vendor and admin configuration files, drop-in files, and masking.
  ([canonical online location](https://uapi-group.org/specifications/specs/configuration_files_specification/))
* [Discoverable Partitions Specification](specs/discoverable_partitions_specification.md):
  Discusses GUID UUIDs for auto-discovery of partition semantics and mount points.
  ([canonical online location](https://uapi-group.org/specifications/specs/discoverable_partitions_specification/))
* [Discoverable Disk Image](specs/discoverable_disk_image.md):
  Describes the Discoverable Disk Image format for self-describing system images.
  ([canonical online location](https://uapi-group.org/specifications/specs/discoverable_disk_image/))
* [Extension Image](specs/extension_image.md):
  Describes the use of Discoverable Disk Images to create extensions to a base image.
  ([canonical online location](https://uapi-group.org/specifications/specs/extension_image/))
* [Linux File System Hierarchy](specs/linux_file_system_hierarchy.md):
  Describes the layout of directories and files in an installation of Linux
  ([canonical online location](https://uapi-group.org/specifications/specs/linux_file_system_hierarchy/))
* [File Hierarchy for the Verification of OS Artifacts (VOA)](specs/file_hierarchy_for_the_verification_of_os_artifacts.md):
  Describes the use of Discoverable Disk Images to create extensions to a base image.
  ([canonical online location](https://uapi-group.org/specifications/specs/file_hierarchy_for_the_verification_of_os_artifacts/))
* [Unified Kernel Image](specs/unified_kernel_image.md):
  Describes the use of UEFI PE binaries to provide a Unified Kernel Image containing the kernel, initrd, command line, and other components.
  ([canonical online location](https://uapi-group.org/specifications/specs/unified_kernel_image/))
* [Version Format Specification](specs/version_format_specification.md):
  Defines semantics of version strings used in the other specifications listed here.
  ([canonical online location](https://uapi-group.org/specifications/specs/version_format_specification/))
* [Linux TPM PCR Registry](specs/linux_tpm_pcr_registry.md):
  An informative list of how TPM PCRs are used on a Linux system.
  ([canonical online location](https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/))
* [Package Metadata for Executable Files](specs/package_metadata_for_executable_files.md):
  Describes the format and mechanism to include packaging metadata in ELF/PE binaries.
  ([canonical online location](https://uapi-group.org/specifications/specs/package_metadata_for_executable_files/))

## Work in Progress

See [open PRs on github](https://github.com/uapi-group/specifications/pulls?q=is%3Apr+is%3Aopen+sort%3Aupdated-desc).

## License

All specifications are licensed under [CC-BY-4.0](https://spdx.org/licenses/CC-BY-4.0.html).

## Versioning

All specifications are versioned.

The versioning format is MAJOR.MINOR.
Compatible changes increment the MINOR version.
Incompatible changes increment the MAJOR version and reset the MINOR version to `0`.

Work in progress specifications have a MAJOR version of `0`.

A `filename/MAJOR.MINOR` git tag will be created when a new version of a given spec is released.

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
- [*UKI*](specs/unified_kernel_image.md) - Unified Kernel Images (UEFI boot stub + kernel + initrd + more)
- [*BLS*](specs/boot_loader_specification.md) - Boot Loader Specification
- [*sysext*](specs/extension_image.md) – System Extension Image
  (type of DDI that is overlayed on top of `/usr/` and `/opt/` via overlayfs and can extend the underlying OS vendor resources in a composable, immutable fashion)
- [*confext*](specs/extension_image.md) – Configuration Extension Image
  (type of DDI that is overlayed on top of `/etc/` via overlayfs and can extend the underlying OS' configuration in a composable, immutable fashion)

## Participate

Please use the [specifications issue tracker](https://github.com/uapi-group/specifications/issues) to engage with the project.
