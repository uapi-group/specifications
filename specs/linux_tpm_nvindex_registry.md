---
title: Linux TPM 2.0 NV Index Registry
version: 1
SPDX-License-Identifier: CC0-1.0
---
# 🔏 Linux TPM 2.0 NV Index Registry 🗒️

The Trusted Computing Group (TCG) maintains a [Registry of Reserved TPM 2.0 Handles and
Localities](https://trustedcomputinggroup.org/resource/registry/) which assigns TPM 2.0 NV index ranges
(among ther things, see section 2.2) to organizations (by convention only!). They have assigned the NV index
range **0x01D10200-0x01D105FF** to the Linux community.

This registry tracks assignments of subranges of this NV index range to various interested Linux
projects. Currently, the following subranges are assigned:

| Subrange             |      # | Project | Reference                                   |
|----------------------|--------|---------|---------------------------------------------|
|0x01D10200-0x01D10224 |     36 | systemd | https://systemd.io/TPM2_NVINDEX_ASSIGNMENTS |

If you maintain a Linux Open Source project and would like to to have a subrange delegated to your project,
please file a suitable [Pull Request](https://github.com/uapi-group/specifications/pulls).

Note that while TPM 2.0 NV indexes are not quite as scarce as PCRs they still aren't free. Hence, please
request only minimal ranges for your purposes.

We will not delegate subranges to projects that aren't under an Open Source license. For NV index delegations
to commercial projects please contact the TCG directly.
