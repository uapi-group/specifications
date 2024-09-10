---
title: UAPI.19 Linux TPM 2.0 NV Index Registry
category: Concepts
layout: default
version: 0.1
SPDX-License-Identifier: CC-BY-4.0
weight: 19
aliases:
- /UAPI.19
- /19
---
# 🔏 UAPI.19 Linux TPM 2.0 NV Index Registry 🗒️

| Version | Changes         |
|---------|-----------------|
| 0.1     | Initial Release |

The Trusted Computing Group (TCG) maintains a [Registry of Reserved
TPM 2.0 Handles and
Localities](https://trustedcomputinggroup.org/resource/registry/)
which assigns TPM 2.0 NV index ranges (among other things, see section
2.2) to organizations (by convention only!). They have assigned the NV
index range **0x01D10200-0x01D105FF** (i.e. 1024 indexes) to the Linux
community.

This registry tracks assignments of subranges of this NV index range
to various interested Linux projects. Currently, the following
subranges are assigned:

| Subrange              |  # | Project | Reference                                                               |
|-----------------------|----|---------|-------------------------------------------------------------------------|
| 0x01D10200-0x01D10224 | 37 | systemd | [TPM2_NVINDEX_ASSIGNMENTS](https://systemd.io/TPM2_NVINDEX_ASSIGNMENTS) |
| 0x01D10225-0x01D10225 |  1 | Linux   | Used to distinguish between kernel and userland TPM usage               |

If you maintain a Linux Open Source project and would like to have
a subrange delegated to your project, please file a suitable [Pull
Request](https://github.com/uapi-group/specifications/pulls). The pull
request should include the name of the project, the number of NV
indexes requested together with a brief justification, and a link to
publicly accessible documentation describing how the project makes use
of each of the indexes.

New subranges are assigned contiguously, starting from the lowest
unassigned NV index of the range.

Note that while TPM 2.0 NV indexes are not quite as scarce as PCRs
they are still a limited resource. Hence, please request only minimal
ranges for your purposes.

We will not delegate subranges to projects that aren't under a license
[approved by the Open Source
Initiative](https://opensource.org/licenses). For NV index delegations
to commercial projects please contact the TCG directly.

Also see [UAPI.7 Linux TPM PCR Registry](linux_tpm_pcr_registry.md).
