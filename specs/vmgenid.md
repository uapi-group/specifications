---
title: UAPI.14 VMGenID
category: Concepts
layout: default
version: 1.0
SPDX-License-Identifier: CC-BY-4.0
weight: 14
aliases:
- /UAPI.14
- /14
---

# UAPI.14 VMGenID: Virtual Machine Generation ID

| Version | Changes |
|---------|---------|
| 1.0     | Initial Release |

Virtual machine operations that restore a VM to an earlier point in time (such as applying snapshots, restoring from backup, cloning, or failover scenarios) can cause serious problems for applications that depend on unique identifiers or cryptographic entropy. The Virtual Machine Generation ID (VMGenID) device provides a mechanism for guest software to detect when such operations have occurred.

The VMGenID is a 128-bit cryptographically random identifier that changes whenever a virtual machine is cloned or restored to an earlier state. This allows applications to detect such events and take appropriate protective measures, such as reseeding random number generators, regenerating unique identifiers, or invalidating cached state.

## The vmgenid_abi Structure

The hypervisor provides 16 bytes in shared memory containing the generation ID. The structure can be represented as two little-endian 64-bit values, and must be placed in an 8-byte aligned buffer.

### Structure Fields

| Offset | Field | Description |
|--------|-------|-------------|
| 0x00 | `uint64_t generation_id_low` | Lower 64 bits of the 128-bit generation ID |
| 0x08 | `uint64_t generation_id_high` | Upper 64 bits of the 128-bit generation ID |

The generation ID is a 128-bit cryptographically random value that is unique across all VM instances and time. All 128 bits are random; it is *not* a Version 4 UUID.

The generation ID changes whenever the VM is restored to an earlier or non-unique state:

  - Snapshot restoration
  - Backup recovery
  - VM cloning/copying/import
  - Disaster recovery failover

The generation ID remains constant during normal VM operations:

  - Pause/resume
  - Shutdown/restart/reboot
  - Host reboot or upgrade
  - Live migration or lossless online failover

Events, such as live migrations, which merely disrupt the VM's clock without changing the uniqueness of its identity do not result in a change to the generation ID. Conversely, cloning (forking) a running VM running on the same host would result in a new generation ID without disrupting the timekeeping. Guests which want to detect clock disruption should use the [VMClock device](vmclock.md) for that purpose.

### GUID interoperability

If the generation ID is represented as a GUID for the purpose of storage or configuration by a Virtual Machine Monitor, it is recommended that:

- The generation ID shared to the guest is the little-endian representation of that GUID
- The textual representation of the GUID, in display or configuration, is the RFC 4122 standard big-endian form


## Discovery via ACPI

To expose VMGenID to the operating system via ACPI, the firmware or hypervisor must:

1. Place the shared `vmgenid_abi` structure somewhere in RAM, ROM or device memory space, which is guaranteed not to be used by the operating system. It must not be in ranges reported as `AddressRangeMemory` or `AddressRangeACPI`, and must not be in the same page as any memory which is expected to be mapped by a page table entry with caching disabled.

2. Expose a device somewhere in the ACPI namespace with:
   - a hardware ID (`_HID`) that is hypervisor-specific
   - a DOS Device Name ID (`_DDN`) of "VM_Gen_Counter"
   - a compatible ID (`_CID`) of "VM_Gen_Counter"

3. Attach to the device an `ADDR` method which when evaluated returns the 64-bit physical address of the generation ID structure as a package containing the low and high 32-bit address components in that order.

4. After the generation ID changes, the device shall raise an ACPI Notify operation using notification code 0x80. The device may raise the notify operation even if the generation ID has not changed.

## Discovery via Device Tree

The firmware or hypervisor must place the `vmgenid_abi` structure in an otherwise unused region of physical memory and advertise its presence to the operating system. The Device Tree binding for the `microsoft,vmgenid` node is as follows:

```yaml
# SPDX-License-Identifier: (GPL-2.0-only OR BSD-2-Clause)
%YAML 1.2
---
$id: http://devicetree.org/schemas/rng/microsoft,vmgenid.yaml#
$schema: http://devicetree.org/meta-schemas/core.yaml#

title: Virtual Machine Generation ID

maintainers:
  - Jason A. Donenfeld <Jason@zx2c4.com>

description:
  Firmwares or hypervisors can use this devicetree to describe an
  interrupt and a shared resource to inject a Virtual Machine Generation ID.
  Virtual Machine Generation ID is a globally unique identifier (GUID) and
  the devicetree binding follows VMGenID specification.

properties:
  compatible:
    const: microsoft,vmgenid

  reg:
    description:
      Specifies a 16-byte VMGenID in endianness-agnostic hexadecimal format.
    maxItems: 1

  interrupts:
    description:
      Interrupt used to notify that a new VMGenID is available.
    maxItems: 1

required:
  - compatible
  - reg
  - interrupts

additionalProperties: false

examples:
  - |
    #include <dt-bindings/interrupt-controller/arm-gic.h>
    rng@80000000 {
      compatible = "microsoft,vmgenid";
      reg = <0x80000000 0x1000>;
      interrupts = <GIC_SPI 35 IRQ_TYPE_EDGE_RISING>;
    };
```
