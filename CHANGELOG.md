# Changelog

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide
(docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — interface

The interface, published before anything is implemented: every `pub fn`
body is a `todo()`, and the signatures, the effect rows and the tests are
the design.

- Three modules, `[]` throughout: `muxframe` (the framing and the
  feed-and-drain decoder), `muxctl` (the eight verbs as values) and
  `muxattach` (the session state a reattach walks through).
- 46 public functions and one `Error` implementation, every body a
  `todo("muxproto-nv.<module>.<fn>")`.
- Three test suites, 27 tests, red on purpose.
- The protocol lifted out of the standard library's `pty` module, where
  it lives as a C frame filter and a read-and-clear global per verb.
  `std.pty` keeps the pseudoterminal and novomux keeps everything the
  frames are about; the README carries both tables.

### Design notes

What this package replaces in the runtime is a forty-byte header buffer
and a frame-state integer in the C pseudoterminal filter, with a second
copy per mirror connection, and the eight `take_*` externs the pump
reads a parsed frame back through: `take_pending_resize`,
`take_kill_request`, `take_capture_request`, `take_capture_index`,
`take_lssession_request`, `take_status_request`, `take_reload_request`
and `take_swsession_request`. `feed` answers the frames directly, in
order, with no read-and-clear global between the parse and the caller.

What moves here is the grammar: the prefix byte, the header, the
length-prefixed verb, the eight words, the bare prologue and the
bookkeeping around an attach. What the multiplexer keeps is everything
the frames are about — the panes and their grids, the splits, the focus,
the keymap that decides a detach key, the configuration a reload
re-reads, the status document, and the serialisation of a grid that a
capture replies with.
