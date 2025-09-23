---
title: File Hierarchy for the Verification of OS Artifacts (VOA)
category: Concepts
layout: default
version: 1
SPDX-License-Identifier: CC-BY-4.0
---

# File Hierarchy for the Verification of OS Artifacts (VOA)

## Motivation

Cryptographic validation of artifacts with the help of digital signatures is a use-case of most Linux distributions.
Different cryptographic technologies exist and can be used for this purpose.
Currently, OpenPGP and X.509 are widely adopted.

As of this writing, no technology-agnostic, standardized location for the distribution of cryptograpic material that serves as verifier for digital signatures exists.
This leaves consumers to either do guesswork, or rely on proprietary, stateful or technology-specific keystore formats and per-application locations.

The _File Hierarchy for the Verification of OS Artifacts (VOA)_ defines a generic approach for storage and retrieval of _signature verifiers_, supporting a wide range of cryptographic technologies.

VOA aims to cover all verification needs around OS artifacts.

Unlike e.g. Mozilla's Network Security Services ([NSS]), which concerns itself with secure network communication, including a curated set of X.509 CA certificates (e.g. often provided on Linux distributions via `/etc/pki/` or `/etc/ca-certificates/`), VOA focusses on the verification of artifacts with support for a diverse set of cryptographic technologies.

## Terminology

This text uses the generic term _"signature verifier"_ for cryptographic objects that are used to verify signatures.
In practice, those may be for example X.509 or OpenPGP certificates (the latter are also referred to as public keys).

All technologies for validation of signatures rely on _public key material_.
However, some systems use bare _public key material_, while others combine key material with additional metadata, forming composite objects that may e.g. also make identity assertions.

This text uses the term _signature verifier_ to refer to two types of verifiers:

- _artifact verifiers_, which are used for the validation of signatures on artifacts,
- _[trust anchor]s_, which are used to ascertain the authenticity of _artifact verifiers_.

### Revocation of verifiers

Some technologies have a concept of invalidation of verifiers, using a mechanism that signals to users that a verifier should not be relied on anymore (e.g. because the private key material has been compromised).

### Classification of signature verification models

For the use in the VOA structure, digital signature technologies may be regarded outside of their usual context.
To simplify and allow a unified view on these technologies, VOA classifies these three distinct families:

#### Point to Point

(e.g. SSH, minisign, signify)

Keys are "atomic" and don't have formal relationships with each other: A key issues a number of signatures, which can be verified.

These systems use "set-style trust": Users choose to rely on some _set_ of keys/verifiers, and are willing to accept signatures made with those.

This model doesn't require or support the use of _trust anchors_ in VOA.
In this model, the set of _artifact verifiers_ is considered implicitly trusted and used directly for the verification of artifacts.

#### Decentralized delegation

(e.g. OpenPGP)

In these systems, key material and certifications (as used in the OpenPGP "Web of Trust") are decentralized, forming a non-hierarchical network.
Users need to specify the _trust anchors_ they want to rely on, within their context.

Two approaches are possible for applications using VOA:

- Directly rely on _artifact verifiers_ for a [purpose].
- Rely on _trust anchors_ which delegate to one or more _artifact verifiers_.
  Anchors must be explicitly defined in the relevant "trust-anchor" VOA directory (see [purpose]).

#### Hierarchical delegation

(e.g. X.509, SSH+CA)

Other systems have an inherent assumption of hierarchical delegation, formalized by cryptographic certifications.
In such systems, globally or locally accepted _trust anchors_ are used as starting points in signature validation.
Certificates and the delegation relations between them in practice often form a tree structure.
Such hierarchical key structures are often referred to as [public key infrastructure] (PKI).
In many X.509 ecosystems a shared set of _trust anchors_ exist, that all actors agree upon (e.g. _Web PKI_).

This model requires explicitly defined _trust anchors_ in VOA, that delegate to _artifact verifiers_.

### Application of signature verification models

Although the [classification of signature verification models] categorizes various technologies for the verification of digital signatures, the reality is more diverse.
In practice VOA only distinguishes between direct verifications and those relying on some form of delegation.
The details of delegation are technology specific and described in their respective [technology] sections.

### Typical distribution format of verifers / short vs. long-lived keys

VOA's goal is to provide a uniform representation of verifiers, to the degree that this is possible with differing technologies.
To achieve this, verifiers are stored in a central file hierarchy that can be used with different technologies.

Some concerns may be handled differently in supported technologies:

