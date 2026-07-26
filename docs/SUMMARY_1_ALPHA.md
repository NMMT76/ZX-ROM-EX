# Checkpoint Alpha — "Free space isn't free"

**Date:** 2026-07-19
**Target:** Sinclair ZX Spectrum 48K ROM (Amstrad/Sinclair `Spectrum48.asm`
as ground truth)
**Deliverable:** `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_LN_EXP_CIRCLE2_patched.asm`

## What this checkpoint contains

A from-scratch, cumulative rewrite of every "slow" math primitive in the
48K ROM (`SQR`, `SIN`, `COS`, `TAN`, `ATN`, `LN`, `EXP`, plus `ASN`/`ACS`/
`^` which inherited speedups for free), one new BASIC capability
(`ADDR(var)`), one replaced random-number generator (`RND`, now matching
Boriel BASIC's own CMWC algorithm), and one graphics command rewritten
from a slow floating-point algorithm to an integer one (`CIRCLE`, Midpoint
Circle Algorithm). Along the way: neutered the printer channel to claim
256 bytes of real RAM as scratch space, confirmed one small piece of
dead code, and worked the ROM's free-space budget down from 1170 bytes
to 290 — hence the codename.

Twelve patches total. Full narrative history, including every dead end
and reconsideration, is in `Decisions.md`. This summary is the map to
that document plus the structural reference (`FUNCTION_SPECS.md`) and
the test artifacts, all included in this archive.

## Project discipline (applies to every patch)

1. **Baseline first.** Every patch starts from a verified, unmodified
   ROM assembly (`baseline.bin`), byte-for-byte checked against a known
   checksum before any edit.
2. **Read before replacing.** Every original routine was read and
   understood — its calling convention, its side effects, what else in
   the ROM depends on it — before any replacement was designed. This
   caught several cases where a naive "just swap the algorithm" plan
   would have broken something else (`DRAW` sharing code with `CIRCLE`;
   `ASN`/`ACS` already composing onto `SQR`/`ATN`; `to-power` composing
   onto `LN`/`EXP`).
3. **Standalone assembly before integration.** Every new routine is
   written and assembled on its own first (with `pasmo`), its exact
   byte layout dumped and manually verified against the intended
   design, before being spliced into the full ROM.
4. **Full byte diff, every single patch.** After every change, the
   complete 16KB ROM is diffed against baseline. Every differing byte
   must be individually explained — which patch, which routine, why.
   Unexplained differences are never accepted as "probably fine."
5. **Dynamic verification, not just static.** Every patch is actually
   *run* in an emulator, not just diffed. This caught real bugs that a
   byte-level check alone would have missed entirely (see "Bugs found
   and overcome" below) — this project's own standing lesson, stated
   explicitly after the `ATN` incident and followed rigorously
   afterward.
6. **Real-hardware test files.** Every function with observable BASIC
   behavior has an independent `.bas`/`.tap` test program — standalone,
   no cross-dependencies on other patched functions — meant to be run
   on real hardware (or Fuse) under both the original and patched ROM
   for direct comparison.

## Test harness

`rom_harness.py`, built on the `kosarev/z80` Python package (a C++-core,
Python-bound Z80 emulator that passes zexall/zexdoc/cputest/8080pre/
8080exer/8080exm — chosen after an earlier harness, `z80js`, was found
to have a real bug: `SET n,(IY+d)`-class indexed-bit instructions
advanced `PC` incorrectly, silently desyncing execution for anything
using `LOOK-VARS`-style indexed addressing).

Key harness facilities:
- `load_rom`, `new_machine` — load a ROM image, get a fresh machine.
- `run_calculator_literal` — the workhorse for testing calculator
  literals (`SIN`, `COS`, `LN`, etc.) directly: sets up `STKEND`,
  `MEM`, a minimal program (`[literal_byte, end-calc]`), runs it, reads
  back the result.
- `run_until` — breakpoint-based execution to a set of target
  addresses, replacing an earlier PC-polling step loop.
- `encode_float`/`decode_float`/`encode_short_int` — ZX Spectrum 5-byte
  floating-point and short-integer format conversion.
- `setup_basic_environment` — full system-variable setup for tests that
  need more than the calculator alone (used for `ADDR`, `RND`,
  `CIRCLE`).
- **`protect_rom` (added during Patch 7):** a write callback that
  silently discards any real Z80 instruction's write to an address
  below `16384`, matching real hardware's documented no-op behavior for
  ROM writes. Added specifically because Patch 7's correctness *depends*
  on this behavior (a "guaranteed always zero" byte in ROM), and no
  earlier patch had happened to need it — the gap in the harness went
  unnoticed until a patch's correctness actually relied on it, then was
  fixed properly (confirmed the callback only intercepts real
  CPU-driven writes, not direct test-setup memory pokes) rather than
  worked around.

## Test methodology by category

- **Calculator-literal functions** (`SQR`/`SIN`/`COS`/`TAN`/`ATN`/`ASN`/
  `ACS`/`LN`/`EXP`): reference values from Python's `math` module,
  compared against the patched ROM's actual output across wide value
  ranges (typically hundreds of test points per function), plus
  boundary/exact-value checks (`sin(0)=0`, `ln(1)=0`, etc.) and error-
  path checks (domain violations still raise the correct report code).
- **`ADDR`**: direct dispatch-routine testing with a constructed BASIC
  environment (variables area, program text), covering scalar and array
  variable references, plus the syntax-check-vs-runtime distinction that
  turned out to matter (see bugs below).
- **`RND`**: statistical (distribution, range) plus exact reference-
  model comparison — a custom Python model of the CMWC algorithm,
  verified independently against the original self-modifying-code
  algorithm before being used as the ground truth for the ROM
  implementation.
- **`CIRCLE`**: pixel-exact comparison against a Python reference
  implementation of the same algorithm, using the real ROM's own
  `PIXEL-ADD` routine (called live in the emulator) as ground truth for
  screen-address/bit computation, plus `OVER 1` toggle-consistency and
  off-screen clipping checks.
- **Regression suite**: every one of the above re-run after every
  subsequent patch, without exception, for the life of the project.

## Bugs found and overcome (project-wide, chronological)

This list is deliberately blunt — every one of these was a real mistake
caught by the process above, not a hypothetical risk. Full detail for
each is in `Decisions.md`; this is the index.

1. **`z80js` harness bug** — indexed-bit instructions (`SET n,(IY+d)`)
   advanced `PC` incorrectly. Found by an isolated 4-instruction test,
   not by a failing patch test — caught proactively. Replaced the whole
   harness with `kosarev/z80`.
2. **`ADDR` array-path INC HL** — a "fix" based on a ROM comment's
   literal wording (making an increment conditional) was disproved by
   real-hardware testing showing a consistent off-by-one; the original,
   unconditional design was correct and the comment had been
   misread. Static reasoning about a comment was wrong; real execution
   was right.
3. **`ADDR` syntax-check safety** — `LOOK-VARS` doesn't perform a real
   search during the automatic syntax-check pass that runs when a typed
   line's ENTER is pressed; `HL` isn't meaningfully set then. The
   original design trusted it unconditionally, which could fail a
   perfectly valid line at entry time despite working fine once loaded
   from tape (which bypasses that pass). Found by the user reporting a
   real, reproducible symptom, not by internal review.
4. **`ADDR` string/numeric check** — found *while fixing* bug 3, by
   testing the fix dynamically rather than trusting the design:
   `CP $C0` tested two `FLAGS` bits together, one of which is always
   clear during syntax-checking regardless of the actual variable
   type, making the numeric path unreachable then independent of bug
   3's fix.
5. **`ATN` missing `exchange`** — the original `CASES` block's own first
   instruction (an `exchange`) was silently dropped when the whole
   block was replaced. Standalone byte-level verification passed
   cleanly (internally consistent, wrong assumption); only caught by
   actually running it. This incident is this project's own explicit
   justification for "always run it, not just diff it."
6. **`PRBUFF` neutering, harness gap** — `rom_harness.py` never modeled
   ROM write-protection; Patch 7 was the first patch to actually depend
   on it. Added properly (`protect_rom`), confirmed it doesn't affect
   test-setup pokes, re-ran the full existing suite to confirm no side
   effects from the harness change itself.
7. **`RND` self-modifying code** — the published CMWC algorithm relies
   on patching its own instruction bytes, which cannot work from ROM.
   Caught by design review before any assembly was written (verified
   the redesign against the original algorithm in Python first).
8. **`RND` dead label** (`L2625`) — a trampoline jumped to a label that
   no longer existed once its enclosing block became a trampoline.
   Caught by tracing what the label actually did before assuming the
   jump target was still valid.
9. **`RND` register clobbering** — `NEW_RND_BYTE` uses `B` internally;
   storing a value there across a second call to the same routine
   silently destroyed it. Found by checking actual register values at
   the call site, not by guessing.
10. **`RND` `stk-data` encoding limit** — `65536` overflows the 6-bit
    exponent field in a `stk-data` header byte; no earlier patch's
    constants had been large enough to hit this limit. Found by
    isolating the failure through a sequence of narrowing tests
    (`end-calc` alone, `duplicate` alone, a known-good constant alone)
    rather than guessing at the cause.
11. **`EXP` accuracy claim mismatch** — a previously-quoted `~7e-6`
    accuracy figure had been verified against a centered reduction
    range; the original's actual reduction is one-sided and gives
    2.27e-4 instead. Found by explicitly checking the claim against the
    actual range before trusting it, not by assuming a formula's
    accuracy transfers across different reduction schemes.
12. **`CIRCLE` test-harness address mismatch** — a relocatable routine
    was loaded into RAM at a different address than it was assembled
    for; its own internal `CALL`/`JP` targets were silently wrong.
13. **`CIRCLE` `PIXEL-ADD` reimplementation bug** — a hand-derived
    Python replica of the ROM's own bit-twiddling had its own bug.
    Caught because the *right number* of pixels were being plotted,
    just not at the expected addresses — a strong signal pointing at
    the checker, not the routine. Fixed by calling the real ROM
    routine as ground truth instead of re-deriving it by hand.
14. **`CIRCLE` `P_FLAG` address** — `PLOT-SUB` reads it via `IY`-relative
    addressing, not a fixed address; a test initially poked the wrong
    absolute address and silently tested `OVER 0` instead of `OVER 1`.
15. **`CIRCLE` first implementation size** — 422 bytes for 8 unrolled,
    nearly-identical point-computation blocks, flagged as excessive by
    the user. Refactored to a shared subroutine (`PLOT4`) called twice,
    328 bytes, reverified against the identical test suite with
    identical results.

## Free space accounting

| | ROM (`$386E`-`$3CFF`) | RAM (`PRBUFF`, `$5B00`-`$5BFF`) |
|---|---:|---:|
| Original | 1170 bytes free | 0 bytes owned (real printer buffer) |
| Current (Checkpoint Alpha) | **290 bytes free** | **230 bytes free** (of 256 owned since Patch 7) |
| Confirmed dead, not reclaimed | 15 bytes (`n-mod-m`) | — |

## Archive contents

- `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_LN_EXP_CIRCLE2_patched.asm`
  — the complete patched ROM source, current deliverable.
- `Decisions.md` — full chronological decision log, every patch, every
  bug, every reconsideration, in narrative detail.
- `FUNCTION_SPECS.md` — structural reference: exact addresses, sizes,
  labels, direct references, and padding for every routine touched or
  added.
- `rom_harness.py` — the Python/`kosarev-z80` test harness.
- `*_test.bas` / `*_test.tap` — one independent, standalone real-
  hardware test program per replaced/added function: `sqr`, `sin`,
  `cos`, `tan`, `atn`, `asn`, `acs`, `ln`, `exp`, `addr`, `rnd`,
  `circle`.

## Post-checkpoint correction

**`circle_test.bas` ordering bug**, found by the user actually running
the test before trusting it (exactly the kind of check this project's
own discipline calls for, applied here to a test file rather than the
ROM itself): the original layout ran the off-screen clipping section
(Part 2) before the performance timing section. Clipping is
patched-ROM-only behavior — the original ROM has no error handling for
a circle that goes off-screen, so it halts the whole program with an
unhandled `Report B` ("Integer out of range") partway through Part 2,
meaning the test *never reached* the performance comparison when run on
the original ROM, defeating the point of a side-by-side test.

Fixed by reordering: basic circles, `OVER 1`, `DRAW`-still-works, and
performance timing all run first (all safe on both ROMs — every `CIRCLE`
call in these sections checked to stay fully within `[0,255]x[0,175]`
for its given centre and radius), with off-screen clipping moved to a
clearly-labeled final section that explicitly tells the user it's
patched-ROM-only and that an error there under the original ROM is
expected, not a bug. This is a test-file fix, not a ROM change — no
`.asm` or `Decisions.md` update needed, only `circle_test.bas`/`.tap`.

**Second correction, same file — `PRINT AT` row bug, unrelated to
`CIRCLE`'s own clipping.** After the reordering fix above, the user
still hit `5 Out of screen` on the *patched* ROM, in what looked at
first like it might be a real clipping bug. Traced precisely rather
than assumed: `Report 5` here is raised by `PO-AT-ERR` (`$0AAC`),
`PRINT AT`'s own row-validation logic — completely unrelated to
graphics pixel coordinates. With the default `DF_SZ=2` (2 lines
reserved for the lower screen), valid `PRINT AT` rows are `0`-`20`
only, not `0`-`23` as assumed when writing the "Part 5 passed" message
— the test itself used `PRINT AT 22,0` and `PRINT AT 23,0`, both
invalid. Confirmed this is unrelated to `CIRCLE`'s clipping logic (which
appears to be working correctly — the error fires *after* the clipped
circles have already drawn without incident, at the final status
message). Fixed by moving those two lines to rows `19`/`20`. Scanned
every other `PRINT AT` call in the file to confirm no other instance
of the same mistake. Test-file fix only, same as above.

## Post-checkpoint addition: consolidated benchmark

**`benchmark_all.bas`/`.tap`** — a single unattended program covering
every patched function's performance (not correctness — that's what
the twelve individual `*_test.bas` files are for). Runs back-to-back
with no `PAUSE`/key-press waits, printing progress as it goes, then a
final summary table with exactly one timing result per function.

Each calculator function loops over 5 mixed values (spanning small,
large, negative, and boundary-ish cases per function's own domain),
repeated 20 times (100 calls total); `RND` runs 100 bare calls;
`^`/`to-power` loops over 5 mixed base/exponent pairs, 20 times; `CIRCLE`
loops over 5 different centres/radii, 4 times (20 draws total, radii
kept modest -- max `30` -- specifically so the original ROM's own much
slower algorithm still completes in reasonable time rather than making
the benchmark impractically slow to run twice for comparison).

**Verified before delivery, not assumed safe:** every one of the 45
calculator-function data values confirmed to run without error on the
patched ROM. The `CIRCLE` data set was checked by hand for the same
off-screen-halts-the-original-ROM failure mode as the `circle_test.bas`
bug above -- and one WAS found this way before delivery (`(10,10,20)`
has `x-r=-10`, invalid) and corrected to `(30,30,20)` before this file
was ever shipped, rather than being discovered later by re-running it.
