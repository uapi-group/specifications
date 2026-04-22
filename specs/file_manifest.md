---
title: UAPI.16 File Manifest
category: Concepts
layout: default
SPDX-License-Identifier: CC-BY-4.0
version: 1.0
weight: 16
aliases:
- /UAPI.16
- /16
---

# UAPI.16 File Manifest

| Version | Changes         |
|---------|-----------------|
| 1.0     | Initial Release |

This format stores information about static, immutable data resources that potentially can be acquired over
the network or stored on a disk. It's in particular supposed to be able to reasonably represent the
properties of files arranged in a POSIX file system tree, as well as partitions in a GPT partition table.

This manifest file format is inspired by the traditional UNIX `SHA256SUMS` and BSD `mtree(5)` file formats,
but has various additional features, among them:

1. It allows referencing remote URLs as source for acquiring file contents.
2. It allows extracting data "slices" from data sources, via offset and range.
3. Various fields of additional per-file metadata may be defined, including various fields for GPT partition table metadata.
4. It's an extensible format permitting vendors and projects to add their own per-file and per-manifest fields.
5. Inline cryptographic signing.

This format is designed with tools such as `mkosi` and `systemd-sysupdate` in mind, but it's intended to be
generally useful as a way to describe collections of files that may potentially be acquired over the network
or from local storage, and verified cryptographically.

## Manifest File Name and Media Type

When stored in a file system directory – alongside the data files it references – the manifest file
should be named `Uapi16Manifest`.

When served via an HTTP server it's recommended to use the media type `application/vnd.uapi.16.manifest`.

## General Structure

