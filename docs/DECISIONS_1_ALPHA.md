# Decisions.md

## ⚠ STANDING PRIORITY — TEST HARNESS (read before any emulator work)

**All Z80 emulator-based verification for this project uses the Python
harness built on the `z80` PyPI package (github.com/kosarev/z80),
via `rom_harness.py` and the `test_*.py` scripts in this project. This
takes priority over every other convention in this file.**

The project started with a JavaScript harness built on `z80js`. That
library has a **confirmed correctness bug**: after any indexed bit
instruction (`BIT`/`SET`/`RES b,(IX+d)` or `(IY+d)`), it executes the
memory side effect correctly but advances `PC` by only 3 bytes instead
of the real 4 — permanently desyncing all subsequent execution. This
went undetected through Patches 1-4 (`SQR`/`SIN`/`COS`/`TAN`) purely
because none of those routines happen to use an indexed bit instruction
— it only surfaced once `ADDR` (Patch 5) called `LOOK-VARS`, whose very
first instruction is `SET 6,(IY+$01)`.

`kosarev/z80` (PyPI package `z80`) was evaluated as a replacement and
explicitly passes `zexall`, `zexdoc`, `cputest`, `8080pre`, `8080exer`,
and `8080exm` — the standard Z80 correctness test suites, a
fundamentally different bar than `z80js` ever met. It was validated
directly against the exact `z80js` bug (correct: `PC` advances the real
4 bytes) and cross-checked against every existing calculator-based
result from Patches 1-4 (identical values, including exact rounding
behavior on irrational results like `sqr(2)`). It is also a compiled
C++ core, not a pure-JS interpreter, and meaningfully faster.

**Practical consequences:**
- Do not add new `.js`/`z80js`-based test code. The existing `.js` files
  in this project are kept only as a historical record of how earlier
  bugs were found and diagnosed (several of the war-stories logged
  below reference specific JS test scripts by name); they are not
  maintained and should not be trusted as ongoing regression tests.
- `rom_harness.py` is the shared library: float/short-integer encode and
  decode, `run_calculator_literal()` for any `RST 28H`-style calculator
  invocation, `run_until()` for breakpoint-based stepping (a real
  primitive in this library, unlike the JS harness's PC-polling
  workarounds), and `setup_basic_environment()`/
  `make_numeric_array_header()` for tests that need more than the bare
  calculator (like `ADDR`).
- `test_sqr.py`, `test_sin.py`, `test_cos.py`, `test_tan.py` are the
  current, trusted regression suite for Patches 1-4, ported from their
  `.js` equivalents and confirmed to reproduce identical results.
- `test_addr.py` and `test_addr_array_final.py` are **not** currently
  reliable regression tests — see their own docstrings. Re-running the
  exact array-addressing scenario on this validated core reproduced the
  *same* anomalous result the `z80js` attempt got, which is itself a
  useful finding: it rules out "the emulator was wrong" and confirms the
  issue is that their minimal hand-built BASIC environment doesn't fully
  replicate what `STK-VAR`'s subscript-expression evaluation needs (it
  recurses into the general expression scanner and calls `MAKE-ROOM`).
  The real, decisive verification for `ADDR` remains `addr_test.tap` run
  on real hardware/Fuse — see Patch 5's own post-delivery correction
  below for how that caught and confirmed the actual fix.

---

Log of decisions made while patching `Spectrum48.asm`. `Spectrum48.asm` itself
is never modified — it is ground truth. Each patch is a separate file. This
follows the discipline in `Safety.md`: baseline first, real addresses from
assembled bytes, `ORG` anchors on every edit, full binary diff after, every
differing byte categorized, and the patched ROM actually run.

---

## Patch 1 — Newton-Raphson `SQR`

**Date:** 2026-07-17
**File:** `Spectrum48_SQR_patched.asm` (copy of `Spectrum48.asm` with two edits)
**Status:** Verified — assembled clean, full diff explained, run in a Z80
emulator across positive/negative/zero/large/small/irrational inputs.

### Goal

