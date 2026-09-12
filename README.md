# muxproto-nv

The terminal multiplexer's control protocol, as a codec that performs
nothing: one stream carries a terminal and a control channel, and this
is what separates them.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

A multiplexer's client and its daemon share one Unix socket.  Keystrokes
go one way, screen bytes come back, and in among them the client has to
be able to say "I resized", "capture that pane", "shut this session
down".  A second connection for the control channel would be simpler and
would lose the **ordering** — a resize that overtook the keystroke before
it redraws at the wrong size — so the frames ride the same stream, and
something has to pull them out.

That something lives inside `std.pty` today, in C, in a filter shared
between the runtime's daemon paths.  This package is it as a value:
bytes in, frames and pass-through bytes out, `[]` throughout, so a
client, a daemon and a test double all read one parser.

| surface | module | reach for it when |
| --- | --- | --- |
| the **framing** | `muxframe` | you are reading or writing bytes |
| the **verbs** | `muxctl` | you want to know what a frame asks for |
| the **session** | `muxattach` | you are deciding who drives and what size it is |

Forty-six public functions and one `Error` implementation, every body a
`todo()`.

## The load-bearing interface

```novo norun:pseudo
pub struct MuxDrained
    decoder: MuxDecoder
    passthrough: Bytes
    frames: [MuxFrame]
    faults: [MuxFault]
```

**Both halves come back, in order, from one call.**  A decoder that
answered only frames would lose the terminal bytes between them; one
that answered only bytes would be no decoder at all; and a caller that
had to ask twice would have to keep the two answers in step itself.
`docs/publishing.md` § How a `core` package takes bytes from its host
calls this feed-and-drain, and it is the right shape here for the reason
it is right for csv-nv: a record is small and the stream is sequential.

Three things fall out of that struct, and each is a defect in the shape
that does not have it:

- **The decoder is per connection.**  Several clients attach to one
  session and each sends its own frames; one shared decoder splices the
  first half of one client's frame onto the second half of another's and
  acts on a verb neither of them sent.  `MuxDecoder` is a value, so a
  server keeps one beside each client and the mistake is not
  expressible.
- **A refused frame is bytes, not a hole.**  Past `max_header_bytes` the
  decoder gives up on the frame and emits everything it held as ordinary
  data.  A fixed buffer that overflowed would drop it, and a terminal
  stream containing `0x1c` and then a long line would lose that line.
- **`faults` is a list beside the frames, not an error return.**  One
  bad frame in a chunk must not discard the good ones or the terminal
  bytes around them, and a daemon that logs the fault can tell a client
  sending garbage from a client sending nothing.

## `0x1c` is not a byte the terminal stream cannot contain

The framing byte is ASCII FS, which a terminal sends when a person types
**Ctrl-\\**.  So the safety is not in the byte, it is in the **scope**:
the decoder runs on the daemon socket, where both ends agreed to speak
this protocol, and never on a terminal a person is typing at directly.
The runtime's own filter says the same thing in a comment — "standalone
mode is unfiltered; a user typing Ctrl-\\ at a local terminal still gets
through" — and it is worth saying out loud here, because a reader who
believed the byte was impossible would put the decoder in the wrong
place.

`muxframe.escaped_prefix` is the doubled form for a client that wants to
send the byte as data, which is what makes the decoder safe to run over
an attached client's input as well.

## The eight verbs

| frame | argument | means | reply |
| --- | --- | --- | --- |
| `NMUX1` | `<rows> <cols>` | this client's terminal is this big | none |
| `KILL` | — | shut the session down | none, connection closes |
| `KEYS` | `<len>` + raw bytes | inject these as though typed | none, connection closes |
| `CAPT` | optional `<pane>` | serialise a pane's screen | the screen, then close |
| `LSES` | — | list the sessions | the list, then close |
| `STAT` | — | the status document | the document, then close |
| `RLOD` | — | re-read the configuration | a summary, then close |
| `SWSE` | `<index>` | switch to this session | none |

Every frame is `0x1c`, four or more ASCII characters, an optional
argument and a newline — readable in a packet dump, which is most of why
it has survived.  `KEYS` is the exception: its header declares a byte
count and exactly that many **raw** bytes follow, because injected
keystrokes contain newlines and escapes and anything else.  So the
framing layer knows exactly one thing about the vocabulary, and
`muxframe.payload_length_of` is that one fact published as a function
rather than buried in a state machine.

## Four places the protocol bites

- **`NMUX1` is sent twice, in two different spellings.**  On connect the
  client sends the size as a BARE line — `NMUX1 <rows> <cols>\n`, no
  prefix byte — because at that moment the daemon is reading a prologue
  and not yet running a frame decoder.  Every later resize is the framed
  form.  A client that sent the framed form as its prologue attaches at
  the daemon's size and the session redraws at the wrong width until the
  next resize.  `muxctl.prologue` and `muxctl.command_frame` are the two,
  named apart.
- **A `SWSE` index is a position, not a name.**  A session closed since
  the list was read moves every session after it down one, so a client
  that cached an index switches to the wrong session, silently.
  `MuxSessionRef` carries the name it was at and
  `muxattach.index_still_names` is the check to make first; the wire
  still carries the index, because that is what the daemon reads.
- **A mirror may kill and may not resize.**  Any attached client may
  shut the session down; a viewer's `NMUX1` changes only its own
  recorded size, because the session's size is the minimum over every
  client and the primary owns the dimensions.  A server that applied a
  mirror's resize resizes everybody to the newest viewer's window.
  `muxctl.mirror_may_send`.
