---
title: Linux TPM PCR Registry
version: 1
SPDX-License-Identifier: CC0-1.0
---

# 🔏 Linux TPM PCR Registry 🗒️

_TPM PCRs are a scarce resource, there are only 24 of them in typical standards compliant TPMs. According to the [TCG PC Client Specific Platform Firmware Profile Specification | Trusted Computing Group](https://trustedcomputinggroup.org/resource/pc-client-specific-platform-firmware-profile-specification/) PCRs 8…15 are for the OS to make use of. In this document we intend to document for Linux platforms which component is using which PCR in order to minimize conflicts._

PCRs owned by the firmware, i.e. PCRs 0–7 are described here just for convenience.
The authoriative description is in the TCG document.
How other operating systems — in particular Windows — use PCRs, is out of scope of this document.

This document is informational in nature: it just describes what is, it is not intended to formally declare “ownership” of a specific PCR, but simply is supposed to reflect which PCR assignments are common in the Linux ecosystems. That said, co-opting PCR usage will likely create problems down the line, in particular if measurement logs are maintained separately. (To be more explicit: on `systemd` systems the warranty is voided if you write to the PCRs it also uses, as per the list below.)

PCR measurements most commonly serve two distinct purposes:

* To implement access policy on TPM sealed objects: policy can dictate that unsealing of such objects shall only be allowed if some PCRs are in a specific literal state, or in any state for which a signature by a specific key pair can be provided. For this it is essential that PCRs only contain measurements for a clearly defined set of objects, that typically is known in advance so that the PCR value can be pre-calculated (hence this is in a way a _forward_-looking use)
* To permit reasoning about the boot process and runtime _so far_, for example for the purpose of remote attestation. In this case it is not that important what objects are measured as long as a record is kept in a measurement log about what it was. The PCRs are in this case used to validate that log (hence this is in a way a _backward_-looking use)

In both cases it is important that data measured into the PCRs is carefully chosen. PCRs that shall be useful for policy binding should only cover data objects known in advance, and thus not contain runtime data that cannot be pre-calculated in advance. PCRs that shall be useful for backward-looking validation should only cover objects that are also written to the appropriate log for the PCR.

<table style="width:100%; display:block; table-layout:fixed;">
  <tr>
   <th><p style="text-align: right"><strong>PCR#</strong></p></th>
   <th><strong>Used by</strong></th>
   <th><strong>From Location</strong></th>
   <th><strong>Measured Objects</strong></th>
   <th><strong>Log</strong></th>
   <th><strong>Use Reported By</strong></th>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>0</strong></p></td>
   <td style="background-color:#838284;"><code style="background-color:#838284;">Firmware 💻</code></td>
   <td>UEFI Boot Component</td>
   <td>Core system firmware executable code</td>
   <td>UEFI TPM event log</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>1</strong></p></td>
   <td style="background-color:#838284;"><code style="background-color:#838284;">Firmware 💻</code></td>
   <td>UEFI Boot Component</td>
   <td>Core system firmware data/host platform configuration; typically contains serial and model numbers</td>
   <td>UEFI TPM event log</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>2</strong></p></td>
   <td style="background-color:#838284;"><code style="background-color:#838284;">Firmware 💻</code></td>
   <td>UEFI Boot Component</td>
   <td>Extended or pluggable executable code; includes option ROMs on pluggable hardware</td>
   <td>UEFI TPM event log</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>3</strong></p></td>
   <td style="background-color:#838284;"><code style="background-color:#838284;">Firmware 💻</code></td>
   <td>UEFI Boot Component</td>
   <td>Extended or pluggable firmware data; includes information about pluggable hardware</td>
   <td>UEFI TPM event log</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>4</strong></p></td>
   <td style="background-color:#838284;"><code style="background-color:#838284;">Firmware 💻</code></td>
   <td>UEFI Boot Component</td>
   <td>Boot loader and additional drivers; binaries and extensions loaded by the boot loader</td>
   <td>UEFI TPM event log</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>5</strong></p></td>
   <td style="background-color:#838284;"><code style="background-color:#838284;">Firmware 💻</code></td>
   <td>UEFI Boot Component</td>
   <td>GPT/Partition table</td>
   <td>UEFI TPM event log</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>7</strong></p></td>
   <td style="background-color:#838284;"><code style="background-color:#838284;">Firmware 💻</code></td>
   <td>UEFI Boot Component</td>
   <td>SecureBoot state</td>
   <td>UEFI TPM event log</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>8</strong></p></td>
   <td style="background-color:#AEA;"><code style="background-color:#AEA;">grub 🍲</code></td>
   <td>UEFI Boot Component</td>
   <td>Commands and kernel command line</td>
   <td>UEFI TPM event log</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>9</strong></p></td>
   <td style="background-color:#AEA;"><code style="background-color:#AEA;">grub 🍲</code></td>
   <td>UEFI Boot Component</td>
   <td>All files read (including kernel image)</td>
   <td>UEFI TPM event log</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"></td>
   <td style="background-color:#b399c2;">Linux kernel 🌰</td>
   <td>Kernel</td>
   <td>All passed initrds (when the new <code>LOAD_FILE2 </code>initrd protocol is used)</td>
   <td>UEFI TPM event log</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>10</strong></p></td>
   <td style="background-color:#a3c2d4;">IMA 📐</td>
   <td>Kernel</td>
   <td>Protection of the IMA measurement log</td>
   <td>IMA event log</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>11</strong></p></td>
   <td style="background-color:#e5c8e6;"><code style="background-color:#e5c8e6;">systemd-stub 🚀</code></td>
   <td>UEFI Stub</td>
   <td>All components of unified kernel images (UKIs)</td>
   <td>UEFI TPM event log</td>
   <td>in EFI variable <code>StubPcrKernelImage</code></td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"></td>
   <td style="background-color:#e5c8e6;"><code style="background-color:#e5c8e6;">systemd-pcrphase 🚀</code></td>
   <td>Userspace</td>
   <td>Boot phase strings, indicating various milestones of the boot process</td>
   <td>Journal (for now)</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>12</strong></p></td>
   <td style="background-color:#e5c8e6;"><code style="background-color:#e5c8e6;">systemd-stub 🚀</code></td>
   <td>UEFI Stub</td>
   <td>Kernel command line, system credentials and system configuration images</td>
   <td>UEFI TPM event log</td>
   <td>in EFI variable <code>StubPcrKernelParameters</code></td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>13</strong></p></td>
   <td style="background-color:#e5c8e6;"><code style="background-color:#e5c8e6;">systemd-stub 🚀</code></td>
   <td>UEFI Stub</td>
   <td>All system extension images for the initrd</td><td>UEFI TPM event log</td>
   <td>in EFI variable <code>StubPcrInitRDSysExts</code></td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>14</strong></p></td>
   <td style="background-color:#f5d97d;"><code style="background-color:#f5d97d;">shim 🔑</code></td>
   <td>UEFI Boot Component</td>
   <td>“MOK” certificates and hashes</td>
   <td>UEFI TPM event log</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"><p style="text-align: right"><strong>15</strong></p></td>
   <td style="background-color:#e5c8e6;"><code style="background-color:#e5c8e6;">systemd-cryptsetup@.service 🚀</code></td>
   <td>Userspace</td>
   <td>Root file system volume encryption key</td>
   <td>Journal (for now)</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"></td>
   <td style="background-color:#e5c8e6;"><code style="background-color:#e5c8e6;">systemd-pcrmachine.service 🚀</code></td>
   <td>Userspace</td>
   <td>Machine ID (<code>/etc/machine-id</code>)</td>
   <td>Journal (for now)</td>
   <td>n/a</td>
  </tr>

  <tr>
   <td style="background-color:#fff3bf;"></td>
   <td style="background-color:#e5c8e6;"><code style="background-color:#e5c8e6;">systemd-pcrfs@.service 🚀</code></td>
   <td>Userspace</td>
   <td>File system mount point, UUID, label, partition UUID label of root file system and <code>/var/</code></td>
   <td>Journal (for now)</td>
   <td>n/a</td>
  </tr>
</table>

PCR 0 changes on firmware updates; PCR 1 changes on basic hardware/CPU/RAM replacements.

PCR 4 changes on boot loader updates.
The shim project will measure the PE binary it chain loads into this PCR.
If the Linux kernel is invoked as UEFI PE binary, it is measured here, too.
[systemd-stub](https://www.freedesktop.org/software/systemd/man/systemd-stub.html)
measures system extension images read from the ESP here too
(see [systemd-sysext](https://www.freedesktop.org/software/systemd/man/systemd-sysext.html)).

PCR 5 changes when partitions are added, modified, or removed.

PCR 7 changes when UEFI SecureBoot mode is enabled/disabled, or firmware certificates (PK, KEK, db, dbx, …) are updated.
The shim project will measure most of its (non-MOK) certificates and SBAT data into this PCR.

PCR 11 and 15 as shown in the list above are used by multiple components of systemd.
These are not conflicting uses;
the involved components are properly ordered to cooperatively guarantee predictable behaviour.

## Sources
* [systemd-cryptenroll(1)](https://www.freedesktop.org/software/systemd/man/systemd-cryptenroll.html#--tpm2-pcrs=PCR)
* [TCG PC Client Specific Platform Firmware Profile Specification](https://trustedcomputinggroup.org/resource/pc-client-specific-platform-firmware-profile-specification/)
* [shim's README.tpm](https://github.com/rhboot/shim/blob/main/README.tpm)
* [Measured Boot - GNU GRUB Manual 2.06](https://www.gnu.org/software/grub/manual/grub/html_node/Measured-Boot.html)
* [Integrity Measurement Architecture (IMA)](https://sourceforge.net/p/linux-ima/wiki/Home/)
* [edk2-TrustedBootChain/4_Other_Trusted_Boot_Chains.md](https://github.com/tianocore-docs/edk2-TrustedBootChain/blob/main/4_Other_Trusted_Boot_Chains.md)
* [Trusted Platform Module - ArchWiki](https://wiki.archlinux.org/title/Trusted_Platform_Module#Accessing_PCR_registers)