- The separation between _trust anchors_ and _artifact verifiers_ may not be reasonable to model in separated directories, because the native representation of a given technology is to store both in a shared file, as a type of "certificate chain".
- By default, VOA suggests that each verifier is stored in an individual file (that ideally reflects a unique identifier of that verifier in its filename).
  However, in some technologies, separating verifiers into individual files may be uncommon or unreasonable.
  In these technologies, bundles of verifiers may be stored in a shared file (this can apply equally to _artifact verifiers_ and _trust anchors_).
- As an edge case, in some technologies, _artifact verifiers_ may be extremely short lived, so it may not be reasonable to store them in VOA.
  In these cases, only a set of _trust anchors_ may be stored in VOA, while the _artifact verifiers_ are distributed with the signed artifacts.

## File Hierarchy

The _File Hierarchy for the Verification of OS Artifacts (VOA)_ is maintained as a directory structure on a system.
It contains cryptographic public-key material for one or more technologies.

The VOA hierarchy organizes _signature verifiers_ by [os], [purpose], [context] and [technology], and stores them in a directory structure of the form `$os/$purpose/$context/$technology/`.

Vendors provide a set of _signature verifier_ files for their OS, in the VOA hierarchy format.

Users of a VOA hierarchy (such as package installation software) can pick the relevant set of _signature verifiers_ for the verification of a specific artifact based on their location in the hierarchy.

VOA does not concern itself with the retrieval of additional _signature verifiers_ or the update of existing ones in the file hierarchy.

### Load paths

VOA defines a list of _load paths_ with descending priority for system mode and user mode.

_Signature verifiers_ are retrieved from hierarchies in these directories to validate artifacts based on specific [load logic].
This is referred to as _verifier lookup_.

The existence of multiple load paths allows system administrators and users to define custom sets of _signature verifiers_ for their system and applications.

#### System mode

The priority of load paths in system mode follows the definition of ["Storage Directories and Overrides" in the Configuration Files Specification], which is also reflected in the following list of paths:

- `/etc/voa/`
- `/run/voa/`
- `/usr/local/share/voa/`
- `/usr/share/voa/`

#### User mode

The priority of load paths in user mode mirrors that of the system mode.
Refer to the [XDG Base Directory Specification] for details on default values.

- `$XDG_CONFIG_HOME/voa/`
- the `./voa/` directory in each directory defined in `$XDG_CONFIG_DIRS`
- `$XDG_RUNTIME_DIR/voa/`
- `$XDG_DATA_HOME/voa/`
- the `./voa/` directory in each directory defined in `$XDG_DATA_DIRS`

### Symlinking

Load paths are constrained to self-contained locations on a host as they provide vital data for the integrity and verification of all components on a system.
However, relative symlinks can be used in the VOA hierarchy to point to files or directories below the same [load path] or one of the _load paths_ in descending priority.

As an example, symlinks to files below the same load path or to another load path with lower priority may be used to deduplicate the use of a single _signature verifier_ for multiple use-cases.
As another example, using symlinks allows to automatically keep _signature verifiers_ in sync with canonical upstream data.

VOA implementations must not consider symlinks under the following conditions and raise a warning for them instead:

- the symlink is for a file or directory in an ephemeral load path (i.e. `/run/voa/` and `$XDG_RUNTIME_DIR/voa/`),
- the symlink is for a file or directory outside of the specified _load paths_,
- the symlink is broken (a.k.a. "dangling"),
- the file name of the symlink (also those of intermediate symlinks in the case of a symlink chain) does not match that of the target,
- or the file type of the source and target of the fully resolved symlink do not match (e.g. a verifier file points to a directory, or a directory to a verifier file).

### Masking

Individual _signature verifiers_ may be masked using a symlink to `/dev/null`, independent of [technology].
This constitutes the only allowed exception to the rule of not symlinking to files or directories external to the load paths.
These symlinks are only expected to be used in writable load paths (that is, only below `/etc/voa/` or `/run/voa/` when in system mode, or below `$XDG_CONFIG_HOME/voa/` or `$XDG_RUNTIME_DIR/voa/` when in user mode).
A _signature verifier_ that is masked is considered invalid in all [load paths] independent of technology and [load logic].

Masking _signature verifiers_ may be desirable in situations where a verifier is considered compromised, but a revocation for it can not be provided to users in time.
VOA offers this mechanism as an alternative to revocation.