- **Expecting a reply and closing the connection are not opposites.**
  `KEYS` is fire-and-forget AND closes; `NMUX1` rides an attached
  client's stream and closes nothing; `CAPT` answers and then closes.  A
  caller that derived one from the other holds a connection open for a
  `KILL` and closes one on a `CAPT`.  Two predicates, not one negated.

## Attach is three states and not a boolean

```novo norun:pseudo
pub enum MuxAttachState
    MuxHeadless
    MuxDriven(mirrors: Int)
    MuxDetached(mirrors: Int)
```

A session is running with nobody watching, or a primary client is
attached with viewers behind it, or the primary left and viewers remain.
A server tracking `attached: Bool` cannot tell the first from the third,
and the difference decides whether the next connection is promoted
straight to primary or filed as another mirror.
`muxattach.next_client_is_primary` is the one question a daemon asks on
every accept, published so the answer is in one place rather than
reimplemented at each accept path — which is how two accept paths in a
multiplexer come to disagree.

## The one example that will work

```novo
use muxframe

// A daemon's read loop: whatever the socket had, split into the
// terminal's bytes and the control frames that were riding in them.
fn tick(d: MuxDecoder, chunk: Bytes) -> MuxDrained
    muxframe.feed(d, chunk)

fn main() [io]
    println("one stream, two channels")
```

## Adding it, and checking it

```console
$ novo pkg add muxproto-nv
$ novo pkg build
$ novo test --isolate tests/frame_tests.nv
```

The suites are **red on purpose**: every body is a `todo()`, so every
assertion reaches `not implemented: muxproto-nv.<module>.<fn>`.
Twenty-seven tests across three suites, all red, every failure that
message.

## The layer, and why

`core`, from the plan, and `[]` throughout.  The socket is unixsock-nv's,
the pseudoterminal is `std.pty`'s, and the screen is novomux's; what is
left is arithmetic over bytes a caller already holds, which is what lets
the same parser serve a client, a daemon and a test.

**No device claim**, and so no `tests/embedded_probe.nv`.  `Str` and
`Bytes` are throughout — a verb and its argument are text, a `KEYS`
payload is a caller's buffer — and the consumer is a multiplexer on a
machine with a filesystem.  The `core-embedded` audit row passes on a
package that makes no claim, and this one does not.

**No dependencies.**  Not unixsock-nv, which carries these frames: a
codec that named its transport could not be tested against a captured
stream or reused over a pipe, a pseudoterminal or a test double.  The
dependency runs the other way round in a program and neither way round
in the manifests.

## What moves, and what novomux keeps

What moves is the **grammar**: the prefix byte, the header, the
length-prefixed verb, the eight words, the bare prologue, and the
bookkeeping around an attach.  It lives today in `bin/novo_rt_pty.c` as
a 40-byte header buffer and a `g_frame_state` integer, with a second
copy per mirror connection, and in `orbit/novomux/src/main.nv` as a
writer that emits `28` and then a string at each of eight call sites.

**What novomux keeps** is everything the frames are ABOUT: the panes and
their grids, the splits and the relayout, the focus, the keymap that
decides a detach key, the configuration a `RLOD` re-reads, the status
document a `STAT` answers, and the serialisation of a grid that a `CAPT`
replies with.  This package says that a frame arrived and what it asked
for; novomux is what does it.

**What `std.pty` keeps** is the pseudoterminal.  The eight `take_*`
externs — `take_pending_resize`, `take_kill_request`,
`take_capture_request`, `take_capture_index`, `take_lssession_request`,
`take_status_request`, `take_reload_request`, `take_swsession_request` —
are the runtime's way of handing a parsed frame up to the pump, and they
are exactly what a decoder in-language replaces: `feed` answers the
frames directly, in order, with no read-and-clear global between the
parse and the caller.

## What is deliberately absent

- **`EVNT`.**  novomux's own comment names a subscription frame that
  would hold a connection open and broadcast on transition, as the
  upgrade path from the 200 ms status poll it uses today.  It is not
  implemented and no daemon answers it, so putting it in the verb set
  would make a decoder accept something nothing handles.  It is the
  ninth verb when somebody writes it.
- **A version negotiation.**  `NMUX1` carries a version in its name and
  there has only ever been one.  `MuxUnknownVerb` is the whole of the
  forward-compatibility story: a newer client's frame is well formed and
  unknown, which a daemon can log as a skew rather than as an error.
- **The reply payloads.**  A `CAPT` answers a serialised grid and a
  `STAT` answers a status document; both are novomux's formats, and a
  codec that defined them would be defining the multiplexer.
- **Encryption and authentication.**  The transport is a Unix socket and
  the kernel's `SO_PEERCRED` is the authentication — unixsock-nv's, not
  this package's.

## Reference

tmux's control mode and its `%begin`/`%end` output blocks, and GNU
screen's `-X` command channel, are the two protocols this one is
measured against; both are older, larger and carry their own reply
grammar.  What this protocol has that neither does is that the control
channel and the terminal share one stream, which is where the framing
byte and this package come from.  The normative source is the
implementation: `bin/novo_rt_pty.c`'s frame filter and
`orbit/novomux/src/main.nv`'s writers, transcribed here as a grammar.

## Licence

Apache-2.0.
