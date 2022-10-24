---
title: Discoverable Disk Image
category: Concepts
layout: default
version: 1
SPDX-License-Identifier: CC-BY-4.0
---
# Discoverable Disk Image (DDI)

DDIs (Discoverable Disk Images) are self-described file system images that follow the DPS
([Discoverable Partitions Specification](discoverable_partitions_specification.md)), wrapped in a GPT
partition table, that may contain root (or `/usr/`) filesystems, system extensions, system configurations,
portable services, containers and more, and are all protected by signed dm-verity all combined into one.
They are designed to be composable and stackable, and provide security by default.

## Image Format
The images use the GPT partition table verbatim, so it will not be redefined here. Each partition contains
a standard Linux filesystem (e.g.: squashfs), so again this will not be redefined here.
The ([DPS](discoverable_partitions_specification.md)) defines the GUIDs to use and the format of the
dm-verity signature partition's JSON content.
