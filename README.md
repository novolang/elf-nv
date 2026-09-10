# elf-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The part of ELF a host tool needs: the file header in all four
class-and-endianness combinations, the section header table with names
resolved through the section name table, the symbol table with names
resolved through *its* string table, and a way to say where a section's
bytes are.

Nothing here opens a file.  The image is a `[u8]` the caller read, or —
for a caller that has no image at all — a sequence of byte ranges this
package asks for and the caller answers.  Both work, and the second is
the reason the first does not hold the bytes.

## Adding it, and checking it

```bash
novo pkg add elf-nv           # into your novo.toml
novo pkg build                # type- and effect-check the package
novo test --isolate tests/elfhdr_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: elf-nv.<module>.<fn>`.  They turn
green one at a time as bodies land.

## The one example that will work

```novo
use elffile
use elfsec
use elfsym

// Where a firmware keeps its deferred-log strings, and what they are.
fn interned(image: [u8]) -> Result<[ElfSymbol], ElfError>
    let f = elffile.parse(image)!
    match elfsec.find(f.sections, ".defmt")
        None    => Ok([])
        Some(s) =>
            let all = elfsym.symbols(f.header, f.sections, image)!
            Ok(elfsym.in_section(all, s.index))
```

## The layer, and why

`core`.  An ELF file is bytes somebody already read, and reading it is
arithmetic: fixed-width integers at known offsets, a table walked by a
stride, NUL-terminated strings indexed by byte offset.  There is not one
effect row in the package, and there is no function that could honestly
have one.

**There is no `tests/embedded_probe.nv` in this release, and its absence
is a decision rather than a defect.**  A `core` package's probe is a
claim that the code runs on a microcontroller, and this code has no
business on one: the ELF of a firmware is read by the machine that built
or is debugging it, never by the firmware.  The audit's `core-embedded`
row passes for a package that makes no such claim, which is the right
answer here.

The claim would also not have held today.  `Result<T, E>` cannot be
spelled at `@tier(embedded)` — the `Error` trait is not in the prelude
at that tier [E2005], and SPEC § 3.4 requires the impl [E2018] — which
is `result-is-unusable-at-tier-embedded-no-error-trait`, open against
the toolchain.  Every refusal in this package is a `Result` and stays
one; a package that dropped its error reporting to pass an audit row
would be reporting a toolchain defect as a design.

## The load-bearing interface

One type, and the fact that it means two things at once.

```novo
pub struct ElfRange
    at: Int
    len: Int

pub fn section_range(f: ElfFile, name: Str) -> ?ElfRange
pub fn section_bytes(f: ElfFile, name: Str, image: [u8]) -> Result<?[u8], ElfError>
```

**Where is it, and may I have it, are two questions.**  A `core` package
cannot read, so every answer this package gives about a position is an
`ElfRange` — and an `ElfRange` reads two ways depending on who is
holding it.  To a tool with the whole firmware in a buffer it is a
*span*: `elfrange.slice` cuts it out.  To a probe reading a target's
flash, or a tool that will not hold a 900 MB debug build in memory, it
is a *request*: read `len` bytes at `at` and hand them back.

That is `docs/publishing.md` § How a `core` package takes bytes from its
host's third shape — "the core asks, the host performs" — and ELF is the
format it was written for.  ELF is an index: a table of headers, each
saying where its bytes are, and the entire point of an index is that you
do not read what you did not want.  A streaming parser would throw that
away; a whole-file parser would forbid the probe.

The consequence is the second load-bearing fact:

```novo
pub struct ElfFile
    header: ElfHeader
    sections: [ElfSection]
```

**An `ElfFile` is the index and not the bytes.**  There is no `image`
field, and there will not be one.  Every function that answers with
bytes takes the image back from the caller, which costs a parameter at
each call and buys two things: the same `ElfFile` works for the caller
who never had a file, and a `core` package is never quietly holding a
megabyte on somebody's behalf.  `elffile.from_parts` is how the
range-reading caller arrives at one, after which every query works
identically for both.

The third is that the header is parsed in **two steps**, because an ELF
header cannot be parsed in one: its own size and its own byte order are
inside it.

```novo
pub fn ident(image: [u8]) -> Result<ElfIdent, ElfError>
pub fn header_range(id: ElfIdent) -> ElfRange
pub fn header_from(id: ElfIdent, head: [u8], at: Int) -> Result<ElfHeader, ElfError>
```

`e_ident` is sixteen bytes with a fixed layout and no endianness;
everything after it is laid out according to what those sixteen bytes
said.  A caller with the file calls `header` and never sees the seam.  A
caller reading ranges *needs* it — it has sixteen bytes and has to know
how many more to ask for before it can ask.

## The two places a file lies about its own shape

Both are resolved here rather than left to the caller, because a caller
that did not know about them reads a table of length zero out of a file
with forty thousand sections and reports nothing wrong.

- A file with more than 65,279 sections writes `e_shnum = 0` and puts
  the real count in section 0's `sh_size`.
