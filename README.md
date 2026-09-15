# muxproto-nv

A terminal multiplexer is a program that keeps a shell session running
on a machine while the person watching it comes and goes. Its client and
its daemon talk over one connection that carries two things at once: the
terminal, and a control channel for messages such as "I resized" and
"shut this session down". This package is the codec that separates them.
It performs no input and no output: bytes go in, and frames and
terminal bytes come out.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

The two channels share one connection rather than using two, because two
connections lose the **ordering**. A resize that overtook the keystroke
before it would redraw the screen at the wrong size. Sharing one
connection means the control messages are mixed into the terminal's
bytes, and something has to pull them out again.

A **frame** is one control message. It begins with the byte 0x1c, which
is the ASCII file separator, called FS here. Then comes a **verb** of
four or more uppercase letters and digits, then an optional **argument**
after a space, then a newline. Everything that is not part of a frame is
**pass-through**: the terminal's own bytes, handed back unchanged.

One verb breaks that pattern. `KEYS` injects keystrokes into the
session, and keystrokes contain newlines and escape characters and
anything else. Its argument is a byte count, and exactly that many raw
bytes follow the newline. This is the only fact about the vocabulary
that the framing layer knows.

There are eight verbs.

| Frame | Argument | Means | Reply |
| --- | --- | --- | --- |
| `NMUX1` | rows and columns | this client's terminal is this big | none |
| `KILL` | none | shut the session down | none; the connection closes |
| `KEYS` | a byte count, then raw bytes | inject these as though typed | none; the connection closes |
| `CAPT` | a pane number, optional | serialise a pane's screen | the screen, then close |
| `LSES` | none | list the sessions | the list, then close |
| `STAT` | none | the status document | the document, then close |
| `RLOD` | none | re-read the configuration | a summary, then close |
| `SWSE` | a session index | switch to this session | none |

A **decoder** is the value that does the separating. It is fed a chunk
and answers four things at once: the decoder to continue from, the
pass-through bytes, the frames that completed, and the frames it
refused. A chunk may end anywhere, including in the middle of a header
or a payload, and the decoder carries what it is holding into the next
chunk.

A session's clients are not all equal. The **primary** is the client
that drives the session; a **mirror** is a viewer attached behind it.
The `MuxAttachState` value records which of three situations a session
is in: running with nobody attached, driven by a primary with some
number of mirrors, or left by its primary with mirrors still watching.
The third is distinct from the first because the next tick promotes one
of those mirrors rather than waiting for a new connection.

| Quantity | Value |
| --- | --- |
| The framing byte | 0x1c, ASCII FS |
| The frame terminator | a newline |
| Shortest valid verb | 4 characters |
| Default longest header, prefix and newline included | 64 bytes |
| Default largest `KEYS` payload | 65536 bytes |

This protocol has no published specification. The grammar above is what
novomux and the pseudoterminal runtime beneath `std.pty` speak today,
written down here as an interface.

## Install

```
novo pkg add muxproto-nv
```

## Example

A daemon's read loop over one chunk from the socket.

