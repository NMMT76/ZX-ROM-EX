# Checkpoint Gamma — "Nothing left on the table"

**Date:** 2026-07-20
**Target:** Sinclair ZX Spectrum 48K ROM (Amstrad/Sinclair `Spectrum48.asm`
as ground truth)
**Deliverable:** `checkpoint_gamma_Spectrum48_patched.asm`

## What this checkpoint contains

Checkpoint Gamma packages the finished output of the Beta repacking
effort as a clean, stable new baseline. It contains exactly the same
math, exactly the same behavior, and exactly the same external
interface as Checkpoint Alpha — every function (`SQR`, `SIN`, `COS`,
`TAN`, `ASN`, `ACS`, `ATN`, `LN`, `EXP`, `ADDR`, `RND`, `CIRCLE`) is
byte-for-byte the same algorithm, verified repeatedly to be
behaviorally identical to Alpha. What changed between Alpha and Gamma
is purely **where things live in the ROM**: seven routines that Alpha
left as `JP`-trampolines into the main free block have been relocated
into previously-wasted trampoline padding elsewhere in the ROM,
reclaiming that space for the contiguous free block instead.

**Net result: 290 → 563 bytes of contiguous free ROM space** — a 94%
increase — achieved with zero algorithm changes, zero accuracy changes,
and zero behavioral differences, each verified independently at every
step.

Full narrative history of how this was done — every patch, every bug
found and fixed, every verification run — is in `checkpoint_BETA_Decisions.md`
(nine entries: one padding audit, one best-fit planning pass, seven
implemented relocation patches). This summary is the map to that
document plus the structural reference (`checkpoint_gamma_FUNCTION_SPECS.md`)
and the current deliverable, all included in this archive.

## Project discipline (inherited from Alpha, held throughout Beta)

Every one of the seven relocation patches in this checkpoint followed
the same discipline Alpha established:

1. **Baseline first.** Every patch started from the previous patch's
   own verified, fully-tested deliverable — never from a hand-edited
   copy that skipped verification.
2. **Complete `ORG` sweep before editing**, not just a search near the
   expected change site. This specific discipline was *added* during
   Beta, in direct response to two stale-`ORG` bugs (Entries 3, 5) that
   a narrower search missed. Every relocation from Entry 6 onward used
   the complete sweep and found zero new stale-`ORG` bugs.
3. **Full byte diff, every single patch**, with every differing byte
   individually explained — the great majority in this checkpoint are
   downstream operand references shifting by a predictable, exactly-
   verified amount as routines moved.
4. **The free-block boundary checked directly**, not inferred from
   arithmetic — and, after Entry 8's `$FF`-counting bug slipped past
   the original boundary-scan method, a *stricter* check was added:
   every byte in the claimed free range individually confirmed to be
   `$FF`, with no method that could hide a gap by skipping past it.
5. **Dynamic verification on every patch**, via A/B comparison against
   the immediately preceding, already-verified build — not just
   against Alpha once at the start. Over 5,200 calculator-literal
   comparisons were re-run, bit-for-bit, at every single one of the
   seven patches.

## Bugs found and fixed during Beta (both caught by process, not luck)

1. **Two stale-`ORG` bugs** (Entries 3, 5): a hardcoded `ORG` statement
   downstream of a relocated routine left pointing at the *old* address
   layout, silently creating an implicit zero-filled gap instead of
   genuine reclaimed space. Both assembled clean; both were caught only
   by directly checking the free-block boundary rather than trusting a
   successful build. Fixed the immediate bug each time, and — from
   Entry 6 onward — fixed the *process* that let it happen (a complete
   file-wide `ORG` sweep before any edit).
2. **A hand-counting mistake in the `$FF` fill block** (Entry 8): 67
   claimed bytes of explicit free-space fill turned out to be only 59,
   an 8-byte implicit gap hiding at the block's edge. The standard
   boundary check didn't catch it (it has its own blind spot: skipping
   past non-`$FF` bytes before counting). Caught by an unexplained diff
   right at the character-bitmap-table boundary. Fixed by regenerating
   the fill block programmatically (byte count verified in Python
   before being written into source) rather than hand-typed with a
   running comment tally — and, in Entry 9, the entire fill block was
   regenerated cleanly from scratch rather than patched an eighth time.