Masking is explicitly designed to be used on individual _signature verifiers_.
The masking (of parts) of the directory structures that contain them on the other hand does not yield meaningful or intelligible results for end-users and would be very complex to reason about.
VOA implementations must not consider masking symlinks for directories and should raise a warning if such symlinks are encountered.

### Load logic

Load logic depends on the [technology] in use.
It is the mechanism by which relevant verifier information from all load paths is retrieved, while considering the [os], [purpose], [context] and [technology] request of a calling application.

For some technologies, the first suitable _signature verifier_ found in the list of [load paths] is used, effectively [overriding] files with the same name in directories lower in the list.

However, in other technologies, more complex [merging] logic may apply, that effectively combines information from different representations of one verifier.

Additionally, the concept of [masking] applies for all technologies.

#### Overriding

In technologies that use overriding, verifier files shadow other variants of the same verifier based on filename equality.

For example, `/etc/voa/fedora/packages/default/ssh/b5bb9d8014a0f9b1d61e21e796d78dccdf1352f23cd32812f4850b878ae4944c.pub` overrides `/usr/share/voa/fedora/package/default/ssh/b5bb9d8014a0f9b1d61e21e796d78dccdf1352f23cd32812f4850b878ae4944c.pub`.

Overriding depends on file name equality and directory structure.

For example, `/etc/voa/fedora/packages/default/ssh/7d865e959b2466918c9863afca942d0fb89d7c9ac0c99bafc3749504ded97730.pub` does not override `/usr/share/voa/fedora/package/default/ssh/b5bb9d8014a0f9b1d61e21e796d78dccdf1352f23cd32812f4850b878ae4944c.pub`, but instead adds an additional _signature verifier_.

Great care has to be taken to ensure the naming scheme has the desired overriding behavior (e.g. to make sure that a revoked version of an SSH public key overrides a non-revoked version of the same SSH public key found in a different load path).

#### Merging

Some technologies use a business logic that "merges" different versions of the same _signature verifier_ (instead of applying overriding semantics).
The purpose is to use the latest available information about each verifier, combining the information from all sources.

When merging, information about a verifier found anywhere in the current _verifier lookup_ must be applied for all uses of a verifier (e.g. for revocation status).

Wherever it is applicable and feasible, VOA technology backend implementations should implement "merging" to obtain a unified view of the latest information for each verifier.

For example, if a verifier has been revoked, but only one of two copies of that verifier in VOA reflects this revocation, "merging" the two leads to stable visibility of the revocation status.
Summarized, in technologies that implement merging, [load path] priority is not relevant (e.g. a non-revoked version in `/etc/voa/` of the same verifier never overrides the revoked version in `/usr/share/voa/`).
The "latest normalized view" of a verifier is not impacted by load path priority.

### Future compatibility

No changes to the overall VOA structure are anticipated, which is why versioning is not encoded in the directory structure.
However, if it turns out that a breaking change to the VOA specification is necessary a separate directory structure (e.g. `/usr/share/voa2/`) can be specified.

Independently, breaking changes of the VOA format _within_ individual technologies may occur.
In this case, versioned variants of the technology in question are added (e.g. `openpgp:2`).
The `:` delimiter is chosen to signify VOA-specific versioning.
This differentiates clearly between versions of the technology itself, as specified upstream (e.g. [OpenPGPv4], [OpenPGPv6]) and how these technologies are used in the context of VOA.

## Identifiers

_Signature verifiers_ are located in directory structures described by the [file hierarchy].
Each of the following identifiers represents a subdirectory layer in that hierarchy.

### OS

The _os_ identifier is used to uniquely identify an Operating System (OS).

The identifier relies on data provided by the ubiquitous [os-release].
The following keywords are understood and their value format follows that established by [os-release] (i.e. no spaces and no characters outside of `0–9`, `a–z`, `"."`, `"_"`, and `"-"` are allowed):

- `ID`: name of OS (e.g. `arch` or `debian`)
- `VERSION_ID`: the version of the OS (e.g. `1.0.0` or `24.12`)
- `VARIANT_ID`: the variant of the OS (e.g. `server` or `workstation`)
- `IMAGE_ID`: the image of an OS (e.g. `cashier-system`)
- `IMAGE_VERSION`: version of the image (e.g. `1.0.0` or `24.12`)

The values for these _parts_ must be provided, in the above order, as a colon-separated string that defines a specific _os_ identifier (e.g. `debian:12:server:company-x:25.01`).
At least the `ID` _part_ must be set, an empty _os_ identifier is invalid.
Trailing colons must be omitted for all _parts_ that are unset (e.g. `arch` instead of `arch::::`).

