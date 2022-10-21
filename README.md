# UAPI group specifications


# Userspace API Specifications

This repo contains specifications for uapi standards as well as informational documentation.
See [specs](specs/) for a list of all specs.

# Glossary

This section clarifies on terms and abbreviations used in specs and other documents throughout the organisation.

## General terms and abbreviations
- *MOK* - Machine Owner Key (shim)
- *PCR* – TPM Platform Configuration Registers
- *TPM* – Trusted Platform Module (security chip)
- *SBAT* – UEFI Secure Boot Advanced Targeting

## Terms and abbreviations specific to UAPI group specifications
- [*DDI*](specs/discoverable_partitions_specification.md) - Discoverable Disk Image
- [*DPS*](specs/discoverable_partitions_specification.md) - Discovery Partition Specification
- [*UKI*](specs/unified_kernel_image.md) - Unified Kernel Images (sd-stub + kernel + initrd + more)
- [*BLS*](specs/boot_loader_specification.md) - Boot Loader Specification
- *sysext* – System Extension Image (type of DDI that is overlayed on top of `/usr/` and `/opt/` via overlayfs and can extend the underlying OS vendor resources in a composable, immutable fashion)
- *syscfg* – System Configuration Image (type of DDI that is overlayed on top of `/etc/` via overlayfs and can extend the underlying OS' configuration in a composable, immutable fashion)

# Website generation

Refer to the [website](website/) folder's [README](website/README.md) for regenerating static web pages.
