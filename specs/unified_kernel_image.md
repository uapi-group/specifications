---
title: Unified Kernel Image
category: Concepts
layout: default
version: 1.0
SPDX-License-Identifier: CC-BY-4.0
---
# Unified Kernel Image (UKI)

| Version | Changes |
|---------|---------|
| 1.0     | Initial Release |

A Unified Kernel Image (UKI) is a combination of an UEFI boot stub program,
a Linux kernel image, an optional initrd, and further resources in a single UEFI PE file.
This file can either be directly invoked by the UEFI firmware
(which is useful in particular in some cloud/Confidential Computing environments)
or through a boot loader
(which is generally useful to allow multiple kernel versions with interactive or
automatic selection of version to boot into).

Various components of the UKI are provided as PE/COFF sections of the
executable.  The stub is a small program that can be executed in UEFI
mode that forms the initial executable part of the combined image.
The stub program loads other resources from its executable, including
in particular the kernel and initrd, and transitions into the kernel.

This specification defines the format and components (mandatory and optional) of UKIs.

[systemd-stub](https://www.freedesktop.org/software/systemd/man/systemd-stub.html)
provides the reference implementation of the stub.

## UKI File Format
The file format for UKIs is PE/COFF (Portable Executable / Common Object File Format).  This is a well-known
industry-standard file format, used for example in UEFI environments, and UKIs follow the standard, so exact
details will not be repeated here.

UKIs are a PE/COFF file with various resources, listed below, stored in PE sections.
In principle this file can be created with a relatively simple `objcopy` invocation,
but the recommended way is to use a helper program
([`ukify`](https://www.freedesktop.org/software/systemd/man/ukify.html)),
which takes care of appropriate alignment and facilitates signing for SecureBoot.

UKIs are UEFI applications images, and hence should initialize the `Subsystem` field of the *optional* PE
header to 0x0A (i.e. `IMAGE_SUBSYSTEM_EFI_APPLICATION`).

## UKI Components
UKIs consist of the following resources:

<!--
NOTE: these components are in canonical order for predictable PCR measurements.
Please add any new components at the bottom of the list and NEVER reorder anything in this list.
-->
* An UEFI boot stub that forms the initial program.
  It contains various PE sections normally required for a program,
  including `.text`, `.reloc`, `.data`, and others.
* The Linux kernel in the `.linux` PE section.
* Optionally, information describing the OS this kernel is intended for, in the `.osrel` section. The contents of this section are derived from `/etc/os-release` of the target OS. They can be useful for presentation of the UKI in the boot loader menu, and ordering it against other entries using the included version information.
* Optionally, the kernel command line in the `.cmdline` section. If this is absent, the loader implementation may allow local overrides instead.
* Optionally, the initrd that the kernel shall unpack and invoke, in the `.initrd` section.
* Optionally, a microcode initrd in the `.ucode` section, to be handed to the kernel before any other initrd.
* Optionally, a splash image to bring to screen before transitioning into the Linux kernel, in the `.splash` section.
* Optionally, a compiled Devicetree, for systems which need it, in the `.dtb` section.
* Optionally, one or more compiled Devicetrees, for systems which need it, each in a separate `.dtbauto` section. The first `.dtbauto` section that matches the current hardware (matching is done either by the first `compatible` property with one from the firmware-provided Devicetree or by the SMBIOS fields using the contents of `.hwids` section as described below) will override the `.dtb` section.
* Optionally, a hardware identification table (also known as [HWID](https://github.com/fwupd/fwupd/blob/main/docs/hwids.md) or [CHID](https://learn.microsoft.com/en-us/windows-hardware/drivers/install/specifying-hardware-ids-for-a-computer)) in the `.hwids` section.
* Optionally, information describing kernel release information (i.e. `uname -r` output) in the `.uname` section. This is also useful for presentation of the UKI in the boot loader menu, and ordering it against other entries.
* Optionally, a CSV file encoding the SBAT metadata for the image, in the `.sbat` section. The [SBAT format is defined by the Shim project](https://github.com/rhboot/shim/blob/main/SBAT.md), and used for UEFI revocation purposes.
* Optionally, a JSON file encoding expected PCR 11 hash values seen from userspace once the UKI has booted up, along with signatures of these expected PCR 11 hash values, in the `.pcrsig` section. The signatures must also match the key pair described below.
* Optionally, the public part of a public-private key pair in PEM format used to sign the expected PCR 11 value of the image, in the `.pcrpkey` section.

Note that all of the sections defined above are singletons:
they may appear at most once,
except for the `.dtbauto` section which may appear multiple times.

Only the `.linux` section is required for the image to be considered a Unified *Kernel* Image.

A UKI will generally also contain various sections required for the boot stub,
but we don't document those here.

Boot menus such as [sd-boot](http://www.freedesktop.org/software/systemd/man/sd-boot.html)
and other consumers of UKIs may place additional requirements,
for example only show kernels with the `.osrel` section present.

## PE Addons

UKIs are PE executables that may be executed directly in UEFI mode, and contain a variety of resources
built-in, as described above. Sometimes it's useful to provide a minimal level of modularity and extend UKIs
dynamically with additional resources from separate files. For this purpose UKIs can be combined with one or
more "PE Addons". This are regular PE UEFI application binaries, that can be authenticated via the usual UEFI
SecureBoot logic, and may contain additional PE sections from the list above, that shall be used in
combination with any PE sections of the UKI itself. At UKI invocation time, the EFI stub contained in the UKI
may load additional of these PE Addons and apply them (after authenticating them via UEFI APIs), combining
them with the resources of the UKI.

PE Addons may *not* contain `.linux` PE sections (this may be used to distinguish them from UKIs, which must
have this section, see above).

PE Addons must contain at least one section of the following types:

* `.cmdline`
* `.dtb`
* `.dtbauto`
* `.ucode`
* `.initrd`

PE Addons should be sorted by their filename, and applied in this order. In case of `.cmdline` all command
lines provided by addons are suffixed in this order to any command line included in the UKI. In case of
`.dtb` and `.dtbauto` any such section included in the UKI shall be applied first, and those provided by add-ons should then
by applied in order as a fix-up. In case of `.ucode` the contained `cpio` archives should be prefixed to the
regular initrds passed to the kernel, in reverse order. In case of `.initrd` the contained `cpio` archives
should be appended to the regular initrds passed to the kernel.

PE Addons may include sections of multiple types (e.g. both a `.cmdline` and a `.dtb` section), in which case
all of them should be applied.

Just like UKIs PE Addons should have the `Subystem` field of the *optional* PE header set to 0x0A.

The PE header's `Machine` field should be set to the local CPU type for the target machine of the Addon. When
enumerating PE Addons to apply, candidates should be skipped when their header field reports a non-native CPU
architecture.

PE Addons may contain executable code in a `.text` section. This code may be useful to write a friendly error
message to the UEFI console when executed as regular programs. The code should be ignored when the addon is
applied on an UKI.

## UKI TPM PCR Measurements

On systems with a Trusted Platform Module (TPM)
the UEFI boot stub shall measure the sections listed above,
starting from the `.linux` section,
in the order as listed
(which should be considered the *canonical order*).
The `.pcrsig` section is not measured.

For each section two measurements shall be made into PCR 11 with the
event code `EV_IPL`:

1. The section name in ASCII (including one trailing NUL byte)
2. The (binary) section contents

The above should be repeated for every section defined above, so that
the measurements are interleaved: section name followed by section
data, followed by the next section name and its section data, and so
on.

If multiple `.dtbauto` sections are present, only the one that is actually in use shall be measured.

## JSON Format for `.pcrsig`
The format is a single JSON object, encoded as a zero-terminated `UTF-8` string. Each name in the object
shall be unique as per recommendations of
[RFC8259](https://datatracker.ietf.org/doc/html/rfc8259#section-4). Strings shall not contain any control
character, nor use `\uXXX` escaping.

When it comes to JSON numbers, this specification assumes that JSON parsers processing this information
are capable of reproducing the full signed 53bit integer range (i.e. -2⁵³+1…+2⁵³-1) as well as the full
64bit IEEE floating point number range losslessly (with the exception of NaN/-inf/+inf, since JSON cannot
encode that), as per recommendations of [RFC8259](https://datatracker.ietf.org/doc/html/rfc8259#page-8).
Fields in these JSON objects are thus permitted to encode numeric values from these ranges as JSON numbers,
and should not use numeric values not covered by these types and ranges.

The content is a JSON object, named after the TPM SHA bank to use, containing an array of measurement
objects, each containing an array of PCRs, the SHA256 fingerprint of the public key (DER) used for the
signature (`pkfp`), the expected hash (`pol`) and the signature encoded in base64 (`sig`).

Example:

```
{
    "sha1": [
        {
            "pcrs": [
                11
            ],
            "pkfp": "2870989436ec5c24461f36f5f070613043c30a156a895903e27fc985d1b2887f",
            "pol": "4a5cfbca5123490989ac060ec8b1755cfa6f0ea37ec39206e988442a9a9023bb",
            "sig": "X9a07Peo0EaEWr0dfUgZIq3Bsf20AGTjAgMilyH3TkLtPBGJLCEFRzK2jkPohG0VXQjao35765Wp/sV1wfctGC0fx9GOsBzK8YKjsFitOw21aLxlnES31D3PbDLPRqkx+fAhwV0/Akd99hNuiyzGdUewNpbbBNo7WXkd4K62RK61dKKI4g//qtLeAyXlee0TLKVxNcT46Ud1t8eUb1GAwRnO7DxBZx8uFyP/D9wpPNK7+M01to74d9ijcsjLXf2eGKcpiDvenUnhI6ua+OvT6CnmgxkFQutLGz/Ka23spSG/YJHfxGT7VpOYveDG19nqBb/fg30HZiY7lVTolS93UA=="
        }
    ],
    "sha256": [
        {
            "pcrs": [
                11
            ],
            "pkfp": "2870989436ec5c24461f36f5f070613043c30a156a895903e27fc985d1b2887f",
            "pol": "707f5d03325822b2a53bfe5d723e0ca290f397c0e6184131b70d00e35224488a",
            "sig": "moQh6GF18LiVlA8CxRkTtbXr2p0NIIBosLazDALZ9lOJQw/w1PB7tcDZ1Kumvzqtx4FO5WVjOkVTnNFrYmXn9K2PpqIDEuTtwaM/lKgP12LtcC635C+VsJMQg3k9sEFfLwBCzrhYxt5GCpxzPrsfwJtsUpueB23sNw27WJS7C+tVnqWw7br6i9vJ59jP9+HXlex+OlZHliHLzZwpuZA8iPMQT0xvm901ak5yoBqNPv4Yya19dlt2sCuO+Iw1LeZW9U83zdG0hn1mxavRIxZ7s0f7a1n/ScrOksgPQB8xfDdFDf9fssGALanOgjCHyD7hRzV31++Qpgah4uc/LJiesg=="
        }
    ]
}
```

The [`systemd-measure`](https://www.freedesktop.org/software/systemd/man/systemd-measure.html) tool can be
used to generate and sign `.pcrsig`.

## Multi-Profile UKIs

In various contexts it is useful to support multiple different configurations ("profiles") an UKI may be
booted into. An example: a single UKI that can be booted with one of three different kernel command lines,
one covering regular boot, one implementing a factory reset logic, and a third one booting into Storage
Target Mode, or similar. In order to support this, *Multi-Profile UKIs* may be defined, as an optional
extension of the regular UKI concept described above.

Multi-profile UKIs extend regular UKIs by introducing an additional PE section with the name `.profile` which
can appear multiple times in a single PE file and both acts as a separator between multiple profiles of the
same UKI, and carries meta-information about the profile it is introducing. All regular UKI PE sections
listed above may appear multiple times in multi-profile UKIs, but only once before the first `.profile` PE
section, once between each subsequent pair of `.profile` sections, and once after the last `.profile` (except
for `.dtbauto`, which is allowed to be defined multiple times anyway, see above). Each `.profile` section
introduces and defines a profile, which are numbered from zero, and typically denoted with an `@` character
before the profile number, i.e. `@0`, `@1`, `@2`, … The sections listed in the PE binary before the first
`.profile` section make up a special profile called the *base profile*.

When a multi-profile UKI is invoked, the EFI stub code will make sure to load the PE sections matching the
selected profile. A profile is (optionally) selected by prefixing the EFI stub's invocation parameters
("command line") with `@0 `, `@1 `, `@2 `, (i.e. an `@` character, the numeric profile index, and a space
character) in order to select the desired profile. The stub combines the PE sections of the selected profile
with any PE sections from the base profile that are not specified in the selected profile. Or in other words:
sections associated with specific profiles comprehensively override those of the same name in the base
profile. If a multi-profile UKI is invoked without specification of a profile selector on its command line,
profile `@0` is automatically selected as default.

The profile selector prefix of the UKI's invocation parameters is stripped after parsing, and is thus neither
passed on to the invoked kernel on the kernel's command line, nor is measured as part of the kernel command
line.

When measuring PE sections before passing control to the contained kernel, only the sections associated with
the selected profile, or the base profile are measured. All others are ignored (neither measured nor used in
any other way).

A `.profile` section may optionally contain meta-information about the profile it introduces that a boot menu
can use to automatically synthesize menu entries from the profiles a UKI defines. It contains text data,
following a similar syntax as `.osrel` sections: environment-block like key-value pairs. Currently, two
fields are defined: `ID=` may contain a brief textual, 7bit ASCII identifier for the profile. `TITLE=` may
contain a brief human readable text string that may be shown in a boot menu that allows profile selection.

A brief example for the structure of a hypothetical multi-profile UKI:

| Section        | Contents                                                    | Profile |
|----------------|-------------------------------------------------------------|---------|
| `.linux`       | ELF kernel                                                  | Base    |
| `.osrel`       | `/etc/os-release`                                           | Base    |
| `.cmdline`     | `"quiet"`                                                   | Base    |
| **`.profile`** | `ID=regular TITLE="Regular boot"`                           | `@0`    |
| **`.profile`** | `ID=factory-reset TITLE="Reset Device to Factory Defaults"` | `@1`    |
| `.cmdline`     | `"quiet systemd.unit=factory-reset.target"`                 | `@1`    |
| **`.profile`** | `ID=storagetm TITLE="Boot into Storage Target Mode"`        | `@2`    |
| `.cmdline`     | `"quiet rd.systemd.unit=storage-target-mode.target"`        | `@2`    |

(Note: in this example, the `.cmdline` shown as part of the base profile might as well be moved into profile
`@0` with identical effect. This is because every other profile overrides it anyway, and thus it only applies
to profile `@0` either way.)

While the primary usecase for multi-profile UKIs are allowing multiple kernel command line sections
(i.e. `.cmdline`) choices, the concept is not limited to that: any of the UKI PE sections may appear in
profiles, for example to allow alternative selection of multiple different CPU microcode or Devicetree blobs.

Note that if the PCR signature mechanism described above is used it is recommended to include a separate
`.pcrsig` PE section in each profile matching precisely the sections that apply to that profile (i.e. the
combination of the profile's own sections and those of the base section).

## Updatability
UKIs wrap all of the above data in a single file, hence all of the above components can be updated in one go
through single file atomic updates, which is useful given that the primary expected storage place for these
UKIs is the UEFI System Partition (ESP), which is a vFAT file system, with its limited data safety guarantees.

## Security
Given UKIs are regular UEFI PE files, they can thus be signed as one for Secure Boot, protecting all of the
individual resources listed above at once, and their combination. Standard Linux tools such as
[`sbsigntool`](https://manpages.debian.org/unstable/sbsigntool/sbsign.1.en.html) and
[`pesign`](https://github.com/rhboot/pesign) can be used to sign UKI files. The signature format and process
again match the ones already used for PE files, so they will not be redefined here.

## Locations for Distribution-built UKIs Installed by Package Managers

UKIs that are built centrally by distributions and installed via the package manager should be installed in
`/usr/lib/modules/$UNAME/`, where `$UNAME` is the output of `uname -r` of the kernel included in the UKI, so
that tools staging or consuming UKIs have a common place to store and look for them.

The installed UKIs should have a filename `<version format specification>.efi`, i.e. the filename is left to
implementers but must be valid for comparisons according to the [Version Format Specification](version_format_specification.md).

## Locations and Naming for UKI Auxiliary Resources

Auxiliary UKI resources (such as PE addons for kernel command line extensions and similar, as well as
systemd-sysext and systemd-confext DDIs) built centrally by distributions and installed via package manager
should be installed into locations depending on whether they should be applied to all UKIs installed in the
ESP, or only to a single specific UKI.

UKI auxiliary resources that apply to *all* installed UKIs should be
installed into `/usr/lib/modules/uki.extra.d/`. UKI auxiliary resources that
apply to *one* specific installed UKI should be instead installed into
`/usr/lib/modules/$UNAME/$UKI.efi.extra.d/`, where `$UNAME` is the output of
`uname -r` of the kernel included in the UKI and `$UKI` is the name of the
corresponding centrally built UKI with the `.efi` extension stripped.

The installed UKI auxiliary resources must have a specific file extension, which
depends on the resource type:
* `.addon.efi` for PE addons,
* `.sysext.raw` for sysext DDIs,
* `.confext.raw` for confext DDIs

### Example

Given a UKI `bar_123.efi` that includes a kernel `6.9.1-1.foo`, consider
* a PE addon `machine-id` that should apply to all installed UKIs,
* a PE addon `proprietary-driver_2000` that is specific to the `bar_123` UKI, and
* a sysext `mysysext_1.23.47^3` that should apply to all installed UKIs.

The resulting paths would be
* `/usr/lib/modules/uki.extra.d/machine-id.addon.efi`,
* `/usr/lib/modules/6.9.1-1.foo/bar_123.efi.extra.d/proprietary-driver_2000.addon.efi`, and
* `/usr/lib/modules/uki.extra.d/mysysext_1.23.47^3.sysext.raw`.
