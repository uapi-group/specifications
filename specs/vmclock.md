---
title: UAPI.13 VMClock
category: Concepts
layout: default
version: 1.0
SPDX-License-Identifier: CC-BY-4.0
weight: 13
aliases:
- /UAPI.13
- /13
---

# UAPI.13 VMClock: Efficient Time Synchronisation for Virtual Machines

| Version | Changes         |
|---------|-----------------|
| 1.0     | Initial Release |

The requirements for accurate synchronisation of application clocks against
real wallclock time are becoming ever more demanding. Increasingly cloud
providers are exposing precision clock devices to virtual machines to allow the
guest operating systems to synchronise their clocks.

Time on modern systems is typically derived from a CPU-internal counter (*TSC,
timebase, arch counter*) which runs at a nominally constant frequency of
typically between 1GHz and 4GHz. In practice, the frequency of the underlying
hardware counter will vary with environmental conditions, with a tolerance of
the order of ±50PPM (parts per million). It is this variance which must constantly be corrected by
synchronising against an external clock.

Synchronisation against an external clock typically works by reading the CPU
counter, then reading the external clock, and finally reading the CPU counter
again — then assuming that the external clock reading was concurrent with a
point in time between the two CPU counter readings to give a pair of `{ CPU counter, real time }`
values. Successive such readings are used to calibrate the
precise rate at which the CPU counter is running, in order to use it for
precision timekeeping.

When applied at scale to virtual machines, there are a number of problems with
this approach. Firstly, where virtual CPUs are overcommitted across a smaller
number of physical CPUs in a host, guests experience "steal time" — time when
their vCPU is not actually running. That steal time is unpredictable and can
occur in the critical period between one read of the CPU counter and the next,
affecting the precision of the estimated reading.

A remedy for this issue is to repeat the reading a number of times, and to use
the result where the latency between first and last CPU counter reading is the
lowest. This exacerbates the second problem, that a large number of separate
guest operating systems on the same host are now repeating the same work of
calibrating the *same* underlying hardware oscillator.

The third major problem of guest-calibrated time is Live Migration, in which a
guest is transparently moved from one host to another for maintenance reasons.
When this happens, the guest can experience a step change in both the frequency
and the value of the CPU counter. The frequency because the migrated guest is
now using a different underlying counter, and the value because correctly
setting the counter value seen by the guest is dependent on the time
synchronisation of each hypervisor host. After a Live Migration, a guest's
clock should be considered inaccurate until it has been resynchronised from
scratch. Failure to do so can lead to data corruption, in cases where database
coherency depends on accurately timestamped transactions.

## The VMClock device

The VMClock device resolves the above issues by allowing the hypervisor to
synchronise the hardware clock against external time, and simply present the
results to each guest in a shared memory region in the form of a formula for
converting the CPU counter into real time. This allows guests to have precision
timestamps even immediately after a Live Migration event, and with no need to
provide further clock devices to the guest or for guests to spend their own CPU
time on calibration.

For guests which do perform their own additional refinement of the clock via
NTP or other means, a disruption signal is provided which allows them to
discard any such refinement after Live Migration, and start again with the data
from the new hypervisor host.

## The vmclock_abi structure