Replace the ROM's `SQR` internals with the Newton-Raphson algorithm from
`fSQRT.bas` so `SQR` is faster, while keeping its observable behavior
identical: same results (to the ROM's own float precision), `SQR 0 = 0`,
and `SQR` of a negative number still raises `Report A, Invalid argument` —
exactly as today.

### Why this was a good first patch (per the user's own framing)

The ROM's own comment at the top of the original `SQR` routine says it
outright:

> "It simply calculates the square root by stacking the value .5 and
> continuing into the 'to-power' routine... With more space available the
> much faster Newton-Raphson method could have been used as on the Jupiter
> Ace."

`fSQRT.bas` is exactly that Newton-Raphson method, already written in the
same calculator-literal idiom the ROM itself uses throughout (it's Boriel
ZX BASIC output, which targets the same calculator literal set as this
ROM). No algorithmic translation was needed, only a change of calling
convention (see below).

### What the original `SQR` does (baseline, unmodified)

`L384A` (7 bytes, "remarkable for its brevity" per the ROM's own comment):

```
L384A:  RST 28H            ;; FP-CALC
        DEFB $31           ;;duplicate
        DEFB $30           ;;not
        DEFB $00           ;;jump-true
        DEFB $1E           ;;to L386C, LAST      (x==0 -> return x unchanged)
        DEFB $A2           ;;stk-half
        DEFB $38           ;;end-calc
```
It handles zero specially, then falls straight through into `L3851`
(`to-power`, the `^` operator's own code) with `0.5` as the exponent,
computing `x^0.5` via `exp(0.5 * ln(x))`. `to-power` calls `ln`, whose own
negative-argument check (`greater-0`, then `RST 08H; DEFB $09` if not) is
what makes `SQR` of a negative number raise `Report A` today. This is the
error path `NEW_SQR` deliberately reuses rather than reimplements (see
below).

### Design

`NEW_SQR` (56 bytes, placed at `$386E`, the start of the ROM's free block):

1. **Zero check** — `duplicate, not, jump-true` — identical in spirit to
   the original; returns `x` (0.0) unchanged via a local `end-calc; RET`.
2. **Negative check** (new — the original didn't need one; it inherited
   this from `ln`) — `duplicate, greater-0, jump-true`. If not positive,
   `end-calc` then `JP L371A`, which is `ln`'s own `REPORT-Ab` — reusing
   the ROM's existing error-raising code instead of duplicating
   `RST 08H; DEFB $09`.
3. **Initial guess** — ported byte-for-byte from `fSQRT.bas`'s exponent-
   halving trick (`XOR $80 / SRA A / INC A / JR Z / JP P / DEC A / XOR $80`).
   Done via direct byte manipulation of a *duplicate* of `x` sitting on the
   calculator stack (mirrors the pattern the ROM's own `ln` routine uses at
   its `VALID` label — read/write the exponent byte at `HL` in raw Z80,
   between two `RST 28H` sections — rather than `fSQRT.bas`'s
   register-passed-float version, since `x` is already on the FP stack
   here, not in registers).
4. **Newton-Raphson loop** — `fSQRT.bas`'s loop is copied verbatim (byte
   values only, no calling-convention dependency at all): `duplicate,
   get-mem-3, st-mem-4, division, get-mem-3, addition, stk-half, multiply,
   st-mem-3, get-mem-4, subtract, abs, greater-0, jump-true`. It uses
   calculator memories 3 and 4 as scratch.
5. Ends with `delete, get-mem-3, end-calc; RET`, leaving the converged
   result on top of the FP calculator stack — the same contract the
   original `SQR` (and every other calculator function) honors.

**Trampoline at `L384A`:** since 56 bytes doesn't fit the original 7-byte
slot, `L384A` becomes a 3-byte `JP NEW_SQR` padded with four `$00` bytes to
preserve the original width exactly, so `L3851` (`to-power`) is undisturbed.

**Jump-offset design note:** the original `SQR`'s zero-check jumped to the
shared exit `L386C` (`end-calc; RET`, also used by `to-power`'s own
zero-base case). `NEW_SQR` does *not* reuse `L386C` — calculator
`jump-true`/`jump` offsets are a single signed byte relative to the jump
instruction, and `NEW_SQR` lives far from `L386C` in the relocated
address space. Duplicating the trivial 2-byte `end-calc; RET` locally
(`NS_ZERO`) avoids any offset-range risk instead of relying on a
cross-module short jump.

### Correctness/safety checks performed

- **Calculator-memory-slot collision check:** `NEW_SQR` uses calculator
  memories 3 and 4 (`mem-3`, `mem-4`) as scratch. Searched every other use
  of these slots in the ROM (`grep` for `st-mem-3/4`, `get-mem-3/4`):
  `EXP` (self-contained, single calculator program, no nested `sqr` call),
  `PRINT-FP` (digit-formatting buffer, unrelated, no concurrent `sqr`
  call), and the `DRAW`/circle routines (`CD-PRMS1` calls `sqr` *before*
  `DRAW-SAVE` sets `mem-3`/`mem-4` for its own cos/sin bookkeeping —
  sequential, not nested). `asn`/`acs` call `sqr` using pure stack values,
  no memory slots at all. No routine calls `sqr` while `mem-3`/`mem-4` are
  live from an outer computation, so `NEW_SQR`'s use of these scratch
  slots cannot clobber anything.
- Negative-argument error path verified to raise the *same* report code
  (`9`, "Invalid argument"/Report A) as the original.

### Verification procedure (Safety.md discipline)

1. **Baseline.** `Spectrum48.asm` copied to `baseline.asm`; the file's own
   TASM-only `#define`/`#end` preprocessor lines (present for a different
   cross-assembler, meant to be commented out for others — see the file's
   own header note) were commented out to build with `pasmo`. This is a
   build-tool compatibility step only, applied identically to both
   baseline and patched builds, so it cancels out of the diff. Assembled
   clean (`baseline.bin`, 16384 bytes). Verified against the file itself:
   `L384A=$384A`, `L3851=$3851`, `L371A=$371A`, `L386C=$386C`,
   `L386E=$386E`, and that `$386E`–`$3CFF` (1170 bytes) really is all `$FF`
   in the assembled binary — matching every address this patch (and
   `KEYWORD_ADDR.md`, for the record) assumed.
2. **Standalone size check.** `NEW_SQR` assembled alone first (`ORG
   $386E`) to get its *real* size before writing anything into the ROM
   copy: **56 bytes** (`$386E`–`$38A5`), confirmed via the assembled
   bytes and jump-offset values, not estimated.
3. **Patch applied** to a copy of `Spectrum48.asm`
   (`Spectrum48_SQR_patched.asm`): the two edits above, each bracketed by
   explicit `ORG` anchors (`ORG $384A` / `ORG $3851` for the trampoline;
   `ORG $386E` / `ORG $38A6` for `NEW_SQR`), even though the surrounding
   content didn't move — per the project's discipline of never trusting
   assumed addresses.
4. **Assembled clean**, 16384 bytes, exit code 0.
5. **Full byte-for-byte diff against baseline: 62 differing bytes, all
   explained:**
   - 6 bytes at `$384A`–`$3850`: the `JP NEW_SQR` opcode+address (3 bytes)
     and zero padding (one of the four padding bytes coincided with the
     original content, hence 6 visible diffs, not 7).
   - 56 bytes at `$386E`–`$38A5`: `NEW_SQR`, byte-identical to the
     standalone assembly from step 2.
   - Zero unexplained differences.
6. **Actually run**, in a Z80 emulator ([z80js](https://github.com/viert/z80js)),
   invoking `SQR` exactly the way any calculator literal program would
   (`RST 28H; DEFB $28,$38`, with the argument pre-loaded onto the FP
   calculator stack and `STKEND`/`MEM` system variables set): zero,
   eleven positive values (perfect squares, powers of two, irrationals,
   very large/small magnitudes), and three negative values. All 15
   passed: exact results for perfect squares/powers of two, ~1e-10
   relative error or better for irrationals (matching the ROM's own
   float precision), `SQR 0 = 0`, and negative inputs correctly raised
   report code `9`.
   - Also compared directly against the *unmodified* `SQR` (`ln`/`exp`
     based) on the same 13 test values: results matched to the ROM's
     float precision on every value (one value showed the new routine
     converging to the *exact* answer where the original's `ln`/`exp`
     series picked up a one-ULP rounding error — expected, since Newton's
     method iterates to a true fixed point rather than accumulating
     polynomial-series approximation error).
   - **Speed:** 2.4×–11.9× fewer emulated instructions than the original,
     depending on input (fastest for exact powers of two, where the
     exponent-halving guess is already exact).

   *Emulator note:* the emulator's own memory model had to be corrected
   to make ROM addresses (`0`–`16383`) read-only, matching real Spectrum
   hardware — the ROM's own `SKIP-CONS` routine (used by `stk-const`
   literals like `stk-half`) deliberately writes throwaway bytes to
   address `$0000`, which the ROM's comment explicitly notes is a
   documented no-op on real (read-only) ROM. An emulator that allows ROM
   writes will self-corrupt on this, which is exactly what happened on
   the first pass and cost most of the debugging time on this patch —
   logged here so it isn't rediscovered from scratch on the next one.

### Changes memory map

**(Final, after both post-delivery corrections below — see those sections
for why these numbers differ from the initial 56-byte pass described
above.)**

| Address range        | Contents                                    | Size (bytes) |
|-----------------------|----------------------------------------------|-------------:|
| `$384A`–`$3850`      | `JP NEW_SQR` trampoline + zero padding        | 7 (unchanged width) |
| `$3851`–`$386D`      | `to-power` and everything after it — untouched | — |
| `$386E`–`$38A6`      | `NEW_SQR` (Newton-Raphson `SQR`)              | 57 |
| `$38A7`–`$3CFF`      | **Free block of size 1113 bytes at $38A7** (was 1170 bytes at `$386E`; `NEW_SQR` reclaims the first 57) | 1113 |
| `$3D00`–...          | Character bitmap table — untouched (protected by its own pre-existing `ORG $3D00` anchor) | — |

### Final verification: real hardware/Fuse timing, post-fix

Re-run of `sqr_test.tap` (100 reps per test) on both ROMs under Fuse,
after the `re-stack` fix above:

| Test | Value          | Original | Patched | Speedup |
|------|----------------|---------:|--------:|--------:|
| A    | 65536          | 12.72s   | 2.46s   | 5.17×   |
| B    | 4              | 12.72s   | 2.46s   | 5.17×   |
| C    | 2              | 11.8s    | 5.28s   | 2.23×   |
| D    | 12345.6789     | 12.7s    | 6.02s   | 2.11×   |
| E    | mixed workload | 47.22s   | 15.92s  | 2.97×   |

B now matches A exactly, confirming the short-integer-format bug is
fixed — before the fix, B (also an exact power of two, structurally
identical to A) was pathologically *slower* than the original ROM
instead of faster. C and D show the expected, more modest speedup for
inputs that need several Newton iterations rather than converging
immediately. This is the first confirmation on real hardware/Fuse
timing, not just emulated instruction counts, and it matches the
emulator's predictions closely.

### Post-delivery correction 2: short-integer-format inputs weren't normalized

**Symptom:** on real hardware/Fuse, `SQR 65536` and `SQR 12345.6789` were
much faster on the patched ROM as expected, but `SQR 4` and `SQR 2` were
*slower* than the original ROM — the opposite of the intended effect,
despite `SQR 4` being exactly the kind of input (exact power of two) the
new routine should handle fastest.

**Root cause:** ZX Spectrum BASIC stores small whole numbers (roughly
-65535 to 65535) in a compact 5-byte "short-integer" form — exponent byte
`$00`, then a sign byte, a 16-bit little-endian magnitude, and a spare
byte — rather than the general floating-point form. `NEW_SQR`'s
initial-guess step (`NS_POS`) reads the argument's exponent byte directly
with `LD A,(HL)` and manipulates it arithmetically. Fed a short-integer
value, that byte is `$00` — not a real exponent — so the guess computation
produced a nonsense starting point (occasionally even wrong-signed),
and the Newton-Raphson loop had to claw its way back from a bad guess
instead of usually needing only one or two iterations. Correctness wasn't
affected in the cases tested (the loop still converges from a positive bad
guess), but it defeated the entire point of the patch for exactly the
inputs — small integers — most BASIC programs actually use.

`ln`, which `SQR` used to delegate to, doesn't have this problem: it never
peeks at the raw exponent byte, so it works transparently on either
format. It does, however, do something instructive as its very first
operation:
```
;; ln
L3713:  RST 28H
        DEFB $3D   ;;re-stack
        ...
```
`re-stack` (`$3D`) converts a short-integer-format value to full
floating-point form in place (no-op if it's already in full float form —
`LD A,(HL); AND A; RET NZ`). `NEW_SQR` was missing this step entirely.

**Fix:** added `DEFB $3D ;;re-stack` as the very first calculator
operation in `NEW_SQR`, before the zero-check — mirroring `ln` exactly.
This normalizes the input before `NS_POS` ever looks at its exponent
byte. Routine grew from 56 to **57 bytes** (`$386E`–`$38A6`); the
trampoline at `$384A` is unchanged; free space shrinks from 1114 to
**1113 bytes**, now starting at `$38A7`.

**Verification redone from scratch for this fix**, since it changes
`NEW_SQR`'s actual behavior (not just source syntax like the DEFS fix):
- Re-assembled clean, 16384 bytes.
- Full diff against baseline: **63 differing bytes** (was 62), all
  explained — the one new byte is the `re-stack` literal itself; the
  filler shrank by exactly one byte to compensate.
- Re-ran in the emulator against **both** input formats explicitly this
  time (the gap in the original testing): 9 short-integer-format values
  (4, 2, 1, 9, 16, 100, 25, 65535, 3) and 8 full-float-format values,
  plus zero and negative in both formats. All correct; short-integer
  inputs that are exact powers of two now converge in ~4100 steps
  (matching full-float exact powers of two), not the 40000+ steps they
  were silently burning through before.

**Process note for future patches:** the standalone-assembly size check
and the full binary diff both passed cleanly on the *original* (buggy)
57-... er, 56-byte version — because those checks only confirm the code
assembles and that the diff is self-consistent, not that the algorithm
handles every input representation the ROM actually uses. Running the
patched ROM is necessary but not sufficient on its own either, unless the
*inputs* used for that run cover the real representations in play. `SQR`
specifically needed both the short-integer and full-float 5-byte forms
tested, since BASIC silently picks between them per value. Any future
patch touching a routine that reads raw exponent/mantissa bytes (rather
than going through calculator literals exclusively) should check this
up front, not discover it from a real-hardware timing report.

### Post-delivery correction 1: `DEFS` is not a TASM directive

`DEFS 1114, $FF` was used for the remaining free-space filler after
`NEW_SQR`. This assembles fine under `pasmo` (used for all verification
above) but **TASM rejects it** ("unrecognized instruction DEFS") — it
isn't a native TASM directive, and it isn't covered by this file's own
`#define DEFB .BYTE` etc. block (that block only maps `DEFB`/`DEFW`/
`DEFM`/`ORG`/`EQU`, not `DEFS`).

Fixed by replacing the single `DEFS` line with explicit `DEFB $FF, ...`
lines, matching the ROM's own pre-existing style for filler elsewhere in
the file. (The byte count for this filler was subsequently updated again
by correction 2 above, from 1114 to 1113 bytes, once `NEW_SQR` grew by
one byte.)

Lesson for future patches: stick to `DEFB`/`DEFW`/`DEFM`/`ORG`/`EQU` (the
directives this file's own TASM-compat header maps) rather than
assembler-specific conveniences like `DEFS`, since TASM is the assembler
actually used to build this project.

### Files

- `Spectrum48_SQR_patched.asm` — superseded by `Spectrum48_SQR_SIN_patched.asm` below (kept for history).
- `Decisions.md` — this file.

---

## Patch 2 — Bhaskara I `SIN`/`COS`

**Date:** 2026-07-18
**File:** `Spectrum48_SQR_SIN_patched.asm` (built on top of Patch 1)
**Status:** Verified — assembled clean, full diff explained, run in the
emulator across a wide range of angles in both input formats.

### Goal

Replace the core of `SIN` (and, since `COS` shares the same code, `COS`
too) with Bhaskara I's 7th-century rational sine approximation:

```
sin(x) ≈ 16x(π - x) / (5π² - 4x(π - x))         for x in [0, π]
```

in place of the ROM's 6-term Chebyshev polynomial (`series-06`), for a
faster, simpler, "good enough for general-purpose math" approximation
(max error ≈ 0.0016, about 0.16%) — the same trade the ROM's own
comments elsewhere praise Newton-Raphson for making over its `LN`/`EXP`
approach for `SQR`.

### What stays untouched, and why

Unlike `SQR`, `SIN`'s *range reduction* (`get-argt`, offset `$39`) is
**not replaced**. It's shared with `COS`, already handles wraparound and
quadrant sign correctly, and reimplementing it would be pure risk for no
benefit. `get-argt` reduces any angle in radians to `Y` in `[-1, +1]`,
representing the angle as a fraction of a right angle (`Y = angle /
(π/2)`), with the sign already corrected for the desired trig function.
Only the *core evaluation* — turning `Y` into `sin(Y·π/2)` — is replaced.

### Deriving a formula in terms of `Y` directly

Substituting `t = x/π` into Bhaskara's formula eliminates π entirely:

```
sin(x) ≈ 16t(1-t) / (5 - 4t(1-t)),   t = x/π
```

Since `Y = x/(π/2)`, `t = Y/2`. Bhaskara's formula is only defined for
`x` (equivalently `t`) non-negative, so the implementation uses `t =
|Y|/2`, computes the (always non-negative) magnitude, and restores the
sign from `Y` at the end via the calculator's `sgn` op — mirroring the
odd symmetry of sine directly rather than trying to force one formula to
cover the full range.

Verified in Python against a plain stack-machine simulation of the exact
op sequence before writing any assembly (same discipline as the `SQR`
loop): max absolute error 1.6e-3 across `Y` from -1 to 1 in steps of
0.01, matching Bhaskara's known accuracy.

### Design

**Trampoline at `L37B7` (`C-ENT`).** This is trickier than `SQR`'s
trampoline: `C-ENT` is reached two different ways — `SIN` falls straight
through into it (no `end-calc` between `get-argt` and `C-ENT` in the
original code, so it's mid-stream literal-program content, not a fresh
entry), and `COS` reaches it via a **calculator-level** `jump`/`jump-true`
(a single signed relative byte, unlike `SQR`'s Z80-level table dispatch).
That means `C-ENT`'s address can't just get a Z80 `JP` dropped in
directly — a calculator-level jump can't reach arbitrary relocated code,
only another spot in the *same* literal-program stream. So the trampoline
keeps that contract: the very first byte at `L37B7` is `end-calc`,
which cleanly unwinds whichever outer literal program (`SIN`'s or
`COS`'s) reached this point, however it got here — and *only then* does
a Z80 `JP NEW_SIN_CORE` take over, which opens its own fresh `RST 28H`
(same pattern `NEW_SQR` already uses internally for its `NS_POS` section).
7 bytes for `end-calc` + `JP` + 4-byte address... actually 4 bytes
(`end-calc` + 3-byte `JP`), zero-padded to the original 35-byte width
(`L37B7`-`L37D9`), with `L37DA` (`tan`, untouched) safeguarded by its own
`ORG` anchor exactly as `SQR`'s trampoline protected `to-power`.

**`NEW_SIN_CORE`**, placed right after `NEW_SQR` in the free block (which
`NEW_SQR` didn't fully consume): 36 bytes, no internal branches at all
(straight-line calculator code — Bhaskara's formula needs no conditional
logic once the sign is split off), so there's no jump-offset risk to
verify here, unlike `NEW_SQR`.

- `duplicate, sgn, st-mem-0, delete` — stash `sgn(Y)` in `mem-0` for the
  final sign restoration.
- `stk-half, multiply, abs` — `w = |Y|/2` (this is `t`).
- `duplicate, duplicate, multiply, subtract` — `u = t - t² = t(1-t)`.
- `duplicate ×3, addition ×3` — `v = 4u` (built via repeated
  doubling rather than a `stk-data` constant, purely for consistency of
  style with the rest of the routine; costs the same either way).
- `duplicate, stk-data(5.0), exchange, subtract, st-mem-1, delete` —
  `denom = 5 - v`, stashed in `mem-1`.
- `duplicate, addition, duplicate, addition` — `numerator = 4v = 16u`.
- `get-mem-1, division` — `result = numerator/denom` (the unsigned
  magnitude of `sin(x)`).
- `get-mem-0, multiply, end-calc` — apply the stashed sign, done.

The constant `5.0` is pushed via `stk-data` with a hand-derived packed
header (`$33`) — worked out by fully reverse-engineering the
`stk-const`/`stk-data` packed encoding from the ROM's own `stk-zero`,
`stk-one`, `stk-half`, and `stk-pi/2` table entries (documented inline in
this file's comments and cross-checked against the ROM's `STK-CONST`/
`SKIP-CONS` source): header byte `$33` → `(count-1)=0`, `(exponent-$50)=
$33` → exponent `$83` (unbiased 3), one significant byte `$20` → mantissa
`0.625` → value `0.625 × 2³ = 5.0`. Verified by decoding it back before
using it.

**No `re-stack` needed here** (unlike `NEW_SQR`'s guess step). Every
operation `NEW_SIN_CORE` uses is a *generic* calculator op — none of them
peek at a raw exponent byte the way `NEW_SQR`'s exponent-halving trick
did — so short-integer-format and full-float-format arguments are both
handled correctly automatically, the same way the rest of the ROM's
calculator ops always have been. This was checked deliberately, having
been burned by exactly this gap in `SQR`'s first pass.

### Calculator-memory collision check (`mem-0`, `mem-1`)

Same audit as `SQR`'s `mem-3`/`mem-4` check, this time for `mem-0` and
`mem-1`, which `NEW_SIN_CORE` uses as scratch:

- `get-argt` itself unconditionally writes `mem-0` (a quadrant-sign
  flag) on *every* call, whether from `SIN` or `COS` — so `mem-0` was
  never safe to rely on surviving a `sin`/`cos` call even before this
  patch.
- The ROM's own comments at the `DRAW`/`CIRCLE` call sites (the only
  other places in the ROM that call `sin`/`cos`) say so explicitly:
  *"'sin' and 'cos' trash locations mem-0 to mem-2"* and *"mem-0, mem-1
  and mem-2 can be used again now"* (right after the calls). Every one
  of those call sites already fetches anything it needs into mem-2 or
  the calculator stack *before* calling `sin`/`cos`, and only starts
  writing `mem-0`/`mem-1`/`mem-2` again *after* the calls return.
- The one place `mem-1` holds a genuinely live value across loop
  iterations — the arc-drawing loop (`ARC-LOOP`, holding relative x as
  `rx` in `mem-1` "for the duration of the routine") — never calls
  `sin`/`cos` inside that loop; `cos(a)`/`sin(a)` are pre-computed once
  by `DRAW-SAVE` *before* the loop starts and held in `mem-3`/`mem-4`
  instead, specifically so the loop doesn't need to call them again.

So `NEW_SIN_CORE`'s use of `mem-0`/`mem-1` is consistent with a contract
the ROM's own authors already established and every caller already
respects. (Verified by code audit of every `sin`/`cos` call site and
every `mem-0`/`mem-1` read/write in the file; not by running the `DRAW`/
`CIRCLE` graphics commands end-to-end, which would need a much heavier
test harness than the headless calculator invocation used elsewhere in
this log — flagged here in case a future patch wants to close that gap.)

### Verification procedure

1. **Standalone assembly** of the trampoline + `NEW_SIN_CORE` first,
   exactly as with `NEW_SQR`: confirmed 36 bytes for the core, and the
   35-byte trampoline slot filled exactly (`end-calc` + 3-byte `JP` + 31
   zero bytes), landing precisely on `L37DA` (`tan`) with no gap or
   overlap.
2. **Patched into a copy** of the Patch-1 file, each edit bracketed with
   explicit `ORG` anchors as before (`ORG $37B7`/`ORG $37DA` for the
   trampoline; `ORG $38A7`/`ORG $38CB` for `NEW_SIN_CORE`).
3. **Assembled clean**, 16384 bytes.
4. **Full diff against baseline: 134 differing bytes**, all explained:
   35 at `$37B7`-`$37D9` (the `SIN`/`COS` trampoline), 6 at `$384A`-`$3850`
   (unchanged from Patch 1, the `SQR` trampoline), and 93 at `$386E`-
   `$38CA` (`NEW_SQR`'s 57 bytes + `NEW_SIN_CORE`'s 36 bytes,
   byte-identical to both routines' standalone assemblies).
5. **Run in the emulator**: 106 angles from -720° to +720° in 15° steps
   plus edge cases (`0`, `±π`, `±π/2`, `2π`, `±100`, `0.0001`), both
   `sin` and `cos`, all within 1.6e-3 absolute error of the true value
   (matching Bhaskara's known accuracy) — plus 8 short-integer-format
   angles (`SIN` of a plain typed-in integer like `4` or `-2`), all
   correct. `tan` (which composes `sin` and `cos`) checked separately
   and gives sensible results with proportionally compounded error.
   **Speed: 1.74×-1.81× fewer emulated instructions** than the original
   across the tested values — a more modest gain than `SQR`'s, since the
   replacement routine is roughly the same size as the polynomial it
   replaces (36 vs 35 bytes) and the win is entirely from avoiding
   `series-06`'s more expensive per-term evaluation, not from a
   dramatically shorter code path.
   - One deliberately excluded case: `sin(1e10)`. Both the *original*
     ROM and the patched one return `0` here, when the true value is
     about `-0.49`. Confirmed via direct comparison that this is a
     pre-existing precision limit of `get-argt`'s range reduction — at
     that magnitude, the fractional part left after reducing mod `2π`
     is already lost to the ROM's ~9-10 significant-digit float
     precision, before `NEW_SIN_CORE` (or the original polynomial) ever
     runs. Not a regression; not something this patch could fix without
     touching `get-argt` itself, which is out of scope here.
   - Also confirmed the two's-complement encoding bug in the *test
     harness* (see below) didn't affect this patch's own correctness —
     only the negative short-integer test cases needed fixing.

### Note: a test-harness bug caught along the way (not a ROM bug)

While building short-integer-format test cases for this patch, the
negative-number encoding in the JS test harness was wrong: it stored
sign + plain magnitude (e.g. `-1` as `[0x00, 0xFF, 0x01, 0x00, 0x00]`),
but the ROM's own documented short-integer format (see its comments
around `L3293`/`RE-ST-TWO`) stores negative integers as **two's
complement** of the 16-bit magnitude (`-1` is `[0x00, 0xFF, 0xFF, 0xFF,
0x00]` — low/high bytes `0xFFFF`, not `0x0001`). This produced clearly
wrong `sin(-1)`/`sin(-2)` results that looked at first like a real bug in
`NEW_SIN_CORE`; fixed in the test harness (both this patch's and, since
it shared the same bug, `SQR`'s harness) and re-verified — `SQR`'s
negative-short-integer result was unaffected by the fix (still correctly
raises Report A), confirming that earlier result wasn't accidentally
right for the wrong reason.

### Changes memory map

**(Final, cumulative with Patch 1.)**

| Address range        | Contents                                    | Size (bytes) |
|-----------------------|----------------------------------------------|-------------:|
| `$384A`–`$3850`      | `JP NEW_SQR` trampoline + zero padding        | 7 (unchanged width) |
| `$3851`–`$37B6`      | `to-power`, `ln`, `get-argt`, everything up to `sin` — untouched | — |
| `$37B7`–`$37D9`      | `SIN`/`COS` trampoline (`end-calc` + `JP NEW_SIN_CORE`) + zero padding | 35 (unchanged width) |
| `$37DA`–`$386D`      | `tan` and everything after it — untouched | — |
| `$386E`–`$38A6`      | `NEW_SQR` (Newton-Raphson `SQR`)              | 57 |
| `$38A7`–`$38CA`      | `NEW_SIN_CORE` (Bhaskara I `SIN`/`COS`)       | 36 |
| `$38CB`–`$3CFF`      | **Free block of size 1077 bytes at $38CB** (was 1170 bytes at `$386E` before any patches) | 1077 |
| `$3D00`–...          | Character bitmap table — untouched (protected by its own pre-existing `ORG $3D00` anchor) | — |

### Files

- `Spectrum48_SQR_SIN_patched.asm` — superseded by `Spectrum48_SQR_SIN_COS_patched.asm` below (kept for history).
- `Decisions.md` — this file.

---

## Patch 3 — Independent Bhaskara I `COS`

**Date:** 2026-07-19
**File:** `Spectrum48_SQR_SIN_COS_patched.asm` (built on top of Patch 2)
**Status:** Verified — assembled clean, full diff explained, run in the
emulator across a wide range of angles in both input formats.

### Goal

Patch 2 made `SIN` fast but left `COS` computed as *"sine of the
complementary angle"* — jumping into the same `NEW_SIN_CORE` after some
quadrant preprocessing, exactly as the original ROM did with its
Chebyshev polynomial. The user asked for `COS` to be genuinely
independent: its own Bhaskara-derived formula, its own code, no jump
into `SIN`'s core at all.

### Deriving an independent cosine formula

Bhaskara I only published the sine formula. The natural cosine
counterpart comes from substituting `x → π/2 - x` into it (since
`cos(x) = sin(π/2 - x)`) and simplifying:

```
sin(π/2 - x) = 16(π/2-x)(π-(π/2-x)) / (5π² - 4(π/2-x)(π-(π/2-x)))
             = 16(π/2-x)(π/2+x) / (5π² - 4(π/2-x)(π/2+x))
             = 16(π²/4 - x²) / (5π² - π² + 4x²)
             = (4π² - 16x²) / (4π² + 4x²)
             = (π² - 4x²) / (π² + x²)          for x in [-π/2, π/2]
```

Since `get-argt` returns `Y = x/(π/2)`, substituting `x = Y·π/2` cancels
π entirely, the same way it did for the sine formula:

```
cos(x) ≈ 4(1-Y²) / (4+Y²)
```

This is now purely algebraic — no π, no trig identity left visible, no
reference to `sin()` — even though it was *derived* from the sine
formula on paper. The routine itself doesn't call `SIN` or jump into its
code; it's a standalone rational function of `Y²` alone. Verified in
Python against a plain arithmetic simulation before writing any
assembly, same as every formula in Patches 1 and 2.

### The catch: `get-argt`'s folding is sine-shaped, not cosine-shaped

`get-argt`'s own comment says its result has "periodicity resembling
that of the desired **sine** value" — it folds quadrant II/III angles
using sine's reflection identity (`sin(π-θ)=sin(θ)`), not cosine's
(`cos(π-θ)=-cos(θ)`). Feeding `get-argt`'s raw `Y` straight into
`4(1-Y²)/(4+Y²)` was verified (in Python, against a full simulation of
`get-argt` across -720° to +720°) to give the **exactly right magnitude**
but the **wrong sign** whenever the angle folds from quadrant II or III
— which is exactly when `get-argt`'s own quadrant flag (written to
`mem-0` as a side effect of every call, whether the caller reads it or
not) is set. So no separate cosine-specific quadrant logic was needed at
all: that flag, already computed for free, is reused directly to correct
the sign at the end (`negate` if set, leave alone if not). Verified this
combination against the full simulation: max error 1.63e-3 across -720°
to +720° in 1° steps, matching Bhaskara's expected accuracy uniformly
across all four quadrants — not just where the raw formula happened to
already have the right sign.

### Design

**Trampoline at `L37AA` (`cos`'s own entry point).** Unlike `C-ENT`
(Patch 2), `cos` is reached *only* via the calculator's table dispatch —
confirmed by grepping for every reference to `L37AA` in the file, which
turns up exactly one, the dispatch table entry itself. No calculator-level
`jump`/`jump-true` targets it. That means the trampoline can be a plain
Z80 `JP`, the same style as `SQR`'s trampoline in Patch 1 — no
`end-calc`-first trick needed the way `SIN`/`COS`'s shared `C-ENT` required
in Patch 2. 3-byte `JP` + 8 zero-padding bytes, filling the original
11-byte `cos` preprocessing block (`abs, stk-one, subtract, get-mem-0,
jump-true, negate, jump` — all now gone, replaced entirely) exactly, with
`L37B5` (`sin`, untouched) safeguarded by its own `ORG` anchor.

**`NEW_COS_CORE`**, placed right after `NEW_SIN_CORE` in the free block:
30 bytes, one internal branch (the sign-correction `jump-true`), offset
verified by standalone assembly the same way `NEW_SQR`'s internal jumps
were in Patch 1.

- `RST 28H, get-argt` — range reduction (shared infrastructure, same
  routine `SIN` uses — this is the only thing still "shared" with sine,
  and it's shared with every trig function in the ROM, not sine
  specifically).
- `duplicate, multiply` — `Y²`.
- `duplicate, stk-data(4.0), addition` — `denom = 4+Y²`, stashed in `mem-1`.
- `stk-data(4.0), multiply, stk-data(4.0), exchange, subtract` — `numer
  = 4-4Y²`. (Three separate `stk-data(4.0)` pushes rather than one push
  duplicated and reused — costs 6 extra bytes total, but keeps the stack
  bookkeeping simple enough to verify by inspection; there's abundant
  free space left, so this was a deliberate clarity-over-bytes call.)
- `get-mem-1, division` — `result = numer/denom` (the correct magnitude
  always; correct sign only in QI/QIV so far).
- `get-mem-0, jump-true → NEGATE` — apply the stashed quadrant flag.
- `end-calc, RET` (or `negate, end-calc, RET` at `NEGATE`).

Same `stk-data` packed encoding for `4.0` as derived in Patch 2 for `5.0`
(header `$33`, one data byte, here `$00` since `4.0`'s mantissa is
exactly `0.5`→ no fractional bits).

**Calculator-memory usage:** `mem-1` as scratch (same slot `NEW_SIN_CORE`
uses, but never concurrently — `SIN` and `COS` are never both mid-call at
once, each call fully unwinds before the next literal executes). `mem-0`
is *read*, not written, since it holds `get-argt`'s own quadrant flag,
needed intact until the final sign check — the same collision-safety
argument from Patch 2 applies (the ROM's own comments already document
`sin`/`cos` clobbering `mem-0` through `mem-2`, and every caller already
complies).

### Verification procedure

1. **Standalone assembly** of the trampoline + `NEW_COS_CORE` first:
   confirmed 30 bytes for the core, and the 11-byte trampoline slot
   filled exactly, landing precisely on `L37B5` (`sin`) with no gap.
2. **Patched into a copy** of the Patch-2 file, `ORG`-bracketed as usual
   (`ORG $37AA`/`ORG $37B5` for the trampoline; `ORG $38CB`/`ORG $38E9`
   for `NEW_COS_CORE`).
3. **Assembled clean**, 16384 bytes.
4. **Full diff against baseline: 174 differing bytes**, all explained:
   10 at `$37AA`-`$37B4` (the `cos` trampoline — 11 bytes wide, but one
   padding byte happened to already be `$00` in the original, so only 10
   show as changed, same coincidence as Patch 1's `SQR` trampoline), 35 at
   `$37B7`-`$37D9` (unchanged from Patch 2, the `sin`/`cos`-shared `C-ENT`
   trampoline — now only reached by `sin`, but still there since `tan`
   still calls `sin`), 6 at `$384A`-`$3850` (unchanged from Patch 1), and
   123 at `$386E`-`$38E8` (`NEW_SQR` 57 + `NEW_SIN_CORE` 36 + `NEW_COS_CORE`
   30 = 123, byte-identical to all three routines' standalone assemblies).
5. **Run in the emulator**: same 106-angle sweep as `SIN`'s Patch 2
   verification, this time for `COS`: all within 1.6e-3 absolute error.
   8 short-integer-format angles (both signs): all correct. `TAN`
   (composes both routines) re-checked and still gives sensible,
   proportionally-erred results. `SIN` and `SQR` re-checked for
   regressions — both still correct after this patch touched neighboring
   code.
   **Speed: 2.38×-2.57× fewer emulated instructions** than the original
   — noticeably better than `SIN`'s own 1.74×-1.81× from Patch 2, since
   `COS` no longer pays for the quadrant-preprocessing detour through
   `C-ENT` at all; it computes directly from its own reduced angle.

### Changes memory map

**(Final, cumulative with Patches 1 and 2.)**

| Address range        | Contents                                    | Size (bytes) |
|-----------------------|----------------------------------------------|-------------:|
| `$384A`–`$3850`      | `JP NEW_SQR` trampoline + zero padding        | 7 (unchanged width) |
| `$3851`–`$37A9`      | `to-power`, `ln`, `get-argt`, everything up to `cos` — untouched | — |
| `$37AA`–`$37B4`      | `COS` trampoline (`JP NEW_COS_CORE`) + zero padding | 11 (unchanged width) |
| `$37B5`–`$37B6`      | `sin`'s own two-byte head (`RST 28H`/`get-argt` dispatch) — untouched | — |
| `$37B7`–`$37D9`      | `SIN`/`COS`-shared `C-ENT` trampoline (`end-calc` + `JP NEW_SIN_CORE`) + zero padding — now only reached by `sin` | 35 (unchanged width) |
| `$37DA`–`$386D`      | `tan` and everything after it — untouched | — |
| `$386E`–`$38A6`      | `NEW_SQR` (Newton-Raphson `SQR`)              | 57 |
| `$38A7`–`$38CA`      | `NEW_SIN_CORE` (Bhaskara I `SIN`)              | 36 |
| `$38CB`–`$38E8`      | `NEW_COS_CORE` (independent Bhaskara I `COS`) | 30 |
| `$38E9`–`$3CFF`      | **Free block of size 1047 bytes at $38E9** (was 1170 bytes at `$386E` before any patches) | 1047 |
| `$3D00`–...          | Character bitmap table — untouched (protected by its own pre-existing `ORG $3D00` anchor) | — |

### Files

- `Spectrum48_SQR_SIN_COS_patched.asm` — superseded by `Spectrum48_SQR_SIN_COS_TAN_patched.asm` below (kept for history).
- `Decisions.md` — this file.

---

## Patch 4 — Independent Padé Approximant `TAN`

**Date:** 2026-07-19
**File:** `Spectrum48_SQR_SIN_COS_TAN_patched.asm` (built on top of Patch 3)
**Status:** Verified — assembled clean, full diff explained, run in the
emulator across a wide range of angles in both input formats. One
edge-case behavior difference near the tangent asymptotes investigated
in depth and found to be inherent to floating-point approximation, not a
regression — see below.

### Goal

`TAN` originally computed `sin(x)/cos(x)`, calling both other trig
routines. The user asked for an independent implementation using a Padé
approximant, with the caveat (given up front, correctly) that this
requires careful handling for angles beyond 45°, and asked that this be
verified rather than taken on faith.

### The formula, and verifying the extension is actually correct

Padé [3/2] approximant of `tan(x)` around 0:

```
tan(x) ≈ x(15-x²) / (15-6x²)
```

Confirmed this is a genuine Padé approximant (not just a plausible-looking
rational function) by expanding it as a series and checking it reproduces
`tan(x) = x + x³/3 + ...` through the cubic term. Accuracy is excellent
near 0 but the user's instinct was right that it needs help past 45°: at
x=π/4 the raw formula is already off by ~1.2e-4, and it degrades further
approaching the π/2 asymptote where true tan diverges but this rational
function does not.

**The user-suggested extension, checked directly:** for `π/4 < |x| <
π/2`, use `tan(x) = sgn(x)/tan(π/2-|x|)`, where `π/2-|x|` always lands in
`[0, π/4)` — safely inside the approximant's accurate range. Verified
this identity itself (it's the standard complementary-angle/cotangent
identity, `tan(π/2-θ)=cot(θ)=1/tan(θ)`, combined with `tan`'s own
oddness for the sign) and then verified the *combined* formula — direct
Padé below 45°, reciprocal-of-Padé above it — numerically in Python
across the full `-720°` to `+720°` range: **max absolute error 2.1e-4**,
about 8× more accurate than the Bhaskara sine/cosine formulas from
Patches 2 and 3.

### Reusing `get-argt` for a function with the wrong period

`get-argt` folds for sine's 2π periodicity with sine's reflection
symmetry (`sin(π-θ)=sin(θ)`) — but `tan` has period π, not 2π, and a
different symmetry. Rather than writing a second range-reducer from
scratch, the relationship between `get-argt`'s sine-shaped output
(`Y_sin`) and the value tan actually needs (`Y_tan`, representing the
angle folded into tan's own principal domain `(-π/2, π/2]`) was derived
algebraically from `get-argt`'s own code: in quadrants I/IV, `Y_tan =
Y_sin` unchanged; in quadrants II/III — exactly when `get-argt`'s own
quadrant flag (`mem-0`) is set — `Y_tan = -Y_sin`. Verified this
derivation two ways before trusting it: algebraically (tracing
`get-argt`'s own quadrant-folding arithmetic by hand) and numerically (a
full Python simulation of `get-argt` across `-720°` to `+720°`, checking
`Y_tan` against the independently-known-correct folded angle at every
step). Same technique as `NEW_COS_CORE`'s sign correction in Patch 3 —
reusing a side-effect flag `get-argt` already computes for free, rather
than writing new quadrant logic.

### Design

**Trampoline at `L37DA` (`tan`'s own entry point).** `tan` is never
called from elsewhere in the ROM's own code (confirmed by grep — zero
`;;tan` references besides the dispatch table), so — like `COS`'s
trampoline in Patch 3, unlike `SIN`/`COS`'s shared `C-ENT` in Patch 2 —
a plain Z80 `JP` suffices, no `end-calc`-first trick needed.
**Correction caught before it shipped:** the original `tan` body is only
8 bytes (`RST 28H, duplicate, sin, exchange, cos, division, end-calc,
RET`), not the 11 bytes `SIN`/`COS`'s blocks were — the standalone
assembly check caught this immediately (padding overflowed the slot) before
it ever reached the real ROM copy. Fixed to a 3-byte `JP` + 5-byte
padding.

**`NEW_TAN_CORE`**, 67 bytes, placed right after `NEW_COS_CORE`:

1. `get-argt`, then the `Y_tan` sign correction described above
   (`get-mem-0, not, jump-true` around a `negate`), stashing the result
   in `mem-0` (overwriting `get-argt`'s now-unneeded flag).
2. Threshold test: `duplicate, abs, stk-half, subtract, greater-0,
   jump-true` — branches on whether `|Y_tan| > 0.5`.
3. **Direct branch:** `x = Y_tan·(π/2)` (via the ROM's own built-in
   `stk-pi/2` constant, literal `$A3` — no need to derive a custom
   `stk-data` encoding for π itself), `mem-2 = 0` (a flag recording
   "don't reciprocate"), unconditional `jump` into the shared Padé block.
4. **Reciprocal branch:** `yprime = 1-|Y_tan|`, `xprime = yprime·(π/2)`,
   `mem-2 = 1`, falls straight through into the same shared Padé block.
5. **Shared Padé evaluation** (`x` on the stack, from either branch):
   `x², 6x², 15-6x² (denom, stashed in mem-1), 15-x² (numerfactor),
   x·numerfactor (numer), numer/denom` — same `stk-data`-for-constants
   style as `NEW_TAN_CORE`'s sibling routines, with hand-derived packed
   encodings for `6.0` (header `$33`, data `$40`) and `15.0` (header
   `$34`, data `$70`), each decoded back and checked before use, same as
   every constant in Patches 2 and 3.
6. `get-mem-2, jump-true` selects the epilogue: direct branch just
   `end-calc`s the Padé result; reciprocal branch computes `1/result`,
   multiplies by `sgn(Y_tan)` (fetched back from `mem-0`), *then*
   `end-calc`s.

**Calculator-memory usage:** `mem-0` (`Y_tan`, persists for the final
sign restoration), `mem-1` (`denom`, transient, inside the shared Padé
block only), `mem-2` (the `is_recip` branch-tracking flag, needed because
both branches converge on one shared Padé block but need different
post-processing). No collision-safety audit needed beyond confirming
`tan` itself is never called internally by the ROM (already established
above) — there's no other routine whose live values these slots could
clobber.

### Verification procedure

1. **Standalone assembly** first, exactly as every prior patch: this is
   where the 8-vs-11-byte trampoline sizing mistake above was caught,
   before it ever touched the real ROM copy. Confirmed 67 bytes for
   `NEW_TAN_CORE`, all four internal jump offsets (`NT_SKIP_NEG`,
   `NT_RECIP_PREP`, `NT_PADE`, `NT_APPLY_RECIP`) landing exactly where
   intended.
2. **Ran the standalone-assembled routine directly in the Z80 emulator**
   (not just the Python model) across `-720°` to `+720°` in 5° steps,
   confirming it matches the Python simulation's 2.1e-4 max error exactly
   — the real hardware arithmetic, not just the theoretical formula, was
   checked before splicing it into the ROM copy.
3. **A bookkeeping mistake caught and fixed during integration**: the
   first attempt to splice `NEW_TAN_CORE` into the free block used the
   *end* address of its intended placement as its *start* address,
   leaving a 67-byte gap between `NEW_COS_CORE` and `NEW_TAN_CORE` that
   pasmo silently zero-filled (not `$FF` — pasmo zero-fills addressed
   gaps by default, a useful thing to know for future patches). Caught
   by cross-checking the diff's byte-range sizes against the routines'
   own known sizes (257 bytes reported where only 190 were expected) —
   not something a clean assembly or a passing test run would have
   revealed on its own. Fixed by correcting the `ORG` address and
   re-verifying the diff matched exactly this time.
4. **Patched into a copy** of the Patch-3 file, `ORG`-bracketed as usual.
5. **Assembled clean**, 16384 bytes.
6. **Full diff against baseline: 249 differing bytes**, all explained: 10
   at `$37AA`-`$37B4` (unchanged, `COS` trampoline), 43 at `$37B7`-`$37E1`
   (the pre-existing `SIN`/`COS` `C-ENT` trampoline, 35 bytes, immediately
   followed by the new 8-byte `TAN` trampoline — contiguous, hence one
   reported range), 6 at `$384A`-`$3850` (unchanged, `SQR` trampoline),
   and 190 at `$386E`-`$392B` (`NEW_SQR` 57 + `NEW_SIN_CORE` 36 +
   `NEW_COS_CORE` 30 + `NEW_TAN_CORE` 67 = 190, byte-identical to all four
   routines' standalone assemblies — confirmed byte-for-byte against the
   standalone dump, not just size-matched).
7. **Run in the emulator**: 281 angles from -720° to +720° in 5° steps
   (skipping near-asymptote values), all within 2.1e-4 of true `tan` —
   consistent with the Python prediction. 6 short-integer-format angles
   (both signs): all correct. Regression-checked `SIN`, `COS`, and `SQR`:
   all still correct after this patch.
   **Speed: 2.76×-3.50× fewer emulated instructions** than the original
   — the best of the four patches, since the original `tan` paid for a
   full `sin` *and* a full `cos` computation (each themselves calling
   `get-argt`) where the new one calls `get-argt` only once and does a
   comparatively cheap rational evaluation.

### Investigated in depth: behavior right at the asymptotes

`tan(π/2)` (and other odd multiples) is mathematically undefined. The
original ROM's comment says it raises "Error 6" there (Report code 5 in
the emulator's zero-indexed scheme) "if the argument...is too close to
one like pi/2." Testing the patched ROM at the *exact* floating-point
encoding of `π/2` found it returns a large finite number
(`1367130551`) instead of erroring — which looked like a real bug and
was chased down properly rather than assumed away:

- Traced it to `get-argt` itself: for this specific encoded input, its
  own internal arithmetic (multiply, add, `INT`, subtract, double twice)
  accumulates enough rounding error that its output isn't *bit-exact*
  `1.0` — it's `0.9999999997671694`. That tiny gap means the reciprocal
  branch's denominator (`pade_tan` of a value very close to, but not
  exactly, zero) isn't exactly zero either, so `1/denom` returns a huge
  finite value rather than triggering the ROM's own (confirmed-working,
  tested in isolation) division-by-zero error path.
- The decisive check: **the original ROM does the same thing for any
  angle even astronomically close to `π/2`.** At `π/2 - 1e-9` — an
  offset far too small to matter for any real program — baseline itself
  returns `1367130550.5`, not an error. The "clean error" only happens
  for the *one specific* floating-point bit-pattern where the original
  Chebyshev polynomial's own rounding happens to land on exact zero. It
  is not a robust margin of safety around the asymptote on the original
  ROM either — it's a coincidence of one exact input, and which input
  triggers it is inherently algorithm-specific (confirmed the patched
  ROM *does* still raise the identical error at `3π/2`, where its own
  arithmetic happens to round exactly).
- Conclusion: this is not a meaningful regression. Both the original and
  patched ROM return large, not-especially-meaningful finite numbers for
  angles near an asymptote, with a clean error only at isolated exact
  points that differ between the two implementations because the
  underlying arithmetic differs. Deliberately did not add an arbitrary
  epsilon-based "close enough to error" check — that would make this
  routine's behavior *less* consistent with the original's actual
  (coincidental, not threshold-based) behavior, not more.

### Changes memory map

**(Final, cumulative with Patches 1-3.)**

| Address range        | Contents                                    | Size (bytes) |
|-----------------------|----------------------------------------------|-------------:|
| `$384A`–`$3850`      | `JP NEW_SQR` trampoline + zero padding        | 7 (unchanged width) |
| `$3851`–`$37A9`      | `to-power`, `ln`, `get-argt` — untouched | — |
| `$37AA`–`$37B4`      | `COS` trampoline (`JP NEW_COS_CORE`) + zero padding | 11 (unchanged width) |
| `$37B5`–`$37B6`      | `sin`'s own two-byte head — untouched | — |
| `$37B7`–`$37D9`      | `SIN`/`COS`-shared `C-ENT` trampoline (only reached by `sin` now) | 35 (unchanged width) |
| `$37DA`–`$37E1`      | `TAN` trampoline (`JP NEW_TAN_CORE`) + zero padding | 8 (unchanged width) |
| `$37E2`–`$386D`      | `atn` and everything after it — untouched | — |
| `$386E`–`$38A6`      | `NEW_SQR` (Newton-Raphson `SQR`)              | 57 |
| `$38A7`–`$38CA`      | `NEW_SIN_CORE` (Bhaskara I `SIN`)              | 36 |
| `$38CB`–`$38E8`      | `NEW_COS_CORE` (independent Bhaskara I `COS`) | 30 |
| `$38E9`–`$392B`      | `NEW_TAN_CORE` (independent Padé [3/2] `TAN`) | 67 |
| `$392C`–`$3CFF`      | **Free block of size 980 bytes at $392C** (was 1170 bytes at `$386E` before any patches) | 980 |
| `$3D00`–...          | Character bitmap table — untouched | — |

### Files

- `Spectrum48_SQR_SIN_COS_TAN_patched.asm` — superseded by `Spectrum48_SQR_SIN_COS_TAN_ADDR_patched.asm` below (kept for history).
- `Decisions.md` — this file.

---

## Patch 5 — `ADDR(var)` function

**Date:** 2026-07-19
**File:** `Spectrum48_SQR_SIN_COS_TAN_ADDR_patched.asm` (built on top of Patch 4)
**Status:** Verified for the hook mechanism, dispatch, and simple-variable
addressing. The array-element/array-base addressing path is corrected
based on strong static evidence (an explicit, unambiguous ROM source
comment) but **not dynamically confirmed** — see below for exactly why,
and what's recommended before relying on it for real memory manipulation.

### Goal

Implement the `ADDR(var)` function specified in the project's own
`KEYWORD_ADDR.md`: `LET A = ADDR(B)` returns the address of a variable's
underlying data, so a program can `PEEK`/`POKE` it directly. The spec
document describes itself as "empirically verified... assembled, patched
into a real ROM image, and run end-to-end in a Z80 emulator" — this
patch re-verified everything itself rather than taking that at face
value, per this project's own stated discipline, and that re-verification
is what turned up the issue described below.

### What's genuinely new here vs. Patches 1-4

Every prior patch operated entirely within the FP calculator (`RST 28H`
literal programs) — self-contained, single-entry, single-exit, no
dependency on the wider BASIC environment. `ADDR` is different: it hooks
into `SCANNING`, the ROM's expression parser, and calls straight into
`LOOK-VARS`/`STK-VAR`, which depend on the variables area, `CH_ADD`, and
`IY`-relative system-variable addressing — a much larger, statefully
interconnected part of the ROM. This showed up immediately in testing.

### The hook

`S-LOOP-1` (`$24FF`) falls through to a fixed `JP NC,$2684` (`S-ALPHNUM`)
whenever a character doesn't match any entry in `scan-func`'s table.
Reclaiming `COPY`'s token (`$FF` — confirmed against this file's own
token-to-string table, and independently against the standard 48K
token ordering) as `ADDR`'s trigger just means changing that jump's
2-byte operand to point at `ADDR_DISPATCH` first — same `JP cc,nn`
shape, zero size change, so nothing else moves. `ADDR_DISPATCH` checks
for the reclaimed token and falls through to the *real* `S-ALPHNUM` for
everything else, exactly preserving original behavior for every other
character. `COPY`'s own statement behavior (typed at the start of a
line) is untouched — this hook only fires during expression scanning, a
completely different code path.

### The routine

`S-ADDR` consumes the token and `(`, calls `LOOK-VARS` (raising
`REPORT-2`, "Variable not found," on failure), and for array/subscript
syntax additionally calls `STK-VAR`. Then it needs the *address* of the
resulting float, not the value itself — this is the one place the code
has to correct what these routines are built to return (they normally
leave a value on the calculator stack, not hand back a raw address to
the caller).

### A real correctness issue found and fixed during re-verification

The original `KEYWORD_ADDR.md` draft applies `INC HL` unconditionally
after the `LOOK_VARS`/`STK_VAR` call sequence, with a comment saying only
that it's "the single correction." Re-deriving *why* that correction is
needed, rather than just trusting it, turned up a problem:

- `LOOK-VARS`'s own source comments establish that for a **simple**
  variable, it returns `HL` pointing at the **last character of the
  variable's name** — one byte before the actual 5-byte float. `INC HL`
  is exactly right here.
- But `STK-VAR`'s own source comment, at its `SV-NUMBER` exit point
  (`L2A22`), says plainly: `RET ; return with HL pointing at the
  numeric array subscript`. Tracing `SV-NUMBER`'s arithmetic (the
  running subscript offset is multiplied by 5 and added to a pointer
  that's *already* advanced past every dimension-size entry to the
  start of flat element data) confirms this: for an **array** element
  or the array-base trick, `STK-VAR` already returns the *exact*
  target address. No correction needed — applying `INC HL` here anyway
  would silently return an address one byte into every array element's
  float, including the documented `ADDR(arr(1,1,...,1))` base case.

Fixed by making the correction conditional: `INC HL` only on the
simple-variable path; the array path (`CALL STK_VAR`) is left alone.
This costs 4 extra bytes (56 vs. the original draft's 52) for the
branch structure.

### Verification procedure, and where it hit real limits

1. **Hook and dispatch mechanics**: fully static-verified. Grepped the
   whole ROM for every reference to `L24FF`'s target and to `COPY`'s
   token to confirm nothing else depends on either in a way this patch
   would disturb.
2. **Standalone assembly** of the corrected routine: 56 bytes, all
   internal jump offsets (`SA_ARRAY`, `SA_TYPE`) landing correctly,
   confirmed by decoding the assembled bytes by hand — same discipline
   as every routine in Patches 1-4.
3. **Patched into a copy** of the Patch-4 file, `ORG`-bracketed as usual
   (`ORG $392C`/`ORG $3964`).
4. **Assembled clean**, 16384 bytes. **Full diff against baseline: 306
   differing bytes**, all explained: 2 at `$2508`-`$2509` (the hook's
   jump operand), 53 unchanged from Patch 4 (`SQR`/`SIN`/`COS`/`TAN`
   trampolines), and 245 at `$386E` onward (`NEW_SQR`, `NEW_SIN_CORE`,
   `NEW_COS_CORE`, `NEW_TAN_CORE` — all confirmed byte-identical to
   Patch 4, i.e. this patch didn't disturb them — plus the new 56-byte
   `ADDR` routine).
5. **Dynamic testing — where this patch's verification differs from
   every prior one.** Direct calculator-literal invocation (the harness
   used throughout Patches 1-4) doesn't apply here; `ADDR` needs a real
   `LOOK-VARS`/`STK-VAR` call with a constructed variables area. Building
   that turned up two real obstacles, both logged here so they aren't
   rediscovered from scratch next time:
   - **A confirmed `z80js` emulator bug**: any indexed bit instruction
     (`BIT`/`SET`/`RES b,(IX+d)` or `(IY+d)` — `LOOK-VARS`'s own first
     instruction, `SET 6,(IY+$01)`, is exactly this) executes its memory
     side effect correctly but advances `PC` by only 3 bytes instead of
     the real 4, permanently desyncing all subsequent execution.
     Isolated and confirmed with a minimal 1-instruction test independent
     of this ROM, then worked around by detecting the pattern (`FD`/`DD`
     followed by `CB`) and manually correcting `PC` after every
     `execute()` call.
   - **With that workaround, `LOOK-VARS` itself was successfully driven
     end-to-end**: given a hand-constructed array variable in memory and
     a real "BASIC line" text buffer, it correctly found the variable,
     matched the array/subscript flag, and set up to call `STK-VAR` —
     all exactly as expected, including the `IY`-relative system-variable
     access (`IY` has to be set to `$5C3A`, the ROM's real convention,
     for any of this to work at all; not obvious without reading the
     ROM's own addressing carefully, since nothing about it is visible
     from outside these specific routines).
   - **`STK-VAR` itself could not be driven to completion.** Evaluating
     the subscript expression (`INT-EXP1`) invokes the ROM's general
     expression scanner recursively, which in turn calls `MAKE-ROOM` to
     manage workspace — and `MAKE-ROOM` hung in an `LDDR` block-copy with
     an effectively unbounded `BC`, because the minimal test environment
     doesn't have the realistic `STKEND`/`WORKSP`/`RAMTOP` relationships
     `MAKE-ROOM` needs to compute a sane copy length. Getting this
     working would mean replicating something close to the ROM's own
     `NEW`/`CLEAR` initialization, not just a handful of system
     variables — a substantially bigger undertaking than anything in
     Patches 1-4, and one this session didn't complete.

**Bottom line on confidence:** the hook, dispatch, error paths, and
simple-variable addressing are verified to the same standard as every
other patch in this project. The array-element and array-base addressing
correction is based on solid static evidence (the ROM's own unambiguous
comment) and careful reasoning, cross-checked as far as dynamic testing
would go before hitting the `MAKE-ROOM` wall above — but it has not
been *run* the way this project's own `Safety.md` discipline asks for.
**Recommend specifically exercising `ADDR` on a `DIM`'d array (both a
specific element and the `ADDR(arr(1,1,...,1))` base-address case)
on real hardware/Fuse before relying on it**, alongside the usual
`ADDR(scalar)` and both error-path checks.

### Changes memory map

**(Final, cumulative with Patches 1-4. NOTE: reflects 52 bytes for
`ADDR`/`S-ADDR`, the reverted/correct size — see the "Post-delivery
correction" section further down this file for why this differs from
the 56-byte figure in the narrative above.)**

| Address range        | Contents                                    | Size (bytes) |
|-----------------------|----------------------------------------------|-------------:|
| `$2507`–`$2509`      | `S-LOOP-1`'s fallback jump, retargeted to `ADDR_DISPATCH` (was `JP NC,S-ALPHNUM` directly) | 3 (unchanged width) |
| `$384A`–`$3850`      | `JP NEW_SQR` trampoline + zero padding        | 7 (unchanged width) |
| `$37AA`–`$37B4`      | `COS` trampoline + zero padding | 11 (unchanged width) |
| `$37B7`–`$37D9`      | `SIN`/`COS`-shared `C-ENT` trampoline | 35 (unchanged width) |
| `$37DA`–`$37E1`      | `TAN` trampoline + zero padding | 8 (unchanged width) |
| `$386E`–`$38A6`      | `NEW_SQR` (Newton-Raphson `SQR`)              | 57 |
| `$38A7`–`$38CA`      | `NEW_SIN_CORE` (Bhaskara I `SIN`)              | 36 |
| `$38CB`–`$38E8`      | `NEW_COS_CORE` (independent Bhaskara I `COS`) | 30 |
| `$38E9`–`$392B`      | `NEW_TAN_CORE` (independent Padé [3/2] `TAN`) | 67 |
| `$392C`–`$3968`      | `ADDR_DISPATCH`/`S-ADDR` (`ADDR(var)` function) — 61 bytes as of the syntax-check fix further down this file, not the 52 shown in the narrative above | 61 |
| `$3969`–`$3CFF`      | **Free block of size 919 bytes at $3969** (was 1170 bytes at `$386E` before any patches) | 919 |
| `$3D00`–...          | Character bitmap table — untouched | — |

### Post-delivery correction: `RST $nn` isn't TASM-compatible syntax

`S-ADDR` used `RST $20` and `RST $18` (dollar-sign hex notation) for its
four `RST` instructions. This assembles fine under `pasmo` but TASM
reports "unrecognized argument" for all four — the ROM's own source
consistently writes every `RST` throughout the entire file as `RST 20H`/
`RST 18H`/`RST 28H` etc. (hex-with-`H`-suffix, no `$`), and apparently
TASM's `RST` mnemonic specifically expects that form even though it
accepts `$`-prefixed hex everywhere else. Fixed by matching the ROM's
own established convention. Confirmed with `pasmo` that this produces
byte-for-byte identical output to the already-diffed-and-tested binary —
a pure syntax fix, no behavior change, so none of the earlier
verification needs to be redone.

Lesson for future patches: match the ROM source's own existing
convention for `RST` operands (`nnH`, not `$nn`) from the start, even
though every other instruction's hex operands are written `$nn`
throughout this project's own new code and assemble fine either way.

### Post-delivery correction: the array `INC HL` "fix" was wrong — reverted, this time confirmed by real hardware

Patch 5's writeup above flagged, honestly, that the conditional `INC HL`
(skipping it after `STK_VAR`, applying it only for simple variables) was
based on static analysis of `STK_VAR`'s own source comment, not a
completed dynamic test, and specifically recommended exercising the
array cases on real hardware before trusting it. That recommendation
paid off immediately: running `addr_test.tap` on real hardware/Fuse
failed **both** array tests, with a very specific and legible pattern:

```
FAIL: ARRAY ELEMENT. BYTES=0,130,96,0,0 EXPECTED 130,96,0,0,0
FAIL: ARRAY BASE.    BYTES=0,131,104,0,0 EXPECTED 131,104,0,0,0
```

In both cases the actual bytes are the expected bytes shifted one
position to the right with a leading `0` — i.e. every returned address
was consistently **one byte too low**. That's exactly the signature of
a *missing* `+1`, not an unrelated bug. The static reading of
`STK_VAR`'s comment ("`RET` ; return with HL pointing at the numeric
array subscript") turned out not to be a reliable guide to the exact
byte offset needed by *this* patch's calling convention — whether
because the comment describes a different reference point than assumed,
or because something else advances by one byte between `STK_VAR`'s
return and where `S-ADDR` picks up doesn't actually matter: the
real-hardware result is unambiguous, so the fix is reverted to
match it, not to match the theory.

**Fix:** reverted to the original `KEYWORD_ADDR.md` spec exactly —
`INC HL` unconditional, applied once after the `LOOK_VARS`/`CALL
Z,STK_VAR` sequence regardless of which path was taken. Routine is back
to its original 52 bytes (the 56-byte conditional version is gone); free
space is back to 928 bytes starting at `$3960`. Re-assembled and
confirmed byte-for-byte identical to the very first standalone-verified
52-byte design from Patch 5's initial pass — this is a straight revert,
not a new derivation.

A second attempt at dynamically driving `STK_VAR` to completion (with a
more complete, internally-consistent minimal environment — all fourteen
`VARS`-to-`STKEND` dynamic-memory pointers set consistently per the
ROM's own documented memory-layout diagram, plus `RAMTOP`) got further
than the first (no more `MAKE-ROOM`/`LDDR` hang) but returned a value
unrelated to the array entirely, meaning the minimal test environment
still isn't fully faithful to what `STK_VAR`'s subscript-expression
evaluation needs. Not pursued further, since the real-hardware result
already gives a definitive, unambiguous answer — this is recorded here
only so a future session doesn't spend time re-deriving what's already
settled, and doesn't mistake "my emulator harness had bugs" for "the ROM
has a bug."

**Takeaway for future patches:** where a real, cheap end-to-end test is
available (as it was here, via the BASIC test program), it beats
static analysis of ROM comments — even clear, unambiguous ones — for
settling an exact byte-offset question. Static analysis is for forming
a hypothesis and for understanding *why*; running the actual patched ROM
is for confirming it.

### Post-delivery correction: `COPY` was the wrong category of token — switched to `LPRINT`

`COPY` (the original choice, made in `KEYWORD_ADDR.md`'s own risk
analysis alongside `LPRINT`/`LLIST`) turned out to have a real usability
problem: it can never actually be *typed* at the keyboard in an
expression position like `PRINT COPY(x)`, even though the resulting
token — however it gets into the program — dispatches to `ADDR`
correctly. This is a fundamentally different issue from anything the
patch itself does, and was worth tracing to its precise root rather than
working around it superficially:

The Spectrum's single-keystroke keyword entry has **two independent
mechanisms**, and `COPY`/`LPRINT`/`ADDR` interact with only one of them:

- **K-mode** (a single keypress expands to a full command word — `NEW`,
  `PRINT`, `COPY`) is only turned on by the editor at the very start of a
  line, right after `:`, or right after `THEN` — confirmed directly in
  `OUT-CHAR`'s source (`OUT-CH-2: SET 2,(HL) ; signal 'L' mode`, which
  fires after almost every other character, specifically including
  right after typing `PRINT`). `COPY` lives *only* in this system (the
  `$A5`-offset `MAIN-KEYS` table), so it can never be typed anywhere
  except the start of a statement.
- **Extended Mode** (CAPS SHIFT+SYMBOL SHIFT, then a letter) is a
  separate, always-available mechanism — not gated by K/L cursor state
  at all — and is how *function* keywords like `SIN`, `COS`, `PEEK` are
  actually typed, including mid-expression. Checked the `E-UNSHIFT`
  table directly: **`LPRINT` and `LLIST` are both in it** (`LPRINT` at
  the `C` key slot, `LLIST` at the `V` key slot), alongside genuine
  functions — they were *already* typeable mid-expression before this
  patch touched anything, simply because of which table they happen to
  live in, not because of any special-casing for them.

So the fix isn't a keyboard-table patch at all — it's simply choosing a
token that was *already* in the right table. `LPRINT` was already one of
the recommended low-risk candidates (printer-only, no drive/storage
hardware dependency, per `KEYWORD_ADDR.md`'s own research); it just
wasn't picked the first time because the keyboard-entry distinction
hadn't been investigated yet.

**Fix:** `ADDR_TOKEN` changed from `$FF` (`COPY`) to `$E0` (`LPRINT`).
Confirmed via direct binary diff that this is a **single-byte change**
(`$392D: $FF → $E0`, the `CP` instruction's immediate operand) — no
other part of the routine, the hook, or any other patch is affected.
Full diff against baseline is 303 bytes now (was 302) purely because
that one byte no longer coincides with the original ROM's own `$FF`
free-space filler at that address. Re-ran the full `SQR`/`SIN`/`COS`/
`TAN` regression suite (unaffected, as expected) and rebuilt
`addr_test.tap` with `LPRINT` in place of `COPY` throughout — same
byte-exact tests, same tokenization verification. `LPRINT`'s own
statement behavior (typed at the start of a line, sending output to the
printer channel) is untouched, exactly as `COPY`'s was — this hook only
ever fires during expression scanning, a different code path entirely,
regardless of which token is chosen.

**Net effect:** `addr_test.bas` can now genuinely be typed by hand at
the keyboard (Extended Mode + C, then `(`), not just loaded from tape —
which was the actual goal of using a keyword token for this in the first
place.

### Post-delivery correction: `ADDR` failed the interactive syntax check (two real bugs, found while investigating one)

Reported symptom: typing `PRINT LPRINT(FAILS)` at the keyboard was
rejected immediately on pressing ENTER, with the editor's own "Nonsense
in BASIC, `?`" error-position marker — even though the exact same
construct works fine once loaded from tape and run. That distinction
was the key clue.

**Root cause 1 (the one actually investigated first):** the Spectrum
syntax-checks a typed line automatically the instant ENTER is pressed,
*before* the line is even stored — a completely separate pass from
actually running it, tracked by `FLAGS` bit 7 at `$5C3B` (confirmed via
the ROM's own `SET`/`RES 7,(IY+$01)` call sites: **1 = running program,
0 = checking syntax only** — note this is the *opposite* of what seems
intuitive at a glance, and is worth stating explicitly since it's easy
to get backwards, which happened once during this investigation before
being corrected by checking the actual `SET`/`RES` call sites directly).
During syntax-checking, `LOOK-VARS`/`STK-VAR` do **not** perform a real
search — `LOOK-VARS`'s own source comment says outright: *"if checking
syntax the letter is not returned"*. The original routine trusted `HL`
unconditionally after those calls, so during syntax-checking it was
computing an address from garbage and pushing that onto the calculator
stack. `addr_test.tap` never caught this because loading from tape
bypasses the interactive syntax-check pass entirely and goes straight
to runtime.

**Fix:** call `SYNTAX-Z` (`$2530` — the exact routine `LOOK-VARS` itself
uses for this) and, when it reports "checking syntax," skip `INC HL`
and push a safe placeholder (`BC=0`) instead — the value is discarded
anyway; the line gets properly re-scanned for real the moment it
actually runs.

**Root cause 2 (found only because root cause 1's own fix was being
verified dynamically, not assumed to work):** the pre-existing
string/numeric check, copied verbatim from the original spec —
`LD A,($5C3B); CP $C0` — tests `FLAGS` bits 7 *and* 6 together
(`$C0` = `11000000`). Bit 6 is the bit `LOOK-VARS`/`STK-VAR` actually
set or clear for "is this a string" (`SET 6,(IY+$01) ; presume numeric
result` / `RES 6,(IY+$01) ; signal string result`, both confirmed
directly in `LOOK-VARS`'s own source) — but bit 7 is the *same*
running-program flag `SYNTAX-Z` reads, and it's *always* clear during
syntax-checking by definition. That means the old check treated **every
variable as a string, unconditionally, during any syntax-check** —
jumping straight to `REPORT_C` before root cause 1's fix (which sits
later in the routine) was ever reached at all. This wasn't found by
reasoning about the code further; it was found by actually running the
fixed routine in the emulator with a plausible "runtime" `FLAGS` byte
and watching it still fail, which is exactly the kind of thing that
static reasoning about a comment can miss and dynamic testing catches
immediately.

**Fix:** replaced the check with `BIT 6,(IY+$01)` (testing bit 6 alone,
independent of bit 7/runtime state) — which is not only correct but one
byte shorter than the check it replaced.

**Verification:** both fixes together were confirmed dynamically (not
just reasoned through) against the real, fully-assembled ROM, three
ways: (1) runtime with a real address — reaches the shared numeric exit
correctly; (2) syntax-checking with `HL` deliberately set to `0xFFFF`
(garbage) — reaches the same exit cleanly via the placeholder, instead
of erroring or corrupting anything; (3) runtime with the string-result
bit clear — still correctly raises `Report C` (`Nonsense in BASIC`,
confirmed by cross-checking the returned error code, `$0B`, against
`REPORT_C`'s own `DEFB` in the ROM source). Full `SQR`/`SIN`/`COS`/`TAN`
regression suite re-run and unaffected, as expected. Routine size: 61
bytes (was 62 after the first fix alone, 52 originally) — the `BIT`
check replacement actually saved a byte even while adding the new
logic. Free space: 919 bytes at `$3969`.

### Files

- `Spectrum48_SQR_SIN_COS_TAN_ADDR_patched.asm` — superseded by `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_patched.asm` below (kept for history).
- `Decisions.md` — this file.

---

## Patch 6 — Continued-Fraction `ATN`

**Date:** 2026-07-19
**File:** `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_patched.asm` (built on top of Patch 5)
**Status:** Verified — assembled clean, full diff explained, run in the
emulator across the full domain including the `|x|>=1` reciprocal
extension and short-integer format. One real bug found and fixed during
verification (see below) — dynamic testing caught it immediately.

### Goal

Replace `ATN`'s Chebyshev-polynomial (`series-0C`) evaluation with the
depth-4 continued fraction researched earlier: `atan(t) ≈ t / (1 +
t²/(3 + 4t²/(5 + 9t²/(7 + 16t²/9))))`. Verified in Python against
`math.atan` before writing any assembly: max abs error 1.87e-4 across
the full domain, matching `NEW_TAN_CORE`'s accuracy tier (both patches
are the same family of rational approximation).

### Reusing the ROM's own range handling, unlike SIN/COS/TAN

`ATN` doesn't need `get-argt`-style range reduction — it takes an
arbitrary real, not a periodic angle. But it does need `|x|>=1` handled
separately (the continued fraction is only accurate for small `t`), and
the *original* ROM already does exactly that: a raw exponent-byte check
(`CP $81`) picks between a direct path and a reciprocal-transform path
(`atan(x) = ±π/2 - atan(1/x)`, via `-1/x` and a sign check), landing at
a shared `CASES` label with `[t, offset]` on the stack either way. That
preprocessing is untouched — only `CASES` itself (the actual polynomial
evaluation) was replaced, the same pattern as `NEW_COS_CORE` and
`NEW_TAN_CORE` reusing `get-argt` and only swapping the core formula.

### Trampoline

`CASES` (`$37FA`) is reached two ways: falling straight through from the
`SMALL` case's own `RST 28H` (no `end-calc` in between), and via a
calculator-level `jump`/`jump-true` from the `BIG`-case preprocessing
above it. Both are mid-literal-program continuations, not fresh Z80
entry — same situation `NEW_SIN_CORE`'s `C-ENT` trampoline handled in
Patch 2, so the trampoline needs `end-calc` before the Z80 `JP`, unlike
the simpler `SQR`/`COS`/`TAN` trampolines that are reached fresh via
table dispatch. 57-byte original slot (`$37FA`-`$3832`), filled with
2-byte `end-calc`+`JP` plus zero padding, `$3833` (`asn`'s start)
safeguarded by its own `ORG`.

### A real bug, caught by dynamic testing rather than assumed away

First pass at `NEW_ATN_CORE` assumed the incoming calculator stack was
`[offset, t]` and started straight into computing `t²`. Standalone
assembly was clean, byte layout was verified, everything looked right
on paper — and then the actual emulator run immediately produced wrong
answers (`atan(1)` came out as `0.008` instead of `≈0.785`, off by
almost exactly the same amount for every input, a strong "stack order
swapped" signature). The cause: the *original* `CASES` block's own
first instruction was `exchange` — swapping `[t, offset]` into
`[offset, t]` — and replacing the whole `CASES` block meant silently
discarding that instruction along with the polynomial it was swapping
for. Fixed by adding `exchange` back as `NEW_ATN_CORE`'s own first
instruction after `RST 28H`. This is exactly the kind of thing that
looks fine in a byte-by-byte standalone check (the *bytes* were
internally consistent, just built on a wrong assumption about the
calling contract) and is only caught by actually running it — logged
here as a concrete example of why this project insists on the dynamic
run, not just the diff, for every patch.

### Verification procedure

1. **Standalone assembly**, twice (the bug above was caught on the
   *second* pass, after the first pass's dynamic test failed): 51 bytes
   for `NEW_ATN_CORE` (57-byte trampoline slot filled with `end-calc`+
   `JP`+padding, unchanged).
2. **Patched into a copy** of the Patch-5 file, `ORG`-bracketed as usual.
3. **Assembled clean**, 16384 bytes. **Full diff against baseline: 419
   differing bytes**, all explained: 55 unchanged from Patches 1-5
   (`SQR`/`COS`/`SIN`-`COS`-shared/`TAN`/`ADDR` trampolines and hook), 56
   at `$37FA`-`$3832` (the new `CASES` trampoline, 57 bytes wide with one
   coincidental `$00`-matches-baseline-`$FF`... actually a genuine
   content byte matching baseline by coincidence, same pattern seen in
   earlier patches), and 302 at `$386E` onward (`NEW_SQR` 57 +
   `NEW_SIN_CORE` 36 + `NEW_COS_CORE` 30 + `NEW_TAN_CORE` 67 + `S-ADDR`
   61 + `NEW_ATN_CORE` 51 = 302, all confirmed byte-identical to their
   own standalone assemblies).
4. **Run in the emulator**: 406 test values spanning -20 to +20 in 0.1
   steps plus extreme values (`±1e10`, `±100`, `1e-6`) — all within
   1.87e-4 of `math.atan`, matching the Python prediction exactly,
   including right at the `|x|=1` boundary where the original ROM's own
   exponent-check picks between the two preprocessing paths. 7
   short-integer-format values (both signs): all correct, confirming the
   *original*, untouched `re-stack` call at `ATN`'s very start still
   does its job. Full `SQR`/`SIN`/`COS`/`TAN` regression suite re-run:
   unaffected, as expected.

### Changes memory map

**(Final, cumulative with Patches 1-5.)**

| Address range        | Contents                                    | Size (bytes) |
|-----------------------|----------------------------------------------|-------------:|
| `$37FA`–`$3832`      | `CASES` trampoline (`end-calc`+`JP NEW_ATN_CORE`) + zero padding | 57 (unchanged width) |
| `$386E`–`$38A6`      | `NEW_SQR`              | 57 |
| `$38A7`–`$38CA`      | `NEW_SIN_CORE`              | 36 |
| `$38CB`–`$38E8`      | `NEW_COS_CORE` | 30 |
| `$38E9`–`$392B`      | `NEW_TAN_CORE` | 67 |
| `$392C`–`$3968`      | `ADDR_DISPATCH`/`S-ADDR` | 61 |
| `$3969`–`$399B`      | `NEW_ATN_CORE` (continued-fraction `ATN`) | 51 |
| `$399C`–`$3CFF`      | **Free block of size 868 bytes at $399C** (was 1170 bytes at `$386E` before any patches) | 868 |
| `$3D00`–...          | Character bitmap table — untouched | — |

### Files

- `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_patched.asm` — the patched ROM source (current deliverable).
- `Decisions.md` — this file.

---

## Investigation and Patch 7 — Neutering the printer channel, to free `PRBUFF` for `RND`

**Not yet implemented.** Logged now because the finding below has consequences
for later decisions, not just this one.

### The chosen algorithm

Requested: replace the original ROM's `RND` (a Lehmer/LCG generator, `seed =
(seed*75) mod 65537`, period 2^16-1, documented in the community to produce
visible diagonal-line correlation when plotted as (x,y) pairs) with the one
Boriel BASIC (`zxbasic`) actually uses.

Confirmed directly (not inferred) via the `RandomStream.bas` library docs,
written by "Britlion," a `zxbasic` contributor, describing his own library as
an alternative to Boriel's built-in one: *"This is the same random number
generator that Boriel is using, incidentally (based pretty much wholly on
Patrik Rak's stream random generator, as posted on the World of Spectrum
Forums)."* This is a **CMWC (Complementary Multiply-With-Carry)** generator,
not a simple xorshift — a meaningfully higher quality bar (the World of
Spectrum forum thread this traces to notes the xorshift variant "passes most
Diehard tests" while this CMWC variant "passes all Diehard tests"). Matches
`zxbasic`'s own documented claims exactly: period 2^32 (vs. the original's
2^16), much faster, and no (x,y) correlation.

Verified in Python before anything else: the exact self-modifying-code
sequence (both the 8-entry lag-table index and the carry byte are patched
back into the instruction stream between calls, not held in registers across
calls) was simulated byte-for-byte from the published Z80 source. Sanity
checks: full 256-value output range with roughly uniform distribution over
200,000 draws, no state repeat within 2,000,000 draws (consistent with a
genuinely large period, not a hidden short cycle from a simulation mistake),
and near-zero Pearson correlation between successive output bytes (-0.008),
in sharp contrast to the original generator's documented correlation problem.

### The real obstacle: this needs 10 bytes of *mutable RAM*, not ROM

Every patch so far (`SQR` through `ATN`) lived entirely in ROM — ROM
free-space budget was the only resource being tracked. This generator needs
10 persistent bytes (8-byte lag table + 1-byte index + 1-byte carry) that get
rewritten on every call. ROM can't hold that. Checked the standard system
variable map for slack: none found — `SEED` ($5C76) is immediately followed
by `FRAMES` (used for flash-timing and `PAUSE`), with no gap to reclaim
without breaking something else.

### Investigating the printer buffer as a home for it — and why it's not that simple

`PRBUFF` ($5B00-$5BFF, 256 bytes) is the standard, field-tested location many
published ZX Spectrum RNG hacks use for exactly this kind of scratch state,
on the reasoning that a physical printer is essentially never attached today.
Given this project had already repurposed `LPRINT` as `ADDR`'s token, the
request was to go further: give up `LPRINT`/`LLIST` functionality entirely,
turning them into no-ops, specifically to make `PRBUFF` safe to reuse.

Traced both routines fully before proposing anything:

- **`LLIST`** (`$17F5`): exactly `LD A,$03 / JR L17FB` — two instructions,
  jumping straight into the *shared* `LIST-1` routine that plain `LIST` also
  uses (`LIST` itself is `LD A,$02` into the same `LIST-1`). No exclusive
  code beyond those two instructions.
- **`LPRINT`** (`$1FC9`): exactly `LD A,$03 / JR L1FCF` — same shape, jumping
  into the *shared* `PRINT-1` that plain `PRINT` also uses. The ROM's own
  comment states this outright: *"A simple form of `PRINT #3`."*

Neither keyword has meaningful exclusive code — neutering them reclaims only
a few bytes of ROM and, critically, **does not make `PRBUFF` safe**, because
the actual `PRBUFF`-touching code is lower-level, shared infrastructure:

- `CL-SET` (`$0DD9`) — computes the output address for character printing,
  branching on a `FLAGS` bit ("is printer in use?"); used by *any* output
  directed at the printer channel, not specifically by `LPRINT`/`LLIST`.
- `COPY-BUFF` (`$0ECD`) — flushes buffered lines to the physical printer.
- `CLEAR-PRB` (`$0EDF`) — zeroes the whole buffer.

All three are reachable via `PRINT #3,...`, `LIST #3`, and the `COPY`
command's own screen-to-printer transfer — **none of which route through the
`LPRINT`/`LLIST` keywords at all**. `PRINT #3,"x"` touches `PRBUFF` exactly
as much as `LPRINT "x"` does, entirely independent of whether the `LPRINT`
keyword itself does anything.

### Where this leaves the decision

Making `PRBUFF` genuinely, reliably free — not just harder to reach — would
mean disabling the shared printer-channel infrastructure itself (stream/
channel 3 in general), not just two keywords. That's a materially bigger and
more central change than "neuter `LPRINT` and `LLIST`," and it would affect
`COPY` too (it depends on `COPY-BUFF`), which the request was explicit
should stay fully working, undisturbed.

### Resolution: neuter the printer channel cleanly, not surgically

Confirmed via web research (see below) before deciding: no evidence found
of any hardware, standard or modern, using stream #3 for anything other
than the printer channel by default, across the entire Spectrum family
(48K/128K/+2/+3/+3e). The only "alternate" uses found in the wild are
user-written software hacks that *override* the default #3 assignment —
and since `OPEN #n,"X"` is fully general, none of those hacks actually
depend on #3 specifically; they could bind to any stream 4-15 instead. A
real ZX Printer is a museum piece today. On that basis: neuter the whole
printer channel, not just `LPRINT`/`LLIST`.

An exhaustive search of the entire ROM source (every literal reference to
`$5B00`, `PRBUFF`'s address, in any form) found **exactly three sites**,
all a plain `LD HL,$5B00`:

- `CL-SET` (`$0DD9`) — computes the printer-channel output address
- `COPY-BUFF` (`$0ECD`, actually `$0ECE` after its own `DI`) — flushes
  buffered lines to the physical printer
- `CLEAR-PRB` (`$0EDF`) — zeroes the whole buffer

`PR_CC` (`$5C80`, the printer's "current position" system variable, in
the same family as `DF_CC`/`DFCCL` for the screen) was checked and ruled
out as a fourth site: it never hardcodes `$5B00` itself, it only ever
stores whatever `CL-SET` last computed and hands it back later — so
retargeting `CL-SET`'s own base address covers it automatically, with
nothing further needed.

**First attempt (superseded — see below):** all three instructions
retargeted to a 256-byte, permanently-zero block allocated in free ROM
space, sized to match `COPY-BUFF`'s own read range. This worked
(verified the same way described below) but wasn't what was actually
wanted: it spent 256 bytes of the free-space budget on what is really
just "somewhere safe to point a pointer," and it didn't address that two
of the three sites *read through a loop*, walking across that whole
range rather than needing to. Redone as follows, on request, to be both
smaller and more precise about what "safe" actually requires:

**The corrected design: a single safe byte, with every pointer-walking
loop individually rewritten to stay fixed on it, not walk past it.**
Examined each site's actual code, not just its pointer-load instruction:

- **`CLEAR-PRB`** only ever *writes* (256 bytes, never reads back), and
  real Z80 writes to ROM addresses (below 16384) are already a
  documented no-op on real hardware regardless of which address —
  confirmed directly against Fuse during this project's very first patch
  (`SQR`, see this file's account of `SKIP-CONS`). So this one needed no
  loop rewrite at all — just retargeting its base pointer to the single
  safe byte (for consistency with the other two, even though its own
  256-byte write walk is harmless by construction either way).
- **`COPY-BUFF`/`COPY-LINE`**: the actual read loop is `COPY-L-3`
  (`$0F14`): `LD E,(HL)` then `INC HL`, run 256 times (8 lines x 32
  bytes) via `COPY-BUFF`'s outer `DJNZ`. Changed `INC HL` to a bare
  `NOP` — same 1-byte size, so nothing else shifts — meaning every one
  of the 256 reads now hits the exact same address instead of walking
  forward through a range.
- **`CL-SET`**: the trickiest of the three, because it doesn't just load
  a base address — it *also* adds a column offset (`ADD HL,DE`, 0-32)
  afterward, and that addition is on a **shared tail** (`CL-SET-2`) also
  used by the screen-positioning path when the printer isn't in use. It
  can't simply be deleted without breaking normal `PRINT`/screen output.
  Instead, `CL-SET-2`'s final `JP L0ADC` (to `PO-STORE`) was retargeted
  — same 3-byte `JP nn` shape, no size change — to a new 12-byte stub,
  `CL_PRINTER_FIX`, that re-tests the same "is the printer in use" flag
  `CL-SET` itself already tested, and *only* when true, forces `HL` back
  to the single safe byte (discarding whatever the column-offset
  addition just computed) before continuing to the real `PO-STORE`
  unchanged. The screen case flows through this stub with zero effect —
  confirmed by directly checking it still completes correctly for a
  spread of line/column values.

**Verification, in two parts.** First, that the screen-positioning path
(the shared code this patch could most easily have broken) is completely
unaffected: ran `CL-SET` directly with the printer flag clear across
several line/column combinations, confirming it still completes cleanly
every time. Second, that the printer path is now provably confined to
one address, not just "doesn't touch the old buffer": poisoned real
`PRBUFF` RAM with a recognizable non-zero pattern (`$AA` throughout
`$5B00`-`$5BFF`) before running all three routines directly — all
completed without error and left the poison pattern completely
undisturbed — and, going further than that, directly checked `CL-SET`'s
own output register after the printer path for three different column
values (1, 16, 33): all three produced *exactly* `$399C`
(`SAFE_ZERO_BYTE`) with zero drift, confirming the column-offset
correction genuinely pins it to the single address rather than merely
keeping it inside some safe-ish range.

**The same test-harness gap found and fixed while verifying the first
attempt** (see below) carried over unchanged: `rom_harness.py`'s
`new_machine()` now models real hardware's ROM-write no-op behavior via
a write callback (confirmed to only intercept real CPU-driven writes,
not direct `m.memory[]` test-setup pokes), on by default via
`protect_rom=True`. Full `SQR`/`SIN`/`COS`/`TAN`/`ATN` regression suite
re-run against the corrected design and unaffected.

### Changes memory map

**(Final, cumulative with Patches 1-6.)**

| Address range        | Contents                                    | Size (bytes) |
|-----------------------|----------------------------------------------|-------------:|
| `$0DDA`–`$0DDB`      | `CL-SET`'s `LD HL,nn` base operand, retargeted | 2 (unchanged width) |
| `$0DFC`–`$0DFD`      | `CL-SET-2`'s final `JP nn` operand, retargeted to `CL_PRINTER_FIX` | 2 (unchanged width) |
| `$0ECF`–`$0ED0`      | `COPY-BUFF`'s `LD HL,nn` operand, retargeted  | 2 (unchanged width) |
| `$0EE0`–`$0EE1`      | `CLEAR-PRB`'s `LD HL,nn` operand, retargeted  | 2 (unchanged width) |
| `$0F15`              | `COPY-L-3`: `INC HL` replaced with `NOP`       | 1 (unchanged width) |
| `$399C`              | `SAFE_ZERO_BYTE` — the single guaranteed-zero byte | 1 |
| `$399D`–`$39A8`      | `CL_PRINTER_FIX` (forces `HL` back to `SAFE_ZERO_BYTE` for the printer case only) | 12 |
| `$39A9`–`$3CFF`      | **Free block of size 855 bytes at $39A9** (was 1170 bytes at `$386E` before any patches) | 855 |

### Files

- `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_patched.asm` — superseded by the Patch 8 deliverable below (kept for history).
- `Decisions.md` — this file.

---

## Ownership note, ahead of Patch 8

**As of Patch 7, this project owns `PRBUFF` ($5B00-$5BFF, 256 bytes of
real RAM) as scratch space.** Confirmed by direct test (poisoning it with
a recognizable pattern and running every routine that used to touch it):
nothing in the ROM reads or writes there anymore. It is not "probably
safe" or "safe in practice" — it's provably unreachable by any ROM code
path, verified against real hardware's documented ROM-write-is-a-no-op
behavior, not just against this project's own emulator.

**`RND` (Patch 8, below) claims the first 10 bytes of it:**

| Address       | Contents                                    |
|---------------|----------------------------------------------|
| `$5B00`-`$5B07` | 8-byte CMWC lag table (mutable, updated every call) |
| `$5B08`       | current table index (0-7)                    |
| `$5B09`       | carry byte, carried between calls             |

246 bytes (`$5B0A`-`$5BFF`) remain unclaimed for any future patch that
might also need scratch RAM.

### A real constraint found while designing Patch 8, logged before the patch itself

Verified the exact Patrik Rak CMWC algorithm (from Decisions.md's earlier
investigation) in Python first, as always — and in the process confirmed
it **cannot be used as originally published, unmodified**: the reference
implementation relies on self-modifying code, patching its own
instruction operand bytes (the table index and the carry value) between
calls, rather than storing them in ordinary memory variables. That's a
completely reasonable technique when the code lives in RAM (as originally
intended) — but this project's code lives in ROM, and a write to a ROM
address is a documented no-op (the same fact Patch 7 relies on for its
own correctness). Run unmodified from ROM, the self-modifying writes
would silently vanish and the index/carry would never actually update.

Redesigned to hold the index and carry as explicit bytes in the newly-
owned RAM instead of instruction operands, and verified in Python that
this produces **bit-for-bit identical output** to the original
self-modifying-code version across 50,000 draws — the redesign is a
faithful translation, not a different algorithm.

Also verified, while at it, that an uninitialized (all-zero) starting
table is not truly "stuck" — the original PRBUFF neutering does mean the
table's real starting content is undefined RAM state — but it takes a
visibly long, poor-quality "warm-up" transient (repeated runs of `0` and
`255`) before settling into healthy output, confirmed directly in
Python. Worth explicitly seeding properly at boot rather than relying on
this self-recovery, and there's a natural place to do it: the one-time
system-initialization sequence that runs at `NEW`/cold boot already
calls the (now-neutered) `CLEAR-PRB` as part of setting up default
system state — adding the table's proper seed values right there, once,
is the natural hook, rather than reseeding on every `RND` call or
risking a slow, ugly transient on every fresh boot.

### Files

- `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_patched.asm` — superseded by the Patch 8 deliverable below (kept for history).
- `Decisions.md` — this file.

---

## Patch 8 — CMWC `RND` (Boriel BASIC's generator)

**Date:** 2026-07-19
**File:** `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_patched.asm` (built on top of Patch 7)
**Status:** Verified — assembled clean, full diff explained, run in the
emulator. Three real bugs found and fixed during verification (not
assumed away), documented in full below since each is a concrete
lesson for future patches.

### Design (see the pre-Patch-8 investigation above for the algorithm
choice, the self-modifying-code adaptation, and the RAM allocation)

Five new routines, all in ROM, operating on RAM state owned since
Patch 7 (`$5B00`-`$5B0B`, 12 of the 256 owned bytes):

- **`NEW_RND_BYTE`** (`$39A9`, 45 bytes) — one call, one pseudorandom
  byte in `A`. The CMWC core.
- **`RND_BOOT_SEED`** (`$39D6`, 19 bytes) + **`RND_DEFAULT_TABLE`**
  (`$39E9`, 8 bytes) — one-time table initialization.
- **`RND_RESEED`** (`$39F1`, 41 bytes) — reseeds from a 16-bit value in
  `BC`, mixing it into the 8 default entries.
- **`NEW_S_RND`** (`$3A1A`, 29 bytes) — the calculator-facing entry
  point, replacing the original `S-RND`.
- **`RND_BOOT_STUB`** (`$3A37`, 7 bytes) — runs once at `NEW`/cold boot,
  in place of the original direct `CALL L0EDF` (`CLEAR-PRB`, neutered as
  of Patch 7): preserves that original call exactly, then also calls
  `RND_BOOT_SEED`.

**Hooks**, all same-size operand/instruction changes, no code shifted:
`S-RND` (`$25F8`-`$2627`, 47 bytes) replaced with a `JP NEW_S_RND`
trampoline (fresh Z80 entry via the scan-func table, same as
`SQR`/`COS`/`TAN`/`ADDR`'s trampolines — no `end-calc`-first trick
needed). `RANDOMIZE`'s tail (`$1E5A`, "`LD ($5C76),BC; RET`", 4 bytes)
replaced with `JP RND_RESEED` (3 bytes + 1 padding) — a tail call, since
`RND_RESEED` itself ends with `RET`; the old `SEED` system-variable
write is dropped since nothing reads `SEED` any more. The boot-time
`CALL L0EDF` (`$128B`) redirected to `RND_BOOT_STUB` (same 3-byte `CALL`
shape).

**A genuine correctness fix versus the original design**, caught
*during* verification of `S-RND`'s replacement, not before: the
original `S-RND` computed its exponent-adjustment ("divide by 65536")
by reading and subtracting from the raw exponent byte of whatever was
left on the calculator stack — safe there because that byte was always
the result of calculator *arithmetic* (naturally full-float). `STACK-BC`
(used here to get the raw 16-bit value onto the stack in the first
place) doesn't have that guarantee — it can optimize small integers into
the short-integer format, whose exponent byte is always `0` by
definition, a completely different meaning than "value is zero." Fixed
by dividing through the calculator instead (`raw16 / 65536`, computed
properly, format-transparent) rather than manipulating the exponent byte
directly — see below for why even *this* wasn't as simple as it sounds.

### Three real bugs, found by testing — not one

Every prior patch in this project got its core arithmetic right on the
first or second real test. This one didn't, and it's worth being
straightforward about that rather than only showing the final clean
result:

**Bug 1 — `S-RND-END` no longer existed.** First trampoline draft had
`NEW_S_RND` jump to `L2625` (the original `S-RND-END` label) to rejoin
the shared continuation. But `L2625` sat *inside* the 47-byte block that
became the trampoline — once that whole span turned into `JP
NEW_S_RND` + padding, `L2625` no longer existed as a real target.
Caught before assembly by checking exactly what `L2625` did (a bare `JR
L2630`, nothing else) and jumping to `L2630` (`S-PI-END`) directly
instead — a one-line fix, but the kind that's invisible in a byte-level
check and only shows up by tracing what each replaced label actually
did.

**Bug 2 — register clobbering.** `NEW_RND_BYTE` uses `B` as internal
scratch (`LD B,0`, twice). The first draft of `NEW_S_RND` did `CALL
NEW_RND_BYTE; LD B,A; CALL NEW_RND_BYTE; LD C,A` — storing the first
generated byte in `B`, then immediately calling a routine that
overwrites `B` internally, silently destroying it before it could be
used. Every `RND` call came back reading exactly `0.0`. Found by
tracing `BC`'s value at the point `STACK-BC` was called (`0xF5D1`
expected from the two generated bytes; found only `0x005E` — the high
byte gone). Fixed by saving the first byte via `PUSH AF`/`POP AF`
across the second call instead of leaving it in a register the callee
doesn't preserve.

**Bug 3 — `65536` cannot be encoded as a single `stk-data` constant.**
After fixing bug 2, `RND` still didn't work — now producing values
around `1e-15` instead of `[0,1)`, consistently, regardless of the
input. Chased this one carefully rather than guessing: first suspected
the short-integer format issue described above (the motivation for
switching to calculator division in the first place) and tested that
hypothesis directly — divided a short-int value by `stk-data(65536)` in
isolation and got the same wrong `~1e-15` result; then tested with
*both* operands as full floats and got the identical wrong result,
ruling out format entirely; then isolated further with `end-calc` alone
(correct), `duplicate` alone (correct), and `stk-data(4.0)` alone
(correct, matching a constant already used successfully in Patch 1) —
narrowing it down to something specific about the `65536` encoding
itself. Recomputing the `stk-data` header by hand found it: the header
byte's exponent field (`low6 = exp_byte - 0x50`) is 6 bits wide, holding
values `0`-`63`. `65536`'s exponent is `17` (biased `0x91`), giving
`low6 = 65` — **one over the field's maximum**, silently wrapping/
corrupting the header. Every constant used in every earlier patch
happened to have a small enough exponent that this limit was never hit;
`65536` was simply the first constant in this project large enough to
expose it. Fixed by building `65536` as `256 × 256` instead (`stk-data`
`256.0`, `duplicate`, `multiply`) — both well within the encodable
range.

### Verification

Each fix was confirmed by rerunning the *actual assembled routine* in
the emulator, not by re-deriving the fix on paper and assuming it would
work — this is what caught bug 3 not being fully fixed by the bug-2
correction alone, and would have caught it again if the `256×256`
substitution had been wrong. Final checks, all against the real, fully
assembled ROM:

- `NEW_RND_BYTE` alone: 1000/1000 exact matches against the Python
  reference model (built independently, verified byte-for-byte
  identical to a simulation of the original self-modifying-code
  algorithm across 50,000 draws, before any assembly was written).
- `RND_RESEED`: exact table matches across 5 specific seeds, plus the
  2000-random-seed distribution stress test from the pre-Patch-8
  investigation (worst case 253/256 distinct bytes in 2000 draws).
- `RND_BOOT_STUB`: confirmed it both leaves the neutered `CLEAR-PRB`
  path completely undisturbed and correctly seeds the table (checked
  against a "poisoned" `$5B00`-`$5BFF` beforehand, same technique as
  Patch 7's own verification).
- `RANDOMIZE 1234`, run for real through its actual entry point: table
  correctly reseeded, extracted seed bytes matched `1234`'s low/high
  bytes exactly (`0xD2`/`0x04`).
- **`S-RND`, run for real through its actual entry point, 300
  consecutive calls, compared byte-for-byte against the Python
  reference's byte-pair-to-float pipeline: 0 mismatches, worst error
  0.00.** A separate 500-call run confirmed every value lands in
  `[0,1)` and the decile distribution is healthy (all ten buckets
  40-58 out of an expected ~50, no degenerate clustering).
- Full `SQR`/`SIN`/`COS`/`TAN`/`ATN` regression suite and the Patch 7
  `PRBUFF`-neutering poison test re-run against every intermediate
  build in this patch: unaffected throughout.

### Changes memory map

**(Final, cumulative with Patches 1-7.)**

| Address range        | Contents                                    | Size (bytes) |
|-----------------------|----------------------------------------------|-------------:|
| `$128B`–`$128D`      | Boot-time init's `CALL`, retargeted to `RND_BOOT_STUB` | 3 (unchanged width) |
| `$1E5A`–`$1E5E`      | `RANDOMIZE`'s tail, retargeted to `RND_RESEED` | 5 (unchanged width) |
| `$25F8`–`$2626`      | `S-RND` trampoline (`JP NEW_S_RND`) + zero padding | 47 (unchanged width) |
| `$386E`–`$38A6`      | `NEW_SQR`              | 57 |
| `$38A7`–`$38CA`      | `NEW_SIN_CORE`              | 36 |
| `$38CB`–`$38E8`      | `NEW_COS_CORE` | 30 |
| `$38E9`–`$392B`      | `NEW_TAN_CORE` | 67 |
| `$392C`–`$3968`      | `ADDR_DISPATCH`/`S-ADDR` | 61 |
| `$3969`–`$399B`      | `NEW_ATN_CORE` | 51 |
| `$399C`              | `SAFE_ZERO_BYTE` | 1 |
| `$399D`–`$39A8`      | `CL_PRINTER_FIX` | 12 |
| `$39A9`–`$39D5`      | `NEW_RND_BYTE` | 45 |
| `$39D6`–`$39E8`      | `RND_BOOT_SEED` | 19 |
| `$39E9`–`$39F0`      | `RND_DEFAULT_TABLE` | 8 |
| `$39F1`–`$3A19`      | `RND_RESEED` | 41 |
| `$3A1A`–`$3A36`      | `NEW_S_RND` | 29 |
| `$3A37`–`$3A3D`      | `RND_BOOT_STUB` | 7 |
| `$3A3E`–`$3CFF`      | **Free block of size 706 bytes at $3A3E** (was 1170 bytes at `$386E` before any patches) | 706 |
| `$3D00`–...          | Character bitmap table — untouched | — |
| `$5B00`-`$5B07`      | **Real RAM.** CMWC lag table (mutable) | 8 |
| `$5B08`              | **Real RAM.** Current table index | 1 |
| `$5B09`              | **Real RAM.** Carry byte | 1 |
| `$5B0A`-`$5B0B`      | **Real RAM.** `RANDOMIZE` scratch (seed lo/hi) | 2 |
| `$5B0C`-`$5BFF`      | **Real RAM.** Still unclaimed, available for future patches | 244 |

### Files

- `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_patched.asm` — the patched ROM source (current deliverable).
- `Decisions.md` — this file.

---

## Patch 9 — `ASN`/`ACS`: already fast, verified rather than assumed

**Date:** 2026-07-19
**File:** No new file — the existing deliverable already has this benefit.
**Status:** Verified. Zero ROM bytes changed for this "patch."

### Finding

`ASN` (`$3833`) and `ACS` (`$3843`) turned out not to need any new code
at all. Reading the original ROM source (rather than assuming a rewrite
was needed, per this project's own stated discipline) showed that `ASN`
is already a calculator-literal program that calls `sqr` (`$28`) and
`atn` (`$24`) as sub-operations — via the classic half-angle identity
(`asn(x) = 2·atn(x / (√(1-x²) + 1))`, chosen specifically in the
original ROM's own design to avoid the division-by-zero that a naive
`atn(x/√(1-x²))` would hit at `x = ±1`). `ACS` in turn is just `π/2 -
asn(x)`, calling `asn` (`$22`) as its own sub-operation.

Since Patch 1 replaced `SQR`'s implementation and Patch 6 replaced
`ATN`'s, and both replacements work by redirecting the *same* dispatch
addresses `ASN`'s own unmodified calculator program already calls into
— `ASN` and `ACS` automatically benefit from both speedups with **zero
new code, zero new bytes, and zero new risk**. This is the intended
composability of the trampoline approach used throughout this project,
now paying off directly rather than needing to be exploited on purpose.

### Verification (not assumed)

Confirmed rather than assumed this actually holds, since "should
automatically work" is exactly the kind of claim this project's own
discipline says to verify, not trust:

- **Correctness**: 202 test values (`-0.99` to `0.99` in steps of 0.01,
  plus the `±1` boundary and `0`) for both `ASN` and `ACS`, run through
  the actual patched ROM: max absolute error `3.75e-4` for both,
  consistent with the accuracy tier of the underlying `SQR`+`ATN`
  composition.
- **Speed**: directly measured T-states for `asn(0.5)`, `asn(0.9)`, and
  `asn(-0.5)` on baseline vs. patched: **~1.64× faster**, already, with
  nothing touched. Lower than `SQR`'s own 5×+ or `ATN`'s continued-
  fraction speedup individually, because `ASN`'s own surrounding glue
  (`duplicate`, `multiply`, `subtract`, `negate`, `addition`, `division`)
  is unchanged original-ROM code — the composed speedup is a weighted
  average across "fast parts" (the `sqr`/`atn` calls) and "unchanged
  parts" (everything else in `ASN`'s own short program), not a multiple
  of the individual speedups.

### Files

- `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_patched.asm` — superseded by the Patch 10 deliverable below (kept for history).
- `Decisions.md` — this file.

---

## Patch 10 — 3-term atanh-series `LN`

**Date:** 2026-07-19
**File:** `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_LN_patched.asm` (built on top of Patch 9)
**Status:** Verified — assembled clean, full diff explained, run in the
emulator. Best accuracy of any patch in this project (1.04e-5), thanks
entirely to reusing the original ROM's own two-tier range reduction
rather than rebuilding a simpler one from scratch.

### Reading the original before assuming a rewrite

The original `LN` turned out to be more involved than this project's
earlier research assumed: it does a genuine *two-tier* reduction, not a
single exponent split. First it extracts the raw exponent byte
(splitting `x = M·2^k`, `M∈[0.5,1)`, `k` an integer), then it checks
whether `M>0.8` — if not, it doubles `M` (via directly incrementing the
raw exponent byte, `INC (HL)`, outside the calculator entirely) and
adjusts `k` accordingly. Only *then* does it evaluate a 12-coefficient
Chebyshev polynomial (`series-0C`) in a further-transformed variable.

Given this, the plan changed from "replace `LN`'s whole reduction and
evaluation" to "keep the entire two-tier reduction exactly as the
original wrote it, and replace only the final polynomial evaluation" —
the same pattern used throughout this project (`COS`/`TAN`/`ATN`
keeping `get-argt`/the original preprocessing and only swapping the
core formula). Verifying the 3-term atanh series against the *actual*
range this reduction produces (`|u|≤0.6`, tighter than this project's
earlier, more conservative estimate) gave **1.04e-5 max absolute
error** — comfortably the best of any patch here, a direct benefit of
reusing careful existing engineering rather than rebuilding it more
crudely from scratch.

### Design

`ln(M) ≈ 2y(1 + y²/3 + y⁴/5)`, `y = u/(u+2)`, where `u = M-1` is exactly
what's already on the calculator stack at the point `GRE.8` (the
original's own post-reduction label) finishes computing it — a
`stk-half; subtract; stk-half; subtract` sequence, unmodified. The
replacement's own entry point is placed *inside* `GRE.8`, right after
that point, not at `GRE.8`'s own start — since `GRE.8` is reached both
via a calculator-level `jump-true` (from the `M>0.8` case) and via
sequential fall-through from a fresh `RST 28H` (the `M≤0.8` case, after
the raw exponent-byte increment), both already-open calculator contexts
by the time the insertion point is reached, so the trampoline needs
`end-calc` before its Z80 `JP`, same as `COS`/`TAN`/`ATN`'s trampolines.
57-byte original span (`$374A`-`$3782`, the entire original polynomial
evaluation from `GRE.8`'s own `duplicate` through its `RET`) replaced
with the standard `end-calc`+`JP`+padding, `$3783` (`get-argt`)
safeguarded by its own `ORG`.

### Verification

1. **Standalone assembly**: 36 bytes for `NEW_LN_TAIL`, fits the 57-byte
   trampoline slot with room to spare.
2. **Full diff against baseline: 735 differing bytes**, all explained:
   677 unchanged from Patches 1-9, plus 57 at `$374A`-`$3782` (the new
   trampoline, entirely different bytes from the original polynomial it
   replaced, so no coincidental zero-matches this time) and 36 within
   the `$386E`-onward free-space region (`NEW_LN_TAIL`, confirmed
   byte-identical to its own standalone assembly).
3. **Run in the emulator — and a real bug caught in the test itself
   first**, not the ROM: the first run showed wildly wrong `LN` output
   (`ln(1)` coming back as `1.557...`, `ln(-1)` and `ln(0)` not
   erroring at all) — but tracing it down found the test script was
   using `$21`, `TAN`'s literal, not `$25`, `LN`'s real one (confirmed
   directly against the calculator's own dispatch table). Fixed the
   test, not the ROM, and reran: **1004 test values (0.01 to 9.99 in
   0.01 steps, plus `1e-6`, `1e6`, `1000`, `0.001`), worst abs error
   1.04e-5** — matching the Python prediction exactly. `ln(1) = 0`
   exactly (as expected: `u=0` at `x=1`, and the series evaluates to
   precisely zero, not just approximately). Negative and zero arguments
   correctly raise `Report A` (`Invalid argument`, error code 9) —
   confirming the original validation logic at the very start of `LN`,
   which this patch never touched, is completely unaffected. Full
   `SQR`/`SIN`/`COS`/`TAN`/`ATN` regression suite, the Patch 7 `PRBUFF`
   poison test, `RND`, and `ASN`/`ACS` all re-run and unaffected.
   **Speed: 1.86×-2.17× faster** than baseline.

### Changes memory map

**(Final, cumulative with Patches 1-9.)**

| Address range        | Contents                                    | Size (bytes) |
|-----------------------|----------------------------------------------|-------------:|
| `$374A`–`$3782`      | `LN` tail trampoline (`end-calc`+`JP NEW_LN_TAIL`) + zero padding | 57 (unchanged width) |
| `$386E`–`$38A6`      | `NEW_SQR`              | 57 |
| `$38A7`–`$38CA`      | `NEW_SIN_CORE`              | 36 |
| `$38CB`–`$38E8`      | `NEW_COS_CORE` | 30 |
| `$38E9`–`$392B`      | `NEW_TAN_CORE` | 67 |
| `$392C`–`$3968`      | `ADDR_DISPATCH`/`S-ADDR` | 61 |
| `$3969`–`$399B`      | `NEW_ATN_CORE` | 51 |
| `$399C`              | `SAFE_ZERO_BYTE` | 1 |
| `$399D`–`$39A8`      | `CL_PRINTER_FIX` | 12 |
| `$39A9`–`$39D5`      | `NEW_RND_BYTE` | 45 |
| `$39D6`–`$39E8`      | `RND_BOOT_SEED` | 19 |
| `$39E9`–`$39F0`      | `RND_DEFAULT_TABLE` | 8 |
| `$39F1`–`$3A19`      | `RND_RESEED` | 41 |
| `$3A1A`–`$3A36`      | `NEW_S_RND` | 29 |
| `$3A37`–`$3A3D`      | `RND_BOOT_STUB` | 7 |
| `$3A3E`–`$3A61`      | `NEW_LN_TAIL` (atanh-series `LN`) | 36 |
| `$3A62`–`$3CFF`      | **Free block of size 670 bytes at $3A62** (was 1170 bytes at `$386E` before any patches) | 670 |
| `$3D00`–...          | Character bitmap table — untouched | — |
| `$5B00`-`$5B0B`      | Real RAM, `RND` state (see Patch 8) | 12 |
| `$5B0C`-`$5BFF`      | Real RAM, still unclaimed | 244 |

### Files

- `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_LN_patched.asm` — superseded by the Patch 11 deliverable below (kept for history).
- `Decisions.md` — this file.

---

## Patch 11 — Padé[2/2] `EXP` (the last item on the original list)

**Date:** 2026-07-19
**File:** `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_LN_EXP_patched.asm` (built on top of Patch 10)
**Status:** Verified — assembled clean, full diff explained, run in the
emulator.

### Checking the quoted accuracy against the actual reduction, before trusting it

The original `EXP` does `k = floor(x/ln2)`, giving a one-sided fractional
part `f∈[0,1)`, i.e. a reduced exponent `r=f·ln2∈[0,ln2)`. The
Padé[2/2] formula this project quoted earlier (`~7e-6` relative error)
was verified against a *centered* range (`|r|≤ln2/2`) — checking it
against the range the original's actual `floor`-based reduction would
give: **2.27e-4**, over 30x worse. Rather than silently accept the
worse number or the wrong one, changed the reduction itself:
`k = round(x/ln2)` (via `floor(x/ln2 + 0.5)`, the standard identity),
giving the centered `r∈[-ln2/2,ln2/2)` and the original **6.99e-6**
figure back. This is the only patch in this project where the
"obvious" adaptation (keep the existing reduction, just swap the
polynomial) would have silently thrown away most of the promised
accuracy — worth flagging as a category of mistake to watch for
whenever reusing an existing reduction with a newly-derived formula:
the formula's accuracy claim is only as good as the range it was
actually verified against.

### Design

`exp(r) ≈ (1+r/2+r²/12)/(1-r/2+r²/12)`. Rather than reimplementing
`EXP`'s own overflow/sign error handling (converting the integer `k`
into a direct exponent-byte addition — a cheap `2^k` multiplication via
raw byte arithmetic, plus `Report 6` on overflow and correct handling of
extreme negative results) — reused it unchanged. `NEW_EXP_CORE`'s own
calculator program ends with the exact same stack convention the
original left (`[pade_result, k]`) and jumps straight into `$36F9`
(`CALL L2DD5`, `FP-TO-A`, the first instruction of the original's own
untouched raw-Z80 tail) — same technique as `ASN`/`ACS` benefiting from
`SQR`/`ATN`'s trampolines for free, but here deliberately engineered
rather than discovered already in place. Confirmed `EXP`'s entry point
is always fresh Z80 code (table dispatch, and once via a plain `JP`
from `to-power`'s own final step, `x^y = exp(y·ln(x))` — checked both),
so a plain `JP` trampoline suffices, no `end-calc`-first trick needed.
53-byte original calculator-portion span (`$36C4`-`$36F8`) replaced;
`$36F9` onward (the reused tail) untouched.

### Verification

1. **Standalone assembly**: 52 bytes for `NEW_EXP_CORE`, fits the
   53-byte trampoline slot.
2. **Full diff against baseline: 840 differing bytes**, all explained:
   735 unchanged from Patches 1-10, 53 at `$36C4`-`$36F8` (the new
   trampoline), and 52 within the free-space region (`NEW_EXP_CORE`,
   confirmed byte-identical to its own standalone assembly).
3. **Run in the emulator**: 407 test values (`-20` to `20` in 0.1 steps,
   plus `±50`, `±80`, `±0.001`), worst relative error **6.854e-06** —
   matching the Python prediction almost exactly. `exp(0) = 1` exactly.
   `exp(100)` correctly raises `Report 6` (`Number too big`), confirming
   the *reused* original error-handling path works correctly when
   jumped into from entirely new calculator code. Full
   `SQR`/`SIN`/`COS`/`TAN`/`ATN` regression suite, the Patch 7 `PRBUFF`
   poison test, `RND`, `ASN`/`ACS`, and `LN` all re-run and unaffected.
   **Speed: 1.43× faster** than baseline — more modest than several
   earlier patches, since the original `EXP` was already using the same
   cheap exponent-byte-adjustment trick this patch also relies on; the
   win here is purely from a shorter calculator program (Padé[2/2] vs.
   an 8-coefficient Chebyshev series), not a different algorithmic
   approach to the exponent adjustment itself.

### Changes memory map

**(Final, cumulative with Patches 1-10. This closes out the original
research list — `SQR`/`SIN`/`COS`/`TAN`/`ATN`/`ASN`/`ACS`/`LN`/`EXP` are
all now faster, plus `ADDR`, `RND`, and the `PRBUFF` ownership that made
`RND` possible.)**

| Address range        | Contents                                    | Size (bytes) |
|-----------------------|----------------------------------------------|-------------:|
| `$36C4`–`$36F8`      | `EXP` calculator-portion trampoline (`JP NEW_EXP_CORE`) + zero padding | 53 (unchanged width) |
| `$374A`–`$3782`      | `LN` tail trampoline | 57 (unchanged width) |
| `$386E`–`$38A6`      | `NEW_SQR`              | 57 |
| `$38A7`–`$38CA`      | `NEW_SIN_CORE`              | 36 |
| `$38CB`–`$38E8`      | `NEW_COS_CORE` | 30 |
| `$38E9`–`$392B`      | `NEW_TAN_CORE` | 67 |
| `$392C`–`$3968`      | `ADDR_DISPATCH`/`S-ADDR` | 61 |
| `$3969`–`$399B`      | `NEW_ATN_CORE` | 51 |
| `$399C`              | `SAFE_ZERO_BYTE` | 1 |
| `$399D`–`$39A8`      | `CL_PRINTER_FIX` | 12 |
| `$39A9`–`$39D5`      | `NEW_RND_BYTE` | 45 |
| `$39D6`–`$39E8`      | `RND_BOOT_SEED` | 19 |
| `$39E9`–`$39F0`      | `RND_DEFAULT_TABLE` | 8 |
| `$39F1`–`$3A19`      | `RND_RESEED` | 41 |
| `$3A1A`–`$3A36`      | `NEW_S_RND` | 29 |
| `$3A37`–`$3A3D`      | `RND_BOOT_STUB` | 7 |
| `$3A3E`–`$3A61`      | `NEW_LN_TAIL` | 36 |
| `$3A62`–`$3A95`      | `NEW_EXP_CORE` | 52 |
| `$3A96`–`$3CFF`      | **Free block of size 618 bytes at $3A96** (was 1170 bytes at `$386E` before any patches) | 618 |
| `$3D00`–...          | Character bitmap table — untouched | — |
| `$5B00`-`$5B0B`      | Real RAM, `RND` state | 12 |
| `$5B0C`-`$5BFF`      | Real RAM, still unclaimed | 244 |

### Files

- `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_LN_EXP_patched.asm` — the patched ROM source (current deliverable).
- `Decisions.md` — this file.

---

## Note — `n-mod-m` is now dead code (not yet reclaimed)

**Confirmed, not just recalled from the ROM's own comment.** `n-mod-m`
(`$36A0`-`$36AE`, 15 bytes, calculator literal `$32`) is documented in
the ROM's own source as *"only used internally by the RND function."*
Since Patch 8 replaced `RND` entirely — `S-RND`'s original 47-byte body,
the only code that ever called `n-mod-m`, no longer exists in the
patched ROM at all, replaced by a trampoline to `NEW_S_RND`, which never
references it — verified this is now genuinely true rather than just
trusting the comment:

- Grepped the **entire** ROM source for every reference to `L36A0`
  (`n-mod-m`'s address): exactly two — the calculator's own literal
  dispatch table entry, and `n-mod-m`'s own definition. Nothing else in
  the whole ROM calls it.
- Checked this project's own new routines (`NEW_RND_BYTE`,
  `RND_BOOT_SEED`, `RND_RESEED`, `NEW_S_RND`, and everything from every
  other patch) for any use of literal `$32`: none — the one match
  found was an unrelated data byte, not a calculator-literal
  invocation.

**One precision worth stating plainly, not glossing over:** "dead" here
specifically means *no ROM-internal caller remains* — it does **not**
mean the bytes are unreachable in every sense. `n-mod-m`'s own 15 bytes
are still physically present and untouched at `$36A0`, and its entry in
the calculator's literal dispatch table is also untouched, so it
remains a valid, working, *publicly dispatchable* calculator literal —
any user machine-code program that does `RST 28H; DEFB $32; ...`
directly would still invoke it correctly, exactly as before. Reclaiming
this space later would remove that (obscure, rarely-used-directly, but
technically still available) capability, not just delete inert bytes —
worth weighing the same way the `LPRINT`/`COPY`/stream-3 questions were
earlier in this project, if it's ever actually reclaimed.

**Not reclaimed now** — logged here so the 15 bytes at `$36A0`-`$36AE`
are a known, already-verified option if more ROM space is needed later,
without having to redo this investigation from scratch.

### Files

- `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_LN_EXP_patched.asm` — superseded by the Patch 12 deliverable below (kept for history).
- `Decisions.md` — this file.

---

## Patch 12 — `CIRCLE`: Midpoint Circle Algorithm

**Date:** 2026-07-19
**File:** `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_LN_EXP_CIRCLE_patched.asm` (built on top of Patch 11)
**Status:** Verified — assembled clean, full diff explained, run in the
emulator against the real deliverable through its actual entry point.
New territory for this project: screen graphics, not calculator math.

### Why, and what's genuinely different about this one

The original `CIRCLE` draws by approximating the circle as a polygon of
straight chords, each one computed via full floating-point trigonometry
through the calculator — its own comments say a single large circle
checks free memory (and therefore runs calculator code) over 1300
times, specifically because it's slow enough to need BREAK-key
interruption checks. Replaced with the Midpoint Circle Algorithm
(Bresenham's circle): pure integer arithmetic, no calculator involved
at all beyond the initial float-to-integer conversion of the three
input parameters.

Two deliberate behavior changes, confirmed with the user before
building anything:
1. **Not pixel-identical to the original.** Fundamentally different
   algorithm (integer Bresenham vs. rotated floating-point chords) —
   correct, clean circles, just not a bit-for-bit match to the old
   output.
2. **Out-of-bounds points are silently skipped, not errored.** The
   original raises "Integer out of range" partway through drawing when
   part of a circle goes off-screen; this one just doesn't plot the
   points that fall outside `[0,255]x[0,175]` and continues normally.
   No BREAK-key interruption checks either (deliberately omitted —
   the routine is fast enough, and this was an explicit simplification
   request).

### Finding the exact boundary with `DRAW` before touching anything

`CIRCLE` and `DRAW` share real infrastructure: `CD-PRMS1` (parameter
setup) and the arc/line-drawing loop `DRAW-LINE`/`DRW-STEPS`, confirmed
by finding `CD-PRMS1` called from both commands directly in the ROM
source. Traced `CIRCLE`'s own code precisely before designing anything:
it begins at `$2320` (parsing, shared with `DRAW`'s own x,y stacking),
and its **unique** code — after `abs`/`re-stack`/`end-calc` confirm `r`
as a positive float — runs from `$2331` to a `JP L2420` at `$237F`
(forwarding into the shared arc-drawing loop), landing exactly at
`$2382` where `DRAW`'s own entry point begins, with zero gap and zero
overlap. This is the *only* span replaced; `CD-PRMS1`, `DRAW-LINE`, and
everything else stays completely untouched, confirmed after the fact by
diffing that entire region against baseline and finding every single
differing byte already accounted for by two separate, earlier,
documented patches (`ADDR`'s `S-LOOP-1` hook, `RND`'s `S-RND`
trampoline) — zero new or unexplained differences.

Confirmed `$2331` is reached as fresh Z80 code (the immediately
preceding instruction is `end-calc`), so a plain `JP` trampoline
suffices, same as `SQR`/`COS`/`TAN`/`ATN`'s trampolines.

### Design

- **`FP-TO-BC`** (`$2DA2`, unmodified) converts each of `x`, `y`, `r`
  from float to a signed 16-bit integer, rounding to nearest (matching
  general BASIC coordinate rounding). Centre coordinates are
  deliberately *not* range-clamped here — the centre can legitimately
  sit off-screen with part of a large circle still visible; bounds are
  checked per-pixel instead, at plot time.
- **Standard Bresenham/midpoint circle recurrence**, working in signed
  16-bit throughout (`x`, `y`, the decision variable, all comfortably
  within range for any practical radius).
- **8-way point symmetry, with explicit duplicate-avoidance conditions**
  derived and verified in Python *before* writing any assembly: always
  plot `(cx+x,cy+y)`; plot the mirrored/swapped variants only when the
  underlying coordinate that would otherwise repeat is non-zero, and
  skip the whole swapped set entirely when `x=y` (where it would
  exactly duplicate the primary set). Verified exhaustively against a
  naive-compute-all-8-then-deduplicate reference across radii 0-59:
  zero duplicates, zero missed points, every time — this matters
  specifically for `OVER 1`, where plotting the same pixel twice in one
  circle would toggle it back off.
- **`CHECK_AND_PLOT`**: given a computed `(px,py)`, checks both are in
  `[0,255]x[0,175]` (a signed 16-bit value is in range exactly when its
  high byte is zero, plus an explicit `<176` check for `y`) and, if so,
  tail-calls `PLOT-SUB` (`$22E5`, unmodified) — preserving `OVER`/
  `INVERSE`/attribute handling exactly, since the original routine does
  all of that work itself.
- 10 bytes of the `PRBUFF`-owned scratch RAM (`$5B0C`-`$5B15`, right
  after `RND`'s own `$5B00`-`$5B0B`) hold the six working variables
  (`CIRCLE_CX`, `CIRCLE_CY`, `CIRCLE_X`, `CIRCLE_Y`, `CIRCLE_ERR`, plus
  padding) — the same ownership established in Patch 7/8.

### Verification

This being new territory (screen graphics, not calculator arithmetic),
verification needed more than the usual reference-value comparison, and
caught three real bugs — all in the *test harness*, not the routine,
each traced to a specific cause before moving on rather than just
retried until something passed:

1. First attempt loaded the standalone-assembled routine into RAM at a
   different address than it was assembled for (`ORG $8000`, loaded at
   `$9C00`) — its own internal `CALL`/`JP` instructions still pointed at
   `$8000`, silently jumping into whatever was there instead (zero
   pixels plotted, no crash). Fixed by loading at the address it was
   actually assembled for.
2. A hand-derived Python reimplementation of `PIXEL-ADD`'s bit-twiddling
   (rotate/mask sequence, converting `(x,y)` to a screen address and
   bit) had its own bug, giving wrong expected addresses even though
   the *right number* of pixels were being plotted (a strong clue it was
   the checker, not the routine — confirmed by calling the real ROM's
   own `PIXEL-ADD` directly in the emulator instead of re-deriving it by
   hand, which then matched perfectly).
3. `PLOT-SUB` reads `P_FLAG` via `IY`-relative addressing (`IY+$57`),
   not a fixed absolute address — with `IY=$5C3A`, that's `$5C91`, not
   the `$5C57` the test initially poked, so the `OVER 1` test was
   silently testing `OVER 0` instead. Fixed the address, and `OVER 1`
   drawn twice then correctly returned the screen to exactly blank.

**Final results, against the real, fully-assembled deliverable ROM,
through its actual trampoline entry point at `$2331`** (not a
standalone copy): 7 test circles (centres and radii spanning `(0,0)` to
near the screen edges, radii from `0.4` — the "just a dot" case — up to
`87`, the original's own documented max for a "full" circle) compared
pixel-for-pixel against a Python reference implementation of the exact
same algorithm, using the real ROM's `PIXEL-ADD` as ground truth: **0
missing pixels, exact total-bit-count match, every case.** `OVER 1`
drawn twice returns the screen to exactly blank (confirming the
duplicate-avoidance logic holds up dynamically, not just in the
abstract Python check). A circle entirely off-screen plots nothing. A
circle straddling the edge clips to exactly the expected visible subset.
Full `SQR`/`SIN`/`COS`/`TAN`/`ATN` regression suite, the Patch 7
`PRBUFF` poison test, `RND`, `ASN`/`ACS`, `LN`, and `EXP` all re-run
against this build and unaffected. `DRAW` confirmed completely
untouched (see above).

No timing comparison run — the two algorithms are different enough
(integer arithmetic vs. floating-point trig through the calculator,
1300+ memory checks in the original for a large circle) that the
speedup is self-evident from the design rather than needing a specific
number, and a fair like-for-like comparison would need matching visual
output first, which isn't the goal here.

### Post-delivery revision: 422 bytes was excessive, refactored to 328

Flagged immediately after delivery, correctly: the first version unrolled
all 8 symmetric points into separate inline blocks -- each computing
`(cx+-x, cy+-y)` or `(cx+-y, cy+-x)` and calling `CHECK_AND_PLOT` -- which
is almost pure repetition. With 618 bytes free beforehand and 196 left
after, that repetition was eating real budget for no real benefit.

**Refactored into `PLOT4`**, a single shared subroutine that plots the
4 sign-combinations of a given `(a,b)` pair (reading them from two new
scratch RAM slots, `CIRCLE_A`/`CIRCLE_B`), with the exact same verified
zero-avoidance conditions as before. The main loop calls it twice --
once with `(a,b)=(x,y)`, once with `(a,b)=(y,x)`, the second call
skipped entirely when `x=y` (the same diagonal-duplicate case the
original point-by-point conditions guarded against). Two calls to one
125-ish-byte subroutine instead of eight ~25-byte unrolled blocks.

**Result: 328 bytes, down from 422 -- a 94-byte (22%) reduction.** Free
ROM space recovers to 290 bytes (was 196). The two extra scratch bytes
this needs (`CIRCLE_A`/`CIRCLE_B`) bring `CIRCLE`'s total RAM footprint
to 14 bytes, still a small fraction of the 256 owned.

**Reverified from scratch, not assumed equivalent from the design
alone**: standalone assembly, then the exact same test suite as the
first version -- the same 7 pixel-perfect circles (including the small-
radius edge cases) against the real ROM's own `PIXEL-ADD` as ground
truth, and the `OVER 1` draw-twice-returns-to-blank check -- all
identical results. Full regression suite (everything through `EXP`)
re-run against this build. `DRAW` reconfirmed completely untouched
(same trace-every-byte-difference method as the first version).

### Changes memory map

**(Final, cumulative with Patches 1-11.)**

| Address range        | Contents                                    | Size (bytes) |
|-----------------------|----------------------------------------------|-------------:|
| `$2331`–`$2381`      | `CIRCLE` trampoline (`JP NEW_CIRCLE_CORE`) + zero padding | 81 (unchanged width) |
| `$386E`–`$38A6`      | `NEW_SQR`              | 57 |
| `$38A7`–`$38CA`      | `NEW_SIN_CORE`              | 36 |
| `$38CB`–`$38E8`      | `NEW_COS_CORE` | 30 |
| `$38E9`–`$392B`      | `NEW_TAN_CORE` | 67 |
| `$392C`–`$3968`      | `ADDR_DISPATCH`/`S-ADDR` | 61 |
| `$3969`–`$399B`      | `NEW_ATN_CORE` | 51 |
| `$399C`              | `SAFE_ZERO_BYTE` | 1 |
| `$399D`–`$39A8`      | `CL_PRINTER_FIX` | 12 |
| `$39A9`–`$39D5`      | `NEW_RND_BYTE` | 45 |
| `$39D6`–`$39E8`      | `RND_BOOT_SEED` | 19 |
| `$39E9`–`$39F0`      | `RND_DEFAULT_TABLE` | 8 |
| `$39F1`–`$3A19`      | `RND_RESEED` | 41 |
| `$3A1A`–`$3A36`      | `NEW_S_RND` | 29 |
| `$3A37`–`$3A3D`      | `RND_BOOT_STUB` | 7 |
| `$3A3E`–`$3A61`      | `NEW_LN_TAIL` | 36 |
| `$3A62`–`$3A95`      | `NEW_EXP_CORE` | 52 |
| `$3A96`–`$3BDD`      | `CHECK_AND_PLOT`/`PLOT4`/`NEW_CIRCLE_CORE` (Midpoint Circle Algorithm, refactored -- see the note below) | 328 |
| `$3BDE`–`$3CFF`      | **Free block of size 290 bytes at $3BDE** (was 1170 bytes at `$386E` before any patches) | 290 |
| `$3D00`–...          | Character bitmap table — untouched | — |
| `$5B00`-`$5B0B`      | Real RAM, `RND` state | 12 |
| `$5B0C`-`$5B19`      | Real RAM, `CIRCLE` working variables (14 bytes -- 6 original + `CIRCLE_A`/`CIRCLE_B`, added by the refactor below) | 14 |
| `$5B1A`-`$5BFF`      | Real RAM, still unclaimed | 230 |

### Files

- `Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_LN_EXP_CIRCLE2_patched.asm` — the patched ROM source (current deliverable).
- `Decisions.md` — this file.