- A file whose section name table is past index 65,279 writes
  `e_shstrndx = SHN_XINDEX` and puts the real index in section 0's
  `sh_link`.

Section 0 is otherwise entirely zero, and holding those two escapes is
its whole purpose.

## What is deliberately not here

Three parts of ELF are absent on purpose, and each has a place it will
go if it is ever wanted.

- **Program headers.**  `ElfHeader` says where they are and how many —
  `program_table`, `program_entry_size`, `program_count` — and does not
  parse them.  They describe how a file is *loaded*, which matters to a
  loader and to a flashing tool and to nothing that reads symbols.  A
  flashing tool that wanted them would take a `elf-load-nv` beside this
  one, sharing `ElfRange` and `ElfHeader`, rather than making every
  consumer of a symbol table carry a segment parser.
- **Relocations.**  `SHT_REL` and `SHT_RELA` sections are found and
  named like any other, and their contents are not decoded.  Relocations
  are a linker's business and their format is per-architecture — the Arm
  types alone fill a document — so a port of them belongs in whatever
  package needs to link, not here.
- **DWARF.**  `.debug_info` and its siblings are sections like any
  other, and this package hands over their ranges.  DWARF is a
  substantial format in its own right and its natural home is a
  `dwarf-nv` that takes elf-nv as a dependency, which is exactly how
  `gimli` sits on top of `object` upstream.

Also absent, and worth saying: **no writing.**  This package reads.  A
tool that patches a section's bytes has the range and the image and can
do it itself; a tool that *builds* an ELF wants a different package with
a different shape.

## What `Int` costs

novo-lang's `Int` is 64 bits and signed, and ELF's `Elf64_Addr` and
`Elf64_Xword` are 64 bits and unsigned.  A value at or above 2^63 —
a kernel address with the top bit set, a size no file has — does not fit,
and this package refuses it as `ElfValueTooWide` naming the field rather
than handing back a negative number that looks like an address.

For everything this package exists to read the question does not arise:
a firmware's addresses are 32-bit, and a host executable's are far below
2^63.  The refusal is there so that the one file where it does arise
says so.

## The reference implementation

`object` (MIT/Apache) for the shape of the API — its separation of a
file's *parsed index* from the data behind it is what `ElfFile` is — and
`goblin` (MIT) for the parsing itself.  The ELF specification (Tool
Interface Standard, and the System V ABI's processor supplements) is the
grammar and the source of the test vectors, along with `readelf`'s
output on real firmware.

Three things change in the port.  `object`'s lifetime-parameterised
borrowing — `&'data [u8]` threaded through every type — becomes
`ElfRange`, because novo-lang has no borrowing slice and because the
range is what the probe-side caller needed anyway.  `goblin`'s
"parse everything at once" `Elf::parse` becomes `elffile.parse` plus a
sequence of range functions, so the two callers share a parser rather
than getting two.  And the section-name and symbol-name string tables
become one primitive, `elfsec.string_at`, rather than two nearly
identical walkers — with `elfsym.name_table` as the function whose only
job is to make it impossible to read symbol names out of the section
name table, which is the mistake that produces names that look almost
right.

## Status

| item | implemented |
| --- | --- |
| `elfrange` — `ElfRange` | type only |
| `elfrange.range`, `.empty`, `.range_end`, `.range_fits`, `.slice`, `.sub` | no |
| `elferr` — `ElfError` | type only |
| `elferr.offset_of`, `.is_truncation`, and the `Error` impl | no |
| `elfhdr` — `ElfClass`, `ElfEndian`, `ElfKind`, `ElfMachine`, `ElfIdent`, `ElfHeader` | types only |
| `elfhdr.is_elf`, `.ident_range`, `.ident`, `.header_range`, `.header`, `.header_from` | no |
| `elfhdr.class_bits`, `.section_entry_size`, `.symbol_entry_size`, `.machine_name`, `.kind_name` | no |
| `elfsec` — `ElfSection`, `ElfSectionKind` | types only |
| `elfsec.section_table_range`, `.sections`, `.sections_unnamed`, `.section_table_range_of` | no |
| `elfsec.name_table_range`, `.with_names`, `.find`, `.find_all`, `.at`, `.string_at` | no |
| `elfsec.kind_name`, `.is_allocated` | no |
| `elfsym` — `ElfSymbol`, `ElfSymbolBind`, `ElfSymbolKind` | types only |
| `elfsym.symbol_table`, `.dynamic_symbol_table`, `.name_table`, `.symbols`, `.symbols_from` | no |
| `elfsym.find`, `.in_section`, `.with_prefix`, `.bind_name`, `.kind_name` | no |
| `elffile` — `ElfFile` | type only |
| `elffile.parse`, `.from_parts`, `.section`, `.sections`, `.section_at` | no |
| `elffile.section_range`, `.section_bytes`, `.symbols`, `.symbol` | no |
| `elffile.class`, `.endian`, `.machine`, `.entry`, `.section_count` | no |