3. **A file-delivery bug outside the ROM itself**: one deliverable
   (Entry 3's first attempt) was built from a pasmo-testing working
   copy with the file's TASM `#define` directives commented out, and
   shipped that way — silently broken for the file's actual intended
   assembler. Caught by the user, not by this project's own process.
   Fixed, confirmed byte-identical to the already-tested binary once
   the directives were restored, and added as a standing checklist item
   for every subsequent patch.

## Free space accounting

| | ROM (main free block) | ROM (scattered trampoline slack) |
|---|---:|---:|
| Checkpoint Alpha | 290 bytes | 328 bytes (10 locations, unused) |
| Checkpoint Gamma | **563 bytes** | **11 bytes** (3 tiny, unusable pads) |

| Entry | What moved | Bytes reclaimed | Running total |
|---|---|---:|---:|
| Alpha (starting point) | — | — | 290 |
| Beta Entry 3 | `NEW_EXP_CORE` → own trampoline slot | 52 | 342 |
| Beta Entry 4 | `NEW_ATN_CORE` → own trampoline slot | 51 | 393 |
| Beta Entry 5 | `NEW_COS_CORE` → `C-ENT`'s spare padding | 30 | 423 |
| Beta Entry 6 | `NEW_LN_TAIL` → own trampoline slot | 36 | 459 |
| Beta Entry 7 | `NEW_S_RND` → own trampoline slot | 29 | 488 |
| Beta Entry 8 | `NEW_TAN_CORE` → `CIRCLE`'s spare padding | 67 | 555 |
| Beta Entry 9 | `RND_DEFAULT_TABLE` → `COS`'s spare padding | 8 | **563** |

Three residual trampoline slack pockets remain, all confirmed too small
for any existing routine and not worth reclaiming for a few bytes each:
`SQR` (4 bytes), `TAN`'s own trampoline (5 bytes), `RANDOMIZE`'s tail
(2 bytes). `n-mod-m` (15 bytes, confirmed dead code since Alpha) remains
a deliberately deferred option, not reclaimed, for the same reason
Alpha left it alone — it's still a technically-available direct-dispatch
calculator literal for any user machine-code program.

## What moved where (quick reference)

| Routine | Alpha location | Gamma location | Fit |
|---|---|---|---|
| `NEW_EXP_CORE` | main block | own trampoline (`$36C4`) | 52/53 bytes, 1 slack |
| `NEW_ATN_CORE` | main block | own trampoline (`$37FA`) | 52/57 bytes, 5 slack |
| `NEW_COS_CORE` | main block | `C-ENT`'s padding (`$37BB`) | 34/35 bytes, 1 slack |
| `NEW_LN_TAIL` | main block | own trampoline (`$374A`) | 37/57 bytes, 20 slack |
| `NEW_S_RND` | main block | own trampoline (`$25F8`) | 29/47 bytes, 18 slack |
| `NEW_TAN_CORE` | main block | `CIRCLE`'s padding (`$2331`) | 70/81 bytes, 11 slack |
| `RND_DEFAULT_TABLE` | main block | `COS`'s padding (`$37AA`) | 11/11 bytes, **0 slack** |

`NEW_SQR`, `NEW_SIN_CORE`, `ADDR_DISPATCH`, `SAFE_ZERO_BYTE`,
`CL_PRINTER_FIX`, `NEW_RND_BYTE`, `RND_BOOT_SEED`, `RND_RESEED`,
`RND_BOOT_STUB`, and the `CIRCLE` core (`CHECK_AND_PLOT`/`PLOT4`/
`NEW_CIRCLE_CORE`) remain in the main free block, in that order,
immediately following each other with no further gaps between them.

## Archive contents

- `checkpoint_gamma_Spectrum48_patched.asm` — the complete patched ROM
  source, current deliverable. TASM-native (directives active); comment
  out the six `#define` lines and the trailing `#end` to build with
  pasmo or another cross-assembler, per the file's own header note.
- `checkpoint_BETA_Decisions.md` — full chronological decision log for
  every relocation patch: the padding audit, the best-fit plan, all
  seven implemented patches, and both bugs found along the way, in
  narrative detail.
- `checkpoint_gamma_FUNCTION_SPECS.md` — structural reference: exact
  current addresses, sizes, and layout for every routine in the ROM,
  reflecting the final post-relocation state.
- `checkpoint_gamma_rom_harness.py` — the Python/`kosarev-z80` test
  harness (unchanged from Alpha/Beta).
- Alpha's own archive (`checkpoint_alpha_Spectrum48_patched.asm`,
  `checkpoint_alpha_Decisions.md`, `checkpoint_alpha_FUNCTION_SPECS.md`,
  `CHECKPOINT_ALPHA_SUMMARY.md`, test `.bas`/`.tap` files) remains the
  ground truth for every algorithm and every original patch's own
  verification history — nothing in it is superseded, only relocated.
