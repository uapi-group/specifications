---
title: "UAPI.17 OSC 2811: Terminal Size Change Notification"
category: Concepts
layout: default
version: 1.0
SPDX-License-Identifier: CC-BY-4.0
weight: 17
aliases:
- /UAPI.17
- /17
---

# UAPI.17 OSC 2811: Terminal Size Change Notification

| Version | Changes         |
|---------|-----------------|
| 1.0     | Initial release |

Interactive programs frequently need to know the dimensions (i.e. width and
height in character cells) of the terminal they are connected to, and need to
learn about changes to them in a timely fashion. On local systems the kernel's
TTY layer provides this information via the `TIOCGWINSZ` `ioctl()`, and
notifies the foreground process group about changes via the `SIGWINCH` signal.
This mechanism has substantial limitations however:

1. On serial links (e.g. serial consoles, various forms of virtualized TTYs)
   there is no out-of-band channel through which dimension information could be
   propagated at all. The kernel's idea of the dimensions of such terminals is
   hence typically not initialized at all, or is initialized only once with
   possibly outdated data, and never updated when the terminal emulator's
   window is resized.

2. `SIGWINCH` is delivered to the foreground process group of the terminal
   only. Processes reading or writing the terminal without being part of it
   never learn about dimension changes.

3. UNIX process signals are a problematic programming interface in many ways:
   they are process-global (and hence awkward to handle from libraries or
   threads), and their delivery is not synchronized with the input stream.

Existing in-band terminal mechanisms for querying the dimensions (e.g.
`XTWINOPS` aka "`CSI 18 t`") are polling interfaces only: a program can ask for
the current dimensions, but is not notified when they change later — timely
updates would require continuous polling, which is wasteful and cannot deliver
prompt notifications.

OSC 2811 closes this gap: it allows a terminal application ("*client*") to
subscribe — in-band and one-shot — to dimension change notifications from the
terminal emulator ("*emulator*").

The OSC sequence number 2811 was picked because — to the best of our knowledge
— it does not conflict with any OSC sequence in public use (see the survey of
known OSC prefixes in [UAPI.15](osc_context.md)).

## Use Cases

1. Correct terminal size notifications can be delivered even if a simple
   transport is used that has no side-channel for terminal dimension
   information, and where it is hence desirable to deliver this information
   inline, i.e. within the terminal data stream itself. Examples for such
   transports are raw RS232 serial links or raw TCP streams: they carry nothing
   but the terminal's input and output bytes, and thus — unlike local
   pseudo-terminals or fully-featured remoting protocols such as SSH — provide
   no other channel through which dimension changes could ever be communicated.

2. Full-screen terminal applications on serial consoles can render correctly,
   and re-render promptly when the terminal emulator window is resized, without
   requiring the user to manually invoke `stty` or `resize`.

3. Shells and other programs can keep the kernel TTY layer's dimension
   information up-to-date on serial links (via `TIOCSWINSZ`), so that classic
   `TIOCGWINSZ`/`SIGWINCH` consumers benefit, too.

4. Programs that are not the terminal's foreground process group can follow
   dimension changes.

5. Event-loop based programs can process dimension changes on the same input
   stream they read keyboard input from, without resorting to UNIX signal
   handling.

## Semantics

