# libtree-sitter-sys

Tree-sitter is a parser generator tool and an incremental parsing
library. It builds a concrete syntax tree for a source file and updates
that tree efficiently as the file is edited. The library is written in
C, and its C API is documented in
[the tree-sitter manual](https://tree-sitter.github.io/tree-sitter/using-parsers)
and in the header `tree_sitter/api.h`. This package declares
thirty-eight of its entry points to novo-lang, one declaration each.

Every function here is a declaration of a function in libtree-sitter.
The package contains no logic of its own, and it does nothing without
the C library installed. The section "What is not included" says which
entry points are not here, and why.

## What it is

A parser turns source text into a tree. Tree-sitter's trees are
*concrete*. Every token in the file has a node, including the brackets
and the commas. An editor uses that to map a position in the text to a
position in the tree.

Tree-sitter is *incremental*. Given the previous tree for a file and a
description of the edits made to it, the parser reuses the parts of the
tree the edits did not touch. A keystroke in a large file costs a small
reparse rather than a whole one.

Tree-sitter is also *error tolerant*. A file with syntax errors still
produces a tree. The parts that could not be understood become error
nodes, and the rest of the tree remains useful.

A **grammar** is a separate thing from the library. Grammars are written
in JavaScript, and the tree-sitter command line tool generates C from
them. That C compiles to a library exporting one function,
`tree_sitter_<name>`, which answers the address of a **language table**.
The parsing library reads that table, and it contains no grammar of its
own. A program that parses JSON therefore needs libtree-sitter and a
compiled JSON grammar, and this package binds the first of the two.

A **query** is a pattern over a tree, written as an S-expression, with
names, called captures, attached to the nodes it should report.
Syntax highlighting, structural search and code navigation are
expressed as queries, and the library runs them itself.

## Install

```
novo pkg add libtree-sitter-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its header come from the system package
`libtree-sitter-dev`:

```
sudo apt install libtree-sitter-dev
```

On macOS the Homebrew formula is `tree-sitter`. On other systems the
library builds from the tree-sitter source with `make`.

A grammar is a second thing to install or build, and there is no system
package for most of them. The tree-sitter command line tool builds one
from its repository.

## Example

One JSON file, parsed:

```novo ignore
use libtree_sitter

// Every compiled grammar exports one function, and it answers the
// address of the language table.  This one comes from the JSON
// grammar, built as libtree-sitter-json.so.
@ffi(library = "tree-sitter-json", symbol = "tree_sitter_json")
fn tree_sitter_json() -> Int [ffi]

fn main() [io, ffi]
    let parser = libtree_sitter.ts_parser_new()
    let assigned = libtree_sitter.ts_parser_set_language(parser, tree_sitter_json())
    if assigned == 0
        println("the grammar was generated for a different tree-sitter")
        return
    let source = "{ \"answer\": 42 }"
    let tree = libtree_sitter.ts_parser_parse_string(parser, 0, source, 16)
    if tree == 0
        println("the parse produced no tree")
        return
    println("parsed, and the tree is at ${tree}")
    libtree_sitter.ts_tree_delete(tree)
    libtree_sitter.ts_parser_delete(parser)
```

The example is not compiled, because it needs a compiled JSON grammar,
which this repository does not ship.

## What the package contains

| Module | Contents |
| --- | --- |
| `libtree_sitter` | Every entry point, in five groups: the parser, the syntax tree, the query, the query cursor and the language table. |

The five groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Parser | 9 | Creates a parser, assigns it a language, parses a buffer, and bounds how long a parse may take. |
| Syntax tree | 3 | Copies a tree, frees a tree, and reads the language that produced it. |
| Query | 10 | Compiles a pattern, counts its parts, names its captures, and disables the parts a caller does not want. |
| Query cursor | 8 | Holds the state of a running query, bounds its work, and drains its matches. |
| Language table | 8 | Reads the node types, the field names and the ABI version out of a compiled grammar. |

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** Every handle the C
   library returns arrives as the address it returned. Every entry point
   that allocates answers zero when it could not.
2. **Every handle is freed by its own function.** `ts_parser_delete`
   frees a parser, `ts_tree_delete` a tree, `ts_query_delete` a query
   and `ts_query_cursor_delete` a query cursor. A language table belongs
   to the grammar and is never freed.
3. **A length is in bytes, and it is always passed.** The C functions
   take a pointer and a length rather than a terminated string, so
   `ts_parser_parse_string` is given the byte length of its buffer and
   `ts_query_new` the byte length of its pattern text.
4. **An out-parameter is the address of a caller-owned slot.**
   `ts_query_new` writes the error offset and the error kind into two
   32-bit slots the caller supplies. `ptr.alloc_word` in the standard
   library returns the address of an eight-byte slot, which holds one,
   and `ptr.free` releases it.
5. **A string the library returns is owned by the library.** The node
   type names, the field names and the capture names are addresses of
   C strings inside the language table or the query. Read them with
   `ptr.read_str` and do not free them.
6. **A grammar and the library must agree on a version.**
   `ts_parser_set_language` answers 0 when they do not, and
   `ts_language_version` says which version the grammar was generated
   for. libtree-sitter 0.20.8 accepts versions 13 and 14.
7. **A query cursor must be executing a query before its matches are
   drained.** The C library does not check, and
   `ts_query_cursor_next_match` on an idle cursor fails inside it rather
   than answering. Starting a query takes `ts_query_cursor_exec`, which
   is not in this package. The next section says why.

## What is not included

- **Every entry point that passes or returns a structure by value.**
  The novo-lang foreign function interface passes integers, floats and
  strings, and no structure by value. `TSNode`, `TSPoint`, `TSRange`,
  `TSTreeCursor` and `TSQueryMatch` are all passed by value in C. That
  rules out the whole node API, which includes `ts_node_type`,
  `ts_node_child` and `ts_node_start_byte`. It also rules out the whole
  tree cursor API, `ts_tree_root_node` and `ts_query_cursor_exec`. A
  program that needs them writes a small C function of its own that
  takes the same values behind pointers, and calls that.
- **The callback-driven parse.** `ts_parser_parse` takes a `TSInput`
  structure holding a function pointer, and a novo-lang function is not
  a C function pointer. `ts_parser_parse_string` is the whole-buffer
  form and it is here.
- **The editing entry points.** `ts_tree_edit` and `ts_node_edit` take a
  `TSInputEdit` structure the caller must lay out byte by byte. They are
  left out of this release.
- **The cancellation flag and the capture iterator.**
  `ts_parser_set_cancellation_flag` and `ts_query_cursor_next_capture`
  are left out of this release.
- **The logger and the graph printers.** `ts_parser_set_logger` takes a
  function pointer. `ts_parser_print_dot_graphs` and
  `ts_tree_print_dot_graph` write to a file descriptor and are debugging
  aids for the C library's own authors.
- **The custom allocator.** `ts_set_allocator` replaces the library's
  `malloc` for the whole process, including every other user of
  libtree-sitter in it.
- **Grammars.** This package binds the parsing library. A grammar is a
  separate shared library, built from a separate repository.

## Related packages

[novo-syntax](https://novo-lang.org/packages/novo-syntax) is the lexer
and parser for novo-lang itself, written in novo-lang with no C library.
It is published as an interface release. Every function in it is
declared and none has a body yet. When its functions have bodies, it is
the choice for a program that reads novo-lang source.

There is no novo-lang port of tree-sitter. A grammar is a table
generated by the tree-sitter tool, and this package lets a program use
the grammars that already exist for it.

## Tests

`tests/libtree_sitter_tests.nv` holds eleven tests over the
thirty-eight entry points. They call the C library, so `novo test` needs
libtree-sitter installed and linkable:

```
novo test tests/libtree_sitter_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

The tests assert what can be observed without a grammar. A fresh parser
has no language, and a parse without one produces no tree. The timeout
is stored and read back. A query over the null language is refused with
the language error code. A fresh query cursor takes a match limit and
reports it back. The calls that need a language table are written out
in full and guarded by a language the suite cannot produce, so the
compiler checks them even where the library cannot be asked.

## Licence

Apache-2.0. See [LICENSE](LICENSE).

Tree-sitter itself is distributed under the MIT licence, and installing
it is the reader's own step.
