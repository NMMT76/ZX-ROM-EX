# ZXROMEX_ATTRIBUTIONS.md

Required credits for everything ZXROMEX builds on top of. This file
should ship alongside the ROM and source in every release — several of
these are not optional courtesy notes but actual conditions of reuse.

---

## 1. The Amstrad 48K ROM itself

Amstrad plc retains copyright over the original Sinclair/Amstrad 48K
ROM. Modification and redistribution of modified ROMs is explicitly
permitted, per:

> Amstrad have kindly given their permission for the redistribution of
> their copyrighted material but retain that copyright.

— Cliff Lawson, Amstrad plc, comp.sys.amstrad.8bit, 31 August 1999.

This permission covers modification; it does not transfer copyright.
Amstrad's copyright over the original ROM applies regardless of how
extensively it has been patched.

## 2. Base disassembly

ZXROMEX is built on top of the `Spectrum 48K/Spectrum48.asm`
disassembly from:

**ZXSpectrumVault/rom-disassemblies**
`https://github.com/ZXSpectrumVault/rom-disassemblies`

Original ROM authors: Steve Vickers & Richard Altwasser, Nine Tiles,
1981.

Disassembly annotators (per the disassembly file's own credit list):
Dr. Ian Logan, Peter Liebert-Adelt, Geoff Wearmouth, Dr Frank O'Hara,
Wilf Rigter, Matthew Wilson, Andrew Owen, Rui Tunes, Paul Farrow, Garry
Lancaster, Ian Collier, Sean Irvine, Mike Dailly, Alvin Albrecht, Andy
Styles.

## 3. ZX0 decompressor (`LLIST(0)` in this ROM)

**ZX0 Standard decompressor**, by Einar Saukas & Urusergi.
`https://github.com/einar-saukas/ZX0`

The embedded `DZX0_STANDARD` routine (`$3C08` in the current build) is
the unmodified upstream "Standard" variant — confirmed byte-for-byte
identical to the authors' own source (verified directly against a copy
of `dzx0_standard.asm` obtained from the author's repository; the only
differences found were internal `CALL` target addresses shifting by the
expected relocation delta when placed at a different ROM address — pure
relocation, zero content change).

Crediting this repository by name and URL in project documentation is
the authors' own stated requirement for reuse, not just a courtesy —
this file, and any release notes or README referencing ZX0 support,
should retain this credit.

## 4. `NEW_SQR` (fast integer-seeded Newton-Raphson square root)

Ported byte-for-byte from Britlion's `fsqrt.bas` (Boriel ZX BASIC
library), including its exponent-guess heuristic.
`https://zxbasic.readthedocs.io/en/docs/library/fsqrt.bas/`

## 5. `NEW_RND_BYTE` (the CMWC random number generator behind `RND`)

Algorithm design: Patrik Rak. Surfaced for this project via Britlion's
`RandomStream.bas`, from which the specific implementation used here was
adapted.

---

## Where this applies

Every one of the above is embedded in, or directly derived from, code
currently shipping in the ZXROMEX ROM image (`ZXROMEX_v1.3.bin` and its
predecessors back through v1.0). None of these credits are satisfied by
a passing mention buried in a changelog — this file exists so there's
one canonical place listing all of them together, and it should be kept
alongside the ROM binary and source in any distribution, not just in the
project's own internal documentation.
