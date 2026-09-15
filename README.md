# elf-nv

ELF is the file format compilers and linkers produce on Linux and on
most embedded targets: an executable, a shared library, an object file
or a firmware image. It is specified by the Tool Interface Standard's
[ELF specification](https://refspecs.linuxfoundation.org/elf/elf.pdf)
and the System V ABI's processor supplements. This package reads the
part of it a host tool needs: the file header, the section table with
names resolved, and the symbol table.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What an ELF file is

An ELF file opens with sixteen bytes called **e_ident**. They have a
fixed layout and no byte order of their own: four bytes of magic, then
the **class**, which says whether addresses are four bytes or eight, and
the **data** byte, which says whether multi-byte fields are
little-endian or big-endian. Everything after those sixteen bytes is
laid out according to what they said.

The rest of the file header says what the file is for, which machine it
is for, where execution starts, and where two tables live. The
**program header table** describes how the file is loaded into memory.
The **section header table** describes the file's contents as named
pieces: `.text` for code, `.data` for initialised data, `.symtab` for
the symbol table, `.debug_info` for debugging information.

A **section header** is a fixed-size record giving a section's kind, its
flags, its address at run time, and the offset and length of its bytes
in the file. Its name is not in the record. The record holds a byte
offset into a **string table**, which is a section whose contents are
NUL-terminated strings, and the header says which section that is.

A **symbol** is a fixed-size record in `.symtab` giving a name offset, a
value, a size, a binding, a kind and the section it belongs to. Its
names live in a different string table, named by the symbol table
section's own link field.

| Quantity | Value |
| --- | --- |
| Bytes of `e_ident` | 16 |
| Magic | `0x7F` `E` `L` `F` |
| File header size, 32-bit and 64-bit | 52 and 64 bytes |
| Section header size, 32-bit and 64-bit | 40 and 64 bytes |
| Symbol record size, 32-bit and 64-bit | 16 and 24 bytes |
| ELF version, since 1995 | 1 |
| Highest ordinary section index | 65279 |
| Section index meaning "none" | 0 |
| Section index meaning "absolute" | `0xFFF1` |

## Install

```
novo pkg add elf-nv
```

## Example

```novo
use std.bytes
use std.fs
use elffile
use elfsym

fn main() [io, fs]
    match fs.read_bytes("firmware.elf")
        None      => println("no such file")
        Some(raw) =>
            // This package takes the image as a list of bytes. It never
            // reads a file itself.
            let image = bytes.to_byte_list(raw)

            match elffile.parse(image)
                Err(e) => println(e.message())
                Ok(f)  =>
                    // The section count, from the parsed index.
                    println("${elffile.section_count(f)} sections")

                    // One symbol by name. The image is handed back,
                    // because the parsed file does not hold it.
                    match elffile.symbol(f, image, "main")
                        Err(e2)     => println(e2.message())
                        Ok(None)    => println("no symbol called main")
                        Ok(Some(s)) => println("${s.value}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: elf-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `elfrange` | A place in the file: an offset and a length, with the arithmetic for cutting one out of an image and for taking a piece of one. |
| `elferr` | The thirteen refusals, each carrying the offset it was found at, and the two questions a caller asks of one. |
| `elfhdr` | The first sixteen bytes and the header they describe, in all four combinations of class and byte order, plus the record sizes and the names of the machines and file kinds. |
| `elfsec` | The section header table: where it is, how to walk it with or without names, how to find a section by name or index, and how to read a string out of a string table. |
| `elfsym` | The symbol table and the dynamic symbol table: which section each is, which string table holds its names, how to read the records, and how to select symbols by name, section or prefix. |
| `elffile` | The two together as one value, for a caller that has the whole image: parse it once, then ask for sections, ranges, bytes and symbols. |

## How to choose an entry point

**`elffile.parse` is for a caller holding the whole image.** It answers
a value carrying the header and every named section, and the rest of
`elffile` asks questions of it.

**`elfhdr.ident`, `elfhdr.header_range` and `elfhdr.header_from` are for
a caller that has no image.** A debug probe reading a target's flash, or
a tool that will not hold a large debug build in memory, asks for a
range, reads it, and hands the bytes back. `elfsec.section_table_range`,
`elfsec.sections_unnamed`, `elfsec.name_table_range` and
`elfsec.with_names` continue the same walk, and `elffile.from_parts`
turns the result into the same value `parse` answers.

**`elffile.section_range` answers where something is, and
`elffile.section_bytes` answers what it is.** The first needs no image.

## The rules a user needs

1. **A parsed file is the index, not the bytes.** `ElfFile` has no image
   field. Every call that answers bytes takes the image back from the
   caller. That is what lets one type serve both the caller who has the
   file and the caller who never had it.
2. **A range is a place, and it reads two ways.** To a caller holding
   the image it is a span, and `elfrange.slice` cuts it out. To a caller
   reading a target it is a request: read this many bytes at this offset
   and hand them back.
3. **The header is parsed in two steps, because it describes itself.**
   Its size and its byte order are inside it. `elfhdr.ident` reads the
   sixteen bytes, `elfhdr.header_range` says how many more are needed,
   and `elfhdr.header_from` reads them. `elfhdr.header` is the two in
   one call for a caller with the file.
4. **All four combinations of class and byte order are read.** 32-bit
   and 64-bit, little-endian and big-endian. A tool that read only
   32-bit little-endian would cover every Cortex-M firmware and fail on
   the first 64-bit RISC-V board.
5. **A file with more than 65279 sections says it has none.** The real
   count is in section 0's size field, and this package resolves it. A
   caller that did not know would read an empty table out of a file with
   forty thousand sections and report nothing wrong.
6. **A file whose name table is past index 65279 hides that index
   too.** It is in section 0's link field, and this package resolves
   that as well. Section 0 is otherwise entirely zero, and holding those
   two escapes is its purpose.
7. **Section names and symbol names come from different string
   tables.** `elfsym.name_table` is the function that answers the right
   one for a symbol table. Reading symbol names out of the section name
   table produces names that look almost right.
8. **Two sections may share a name.** `elfsec.find` answers the first
   and `elfsec.find_all` answers all of them.
9. **A missing section is not a fault.** `elffile.section` and
   `elfsec.find` answer nothing. A missing symbol table is a refusal,
   because a caller reaching for symbols has already decided it needs
   them, and a stripped release build is the ordinary way to have none.
10. **A section with no file bytes has an empty range.** `.bss` occupies
    memory at run time and nothing in the file.
11. **A 64-bit field above `Int`'s range is refused.** novo-lang's `Int`
    is signed and 64 bits wide, and ELF's addresses and sizes are
    unsigned. `ElfValueTooWide` names the field rather than answering a
    negative number that looks like an address. A firmware's addresses
    are 32-bit and a host executable's are far below the limit, so this
    is about the one file where it does arise.
12. **A name that is not UTF-8 is refused as a conversion, not as a
    file.** ELF names are bytes and the specification says nothing about
    their encoding. A caller that wants those bytes takes the section's
    range and reads them.
13. **A string table offset that never meets a NUL is refused.**
    `ElfUnterminatedString` names the offset.

## What is not included

- **Program headers.** The file header says where they are and how many,
  and this package does not parse them. They describe how a file is
  loaded, which matters to a loader and to a flashing tool and to
  nothing that reads symbols.
- **Relocations.** `SHT_REL` and `SHT_RELA` sections are found and named
  like any other, and their contents are not decoded. The record types
  are per-architecture, and the Arm ones alone fill a document.
- **DWARF.** `.debug_info` and its siblings are sections like any other,
  and this package hands over their ranges. DWARF is a large format and
  belongs to a package that depends on this one.
- **Writing.** This package reads. A tool patching a section's bytes has
  the range and the image and can do it itself.
- **A build for a microcontroller.** A firmware's ELF is read by the
  machine that built it or is debugging it, never by the firmware, so
  this package makes no device claim and carries no probe. The claim
  would not hold today in any case: every refusal here is a `Result`,
  and a `Result` cannot be spelled at the embedded tier, because the
  `Error` trait is not in that tier's prelude.
- **Any input or output.** Nothing here opens a file. The image arrives
  as a list of bytes, or as answers to range requests.

## Related packages

- [deflog-parser](https://novo-lang.org/packages/deflog-parser) and
  [deflog-decoder](https://novo-lang.org/packages/deflog-decoder) read a
  firmware's interned log strings, which live in a section of its ELF
  and are addressed by symbol.
- [rzcobs-nv](https://novo-lang.org/packages/rzcobs-nv) frames the log
  stream those strings are reconstructed from.
- [dfu-nv](https://novo-lang.org/packages/dfu-nv) writes firmware onto a
  device. It needs the program headers this package does not parse.
- [archive-nv](https://novo-lang.org/packages/archive-nv),
  [tar-nv](https://novo-lang.org/packages/tar-nv) and
  [zip-nv](https://novo-lang.org/packages/zip-nv) are the other
  container formats on the registry. Each holds files; ELF holds
  sections of one program.

## Tests

```bash
novo test tests/elfhdr_tests.nv       #  9 tests: the first sixteen bytes and the header
novo test tests/elffile_tests.nv      # 11 tests: sections, symbols and ranges
novo test tests/elfwalk_tests.nv      # 11 tests: the walk a caller with no image performs
```

The grammar and the vectors come from the ELF specification and the
System V ABI's processor supplements, checked against `readelf`'s output
on real firmware. The API shape follows the `object` crate in Rust,
whose separation of a parsed index from the data behind it is what
`ElfFile` is, and the parsing follows `goblin`.

The suite asserts that the magic is four bytes and no more, that the
identification is the first sixteen, that the range shape needs no
image, that both widths have a stride the package knows, that a buffer
which is not an ELF is refused at byte zero, that a truncation says how
much was missing, that a parsed file does not hold the image, that a
place is answerable without the bytes, that a section with no file bytes
has an empty range, that an absent section is not a fault and an absent
index is, that a name is read out of a string table by byte offset, that
a string which never ends is refused, that a symbol names its own string
table, that the dynamic symbol table is not the symbol table, and that
the whole-file walk and the range walk answer alike.

The tests compile today and fail at run, each on the `not implemented`
panic that is its body. That is the expected state of an interface
release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `elfrange.range`, `.empty`, `.range_end`, `.range_fits`, `.slice`, `.sub` | no |
| `elferr.offset_of`, `.is_truncation`, `ElfError.message` | no |
| `elfhdr.is_elf`, `.ident_range`, `.ident`, `.header_range`, `.header`, `.header_from` | no |
| `elfhdr.class_bits`, `.section_entry_size`, `.symbol_entry_size`, `.machine_name`, `.kind_name` | no |
| `elfsec.section_table_range`, `.sections`, `.sections_unnamed`, `.section_table_range_of` | no |
| `elfsec.name_table_range`, `.with_names`, `.find`, `.find_all`, `.at`, `.string_at` | no |
| `elfsec.kind_name`, `.is_allocated` | no |
| `elfsym.symbol_table`, `.dynamic_symbol_table`, `.name_table`, `.symbols`, `.symbols_from` | no |
| `elfsym.find`, `.in_section`, `.with_prefix`, `.bind_name`, `.kind_name` | no |
| `elffile.parse`, `.from_parts`, `.section`, `.sections`, `.section_at` | no |
| `elffile.section_range`, `.section_bytes`, `.symbols`, `.symbol` | no |
| `elffile.class`, `.endian`, `.machine`, `.entry`, `.section_count` | no |

The types themselves — `ElfRange`, `ElfError`, `ElfIdent`, `ElfHeader`,
`ElfSection`, `ElfSymbol`, `ElfFile` and the enumerations beside them —
are declared with their fields and are what a reviewer reads.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
