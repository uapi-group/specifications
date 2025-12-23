---
title: UAPI Group Specifications
BookToC: false
---

# UAPI Group Specifications

The following specifications have been accepted by the UAPI group:

* [UAPI.1 Boot Loader Specification](specs/boot_loader_specification.md):
  Defines a set  of file formats and naming conventions to allow distribution independent boot loader menus supportable by multiple bootloaders.
  ([canonical online location](https://uapi-group.org/specifications/specs/boot_loader_specification/))
* [UAPI.2 Discoverable Partitions Specification](specs/discoverable_partitions_specification.md):
  Discusses GUID UUIDs for auto-discovery of partition semantics and mount points.
  ([canonical online location](https://uapi-group.org/specifications/specs/discoverable_partitions_specification/))
* [UAPI.3 Discoverable Disk Images](specs/discoverable_disk_image.md):
  Describes the Discoverable Disk Image format for self-describing system images.
  ([canonical online location](https://uapi-group.org/specifications/specs/discoverable_disk_image/))
* [UAPI.4 Extension Images](specs/extension_image.md):
  Describes the use of Discoverable Disk Images to create extensions to a base image.
  ([canonical online location](https://uapi-group.org/specifications/specs/extension_image/))
* [UAPI.5 Unified Kernel Images](specs/unified_kernel_image.md):
  Describes the use of UEFI PE binaries to provide a Unified Kernel Image containing the kernel, initrd, command line, and other components.
  ([canonical online location](https://uapi-group.org/specifications/specs/unified_kernel_image/))
* [UAPI.6 Configuration Files Specification](specs/configuration_files_specification.md):
  Standardises default locations and environment variables for locating common files or base directories.
  This is derived from, and extends, the [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir/latest/),
  to allow for separation between vendor and admin configuration files, drop-in files, and masking.
  ([canonical online location](https://uapi-group.org/specifications/specs/configuration_files_specification/))
* [UAPI.7 Linux TPM PCR Registry](specs/linux_tpm_pcr_registry.md):
  An informative list of how TPM PCRs are used on a Linux system.
  ([canonical online location](https://uapi-group.org/specifications/specs/linux_tpm_pcr_registry/))
* [UAPI.8 Package Metadata for Executable Files](specs/package_metadata_for_executable_files.md):
  Describes the format and mechanism to include packaging metadata in ELF/PE binaries.
  ([canonical online location](https://uapi-group.org/specifications/specs/package_metadata_for_executable_files/))
* [UAPI.9 Linux File System Hierarchy](specs/linux_file_system_hierarchy.md):
  Describes the layout of directories and files in an installation of Linux
  ([canonical online location](https://uapi-group.org/specifications/specs/linux_file_system_hierarchy/))
* [UAPI.10 Version Format Specification](specs/version_format_specification.md):
  Defines semantics of version strings used in the other specifications listed here.
  ([canonical online location](https://uapi-group.org/specifications/specs/version_format_specification/))
* [UAPI.11 File Hierarchy for the Verification of OS Artifacts (VOA)](specs/file_hierarchy_for_the_verification_of_os_artifacts.md):
  Describes the use of Discoverable Disk Images to create extensions to a base image.
  ([canonical online location](https://uapi-group.org/specifications/specs/file_hierarchy_for_the_verification_of_os_artifacts/))
* [UAPI.12 dlopen() Metadata for ELF Files](specs/elf_dlopen_metadata.md):
  Describes the format and mechanism to include dynamically loaded libraries metadata in ELF binaries.
  ([canonical online location](https://uapi-group.org/specifications/specs/elf_dlopen_metadata/))
* [UAPI.13 Efficient Time Synchronisation for Virtual Machines](specs/vmclock.md):
  Describes the format and mechanism to synchronize the guest clock.
  ([canonical online location](https://uapi-group.org/specifications/specs/vmclock/))
* [UAPI.14 Virtual Machine Generation ID](specs/vmgenid.md):
  Describes the mechanism for detecting virtual machine rollback events.
  ([canonical online location](https://uapi-group.org/specifications/specs/vmgenid/))
* [UAPI.15 OSC 3008: Hierarchical Context Signalling](specs/osc_context.md):
  Defines a mechanism for terminal emulators to follow the context hierarchy of what's on screen.
  ([canonical online location](https://uapi-group.org/specifications/specs/osc_context/))

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
- *ELF* – Executable and Linkable Format (Linux executable binary format)
- *MOK* – Machine Owner Key (shim)
- *PCR* – TPM Platform Configuration Registers
- *PE* – Portable Executable (UEFI executable binary format)
- *SBAT* – UEFI Secure Boot Advanced Targeting
- *TPM* – Trusted Platform Module (security chip)

## Terms and abbreviations specific to UAPI group specifications
- [*BLS*](specs/boot_loader_specification.md) - Boot Loader Specification
- [*confext*](specs/extension_image.md) – Configuration Extension Image
  (type of DDI that is overlayed on top of `/etc/` via overlayfs and can extend the underlying OS' configuration in a composable, immutable fashion)
- [*DDI*](specs/discoverable_disk_image.md) - Discoverable Disk Image
- [*DPS*](specs/discoverable_partitions_specification.md) - Discovery Partition Specification
- [*sysext*](specs/extension_image.md) – System Extension Image
  (type of DDI that is overlayed on top of `/usr/` and `/opt/` via overlayfs and can extend the underlying OS vendor resources in a composable, immutable fashion)
- [*UKI*](specs/unified_kernel_image.md) – Unified Kernel Images (UEFI boot stub + kernel + initrd + more)
- [*VMClock*](specs/vmclock.md) – Virtual Machine Clock (efficient time synchronisation for virtual machines)
- [*VMGenID*](specs/vmgenid.md) – Virtual Machine Generation ID (mechanism for detecting VM rollback events)
- [*VOA*](specs/file_hierarchy_for_the_verification_of_os_artifacts.md) – Verification of OS Artifacts

## Participate

Please use the [specifications issue tracker](https://github.com/uapi-group/specifications/issues) to engage with the project.
