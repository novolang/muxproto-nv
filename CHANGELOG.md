# Changelog

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