The _os_ identifier may be extended with further _parts_ in an updated specification.
Users encountering an _os_ identifier with more than 5 _parts_ should ignore these directories.

Users of VOA (e.g. a Linux distribution creator) are free to store and retrieve verifiers in more or less specific _os_ identifiers, at their own discretion. VOA supports both generic identifiers (e.g. `arch`), and very specific identifiers (e.g. `fedora:41:workstation:cashier-system:1.0.0`).

VOA is not flexible regarding _os_ strings:
When an application looks up verifiers for e.g. `fedora:41:workstation`, then no verifiers from alternate _os_ strings are implicitly considered (e.g. from the more general location `fedora:41`).

However, VOA libraries may offer an API that allows applications to use a list of _os_ strings (e.g. a list of both `fedora:41:workstation` and `fedora:41`).
Such a call will use the combination of all verifiers found in both directory hierarchies.
Applications that use VOA (via a VOA library) can therefore opt to pass a list of _os_ strings, if verifiers are known to be spread over different _os_ directories.
VOA users can omit specific parts, e.g. `IMAGE_ID` and `IMAGE_VERSION`, if they know that these parts are not used in the [VOA hierarchy] for their _os_.

### Purpose

A _purpose_ combines a _role_ and a usage _mode_:

- A _role_ acts as a trust domain (e.g. the "package" _role_ is used for signatures for package verification).
- There are two _modes_ in which _signature verifiers_ can be stored in VOA:
  - Verifiers used for direct artifact verification
  - Verifiers that serve as _trust anchors_

A _role_ and a _mode_ in combination form a _purpose_.

_Trust anchor_ verifiers are always used to ascertain the validity of the associated _artifact verifiers_ (e.g. _trust anchor_ verifiers for the "package" _role_ are used to validate the _artifact verifiers_ for the "package" _role_).

To use _signature verifiers_ in more than one _mode_ [symlinking] may be used.

#### Directory naming

_Purpose_ directories for direct _artifact verifiers_ are named `$role` (e.g. `package`), while _trust anchors_ are stored below directories named `trust-anchor-$role` (e.g. `trust-anchor-package`).

Note that _trust anchor_ directory names always start with a "trust-anchor-" fragment.

Either two or one _purpose_ directories may exist per _role_:

- One directory which contains _artifact verifiers_ (e.g. `package`), and a second directory which contains the corresponding _trust anchors_ (e.g. `trust-anchor-package`).
- Just one directory which contains _artifact verifiers_ (e.g. `package`).

The _purpose_ directory name must not contain characters outside of `0–9`, `a–z`, `"."`, `"_"`, and `"-"`.
VOA implementations must not consider invalid directories and should raise a warning if such a directory is encountered.

A _role_'s name must not start with the string "trust-anchor-", so that there can be no confusion with a purpose with _trust anchor_ usage _mode_.

##### Two purpose directories: Verification relying on trust anchors

For example, verifiers for the "package" _role_ may be stored in two _purpose_ directories:

- `package` ("_artifact verifiers_ of actors who are designated for package signing"), and
- `trust-anchor-package` ("_trust anchors_ for _artifact verifiers_ used for package signing")

In this scenario, verification based on _trust anchors_ is performed.

##### One purpose directory: Direct verification

In another example, verifiers for the "image" _role_ may be stored in only one _purpose_ directory:

- `image` ("_artifact verifiers_ of actors who are designated for OS image signing")

In this scenario, only direct verification can be performed.

#### Roles as trust domains

Having distinct _roles_ allows the use of separate sets of _signature verifiers_ per role.
This is useful if different actors are expected to issue signatures for each _role_.
Thus, each _role_ acts as a trust domain, e.g. the "package" _role_ is used for signatures for packages, while the "image" _role_ is used for signatures for OS images.
The work on each of the _roles_ may be performed by different teams, using different verifiers.

#### Standard roles

The standard roles defined by the VOA specification are:

* **package**: Verifying signatures for packages
* **repository-metadata**: Verifying signatures for repository metadata
* **image**: Verifying signatures for OS images

The corresponding _purpose_ directories are:

- `package` and `trust-anchor-package`
- `repository-metadata` and `trust-anchor-repository-metadata`
- `image` and `trust-anchor-image`

The above list of standard roles can be extended by users of VOA.

For more in-depth explanation on the use of the _purpose_ identifier, see the [examples] section.

### Context

