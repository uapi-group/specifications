---
title: Unified Kernel Image
category: Concepts
layout: default
version: 1
SPDX-License-Identifier: CC-BY-4.0
---
# Unified Kernel Image (UKI)

A Unified Kernel Image (UKI) is a combination of an UEFI boot stub program,
a Linux kernel image, an initrd, and further resources in a single UEFI PE file.
This file can either be directly invoked by the UEFI firmware
(which is useful in particular in some cloud/Confidential Computing environments)
or through a boot loader
(which is generally useful to allow multiple kernel versions with interactive or
automatic selection of version to boot into).

Various components of the UKI are provided as PE/COFF sections of the executable.
The stub is a small program that can be executed in UEFI mode
that forms the main executable part of the combined image.
The stub program loads other resources from its file,
including in particular the kernel and initrd,
and transitions into the kernel.

This specification defines the format and components (mandatory and optional) of UKIs.

[systemd-stub](https://www.freedesktop.org/software/systemd/man/systemd-stub.html)
provides the reference implementation of the stub.

## File Format
The file format is PE/COFF (Portable Executable / Common Object File Format). This is a well-known
industry-standard file format, used for example in UEFI environments,
and UKIs follow the standard, so exact details will not be repeated here.

UKIs can be generated via a [single, relatively simple `objcopy`
invocation](https://www.freedesktop.org/software/systemd/man/systemd-stub.html#Assembling%20Kernel%20Images),
that glues the listed components together, generating one PE binary that then can be signed for SecureBoot.

## UKI Components
UKIs consist of the following resources:

* An UEFI boot stub that forms the initial program.
  It contains various sections normally required for a program,
  including `.text`, `.reloc`, `.data`, and others.
* The Linux kernel in the `.linux` section.
* The initrd that the kernel shall unpack and invoke, in the `.initrd` section.
* Optionally, the kernel command line in the `.cmdline` section.
* Optionally, information describing the OS this kernel is intended for, in the `.osrel` section. The contents of this section are derived from `/etc/os-release` of the target OS. They can be useful for presentation of the UKI in the boot loader menu, and ordering it against other entries using the included version information.
* Optionally, information describing kernel release information (i.e. `uname -r` output) in the `.uname` section. This is also useful for presentation of the UKI in the boot loader menu, and ordering it against other entries.
* Optionally, a splash image to bring to screen before transitioning into the Linux kernel, in the `.splash` section.
* Optionally, a compiled Devicetree database file, for systems which need it, in the `.dtb` section.
* Optionally, the public part of a public-private key pair in PEM format used to sign the image, in the `.pcrpkey` section.
* Optionally, a signature made using the abovementioned key pair, in the `.pcrsig` section.
* Optionally, a JSON file encoding expected PCR 11 hash values seen from userspace once the UKI has booted up, along with signatures of these expected PCR 11 hash values, in the `.pcrsig` section. The signatures must also match the abovementioned key pair.

### JSON Format for `.pcrsig`
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

## Updatability
UKIs wrap all of the above data in a single file, hence all of the above components can be updated in one go
through single file atomic updates, which is useful given that the primary expected storage place for these
UKIs is the UEFI System Partition (ESP), which is a vFAT file system, with its limited data safety guarantees.

## Security
Given UKIs are regular UEFI PE files, they can thus be signed as one for SecureBoot, protecting all of the
individual resources listed above at once, and their combination. Standard Linux tools such as
[`sbsigntool`](https://manpages.debian.org/unstable/sbsigntool/sbsign.1.en.html) and
[`pesign`](https://github.com/rhboot/pesign) can be used to sign UKI files. The signature format and process
again match the ones already used for PE files, so they will not be redefined here.

## Distribution-built UKIs Installed by Package Managers

UKIs that are built centrally by distributions and installed via the package manager should be installed in
`/usr/lib/modules/$UNAME/`, where `$UNAME` is the output of `uname -r` of the kernel included in the UKI, so
that tools staging or consuming UKIs have a common place to store and look for them.

The installed UKIs should have a filename `<version format specification>.efi`, i.e. the filename is left to
implementers but must be valid for comparisons according to the [Version Format Specification](version_format_specification.md).
