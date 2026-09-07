---
title: UAPI.20 Varlink IPC
category: Concepts
layout: default
version: 0.1
SPDX-License-Identifier: CC-BY-4.0
weight: 20
aliases:
- /UAPI.20
- /20
---

# UAPI.20 Varlink IPC

| Version | Changes         |
|---------|-----------------|
| 0.1     | Initial release |

Varlink is an interface description format and IPC protocol that aims
to make services accessible to both humans and machines in the
simplest feasible way.  A Varlink interface combines what the classic
UNIX command line options, `STDIN`/`STDOUT`/`STDERR` text formats, man
pages and service metadata provide, and makes the equivalent available
over a single file descriptor.

Varlink is plain-text, type-safe, discoverable, self-documenting,
remotable, testable and easy to debug. It is accessible from any
programming environment: clients do not require complex modules or
libraries, existing JSON and local socket communication facilities are
sufficient for basic operation.

This specification consolidates the protocol documentation originally
published at [varlink.org](https://varlink.org/), and extends it with
a definition of how the protocol is bound to specific transports:
`AF_UNIX` stream sockets (including how such sockets are marked so
that they can be recognized as Varlink sockets) and HTTP (in a simple
request/response mode and in a WebSocket mode).

## Design Principles

* **Simplicity.** Varlink aims to be as simple as possible. It is not
  specifically optimized for anything else but ease-of-use and
  maintainability.

* **Discoverability.** Varlink services describe themselves, with a
  machine-readable interface definition and human-readable
  documentation. The documentation is provided to clients as
  structured data on a well-known service interface.

* **Remotability.** It should be easy to forward, bridge and redirect
  Varlink interfaces over any connection-oriented transport. Varlink
  should minimize exposure to side-effects of local APIs. Interactions
  are designed to be simple messages on a network, not carrying direct
  references to locally stored file paths or needlessly relying on
  passing file descriptors.

* **Focus.** Varlink is the protocol and the definition of interfaces,
  but it does not define or provide any significant functionality
  itself.

* **Errors.** Varlink errors carry enough information to be consumed
  by machines and automated rules.

Varlink uses direct connections, has no central message handling component,
and is hence easily debugged, secured, isolated and tested. Everything is
readable: Varlink uses plain text messages, has no magic numbers and no
unnamed values, and is easily debuggable with tools such as `strace`, `netcat`
or `jq`.

## Terminology

* An **interface** is a named collection of types, methods and errors,
  described in the Varlink interface definition language (see below).

* A **service** is a program that implements one or more interfaces and
  accepts connections from clients.

* A **client** is a program that connects to a service and invokes methods.

* A **connection** is a bidirectional, ordered, reliable byte stream between a
  client and a service, established over a *transport*.

* A **message** is a single JSON object exchanged over a connection, either a
  *call* (client to service) or a *reply* (service to client).

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in BCP
14 [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) [RFC
8174](https://www.rfc-editor.org/rfc/rfc8174) when, and only when,
they appear in all capitals, as shown here.

## Interface Definition

A Varlink interface has a reverse-domain name and specifies which methods the
interface implements. Each method has named and typed input and output
parameters. Complex types can be aliased with the `type` keyword to allow
reusing them and to make method signatures easier to read. The interface also
specifies the errors that may be returned from its method calls.

Everything can be documented by adding a comment immediately before it.

Interface definition files are named after the interface they implement with
the suffix `.varlink`, e.g. `org.example.ftl.varlink`.

### Example

```
# Interface to jump a spacecraft to another point in space.
# The FTL Drive is the propulsion system to achieve
# faster-than-light travel through space. A ship making a
# properly calculated jump can arrive safely in planetary
# orbit, or alongside other ships or spaceborne objects.
interface org.example.ftl

# The current state of the FTL drive and the amount of
# fuel available to jump.
type DriveCondition (
  state: (idle, spooling, busy),
  tylium_level: int
)

# Speed, trajectory and jump duration is calculated prior
# to activating the FTL drive.
type DriveConfiguration (
  speed: int,
  trajectory: int,
  duration: int
)

# The galactic coordinates use the Sun as the origin.
# Galactic longitude is measured with primary direction
# from the Sun to the center of the galaxy in the galactic
# plane, while the galactic latitude measures the angle
# of the object above the galactic plane.
type Coordinate (
  longitude: float,
  latitude: float,
  distance: int
)

# Monitor the drive. The method will reply with an update
# whenever the drive's state changes
method Monitor() -> (condition: DriveCondition)

# Calculate the drive's jump parameters from the current
# position to the target position in the galaxy
method CalculateConfiguration(
  current: Coordinate,
  target: Coordinate
) -> (configuration: DriveConfiguration)

# Jump to the calculated point in space
method Jump(configuration: DriveConfiguration) -> ()

# There is not enough tylium to jump with the given
# parameters
error NotEnoughEnergy ()

# The supplied parameters are outside the supported range
error ParameterOutOfRange (field: string)
```

### Keywords

| Keyword     | Description                                                                                                 |
|-------------|-------------------------------------------------------------------------------------------------------------|
| `interface` | The name of the interface in reverse-domain name notation. It MUST be the first keyword in the description. |
| `type`      | A custom type: the definition of an object/record/structure, or of an enumeration.                          |
| `method`    | A method with input and output parameter objects.                                                           |
| `error`     | The name of an error, and an optional data object describing the error.                                     |

Types, methods and error names MUST be unique identifiers within an interface
definition; they share the same namespace. They are referenced by their
fully-qualified reverse-domain name. The method `GetInfo()` in the interface
`org.example.test` is called `org.example.test.GetInfo`.

### Names

* **Interface names** use reverse-domain notation. They consist of at least
  two dot-separated components. Each component starts with a letter (or, for
  all but the first component, a digit), and continues with letters, digits
  and non-leading, non-trailing hyphens. See the grammar below for the exact
  syntax.

* **Type, method and error names** start with an uppercase ASCII letter and
  continue with ASCII letters and digits.

* **Field names** (of structures and enumerations, and of method parameters)
  start with an ASCII letter and continue with ASCII letters and digits. An
  underscore (`_`) may appear between two such characters, but not at the
  beginning or the end of the name, and not doubled.

### Types

With `type` a structure of fields (an object) or an enumeration can be
defined. Following the `type` keyword the type name has to be specified.
Enclosed in `()` the declaration of the various fields follows. Field
declarations specify a field name followed by a colon and a type. Fields are
separated by commas.

```
type MyType (
   example_bool: bool,
   example_int: int,
   example_float: float,
   example_string: string,
   example_object: object,
   example_any: any,
   example_enum: (one, two, three),
   example_struct: (first: int, second: string),
   example_array: []string,
   example_dictionary: [string]string,
   example_stringset: [string](),
   example_nullable: ?string,
   example_nullable_array_struct: ?[](first: int, second: string),
   example_other_type: MyOtherType
)
```

| Type           | Keyword    | Description                                                                                                          | Example                        |
|----------------|------------|----------------------------------------------------------------------------------------------------------------------|--------------------------------|
| Boolean        | `bool`     | `true` or `false`                                                                                                    | `flag: bool`                   |
| Integer        | `int`      | Signed integer, implementation-specific size, usually 64-bit                                                         | `whole: int`                   |
| Floating Point | `float`    | Implementation-specific size, usually IEEE 754 double                                                                | `half: float`                  |
| String         | `string`   | Unicode text                                                                                                         | `name: string`                 |
| Object         | `()`       | Structure of named, typed fields, separated by comma                                                                 | `(first: int, second: string)` |
| Enum           | `()`       | List of names without types, separated by comma                                                                      | `(this, that, other)`          |
| Map/Dictionary | `[string]` | Map of values of one type, keyed by unique strings                                                                   | `[string]int`                  |
| Array          | `[]`       | Ordered list of values of one type                                                                                   | `[]int`                        |
| Nullable       | `?`        | Nullable type ("maybe")                                                                                              | `?string`                      |
| Foreign Object | `object`   | Foreign, untyped/raw JSON object                                                                                     | `payload: object`              |
| Any            | `any`      | Any value: Boolean, Integer, Floating Point, String, (Foreign) Object, Enum, Map/Dictionary, Array, but not Nullable | `value: any`                   |

A *set of strings* can be expressed as a map with an empty structure as value
type, i.e. `[string]()`.

Structures and enumerations may be declared anonymously inline (as in the
example above), or given a name via `type` and referenced by that name.

### Methods

Method names start with an uppercase character and continue with alphanumeric
characters. A method specifies an input and an output parameter object:

```
method Add(a: int, b: int) -> (sum: int)
method Foo(a: int, b: MyType) -> (bar: MyOtherType, baz: float, more: (i: int, f: float, s: string))
```

Both the input and the output parameter list may be empty, expressed as `()`.

### Errors

Error names start with an uppercase character and continue with
alphanumeric characters, and optionally carry parameters describing
the error. The error is identified by its fully-qualified
name. Varlink calls SHOULD only return well-defined errors specified
in the interface description (or in the `org.varlink.service`
interface, see below).

```
error UnknownAction(action: string, more_data: DetailedError)
```

### Documentation Comments

All whitespace (space, `\t`, `\r`, or `\n`) is ignored, except inside
comments, which are started with `#` and extend to the end of the line, and
except that the interface name and each member declaration MUST be followed
by a line break (see the grammar below).

Comments that start on a new line and are immediately followed by
`interface`, `method`, `type` or `error` are documentation for that following
declaration. These comments may span multiple lines. Comments immediately
preceding a field declaration inside a structure, enumeration or parameter
list are documentation for that field.

A single space following the `#` character is conventionally used as
separator and is not part of the documentation text.

Because Varlink interfaces are retrieved as plain text at runtime (see
`org.varlink.service.GetInterfaceDescription` below), the documentation
comments are transmitted to clients together with the interface definition,
and thus available for introspection and for generating user-visible help.

### Grammar

```
interface
        "interface" interface_name members

members
        member
        member members

member
        type_alias
        method
        error

type_alias
        "type" name struct
        "type" name enum

method
        "method" name struct "->" struct

error
        "error" name struct

type
        element_type
        "[]" type
        "[string]" type
        "?" element_type
        "?" "[]" type
        "?" "[string]" type

element_type
        struct
        enum
        name
        "bool"
        "int"
        "float"
        "string"
        "object"
        "any"

struct
        "(" struct_fields ")"
        "(" ")"

struct_fields
        struct_field
        struct_field "," struct_fields

struct_field
        field_name ":" type

enum
        "(" enum_fields ")"

enum_fields
        field_name
        field_name "," enum_fields

interface_name
        [A-Za-z]([-]*[A-Za-z0-9])*(\.[A-Za-z0-9]([-]*[A-Za-z0-9])*)+

name
        [A-Z][A-Za-z0-9]*

field_name
        [A-Za-z](_?[A-Za-z0-9])*
```

### Parsing Expression Grammar

The following is the same grammar expressed as a Parsing Expression Grammar
(PEG), including whitespace and comment handling:

```
whitespace /* Modeled after ECMA-262, 5th ed., 7.2. \v\f removed */
    = [ \t\u{00A0}\u{FEFF}\u{1680}\u{180E}\u{2000}-\u{200A}\u{202F}\u{205F}\u{3000}]

eol_r /* Modeled after ECMA-262, 5th ed., 7.3. */
  = "\n"
  / "\r\n"
  / "\r"
  / "\u{2028}"
  / "\u{2029}"

comment
    = "#" [^\n\r\u{2028}\u{2029}]* eol_r

eol
    = whitespace* eol_r
    / comment

_
    = whitespace / comment / eol_r

field_name
    = [A-Za-z]('_'?[A-Za-z0-9])*

name
    = [A-Z][A-Za-z0-9]*

interface_name
    = [A-Za-z]([-]* [A-Za-z0-9])* ( '.' [A-Za-z0-9]([-]*[A-Za-z0-9])* )+

dict
    = "[string]"

array
    = "[]"

maybe
    = "?"

element_type
    = "bool"
    / "int"
    / "float"
    / "string"
    / "object"
    / "any"
    / name
    / venum
    / vstruct

type
    = element_type
    / maybe element_type
    / array type
    / dict type
    / maybe array type
    / maybe dict type

venum
    = '(' ( _* field_name ** ',' ) _* ')'

argument
    = _* field_name _* ':' _* type

vstruct
    = '(' ( argument ** ',' ) _* ')'

vtypedef
    = "type" _+ name _* vstruct
    / "type" _+ name _* venum

error
    = "error" _+ name _* vstruct

method
    = "method" _+ name _* vstruct _* "->" _* vstruct

member
    = _* m:method
    / _* t:vtypedef
    / _* e:error

interface
    = _* "interface" _+ interface_name eol ( member ++ eol ) _*
```

### Interface Evolution

Varlink interfaces do not have a version number, they only have a feature set
described in detail by the interface definition, which is part of the wire
protocol. Clients can fully introspect a service and figure out which features
the interface supports; this is more expressive than any simple numbering
scheme.

Varlink does not use positional parameters or fixed-size objects in its
interface definition or on the wire: all parameters are identified by their
name and can be extended later. Interfaces SHOULD be extended in a backwards
compatible way:

* New fields SHOULD be added as nullable (`?`) fields, both in structures and
  in method input/output parameters. The result of a method, when passed
  parameters without the new optional fields, SHOULD be the same as in older
  versions. The expected behavior for omitted fields SHOULD be documented.

* New methods, types and errors may be added. New enumeration values may be
  added to enumerations used in method input parameters; adding values to
  enumerations that appear in replies is only backwards compatible if clients
  are prepared to handle unknown values, which SHOULD be documented.

* Removing or incompatibly changing fields, types, errors, enumeration values
  or methods is not backwards compatible and SHOULD be avoided. If an
  interface must be changed incompatibly, it SHOULD be released under a new
  name, commonly by adding a number to the interface name, e.g.
  `org.example.interface2`.

Services SHOULD refuse unrecognized fields in method calls with an
`org.varlink.service.InvalidParameter` error sent back to the client.

Clients SHOULD ignore fields in replies they do not understand, so that
services can extend their output.

## Mapping the Type System to JSON

Varlink values are encoded as JSON ([RFC 8259](https://www.rfc-editor.org/rfc/rfc8259)).
The mapping is as follows:

| Varlink type | JSON encoding                                                                    |
|--------------|----------------------------------------------------------------------------------|
| `bool`       | JSON `true` or `false`                                                           |
| `int`        | JSON number without fractional part or exponent                                  |
| `float`      | JSON number                                                                      |
| `string`     | JSON string                                                                      |
| structure    | JSON object; each field a name/value pair                                        |
| enumeration  | JSON string containing the enumeration value's name                              |
| `[string]T`  | JSON object; each map element a name/value pair                                  |
| `[]T`        | JSON array                                                                       |
| `?T`         | Either the encoding of `T`, or JSON `null`, or the field is absent (see below)   |
| `object`     | Any JSON object                                                                  |
| `any`        | Any JSON value: `true`, `false`, numbers, strings, object, array, but not `null` |

### Integers

Like JSON, Varlink is intentionally underspecified regarding integer
sizes and does not distinguish signed and unsigned integer
types. Implementations SHOULD support signed integers, and return an
error if they cannot handle a received number.

Implementations SHOULD support at least the range of signed 64-bit
integers, i.e. −2⁶³ … 2⁶³−1, when generating and parsing
messages. Support for the unsigned 64-bit integer range is RECOMMENDED
(i.e. supporting the combined signed/unsigned integer range of −2⁶³ …
2⁶⁴−1). Interfaces that need to transport values outside of this
range, or environments that cannot represent this range natively
(e.g. JavaScript's IEEE 754 `Number`) MAY choose to transport such
values encoded as `string` instead. Implementations MAY be lenient and
accept a JSON string containing a formatted decimal integer where an
`int` is expected.

### Floating Point Numbers

Floating point numbers SHOULD be treated as IEEE 754 double precision (64-bit)
values. Note that JSON cannot represent the special values NaN, +Inf and −Inf,
hence these MUST NOT be generated. Interfaces that need to transport such
values SHOULD encode them differently, e.g. as `string`, or as a nullable
`float` with the special values mapped to `null`.

### Nullable Fields and Absent Fields

For a nullable field (`?T`), the following two encodings MUST be treated as
equivalent by receivers: the field being absent from the enclosing JSON
object, and the field being present with the value `null`. Note that for
fields of type `?[]T` or `?[string]T` an empty array or object respectively is
*not* equivalent to `null`: an empty collection is a valid non-null value.

Senders MAY omit fields whose value is `null` rather than encoding them
explicitly.

A `null` value (or an absent field) generally means "use the default"
or "does not apply", where the meaning of "default" is specific to the
interface and SHOULD be documented.

Non-nullable fields must be present, and must not be `null`.

### Unknown Fields

A service SHOULD reject method call parameters containing fields that are not
declared in the interface with the `org.varlink.service.InvalidParameter`
error, and SHOULD equally reject values that do not match the declared types.
Clients SHOULD ignore undeclared fields in replies (see "Interface Evolution"
above).

## Wire Protocol

All messages are encoded as JSON objects. Over stream transports, each
message is terminated with a single `NUL` byte (`0x00`). Conceptually the
`NUL` byte belongs to the transport, not to the Varlink message: if Varlink
messages are encapsulated in a transport that provides its own framing, such
as HTTP, that transport's framing is used instead (see below). Since a valid
JSON document never contains a raw `NUL` byte (inside strings it has to be
escaped as `\u0000`), the terminator is unambiguous.

Messages MUST be encoded in UTF-8.

A service responds to requests in the same order that they are received —
messages are never multiplexed, and there are no sequence numbers in calls and
replies. However, multiple requests can be queued on a connection to enable
pipelining. This simplifies and minimizes the amount of state clients need to
track. Complex interfaces can use multiple connections at the same time.

The common case is a simple method call with a single reply. To support
monitoring calls, subscriptions, chunked data and streaming, calls may carry
instructions for the server to not reply, or to reply multiple times to a
single method call.

### Call

Call and reply messages are objects with properties, conceptually similar to
HTTP headers. The `parameters` member is the payload.

A call is defined as:

```
(
  method: string,
  parameters: ?object,
  oneway: ?bool,
  more: ?bool,
  upgrade: ?bool
)
```

| Member       | Description                                                                        |
|--------------|------------------------------------------------------------------------------------|
| `method`     | The fully-qualified method name, i.e. *interface*.*method*. REQUIRED.              |
| `parameters` | The input parameters, an object matching the method's input parameter declaration. |
| `oneway`     | Instructs the server to suppress its reply. See below.                             |
| `more`       | Requests possibly multiple replies to the same call. See below.                    |
| `upgrade`    | Requests the connection to be taken over by a custom protocol/payload. See below.  |

At most one of `oneway`, `more` and `upgrade` may be set to `true` in a single
call. A server MUST reject calls that set more than one of them, or that set
any of them to a non-boolean value, by closing the connection (there is no
well-defined way to reply to such a call).

The `parameters` member is optional: if the method takes no input parameters,
or all of them are nullable and shall be omitted, the member may be absent,
`null` or `{}`. All three encodings MUST be treated as equivalent by servers.
Senders SHOULD omit the member rather than sending `null` or `{}`.

### Reply

A reply is defined as:

```
(
  parameters: ?object,
  continues: ?bool,
  error: ?string
)
```

| Member       | Description                                                                                                       |
|--------------|-------------------------------------------------------------------------------------------------------------------|
| `parameters` | The output parameters of the reply, or the parameters of the error.                                               |
| `continues`  | Instructs the client to expect further replies to the same call. Only valid if the call requested it with `more`. |
| `error`      | The fully-qualified reverse-domain error name. Its presence indicates that the method call has failed.            |

As for calls, an absent `parameters` member, `null`, and `{}` MUST be treated
as equivalent by clients, and servers SHOULD omit the member when there are no
parameters to transmit.

`continues` MUST NOT be set in an error reply: an error reply always
terminates the call it belongs to.

### Simple Method Call

Requests specify the fully-qualified `method` that should be called, along
with its input parameters:

```json
{
  "method": "org.example.ftl.CalculateConfiguration",
  "parameters": {
    "current": {
      "longitude": 27.13,
      "latitude": -12.4,
      "distance": 48732498234
    },
    "target": {
      "longitude": -48.7,
      "latitude": 12.9,
      "distance": 354667658787
    }
  }
}
```

A service replies with an object that contains the output `parameters`:

```json
{
  "parameters": {
    "configuration": {
      "speed": 32434234,
      "trajectory": 686787,
      "duration": 13256445
    }
  }
}
```

Errors contain the fully-qualified error name and optional `parameters` as
specified and documented in the Varlink interface file:

```json
{
  "error": "org.example.ftl.ParameterOutOfRange",
  "parameters": {
    "field": "current.distance"
  }
}
```

### `oneway`

Setting `oneway` to `true` in a call instructs the server to not send any
reply, neither a success reply nor an error. The server MUST adhere to the
instruction, to allow clients to associate the next reply with the next call
issued without `oneway`.

This is useful when many messages are sent as part of one transaction, and
only one single reply is needed to confirm the transaction. It might also be
useful for non-critical data such as a debugging interface, to minimize
round-trip handling. Note that the client will not learn about any failure of
a `oneway` call, including invocation of a non-existent method.

Because of the strict ordering requirement of the protocol, all servers MUST
implement `oneway`, even if none of their methods have a specific use for it.

### `more`

Setting `more` to `true` in a call requests possibly multiple replies to the
same call. Support for `more` is optional on the server side and per method:

* If the server does not support `more` for the method, it replies with a
  single reply without `continues`, i.e. the call behaves like a simple method
  call and the connection is free to be used for the next call.

* If the server does support it, it sends zero or more replies with
  `continues` set to `true`, followed by exactly one final reply without
  `continues` (or with it set to `false`), or by an error reply. While replies
  with `continues` are being sent, the connection is busy: the server MUST NOT
  process further calls on the connection until the final reply has been sent.
  (Clients may still enqueue further calls, in the sense of pipelining.)

* A client which is no longer interested in the replies from the server
  simply closes the connection.

Servers MUST NOT set `continues` in replies to calls that did not set `more`.

Some methods only make sense with `more` set (for instance, methods that
enumerate an unbounded set of objects, one reply per object). Servers SHOULD
reply with the `org.varlink.service.ExpectedMore` error if such methods are
called without `more`.

Varlink connections can thus be turned into monitor connections, which are
soft-negotiated between the client and the service. To avoid race conditions
between the reading of the current state and subscribing to updates, a typical
Varlink monitor method call first returns the current state and then sends
changes in subsequent replies to the same request on the same connection.

The IDL definition of a method that supports `more` MUST carry the combination
of all possible replies as reply message definition, and MUST set `?` on all
fields that may be absent in at least one valid reply of the series.

### `upgrade`

Setting `upgrade` to `true` in a call requests the connection to be taken over
by a custom protocol/payload. The server confirms the upgrade by sending a
single reply message (which may carry parameters). After this reply the
connection is no longer controlled by Varlink, and both sides may exchange
arbitrary data on it. To speak the Varlink protocol to the service again, the
client needs to open a new connection.

If the server replies with an error, the connection remains a Varlink
connection.

Servers MUST make sure that they do not consume any bytes from the connection
beyond the `NUL` byte terminating the `upgrade` call before confirming the
upgrade, since any such bytes belong to the upgraded protocol. Clients
likewise MUST NOT consume bytes beyond the terminator of the confirming reply.
Clients SHOULD NOT send any data belonging to the upgraded protocol before
having received the confirmation.

This mechanism is useful for transferring larger amounts of data, or foreign
data that is not reasonably encoded in JSON. It is conceptually similar to
the WebSocket upgrade in HTTP.

A server SHOULD reply with the `org.varlink.service.ExpectedUpgrade`
error to a call of a method that requires the flag but did not set it.

### Errors

Varlink errors are interface-specific and identified by a fully-qualified
string. There are no error numbers and no mandatory human-readable messages.
If required, interfaces MAY add a `message` field to the error parameters
which contains human readable text. The primary focus of Varlink errors is
machine consumption: in most cases a carefully chosen descriptive CamelCase
error name sufficiently describes the error to humans as well.

Errors of the interface a method belongs to, and the generic errors defined in
`org.varlink.service` (see below) may be returned by any method.

### Vendor Extensions

The message headers (i.e. the members of call and reply objects besides
`parameters`) MAY be extended by vendors if needed. Added members MUST be
nested in an object named after the vendor in reverse-domain notation, e.g.
`"org.example": { … }`. Such names always contain at least one `.`
character, which distinguishes them from the members defined by this
specification. Vendors MUST NOT add unprefixed keys or keys with an
`X-`-style prefix, since they may clash with other vendors or future Varlink
protocol specifications.

Implementations MUST reject messages with unknown unprefixed members by
closing the connection, and SHOULD ignore vendor extension objects they do
not understand.

### Connection Lifecycle

A connection is established by the client. Either side may close the
connection at any time. When a client closes a connection while a call is
being processed, the server SHOULD abort the operation if possible, and MUST
discard any pending replies.

Servers MUST close connections on which they receive malformed messages
(invalid JSON, non-object top-level values, missing `method`, unknown
unprefixed header fields, non-boolean flags), since there is no well-defined
reply for such messages.

Servers MAY enforce limits on message sizes and on the number of concurrent
connections, and SHOULD close connections that exceed them.

## The `org.varlink.service` Interface

Every Varlink service SHOULD implement the `org.varlink.service`
interface. It allows clients to retrieve a description of all the
interfaces a service implements, and information which describes the
service implementation. The errors defined in this interface are
generic errors that may be returned by any method of any interface.

```
# The Varlink Service Interface should be provided by every varlink service.
# It describes the service and the interfaces it implements.
interface org.varlink.service

# Get a list of all the interfaces a service provides and information
# about the implementation.
method GetInfo() -> (
  vendor: string,
  product: string,
  version: string,
  url: string,
  interfaces: []string
)

# Get the description of an interface that is implemented by this service.
method GetInterfaceDescription(interface: string) -> (description: string)

# The requested interface was not found.
error InterfaceNotFound (interface: string)

# The requested method was not found
error MethodNotFound (method: string)

# The interface defines the requested method, but the service does not
# implement it.
error MethodNotImplemented (method: string)

# One of the passed parameters is invalid.
error InvalidParameter (parameter: string)

# Client is denied access
error PermissionDenied ()

# Method is expected to be called with 'more' set to true, but wasn't
error ExpectedMore ()

# Method is expected to be called with 'upgrade' set to true, but wasn't
error ExpectedUpgrade ()
```

### Methods

* `GetInfo()` returns information about the service implementation
  (`vendor`, `product`, `version`, `url`), and the list of interfaces it
  implements in `interfaces`. The list MUST include `org.varlink.service`
  itself.

* `GetInterfaceDescription(interface)` returns the interface definition of
  one interface, in the interface definition language described above,
  including all documentation comments, as a single string. If the requested
  interface is not implemented, `InterfaceNotFound` is returned.

Example service information:

| Field     | Value                                    |
|-----------|------------------------------------------|
| `vendor`  | `Fedora`                                 |
| `product` | `Workstation`                            |
| `version` | `27`                                     |
| `url`     | `https://getfedora.org/en/workstation/`  |

| Field     | Value                                    |
|-----------|------------------------------------------|
| `vendor`  | `Jon Doe`                                |
| `product` | `Fidget Service`                         |
| `version` | `0.2`                                    |
| `url`     | `https://example.com/jon/fidget`         |

### Errors

| Error                  | Meaning                                                                                                          |
|------------------------|------------------------------------------------------------------------------------------------------------------|
| `InterfaceNotFound`    | A method was called on an interface the service does not implement. `interface` names it.                        |
| `MethodNotFound`       | A method was called that the (known) interface does not define. `method` names it.                               |
| `MethodNotImplemented` | The interface defines the method, but this service does not implement it. `method` names it.                     |
| `InvalidParameter`     | One of the passed parameters is invalid (missing, wrong type, unknown, or out of range). `parameter` names it.    |
| `PermissionDenied`     | The client is denied access to the method or resource.                                                           |
| `ExpectedMore`         | The method has to be called with the `more` flag set, but was not.                                               |
| `ExpectedUpgrade`      | The method has to be called with the `upgrade` flag set, but was not.                                            |

The `ExpectedUpgrade` error is an addition to the original `org.varlink.service`
interface as published on varlink.org, mirroring `ExpectedMore`.

## Service Addresses

Varlink services are accessible through addresses, typically
referencing an entrypoint network socket address. They are expressed in
URI-like notation. All properties after a `;` character SHOULD be
ignored by implementations that do not understand them, to allow
future extensions.

| Type                            | Example                          | Comment                                              |
|---------------------------------|----------------------------------|------------------------------------------------------|
| UNIX socket                     | `unix:/run/org.example.ftl`      | Path of an `AF_UNIX` socket inode in the file system |
| UNIX abstract namespace socket  | `unix:@org.example.ftl`          | Linux abstract socket namespace                      |
| TCP (with hostname)             | `tcp:localhost:12345`            | Hostname and port                                    |
| TCP (with IPv4 address)         | `tcp:127.0.0.1:12345`            | IPv4 address and port                                |
| TCP (with IPv6 address)         | `tcp:[::1]:12345`                | IPv6 address and port                                |

Support for UNIX sockets as transport is RECOMMENDED, support for TCP is OPTIONAL.

Transports beyond these (such as HTTP, see below) may define their own address
schemes.

## Socket Activation

A listening or connection socket MAY be passed to a Varlink service
program at startup, following the [systemd](https://systemd.io/)
socket activation protocol (see
[`sd_listen_fds(3)`](https://www.freedesktop.org/software/systemd/man/latest/sd_listen_fds.html)):

| Environment variable | Example                        | Comment                                                                     |
|----------------------|--------------------------------|-----------------------------------------------------------------------------|
| `LISTEN_FDS`         | `LISTEN_FDS=2`                 | Number of file descriptors passed to the service, starting at file descriptor 3. If more than one file descriptor is passed, `LISTEN_FDNAMES` identifies the Varlink file descriptor. |
| `LISTEN_PID`         | `LISTEN_PID=4711`              | The process ID of the started service. If this does not match the PID of the receiving service, the variables MUST be ignored. |
| `LISTEN_PIDFDID`     | `LISTEN_PIDFDID=6783345`       | The inode number of the pidfd referring to the process of the started service (specific to modern Linux). If set and this does not match the pidfd inode ID of the receiving service, the variables MUST be ignored. |
| `LISTEN_FDNAMES`     | `LISTEN_FDNAMES=other:varlink` | A colon separated list of names of the passed file descriptors. The Varlink listening socket MUST be named `varlink`. |

Because Varlink is strictly point-to-point and there is no buffering
component involved, services can implement exit-on-idle in a race-free manner:
when there is no background task running and no clients connected, the
service can close the listening socket and exit; the service activator queues
incoming connections and starts the service again as needed.

All sockets passed in via socket activation SHOULD be of type
`SOCK_STREAM`, and either be a listening socket (i.e. a socket
`listen()` has been called on) or an established connection socket (i.e. a socket
`connect()` has been invoked on or a socket acquired via `accept()`
or `socketpair()` or equivalent calls).

## Transport: `AF_UNIX` Stream Sockets

The primary transport for Varlink on Linux is `AF_UNIX` sockets of type
`SOCK_STREAM`. This section specifies how the protocol is bound to this
transport.

### Framing

Each Varlink message is written to the socket as its UTF-8 encoded JSON text,
followed by a single `NUL` byte. Implementations MUST NOT assume that a single
`read()` returns exactly one message: messages may be split across multiple
reads and multiple messages may be returned by a single read. Implementations
MUST hence buffer and search for the `NUL` terminator.

Implementations SHOULD NOT emit whitespace (including newlines) outside of
JSON strings, but MUST accept it, in accordance with the JSON grammar.

### Listening Sockets and Access Control

Services SHOULD listen on a socket bound to a path in the file system, which
is the *entrypoint* clients connect to. Sockets in the Linux abstract namespace
(`unix:@…`) MAY be used, but note that they do not offer file-system based
access control, and hence are not recommended.

The most commonly used method to restrict access to a service is to rely on
the permissions of the socket inode in the file system: `connect()` on an
`AF_UNIX` socket is subject to a write access check on the socket inode. It is
RECOMMENDED to grant read access on the socket inode even when write access is
restricted (e.g. a mode of `0644` rather than `0600`), since this allows
unprivileged clients to read the extended attributes identifying the socket as
a Varlink socket (see below) without granting them the ability to connect.

### Peer Authentication

Services which need more fine-grained access control than the
file-system permissions of the entrypoint socket inode SHOULD check
the credentials of the connecting peer and decide per call whether to
act on behalf of the client.  On Linux, the following facilities
exist:

* `SO_PEERCRED` (via `getsockopt(2)`) returns the UID, GID and PID of the peer
  process at the time of `connect()`, resp. `listen()`. This is the
  RECOMMENDED mechanism: it is available for every connection without
  cooperation of the client, and cannot be spoofed.

* `SO_PEERPIDFD` (available since Linux 6.5) returns a pidfd referring to the
  peer process at the time of connection. Services that need to act on the
  peer *process* (rather than just its UID) SHOULD use this instead of the PID
  from `SO_PEERCRED`, since the PID may be recycled.

The peer credentials are properties of the connection, not of individual
messages. Services MAY acquire them once when a connection is accepted.

Note that the credentials identify the process that established the
connection, which is not necessarily the process that sends the
messages if the socket is passed on to other processes. Services that
consider this a concern SHOULD document that their sockets are not to
be shared. Also note that when a connection has been bridged (e.g. via
the HTTP transport below), the credentials identify the bridge, not
the original client.

### Authorization

Beyond the mechanisms above, services MAY require interactive or policy-based
authorization for individual method calls (e.g. via `polkit`). Services that
require interactive authorization but are unable to perform it in the
context of the current call SHOULD indicate this with a dedicated error.

### Passing File Descriptors

`AF_UNIX` sockets allow passing file descriptors between processes via
`SCM_RIGHTS` ancillary messages. The Varlink protocol itself is intentionally
free of such transport-specific side channels: it is one of its design goals
to be remotable and encapsulatable in other protocols, and file descriptors
cannot be forwarded across such encapsulations. Interfaces SHOULD hence
implement their functionality purely in-band, i.e. via JSON-encoded
parameters, wherever that is feasible. For example, rather than pointing a
service to a file local to the client, the contents of the file SHOULD be
transmitted.

Interfaces MAY nevertheless make use of file descriptor passing where a purely
in-band implementation is not possible, or would be unreasonably inefficient
or insecure. Typical cases are the transfer of large amounts of data via
pipes or memory-file descriptors, handing over kernel objects that cannot be
expressed in JSON (sockets, pidfds, mount file descriptors, TTYs), or
delegating access to a resource without disclosing its location. In such
cases the following rules apply:

* File descriptors are passed in `SCM_RIGHTS` ancillary messages
  attached to the `sendmsg(2)` call that transmits the Varlink message
  they belong to.  They are logically associated with that message,
  and referenced from its parameters by their index (starting at 0) in
  the ancillary data of the message. An interface SHOULD document
  which of its methods send and receive file descriptors, and which
  parameter fields carry indices referencing them.

* The file descriptors MUST be sent with the first byte of the message they
  belong to. Receivers MUST associate the file descriptors received with a
  chunk of data with the message that chunk begins, and MUST NOT accept file
  descriptors that arrive with a chunk that does not start a new message.

* Receiving file descriptors is opt-in. Services MUST NOT accept file
  descriptors on connections unless they implement an interface that makes
  use of them, and SHOULD turn off reception of file descriptors on the
  listening socket (via the `SO_PASSRIGHTS` socket option, available since
  Linux 6.16, where available) otherwise, so that unwanted file descriptors
  are not queued. Clients likewise MUST NOT accept file descriptors unless
  they invoke a method that returns them.

* Methods that consume file descriptors MUST be robust against receiving a
  different number of file descriptors than expected, and SHOULD reply with
  `org.varlink.service.InvalidParameter` in that case.

* Interfaces using file descriptor passing SHOULD document what
  happens when they are used over a transport that does not support it
  (typically: the relevant methods fail with a suitable
  error). Bridges encapsulating the `AF_UNIX` transport in another one
  SHOULD refuse or drop file descriptors and MUST NOT silently forward
  messages whose parameters reference file descriptors that were not
  forwarded.

### Protocol Upgrade

After an `upgrade` call has been confirmed (see above), both sides of the
`AF_UNIX` connection continue using the same socket for the upgraded
protocol. The upgraded protocol may make use of file descriptor passing and
other socket features, subject to the same opt-in rules as above.

### Recognizing Varlink Socket Inodes

On kernels that support extended attributes on socket inodes (available since
Linux 7.0), implementations SHOULD tag the `AF_UNIX` socket inodes they create
and manage with the `user.varlink` extended attribute. The attribute records
the *role* the socket plays in the Varlink communication, allowing other
processes to discover and classify Varlink sockets, e.g. to enumerate all
Varlink services available on a system, or to tell Varlink connections apart
from other `AF_UNIX` connections when inspecting a process.

Four distinct roles, i.e. attribute values, are defined:

| Value        | Meaning                                                                                          |
|--------------|--------------------------------------------------------------------------------------------------|
| `client`     | Set on the connecting end of a connection, i.e. a socket created via `socket()` + `connect()`. Only for inodes on the anonymous socket file system (`sockfs`). |
| `server`     | Set on the serving end of a connection, i.e. a socket obtained via `accept()` on a listening socket. Only for inodes on the anonymous socket file system (`sockfs`). |
| `listen`     | Set on a listening socket, i.e. a socket created via `socket()` + `listen()`. Only for inodes on the anonymous socket file system (`sockfs`). |
| `entrypoint` | Set on the entrypoint socket inode, i.e. the socket node bound into the regular file system that clients connect to. Unlike the other three, this attribute lives on a real file-system inode (not on `sockfs`) and is hence visible to any process that can resolve the socket path and has read access to it. |

Note that a path-bound listening socket thus carries two distinct
attributes: `listen` on the `sockfs` inode (visible e.g. via
`/proc/PID/fd/`), and `entrypoint` on the file-system inode (visible
via the socket's path as encoded in the `.sun_path` field of `struct
sockaddr_un`).

The attribute SHOULD be set by the code that creates the respective socket,
i.e. the client for `client`, and the service for `listen`, `server` and
`entrypoint`. Service managers that create listening sockets on behalf of
services (socket activation) SHOULD set `listen` and `entrypoint` (and, when
accepting connections on behalf of the service, `server`) themselves.

Setting the attribute SHOULD be treated as best-effort: failure to set
it (e.g.  because the kernel or file system does not support it)
SHOULD NOT be considered an error. Consumers of the attribute MUST
tolerate its absence.

Consumers SHOULD NOT treat a socket inode as a Varlink socket merely because
some `user.varlink` attribute is set; the value SHOULD match one of the roles
above. Unknown values SHOULD be ignored, to allow the definition of further
roles in the future. Also note that the attribute is informational: since
`user.*` extended attributes can be set by any process with write access to
the inode, its presence is not a security guarantee, and a socket without
the attribute may still implement Varlink.

In combination with the access mode recommendation above (i.e. `0644` rather
than `0600` for sockets that shall not be world-accessible), this allows
unprivileged processes to enumerate the Varlink services on a system even if
they cannot connect to them.

For socket-activated Varlink services managed by a service manager,
the extended attributes SHOULD be set by the service manager when
allocating them, and not be delayed until the service they are
associated with ultimately gets activated.

## Transport: HTTP

Varlink can be encapsulated in HTTP, in order to make services
available remotely, in particular to environments where the `AF_UNIX`
transport is not available, such as web browsers or hosts of virtual
machines. This section specifies such an encapsulation. It is
typically implemented by a bridge that accepts HTTP connections and
forwards the calls to local services over their `AF_UNIX` sockets, but
MAY also be implemented directly by a service.

Two modes are defined: a *simple HTTP mode* in which each HTTP request
maps to one Varlink method call, and a *WebSocket mode* in which the
Varlink wire protocol is tunneled verbatim.

The bridge serves a set of Varlink services, each identified by their
name.

### URL Schema

All URLs defined here shall be understood relative to a base URL used
as service address.

| Method | Path                               | Description                                                                                                     |
|--------|------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| `POST` | `…/call/{method}`                  | Invoke a method. The service is derived from the interface prefix of the method name, or given via `?service=`. |
| `POST` | `…/call/{service}/{method}`        | Invoke a method on an explicitly given service.                                                                 |
| `GET`  | `…/services`                       | List the available services.                                                                                    |
| `GET`  | `…/services/{service}`             | Service information of one service (as returned by `org.varlink.service.GetInfo`).                              |
| `GET`  | `…/services/{service}/{interface}` | Method names of one interface implemented by a service.                                                         |
| `GET`  | `…/ws/services/{service}`          | WebSocket endpoint tunneling the Varlink wire protocol to a service.                                            |
| `GET`  | `…/health`                         | Health check, returns `200 OK` if the bridge is operational.                                                    |

(The `…` prefix in the table above shall be replaced by the base URL.)

All request and response bodies in the simple mode are JSON documents
with the `application/json` media type, unless streaming mode is used,
in which case `application/json-seq` is used. Every request MUST be
made with an `Accept: application/json` header, except if a streaming
reply is expected, in which case `Accept: application/json-seq`
should be sent. A `charset=` parameter SHOULD not be added.

### Simple HTTP Mode

In simple HTTP mode a method is invoked by a `POST` request to
`…/call/{method}` or `…/call/{service}/{method}`, where `{method}` is the
fully-qualified method name. The request body is a JSON object containing the
method's input parameters — i.e. the value of the `parameters` member of a
Varlink call, *not* the call object itself. An empty object `{}` is sent if
there are no parameters. Bridges SHOULD accept an empty request body as
equivalent.

For `…/call/{method}`, the service is derived from the method name
extracting the interface, i.e. by stripping the last component,
e.g. `io.systemd.Hostname.Describe` is invoked on the service
`io.systemd.Hostname`. The `?service=` query parameter overrides this
for cross-interface calls, e.g. to call `org.varlink.service.GetInfo`
on the `io.systemd.Hostname` service. The `…/call/{service}/{method}`
form makes the service explicit.

The bridge issues the call over a Varlink connection to the service (the
connection MAY be reused for subsequent requests, subject to the ordering
rules of the protocol) and maps the reply to the HTTP response as follows:

* **Success.** Status `200 OK`, with the reply's output parameters as JSON
  body — again the value of the `parameters` member rather than the reply
  object, i.e. the reply is *unwrapped*. An absent `parameters` member is
  returned as `{}`.

* **Varlink error.** A reply carrying an `error` member is mapped to an HTTP
  status code, with a JSON body of the form
  `{"error": "<fully-qualified error name>", "parameters": {…}}`:

  | Varlink error                              | HTTP status                 |
  |--------------------------------------------|-----------------------------|
  | `org.varlink.service.InvalidParameter`     | `400 Bad Request`           |
  | `org.varlink.service.ExpectedMore`         | `400 Bad Request`           |
  | `org.varlink.service.ExpectedUpgrade`      | `400 Bad Request`           |
  | `org.varlink.service.InterfaceNotFound`    | `404 Not Found`             |
  | `org.varlink.service.MethodNotFound`       | `404 Not Found`             |
  | `org.varlink.service.MethodNotImplemented` | `501 Not Implemented`       |
  | `org.varlink.service.PermissionDenied`     | `403 Forbidden`             |
  | Any other error                            | `500 Internal Server Error` |

* **Transport failure.** If the bridge cannot connect to the service's
  socket, or the connection fails while the call is in progress,
  status `502 Bad Gateway` is returned. Malformed requests (invalid
  JSON body, unknown service) yield `400 Bad Request` or `404 Not
  Found` as appropriate. Requests that fail authentication yield `401
  Unauthorized`.

  In these cases, the body is a JSON object of the form
  `{"message": "<human readable message>"}`.

Note that HTTP status codes only carry the broad classification; clients
SHOULD consult the `error` member of the body for the actual error name.

Example:

```console
$ curl -s -X POST https://host:1031/waldo/call/io.systemd.Hostname.Describe \
    -H "Content-Type: application/json" -d '{}'
{"Hostname":"myhost","StaticHostname":"myhost",…}
```

(The base URL is `https://host:1031/waldo/` in this example.)

#### Streaming Replies

Methods that reply multiple times (i.e. use the `more` flag) are invoked in
simple mode by sending the request header `Accept: application/json-seq`. The
bridge then sets `more` on the Varlink call and streams the replies to the
client as a JSON text sequence
([RFC 7464](https://www.rfc-editor.org/rfc/rfc7464)), with the response media
type `application/json-seq`. Each record consists of a Record Separator
character (`0x1E`), followed by the JSON text, followed by a Line Feed
(`0x0A`).

Each successful reply is encoded as one record containing its output
parameters (unwrapped, as above). The `continues` flag is not transmitted: the
sequence ends after the final reply, i.e. when the HTTP response body ends.
An error reply terminates the sequence with a record of the form
`{"error": "<fully-qualified error name>", "parameters": {…}}`; a transport
failure during streaming terminates it with a record of the form
`{"error": "<message>"}`. Since the HTTP status line has already been sent
when such an error occurs, the status is `200 OK` regardless.

The interface definition at this time carries no indication whether a
method supports or requires `more`, hence the bridge cannot determine
this up front. If a method that requires `more` is invoked without the
`application/json-seq` accept header, the service's `ExpectedMore`
error is passed through as `400 Bad Request`.

Example:

```console
$ curl -s -H "Accept: application/json-seq" -H "Content-Type: application/json" \
    https://host:1031/waldo/call/io.systemd.UserDatabase.GetUserRecord \
    -d '{"service":"io.systemd.Multiplexer"}' | jq --seq
```

#### Limitations

Simple HTTP mode maps one HTTP request to one Varlink call. It
therefore provides no way to issue `oneway` calls, to upgrade the
connection with the `upgrade` flag or to pipeline multiple calls on
one Varlink connection. Clients that need these features should use
WebSocket mode.

#### Introspection

* `GET …/services` returns `{"services": ["<name>", …]}`, the list of service
  names available via the bridge.

* `GET …/services/{service}` returns the output parameters of
  `org.varlink.service.GetInfo` as invoked on that service, i.e. an object with
  the `vendor`, `product`, `version`, `url` and `interfaces` members.

* `GET …/services/{service}/{interface}` returns
  `{"methods": ["<method>", …]}`, the unqualified names of the methods of
  that interface as declared in the interface definition retrieved via
  `org.varlink.service.GetInterfaceDescription`.

### WebSocket Mode

In WebSocket mode the client performs an HTTP upgrade to the WebSocket
protocol ([RFC 6455](https://www.rfc-editor.org/rfc/rfc6455), [RFC
8441](https://www.rfc-editor.org/rfc/rfc8441), [RFC
9220](https://www.rfc-editor.org/rfc/rfc9220)) on
`…/ws/services/{service}`. On success the bridge connects to the
service's socket, and from then on acts as a transparent byte bridge
between the WebSocket and the Varlink connection, in both
directions. The client speaks the raw Varlink wire protocol, exactly
as it would over an `AF_UNIX` socket: JSON call objects terminated by
`NUL`, receiving JSON reply objects terminated by `NUL`.  The full
protocol is available, including `oneway`, `more` and `upgrade`, as
well as pipelining.

Framing rules:

* Data from the client to the service is taken from WebSocket binary frames.
  Text frames MUST also be accepted, and treated identically. The bridge MUST
  NOT assume that one frame contains exactly one message: frames are
  concatenated into the byte stream as they are received.

* Data from the service to the client is sent in WebSocket binary frames. The
  bridge SHOULD send each Varlink message, *including* its terminating `NUL`
  byte, as one frame, i.e. re-frame the byte stream at message boundaries.
  Clients MUST nevertheless not rely on this, and MUST reassemble messages
  based on the `NUL` terminator only.

* Once an `upgrade` call has been detected on the connection and confirmed by
  the service, the bridge MUST stop re-framing and forward bytes in both
  directions verbatim, in frames of arbitrary size, since it can no longer
  make any assumptions about the payload.

* Ping and Pong control frames are handled at the WebSocket layer and never
  forwarded. When either side closes its connection, the bridge closes the
  other one; a WebSocket Close frame is answered with the closing handshake.

* If the bridge cannot connect to the service's socket, the HTTP
  upgrade request fails with `502 Bad Gateway`, and no WebSocket
  connection is established.

Since neither WebSockets nor HTTP can carry file descriptors, methods relying
on file descriptor passing are not available in this mode; the `SO_PASSRIGHTS`
rules above apply to the bridge's `AF_UNIX` connection, which MUST NOT accept
file descriptors from the service.

Example, using a generic WebSocket client:

```console
$ printf '{"method":"io.systemd.Hostname.Describe"}\0' | \
    websocat --binary wss://host:1031/waldo/ws/services/io.systemd.Hostname | tr -d '\0' | jq
{
  "parameters": {
    "Hostname": "myhost",
    …
  }
}
```

WebSocket mode makes the bridge usable as a *fully-featured* Varlink
bridge: a local Varlink client can connect to a helper program that
establishes the WebSocket and relays the byte stream, so that remote
services appear like local ones.  Address URIs of the form
`ws://host:port/something/ws/services/{service}` and
`wss://host:port/something/ws/services/{service}` (and `http://`/`https://` with
the same path, implying the WebSocket upgrade) may be used to refer to
such remote services.

### Authentication and Transport Security

A Varlink HTTP bridge typically exposes privileged services over the network.
Deployments SHOULD hence use TLS (i.e. `https://`/`wss://`) and SHOULD
authenticate clients, unless the network is trusted by construction. One
mechanism is currently defined:

* **Mutual TLS.** The bridge is configured with a CA certificate, and requires
  clients to present a certificate signed by it. Without a configured server
  certificate the bridge MAY generate a self-signed one, in which case clients
  SHOULD pin the server's public key on first contact (in the manner of SSH
  `known_hosts`), and refuse connections if it later changes.

Authentication happens at the HTTP layer, on the bridge. The Varlink services
behind the bridge see the bridge's peer credentials (see the `AF_UNIX`
transport section), not the remote client's. Any per-client authorization the
services implement based on peer credentials therefore applies to the bridge,
and the bridge's own authorization policy has to be relied on instead.

## References

* [Varlink](https://varlink.org/), the original project documentation:
  [Interface Definition](https://varlink.org/Interface-Definition),
  [Method Call](https://varlink.org/Method-Call),
  [Service](https://varlink.org/Service),
  [Ideals](https://varlink.org/Ideals),
  [FAQ](https://varlink.org/FAQ)
* [systemd `sd-varlink(3)`](https://www.freedesktop.org/software/systemd/man/latest/sd-varlink.html),
  a Varlink implementation for C, and
  [`varlinkctl(1)`](https://www.freedesktop.org/software/systemd/man/latest/varlinkctl.html),
  a command line tool for introspecting and invoking Varlink services
* [varlink-http-bridge](https://github.com/systemd/varlink-http-bridge),
  systemd's Varlink HTTP bridge
* [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259), The JavaScript Object
  Notation (JSON) Data Interchange Format
* [RFC 7464](https://www.rfc-editor.org/rfc/rfc7464), JavaScript Object
  Notation (JSON) Text Sequences
* [RFC 6455](https://www.rfc-editor.org/rfc/rfc6455), The WebSocket Protocol
* [RFC 8441](https://www.rfc-editor.org/rfc/rfc8441), Bootstrapping WebSockets with HTTP/2
* [RFC 9220](https://www.rfc-editor.org/rfc/rfc9220), Bootstrapping WebSockets with HTTP/3
* [`unix(7)`](https://man7.org/linux/man-pages/man7/unix.7.html), documenting
  `SO_PEERCRED`, `SO_PEERPIDFD` and `SO_PASSRIGHTS`