The **context** identifier allows defining specific verifiers for a particular context within an [os]'s [purpose].

The _context_ identifier allows modelling finer grained trust domains within a _purpose_.
This can be necessary if different actors are responsible for signing in subsets of a _purpose_.

Specific examples for **context** are

- the name of a specific software repository when certificates are used within the **package** [purpose] (e.g. `core` for a repository named "core")
- how an OS image is used within the **image** [purpose] (e.g. `installation-medium`, `virtual-machine`)

If no specific context is required, the **context** directory `default` must be used.

Analogous to the os-release data, **context** strings may only contain `[a-z]`, `[0-9]`, `_`, `.`, and `-`.

For more in-depth discussion on the use of the _context_ identifier, see the [examples] section.

### Technology

The following sections outline specifics about the supported technologies.

In VOA, technology-specific backends implement the logic for each supported technology.
Different parties can implement different backends and this document outlines the approach, including the commonalities between all technology backends.

The details of verifier [load logic] is defined per technology to leverage the individual features and strengths of each technology, while offering semantics that are closely shared between all VOA technologies.

Currently, only the OpenPGP technology is specified in detail.
Further specification work is required before implementing further technologies.

Each technology specifies file suffixes for verifiers.
Note that these suffixes may overlap.

A _technology_ directory name must not contain characters outside of `0–9`, `a–z`, `"."`, `"_"`, and `"-"`.
VOA implementations must not consider invalid directories and should raise a warning if such a directory is encountered.

#### OpenPGP

[OpenPGP] is a widely adopted decentralized system for signature verification and user authentication.

This technology is named `openpgp` in the VOA structure.

Most Linux distributions use OpenPGP, often combined with _PGPKI_ (a.k.a. [Web of Trust (WoT)]), in which chains of trust are evaluated during signature validation.

OpenPGP has a rich model of validity, both for certificates and signatures.
Certificates can expire or be revoked by the key holder (via [OpenPGP certificate revocation]).
Similarly, [third-party identity certifications] (as used in the _PGPKI_) can be revoked by their issuers (using [OpenPGP signature revocation]).

OpenPGP VOA backends are expected to handle validity.
In particular they should take expiration and revocation into account in all layers of the [VOA hierarchy], by implementing [merging] semantics.
For example, even if a revoked version of a certificate is overlaid by an unrevoked version, the revocation must be considered.

OpenPGP VOA backends implementations must offer appropriate facilities to calculate the validity of _artifact verifiers_.
This may include:

- specifying how many _trust anchors_ are required to have valid paths to each _artifact verifier_,
- limiting potentially accepted User IDs of _artifact verifiers_ (e.g. by domain of the email address).

In VOA, by convention, OpenPGP certificates must be stored in individual files and provided in ASCII armored form.

These files must be named by certificate fingerprint (in lowercase hex notation) with the file ending `.openpgp` (e.g. `d8afdda07a5b6edfa7d8ccdad6d055f927843f1c.openpgp`).

OpenPGP VOA backends must reject files where the certificate fingerprint of the file name does not match the actual fingerprint of the contained OpenPGP certificate's primary key and should emit a warning if such a file is encountered.

By default, in VOA, OpenPGP certificates that act as _trust anchors_ are considered with a _trust amount_ of 40 at a _trust depth_ of 1.
More complex delegation setups are possible, but must be implemented using an application specific configuration mechanism.

#### SSH

---

**NOTE**: This technology is in draft mode.
Before implementing an SSH VOA backend, this section needs to be specified further to describe dedicated semantics for key revocation lists.

---

SSH can be used in two distinct [signature verification models]:

- [point to point]: with this model the targeted [purpose] only requires _artifact verifiers_, it does not support _trust anchors_
- [hierarchical delegation]: when relying on [SSH CA] this model expects both _artifact verifiers_ and _trust anchors_

In both cases the [technology] is referred to as `ssh` in the VOA structure.

The file ending `.pub` is used for _artifact verifier_ files, following the default naming convention of OpenSSH's [`ssh-keygen`] output.

File names must consist of the sha256 fingerprint of the SSH public key, encoded in lowercase hex notation (e.g. `b5bb9d8014a0f9b1d61e21e796d78dccdf1352f23cd32812f4850b878ae4944c.pub`).
This format can for example be generated via `ssh-keygen -l -f ~/.ssh/id_rsa.pub | cut -f 2 -d ' ' | cut -d ':' -f 2 | base64 -d | xxd -p -c32`.
SSH VOA backends must reject files where the sha256 fingerprint of the file name does not match the actual sha256 fingerprint of the contained SSH public key and should emit a warning if such a file is encountered.