The protocol knows three message types: the *subscribe sequence* and the
*cancel sequence*, both sent by the client to the emulator; and the *reply
sequence*, sent by the emulator to the client (i.e. inserted into the
terminal's input stream).

A client that wants to be notified about dimension changes first determines its
current understanding of the terminal dimensions (typically via the
`TIOCGWINSZ` `ioctl()`; on serial links possibly from earlier OSC 2811 replies,
or simply `0`×`0` if it has no information). It then sends a subscribe sequence
to the emulator, carrying these dimensions, i.e. declaring "*this is what I
believe the terminal dimensions are*".

Upon receiving a subscribe sequence the emulator compares the client's declared
dimensions with the actual dimensions of the terminal:

1. If they *differ*, the emulator immediately sends a reply sequence carrying
   the actual dimensions. The subscription is thereby consumed.

2. If they *match*, the emulator records the subscription and sends nothing for
   now. When the terminal dimensions later change, it sends a single reply
   sequence carrying the new dimensions. The subscription is thereby consumed.

Subscriptions are strictly *one-shot*: an emulator sends at most one reply
sequence per subscribe sequence received. A client that wants continuous
updates must send a new subscribe sequence — typically declaring the dimensions
it just learned from the reply — each time it processes a reply. This
"resubscribe with current belief" scheme is inherently race-free: if the
dimensions changed again between the emulator sending a reply and the client
resubscribing, the resubscription's declared dimensions are already outdated,
and the emulator replies immediately once more.

A client may declare dimensions of `0`×`0` (which are never the actual
dimensions of any terminal) to unconditionally request an immediate reply, for
example to initially learn the dimensions on a serial link, or to determine if
the feature is available.

At most one subscription is pending per terminal at any time: a subscribe
sequence received while another subscription is pending replaces the earlier
one (which is discarded without a reply).

A client may send a cancel sequence (a subscribe sequence without any fields)
to discard any pending subscription without a reply. Clients should do so when
exiting (in particular full-screen applications when restoring the terminal),
so that a later application on the same terminal does not receive a reply it
did not ask for.

Clients must be prepared that a reply might never arrive — either because the
declared dimensions were correct and simply never changed, or because the
emulator does not implement this specification — or that it arrives only after
an arbitrarily long delay. Clients hence must never synchronously block on a
reply, but process replies asynchronously from their regular input stream, and
must continue to support the classic `TIOCGWINSZ`/`SIGWINCH` mechanism where
available.

Replies may appear at any position in the input stream, interleaved with
keyboard input and other terminal responses; clients must be prepared to parse
them from such a mixed stream.

It is recommended that clients on serial links propagate dimensions learned
from reply sequences into the kernel TTY layer via the `TIOCSWINSZ` `ioctl()`,
so that other processes using the same terminal — as well as the client's own
`TIOCGWINSZ` calls — return correct information.

Terminal multiplexers (e.g. `tmux`, `screen`) should implement this protocol
towards their clients themselves, replying with the dimensions of the pane or
window the client is connected to, rather than passing sequences through to the
outer terminal.

## General Syntax

This builds on ECMA-48, and reuses the OSC and ST concepts introduced there.

For sequences following this specification it is recommended to encode OSC as
0x1B 0x5D, and ST as 0x1B 0x5C.

The subscribe sequence begins with OSC, followed by the string `2811;?` (the
question mark marking the sequence as a request, following the convention
established by the xterm color query sequences), followed by the fields
described below, each preceded by a semicolon (`;`). Each field consists of a
string identifying the field, followed by an equal sign (`=`), and the field
value. The sequence ends in ST.

The cancel sequence is simply a subscribe sequence without any fields, i.e. OSC
followed by `2811;?`, ending in ST.

The reply sequence uses the same syntax as the subscribe sequence, but carries
no `?` marker: it begins with OSC, followed by the string `2811`, followed by
the fields, ending in ST. Requests and replies are hence distinguishable by
their bytes alone, without knowledge of the direction they travelled in.

## Fields

The following fields are currently defined, for both the subscribe and the
reply sequence:

| Field      | Description                                             |
|------------|---------------------------------------------------------|
| `columns=` | The terminal width in character cells (i.e. columns), a decimal integer in the range 0…65535 |
| `lines=`   | The terminal height in character cells (i.e. rows), a decimal integer in the range 0…65535 |

The order of the fields is undefined, they may appear in any order. Emulators
should send `columns=` before `lines=` in replies, but clients must accept any
order.

In a subscribe sequence a field that cannot be parsed shall be treated by the
emulator as if it carried the value `0` (and hence never matches the actual
dimensions, triggering an immediate reply). A subscribe sequence without any
fields at all is the cancel sequence, see above.

Unknown fields shall be ignored, in both directions. Future revisions of this
specification might define additional fields (for example carrying the terminal
dimensions in pixels).

Each field may appear at most once per sequence. A sequence in which the same
field is mentioned more than once is invalid.

An emulator may include more fields in a reply sequence than the client
mentioned in its subscribe sequence. Clients hence must not assume that the set
of fields in a reply matches the set of fields they declared, and must simply
ignore any fields they do not recognize or did not ask for.

## Processing, Limits, Security

An emulator implementing this specification must only ever send a reply
sequence in response to a subscribe sequence it previously received, never
unsolicited. Since replies consist exclusively of fields with decimal integer
values in a fixed syntax, the surface for injecting unexpected data into the
client's input stream is minimal.

The usual terminal reset sequences (both soft and hard reset) should discard
any pending subscription without a reply, so that a freshly initialized
application does not inherit a stale subscription of its predecessor. The same
applies to a TTY hangup (`vhangup()`).

All received data should be processed in a lenient, graceful fashion: invalid
fields should be ignored (or, in subscribe sequences, treated as `0`, see
above), and unknown fields skipped over.

The `?` marker also makes the protocol robust against reflection: an emulator
receiving a sequence *without* the `?` marker shall ignore it — this can happen
for example if the TTY layer's local echo reflects a reply sequence back into
the output stream. Conversely, a client encountering a sequence *with* the `?`
marker in its input stream shall ignore it.

Since subscriptions are one-shot and at most one subscription is pending per
terminal at any time, this protocol is naturally rate-limited and requires no
per-client resource accounting in the emulator.

## Extensibility

Future versions of this specification might add support for additional general
terminal properties that may be queried and subscribed to via this mechanism.
Any such extensions will be implemented via additional fields, on top of
`columns=` and `lines=`. Both clients and emulators thus must be prepared to
handle unrecognized fields gracefully and ignore them, pretending they weren't
set.

If an emulator receives a subscribe sequence with a non-zero number of fields
but of which it recognizes none, it should consider this equivalent to such a
sequence with a zero number of fields, i.e. as a cancel request.

## Examples

1. A client determines the terminal dimensions to be 80×24 via `TIOCGWINSZ` and
   subscribes to change notifications: `OSC "2811;?;columns=80;lines=24" ST`

2. The user resizes the terminal window to 120×40, the emulator replies:
   `OSC "2811;columns=120;lines=40" ST`

3. The client processes the reply, re-renders its screen, and resubscribes:
   `OSC "2811;?;columns=120;lines=40" ST`

4. A client on a serial console with no prior knowledge of the dimensions
   requests an immediate report: `OSC "2811;?;columns=0;lines=0" ST`

5. A full-screen application exits and discards its pending subscription:
   `OSC "2811;?" ST`

## Syntax in ABNF

```abnf
OSC       = %x1B %x5D
ST        = %x1B %x5C

DECIMAL   = "0"-"9"
UINT16    = 1*5DECIMAL

COLUMNS   = "columns=" UINT16
LINES     = "lines=" UINT16

FIELD     = COLUMNS / LINES

SUBSCRIBE = OSC "2811;?" 1*(";" FIELD) ST
CANCEL    = OSC "2811;?" ST
REPLY     = OSC "2811" 1*(";" FIELD) ST
```