The file is organized as an [RFC7464 JSON-SEQ](https://datatracker.ietf.org/doc/html/rfc7464) sequence of
file objects which are described below.

The first object in the sequence always describes the enveloping directory. This *root* file object must
*not* have a `name` field set (which identifies it as the *root* file object), however, it must have a
`mediaType` field set (which identifies the sequence as a UAPI.16 manifest). File objects for directory
entries `Uapi16Manifest` (or any filename with a prefix of `Uapi16Manifest.`), `.`, `..` or any path ending
in `/.`, `/..` are not permitted.

When applied to a GPT partition table the first object may also describe fields in the overall GPT
partition table header of the disk.

Within each directory files may appear in any order. However, files nested into subdirectories must be listed
immediately following the file object of the subdirectory object. (Even if ordering within directories is
not mandated, it is definitely a good idea to sort the entries alphabetically, or by version
sort. Alternatively, historical ordering – i.e. add new additions purely at the end – might be a good
idea. Sorting entries ensures the manifests become reproducible as long as the same tool is used, which is
definitely a welcome property.)

The `name` fields of file objects must be uniquely assigned within a manifest file.

If the manifest encodes a GPT partition table, the `name` field must still be assigned for every file object, even
if `gptLabel` is defined too. The former must still be unique within the manifest; the latter does not have to be.

For manifests covering a directory tree with nested directories (as opposed to a flat directory listing
without file objects of type `dir`), subdirectory listings may either be specified inline (typically
preferable) or be imported via a separate "sub-manifest" file covering just this subdirectory (and potential
sub-subdirectories, …), by referencing the separate UAPI.16 manifest file in the `contents` field of the file
object.

Sizes, offsets, timestamps shall use non-negative integer JSON numbers, and may use values above 2⁵³. (This means
– strictly speaking – the format cannot be processed losslessly by JSON parsers that exclusively use 64bit
floating point for JSON numbers, if there are any files of such excessive sizes. Since it's unlikely that
files included in a UAPI.16 manifest file are this large this should not be a practical limitation. A JSON
implementation supporting unsigned 64bit numbers is still recommended.)

## File Object

This is an object that encodes information about an individual file, or about the root directory of the manifest.

```json
{
        "name" : "…",
        "mediaType" : "application/vnd.uapi.16.manifest",
        "type" : "reg",
        "size" : 33,
        "major" : 4711,
        "minor" : 815,
        "mode" : 123,
        "uid" : 1000,
        "userName" : "lennart",
        "gid" : 1000,
        "groupName" : "lennart",
        "mTime" : 17776151691231231233,
        "inodeToken" : 4711,
        "gptLabel" : "…",
        "gptUuid" : "…",
        "gptTypeUuid" : "…",
        "gptDiskUuid" : "…",
        "gptFlagNoAuto" : true,
        "gptFlagGrowFileSystem" : true,
        "readOnly" : true,
        "contents" : [{…}, … ],
        "sha256" : "…",
        "validAfterUSec": 4711,
        "validBeforeUSec" : 4711,
        "validity" : "revoked"
}
```

### Fields

All fields are optional. If a field shall not be set it can either be set to `null` or simply
omitted in the JSON object, both ways shall be considered equivalent.

If not otherwise indicated all fields are supposed to be of type string.

The `mediaType` field is mandatory on the first file object in the sequence (i.e. the top-level root
directory object), and must have the value `"application/vnd.uapi.16.manifest"`. It should not be used on
any other file object in the sequence.

If `name` is not specified the record stores information about the top-level root file object. This file
object always comes first in the file object sequence. Or in other words, a file object has exactly one of
`mediaType` (if root file object/first object in sequence) and `name` (if not root file object/not first
object in sequence) set.

The `name` must be a valid UTF-8 POSIX path (i.e. using the slash `/` as component separator), may not
contain control characters (ASCII 0…31, 127) and may not contain "`.`", "`..`", "``" as components of the
path. It may not start nor end with a slash, nor may it have multiple slashes in immediate sequence (or in other
words, the path must be normalized and relative). The name may not be identical to "`Uapi16Manifest`" or be
prefixed with "`Uapi16Manifest.`" (it is however acceptable if these filenames appear as components of paths
further down the tree). Files whose paths do not follow these rules cannot be encoded in manifest files
defined by this specification.

The `type` field encodes the inode type of the file object. It takes one of `reg`, `dir`, `lnk`, `fifo`,
`chr`, `blk`, `sock` (these values follow low-level UNIX/Linux naming of these inode types). If not specified
defaults to `dir` for the first object in the sequence and `reg` for all others. Note that when using the
manifest format to list partition contents each partition should be listed as a file of type `reg`.

The `size` (unsigned integer) field only applies to file objects of type `reg`, `lnk`, `dir` and encodes the
size of the contents of the file, in bytes. Specifically, for regular files this is the size as reported by
the file system. In case of symlinks: this is the length of the symlink target string. In case of `dir` this
is the size of a UAPI.16 file manifest generated for the subdirectory (which only applies if the directory
contents are not provided in-line, but via a reference to a separate sub-manifest file, see above).

The `major` (unsigned integer) field only applies to file objects of type `blk` and `chr`, it encodes the
major device number of the device node. Similarly, `minor` (same type and condition) encodes the minor device
number.

The `mode` (unsigned integer) field encodes the UNIX access mode of the file object. It applies to all
inode types, except `lnk`. Note that while UNIX access modes are typically written in octal, this one is
encoded in a regular JSON number, i.e. decimal. The valid range is 0…4095 (i.e. `0o0000` to `0o7777`).

The `uid` (unsigned integer) is the numeric owning UNIX user ID of the file. Similarly, `gid` (same type) is the
numeric owning UNIX group ID of the file.

The `userName` field is the owning UNIX user name of the file; similarly, `groupName` is the owning UNIX group
name of the file. Both fields must contain a valid UNIX user name. (Note that UNIX user/group name validity
is not very well defined, hence it's recommended to liberally accept strings here, maybe with the exception
of empty strings, fully numeric strings, and those containing control characters).

The `mTime` field (unsigned integer) is the UNIX file modification timestamp of the file object, in
nanoseconds since the UNIX epoch (Jan 1st, 1970).

The `inodeToken` field (unsigned integer) shall be a unique numeric identifier for any file that shall be
hardlinked within the manifest tree. If unspecified or 0 the file is not subject to hardlinks, otherwise any
pair of file objects that shall be hardlinks to the same inode shall carry the same `inodeToken` value. In
order to ensure reproducibility `inodeToken` shall be assigned as a counter starting at 1 for the first file
that shall be hardlinked, and increasing by 1 for each subsequent file that shall be hardlinked but has not
been hardlinked in the manifest so far. When sub-manifests are used, hardlinking between the upper and lower
manifest is not supported – the `inodeToken` identifiers are specific to the manifest they are specified in.

The `gpt*` fields encode fields that are needed when placing these resources in a GPT partition table
entry. They only apply to file objects of inode type `reg`, with the exception of `gptDiskUuid` which only
applies to the root file object (which has an inode type of `dir`).

The `gptFlag*` fields are of boolean type. They correspond to the flags of the same name from the GPT/UEFI
specification.

The `gpt*Uuid` fields take a UUID in traditional string formatting (case insensitive). `gptUuid` stores the
partition UUID for the entry, and `gptTypeUuid` the partition type UUID.

The `gptLabel` field may be no longer than 36 UTF-16 code units (72 bytes), as defined by the UEFI
specification. If `gptLabel` is unspecified, and the data is
written to a GPT partition it should use the value of `name` as label instead. Note that this might require
mangling, as GPT partition labels have smaller size limits than file names. Also note that unlike the `name`
this field does not have to be uniquely assigned among all entries in the same manifest.

The `gptDiskUuid` field takes a UUID in traditional string formatting (case insensitive). It encodes the disk
UUID for the GPT partition table. In contrast to the other `gpt*` fields this one may only be set on the root
directory object (i.e. the first file object in the sequence), as it reflects a field of the GPT partition
table header rather than the GPT partition entry header.

`readOnly` (a boolean) is relevant for initializing the read-only bit of GPT partition table entries. It
should also be reflected in the POSIX 'w' access mode bit when storing the data in a regular file on disk. If
unspecified, defaults to false. If both `mode` and `readOnly` are set for the same file object, they
must be consistent: if any of the three 'w' bits in `mode` is set, `readOnly` must be off, otherwise
on.

`contents` is an array of contents objects, as described below. This field only applies to file objects of
type `reg`, `lnk`, `dir`. In case of regular files this references or contains the file contents. In case
of symlinks this references or contains the symlink target string. In case of directories this references or
contains the sub-manifest for the directory (if applicable, see above). If this field is not set and the file
object is of type `reg`, an array with a single contents object shall be implied, that uses `file` to
reference a file of the name specified in `name` (without encoding or offset). Or in other words: if no
`contents` is specified in a file object of an `Uapi16Manifest` file, then it should be looked for in a file
of the name `name` in the same directory.

`sha256` is the SHA256 hash of the file contents, formatted in 64 hexadecimal characters. Parsers should
parse this case-insensitively. This field only applies to file objects of type `reg`, `lnk`, `dir`. The hash
value covers the contents referenced or contained in the `contents` field described above. Additional hash
algorithms may be defined in future, in which case separate fields will be defined for each.

`valid*USec` is in µs since UNIX epoch UTC, and shall be an unsigned integer. It defines a validity time
window of the resource. The file object should not be accessed or consumed outside of the specified time
window. If `validAfterUSec` is not specified it should default to 0, if `validBeforeUSec` is not specified it
never expires. If a directory file object is invalidated via these two settings, then all file objects within
it are invalidated implicitly too. This specifically means that the time window specified on the root
directory object (i.e. the first one in the sequence) may invalidate the whole manifest file.

`validity` takes a string, and implements an enumeration: one of `good`, `revoked`, `stepping-stone`. If
unspecified, defaults to `good`. If set to `revoked` the file should not be considered anymore for automatic
consumption by the client, and if a client already acquired it before it should consider removing it again or
otherwise ensure it's not used anymore. If set to `stepping-stone` and the manifest file is processed by a
software update tool, and lists multiple versions of the same resource in individual files, then the update
tool should ensure that files where this is set are never skipped for version updates, and installed and
activated before considering any later versions of the same resource.

## Out of Scope

Note that the above does not support – on purpose – the following common UNIX file or GPT partition table
properties:

1. Access timestamps, birth timestamps, change timestamps
2. Inode numbers and other file IDs (such as `name_to_handle_at()` style FID)
3. Backing device node major/minor
4. Hardlink link counts

As all of these quite likely will differ if the same manifest is applied to different file systems.

## Contents Object

```json
{
        "file" : "…",
        "url" : "https://…",
        "literal" : "TmV2ZXIgZ29ubmEgZ2l2ZSB5b3UgdXAsIG5ldmVyIGdvbm5hIGxldCB5b3UgZG93bg==",
        "encoding" : "gzip",
        "encodedSize" : 2011,
        "originalSize" : 4711,
        "offset" : 22
}
```

Contents objects indicate where the data of a `reg`, `lnk`, `dir` type file object may be found. The
`contents` field of the file objects (see above) contain an array of objects of this type.

### Fields

At most one of `file`, `url`, `literal` must be set. `file` references a file in the same location as the
manifest file itself was found, `url` references a full URL, and `literal` contains in-line Base64
data. `url` may use `http://…` and `https://…` schemes only. It is recommended (but not required) to use
`literal` to encode the target of symbolic link file objects. If none of these fields are set, behaviour
equivalent to `file` being set to the file object's `name` field should be implied.

`literal` may be encoded either in regular Base64 or in URL-safe Base64. Consumers must be able to deal
with either decoding vocabulary.

`encoding` specifies the encoding of the specified resource. It accepts the same encoding specifiers as
HTTP's `Content-Encoding:` field, i.e. `gzip` or `zstd`.

`originalSize` is the size of the full, decoded data, `encodedSize` of the full, encoded data. If
`encoding` is not specified, `encodedSize` should not be specified. Both are unsigned integer type.

If `offset` is specified, then the contents of the file should be acquired from the specified offset in the
*decoded* data. If the field is not specified it defaults to 0. The data to extract begins at the specified
offset, and extends over the size specified in the `size` field of the file object. Note that this means that
the sum of `offset` and `size` must be less than or equal to `originalSize`, so that the slice does not extend
beyond the source data.

## Signature Objects

Optionally, manifest files may be suffixed with a signature object. This object should appear in the JSON-SEQ
sequence immediately after the last file object. This object should follow the following general structure:

```json
{
        "mediaType" : "application/vnd.uapi.16.signature",
        "signatures" : [{…}, …]
}
```

### Fields

The `mediaType` should be used to recognize the signature object, and distinguish it from a file object.

The `signatures` field should be set to an array of signature data objects (or in other words: multiple
signatures with distinct keys or mechanisms are supported within the same manifest file).

```json
{
        "mechanism" : "openpgp",
        "key" : "…",
        "data" : "…"
}
```

The `mechanism` field should be set to a string identifying the signature mechanism. Currently, only
`openpgp` is defined, for OpenPGP compatible signatures. This is the only required field, the other fields'
existence depend on the used mechanism.

The `key` field may point to a key identifier, as appropriate for the selected mechanism. For the `openpgp`
mechanism this is the key fingerprint in hexadecimal characters.

The `data` field contains the signature data in Base64 encoding. For the `openpgp` mechanism this is the
binary signature (i.e. without ASCII armor).

The cryptographic signature should cover the exact binary data of the JSON-SEQ sequence up to the ASCII RS
(0x1e) byte immediately preceding the signature object.

If multiple signatures are provided, a tool processing manifest files may freely pick *one* of them to
authenticate the manifest file, and ignore all others. Which one it picks is up to the tool, but should
typically be dependent on locally supported mechanisms and recognized keys.

## Compression

Optionally, manifest files may be compressed with mechanisms such as `gzip`, `zstd` or similar. In this case
the suggested filename shall be `Uapi16Manifest.gz`, `Uapi16Manifest.zst` and so on, as appropriate for the
chosen algorithm.

Such compressed manifest files need to be decompressed before validating the embedded signatures (as
described above).

## Relationship to SHA256SUMS

A UAPI.16 manifest file that only contains the top-level `dir` file object in combination with any number of
`reg` file objects – of which each only use the fields `name` and `sha256` – can be mapped 1:1 to a SHA256SUMS file
and back.

## Acquiring a File Remotely

When acquiring a file listed in a UAPI.16 File Manifest from a web service, the following logic should be
implemented.

1. If `validity` is set to `revoked`, the download should immediately fail.
2. If `validAfterUSec` is set and greater than the current time, the download should immediately fail.
3. If `validBeforeUSec` is set and lower than the current time, the download should immediately fail.
4. The `contents` array should be iterated, and a suitable entry be picked as source for the file contents. If no `contents` array is specified, proceed as in step 7, with an implied `file` field set to the `name` of the file object.
5. If the selected contents object has `literal` set, it should be Base64 decoded, continue in step 8.
6. Otherwise, if `url` is set, the data from the URL should be acquired, continue in step 8.
7. Otherwise, if `file` is set, the data should be acquired from the same URL as the manifest itself, however with the last component of the URL replaced by the file name encoded in `file`, following the same semantics as HTML relative links. Continue in step 8.
8. If both `encodedSize` and `encoding` are set the size of the acquired data shall be checked against `encodedSize`.
9. If `encodedSize` is not set, but `originalSize` is: the size of the acquired data shall be checked against `originalSize`.
10. If `encoding` is set: the acquired data shall be decoded according to the algorithm indicated in `encoding`.
11. If `encoding` and `originalSize` are set: the resulting decoded data shall be checked against `originalSize`.
12. The byte range indicated by `offset` (if not set: 0) and `size` (if not set, the remainder of the file) shall be extracted from the decoded data.
13. The hash of the extracted data shall be matched against the hash encoded in `sha256` (if set).

If all these steps succeed the extracted data from step 12 is the result of the operation.

Of course, many of the steps described above should typically be done together rather than serially for
robustness, efficiency and security reasons. For example, if `encoding` is not used, it is recommended to
include the `offset` field (of the contents object) and the `size` field (of the file object) in HTTP range
request fields already (in order to avoid downloading redundant data). Moreover, downloads should fail
immediately once the encoded or decoded data goes beyond the indicated encoded or original data sizes. Then,
the decoding should be done on-the-fly while the data is downloaded, and the checksum should be calculated
on-the-fly too.

## Extensibility

Additional fields can be defined freely by implementors. Each such extension field should be named in the
style of `x<Vendor>Foobar` to minimize risk of conflicts. For example `xAmutableFrobnicate` or
`xMyAwesomeProjectEfficiencyVector`.

Additional objects may be inserted at any place in the JSON-SEQ sequence, with the exception of before the
first object. They must be marked via a `mediaType` field, that describes what the object contains. When
processing a manifest file a reader should ignore (but probably log) objects in the JSON-SEQ stream it
does not recognize. Note that any objects inserted before a signature object will be protected by the
signature, but any objects inserted after will not be. A tool processing manifest files must take this into
account and potentially ignore any objects appended to the JSON stream as unauthenticated data.

## Future Revisions

This specification is intended to be incrementally improved. Additional fields may be defined at any time,
with additional, optional metadata. Should a breaking change be necessary the media type will be changed.

A number of future extensions are envisioned:

* Fields for file extended attributes, POSIX ACLs

* Fields for extended file flags (Linux `chattr`, DOS file system flags, …)

* Additional hashing schemes

* Additional signature schemes

## Example

`<ASCII-RS>`
```json
{
    "mediaType" : "application/vnd.uapi.16.manifest"
}
```
`<ASCII-RS>`
```json
{
    "name" : "FooOS.raw",
    "contents" : [{
        "file" : "FooOS.raw.gz",
        "encoding": "gzip",
        "encodedSize": 5642649603
    }],
    "size" : 7523532800,
    "sha256" : "922a9bae0e02b4ffac3e5ed5054230d0689b9c2e25b0178ba82b925f2a0c3e48",
    "validBeforeUSec" : 1776856773123234
}
```
`<ASCII-RS>`
```json
{
    "name" : "FooOS-esp.raw",
    "contents" : [{
        "file" : "FooOS.raw.gz",
        "encoding" : "gzip",
        "encodedSize" : 5642649603,
        "originalSize" : 7523532800,
        "offset" : 2097152
    }],
    "size" : 149175808,
    "sha256" : "5dcfd837a4868550cc61c256d9567a974e32a20985afa9e100b8b96755a20cae",
    "gptLabel" : "EFI System Partition",
    "gptUuid" : "8a7f0bce-5fd7-4d23-9c12-3b6a4e1d9f07",
    "gptTypeUuid" : "c12a7328-f81f-11d2-ba4b-00a0c93ec93b",
    "validBeforeUSec" : 1776856773123234
}
```
`<ASCII-RS>`
```json
{
    "name" : "FooOS-root.raw",
    "contents" : [{
        "file" : "FooOS.raw.gz",
        "encoding": "gzip",
        "encodedSize" : 5642649603,
        "originalSize" : 7523532800,
        "offset" : 351272960
    }],
    "size" : 234003200,
    "sha256" : "00b70a0813e15f309828d3a36156283cba87576c26755e0b2d4cf0951eff8163",
    "gptLabel" : "FooOS-root",
    "gptUuid" : "1d2e3f40-51a6-4b78-9c0d-2e4f6a8b1c3d",
    "gptTypeUuid": "4f68bce3-e8cd-4db1-96e7-fbcaf984b709",
    "readOnly": true,
    "validBeforeUSec" : 1776856773123234
}
```
`<ASCII-RS>`
```json
{
        "mediaType" : "application/vnd.uapi.16.signature",
        "signatures" : [{
            "mechanism" : "openpgp",
            "key" : "38110B0FAC93C324",
            "data" : "iQIzBAABCgAdFiEEFrHE7sC8AhrHd/aBtjshh5w0hbAFAmn8rUoACgkQtjshh5w0hbAz3g//eyWO1swKXivio1PH2oJclEAYqpyqkv9LC293ePyDp5K0zL7BEVt9ijCGb22Wcu+xBXug4SBpSsCMOcro633Kn0ACBeOHwsyu1rnumb85/xvx3lSiq7ke1HcP5OWh5XX4lTm3KYVaPLQy0P3ThupJJsCa0o7uDdl2fXEgcZtRGhgp7GnpQabEuUUGW+1YZ1DFhhIdewZ6+eZZac0DPFjevBbI14jf9wEp3uWT9YoULF0wWBUi45vxdievwIRbcQHQVhxKMaTm0yuTbT5kSUAZccsOkJ4gqrVoNmFx0kYPve0uWcEkFQ16zEKVESr/fhDuCW19v3D46JNvwxZgIxP3o5rp3wbOja6n5D8Otl0kUQaE7FPOmYI1SZ2F97cYsKNViuUyrMcPTNGkI4E18H4WgWLf1CEyGHT16KmoCmldqVhQiTFCtOzkJdbwJgnSL/f0n2y3H/iB3lL0hglv2SDDVnVeMBx+qs+j2DMys3SVOOAYqGrMay1PYxLTEct+M1D0aAmH+XObuoVcm3XMtPPt6NCzxes0cdmFCdMTGERYLY522YCruJpO7p4h9IEoKqDACot0vDD0qwnjBCPu7ETrb02VvJ/W3GJ7/3Y8lIvZFtJqs5TQuxU0d4OiDHe08XdwZrcAondrX5h3Up7mpQIIF4RAMqUaYdOWEf7WzFZBKvY="
        }]
}
```
The above provides three separate files `FooOS.raw`, `FooOS-esp.raw`, `FooOS-root.raw`. All files are backed by the same data file. The latter two are
defined as slices of the former, each encapsulating an individual partition. The data file is encoded via
`gzip`.