The hypervisor provides a structure in shared memory which is readable by the
guest, and advertises it via either ACPI or device-tree devices as described
below. Where possible, these fields and their values are aligned with the
definitions in the [virtio-rtc](https://virtio-rtc) standard. As with virtio,
all fields are stored in little-endian form.

The fields up to and including `time_type` are constant and shall not change
during the lifetime of the device. The subsequent fields may be updated
dynamically, using `seq_count` as a synchronisation mechanism as follows:

1. Increase `seq_count` to an odd value.
2. Update the remaining fields in the structure.
3. Increase `seq_count` again to an even value.
4. If `VMCLOCK_FLAG_NOTIFICATION_PRESENT` is set in the `flags` field, raise an interrupt or ACPI notification.

If memory barriers are necessary to ensure that changes to the memory are
visible to the guest, they should be present at each stage. The total amount of
time during which `seq_count` remains at an odd value shall be short enough
that it is reasonable for a guest to *spin* while waiting for the update to
complete, as described below.

### Structure Fields

<table>
  <thead>
    <tr>
      <th>Offset</th>
      <th>Field</th>
      <th>Description</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>0x00</td>
      <td><code>uint32_t magic</code></td>
      <td>Magic value <code>0x4b4c4356</code> (“VCLK”)</td>
    </tr>
    <tr>
      <td>0x04</td>
      <td><code>uint32_t size</code></td>
      <td>Size of region containing this structure (typically a full page at
        the granularity at which the hypervisor maps memory to the guest)</td>
    </tr>
    <tr>
      <td>0x08</td>
      <td><code>uint16_t version</code></td>
      <td>This standard defines version 1. Since the <code>flags</code> field
        allows for extensions to the data structure without breaking backward
        compatibility, it is not anticipated that the <code>version</code> field
        will ever need to change.</td>
    </tr>
    <tr>
      <td>0x0a</td>
      <td><code>uint8_t counter_id</code></td>
      <td>The hardware counter used as the basis for clock readings. The
        values of this field correspond to the
        <code>VIRTIO_RTC_COUNTER_xxx</code> values:
        <ul>
          <li><code>0x00</code>: <code>VMCLOCK_COUNTER_ARM_VCNT</code>: The Arm
            architectural timer (virtual)</li>
          <li><code>0x01</code>: <code>VMCLOCK_COUNTER_X86_TSC</code>: The x86
            Time Stamp Counter</li>
          <li><code>0xFF</code>: <code>VMCLOCK_COUNTER_INVALID</code>: No
            precision clock is advertised</li>
        </ul>
      </td>
    </tr>
    <tr>
      <td>0x0b</td>
      <td><code>uint8_t time_type</code></td>
      <td>Indicates the type of clock exposed through this interface. The
        values of this field correspond to the <code>VIRTIO_RTC_CLOCK_xxx</code>
        values, except that smearing of clocks is not supported as it is
        antithetical to precision:
        <ul>
          <li><code>0x00</code>: <code>VMCLOCK_TIME_UTC</code> <em>(Not
              recommended)</em></li>
          <li><code>0x01</code>: <code>VMCLOCK_TIME_TAI</code></li>
          <li><code>0x02</code>: <code>VMCLOCK_MONOTONIC</code></li>
        </ul>
        For UTC and TAI, the calculation results in a number of seconds
        since midnight on 1970-01-01. A monotonic clock has no defined epoch.
        Since UTC has leap seconds and a given numbered second may occur more
        than once, its use is <strong>NOT RECOMMENDED</strong> in VMClock.
        Implementations should advertise TAI, with a correct UTC offset.
      </td>
    </tr>
    <tr>
      <td>0x0c</td>
      <td><code>uint32_t seq_count</code></td>
      <td>This field is used to provide a sequence-based read/write lock for
        the non-constant fields which follow. To perform an update, the device
        will:
        <ul>
          <li>Increment this field to an odd value (with the low bit set)</li>
          <li>Change other fields as appropriate.</li>
          <li>Increment this field again to an even value.</li>
        </ul>
      </td>
    </tr>
    <tr>
      <td>0x10</td>
      <td><code>uint64_t disruption_marker</code></td>
      <td>This field is changed each time there may be a disruption to the
        hardware counter referenced by <code>counter_id</code>, for example
        through live migration to a new hypervisor host.</td>
    </tr>
    <tr>
      <td>0x18</td>
      <td><code>uint64_t flags</code></td>
      <td>Feature flags (see below)</td>
    </tr>
    <tr>
      <td>0x20</td>
      <td><code>uint16_t pad</code></td>
      <td>Unused</td>
    </tr>
    <tr>
      <td>0x22</td>
      <td><code>uint8_t clock_status</code></td>
      <td>Synchronisation status of the clock (see below)</td>
    </tr>
    <tr>
      <td>0x23</td>
      <td><code>uint8_t leap_second_smearing_hint</code></td>
      <td>Smearing hint for guest OS (see below)</td>
    </tr>
    <tr>
      <td>0x24</td>
      <td><code>int16_t tai_offset_sec</code></td>
      <td>Signed offset from TAI to UTC at the reference time specified in
        <code>time_sec</code> and <code>time_frac_sec</code>, in seconds. Valid
        if the corresponding bit in the flags field is set. Implementations
        SHOULD populate this field; the value at time of writing is 37.</td>
    </tr>
    <tr>
      <td>0x26</td>
      <td><code>uint8_t leap_indicator</code></td>
      <td>Indicates the presence and direction of a leap second occurring in
        the near future or recent past (see below)</td>
    </tr>
    <tr>
      <td>0x27</td>
      <td><code>uint8_t counter_period_shift</code></td>
      <td>Additional shift applied to all the
        <code>counter_period*_frac_sec</code> fixed-point fields.</td>
    </tr>
    <tr>
      <td>0x28</td>
      <td><code>uint64_t counter_value</code></td>
      <td>Value of the hardware counter at the time represented by
        <code>time_sec</code> + <code>time_frac_sec</code>.</td>
    </tr>
    <tr>
      <td>0x30</td>
      <td><code>uint64_t counter_period_frac_sec</code></td>
      <td>Period of a single counter tick, in units of 1 &gt;&gt; (64 +
        <code>counter_period_shift</code>)</td>
    </tr>
    <tr>
      <td>0x38</td>
      <td><code>uint64_t counter_period_esterror_rate_frac_sec</code></td>
      <td>Estimated ± error of <code>counter_period_frac_sec</code> in the
        same units.</td>
    </tr>
    <tr>
      <td>0x40</td>
      <td><code>uint64_t counter_period_maxerror_rate_frac_sec</code></td>
      <td>Maximum ± error of <code>counter_period_frac_sec</code> in the same
        units.</td>
    </tr>
    <tr>
      <td>0x48</td>
      <td><code>uint64_t time_sec</code></td>
      <td>Reference time point, seconds since epoch defined by
        <code>time_type</code> field.</td>
    </tr>
    <tr>
      <td>0x50</td>
      <td><code>uint64_t time_frac_sec</code></td>
      <td>Fractional part of reference time, in units of second / 2⁶⁴.</td>
    </tr>
    <tr>
      <td>0x58</td>
      <td><code>uint64_t time_esterror_nanosec</code></td>
      <td>Estimated ± error of the time given in <code>time_sec</code> +
        <code>time_frac_sec</code>, in nanoseconds</td>
    </tr>
    <tr>
      <td>0x60</td>
      <td><code>uint64_t time_maxerror_nanosec</code></td>
      <td>Maximum ± error of the time given in <code>time_sec</code> +
        <code>time_frac_sec</code>, in nanoseconds</td>
    </tr>
    <tr>
      <td>0x64</td>
      <td><code>uint64_t vm_generation_count</code></td>
      <td>A change in this field indicates that the guest has been cloned or
        loaded from a snapshot (see below).</td>
    </tr>
    <tr>
      <td>0x68</td>
      <td>…</td>
      <td>The size of the memory region containing this structure is given in
        the <code>size</code> field, which will typically be a full 4KiB page.
        New fields may be added here, advertised by newly-defined bits in the
        <code>flags</code> field, without changing the <code>version</code>
        field.</td>
    </tr>
  </tbody>
</table>

### Feature Flags (0x18)

| Bit | Flag                                  | Description                                                                                                                          |
|-----|---------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| 0   | `VMCLOCK_FLAG_TAI_OFFSET_VALID`       | Indicates that the `tai_offset` field below contains a correct value. All implementations SHOULD set this bit.                       |
| 1   | `VMCLOCK_FLAG_DISRUPTION_SOON`        | Indicates that a clock disruption event (e.g. live migration) is expected to happen in the next day or so.                           |
| 2   | `VMCLOCK_FLAG_DISRUPTION_IMMINENT`    | Indicates that a clock disruption event is expected to happen within the next hour or so.                                            |
| 3   | `VMCLOCK_FLAG_PERIOD_ESTERROR_VALID`  | Indicates that `counter_period_esterror_rate_frac_sec` contains valid data.                                                          |
| 4   | `VMCLOCK_FLAG_PERIOD_MAXERROR_VALID`  | Indicates that `counter_period_maxerror_rate_frac_sec` contains valid data.                                                          |
| 5   | `VMCLOCK_FLAG_TIME_ESTERROR_VALID`    | Indicates that `time_esterror_nanosec` contains valid data.                                                                          |
| 6   | `VMCLOCK_FLAG_TIME_MAXERROR_VALID`    | Indicates that `time_maxerror_nanosec` contains valid data.                                                                          |
| 7   | `VMCLOCK_FLAG_VM_GEN_COUNTER_PRESENT` | Indicates that the `vm_generation_counter` field is present.                                                                         |
| 8   | `VMCLOCK_FLAG_NOTIFICATION_PRESENT`   | Indicates that the VMClock device will send an interrupt or ACPI notification every time it updates `seq_count` to a new even value. |

Unknown flags set by the device can safely be ignored. If a change in behaviour
is required by a future version of this specification, it would come with a new
value of the `version` field or a new `time_type` to avoid breaking
compatibility with existing users.

### Clock Status (0x22)

| Value | Status                        | Description                                                                                                                                                                        |
|-------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 0x00  | `VMCLOCK_STATUS_UNKNOWN`      | The clock is in an indeterminate state. Clock parameters in the VMClock structure are not valid and should not be relied upon.                                                     |
| 0x01  | `VMCLOCK_STATUS_INITIALIZING` | The clock is being initialized and is not yet synchronized. Clock parameters in the VMClock structure are not valid and should not be relied upon.                                 |
| 0x02  | `VMCLOCK_STATUS_SYNCHRONIZED` | The clock is synchronized. Clock parameters in the VMClock structure are expected to be correct and may be relied upon.                                                            |
| 0x03  | `VMCLOCK_STATUS_FREERUNNING`  | The clock has transitioned away from being synchronized and is in a free-running state. Clock parameters in the VMClock structure are expected to be valid and may be relied upon. |
| 0x04  | `VMCLOCK_STATUS_UNRELIABLE`   | The clock is considered broken. Clock parameters in the VMClock structure should not be relied upon.                                                                               |

### Leap Second Smearing Hint (0x23)

The time exposed through the VMClock device shall never be smeared. This field
corresponds to the `subtype` field in virtio-rtc, which indicates a smearing
method. In this case it merely provides a hint to the guest operating system,
such that if the guest OS wants to provide its users with an alternative clock
which does not follow UTC, it may do so in a fashion consistent with the other
systems in the nearby environment.

| Value | Hint                           |
|-------|--------------------------------|
| 0x00  | `VMCLOCK_SMEARING_STRICT`      |
| 0x01  | `VMCLOCK_SMEARING_NOON_LINEAR` |
| 0x02  | `VMCLOCK_SMEARING_UTC_SLS`     |

### Leap Indicator (0x26)

The value of this field shall be valid for the point in time referenced by the
`time_sec` and `time_frac_sec` fields.

| Value | Indicator               | Description                                                                    |
|-------|-------------------------|--------------------------------------------------------------------------------|
| 0x00  | `VMCLOCK_LEAP_NONE`     | No known nearby leap second                                                    |
| 0x01  | `VMCLOCK_LEAP_PRE_POS`  | A positive leap second will occur at the end of the present month              |
| 0x02  | `VMCLOCK_LEAP_PRE_NEG`  | A negative leap second will occur at the end of the present month              |
| 0x03  | `VMCLOCK_LEAP_POS`      | A positive leap second is currently occurring (set during the 23:59:60 second) |
| 0x04  | `VMCLOCK_LEAP_POST_POS` | A positive leap second occurred at the end of the previous month               |
| 0x05  | `VMCLOCK_LEAP_POST_NEG` | A negative leap second occurred at the end of the previous month               |

### VM Generation Count (0x64)

This field indicates that the guest has been cloned or loaded from a snapshot. The operating system may wish to regenerate unique identifiers, reset network connections or reseed entropy, etc.

The conditions under which this counter changes are identical to those of the [VMGenID device](vmgenid.md). The `vm_generation_count` changes whenever the VM is restored to an earlier or non-unique state:

  - Snapshot restoration
  - Backup recovery
  - VM cloning/copying/import
  - Disaster recovery failover

The `vm_generation_count` remains constant during normal VM operations:

  - Pause/resume
  - Shutdown/restart/reboot
  - Host reboot or upgrade
  - Live migration or lossless online failover

The `disruption_marker` and `vm_generation_count` fields indicate two orthogonal, but sometimes correlated, types of event. It is generally likely that the `disruption_marker` would also be changed when the `vm_generation_count` changes, but not necessarily vice versa.

It is possible that a VM could be cloned (forked) while running on the same host, such that the precision of the hardware counter is not lost, but the uniqueness is. That would be the rare case where the `vm_generation_count` would be changed but not the `disruption_marker`.

## Calculating time

The VMClock structure provides the following values:

- Reference time T₁ in the `time_sec` and `time_frac_sec` fields
- Counter value C₁ of the hardware counter at time T₁ in the `counter_value` field.
- The period P of a single counter tick is given by `counter_period_frac_sec` >> `counter_period_shift`.

For example, a 1GHz clock would have a period of 1ns, which could naïvely be
represented as `0x44B82FA0A / 2⁶⁴` by putting that value in
`counter_period_frac_sec`. Over long periods of time, however, the loss of
precision would be noticeable. So the same 1ns period should be more precisely
represented as `0x89705F4136B4A597 / 2^(64+29)` by using that value in
`counter_period_frac_sec` and setting `counter_period_shift` to 29.

To calculate the time, the guest shall first read the `seq_count` field and
wait until it returns an even value, then read the hardware counter C_now and
calculate the time accordingly as **T₁ + P(C_now - C₁)**. Finally, read the
`seq_count` field again. If the value of the `seq_count` field has changed,
discard the result and repeat the procedure from the beginning.

Where UTC is involved, a correct implementation will need to cope with the case
where a leap second has occurred since the reference time T₁, and the result
needs to be adjusted accordingly. The `leap_indicator` field exists to resolve
the technical ambiguity but using TAI is simpler and less error prone. It is
strongly recommended that implementations use TAI as the time standard and
advertise a correct TAI offset, to avoid this complexity.

## Time error calculation

The VMClock structure optionally advertises maximum error bounds for the clock
data it provides, in the form of deltas to the T₁ and P values used above. The
true time is guaranteed to be within:

**T₁ ± T_maxerr + P ± P_maxerr(C_now - C₁)**

where T_maxerr and P_maxerr are the `time_maxerror_nanosec` and `counter_period_maxerror_rate_frac_sec` fields, respectively.

The device may update the time calibration fields at any time, by incrementing
the `seq_count` to an odd value, adjusting the parameters, then incrementing
`seq_count` again to an even value. For any given historical counter reading
and the error bounds calculated according to VMClock at that moment, it is
guaranteed that any *subsequent* update to the VMClock fields shall also result
in a calculation for that same counter value which falls between the earliest
and latest times that were previously indicated.

## Discovery via ACPI

To expose VMClock to the operating system via ACPI, the firmware or hypervisor must:

1. Place the shared `vmclock_abi` structure somewhere in RAM, ROM or device memory space, which is guaranteed not to be used by the operating system. It must not be in ranges reported as `AddressRangeMemory` or `AddressRangeACPI`, and must not be in the same page as any memory which is expected to be mapped by a page table entry with caching disabled.

2. Expose a device somewhere in the ACPI namespace with:
   - a hardware ID (`_HID`) of "AMZNC10C"
   - a DOS Device Name ID (`_DDN`) of "VMCLOCK"
   - a compatible ID (`_CID`) of "VMCLOCK"

3. Attach to the device a "`_CRS`" method which when evaluated describes the shared memory page where the hypervisor has stored the `vmclock_abi` structure.

4. Optionally, the device can raise an ACPI Notify operation using notification code 0x80, every time the `seq_count` field changes to a new even number. If implemented, the hypervisor must advertise the notification feature to the driver by setting the `VMCLOCK_FLAG_NOTIFICATION_PRESENT` bit in the `flags` field.

## Discovery via Device Tree

Similar to the ACPI binding above, the firmware or hypervisor must place the
`vmclock_abi` structure in an otherwise unused region of physical memory and
advertise its presence to the operating system. The Device Tree binding for the
`amazon,vmclock` node is as follows:

```yaml
# SPDX-License-Identifier: (GPL-2.0-only OR BSD-2-Clause)
%YAML 1.2
---
$id: http://devicetree.org/schemas/clock/amazon,vmclock.yaml#
$schema: http://devicetree.org/meta-schemas/core.yaml#

title: Virtual Machine Clock

maintainers:
  - David Woodhouse <dwmw2@infradead.org>

description:
  The vmclock device provides a precise clock source and allows for
  accurate timekeeping across live migration and snapshot/restore
  operations. The full specification of the shared data structure
  is available at https://david.woodhou.se/VMClock.pdf

properties:
  compatible:
    const: amazon,vmclock

  reg:
    description:
      Specifies the shared memory region containing the vmclock_abi structure.
    maxItems: 1

  interrupts:
    description:
      Interrupt used to notify when the contents of the vmclock_abi structure
      have been updated.
    maxItems: 1

required:
  - compatible
  - reg

additionalProperties: false

examples:
  - |
    #include <dt-bindings/interrupt-controller/arm-gic.h>
    ptp@80000000 {
      compatible = "amazon,vmclock";
      reg = <0x80000000 0x1000>;
      interrupts = <GIC_SPI 36 IRQ_TYPE_EDGE_RISING>;
    };
```

## Hardware implementation

It is possible for a hardware implementation of VMClock to exist, in the
absence of a hypervisor or virtualization. Using mechanisms such as PCIe PTP, a
device could synchronise the CPU's counter directly against real time and
advertise the result to the operating system.

Such an implementation is outside the scope of this specification for now, but
only just. We may need to add a new option for the `counter_id` field which
references the hardware clock available to the PCIe device for PTM
synchronisation, for example the Intel Always Running Timer.
