# Changelog

All notable changes to elf-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-10

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `elfrange` — `ElfRange`, the type that makes the package sans-IO. It
  means a *span* into a buffer the caller holds and a *request* for
  bytes the caller has not read, and it is one type because those are
  one fact. Every answer this package gives about where something is
  comes back as one.
- `elfhdr` — the identification and the file header, parsed in two
  steps because an ELF header's own size and byte order are inside it.
  `ident_range`, `ident`, `header_range` and `header_from` are the
  sequence a caller reading from a probe uses; `header` is the same
  thing for a caller holding the file. All four class-and-endianness
  combinations, because a host tool that read only 32-bit little-endian
  would fail on the first RISC-V64 board.
- `elfsec` — the section header table with names resolved, and
  `string_at`, the one primitive both string tables are read with. The
  two escapes a file uses to describe itself when it outgrows 16 bits —
  `e_shnum = 0` and `SHN_XINDEX`, both stored in section 0 — are
  resolved here rather than left to the caller, because a caller that
  did not know about them reads an empty table out of a file with forty
  thousand sections and reports nothing wrong.
- `elfsym` — the symbol table with names resolved through the string
  table `sh_link` names, and `name_table`, whose only job is to make it
  impossible to read symbol names out of the *section* name table.
  `in_section` is the query the deferred-logging story's interned-string
  table is built with.
- `elffile` — `ElfFile`, the parsed index. It deliberately does not
  hold the image: every function that answers with bytes takes it back
  from the caller, which costs a parameter per call and lets the same
  index serve a probe that never held a firmware.
- `elferr` — fifteen reasons, each carrying the byte in the image it
  stopped at where there is one. `offset_of` is `?Int` rather than an
  `Int` with a sentinel, so an arm with no place cannot put a plausible
  `0` in somebody's log. `is_truncation` is what tells a caller reading
  in ranges that reading more would help.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  elf-nv.<module>.<fn>`. Run it with `--isolate` for one verdict per
  test naming the function it stopped at.
- Program headers, relocations and DWARF are deliberately absent. The
  README says where each would go.
