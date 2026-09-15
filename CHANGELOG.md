# Changelog

All notable changes to libtree-sitter-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-15

The first release: thirty-eight entry points of the libtree-sitter C
API, one `@ffi` declaration each, and no logic.

### Added

- `libtree_sitter` — the whole surface, in five groups.
  - The parser: `ts_parser_new`, `ts_parser_delete`,
    `ts_parser_set_language`, `ts_parser_language`,
    `ts_parser_parse_string`, `ts_parser_parse_string_encoding`,
    `ts_parser_reset`, `ts_parser_set_timeout_micros` and
    `ts_parser_timeout_micros`.
  - The syntax tree: `ts_tree_copy`, `ts_tree_delete` and
    `ts_tree_language`.
  - The query: `ts_query_new`, `ts_query_delete`, the three counts, the
    pattern offset, the two identifier lookups, and the two disable
    calls.
  - The query cursor: `ts_query_cursor_new`, `ts_query_cursor_delete`,
    the match limit pair, the limit-exceeded flag, the byte range,
    `ts_query_cursor_next_match` and `ts_query_cursor_remove_match`.
  - The language table: the symbol count, the two symbol lookups, the
    field count, the two field lookups, the symbol kind and the ABI
    version.
- `tests/libtree_sitter_tests.nv` — eleven tests over the signatures.
  They call the C library, so they need libtree-sitter installed.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Named as missing

**Passing a structure by value.** The novo-lang foreign function
interface passes integers, floats and strings. Every libtree-sitter
entry point that takes or returns a `TSNode`, a `TSPoint`, a `TSRange`,
a `TSTreeCursor` or a `TSQueryMatch` by value is therefore absent, and
that is most of the node API and all of the tree cursor API. A program
that needs them writes a small C function of its own that takes the
same values behind pointers.
