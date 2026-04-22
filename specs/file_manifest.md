---
title: UAPI.16 File Manifest
category: Concepts
layout: default
SPDX-License-Identifier: CC-BY-4.0
version: 0.1
weight: 16
aliases:
- /UAPI.16
- /16
---

# UAPI.16 File Manifest

| Version | Changes         |
|---------|-----------------|
| 0.1     | Initial Release |

This document describes a textual file manifest format that describes
a collection of resources (or "blobs"). These resources may
potentially be acquired over the network or stored on a disk. In
particular, it can represent the properties of files arranged in a
[POSIX](https://pubs.opengroup.org/onlinepubs/9799919799/) file system
tree as well as partitions in a
[GPT](https://uefi.org/specs/UEFI/2.11/05_GUID_Partition_Table_Format.html)
partition table.

This manifest file format is inspired by the traditional UNIX
[`SHA256SUMS`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
and BSD
[`mtree(5)`](https://man.freebsd.org/cgi/man.cgi?query=mtree&sektion=5)
file formats, but has various
additional features, among them:

1. It allows referencing remote
   [URLs](https://www.rfc-editor.org/rfc/rfc3986) as source for acquiring file
   contents.

2. It allows extracting data "slices" from data sources, via offset
   and range.

3. Various fields of additional per-file metadata may be defined,
   including various fields for GPT partition table metadata.

4. It's an extensible format permitting vendors and projects to add
   their own per-file and per-manifest fields.

5. It supports inline cryptographic signatures.

This format is designed with tools such as
[`mkosi`](https://github.com/systemd/mkosi) and
[`systemd-sysupdate`](https://www.freedesktop.org/software/systemd/man/latest/systemd-sysupdate.html)
in mind, but it's intended to be generally useful
as a way to describe collections of files that may potentially be
acquired over the network or from local storage, and verified
cryptographically.

## Terminology

* A *file object* (or just *object* for short) describes a file in the file
  system hierarchy or a partition in a partition table: a continuous,
  sized bytestream, along with various metadata fields.

* A *manifest* is a sequence of file, sinature and trailer objects.

* A *root file object* (or *root directory object*) is the top level
  file object of which all others are direct or indirect children, in
  a tree structure.

* A *signature object* carries cryptographic signature metadata that
  protects all file objects listed before it.

* A *trailer object* is the last object in the manifest and indicates
  termination.

The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”,
“SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and
“OPTIONAL” in this document are to be interpreted as described in
[BCP 14](https://www.rfc-editor.org/info/bcp14) [RFC
2119](https://www.rfc-editor.org/rfc/rfc2119) [RFC
8174](https://www.rfc-editor.org/rfc/rfc8174) when, and only when,
they appear in all capitals, as shown here.

## Manifest File Name and Media Type

When stored in a file system directory – alongside the data files it
references – the manifest file SHOULD be named `Uapi16Manifest`.

This file format is identified by the [media
type](https://www.rfc-editor.org/rfc/rfc6838) `application/vnd.uapi.16.manifest`.

## General Structure

The file is organized as an [RFC 7464
JSON-SEQ](https://www.rfc-editor.org/rfc/rfc7464) sequence of
file, signature and trailer objects which are described below.

The first object in the sequence always describes the enveloping
directory (or a GPT partition table header). This *root* file object
MUST NOT have a `name` field set (which identifies it as the *root*
file object), however, it MUST have a `mediaType` field set to
`application/vnd.uapi.16.manifest` (which identifies the sequence as a
UAPI.16 manifest).

File objects for directory entries `Uapi16Manifest` (or any filename
with a prefix of `Uapi16Manifest.`), `.`, `..` or any path ending in
`/.`, `/..` MUST NOT exist in the sequence.

When applied to a GPT partition table the first object MAY also
describe fields in the overall GPT partition table header of the disk.

Within each directory files MAY appear in any order. However, files
nested into subdirectories MUST be listed immediately following the
file object of the subdirectory object. Even if ordering within
directories is not mandated, it is definitely RECOMMENDED to sort
the entries by one of the following orderings:

1. Alphabetically

2. Via [UAPI.10 Version Format
   Specification](version_format_specification.md) version ordering

3. Time-based ordering (i.e. appending new entries strictly at the end)

Sorting entries ensures the manifests become reproducible as long as
the same tool and sorting algorithm is used.

The `name` field of file objects MUST be uniquely assigned within a
manifest file.

If the manifest encodes a GPT partition table, the `name` field MUST
still be assigned for every file object, even if `gptLabel` is defined
too. The former MUST still be unique within the manifest; the latter
does not have to be.

For manifests covering a directory tree with nested directories – as
opposed to a flat directory listing without file objects of type
`dir` –, subdirectory listings MAY either be specified inline
(typically preferable) or MAY be imported via a separate
"sub-manifest" file covering just this subdirectory (and potential
sub-subdirectories, …), by referencing the separate UAPI.16 manifest
file in the `contents` field of the file object.

Zero, one or more signature objects MAY be placed in the manifest
after all file objects. They provide cryptographic protection for all
file objects coming before. Signature objects are recognized by their
`mediaType` field set to `application/vnd.uapi.16.signature`.

Finally, a single trailer object MUST be placed in the manifest, as
last object in the sequence. Trailer objects are recognized by their
`mediaType` field set to `application/vnd.uapi.16.trailer`. Each
sequence has exactly one trailer object. Any data coming after the
trailer objects is invalid, and MUST result in a parse failure.

```
┌─────────────────────────────────────────────────────────┐
│ Root file object    (application/vnd.uapi.16.manifest)  │
├─────────────────────────────────────────────────────────┤
│ File object #1                                          │
│ File object #2                                          │
│ File object #3                                          │
│ File object …                                           │
├─────────────────────────────────────────────────────────┤
│ Signature object #1 (application/vnd.uapi.16.signature) │
│ Signature object #2 (application/vnd.uapi.16.signature) │
│ Signature object #3 (application/vnd.uapi.16.signature) │
│ Signature object …                                      │
├─────────────────────────────────────────────────────────┤
│ Trailer object      (application/vnd.uapi.16.trailer)   │
└─────────────────────────────────────────────────────────┘
```

A manifest MUST consist of at least the root file object and the
trailer object, the file and signature objects are optional.

Sizes, offsets and timestamps shall use non-negative integer
[JSON](https://www.rfc-editor.org/rfc/rfc8259) numbers, and MAY use
values above 2⁵³. (This means – strictly speaking – the format cannot
be processed losslessly by JSON parsers that exclusively use 64-bit
floating point for JSON numbers, if there are any files of such
excessive sizes. Since it's unlikely that files included in a UAPI.16
manifest file are this large this should not be a practical
limitation. A JSON implementation supporting unsigned 64-bit numbers
is still RECOMMENDED.)

## File Object

This is an object that encodes information about an individual file,
or about the root directory of the manifest.

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
        "mTimeNSec" : 1777615169123123123,
        "inodeToken" : 4711,
        "gptLabel" : "…",
        "gptUuid" : "…",
        "gptTypeUuid" : "…",
        "gptDiskUuid" : "…",
        "gptFlags" : 4711,
        "gptFlagNoAuto" : true,
        "gptFlagGrowFileSystem" : true,
        "readOnly" : true,
        "contents" : [{…}, …],
        "sha256" : "…",
        "validSinceUSec" : 4711,
        "validUntilUSec" : 4711,
        "validity" : "revoked"
}
```

### Fields

All fields are generally optional (but might be REQUIRED for some
objects under certain conditions, see below). If a field shall not be
set it can either be set to `null` or simply be omitted in the JSON
object (RECOMMENDED), both ways MUST be considered equivalent.

Unnecessary whitespace/line breaks SHOULD be omitted when formatting
the JSON records. Fields within JSON objects SHOULD be sorted
alphabetically by their field names.

If not otherwise indicated, fields are of type string.

The `mediaType` field is REQUIRED on the first file object in the
sequence (i.e. the top-level root directory object), and MUST have the
value `"application/vnd.uapi.16.manifest"`. It MAY also be set on
other file objects in the sequence (to the same value), but this is
NOT RECOMMENDED.

If `name` is not specified the file object stores information about
the top-level root file object. This file object always comes first
in the file object sequence. Or in other words, the root file object
(the first object in the sequence) has `mediaType` but not `name`
set, while all other file objects have `name` set.

The `name` MUST be a valid
[UTF-8](https://www.rfc-editor.org/rfc/rfc3629) POSIX path (i.e. using the slash `/`
as component separator), MUST NOT contain control characters
([ASCII](https://www.rfc-editor.org/rfc/rfc20) 0…31, 127) and MUST NOT contain "`.`", "`..`" or empty strings as components
of the path. It MUST NOT start or end with a slash. It MUST NOT have
multiple slashes in immediate sequence (or in other words, the path
must be normalized and relative). The name MUST NOT be identical to
"`Uapi16Manifest`" or be prefixed with "`Uapi16Manifest.`" (it is
however acceptable if these filenames appear as components of paths
further down the tree). Files whose paths do not follow these rules
cannot be encoded in manifest files defined by this
specification. The `name` MUST be uniquely assigned within a
manifest.

The `type` field encodes the inode type of the file object. It takes
one of `reg`, `dir`, `lnk`, `fifo`, `chr`, `blk`, `sock` (these values
follow low-level UNIX/Linux naming of these inode types). Other values
MUST NOT be used. If not specified it defaults to `dir` for the first
object in the sequence (i.e. the root file object) and `reg` for all
others. Note that when using the manifest format to list partition
contents each partition MUST be listed as a file of type `reg`.

The `size` (unsigned integer) field MAY be used in file objects of
type `reg`, `lnk`, `dir` and encodes the size of the contents of the
file, in bytes. It MUST NOT be used on file objects of other
types. Specifically, for regular files this is the size as reported by
the file system. In case of symlinks: this is the length of the
symlink target string. In case of `dir` this is the size of a UAPI.16
file manifest generated for the subdirectory (which only applies if
the directory contents are not provided inline, but via a reference
to a separate sub-manifest file, see above).

The `major` (unsigned integer) field MAY be used in file objects of
type `blk` and `chr`, it encodes the major device number of the device
node. It MUST NOT be used on objects of other types. Similarly,
`minor` (same type and condition) encodes the minor device number.

The `mode` (unsigned integer) field encodes the UNIX access mode of
the file object. It MAY be used in file objects of all inode types,
except `lnk`, where it MUST NOT be used. Note that while UNIX access
modes are typically written in octal, this one is encoded in a regular
JSON number, i.e. decimal. The valid range is 0…4095 (i.e. `0o0000` to
`0o7777`). Values outside this range MUST NOT be used.

The `uid` (unsigned integer) field is the numeric owning UNIX user ID of the
file. Similarly, `gid` (same type) is the numeric owning UNIX group ID
of the file. Both fields – if set – MUST be in the range 0…2³²-2.

The `userName` field is the owning UNIX user name of the file;
similarly, `groupName` is the owning UNIX group name of the file. Both
fields SHOULD contain a valid UNIX user name or group name,
respectively. (Note that UNIX user/group
name validity is not very well defined, hence it's recommended to
liberally accept strings here, maybe with the exception of empty
strings, fully numeric strings, and those containing control
characters).

The `mTimeNSec` field (unsigned integer) is the UNIX file modification
timestamp of the file object, in nanoseconds since the UNIX epoch (Jan
1st, 1970).

The `inodeToken` field (unsigned integer) shall be a unique numeric
identifier for any file that SHALL be hardlinked within the manifest
tree. If unspecified or 0 the file is not subject to hardlinks,
otherwise any pair of file objects that shall be hardlinks to the same
inode SHALL carry the same `inodeToken` value. In order to ensure
reproducibility `inodeToken` SHALL be assigned as a counter starting
at 1 for the first file that shall be hardlinked, and increasing by 1
for each subsequent file that shall be hardlinked but has not been
hardlinked in the manifest so far. When sub-manifests are used,
hardlinking between the upper and lower manifest is not supported –
the `inodeToken` identifiers are specific to the manifest they are
specified in. This field MUST NOT be used for file objects of type
`dir`.

The `gpt*` fields encode fields that are needed when placing these
resources in a GPT partition table entry. They only apply to file
objects of inode type `reg`, with the exception of `gptDiskUuid` which
only applies to the root file object (which has an inode type of
`dir`). These fields MUST NOT be used on file objects of other types.

The `gptFlags` field (unsigned integer) encodes the raw 64-bit
partition attribute flags of the GPT partition table entry, as defined
by the [UEFI
specification](https://uefi.org/specs/UEFI/2.11/05_GUID_Partition_Table_Format.html).

The `gptFlag*` fields are of boolean type. They provide access to
individual partition attribute flags by name: `gptFlagNoAuto` and
`gptFlagGrowFileSystem` correspond to the flags of the same name
defined in the [Discoverable Partitions
Specification](discoverable_partitions_specification.md) (bits 63 and
59, respectively). If both `gptFlags` and a `gptFlag*` field are set
for the same file object, they MUST be consistent.

The `gpt*Uuid` fields take a UUID, and MUST be formatted as per [RFC
9562 Section
4](https://www.rfc-editor.org/rfc/rfc9562#section-4). It SHOULD be
formatted in lowercase. `gptUuid` stores the partition UUID for the
entry, and `gptTypeUuid` the partition type UUID.

The `gptLabel` field MUST NOT be longer than 36
[UTF-16](https://www.rfc-editor.org/rfc/rfc2781) code units (72
bytes), as defined by the UEFI specification. If `gptLabel` is
unspecified, and the data is written to a GPT partition it SHOULD use
the value of `name` as label instead. Note that this might require
mangling, as GPT partition labels have smaller size limits than file
names. Also note that unlike the `name` this field does not have to be
uniquely assigned among all entries in the same manifest.

The `gptDiskUuid` field MUST be a UUID in RFC 9562 Section 4
formatting. It SHOULD be formatted in lowercase. It encodes the disk
UUID for the GPT partition table. In contrast to the other `gpt*`
fields this one MUST NOT be set on any object except for the root
directory object (i.e. the first file object in the sequence), as it
encodes a field of the GPT partition table header rather than the GPT
partition entry header.

`readOnly` (a boolean) is relevant for initializing the read-only flag
(bit 60, as defined in the Discoverable Partitions Specification) of
GPT partition table entries. It should also be reflected in the
POSIX 'w' access mode bit when storing the data in a regular file on
disk. If unspecified, defaults to false. If both `mode` and `readOnly`
are set for the same file object, they MUST be consistent: if any of
the three 'w' bits in `mode` is set, `readOnly` MUST be false, otherwise
it MUST be true.

`contents` is an array of contents objects, as described below. This
field only applies to file objects of type `reg`, `lnk`, `dir`. It
MUST NOT be used on file objects of other types. In case of regular
files this references or contains the file contents. In case of
symlinks this references or contains the symlink target string. In
case of directories this references or contains the sub-manifest for
the directory (if applicable, see above). If this field is not set and
the file object is of type `reg`, an array with a single contents
object SHALL be implied, that uses `file` to reference a file of the
name specified in `name` (without encoding or offset). Or in other
words: if no `contents` is specified in a file object of a
`Uapi16Manifest` file, then it should be looked for in a file of the
name `name` in the same directory.

`sha256` is the
[SHA256](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.180-4.pdf)
hash of the file contents. It MUST be formatted
in 64 hexadecimal characters. Parsers MUST parse this
case-insensitively, but it SHALL be formatted in lowercase. This field
only applies to file objects of type `reg`, `lnk`, `dir`. It MUST NOT
be used on file objects of other types. The hash value covers the
contents referenced or contained in the `contents` field described
above. Additional hash algorithms may be defined in future, in which
case separate fields will be defined for each. If the `size` field is
zero, this field MUST either be unset (RECOMMENDED) or be identical to
`e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`,
i.e. the hash of an empty string.

`valid*USec` is in µs since UNIX epoch UTC, and shall be an unsigned
integer. It defines a validity time window of the resource. The file
object SHOULD NOT be accessed or consumed outside of the specified
time window. If `validSinceUSec` is not specified it MUST default to
0, if `validUntilUSec` is not specified the object never expires. If
a directory file object is invalidated via these two settings, then
all file objects within it are invalidated implicitly too. This
specifically means that the time window specified on the root
directory object (i.e. the first one in the sequence) may invalidate
the whole manifest file.

`validity` takes a string, and implements an enumeration: one string
of `good`, `revoked`, `stepping-stone`. Other values MUST NOT be used
at this time (if encountered while parsing any unrecognized strings
MUST be considered equivalent to `good`, in order to cater for future
extensions of this field). If unspecified, defaults to `good`. If set
to `revoked` the file SHOULD NOT be considered anymore for automatic
consumption by the client, and if a client already acquired it before
it SHOULD consider removing it again or otherwise ensure it's not used
anymore. If set to `stepping-stone` and the manifest file is processed
by a software update tool, and lists multiple versions of the same
resource in individual files, then the update tool SHOULD ensure that
files where this is set are never skipped for version updates, and
installed and activated before considering any later versions of the
same resource.

## Out of Scope

Note that the above does not support – on purpose – the following
common UNIX file or GPT partition table properties:

1. Access timestamps, birth timestamps, change timestamps
2. Inode numbers and other non-portable file IDs (such as [`name_to_handle_at()`](https://man7.org/linux/man-pages/man2/name_to_handle_at.2.html) style FID)
3. Backing device inode major/minor
4. Hardlink link counts

This is because all of these will quite likely differ if the same manifest is applied to different file systems.

## Contents Object

```json
{
        "file" : "…",
        "url" : "https://…",
        "literal" : "TmV2ZXIgZ29ubmEgZ2l2ZSB5b3UgdXAsIG5ldmVyIGdvbm5hIGxldCB5b3UgZG93bg==",
        "encoding" : "gzip",
        "encodedSize" : 2011,
        "encodedSha256" : "0ef149998289474e4bb31813edda6ad7f3c991b2d8dec6e8fe4db7a1f039f2d1",
        "originalSize" : 4711,
        "originalSha256" : "afe2882a75c0c603b8b8de5efc20b9043fa33eb0e0cc0d91e54e1dfc2f5fdb8e",
        "offset" : 22
}
```

Contents objects indicate where the data of a `reg`, `lnk`, `dir` type
file object may be found. The `contents` field of the file objects (see
above) contains an array of objects of this type.

### Fields

`file`, `url` and `literal` are mutually exclusive: at most one of them
MAY be set. `file` references
a file in the same location as the manifest file itself was found,
`url` references a full URL, and `literal` contains inline
[Base64](https://www.rfc-editor.org/rfc/rfc4648) data. `url` MUST NOT use any other schemes than `http://…` and
`https://…`. It is RECOMMENDED to use `literal` to encode the target
of symbolic link file objects. If none of these fields are set,
behavior equivalent to `file` being set to the file object's `name`
field SHOULD be implied.

`literal` may be encoded either in regular Base64 ([RFC 4648 Section
4](https://www.rfc-editor.org/rfc/rfc4648#section-4)) or in URL-safe
Base64 ([RFC 4648 Section
5](https://www.rfc-editor.org/rfc/rfc4648#section-5)). Consumers MUST
be able to deal with either alphabet.

`encoding` specifies the encoding of the specified resource. It
accepts the same encoding specifiers as HTTP's
[`Content-Encoding:`](https://www.rfc-editor.org/rfc/rfc9110#section-8.4)
field, e.g. [`gzip`](https://www.rfc-editor.org/rfc/rfc1952) or
[`zstd`](https://www.rfc-editor.org/rfc/rfc8878). If not set, no encoding is used.

`originalSize` is the size of the full, decoded data, `encodedSize` of
the full, encoded data. If `encoding` is not specified, `encodedSize`
SHOULD NOT be specified. (If specified anyway, it MUST match
`originalSize`). Both fields are unsigned integer type.

`originalSha256` is the SHA256 sum of the full, decoded data,
`encodedSha256` of the full, encoded data. If `encoding` is not
specified, `encodedSha256` SHOULD NOT be specified (if specified
anyway it MUST match `originalSha256`). Both are 64 character
hexadecimal strings (case-insensitive, but lowercase RECOMMENDED). If
`encodedSize` is zero then `encodedSha256` MUST be unset (RECOMMENDED)
or be identical to
`e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`,
i.e. the hash of an empty string. Similarly, if `originalSize` is zero,
`originalSha256` MUST be unset (RECOMMENDED) or be the hash of an
empty string.

If `offset` is specified, then the contents of the file SHALL be
acquired from the specified offset in the *decoded* data. If the field
is not specified it defaults to 0. The data to extract begins at the
specified offset, and extends over the size specified in the `size`
field of the file object. Note that this means that the sum of
`offset` and `size` MUST be less than or equal to `originalSize` (if
all are specified), so that the slice does not extend beyond the
source data.

## Signature Objects

Optionally, manifest files MAY include one or more signature
objects. If included these objects MUST appear in the JSON-SEQ
sequence after the last file object. Each such object MUST follow the
following general structure:

```json
{
        "mediaType" : "application/vnd.uapi.16.signature",
        "signatures" : [{…}, …]
}
```

### Fields

The `mediaType` is set to `application/vnd.uapi.16.signature` for
objects of this type. This field MUST be used to recognize the
signature object, and distinguish it from a file object.

The `signatures` field MUST be set to an array of signature data
objects (or in other words: multiple signatures with distinct keys or
mechanisms are supported within the same manifest file).

```json
{
        "mechanism" : "openpgp",
        "key" : "…",
        "data" : "…"
}
```

The `mechanism` field MUST be set to a string identifying the
signature mechanism. Currently defined are `openpgp` for
[OpenPGP](https://www.rfc-editor.org/rfc/rfc9580) compatible
signatures, and `pkcs7` for [PKCS
#7](https://www.rfc-editor.org/rfc/rfc2315). This is the only
required field, the other fields' existence depends on the used
mechanism.

The `key` field MAY point to a key identifier, as appropriate for the
selected mechanism. For the `openpgp` mechanism this is the key
fingerprint in hexadecimal characters (lowercase). For `pkcs7` this is
the SHA256 hash of the signing key's
[X.509](https://www.rfc-editor.org/rfc/rfc5280) certificate in
[DER](https://www.itu.int/rec/T-REC-X.690) form,
formatted in lowercase hexadecimal characters. The `key` field may be
used to optimize searching for the right key to authenticate with, but
its inclusion is OPTIONAL.

The `data` field contains the signature data in Base64 encoding
(either alphabet, as for the `literal` field of contents objects). For
the `openpgp` mechanism this is the binary signature (i.e. without
[ASCII armor](https://www.rfc-editor.org/rfc/rfc9580#section-6)). For
`pkcs7` this is the binary signature in DER.

The cryptographic signature MUST cover the exact, complete binary data
of the JSON-SEQ sequence up to (but not including) the ASCII RS (0x1e)
byte immediately preceding the *first* signature object.

If multiple signatures are provided, a tool processing manifest files
may freely pick *one* of them to authenticate the manifest file, and
ignore all others. Which one it picks is up to the tool, but SHOULD
typically be dependent on locally supported mechanisms and recognized
keys.

Note that the payload the signatures cover only the objects preceding
the first signature object in the sequence, and excludes all signature
objects and the trailer object.

## Trailer Objects

A manifest MUST contain a single trailer object. This object
MUST appear in the JSON-SEQ sequence as very last object, and
terminates the sequence. It only has a single field:

```json
{
        "mediaType" : "application/vnd.uapi.16.trailer",
}
```

Parsers MUST terminate once this object is seen, and fail if there is
any further data (after the LF character that terminates JSON-SEQ
object) left in the byte stream. If a parser encounters a manifest
without the trailer object it MUST assume the file is incomplete and
generate an appropriate parsing failure.

### Fields

The `mediaType` is set to `application/vnd.uapi.16.trailer` for
objects of this type. This field MUST be used to recognize the
tailer object, and distinguish it from a file or signature object.

## Compression

Optionally, manifest files MAY be compressed with mechanisms such as
`gzip`, `zstd` or similar. In this case the filename SHOULD
be `Uapi16Manifest.gz`, `Uapi16Manifest.zst` and so on, as appropriate
for the chosen algorithm.

Such compressed manifest files MUST be decompressed before
validating the embedded signatures (as described above).

## Relationship to SHA256SUMS

A UAPI.16 manifest file that only contains the top-level `dir` file
object in combination with any number of `reg` file objects – of which
each only use the fields `name` and `sha256` – may be mapped 1:1 to a
SHA256SUMS file and back.

## Acquiring a File Remotely

When acquiring a file listed in a UAPI.16 File Manifest from a web
service, the following logic should be implemented.

1. If `validity` is set to `revoked`, the download SHOULD immediately
   fail.

2. If `validSinceUSec` is set and greater than the current time, the
   download SHOULD immediately fail.

3. If `validUntilUSec` is set and lower than the current time, the
   download SHOULD immediately fail.

4. The `contents` array MUST be iterated, and a suitable entry be
   picked as source for the file contents. If no `contents` array is
   specified, proceed as in step 7, with an implied `file` field set
   to the `name` of the file object.

5. If the selected contents object has `literal` set, it MUST be
   Base64 decoded, continue in step 8.

6. Otherwise, if `url` is set, the data from the URL MUST be
   acquired, continue in step 8.

7. Otherwise, if `file` is set, the data MUST be acquired from the
   same location as the manifest itself, however with the last
   component of the location replaced by the file name encoded in
   `file`, following the same semantics as URI [relative
   references](https://www.rfc-editor.org/rfc/rfc3986#section-4.2). Continue in step 8.

8. If `encodedSize` is set the size of the acquired data SHOULD be
   checked against `encodedSize`. (This check – and similar checks
   below – may be omitted if for efficiency reasons the encoded file
   is only acquired partially.)

9. If `encodedSha256` is set, the hash of the acquired data SHOULD be
   matched against it.

10. If `encoding` is not set, but `originalSize` is, the size of the
    acquired data SHOULD be checked against `originalSize`.

11. If `encoding` is not set, but `originalSha256` is, the hash of the
    acquired data SHOULD be checked against `originalSha256`.

12. If `encoding` is set, the acquired data MUST be decoded according
    to the algorithm indicated in `encoding`.

13. If `encoding` and `originalSize` are set, the size of the
    resulting decoded data SHOULD be checked against `originalSize`.

14. If `encoding` and `originalSha256` are set, the hash of the decoded
    data SHOULD be matched against it.

15. The byte range indicated by `offset` (if not set: 0) and `size`
    (if not set, the remainder of the file) MUST be extracted from
    the decoded data.

16. The hash of the extracted data MUST be matched against the hash
    encoded in `sha256` (if set).

If all these steps succeed the extracted data from step 15 is the
result of the operation.

Of course, many of the steps described above SHOULD typically be done
together rather than serially for robustness, efficiency and security
reasons. For example, if `encoding` is not used, it is recommended to
include the `offset` field (of the contents object) and the `size`
field (of the file object) in [HTTP range
request](https://www.rfc-editor.org/rfc/rfc9110#section-14) fields already (in
order to avoid downloading redundant data). Moreover, downloads SHOULD
fail immediately once the encoded or decoded data as it is acquired
goes beyond the indicated encoded or original data sizes. Then, the
decoding SHOULD be done on-the-fly while the data is downloaded, and
the checksum should be calculated on-the-fly too.

## Contents Source Considerations

Applications may freely compile content object arrays when generating
manifests, and are free to process the specified content object arrays
in any order of preference when consuming them. A few suggestions:

* An application might want to prefer placing file contents in
  `literal` fields whenever the (encoded) file size is below some
  threshold, because embedding the data in the manifest (even if it is
  slightly increased in size because of that) might be more efficient
  in both space and processing time than referencing external resources
  that might need to be acquired in separate requests.

* An application might want to prefer not encoding (i.e. compressing)
  files below some size threshold, simply because the space savings the
  compression brings might not be worth the more expensive (in
  both space and processing time) acquisition of the resources.

* When including file contents in `literal` fields an application
  might choose not to add the `sha256` field of the same data, since it
  is redundant given that the literal data is also present in the
  manifest.

* When acquiring file contents from a manifest an application has
  freedom to prefer certain sources over others, or exclude some. For
  example, it is probably wise to prefer `literal` sources over the
  others, as well as local `file` sources over (remote) `url`
  sources. It might also choose to retry acquiring the source with a
  different source of the specified ones if one does not work (for
  example because the source is not accessible or doesn't actually
  have the resource in question). It might also prefer some `url`
  sources over others, for example by determining that they are
  located closer on the network. It might also opt for not
  implementing the `url` source type at all, and instead requiring
  `literal` or `file` sources, depending on context.

## Reproducibility

This specification makes no attempt to define a single strictly
reproducible profile of JSON or the manifest format. Its authors
acknowledge that different JSON implementations will format objects
and order fields differently depending on data, context and intended
audience. Implementations may choose to include file metadata in
varying detail in the manifests, for example UID/GID ownership might
be relevant in some contexts and not in others. Because of that the
cryptographic signatures are based on the actually formatted manifest
file in the format the JSON library in use prefers, rather than a
normalized representation of it.

Nonetheless implementations SHOULD generate manifests reproducibly for
the same input and the same selected fields – as long as the same
tooling in the same version is used. At the most basic this means
stable file object sorting should be employed, even if no specific one
is mandated by this specification. The natural ordering of the
underlying file system SHOULD NOT be propagated into manifests
generated from it.

## Extensibility

Additional fields MAY be defined freely by implementors. Each such
extension field MUST be named in the style of `x<Vendor>Foobar` to
minimize risk of conflicts. For example `xAmutableFrobnicate` or
`xMyAwesomeProjectEfficiencyVector`. When reading implementations MUST
ignore any fields they do not recognize.

Additional objects MAY be inserted at any place in the JSON-SEQ
sequence, except in front of the first (i.e. root file) object or
after the last (i.e. trailer object). They MUST be marked via a
`mediaType` field, that describes what the object contains. When
processing a manifest file a reader MUST ignore (but SHOULD probably
log about) objects in the JSON-SEQ stream it does not recognize. Note
that any objects inserted before the first signature object will be
protected by the signature, but any objects inserted after will not
be. A tool processing manifest files MUST take this into account and
potentially ignore/refuse any objects appearing after the signatures
in the JSON sequence as unauthenticated data.

## Future Revisions

This specification is intended to be incrementally
improved. Additional fields may be defined at any time, with
additional, optional metadata. Should a breaking change be necessary
the media type will be changed.

A number of future extensions are envisioned:

* Fields for file [extended
  attributes](https://man7.org/linux/man-pages/man7/xattr.7.html),
  [POSIX ACLs](https://man7.org/linux/man-pages/man5/acl.5.html)

* Fields for extended file flags (Linux
  [`chattr`](https://man7.org/linux/man-pages/man1/chattr.1.html), DOS
  file system flags, …)

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
        "encoding" : "gzip",
        "encodedSize" : 5642649603
    }],
    "size" : 7523532800,
    "sha256" : "922a9bae0e02b4ffac3e5ed5054230d0689b9c2e25b0178ba82b925f2a0c3e48",
    "validUntilUSec" : 1776856773123234
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
    "validUntilUSec" : 1776856773123234
}
```
`<ASCII-RS>`
```json
{
    "name" : "FooOS-root.raw",
    "contents" : [{
        "file" : "FooOS.raw.gz",
        "encoding" : "gzip",
        "encodedSize" : 5642649603,
        "originalSize" : 7523532800,
        "offset" : 351272960
    }],
    "size" : 234003200,
    "sha256" : "00b70a0813e15f309828d3a36156283cba87576c26755e0b2d4cf0951eff8163",
    "gptLabel" : "FooOS-root",
    "gptUuid" : "1d2e3f40-51a6-4b78-9c0d-2e4f6a8b1c3d",
    "gptTypeUuid" : "4f68bce3-e8cd-4db1-96e7-fbcaf984b709",
    "readOnly" : true,
    "validUntilUSec" : 1776856773123234
}
```
`<ASCII-RS>`
```json
{
    "mediaType" : "application/vnd.uapi.16.signature",
    "signatures" : [{
        "mechanism" : "openpgp",
        "key" : "16b1c4eec0bc021ac777f681b63b21879c3485b0",
        "data" : "iQIzBAABCgAdFiEEFrHE7sC8AhrHd/aBtjshh5w0hbAFAmn8rUoACgkQtjshh5w0hbAz3g//eyWO1swKXivio1PH2oJclEAYqpyqkv9LC293ePyDp5K0zL7BEVt9ijCGb22Wcu+xBXug4SBpSsCMOcro633Kn0ACBeOHwsyu1rnumb85/xvx3lSiq7ke1HcP5OWh5XX4lTm3KYVaPLQy0P3ThupJJsCa0o7uDdl2fXEgcZtRGhgp7GnpQabEuUUGW+1YZ1DFhhIdewZ6+eZZac0DPFjevBbI14jf9wEp3uWT9YoULF0wWBUi45vxdievwIRbcQHQVhxKMaTm0yuTbT5kSUAZccsOkJ4gqrVoNmFx0kYPve0uWcEkFQ16zEKVESr/fhDuCW19v3D46JNvwxZgIxP3o5rp3wbOja6n5D8Otl0kUQaE7FPOmYI1SZ2F97cYsKNViuUyrMcPTNGkI4E18H4WgWLf1CEyGHT16KmoCmldqVhQiTFCtOzkJdbwJgnSL/f0n2y3H/iB3lL0hglv2SDDVnVeMBx+qs+j2DMys3SVOOAYqGrMay1PYxLTEct+M1D0aAmH+XObuoVcm3XMtPPt6NCzxes0cdmFCdMTGERYLY522YCruJpO7p4h9IEoKqDACot0vDD0qwnjBCPu7ETrb02VvJ/W3GJ7/3Y8lIvZFtJqs5TQuxU0d4OiDHe08XdwZrcAondrX5h3Up7mpQIIF4RAMqUaYdOWEf7WzFZBKvY="
    }]
}
```
`<ASCII-RS>`
```json
{
    "mediaType" : "application/vnd.uapi.16.trailer"
}
```

The above defines three separate files `FooOS.raw`, `FooOS-esp.raw`
and `FooOS-root.raw`. All files may be acquired via the same data file
`FooOS.raw.gz` (which itself is however *not* listed in the manifest
as a file object). Note that the first defined file covers the whole
disk image, while the other two cover slices of it, each encapsulating
an individual partition. The data file is encoded via `gzip`.
