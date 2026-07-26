# checkpoint_BETA_Decisions.md

Full chronological decision log for Checkpoint Beta, in the same style
as `checkpoint_alpha_Decisions.md`. This file picks up immediately
where Checkpoint Alpha left off — see `CHECKPOINT_ALPHA_SUMMARY.md`,
`checkpoint_alpha_FUNCTION_SPECS.md`, and `checkpoint_alpha_Decisions.md`
for everything prior to this point.

---

## Entry 1 — Audit: stranded trampoline padding, not counted in the "290 bytes free" figure

**Date:** 2026-07-20
**Status:** Investigation only. No code changed, no bytes reclaimed yet.
This entry exists to record the findings and hand off a concrete,
already-verified worklist for the actual reclamation patches that
follow it in this file.

### Why this was worth checking

Checkpoint Alpha's headline free-space number — **290 bytes**, at
`$3BDE`-`$3CFF` — is the one *contiguous* unused block left in the ROM.
But twelve patches' worth of trampolines were each spliced into the
*original* routine's slot, and in every case that original slot was
wider than the `JP` (or `end-calc`+`JP`) needed to reach the relocated
code. The leftover bytes were zero-padded to preserve the original slot
width (so the *next* routine's `ORG` stays put) and then never revisited.
That padding is real, reclaimable ROM space that the "290" figure
doesn't include, because it isn't contiguous with the main block —
it's scattered across ten separate slots.

### Method

Not taken on trust from `FUNCTION_SPECS.md`'s prose descriptions.
Every trampoline was located in `checkpoint_alpha_Spectrum48_patched.asm`
directly (`grep` for each `ORG` address, then `view` of the surrounding
lines) and its actual `DEFB $00,...` padding counted byte-by-byte
against the following `ORG` (the next routine's safeguarded start),
the same "trust but verify against the real file" discipline used
throughout Checkpoint Alpha.

Also checked, to make sure nothing was missed: every *other* address-map
entry from Checkpoint Alpha's consolidated memory map that wasn't a
full trampoline — the small 1-2 byte "operand retargeted" entries
(`$0DDA`, `$0DFC`, `$0ECF`, `$0EE0`, `$0F15`, `$128C`, `$2508`). These
turned out to be genuinely zero-waste: each is a 2-3 byte operand field
inside an *existing, unchanged-length* `CALL`/`JP`/`JR` instruction that
simply got retargeted to a new address (e.g. `S-LOOP-1`'s `JP
NC,ADDR_DISPATCH` is the exact same 3-byte `JP cc,nn` shape as the
original `JP NC,S-ALPHNUM`). Nothing shrank there, so there's nothing
stranded. Confirmed directly rather than assumed, since this project's
own standing rule is "unexplained differences are never accepted as
probably fine" — the same rule applies to *assumed-fine* differences.

### Findings — ten stranded-padding trampolines, verified byte-by-byte

| Trampoline | Slot address | Slot size | Instruction used | **Stranded padding** |
|---|---|---:|---:|---:|
| `SQR` (`L384A`) | `$384A`-`$3850` | 7 | `JP` (3) | **4** |
| `SIN`/`COS` `C-ENT` (`L37B7`) | `$37B7`-`$37D9` | 35 | `end-calc`+`JP` (4) | **31** |
| `COS` (`L37AA`) | `$37AA`-`$37B4` | 11 | `JP` (3) | **8** |
| `TAN` (`L37DA`) | `$37DA`-`$37E1` | 8 | `JP` (3) | **5** |
| `ATN` `CASES` (`L37FA`) | `$37FA`-`$3832` | 57 | `end-calc`+`JP` (4) | **53** |
| `LN` tail (`L374A`) | `$374A`-`$3782` | 57 | `end-calc`+`JP` (4) | **53** |
| `EXP` (`L36C4`) | `$36C4`-`$36F8` | 53 | `JP` (3) | **50** |
| `S-RND` (`L25F8`) | `$25F8`-`$2626` | 47 | `JP` (3) | **44** |
| `CIRCLE` (`$2331`) | `$2331`-`$2381` | 81 | `JP` (3) | **78** |
| `RANDOMIZE` tail (`L1E5A`) | `$1E5A`-`$1E5E` | 5 | `JP` (3) | **2** |
| **Total** | | | | **328 bytes** |

Each row was confirmed by direct inspection of the padding `DEFB`s and
the following `ORG` line, not inferred from slot-size arithmetic alone —
e.g. `L37FA` reads `end-calc` / `JP NEW_ATN_CORE` / 53 bytes of `DEFB
$00` / `ORG $3833` (safeguarding `asn`), matching the table exactly.

### Combined with the already-logged dead code

`checkpoint_alpha_Decisions.md` separately logged `n-mod-m`
(`$36A0`-`$36AE`, 15 bytes) as confirmed dead — no ROM-internal caller
remains after Patch 8 replaced `RND` — but explicitly **not reclaimed**,
on the grounds that it's still a publicly dispatchable calculator
literal (`RST 28H; DEFB $32`) for any user machine-code program, and
that tradeoff was deliberately deferred rather than decided.

### True reclaimable total

| | Bytes |
|---|---:|
| Contiguous free block (`$3BDE`-`$3CFF`) — Alpha's headline "290" | 290 |
| Stranded trampoline padding (10 locations, table above) | 328 |
| `n-mod-m` — confirmed dead, still occupying its slot, not yet reclaimed | 15 |
| **True total reclaimable ROM space** | **633** |

More than double the 290-byte figure Alpha's summary leads with — it's
just scattered across 12 locations instead of sitting in one block.

### Notes for the reclamation work this file will track

- **Not contiguous, so no single `ORG` bump reclaims it.** Each slot has
  to be dealt with individually: either shrink the trampoline further
  (little room left in most — many are already the minimal `JP`/
  `end-calc`+`JP` shape) or repurpose the slot's own padding in place as
  small satellite scratch space for a future patch, the same way
  `NEW_SQR` etc. use the main free block today.
- **Priority by size, if reclaiming for a future patch's use:**
  `CIRCLE` (78), `ATN` (53) and `LN` (53) together account for 184 of
  the 328 stranded bytes — over half — from just three slots. Any patch
  needing a modest amount of extra room should look at these three
  first before assuming the main block is the only option.
- **`n-mod-m`'s 15 bytes are a different kind of decision**, not a
  mechanical reclaim — taking them back removes a still-technically-
  available (if obscure) user-facing capability. Revisit the tradeoff
  explicitly if/when a patch actually needs those 15 bytes; don't
  reclaim it silently as a side effect of something else.
- **Every number in this entry needs to survive Checkpoint Alpha's own
  standing discipline once any of it is acted on**: baseline-diffed,
  standalone-assembled, dynamically run — not just diffed — before any
  patch here is considered done. This entry is an audit, not a patch;
  none of Alpha's verification steps have been applied to *changing*
  any of these bytes yet.

### Files

- `checkpoint_BETA_Decisions.md` — this file.

---

## Entry 2 — Best-fit plan: relocating existing routines into stranded trampoline padding

**Date:** 2026-07-20
**Status:** Plan only, fully specified and cross-checked against the
actual patched source, but **not yet applied**. Every move below still
needs the full `Safety.md` treatment (standalone reassembly, full byte
diff against baseline, dynamic emulator run) before it counts as done,
exactly like every patch in Checkpoint Alpha.

### Goal

Entry 1 identified 328 bytes of zero-padding stranded across the ten
trampoline slots left behind by Checkpoint Alpha's twelve patches — real
ROM space, just not contiguous with the main 290-byte free block. This
entry works out which of the *already-written* new routines can be
moved into that stranded space, and how much of it actually converts
into genuinely reclaimed, contiguous free space at the routines' old
locations in the main block.

### Method

For each of the ten stranded slots, checked two things in order:

1. **Self-fit first** — does the slot's *own* destination routine (the
   one its trampoline currently `JP`s to) fit back inside the slot,
   directly following whatever prefix is structurally required
   (`end-calc` if the slot is reached via a calculator-level
   `jump`/`jump-true`, otherwise nothing)? If so, the `JP` itself is no
   longer needed at all — the routine just falls straight into place —
   which is strictly better than any cross-routine match, since it also
   removes 3 bytes of jump overhead.
2. **Best remaining fit otherwise** — for slots too small for their own
   routine, checked every other new routine for the tightest fit,
   confirming each candidate is reached only via an absolute `JP`/`CALL`
   (or, for calculator-literal routines, self-contained internal
   `jump`/`jump-true` offsets computed relative to their own labels —
   confirmed by inspecting the actual `DEFB label-$` bytes, e.g.
   `NEW_TAN_CORE`'s `NT_SKIP_NEG-$`/`NT_RECIP_PREP-$`/`NT_PADE-$`), so
   relocating the whole routine as a block doesn't disturb anything
   inside it.

Every candidate was read directly from
`checkpoint_alpha_Spectrum48_patched.asm` — entry instruction, exit
instruction, and internal jump style — not inferred from `FUNCTION_SPECS.md`'s
prose alone.

### Results

| Padding slot | Capacity for inlined code | Best-fit routine | Size | Local waste | Bytes reclaimed |
|---|---:|---|---:|---:|---:|
| `EXP` trampoline (`$36C4`) | 53 | `NEW_EXP_CORE` (self-fit) | 52 | 1 | 52 |
| `C-ENT`/`SIN` trampoline (`$37B7`) | 34 (after mandatory `end-calc`+`JP NEW_SIN_CORE`) | `NEW_COS_CORE` | 30 | 1 | 30 |
| `CIRCLE` trampoline (`$2331`) | 78 (after mandatory `JP` to the real core) | `NEW_TAN_CORE` | 67 | 11 | 67 |
| `ATN CASES` trampoline (`$37FA`) | 56 (after mandatory `end-calc`) | `NEW_ATN_CORE` (self-fit) | 51 | 5 | 51 |
| `LN` tail trampoline (`$374A`) | 56 (after mandatory `end-calc`) | `NEW_LN_TAIL` (self-fit) | 36 | 20 | 36 |
| `S-RND` trampoline (`$25F8`) | 47 | `NEW_S_RND` (self-fit) | 29 | 18 | 29 |
| `COS` trampoline (`$37AA`) | 8 | `RND_DEFAULT_TABLE` | 8 | 0 | 8 |
| `SQR` trampoline (`$384A`) | 4 | none — too small to be worth it | — | — | 0 |
| `TAN` trampoline (`$37DA`) | 5 | none — too small to be worth it | — | — | 0 |
| `RANDOMIZE` tail (`$1E5A`) | 2 | none — too small to be worth it | — | — | 0 |
| **Total reclaimed into the contiguous pool** | | | | | **273** |

### Notes on the non-obvious matches

- **`EXP` is the tightest and safest fit of the set.** `NEW_EXP_CORE`
  is exactly 52 bytes and already ends with an absolute `JP $36F9`
  (the reused, untouched overflow-handling tail) — it doesn't care
  where it physically sits, so inlining it costs nothing beyond 1
  wasted byte.
- **`C-ENT` and `CIRCLE` end up hosting a *different* routine than the
  one their own trampoline points to.** Their own destination core
  (`NEW_SIN_CORE`, `NEW_CIRCLE_CORE`) is too large to fit locally, but
  each slot's own required header — `end-calc`+`JP` (4 bytes) or just
  `JP` (3 bytes) — leaves real spare room *after* it that is never
  locally fallen-into, so it's safe to use as plain addressable storage
  for an unrelated routine reached via its own separate `JP` elsewhere
  (`COS`'s trampoline, `TAN`'s trampoline respectively). This is the
  same principle every "SPARE" main-block routine already relies on —
  just applied to a smaller, non-contiguous pocket instead of the big
  block.
- **`RND_DEFAULT_TABLE`→`COS`'s 8-byte pad is a perfect, zero-waste
  fit.** It's pure data (`DEFB 82,97,120,111,102,116,20,12`), referenced
  only by `LD HL,RND_DEFAULT_TABLE` from `RND_BOOT_SEED` and
  `RND_RESEED` — no proximity dependency at all.
- **`SQR` (4B), `TAN` (5B), and `RANDOMIZE`'s tail (2B) are left
  empty.** Only the 1-byte `SAFE_ZERO_BYTE` would technically fit
  anywhere in them, and reclaiming a single byte isn't worth a full
  baseline-diff-and-dynamic-test cycle under this project's own
  standing discipline. Noted, not recommended.

### Net effect, if applied

| | Bytes |
|---|---:|
| Contiguous free block today (`$3BDE`-`$3CFF`) | 290 |
| Reclaimed via this repacking (7 routines relocated) | 273 |
| **Contiguous free block after repacking** | **563** |

No algorithm, accuracy, or behavior changes anywhere in this plan — it
is pure relocation of already-verified code into already-identified
dead space. `n-mod-m`'s 15 bytes remain a separate, deliberately
deferred decision (Alpha `Decisions.md`), untouched by this entry.

### Verification still required before this counts as done

Per this project's own standing rule ("dynamic verification, not just
static" — the `ATN` incident is the standing example of why): each of
the seven relocations above must still go through baseline diff,
standalone reassembly, full 16KB byte diff against the pre-repack
build, and a real emulator run of the regression suite, one at a time,
before being considered complete. This entry is the worklist, not the
patch.

### Safety check: no labels inside any padding region

Before treating any of the ten slots as free to write into, checked
that nothing in the ROM still targets an address inside the padding
itself — a leftover label pointing into what's now a run of `DEFB $00`
would mean something still expects real code there, and overwriting it
would silently break whatever refers to it.

Two independent checks, both clean:

1. **Direct visual inspection.** Printed every padding line (the
   `DEFB $00,...` lines only, excluding each slot's own entry
   instruction and the following `ORG`) for all ten slots straight from
   `checkpoint_alpha_Spectrum48_patched.asm`. Every line is a bare
   `DEFB $00,...` — no label, no comment implying a jump target, in any
   of the ten regions.
2. **Programmatic cross-check.** Parsed every `Lxxxx:`-style address
   label in the entire file (1,121 total) and checked each one's
   numeric address against all ten padding ranges. Exactly one label
   falls inside each range in every case — and in every case it's the
   slot's own entry point (`L384A`, `L37AA`, `L37B7`, `L37DA`, `L37FA`,
   `L374A`, `L36C4`, `L25F8`, `L1E5A`), i.e. the very first byte of the
   slot, not a stray label inside the zero-padding that follows it. The
   `CIRCLE` trampoline at `$2331` has no label at all inside its range
   (it's reached only via the original ROM's own fixed dispatch address,
   never referenced by name elsewhere in the source). Zero labels found
   strictly inside any padding region.

**Conclusion: safe.** None of the 328 stranded bytes identified in
Entry 1 have anything still pointing into them. Nothing found here
changes the repacking plan above.

### Files

- `checkpoint_BETA_Decisions.md` — this file (Entry 1: padding audit;
  Entry 2: this repacking plan, plus the label-safety check).

---

## Entry 3 — `EXP` relocated into its own trampoline slot (first repacking patch, done)

**Date:** 2026-07-20
**File:** `checkpoint_beta_EXP_relocated_Spectrum48_patched.asm`, built on
top of the Checkpoint Alpha deliverable.
**Status:** Implemented and verified to this project's full `Safety.md`
standard — baseline diff, standalone reassembly, full byte diff, and
dynamic emulator testing, all done, not just planned.

### What changed

`NEW_EXP_CORE` (52 bytes) moved from its old spot in the main free
block (`$3A62`-`$3A95`) to sit *inline*, directly inside its own
original trampoline slot at `$36C4`-`$36F8` (the first candidate
identified in Entry 2 — the tightest fit of the set, 1 byte of local
slack). The `JP NEW_EXP_CORE` instruction is gone entirely; execution
now falls straight from the `exp` dispatch table entry into the Pade
approximant code, exactly as the *original* ROM's `exp` did before
Checkpoint Alpha touched it. The routine's own logic is byte-for-byte
unchanged — this is a pure relocation, copied verbatim from the
Alpha source.

Two follow-on structural changes were required to keep the ROM
contiguous and correct:

1. **`CHECK_AND_PLOT`/`PLOT4`/`NEW_CIRCLE_CORE`'s `ORG` moved from
   `$3A96` to `$3A62`** — it now begins immediately after `NEW_LN_TAIL`
   instead of after the (now-vacated) `NEW_EXP_CORE`. This shifts the
   entire 328-byte `CIRCLE` block 52 bytes earlier in address space.
2. **The main free-space block's `ORG` moved from `$3BDE` to `$3BAA`**,
   and 52 more bytes of explicit `$FF` fill were added, growing it from
   290 to 342 bytes.

### A mistake caught mid-patch, worth logging

The first assembly attempt (before fixing #2 above) **assembled clean
and produced a 16384-byte ROM with no errors** — but the free-space
block's `ORG` was left at Checkpoint Alpha's old, now-stale `$3BDE`.
Since the actual code above it now ended 52 bytes earlier, this created
an *implicit* forward-jump gap that pasmo silently zero-filled, and the
existing 290 bytes of explicit `$FF` were then tacked on after that —
netting the same total free byte count, but 52 bytes of it were an
un-flagged `$00` gap rather than genuine, explicitly-accounted free
space. Caught not by assembly failing (it didn't) but by directly
checking where the trailing `$FF` run actually started and finding it
unchanged from Alpha's `$3BDE`, despite 52 bytes having supposedly been
reclaimed. This is exactly the kind of thing this project's own
discipline (full diff, every byte explained; verify structural claims
directly rather than trusting a clean assembly) exists to catch — logged
here rather than quietly fixed and forgotten, per the project's own
precedent for bugs found during a patch (see the `ATN` incident in
Alpha's `Decisions.md`).

### Verification

**Static — full byte diff against the Alpha baseline**, 418 differing
bytes total, all explained:
- `$2332`-`$2333` (2 bytes): the `CIRCLE` trampoline's `JP` operand,
  automatically updated by reassembly to point at `NEW_CIRCLE_CORE`'s
  new address — decodes to exactly a 52-byte shift (`$3B22`→`$3AEE`),
  matching the block move precisely.
- `$36C4`-`$36F7` (52 bytes): the inlined `NEW_EXP_CORE` code, replacing
  the old `JP`+padding.
- `$3A62`-`$3BDD` (the remaining ranges): the `CIRCLE` block's content,
  shifted 52 bytes earlier, plus the free-block boundary moving 52
  bytes earlier with it. Confirmed no unexplained zero-runs anywhere in
  the touched region — every contiguous zero-run of 4+ bytes in
  `$36F9`-`$3BAA` was cross-checked against the six known, pre-existing
  Alpha trampoline paddings (`LN`, `COS`, `C-ENT`, `TAN`, `ATN`, `SQR`)
  and matches each exactly; nothing new or stray.
- Character bitmap table (`$3D00`-`$3FFF`): confirmed byte-identical
  between the two builds.

**Dynamic — three separate test passes, all against the real, fully
assembled ROM through its actual entry points, not a standalone copy:**

1. **`EXP` accuracy sweep**, 408 test points (`-20` to `20` in `0.1`
   steps plus extremes): zero unexpected errors, max relative error
   within the same `~7e-6` band Alpha itself documented, `exp(0)=1`
   exactly, overflow/underflow error paths unchanged.
2. **Direct A/B equivalence test**, 4,215 calculator-literal
   comparisons across `sin`/`cos`/`tan`/`atn`/`asn`/`acs`/`ln`/`sqr`/`exp`,
   same inputs run against the Alpha baseline and this Beta patch side
   by side: **zero mismatches** — every single result bit-for-bit
   identical between the two builds. (An initial pass showed apparent
   "failures" against fixed accuracy thresholds for `sin`/`cos`/`ln`/`exp`'s
   error-report code; re-running the identical test against the
   *unpatched* Alpha baseline reproduced the exact same numbers, proving
   these were pre-existing characteristics of the test's own assumptions
   — report-code numbering, near-zero relative-error blowup — not
   regressions. Logged so the same false trail isn't re-walked next
   time.)
3. **`CIRCLE` dynamic A/B test**, run through its real trampoline entry
   point at `$2331` (matching Alpha's own documented entry contract:
   `x,y,r` pre-stacked on the FP calculator stack, `IY=$5C3A`, fresh Z80
   entry) for 5 circles spanning small radii, off-center positions, and
   the original's own documented near-maximum radius (`87`): **screen
   memory bit-for-bit identical** between Alpha and Beta in every case,
   with identical tick counts (same execution path, same timing).

### Net result

| | Before (Alpha) | After (this patch) |
|---|---:|---:|
| Contiguous free ROM block | 290 bytes | **342 bytes** |
| `EXP` behavior | baseline | **provably identical** (0/4215 mismatches) |
| `CIRCLE` behavior | baseline | **provably identical** (5/5 screens match) |

Zero algorithm or accuracy changes. Six repacking candidates from
Entry 2 remain: `NEW_ATN_CORE`→its own slot (+51), `NEW_LN_TAIL`→its
own slot (+36), `NEW_S_RND`→its own slot (+29), `NEW_COS_CORE`→`C-ENT`
padding (+30), `NEW_TAN_CORE`→`CIRCLE` padding (+67),
`RND_DEFAULT_TABLE`→`COS` padding (+8) — 221 bytes still on the table,
which would bring the contiguous free block to 563 total once done.

### Files

- `checkpoint_beta_EXP_relocated_Spectrum48_patched.asm` — the patched
  ROM source, current Beta deliverable.
- `beta_patch1_exp_accuracy_test.py` — the `EXP` accuracy sweep.
- `beta_patch1_exp_ab_equivalence_test.py` — the 4,215-point A/B
  equivalence test across every calculator-literal function.
- `beta_patch1_circle_ab_test.py` — the `CIRCLE` dynamic A/B screen-diff
  test.
- `checkpoint_BETA_Decisions.md` — this file.

### Post-delivery correction: TASM directives were stripped from the deliverable

**A real mistake in the delivered file, caught by the user, not by this
project's own process.** Verification was done with `pasmo` (per this
project's standard toolchain, since the ROM's own header requires the
TASM `#define DEFB .BYTE` / etc. lines to be commented out for any
other assembler — see the header's own note). The working copy used for
that verification had those 6 `#define` lines and the trailing `#end`
commented out, exactly as every prior patch in this project has always
done for its own pasmo builds.

The mistake: the delivered `.asm` file was built starting from that
*commented-out* working copy, instead of from a TASM-native copy with
the directives left active. The result assembled and verified correctly
under pasmo, but was silently broken for the file's actual intended
assembler, TASM — which every prior Alpha/Beta deliverable up to this
point had correctly preserved.

**Fixed by restoring the 7 directive lines to their active, uncommented
form in the actual delivered file**, then re-verifying this didn't
silently change anything else: recommented those same 7 lines in a
throwaway copy (the same transformation this project always applies
before a pasmo build), reassembled, and confirmed the resulting binary
is **byte-for-byte identical** to the already-verified
`beta_patch1_exp.bin` (the one behind all of this entry's 4,623 dynamic
test comparisons above). So the fix only restores the directives —
none of the actual patch content changed, and nothing needs to be
re-verified beyond this file-identity check.

**Lesson for future patches in this project:** always build the
pasmo-testable copy *from* the TASM-native deliverable (comment,
build, discard the copy), never edit the commented copy directly and
promote it to deliverable status. Worth a standing checklist item
before any future patch is called done.

---

## Entry 4 — `ATN` relocated into its own trampoline slot (second repacking patch, done)

**Date:** 2026-07-20
**File:** `checkpoint_beta_EXP_ATN_relocated_Spectrum48_patched.asm`,
built on top of Entry 3's deliverable. Supersedes it.
**Status:** Implemented and verified to this project's full `Safety.md`
standard.

### What changed

`NEW_ATN_CORE` (51 bytes) moved from its old spot in the main free
block (`$3969`-`$399B`) to sit inline, directly inside its own original
`CASES` trampoline slot at `$37FA`-`$3832` — the same move as Entry 3's
`EXP` relocation, and the second candidate identified in Entry 2. The
required `end-calc` prefix stays (this slot is reached via a
calculator-level `jump`/`jump-true`, same as `SIN`/`COS`'s `C-ENT`), but
the `JP NEW_ATN_CORE` that used to follow it is gone — execution now
falls straight from `end-calc` into the routine's own fresh `RST 28H`,
the exact same pattern `NEW_SQR` already used. 5 bytes of local slack
remain in the 57-byte slot. The routine's own logic is byte-for-byte
unchanged, copied verbatim from the prior source.

### The ripple effect this one has, that EXP's relocation didn't

`NEW_ATN_CORE` sat *earlier* in the main block's sequential chain than
`NEW_EXP_CORE` did — specifically, immediately after `ADDR_DISPATCH`
and before `SAFE_ZERO_BYTE`, `CL_PRINTER_FIX`, `NEW_RND_BYTE`,
`RND_BOOT_SEED`, `RND_DEFAULT_TABLE`, `RND_RESEED`, `NEW_S_RND`,
`RND_BOOT_STUB`, `NEW_LN_TAIL`, and the `CIRCLE` core. Removing it
means **all nine of those** shift 51 bytes earlier too, not just the
one thing immediately downstream. Six hardcoded `ORG` statements needed
updating to match (`SAFE_ZERO_BYTE`, `NEW_RND_BYTE`, `RND_BOOT_STUB`,
`NEW_LN_TAIL`, `CHECK_AND_PLOT`, and the free-space block), each
shifted down by exactly 51 from Entry 3's addresses.

This also means every trampoline anywhere in the ROM that points at any
of those nine relocated things needs its jump operand updated — which
happens automatically on reassembly since they're all symbolic label
references, not hardcoded addresses, but it does mean a noticeably
wider full-byte-diff footprint than Entry 3's (607 bytes touched here
vs. 418 for `EXP`, even though the actual code change is smaller).

### Verification

**Static — full byte diff against the Entry 3 baseline**, 607 differing
bytes, all explained:
- Nine single- or double-byte diffs at fixed operand sites
  (`$0DDA`/`$0DFC`/`$0ECF`/`$0EE0` — `SAFE_ZERO_BYTE` references from
  `CL-SET`/`CL-SET-2`/`COPY-BUFF`/`CLEAR-PRB`; `$128C` — boot-time
  `RND_BOOT_STUB` call; `$1E5B` — `RANDOMIZE`'s `RND_RESEED` tail call;
  `$2332` — `CIRCLE` trampoline's `JP` target; `$25F9` — `S-RND`
  trampoline's `JP` target; `$374C` — `LN` trampoline's `JP` target).
  **Every one decoded as a 16-bit little-endian address operand and
  confirmed to shift by exactly `51`**, not just eyeballed — checked
  programmatically against both builds.
- The `$37FB`-`$382D` region: the inlined `NEW_ATN_CORE` code.
- Everything from `$3969` onward: the nine downstream routines' content
  shifted 51 bytes earlier, plus the free-block boundary moving with
  them.
- Character bitmap table (`$3D00`-`$3FFF`): confirmed byte-identical.
- Free-block boundary directly checked (not inferred from the diff):
  grew from 342 to **393 bytes**, exactly 51 more. Only two contiguous
  zero-runs remain in the `$37FA`-`$3B77` window — `ATN`'s own new
  5-byte slack and the pre-existing, untouched `SQR` 4-byte slack —
  cross-checked against expectation, nothing stray.

**Dynamic — four test passes, against the real ROM through its actual
entry points:**

1. **5,216-point direct A/B equivalence** across
   `sin`/`cos`/`tan`/`atn`/`asn`/`acs`/`ln`/`sqr`/`exp`: zero mismatches.
2. **`ATN` accuracy sweep**, ~440 points including `±1e10` and `±1e-6`
   extremes: max error within the documented `1.87e-4` band.
3. **`RND` A/B sequence test** — direct entry through `RND_RESEED` (now
   at `$39BE`, was `$39F1`) to seed, then 20 calls through the `S-RND`
   trampoline (unmoved at `$25F8`, but its own `JP` target moved):
   identical 20-value sequence between the pre- and post-relocation
   ROMs. This specifically exercises the three routines that moved the
   furthest as a side effect of this patch (`RND_RESEED`, `NEW_S_RND`,
   `RND_BOOT_STUB`) and confirms none of their internal logic or cross-
   references broke.
4. **`CIRCLE` A/B screen-memory test**, same 5 circles as Entry 3's:
   screen memory bit-for-bit identical in every case (confirms
   `CHECK_AND_PLOT`/`PLOT4`/`NEW_CIRCLE_CORE`, now shifted a cumulative
   103 bytes from their original Alpha position, still work correctly).

### Net result

| | Before (Entry 3) | After (this patch) |
|---|---:|---:|
| Contiguous free ROM block | 342 bytes | **393 bytes** |
| `ATN` behavior | baseline | **provably identical** (0/5216 mismatches) |
| `RND` behavior | baseline | **provably identical** (20/20 sequence match) |
| `CIRCLE` behavior | baseline | **provably identical** (5/5 screens match) |

Five repacking candidates from Entry 2 remain: `NEW_LN_TAIL`→its own
slot (+36), `NEW_S_RND`→its own slot (+29), `NEW_COS_CORE`→`C-ENT`
padding (+30), `NEW_TAN_CORE`→`CIRCLE` padding (+67),
`RND_DEFAULT_TABLE`→`COS` padding (+8) — 170 bytes still on the table,
which would bring the contiguous free block to 563 total once done
(unchanged from the original Entry 2 estimate, since all of Entry 2's
byte-accounting was already done relative to Alpha, not order-dependent).

### Files

- `checkpoint_beta_EXP_ATN_relocated_Spectrum48_patched.asm` — the
  patched ROM source, current Beta deliverable (supersedes Entry 3's).
- `beta_patch2_atn_regression_test.py` — the four-part dynamic
  regression suite (A/B equivalence, ATN accuracy, RND sequence,
  CIRCLE screen-diff).
- `checkpoint_BETA_Decisions.md` — this file.

---

## Entry 5 — `COS` relocated into `C-ENT`'s spare padding (third repacking patch, done)

**Date:** 2026-07-20
**File:** `checkpoint_beta_EXP_ATN_COS_relocated_Spectrum48_patched.asm`,
built on top of Entry 4's deliverable. Supersedes it.
**Status:** Implemented and verified to this project's full `Safety.md`
standard.

### What changed, and why this one's different from Entries 3 and 4

`NEW_COS_CORE` (30 bytes) moved from its old spot in the main free
block (`$38CB`-`$38E8`) to `$37BB` — **not** its own trampoline slot
(`COS`'s own trampoline at `$37AA` is only 11 bytes total, nowhere near
enough), but the *spare padding inside a different function's
trampoline*: `SIN`/`COS`'s shared `C-ENT` slot at `$37B7`. That slot's
own required content — `end-calc` + `JP NEW_SIN_CORE` (4 bytes) — is
completely unrelated to `NEW_COS_CORE` and stays exactly as it was; the
30 bytes of `NEW_COS_CORE`'s code simply occupy the padding that used
to be zero-filled *after* that `JP`, reachable only via a separate `JP`
from `COS`'s own trampoline (never fallen into locally, so safe to use
as pure storage — the same "different routine hosted in a slot's own
slack" pattern flagged as available in Entry 2, first one actually
used). `COS`'s trampoline text is unchanged (`JP NEW_COS_CORE`); only
what that symbol resolves to moved. 1 byte of slack remains in the
35-byte `C-ENT` slot.

### A second stale-`ORG` bug, same class as Entry 3's, caught the same way

Removing `NEW_COS_CORE` from the main block meant `NEW_TAN_CORE` (which
immediately follows it there) should close the 30-byte gap automatically
— but a hardcoded `ORG $38E9` sat directly in front of `NEW_TAN_CORE`'s
own label, left over from Alpha, that I missed on the first editing
pass (my downstream-`ORG` sweep searched for `$39xx`/`$3Axx` patterns
and skipped over this `$38xx` one). Assembly succeeded cleanly at
16384 bytes regardless. Caught the same way as Entry 3's version of
this mistake: checking the free-block boundary directly rather than
trusting a clean build, which showed growth of only 30 bytes *and* an
unexplained 30-byte zero-run sitting exactly at the old `NEW_COS_CORE`
address (`$38CB`-`$38E8`) — a dead giveaway. Fixed by retargeting that
`ORG` to `$38CB` (immediately after `NEW_SIN_CORE`).

**Standing lesson reinforced**: this project now has two independent
instances of the same failure mode (Entry 3, Entry 5) — a downstream
hardcoded `ORG` left stale after removing something upstream, assembling
clean either way, only caught by directly checking the free-block
boundary. Worth treating "grep for every `ORG $3xxx` in the file, not
just the ones near where I'm editing" as a mandatory step before calling
any future relocation patch done, not an optional sanity check.

### Verification

**Static — full byte diff against the Entry 4 baseline** (post-fix),
698 differing bytes, all explained:
- Eleven small operand sites (`SAFE_ZERO_BYTE`'s four references,
  boot-time init, `RANDOMIZE` tail, `CIRCLE`/`S-RND`/`LN` trampoline
  targets, and — new this patch — the `S-LOOP-1`→`ADDR_DISPATCH` hook
  at `$2508`, since `ADDR_DISPATCH` is downstream of the removed
  `NEW_COS_CORE` too): **ten of eleven decode to exactly a `-30` shift**,
  checked programmatically. The eleventh, `COS`'s own trampoline operand
  at `$37AB`, correctly decodes to a `-272`-byte jump (`$38CB`→`$37BB`)
  — not a ripple shift, but the actual relocation itself, confirmed to
  be the expected value rather than assumed.
- The `C-ENT` slot region (`$37BB` onward): the inlined `NEW_COS_CORE`.
- Everything from `$38CD` onward: the downstream routines' content
  shifted 30 bytes earlier, plus the free-block boundary moving with
  them.
- Free-block boundary directly checked: grew from 393 to **423 bytes**,
  exactly 30 more. Only three contiguous zero-runs remain in the
  `$37B7`-`$3B59` window (`TAN`'s pre-existing 5-byte slack, `ATN`'s
  5-byte slack from Entry 4, `SQR`'s 4-byte slack) — cross-checked,
  nothing stray, confirming the `ORG` fix above actually worked.
- Character bitmap table: confirmed byte-identical.

**Dynamic — same four-part suite as Entries 3/4, addresses updated for
this patch's shifts:**

1. **5,216-point direct A/B equivalence** across all nine calculator
   functions: zero mismatches.
2. **`COS` accuracy sweep** against `math.cos`: within the same
   Bhaskara-tier error band Alpha itself documented.
3. **`RND` A/B sequence test**, reseeded through `RND_RESEED` (now at
   `$39A0`, shifted again) and 20 calls through `S-RND`: identical
   sequence pre/post.
4. **`CIRCLE` A/B screen-memory test**, same 5 circles: bit-for-bit
   identical screens (the core has now shifted a cumulative 133 bytes
   from its original Alpha position and still checks out).

### Net result

| | Before (Entry 4) | After (this patch) |
|---|---:|---:|
| Contiguous free ROM block | 393 bytes | **423 bytes** |
| `COS` behavior | baseline | **provably identical** (0/5216 mismatches) |
| `RND` behavior | baseline | **provably identical** (20/20 sequence match) |
| `CIRCLE` behavior | baseline | **provably identical** (5/5 screens match) |

Four repacking candidates from Entry 2 remain: `NEW_LN_TAIL`→its own
slot (+36), `NEW_S_RND`→its own slot (+29), `NEW_TAN_CORE`→`CIRCLE`
padding (+67), `RND_DEFAULT_TABLE`→`COS` padding (+8) — 140 bytes still
on the table, which would bring the contiguous free block to 563 total
once done.

### Files

- `checkpoint_beta_EXP_ATN_COS_relocated_Spectrum48_patched.asm` — the
  patched ROM source, current Beta deliverable (supersedes Entry 4's).
- `beta_patch3_cos_regression_test.py` — the four-part dynamic
  regression suite, adapted for this patch.
- `checkpoint_BETA_Decisions.md` — this file.

---

## Entry 6 — `LN` relocated into its own trampoline slot (fourth repacking patch, done)

**Date:** 2026-07-20
**File:** `checkpoint_beta_EXP_ATN_COS_LN_relocated_Spectrum48_patched.asm`,
built on top of Entry 5's deliverable. Supersedes it.
**Status:** Implemented and verified to this project's full `Safety.md`
standard.

### What changed

`NEW_LN_TAIL` (36 bytes) moved from its old spot in the main free block
to sit inline, directly inside its own original `GRE.8` trampoline slot
at `$374A`-`$3782` — the same "self-fit" move as Entries 3/4's
`EXP`/`ATN` relocations. The `end-calc` prefix stays (this slot is
reached mid-literal-stream, same reasoning as `C-ENT`/`CASES`), but the
`JP NEW_LN_TAIL` that followed it is gone — execution falls straight
into the routine's own fresh `RST 28H`. 20 bytes of local slack remain
in the 57-byte slot, the widest margin of any self-fit relocation so
far (Entry 2 flagged this one as the least tight of the three
same-slot fits).

### This time, the downstream ripple was checked exhaustively up front

Learning directly from the two stale-`ORG` bugs in Entries 3 and 5
(both caused by an incomplete search for hardcoded `ORG` statements),
this patch started with `grep -n "^\s*ORG\s+\$3[89A-F]"` across the
**entire** file before editing anything, to get a complete list rather
than searching only near where I expected changes. That list showed
only two `ORG`s downstream of `NEW_LN_TAIL`'s old position needed
updating (`CHECK_AND_PLOT` and the free-space block) — everything
upstream (`SAFE_ZERO_BYTE`, `NEW_RND_BYTE`, `RND_BOOT_STUB`, `ADDR_DISPATCH`,
etc.) is unaffected, since `NEW_LN_TAIL` was the *last* thing before
the `CIRCLE` core in the chain, not somewhere in the middle like `ATN`
or `COS` were. This produced a much narrower diff footprint than
Entries 4/5's ripple-heavy patches: only one small operand site changed
(the `CIRCLE` trampoline's own `JP` target, confirmed to decode to
exactly `-36`), not nine or eleven. No new stale-`ORG` bugs found this
time — the free-block boundary check confirmed exactly 36 bytes of
growth with no unexplained gaps on the first assembly attempt.

### Verification

**Static — full byte diff against the Entry 5 baseline**, 377
differing bytes, all explained:
- `$2332` (1 byte): `CIRCLE` trampoline's `JP` operand, decoded
  programmatically to a shift of exactly `36` (`$3A9D`→`$3A79`).
- `$374B` onward: the inlined `NEW_LN_TAIL` code and its 20-byte local
  slack.
- `$39ED` onward: the `CIRCLE` core's content shifted 36 bytes earlier,
  plus the free-block boundary moving with it.
- Free-block boundary directly checked: grew from 423 to **459 bytes**,
  exactly 36 more. Five contiguous zero-runs remain in the touched
  window — `LN`'s own new 20-byte slack, plus the four already-known,
  untouched pads (`COS` 8B, `TAN` 5B, `ATN` 5B, `SQR` 4B) — all
  cross-checked, nothing stray.
- Character bitmap table: confirmed byte-identical.

**Dynamic — same four-part suite as Entries 3-5, addresses updated:**

1. **5,216-point direct A/B equivalence** across all nine calculator
   functions: zero mismatches.
2. **`LN` accuracy sweep** against `math.log`, including `1e-6` to
   `1e6` extremes: within the documented `1.04e-5` band.
3. **`RND` A/B sequence test**: identical 20-value sequence (addresses
   for `RND_RESEED`/`RND_BOOT_STUB` unchanged this patch, exactly as
   predicted by the diff showing no operand changes near them).
4. **`CIRCLE` A/B screen-memory test**, same 5 circles: bit-for-bit
   identical screens (the core has now shifted a cumulative 169 bytes
   from its original Alpha position).

### Net result

| | Before (Entry 5) | After (this patch) |
|---|---:|---:|
| Contiguous free ROM block | 423 bytes | **459 bytes** |
| `LN` behavior | baseline | **provably identical** (0/5216 mismatches) |
| `RND` behavior | baseline | **provably identical** (20/20 sequence match) |
| `CIRCLE` behavior | baseline | **provably identical** (5/5 screens match) |

Three repacking candidates from Entry 2 remain: `NEW_S_RND`→its own
slot (+29), `NEW_TAN_CORE`→`CIRCLE` padding (+67),
`RND_DEFAULT_TABLE`→`COS` padding (+8) — 104 bytes still on the table,
which would bring the contiguous free block to 563 total once done.

### Files

- `checkpoint_beta_EXP_ATN_COS_LN_relocated_Spectrum48_patched.asm` —
  the patched ROM source, current Beta deliverable (supersedes
  Entry 5's).
- `beta_patch4_ln_regression_test.py` — the four-part dynamic
  regression suite, adapted for this patch.
- `checkpoint_BETA_Decisions.md` — this file.

---

## Entry 7 — `S-RND` relocated into its own trampoline slot (fifth repacking patch, done)

**Date:** 2026-07-20
**File:** `checkpoint_beta_EXP_ATN_COS_LN_SRND_relocated_Spectrum48_patched.asm`,
built on top of Entry 6's deliverable. Supersedes it.
**Status:** Implemented and verified to this project's full `Safety.md`
standard.

### What changed

`NEW_S_RND` (29 bytes) moved from its old spot in the main free block
to sit inline, directly inside its own original `S-RND` trampoline slot
at `$25F8`-`$2626` — the same self-fit move as Entries 3/4/6. `S-RND`
is entered as fresh Z80 code (its own first instruction is a plain
`CALL`, not a calculator literal), so — like `EXP` — no `end-calc`
prefix was ever needed; the `JP NEW_S_RND` is simply gone, and
execution falls straight into the routine's own first `CALL`. 18 bytes
of local slack remain in the 47-byte slot. The routine's own internal
relative jump (`JR Z,RND_SYNTAX_SKIP`) is self-contained within the
29 bytes and needed no adjustment — confirmed safe the same way
`NEW_TAN_CORE`'s internal offsets were in Entry 2's planning.

### Complete downstream `ORG` sweep, same discipline as Entry 6

Following Entry 6's successful approach (a full `grep` across the whole
file for every `ORG $2xxx`/`$3xxx` before editing, not just near the
expected site), this patch found exactly two downstream `ORG`s needing
a `-29` shift: `RND_BOOT_STUB` and `CHECK_AND_PLOT`, plus the free-space
block. No new stale-`ORG` bugs — the free-block boundary check
confirmed exactly 29 bytes of growth with no unexplained gaps on the
first assembly attempt, same clean result as Entry 6.

### Verification

**Static — full byte diff against the Entry 6 baseline**, 353
differing bytes, all explained:
- Two small operand sites, both decoded programmatically to a shift of
  exactly `29`: `$128C` (the boot-time `CALL RND_BOOT_STUB`) and
  `$2332` (`CIRCLE` trampoline's `JP` target).
- `$25F8` onward: the inlined `NEW_S_RND` code and its 18-byte local
  slack.
- `$39CA` onward: `RND_BOOT_STUB` and the `CIRCLE` core's content
  shifted 29 bytes earlier, plus the free-block boundary moving with
  them.
- Free-block boundary directly checked: grew from 459 to **488 bytes**,
  exactly 29 more. Six contiguous zero-runs remain in the touched
  window — `S-RND`'s own new 18-byte slack, plus the five already-known,
  untouched pads (`LN` 20B, `COS` 8B, `TAN` 5B, `ATN` 5B, `SQR` 4B) —
  all cross-checked, nothing stray.
- Character bitmap table: confirmed byte-identical.

**Dynamic — same suite as Entries 3-6** (no separate accuracy sweep
this time, since `S-RND` isn't a calculator literal — its correctness
is entirely covered by the sequence test):

1. **5,216-point direct A/B equivalence** across all nine calculator
   functions: zero mismatches.
2. **`RND` A/B sequence test**, this time exercised through `S-RND`'s
   own entry point directly (unmoved at `$25F8` — it's the trampoline
   address itself, always fixed) rather than through a `JP`: identical
   20-value sequence pre/post.
3. **`CIRCLE` A/B screen-memory test**, same 5 circles: bit-for-bit
   identical screens (the core has now shifted a cumulative 198 bytes
   from its original Alpha position).

### Net result

| | Before (Entry 6) | After (this patch) |
|---|---:|---:|
| Contiguous free ROM block | 459 bytes | **488 bytes** |
| `RND` behavior | baseline | **provably identical** (20/20 sequence match) |
| `CIRCLE` behavior | baseline | **provably identical** (5/5 screens match) |

Two repacking candidates from Entry 2 remain: `NEW_TAN_CORE`→`CIRCLE`
padding (+67), `RND_DEFAULT_TABLE`→`COS` padding (+8) — 75 bytes still
on the table, which would bring the contiguous free block to 563 total
once done.

### Files

- `checkpoint_beta_EXP_ATN_COS_LN_SRND_relocated_Spectrum48_patched.asm`
  — the patched ROM source, current Beta deliverable (supersedes
  Entry 6's).
- `beta_patch5_srnd_regression_test.py` — the dynamic regression suite,
  adapted for this patch.
- `checkpoint_BETA_Decisions.md` — this file.

---

## Entry 8 — `TAN` relocated into `CIRCLE`'s spare padding (sixth repacking patch, done)

**Date:** 2026-07-20
**File:** `checkpoint_beta_EXP_ATN_COS_LN_SRND_TAN_relocated_Spectrum48_patched.asm`,
built on top of Entry 7's deliverable. Supersedes it.
**Status:** Implemented and verified to this project's full `Safety.md`
standard, including a real bug found and fixed mid-verification (see
below).

### What changed

`NEW_TAN_CORE` (67 bytes) moved from its old spot in the main free
block to `$2334` — the spare padding inside `CIRCLE`'s own trampoline
at `$2331`, the same "different routine hosted in a slot's own slack"
pattern Entry 5 used for `NEW_COS_CORE` in `C-ENT`. `CIRCLE`'s own
required header (`JP NEW_CIRCLE_CORE`, 3 bytes) is untouched; `TAN`'s
own trampoline at `$37DA` still reads `JP NEW_TAN_CORE`, just resolving
to the new address now. 11 bytes of slack remain in the 81-byte slot —
the tightest cross-slot fit attempted so far (Entry 2's own estimate).
The routine's internal relative-offset labels (`NT_SKIP_NEG`,
`NT_RECIP_PREP`, `NT_PADE`, `NT_APPLY_RECIP`) are self-contained and
needed no adjustment, exactly as predicted when this move was first
identified as safe back in Entry 2.

Six downstream `ORG`s needed a `-67` shift this time (`ADDR_DISPATCH`,
`SAFE_ZERO_BYTE`, `NEW_RND_BYTE`, `RND_BOOT_STUB`, `CHECK_AND_PLOT`, and
the free-space block) — found by the same complete-file `ORG` sweep
that's worked cleanly since Entry 6, with no missed hardcoded addresses
this time either.

### A real bug: a manual `$FF`-counting mistake, caught mid-verification, not by the process that was supposed to catch it

While extending the free-space `$FF` fill block by the claimed 67
bytes, the hand-written `DEFB` rows actually totaled only **59** bytes
— an off-by-8 miscount in the raw row/comment bookkeeping (8 rows of 8
mislabeled as if there were 8 full rows when there were really 7 plus a
3-byte remainder, i.e. `7×8+3=59`, not `8×8+3=67`). Assembly succeeded
cleanly regardless, and — critically — **the standard free-block
boundary check from Entries 3–7 also didn't catch it**: that check
scans backward from `$3D00`, first skipping past any non-`$FF` bytes
before it starts counting the `$FF` run, so an 8-byte implicit
zero-filled gap sitting right at the edge of the claimed free block was
silently excluded from the count on both sides of the comparison,
making "growth = 67 bytes, exactly as expected" report cleanly even
though 8 of those 67 bytes were actually an unflagged gap, not genuine
free space.

Caught only because the full byte diff showed an unexplained change at
`$3CF8`-`$3CFF`, right at the boundary before the character bitmap
table — a location no relocation in this patch should have touched.
Investigating that anomaly (rather than dismissing it as diff noise)
led directly to the miscount.

**Two things fixed, not one:**
1. **The immediate bug** — corrected the `DEFB` rows to the true 67
   bytes, verified this time by generating the block programmatically
   in Python and checking the count before writing it, not just
   eyeballing row/comment arithmetic by hand.
2. **The verification method's own blind spot** — added a second,
   stricter free-block check alongside the existing boundary scan: read
   every byte from the claimed free-block start straight through to
   `$3D00` and assert all of them are `0xFF`, with no backward-skip step
   that could hide a gap. Confirmed this stricter check independently
   agrees with a precise `DEFB`-only source-level byte count (deliberately
   *not* a naive substring search for `"$FF"` in the source, which
   over-counts by also matching the literal text `$FF` inside comment
   prose like "written as explicit $FF fill" — caught this exact
   over-count trap while investigating, logged here so it isn't
   rediscovered). All three methods (stricter binary scan, precise
   source count, and the original boundary-scan check) now agree
   exactly: 555 bytes, all genuinely `0xFF`.

**Standing lesson, third of its kind in this project** (after the two
stale-`ORG` bugs in Entries 3/5): hand-written repetitive `DEFB` blocks
need the same "don't trust arithmetic done by eye" discipline as
address arithmetic does. Going forward, any `$FF`-fill block extension
should be generated and byte-counted programmatically before being
pasted into the source, not hand-typed with a running comment tally.

### Verification (after the fix)

**Static — full byte diff against the Entry 7 baseline**, 636
differing bytes, all explained:
- Eight small operand sites decoded programmatically: `SAFE_ZERO_BYTE`'s
  four references, boot-time init, `RANDOMIZE` tail, `CIRCLE`/`S-LOOP-1`
  hooks — all confirmed to shift by exactly `67`. Two more 1-byte sites
  inside `S-RND`'s own inlined block (`$25FE`, `$2602`) turned out to be
  its internal `CALL NEW_RND_BYTE` operands, updated because
  `NEW_RND_BYTE` itself is downstream and shifted too — not a new bug,
  just a ripple effect one level deeper than prior patches, confirmed
  once traced.
- `TAN`'s own trampoline operand at `$37DB`: correctly decodes to the
  real relocation jump (`$38CB`→`$2334`), not a ripple shift.
- Free-block boundary: grew from 488 to **555 bytes**, exactly 67 more,
  confirmed by the new stricter every-byte-is-`0xFF` check, not just the
  original boundary scan. Seven contiguous zero-runs remain in the
  touched window — `TAN`'s own new 11-byte slack plus the six
  already-known, untouched pads — cross-checked, nothing stray.
- Character bitmap table: confirmed byte-identical (this specific check
  is what caught the `$FF`-count bug in the first place).

**Dynamic — same suite as Entries 3-7:**

1. **5,216-point direct A/B equivalence** across all nine calculator
   functions: zero mismatches.
2. **`TAN` accuracy sweep** against `math.tan` (skipping points near
   asymptotes, same convention as Alpha's own testing): within
   tolerance, consistent with the documented `~2e-4` near-zero accuracy.
3. **`RND` A/B sequence test**: identical 20-value sequence, addresses
   updated for this patch's shift.
4. **`CIRCLE` A/B screen-memory test**, same 5 circles: bit-for-bit
   identical screens (the core has now shifted a cumulative 265 bytes
   from its original Alpha position).

### Net result

| | Before (Entry 7) | After (this patch) |
|---|---:|---:|
| Contiguous free ROM block | 488 bytes | **555 bytes** |
| `TAN` behavior | baseline | **provably identical** (0/5216 mismatches) |
| `RND` behavior | baseline | **provably identical** (20/20 sequence match) |
| `CIRCLE` behavior | baseline | **provably identical** (5/5 screens match) |

One repacking candidate from Entry 2 remains: `RND_DEFAULT_TABLE`→`COS`
padding (+8 bytes, a perfect zero-waste fit) — the last item on Entry 2's
original worklist, which would bring the contiguous free block to the
full **563 bytes** Entry 2 projected.

### Files

- `checkpoint_beta_EXP_ATN_COS_LN_SRND_TAN_relocated_Spectrum48_patched.asm`
  — the patched ROM source, current Beta deliverable (supersedes
  Entry 7's).
- `beta_patch6_tan_regression_test.py` — the dynamic regression suite,
  adapted for this patch.
- `checkpoint_BETA_Decisions.md` — this file.

---

## Entry 9 — `RND_DEFAULT_TABLE` relocated into `COS`'s spare padding (seventh and final repacking patch, done)

**Date:** 2026-07-20
**File:** `checkpoint_beta_FULLY_REPACKED_Spectrum48_patched.asm`, built
on top of Entry 8's deliverable. Supersedes it. Renamed from the
function-name-concatenation convention used through Entry 8 (which was
becoming unwieldy at seven relocations deep) to reflect that this
entry closes out Entry 2's entire worklist.
**Status:** Implemented and verified to this project's full `Safety.md`
standard.

### What changed

`RND_DEFAULT_TABLE` (8 bytes of pure data — `DEFB 82,97,120,111,102,116,20,12`,
the CMWC generator's default seed table) moved from the main free block
into `COS`'s own spare trampoline padding at `$37AD`. This is the
tightest possible fit in the entire project: `COS`'s slot is exactly
11 bytes (`JP NEW_COS_CORE`, 3 bytes, + 8 bytes of padding that used to
be pure waste), and the table is exactly 8 bytes — **zero bytes of
local slack**, the only perfect zero-waste cross-slot relocation done
across all seven patches. Referenced only via `LD HL,RND_DEFAULT_TABLE`
from `RND_BOOT_SEED` and `RND_RESEED` (both absolute addressing, no
proximity dependency), so the move is safe by the same reasoning
established for every other pure-data or plain-code relocation in this
project.

Two downstream `ORG`s needed a `-8` shift: `RND_BOOT_STUB` and
`CHECK_AND_PLOT`, found via the same complete-file `ORG` sweep used
since Entry 6, plus the free-space block.

### The free-space block was regenerated cleanly this time, not patched again

After Entry 8's `$FF`-counting bug, continuing to hand-append yet
another annotated chunk to an already seven-times-patched block felt
like compounding the same risk rather than fixing it. Instead, this
entry replaced the **entire** `SPARE LOCATIONS` block — header comment
and all `DEFB` rows — with a single freshly generated version: the
exact byte count (`563 = 0x3D00 - 0x3ACD`) was computed and verified in
Python *before* any text was written into the source, not derived from
hand-counted rows with running comments. The resulting block is 71
`DEFB` lines with no more per-entry annotation fragments to keep
consistent by hand.

### Verification

**Static:**
- **Full byte diff against the Entry 8 baseline**: 388 differing bytes.
  Three small operand sites (`$128C`, `$1E5B`, `$2332`) decoded
  programmatically, all confirming exactly `-8`. The remaining ranges
  are `RND_BOOT_SEED`/`RND_RESEED`'s own content (which now embeds a
  much larger absolute-address change, since `RND_DEFAULT_TABLE` didn't
  just ripple-shift by 8 like everything else — it jumped clear across
  the ROM into `COS`'s padding) plus the `CIRCLE` core's content shifted
  8 bytes earlier.
- **Free-block boundary, checked with the stricter every-byte method
  introduced in Entry 8** (not just the original boundary scan): every
  byte from `$3ACD` to `$3CFF` confirmed genuinely `0xFF`. Grew from 555
  to **563 bytes**, exactly 8 more.
- Six contiguous zero-runs remain in the touched window, all matching
  known pre-existing pads (`TAN`'s 11B, `S-RND`'s 18B, `LN`'s 20B,
  `TAN` trampoline's 5B, `ATN`'s 5B, `SQR`'s 4B) — and **`COS`'s own
  8-byte pad is gone entirely**, confirming the zero-waste fit worked
  exactly as designed.
- Character bitmap table: confirmed byte-identical.

**Dynamic:**

1. **5,216-point direct A/B equivalence** across all nine calculator
   functions: zero mismatches.
2. **`RND` A/B sequence test** (exercises the `RND_RESEED` call site):
   identical 20-value sequence pre/post.
3. **A dedicated `RND_BOOT_STUB` cold-boot test**, added specifically
   for this patch since `RND_DEFAULT_TABLE` has *two* independent
   readers and the sequence test above only exercises one of them:
   called `RND_BOOT_STUB` directly on both builds and compared the
   resulting `RND_TABLE` RAM contents byte-for-byte. Both matched each
   other and the known raw default table exactly
   (`5261786f6674140c`), confirming the *other* call site
   (`RND_BOOT_SEED`) also reads the relocated table correctly.
4. **`CIRCLE` A/B screen-memory test**, same 5 circles: bit-for-bit
   identical screens (the core has now shifted a cumulative 273 bytes
   from its original Alpha position).

### Net result — Entry 2's plan, closed out

| | Before (Entry 8) | After (this patch) |
|---|---:|---:|
| Contiguous free ROM block | 555 bytes | **563 bytes** |
| `RND` behavior (both call sites) | baseline | **provably identical** |
| `CIRCLE` behavior | baseline | **provably identical** (5/5 screens match) |

| Entry | What moved | Bytes reclaimed | Running free-block total |
|---|---|---:|---:|
| Alpha (starting point) | — | — | 290 |
| 3 | `NEW_EXP_CORE` → own slot | 52 | 342 |
| 4 | `NEW_ATN_CORE` → own slot | 51 | 393 |
| 5 | `NEW_COS_CORE` → `C-ENT` padding | 30 | 423 |
| 6 | `NEW_LN_TAIL` → own slot | 36 | 459 |
| 7 | `NEW_S_RND` → own slot | 29 | 488 |
| 8 | `NEW_TAN_CORE` → `CIRCLE` padding | 67 | 555 |
| 9 | `RND_DEFAULT_TABLE` → `COS` padding | 8 | **563** |

**563 bytes — exactly what Entry 2 projected**, achieved through seven
patches, each independently baseline-diffed, standalone-verified, and
dynamically A/B-tested against its immediate predecessor with zero
behavioral change at every step. Two real bugs were found and fixed
along the way (stale `ORG`s in Entries 3/5, a hand-counting error in
Entry 8's `$FF` fill) — both caught by this project's own verification
discipline, not by assembly failures, and both logged with enough
detail that neither class of mistake should recur.

No repacking candidates remain from Entry 2. `n-mod-m`'s 15 bytes
(Alpha `Decisions.md`) remain the one deliberately deferred item, still
available if a future patch specifically needs them and is willing to
give up its (obscure) direct-dispatch capability.

### Files

- `checkpoint_beta_FULLY_REPACKED_Spectrum48_patched.asm` — the patched
  ROM source, current and final Beta deliverable for this repacking
  effort.
- `beta_patch7_rndtable_regression_test.py` — the dynamic regression
  suite, adapted for this patch.
- `beta_patch7_rndtable_bootstub_test.py` — the dedicated
  `RND_BOOT_STUB` cold-boot path test.
- `checkpoint_BETA_Decisions.md` — this file.