```novo
use std.bytes
use std.list
use std.str
use muxframe
use muxctl

fn main() [io]
    // One decoder per connection, with the default bounds.
    var d = muxframe.decoder(muxframe.default_limits())

    // A chunk as it came off the socket: some screen output, and a
    // resize frame riding in the same stream behind it.
    let chunk = bytes.concat([bytes.from_str("hello"),
                              muxctl.encode_command(MuxResize(24, 80))])

    let out = muxframe.feed(d, chunk)
    // Keep the decoder the chunk returned; it is the one to feed next.
    d = out.decoder

    // Everything that was not part of a frame. This goes to the terminal.
    println(bytes.to_str(out.passthrough))

    // The frames that completed in this chunk, in the order they arrived.
    for f in out.frames
        match muxctl.command_of(f)
            Ok(MuxResize(rows, cols)) =>
                println(str.from_int(rows) + "x" + str.from_int(cols))
            Ok(_)  => println(f.verb)
            Err(_) => println("refused")

    // A frame the decoder could not accept is reported here, beside the
    // good frames rather than instead of them.
    println(str.from_int(list.len(out.faults)))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: muxproto-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `muxframe` | The framing: the frame, the decoder and its two bounds, the feed-and-drain call, the faults, the encoder, the one-frame parser for a caller that has a whole buffer, and the escape for a client that wants to send the framing byte as data. |
| `muxctl` | The vocabulary: the eight verbs as a value, the conversions to and from a frame, the two spellings of the size message, and the four questions a server asks about a command before acting on it. |
| `muxattach` | The session bookkeeping: the three attach states and the transitions between them, what a client and a daemon have agreed on one connection, the session and window lists as text and back, and the smallest size across attached clients. |

## How to choose an entry point

**`muxframe.feed` is what a read loop calls.** It takes whatever the
socket gave and answers both channels in order.

**`muxframe.parse_at` reads one frame out of a buffer at an offset.** It
is for a caller that already holds a complete buffer, such as a test
with a captured stream.

**`muxctl.command_of` turns a frame into a verb value.** `muxframe` will
give you the verb as text; this is what says what it asks for and checks
the argument.

**`muxctl.encode_command` writes a command as bytes.** `muxframe.encode`
is the level below, for a verb this package does not name.

**`muxframe.flush` answers what a decoder is still holding.** Call it
when the connection closed, to see whether it went quiet mid-frame.

## The rules a user needs

1. **One decoder per connection.** Several clients attach to one
   session, and each sends its own frames. A shared decoder splices the
   first half of one client's frame onto the second half of another's
   and acts on a verb neither sent. `MuxDecoder` is a value, so a server
   keeps one beside each client.
2. **Both channels come back from one call.** `MuxDrained` carries the
   pass-through bytes and the frames together, in order. Asking for them
   separately would leave the caller to keep the two answers in step.
3. **0x1c is a byte a terminal stream can contain.** It is what a
   terminal sends when a person types Ctrl and backslash. The framing is
   safe because of where the decoder runs, not because the byte is
   impossible: it runs on the connection between a client and a daemon,
   where both ends agreed to speak this protocol, and never on a
   terminal a person is typing at. `muxframe.escaped_prefix` is the
   doubled form for a client that wants to send the byte as data.
4. **A refused frame comes back as bytes, not as a hole.** Past
   `max_header_bytes` the decoder gives up on the frame and puts
   everything it held into `passthrough`. A terminal stream containing
   0x1c followed by a long line therefore loses nothing.
5. **`faults` is a list beside the frames, not an error return.** One
   bad frame must not discard the good frames or the terminal bytes
   around it.
6. **`MuxUnknownVerb` and `MuxBadVerb` are different faults.** A newer
   client talking to an older daemon sends a well-formed frame nobody
   there can act on. That is a version skew, and reporting it as a
   protocol error blames the wrong party.
7. **A bad `KEYS` length is reported before the payload is
   accumulated.** That is what makes `max_payload_bytes` a bound rather
   than a post-mortem.
8. **The size message is sent twice, in two different spellings.** On
   connect the client sends a bare line, `NMUX1 <rows> <cols>` and a
   newline, with no framing byte, because the daemon is reading a
   prologue and not yet running a decoder. Every later resize is the
   framed form. A client that sent the framed form as its prologue
   attaches at the daemon's size and redraws at the wrong width until
   its next resize. `muxctl.prologue` writes the first and
   `muxctl.command_frame` the second.
9. **A `SWSE` index is a position, not a name.** A session closed since
   the list was read moves every session after it down one, so a cached
   index switches to the wrong session silently. `MuxSessionRef` carries
   the name the index was at, and `muxattach.index_still_names` is the
   check to make before sending.
10. **A mirror may kill and may not resize.** Any attached client may
    shut the session down. A mirror's size message changes only its own
    recorded size, because the session's size is the smallest over every
    client and the primary owns the dimensions.
    `muxctl.mirror_may_send` answers which commands a mirror may send.
11. **Expecting a reply and closing the connection are two questions.**
    `KEYS` expects no reply and closes. A resize expects no reply and
    closes nothing. `CAPT` replies and then closes.
    `muxctl.expects_reply` and `muxctl.closes_connection` are separate
    predicates, and neither is the other negated.
12. **The attach state has three cases, not two.** A server tracking a
    boolean cannot tell a headless session from one whose primary left,
    and the difference decides whether the next connection is promoted
    to primary or filed as another mirror.
    `muxattach.next_client_is_primary` is the question a daemon asks on
    every accept.
13. **`MuxAttachment.size` is one client's size, not the session's.**
    The session's size is the smallest over every attached client, which
    is `muxattach.smallest_size` over the set the caller owns.
14. **The session list is tab-separated, not aligned.** It is parsed by
    a client far more often than it is read by a person, and aligning
    the columns would make the parse depend on the longest name.
15. **`muxattach.take_reattaches` reads and clears.** The counter it
    answers is the number of reattaches since the last call.

## What is not included

- **A subscription frame.** A ninth verb that held a connection open and
  sent a message on every change would replace the status poll a client
  runs today. Nothing answers it yet, and a decoder that accepted a verb
  no daemon handles would be worse than one that does not.
- **Version negotiation.** The size verb carries a version in its name
  and there has only ever been one. `MuxUnknownVerb` is the whole of the
  forward-compatibility story.
- **The reply payloads.** `CAPT` answers a serialised screen and `STAT`
  answers a status document. Both are the multiplexer's own formats, and
  a codec that defined them would be defining the multiplexer.
- **Encryption and authentication.** The transport is a Unix socket, and
  the kernel's peer credentials are the authentication. That belongs to
  whatever owns the socket.
- **The transport itself.** See "Related packages".
- **A microcontroller build.** `Str` and `Bytes` run through the whole
  surface, and the consumer is a multiplexer on a machine with a
  filesystem. This package makes no device claim.

## Related packages

- [unixsock-nv](https://novo-lang.org/packages/unixsock-nv) is the Unix
  domain socket these frames travel over. This package does not depend
  on it, and could not: a codec that named its transport could not be
  tested against a captured stream, or reused over a pipe, a
  pseudoterminal or a test double. In a program the dependency runs the
  other way round.
- [pty-nv](https://novo-lang.org/packages/pty-nv) is the pseudoterminal
  the session runs on, and the readiness loop a daemon runs over many of
  them.
- [novo-vte](https://novo-lang.org/packages/novo-vte) is the escape
  sequence parser and grid model that turns the pass-through bytes into
  a screen. What a `CAPT` serialises is one of its grids.
- [ansi-nv](https://novo-lang.org/packages/ansi-nv) is the escape
  sequence layer with no grid under it, for a consumer that wants the
  actions rather than the screen.
- `std.pty` in the standard library is the pseudoterminal primitives the
  runtime binds: spawn a shell on one, read and write it, resize it.
  Frame separation lives inside that runtime today, in C, behind those
  primitives. This package is that filter as a value, which is what lets
  a client, a daemon and a test read one parser.

## Tests

```bash
novo test --isolate tests/frame_tests.nv     # 12 tests: the framing
novo test --isolate tests/command_tests.nv   #  7 tests: the verbs
novo test --isolate tests/attach_tests.nv    #  8 tests: the session state
```

The protocols this one is measured against are tmux's control mode, with
its `%begin` and `%end` output blocks, and GNU screen's `-X` command
channel. Both are older and larger and carry their own reply grammar.
What this one has that neither does is a control channel sharing a
stream with the terminal, which is where the framing byte comes from.
The behaviour the suites assert is the behaviour of the multiplexer that
speaks it today.

No test opens a socket. Every case is a chunk of bytes the test writes
out, so a whole conversation is a value. The cases that matter are a
frame split across two chunks, a header that runs past the limit coming
back as pass-through, a `KEYS` payload with a newline in it, a bad
length refused before anything is allocated, an unknown verb reported
apart from a malformed one, the two spellings of the size message, and a
session index that no longer names the session it named.

The tests compile today and fail at run, each on the
`not implemented: muxproto-nv.<module>.<fn>` panic that is its body.
That is the expected state of an interface release. They turn green one
at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `muxframe.frame_prefix`, `.frame_terminator`, `.default_limits`, `.limits` | no |
| `muxframe.decoder`, `.feed`, `.flush`, `.in_frame` | no |
| `muxframe.payload_length_of`, `.parse_at` | no |
| `muxframe.encode`, `.frame`, `.frame_with`, `.frame_payload`, `.escaped_prefix` | no |
| `muxframe.is_known_verb`, `.known_verbs` | no |
| `MuxFault`'s `Error` implementation | no |
| `muxctl.verb_of`, `.command_of`, `.command_frame`, `.encode_command` | no |
| `muxctl.prologue`, `.parse_prologue` | no |
| `muxctl.expects_reply`, `.closes_connection`, `.mirror_may_send`, `.capture_target` | no |
| `muxattach.headless`, `.has_clients`, `.has_primary`, `.next_client_is_primary` | no |
| `muxattach.on_attach`, `.on_primary_detach`, `.on_mirror_detach`, `.on_promote`, `.mirror_count` | no |
| `muxattach.attachment`, `.with_size`, `.smallest_size` | no |
| `muxattach.count_reattach`, `.take_reattaches` | no |
| `muxattach.session_list_text`, `.parse_session_list`, `.index_still_names` | no |
| `muxattach.window_list_text`, `.parse_window_list` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