SSH signatures support the concept of namespaces, which declare a specific scope in which a signature is to be created and used.
In VOA, the equivalent to an SSH namespace is the combination of _signature verifier_ [purpose] and [context].
When verifying SSH signatures with a _signature verifier_, their namespaces should be matched against the combination of the _signature verifier_'s [purpose]/[context] (e.g. `package/core` ).

SSH has various concepts for revoking keys (see [sshd.8#SSH_KNOWN_HOSTS_FILE_FORMAT] and [ssh-keygen.1#KEY_REVOCATION_LISTS]).

VOA does not interpret the global revocation lists and configurations used by [`sshd`] or [`ssh`], but expects revocation information in VOA-specific well-known locations.

Revocations must be handled with [merging] semantics.
For example, a revocation found in any location considered in the current _verifier lookup_ must be applied for all uses of a verifier.

#### X.509

---

**NOTE**: This technology is in draft mode.
Before implementing an X.509 VOA backend, this section needs to be specified further to describe dedicated semantics.

---

X.509 is a widely adopted hierarchical model, that is often used for signature validation.

This technology is named `x509` in the VOA structure.

X.509 _artifact verifier_ files must be used in [Privacy-Enhanced Mail] (PEM) format, with the file ending `.pem`.
Analogous to the use in systemd, file names of the form `*-certificate.pem` must be used for X.509 certificates and `*-public-key.pem` for X.509 public keys.

X.509 VOA backends must implement appropriate [merging] semantics.
In particular, revocations must be considered for all uses of a certificate in a _verifier lookup_, even if some copies don't contain the revocation information.
Further, an implementation must check that the extended key usage of a _signature verifier_ is aligned with its [purpose] (e.g. `codesigning` for certain _artifact verifiers_ and `certificate authority` for _trust anchors_).

It is strongly recommended to use signature formats that include metadata about the raw signature, in particular the signature creation time (e.g. [PKCS#7]/ [CMS]).

#### Minisign

---

**NOTE**: This technology is in draft mode.
Before implementing a minisign VOA backend, this section needs to be specified further to describe dedicated semantics.

---

This technology is referred to as `minisign` in the VOA structure.

Minisign is a "point to point" signing technology.
The targeted [purpose] only requires plain _artifact verifiers_, as there is no support for _trust anchors_ in this technology (i.e. any verifiers in "trust-anchor-*" purpose directories must be ignored, also when the target of a symlink).

There is no concept of [verifier revocation].

The file ending `.pub` is used for _artifact verifier_ files, following the default naming convention of the canonical [`minisign`] output.

#### Signify

---

**NOTE**: This technology is in draft mode.
Before implementing a signify VOA backend, this section needs to be specified further to describe dedicated semantics.

---

This technology is referred to as `signify` in the VOA structure.

Signify is a [point to point] signing technology.
The targeted [purpose] only requires plain _artifact verifiers_, as there is no support for _trust anchors_ in this technology (i.e. any verifiers in "trust-anchor-*" purpose directories must be ignored, also when the target of a symlink).

There is no concept of [verifier revocation].

The file ending `.pub` is used for _artifact verifier_ files, following the default naming convention of the canonical [`signify`] output.

## Examples

The following examples provide an overview for several (hypothetical) scenarios in which VOA may be used.
For more details on valid components for [os] identifiers refer to the documentation of [os-release].

### Package manager verifies package signatures, with trust anchor

In this example, we'll consider the use case of a package management software that uses VOA to verify the signature of a package file on a custom image-based Arch Linux OS.

The package manager issues a lookup for OpenPGP _signature verifiers_ from VOA, based on two [os] strings: `arch:::cashier-system:1.0.0` and `arch`.

A VOA library will search in the [load path] `/usr/share` - in our example the only location with verifier data - and consider both the _trust anchor_ and the _artifact verifier_ paths.
In our example, it will find the following verifier files for the `arch:::cashier-system:1.0.0` _os_ string:

```
/usr/share/voa/arch:::cashier-system:1.0.0/trust-anchor-package/default/openpgp/0beec7b5ea3f0fdbc95d0dd47f3c5bc275da8a33.openpgp
/usr/share/voa/arch:::cashier-system:1.0.0/package/default/openpgp/62cdb7020ff920e5aa642c3d4066950dd1f01f4d.openpgp
```

And additionally, the following verifier files for the `arch` [os] string:

```
/usr/share/voa/arch/package/default/openpgp/bbe960a25ea311d21d40669e93df2003ba9b90a2.openpgp
```

The VOA library will also check for verifier files in the other layers, in our example, there is one additional verifier file in the `/etc` VOA layer:

```
/etc/voa/arch/package/default/openpgp/bbe960a25ea311d21d40669e93df2003ba9b90a2.openpgp
```

Note that this verifier has the same filename as the one above, under `/usr/share/voa/`.
This signifies that both files contain information about the same _artifact verifier_.
If the contents of the files differ, the VOA library will calculate a merged view of both sources, consolidating all available information.
In this example, the OpenPGP certificate in `/usr/share/voa/arch/package/default/openpgp/bbe960a25ea311d21d40669e93df2003ba9b90a2.openpgp`, while the OpenPGP certificate in `/etc/voa/arch/package/default/openpgp/bbe960a25ea311d21d40669e93df2003ba9b90a2.openpgp` is not.
Due to [merging] semantics, the OpenPGP certificate is considered revoked.

The VOA library has effectively found one _trust anchor_ verifier file, and one valid _artifact verifier_ file.
This means that package files will be verified using the one _artifact verifier_ (`/usr/share/voa/arch:::cashier-system:1.0.0/package/default/openpgp/62cdb7020ff920e5aa642c3d4066950dd1f01f4d.openpgp`), which in turn is checked for validity based on the _trust anchor_ verifier (`/usr/share/voa/arch:::cashier-system:1.0.0/trust-anchor-package/default/openpgp/0beec7b5ea3f0fdbc95d0dd47f3c5bc275da8a33.openpgp`).

### Download tool verifies installation medium, without trust anchor

In this example, we'll consider the use case of a download tool that uses VOA to verify the signature of a Fedora 41 installation medium.

The download tool issues a lookup for OpenPGP verifiers from VOA, based on a single [os] string: `fedora:41`.
The tool is configured to perform validation with _artifact verifiers_ (i.e. it specifies that it does not require the use of _trust anchors_).

A VOA library will search in the [load path] `/usr/share/voa/` - in our example the only location with verifier data - and consider the basic verifier paths.
In our example, it will find the following verifier file for the `fedora:41` _os_ string:

```
/usr/share/voa/fedora:41/image/installation-medium/openpgp/62cdb7020ff920e5aa642c3d4066950dd1f01f4d.openpgp
```

The VOA library will also check for verifier files in the other layers, but in in this example, there are none.

The VOA library has found one _artifact verifier_ file (`/usr/share/voa/fedora:41/image/installation-medium/openpgp/62cdb7020ff920e5aa642c3d4066950dd1f01f4d.openpgp`) and proceeds to use it to verify the installation medium file.

## Considerations for implementers and users

### Avoiding duplication of data in the directory structure

Symlinks are explicitly supported when forming VOA structures.
In many cases, it will be desirable to make heavy use of symlinks, to form the appropriate directory structures while avoiding duplication of data.
However, the [VOA hierarchy] is intended as a self-contained data structure that specifies how to authenticate signatures.
Therefore, symlinks in VOA should only point to other files or directories in any of the [VOA hierarchy] paths.
Symlinks may be relative or absolute.
Symlinking files external to the VOA structure is not permitted and should be ignored while raising a warning.

The use of hardlinks in the VOA structure is discouraged to prevent confusion.
However, VOA implementations are not required to check for the use of hardlinks.

### Access library API considerations

As [os] strings are relatively complex, a VOA access library may offer convenient APIs for enumeration/searching of _os_ directories (e.g. looking up a particular _os_ in the set of available _os_ strings, by searching by the _ID_ part of the [os-release] information).

### Constraining verifiers

A VOA library may allow applications to set constraints on successful verification.

These constraints may include scenarios such as:

- an application may require that _trust anchors_ must be present
- an application may mandate the set of verifiers that must be present (either as _trust anchors_, or as _artifact verifiers_)

### Verifier optimization

When importing verifiers, a VOA library may support normalization operations on the verifier representation.
For example, OpenPGP decryption and authentication component keys may be dropped, as they are not needed in any VOA context.

### Masking verifiers

The action of [masking] _signature verifiers_ should be able to distinguish between persistent and runtime directories (e.g. `/etc/voa/` and `/run/voa/`).
By default, persistent locations should be preferred over runtime ones.

### Threshold signing

In some scenarios users of VOA may want to rely on verification schemes that require more than one valid digital signature for a given artifact.
These scenarios are explicitly not part of VOA, but may be implemented using dedicated configuration-based approaches per [technology].

### Retrieval of signature verifiers

By default the _signature verifiers_ found in VOA are usually either provided by vendor updates to the operating system (e.g. in `/usr/share/voa/`) or are created by system administrators (e.g. in `/etc/voa/`).

In certain scenarios, applications may want to retrieve additional or updated _signature verifiers_ from locations outside of the VOA hierarchy.
These _signature verifiers_ should be placed in ephemeral runtime directories (e.g. `/run/voa/`).

### Validity of signatures

Each [technology] may handle validity differently.
Generally, signature validity should be considered at the time of signature creation.
The _signature verifier_ must be valid at creation time.
If the _signature verifier_ has expired after signature creation, this does not impact signature validity.
Only in the case of revocations, that indicate a key material compromise, all signatures by the key in question should be considered invalid, regardless of creation time.

#### Use of Time Stamp Authority

VOA is mainly used for the verification of data signatures.
Often these signatures contain a claimed creation time.

Some technologies support the use of _Time Stamp Authorities_ (e.g. based on the [time stamp protocol]) to validate the creation time claims of signatures.
It is recommended to use timestamping services to validate signature creation time, where possible.
Details on how to use timestamping services are currently out of scope for this specification.

Extending the [purpose] scheme for verifiers used for timestamping purposes may be desired and is possible.
If the need arises, this specification should be extended accordingly.

["Storage Directories and Overrides" in the Configuration Files Sepcification]: https://uapi-group.org/specifications/specs/configuration_files_specification/#storage-directories-and-overrides
[CMS]: https://en.wikipedia.org/wiki/Cryptographic_Message_Syntax
[NSS]: https://firefox-source-docs.mozilla.org/security/nss/index.html
[OpenPGP certificate revocation]: https://openpgp.dev/book/certificates.html#revocations
[OpenPGP signature revocation]: https://openpgp.dev/book/verification.html#revocations
[OpenPGP]: https://openpgp.org
[OpenPGPv4]: https://datatracker.ietf.org/doc/html/rfc4880
[OpenPGPv6]: https://datatracker.ietf.org/doc/html/rfc9580
[SSH CA]: https://liw.fi/sshca/
[PKCS#7]: https://en.wikipedia.org/wiki/PKCS_7
[Privacy-Enhanced Mail]: https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail
[VOA hierarchy]: #file-hierarchy
[Web of Trust (WoT)]: https://openpgp.dev/book/signing_components.html#wot
[XDG Base Directory Specification]: https://specifications.freedesktop.org/basedir-spec/latest/
[`signify`]: https://man.archlinux.org/man/signify.1
[`ssh-keygen`]: https://man.archlinux.org/man/ssh-keygen.1
[`ssh`]: https://man.archlinux.org/man/ssh.1
[`sshd`]: https://man.archlinux.org/man/sshd.8
[`minisign`]: https://man.archlinux.org/man/minisign.1
[classification of signature verification models]: #classification-of-signature-verification-models
[context]: #context
[examples]: #examples
[file hierarchy]: #file-hierarchy
[hierarchical delegation]: #hierarchical-delegation
[load logic]: #load-logic
[load path]: #load-paths
[load paths]: #load-paths
[masking]: #masking
[merging]: #merging
[os-release]: https://www.freedesktop.org/software/systemd/man/latest/os-release.html
[os]: #os
[overriding]: #overriding
[point to point]: #point-to-point
[public key infrastructure]: https://en.wikipedia.org/wiki/Public_key_infrastructure
[purpose]: #purpose
[role]: #purpose
[signature verification models]: #signature-verification-models
[ssh-keygen.1#KEY_REVOCATION_LISTS]: https://man.archlinux.org/man/ssh-keygen.1.en#KEY_REVOCATION_LISTS
[sshd.8#SSH_KNOWN_HOSTS_FILE_FORMAT]: https://man.archlinux.org/man/sshd.8#SSH_KNOWN_HOSTS_FILE_FORMAT
[symlinking]: #symlinking
[technology]: #technology
[third-party identity certifications]: https://openpgp.dev/book/certificates.html#third-party-identity-certifications
[time stamp protocol]: https://en.wikipedia.org/wiki/Time_stamp_protocol
[trust anchor]: https://en.wikipedia.org/wiki/Trust_anchor
[verifier revocation]: #revocation-of-verifiers
