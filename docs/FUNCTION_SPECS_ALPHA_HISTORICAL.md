# FUNCTION_SPECS.md — Checkpoint Alpha ("Free space isn't free")

Detailed technical reference for every routine replaced or added in this
project, as of Checkpoint Alpha. For narrative decisions, bugs, and
rationale, see `Decisions.md`. This document is the structural
reference: exact addresses, byte sizes, labels, direct references, and
padding, for every patch.

All addresses are as assembled in the current deliverable,
`Spectrum48_SQR_SIN_COS_TAN_ADDR_ATN_PRBUFF_RND_LN_EXP_CIRCLE2_patched.asm`.
Verified against `baseline.bin` (unmodified Spectrum 48K ROM) throughout
— every byte difference between baseline and this deliverable is
accounted for by exactly one of the entries below.

---

## Patch 1 — `SQR` (Newton-Raphson)

**Original:** `sqr` (`$384A`, 7 bytes) — `duplicate; not; jump-true LAST;
stk-half; end-calc`, i.e. `x^0.5` via `to-power` (shared with the `^`
operator; NOT touched or reclaimed — `to-power` remains fully alive).

**Replacement entry:** `L384A` retargeted. Since 7 bytes doesn't fit a
`JP` (3 bytes) is not the issue — it's that the *original* 7-byte
routine must become a trampoline to free space, because Newton-Raphson
needs far more than 7 bytes.

- **Trampoline:** `$384A`, `JP NEW_SQR`, zero-padded to fill the
  original slot width up to `$3850` inclusive (7 bytes total, matching
  the original routine's own width — verified the *next* routine,
  `sgn`, is safeguarded via `ORG` immediately after).
- **Core routine:** `NEW_SQR`, at `$386E`, **57 bytes**.
  - Entry contract: `x` on the FP calculator stack (may be short-integer
    or full-float format — routine begins with `re-stack` to normalize).
  - Algorithm: one iteration of Newton-Raphson (`y_new = 0.5*(y +
    x/y)`), seeded from a bit-manipulated initial guess derived directly
    from `x`'s own exponent/mantissa bytes (halving the exponent
    approximates `sqrt`).
  - Direct references: none to other ROM routines — pure calculator
    literals plus raw exponent-byte manipulation of the input's own FP
    representation in memory.
  - Error handling: negative input handling preserved via the
    calculator's own `greater-0`/`jump-true` idiom, matching original
    semantics (`sqr` of a negative number raises the standard error).
- **Padding:** none beyond the original 7-byte slot's own width (0
  bytes of filler needed there; slot exactly fits `JP` + 4 bytes
  padding).

**Speed:** 5×+ instruction-count reduction in the emulator (best case
of any patch in this project).

---

## Patch 2 — `SIN`/`COS` shared `C-ENT` + `SIN` core

**Original:** `sin` and `cos` share range-reduction (`get-argt`) and a
common re-entry point, `C-ENT` (`$37B7`), reached both via fresh
Z80 entry (from `sin`'s own dispatch) and via a **mid-calculator-literal
jump** from within `get-argt`'s own logic (a `jump`/`jump-true`
literal, not a Z80 `JP`) — this is why its trampoline needs `end-calc`
first, unlike the simpler cases.

- **Trampoline:** `$37B7`–`$37D9`, **35 bytes**. First byte is
  `end-calc` (`$38`), then `JP NEW_SIN_CORE`, then zero padding up to
  `$37D9` inclusive; `$37DA` (`tan`'s own original entry, later also
  replaced by Patch 4) safeguarded by its own `ORG`.
- **Core routine:** `NEW_SIN_CORE`, at `$38A7`, **36 bytes**.
  - Entry contract: reached via `get-argt`'s own reduction, which
    leaves a reduced angle (already folded into `[-π/2,π/2]`-equivalent
    range via quadrant logic) on the calculator stack.
  - Algorithm: Bhaskara I's approximation, `sin(x) ≈ 16x(π−x) /
    (5π²−4x(π−x))`, adapted to the calculator's own pre-scaled `Y`
    variable from `get-argt` (no `π` multiplication needed — cancels
    algebraically in the derivation).
  - Direct references: none beyond the calculator's own literal
    dispatch; consumes `get-argt`'s output directly, no shared mem-slot
    usage beyond what `get-argt` itself already established.
- **`get-argt` itself: untouched, its own address unchanged.** Verified
  by grepping every reference to it in the ROM source — still called
  identically by `NEW_SIN_CORE`, `NEW_COS_CORE`, and `NEW_TAN_CORE`.

**Speed:** 2.38×–2.57× instruction-count reduction.

---

## Patch 3 — independent `COS` (Bhaskara I, sign-corrected)

**Original:** `cos`'s own entry point, `$37AA`, computed `cos` by
transforming into a `sin` call (via a phase shift) — meaning the
*original* `cos` had no independent formula at all, just reused `sin`'s
machinery with an offset.

- **Trampoline:** `$37AA`–`$37B4`, **11 bytes** (unchanged width).
  `JP NEW_COS_CORE`, zero-padded. Confirmed via grep that `$37AA` is
  *only* reached via table dispatch (never mid-calculator-jumped-to
  from elsewhere), so a plain `JP` suffices — no `end-calc`-first trick
  needed here, unlike `C-ENT`.
- **Core routine:** `NEW_COS_CORE`, at `$38CB`, **30 bytes**.
  - Entry contract: same `get-argt`-reduced `Y` variable as `NEW_SIN_CORE`.
  - Algorithm: derived independently via `cos(x) = 4(1−Y²)/(4+Y²)` (the
    `π/2−x` substitution into Bhaskara's own sine formula, with `π`
    cancelling algebraically). Requires an explicit **sign correction**
    using `get-argt`'s own quadrant flag (`mem-0`, a side effect
    `get-argt` already produces) — `get-argt`'s folding is sine-shaped,
    so the raw formula gives the correct *magnitude* but wrong *sign* in
    quadrants II/III.
  - Direct references: reads `mem-0` (quadrant flag, written by
    `get-argt` — a pre-existing side effect, not something this routine
    sets up itself) and uses `mem-1` as scratch. Confirmed via grep
    that no other surviving ROM routine reads `mem-0`/`mem-1` in a way
    that would conflict.
  - `stk-data` constant used: `4.0` (header `$33`, data `$00`).

**Speed:** 2.38×–2.57× instruction-count reduction (paired with `SIN`).

---

## Patch 4 — independent `TAN` (Padé [3/2])

**Original:** `tan`'s own entry point, `$37DA`, computed `tan` as
`sin/cos` — i.e. it called *both* underlying trig computations and then
divided, meaning the original `tan` was strictly more expensive than
either `sin` or `cos` alone.

- **Trampoline:** `$37DA`–`$37E1`, **8 bytes** (unchanged width — note
  this is smaller than `cos`'s 11-byte slot; the original `tan` body
  really was only 8 bytes, a sizing detail caught and corrected before
  shipping during a standalone-assembly size check). `JP NEW_TAN_CORE`,
  zero-padded. Confirmed via grep: `$37DA` reached only via table
  dispatch, plain `JP` sufficient.
- **Core routine:** `NEW_TAN_CORE`, at `$38E9`, **67 bytes** (largest of
  the four trig cores, reflecting the extra branch logic below).
  - Entry contract: `get-argt`-reduced `Y` variable, valid up to
    `|x|=π/4` directly; beyond that, uses the reciprocal identity
    `tan(x) = sgn(x)/tan(π/2−|x|)`.
  - Algorithm: Padé [3/2] rational approximant,
    `tan(x) ≈ x(15−x²)/(15−6x²)`.
  - **Sign-correction logic distinct from `COS`'s**: derived
    algebraically that for quadrants II/III (where `get-argt`'s `mem-0`
    flag is set), `Y_tan = −Y_sin`; for quadrants I/IV, `Y_tan = Y_sin`
    unchanged — the opposite correction shape from `COS`'s, since
    `tan`'s period differs from `sin`'s folding.
  - Direct references: `mem-0` (quadrant flag from `get-argt`), `mem-1`
    (transient, Padé denominator), `mem-2` (`is_recip` flag, tracks
    whether the reciprocal branch was taken).
  - `stk-data` constants used: `6.0` (header `$33`, data `$40`), `15.0`
    (header `$34`, data `$70`); `stk-pi/2` via the existing literal
    `$A3` (not a custom constant — confirmed against ROM source).
  - **Known, deliberate non-fix:** `tan(π/2)` returns a large finite
    number rather than a clean error, because `get-argt`'s own rounding
    doesn't land exactly on the boundary for this specific bit pattern.
    Verified on real hardware that the *original*, unpatched ROM has the
    identical behavior for any `x` near (but not exactly at) `π/2` — this
    is not a regression, and an epsilon-threshold "fix" was deliberately
    not added, since it would make the new behavior diverge *further*
    from the original's own (already inexact) boundary behavior.

**Speed:** 2.76×–3.50× instruction-count reduction — the best of the
four trig patches, since the original `tan` paid for a full `sin`+`cos`
computation and this one doesn't.

---

## Patch 5 — `ADDR(var)` (new BASIC function)

**Not a replacement — new capability.** No original to compare against.

**Token:** reuses `LPRINT`'s existing token byte, `$E0` (E-UNSHIFT
range, so it's typeable via Extended Mode anywhere in an expression,
unlike the `K`-mode-only `COPY` token originally considered and
rejected — `COPY` can only appear at statement-start, which would have
made `ADDR` unusable mid-expression).

**Hook:** `S-LOOP-1` (`$24FF`)'s own fallback jump, at `$2507`–`$2509`,
**3 bytes** (unchanged width) — retargeted from `JP NC,S-ALPHNUM`
(`$2684`) to `JP NC,ADDR_DISPATCH`. `ADDR_DISPATCH` itself checks for
the `$E0` token and falls through to the *original* `S-ALPHNUM` for
every other case, so ordinary variable-name scanning is unaffected.

**Core routine:** `ADDR_DISPATCH`/`S_ADDR`, at `$392C`, **61 bytes**.

- Parses `(`, calls `LOOK_VARS` (`$28B2`, unmodified) to resolve the
  variable, optionally `STK_VAR` (`$2996`, unmodified) for array/
  subscript syntax, then `+1` corrects `LOOK_VARS`'/`STK_VAR`'s own
  "start of name/data, not value" convention to the actual value
  address.
- **Two real bugs found and fixed post-delivery, both logged in
  `Decisions.md` in full — summarized here for the technical record:**
  1. **Syntax-check safety.** During the automatic syntax-check pass
     that runs the instant a typed line's ENTER is pressed (before the
     line is even stored), `LOOK_VARS`/`STK_VAR` don't perform a real
     search and `HL` is not meaningfully set — confirmed via
     `LOOK_VARS`'s own source comment, *"if checking syntax the letter
     is not returned."* Fixed by checking `SYNTAX_Z` (`$2530`) and
     using a placeholder value instead of trusting `HL` when only
     checking syntax.
  2. **String/numeric check bug.** The original design's `LD
     A,($5C3B); CP $C0` check (testing FLAGS bits 7 and 6 together)
     incorrectly required bit 7 (runtime) to also be set, making the
     numeric path *structurally unreachable* during any syntax-check,
     independent of fix #1. Replaced with `BIT 6,(IY+$01)` (testing bit
     6 alone — the bit `LOOK_VARS` actually sets/clears for numeric-
     vs-string), which is both correct and one byte shorter than what
     it replaced.
- Direct references: `LOOK_VARS` (`$28B2`), `STK_VAR` (`$2996`),
  `SYNTAX_Z` (`$2530`), `STACK_A`/error report routines (`REPORT_C`
  `$1C8A`, `REPORT_2` `$1C2E`), `S_NUMERIC` (`$26C3`, the shared
  scanning-continuation tail every function-token handler rejoins).
- **Padding:** none — 61 bytes is the routine's exact, final size after
  both fixes (no unused trailing padding).

---

## Patch 6 — independent `ATN` (continued fraction)

**Original:** `atn`'s own preprocessing (`|x|≥1` reciprocal-transform,
sign handling via `less-0`/`stk-pi/2`) is **kept, untouched** — only
the final polynomial evaluation (`CASES`, the original's 12-coefficient
Chebyshev series) is replaced, the same "keep the reduction, swap the
core formula" pattern as `COS`/`TAN`.

- **Trampoline:** `$37FA`–`$3832` (`CASES`'s original span), **57
  bytes**. First byte `end-calc` (reached via both calculator-level
  `jump`/`jump-true` from the `|x|≥1` branch, and sequential
  fall-through from a fresh `RST 28H` in the `|x|<1` branch — both
  already-open calculator contexts, hence `end-calc`-first, same
  reasoning as `C-ENT`). `$3833` (`asn`'s own entry) safeguarded.
- **Core routine:** `NEW_ATN_CORE`, at `$3969`, **51 bytes**.
  - Entry contract: `[t, offset]` on the calculator stack — `t` is
    either `−1/x` (reciprocal-transformed, `|x|≥1` case) or `x` directly
    (`|x|<1` case); `offset` is `0`, `+π/2`, or `−π/2` accordingly, from
    the untouched preprocessing above. **The original `CASES`'s own
    first instruction was `exchange`** (swapping this pair into the
    order the polynomial needed) — a real bug on the first pass of this
    patch forgot to replicate that exchange when replacing the whole
    block, producing plausible-looking but silently wrong output
    (`atn(1)` came back as `0.008` instead of `≈0.785`) until caught by
    actually running the routine, not just checking its bytes. Fixed by
    restoring `exchange` as this routine's own first instruction.
  - Algorithm: depth-4 continued fraction, `atan(t) ≈ t / (1 +
    t²/(3 + 4t²/(5 + 9t²/(7 + 16t²/9))))`.
  - Uses `mem-0` to hold `t²`, re-fetched at each of the four levels
    rather than kept on the calculator stack (which already carries
    `offset`, `t`, and the running numerator/denominator
    simultaneously).
  - `stk-data` constants: `16.0` (`$35`/`$00`), `9.0` (`$34`/`$10`),
    `7.0` (`$33`/`$60`), `5.0` (`$33`/`$20`), `3.0` (`$32`/`$40`).
  - Direct references: none beyond calculator literals and `mem-0`; no
    collision risk since `asn`/`acs` (which call `atn` internally, see
    Patch 9) don't rely on `mem-0` surviving across the `atn` call.

**Speed:** matches the accuracy/speed tier described in `Decisions.md`
(continued-fraction family, same tier as `TAN`'s Padé approximant).

---

## Patch 7 — `PRBUFF` neutering (enabling patch for `RND`)

**Not a function replacement — infrastructure change.** Neuters the
real printer buffer (`$5B00`-`$5BFF`, 256 bytes of RAM) so it becomes
genuinely, provably free RAM, without touching any of `COPY`/`LPRINT`/
`LLIST`/`PRINT #3`/`LIST #3`'s own statement-handling logic (which
remains fully intact and still executes — it just now touches a
guaranteed-zero byte instead of real RAM, producing no visible output
rather than erroring).

**Exhaustively confirmed** (grep for every literal `$5B00` reference in
the entire ROM source) that exactly three sites hardcode the real
buffer's address, plus one read-loop site that needed separate handling:

| Site | Address | Original | Patched | Width |
|---|---|---|---|---|
| `CL-SET` (character-position calc) | `$0DD9`-`$0DDB` | `LD HL,$5B00` | `LD HL,SAFE_ZERO_BYTE` | 3 (operand only changes, 2 bytes) |
| `CL-SET-2`'s tail | `$0DFA`-`$0DFD` | `JP L0ADC` (`PO-STORE`) | `JP CL_PRINTER_FIX` | 3 (operand only changes, 2 bytes) |
| `COPY-BUFF` (flush-to-printer) | `$0ECE`-`$0ED0` | `LD HL,$5B00` | `LD HL,SAFE_ZERO_BYTE` | 3 (operand only changes, 2 bytes) |
| `COPY-L-3` (the actual read loop) | `$0F15` | `INC HL` | `NOP` | 1 (same size) |
| `CLEAR-PRB` (buffer zeroing) | `$0EDF`-`$0EE1` | `LD HL,$5B00` | `LD HL,SAFE_ZERO_BYTE` | 3 (operand only changes, 2 bytes) |

`PR_CC` (`$5C80`, the printer's "current position" system variable)
checked and ruled out as a fourth site needing its own fix: it never
hardcodes `$5B00` itself, it only ever stores whatever `CL-SET` last
computed — so retargeting `CL-SET`'s own base address covers it
automatically.

**New code, at `$399C`, 13 bytes total:**
- `SAFE_ZERO_BYTE` (`$399C`, 1 byte) — a single byte, permanently `$00`.
  Real Z80 writes to ROM addresses (below `16384`) are a documented
  no-op on real hardware (confirmed against Fuse during Patch 1's own
  verification), so this byte is *provably*, not just probably, always
  zero.
- `CL_PRINTER_FIX` (`$399D`-`$39A8`, 12 bytes) — re-tests the same
  "printer in use" `FLAGS` bit `CL-SET` itself already tested, and only
  when true, forces `HL` back to `SAFE_ZERO_BYTE` (discarding whatever
  `CL-SET`'s own column-offset addition computed — up to ±32 bytes off
  the base, since that addition is on a tail *shared* with normal
  screen positioning and can't simply be deleted) before continuing to
  the real `PO-STORE` unchanged. The screen-positioning case flows
  through untouched.

**Design note — single byte, not a range:** the first attempt used a
256-byte permanently-zero *block* (matching `COPY-BUFF`'s own read
range). Corrected on request to a single byte, with each of the three
pointer-walking/reading sites individually rewritten to stay pinned on
it (`COPY-L-3`'s own `INC HL`→`NOP`, and `CL_PRINTER_FIX`'s explicit
re-pin after `CL-SET-2`'s shared addition) rather than being given a
range to wander across. `CLEAR-PRB` alone needed no loop rewrite,
since it's write-only and ROM writes are already no-ops regardless of
destination address.

**Verification:** confirmed provably, not just observationally — "poisoned"
real `PRBUFF` RAM with a recognizable non-zero pattern (`$AA` throughout)
before running all three routines directly: all completed without error
and left the poison pattern completely undisturbed. Separately confirmed
`CL-SET`'s printer-path output is *exactly* `$399C` for multiple
different column values (not just "in a safe range"). Screen positioning
(the shared code this could most easily have broken) verified for both
upper- and lower-screen cases across a spread of line/column values,
byte-identical to baseline for the actual computation logic (only the
entry/exit operands differ).

## Patch 8 — `RND` (CMWC generator)

**Original:** `S-RND` (`$25F8`-`$2626`, 47 bytes) — a Lehmer/LCG
generator, `seed = (seed*75) mod 65537`, period `2^16-1`, reading/
writing the `SEED` system variable (`$5C76`, 2 bytes) — documented in
the wider community as producing visible diagonal-line correlation when
plotted as `(x,y)` pairs.

**Algorithm confirmed, not assumed:** Boriel BASIC's own `RND` is
Patrik Rak's CMWC (Complementary Multiply-With-Carry) generator —
confirmed directly via the `RandomStream.bas` library's own docs
(written by a `zxbasic` contributor as an alternative), which states
outright it's "the same random number generator that Boriel is
using... based pretty much wholly on Patrik Rak's stream random
generator." A CMWC generator, not the simpler xorshift variant also
circulating in the same community threads (the xorshift "passes most
Diehard tests"; this CMWC variant "passes all Diehard tests").

**Necessary adaptation:** the original published algorithm relies on
self-modifying code (patching its own instruction operand bytes for the
lag-table index and the carry value between calls) — this **cannot
work from ROM** (writes to ROM are a no-op, the same fact Patch 7
relies on). Redesigned to hold the index and carry as explicit RAM
bytes instead, verified in Python to produce bit-for-bit identical
output to the original self-modifying-code algorithm across 50,000
draws — a faithful translation, not a different algorithm.

**RAM state, in `PRBUFF`-owned space (Patch 7), `$5B00`-`$5B0B`, 12 bytes:**

| Address | Contents |
|---|---|
| `$5B00`-`$5B07` | 8-byte CMWC lag table (mutable) |
| `$5B08` | current table index (0-7) |
| `$5B09` | carry byte |
| `$5B0A`-`$5B0B` | scratch (seed lo/hi) used only during `RANDOMIZE` |

**New code, five routines:**

| Routine | Address | Size | Purpose |
|---|---|---|---|
| `NEW_RND_BYTE` | `$39A9` | 45 bytes | Core CMWC byte generator, one call = one pseudorandom byte in `A` |
| `RND_BOOT_SEED` | `$39D6` | 19 bytes | One-time table init from `RND_DEFAULT_TABLE` |
| `RND_DEFAULT_TABLE` | `$39E9` | 8 bytes | The well-chosen default seed values: `82,97,120,111,102,116,20,12` |
| `RND_RESEED` | `$39F1` | 41 bytes | Reseeds from a 16-bit value in `BC` (the `RANDOMIZE` parameter), mixing it into the 8 default entries rather than replacing them outright |
| `NEW_S_RND` | `$3A1A` | 29 bytes | The calculator-facing entry point |
| `RND_BOOT_STUB` | `$3A37` | 7 bytes | Runs once at `NEW`/cold boot; preserves the original (now-neutered) `CALL L0EDF` exactly, then also calls `RND_BOOT_SEED` |

**Hooks, all same-size operand/instruction changes:**

| Site | Address | Original | Patched | Width |
|---|---|---|---|---|
| `S-RND` | `$25F8`-`$2626` | 47-byte LCG routine | `JP NEW_S_RND` + zero padding | 47 (unchanged) |
| `RANDOMIZE`'s tail | `$1E5A`-`$1E5E` | `LD ($5C76),BC; RET` | `JP RND_RESEED` + 2 padding | 5 (unchanged) |
| Boot-time init | `$128B`-`$128D` | `CALL L0EDF` | `CALL RND_BOOT_STUB` | 3 (unchanged) |

**`NEW_S_RND`'s design, and the three bugs found fixing it (all logged
in full in `Decisions.md`, summarized here):**
- Builds a 16-bit value from two `NEW_RND_BYTE` calls, pushes it via
  `STACK-BC` (`$2D2B`, unmodified), then divides by `65536` through
  real calculator division rather than the original's "check exponent
  byte, subtract 16" trick.
- **Bug 1:** first draft jumped to `L2625` (the original `S-RND-END`)
  to rejoin the shared continuation — but that label sat inside the
  47-byte span that became the trampoline, so it no longer existed.
  Fixed by jumping directly to `L2630` (`S-PI-END`), which is all
  `L2625` ever did (`JR L2630`, nothing else).
- **Bug 2:** `NEW_RND_BYTE` uses `B` as internal scratch. Storing the
  first generated byte in `B` and calling `NEW_RND_BYTE` again silently
  destroyed it — every draw came back as `0.0`. Fixed via `PUSH
  AF`/`POP AF` across the second call.
- **Bug 3 (the real one):** `65536` cannot be encoded as a single
  `stk-data` constant — its exponent overflows the header byte's 6-bit
  field (`low6 = exp_byte - 0x50` must be `0`-`63`; `65536`'s exponent
  gives `65`). No earlier patch's constants happened to be large enough
  to hit this. Fixed by building `65536` as `256x256` instead (both
  safely encodable).
- Direct references: `STACK_BC` (`$2D2B`), `SYNTAX_Z` (`$2530`), `L2630`
  (`S-PI-END`).

**Verification:** `NEW_RND_BYTE` alone, 1000/1000 exact matches against
the Python reference model. `RND_RESEED` exact table matches across 5
specific seeds plus a 2000-random-seed distribution stress test (worst
case 253/256 distinct bytes in 2000 draws). `S-RND`, run through its
real entry point, 300 consecutive calls compared byte-for-byte against
the Python reference's byte-pair-to-float pipeline: 0 mismatches,
worst error 0.00.

---

## Patch 9 — `ASN`/`ACS`, `^` (`to-power`): free speedups, zero new code

**Not a patch — a verified finding.** Reading the original ROM source
(rather than assuming a rewrite was needed) showed `ASN` (`$3833`) is
already a calculator-literal program calling `sqr` (`$28`) and `atn`
(`$24`) as sub-operations, via the half-angle identity `asn(x) =
2*atn(x / (sqrt(1-x^2) + 1))` (chosen in the original ROM's own design
specifically to avoid the division-by-zero a naive `atn(x/sqrt(1-x^2))`
would hit at `x=+-1`). `ACS` (`$3843`) is `pi/2 - asn(x)`, calling `asn`
(`$22`) in turn. `to-power` similarly calls `ln` then jumps directly
into `L36C4` (`EXP`'s own entry point).

Since Patches 1, 6, 10, and 11 all work by redirecting the exact
dispatch addresses these routines already call into, `ASN`/`ACS`/`^`
inherited every relevant speedup automatically — zero new bytes,
zero new code, zero new risk.

**Verified, not assumed:** 202 test values for `ASN`/`ACS` through the
real patched ROM, max absolute error `3.75e-4` (consistent with the
`SQR`+`ATN` composition's own accuracy tier) — confirmed directly:
`acs(1)` itself comes out as `0.000375`, not exactly `0`, which is this
same expected error propagating through subtraction from an exact `pi/2`
constant, not a bug (a test that assumed exact-zero there with too
tight a tolerance was corrected separately). Directly measured T-states:
`ASN` ~1.64x faster, `^` 1.54x-1.82x faster, already, with nothing touched.

---

## Patch 10 — 3-term atanh-series `LN`

**Original:** a genuine two-tier reduction (exponent split, `x=M*2^k`,
then a threshold check that doubles `M` via directly incrementing its
raw exponent byte if `M<=0.8`, tightening the range before the final
evaluation) feeding a 12-coefficient Chebyshev polynomial (`series-0C`).
More involved than this project's own earlier research had assumed —
kept the entire two-tier reduction exactly as written, replaced only
the final polynomial evaluation, the same pattern used throughout this
project.

**Trampoline:** inside `GRE.8` (the original's own post-reduction
label), at `$374A`-`$3782`, **57 bytes** — not at `GRE.8`'s own start,
but right after it finishes computing `u = M-1` (a `stk-half; subtract;
stk-half; subtract` sequence, left completely unmodified). Reached both
via a calculator-level `jump-true` (`M>0.8` case) and sequential
fall-through from a fresh `RST 28H` (`M<=0.8` case) — both already-open
calculator contexts, hence `end-calc`-first. `$3783` (`get-argt`,
shared with the trig functions) safeguarded.

**Core routine:** `NEW_LN_TAIL`, at `$3A3E`, **36 bytes**.
- Entry contract: `[k*ln2, u]` on the calculator stack, `u=M-1` from the
  untouched reduction above.
- Algorithm: `ln(M) ~= 2y(1 + y^2/3 + y^4/5)`, `y = u/(u+2)` — verified
  in Python against the *actual* range this reduction produces
  (`|u|<=0.6`, tighter than a naive assumption would suggest): max abs
  error **1.04e-5**, the best accuracy of any patch in this project.
- `stk-data` constants: `2.0` (`$32`/`$00`), `0.2` (`$EE`/`4C CC CC
  CD`), `1/3` (`$EF`/`2A AA AA AB`).
- Direct references: none beyond calculator literals; exit matches the
  original's own final "addition" exactly (`k*ln2 + ln(M)`).

**Verification:** 1004 test values (0.01 to 9.99 in 0.01 steps, plus
extremes), worst abs error 1.04e-5, matching the Python prediction
exactly. `ln(1) = 0` exactly. Negative/zero correctly raise Report A
(error code 9), confirming the untouched original validation logic.
One test-script bug caught along the way: the first test run used
literal `$21` (actually `TAN`'s literal) instead of `$25` (`LN`'s real
one) — fixed the test, not the ROM, confirmed against the calculator's
own dispatch table. Speed: 1.86x-2.17x faster.

---

## Patch 11 — Pade[2/2] `EXP`

**Original:** `k = floor(x/ln2)`, giving a one-sided fractional part
`f in [0,1)` -- i.e. a reduced exponent `r=f*ln2 in [0,ln2)` -- feeding
an 8-coefficient Chebyshev polynomial (`series-08`), then a raw
exponent-byte addition trick (`k` converted to a direct add on the
result's own exponent byte, i.e. `2^k` multiplication essentially free)
plus overflow/sign error handling (Report 6, "Number too big").

**A caught mistake worth flagging on its own:** the Pade[2/2] formula's
`~7e-6` accuracy claim, quoted earlier in this project, was verified
against a CENTERED range (`|r|<=ln2/2`). Checked against the range the
original's actual `floor`-based reduction would give: 2.27e-4, over 30x
worse. Fixed by changing the reduction itself to `k = round(x/ln2)`
(via `floor(x/ln2 + 0.5)`), recovering the centered range and the real
6.99e-6 figure. The lesson, stated plainly: a formula's accuracy claim
is only as good as the range it was actually verified against, not the
range a "just swap the polynomial" instinct assumes.

**Trampoline:** `$36C4`-`$36F8`, **53 bytes** — this is only the
calculator-portion of the original (`$36C4` through immediately before
`CALL L2DD5`); `$36F9` onward (`FP-TO-A` plus all of the original's own
overflow/sign error handling) is **reused unchanged**, not replaced.
Confirmed `$36C4` is always fresh Z80 entry (table dispatch, and once
via a plain `JP` from `to-power`'s own final step) so a plain `JP`
trampoline suffices.

**Core routine:** `NEW_EXP_CORE`, at `$3A62`, **52 bytes**.
- Algorithm: `exp(r) ~= (1+r/2+r^2/12)/(1-r/2+r^2/12)`.
- Ends its own calculator program with the exact same stack convention
  the original left (`[pade_result, k]`) and jumps straight into
  `$36F9`, reusing the original's own error handling rather than
  reimplementing it.
- `stk-data` constants: `1/ln2` (`$F1`/`38 AA 3B 29` — matches the
  ROM's own existing constant exactly, confirmed by decode), `ln2`
  (`$F0`/`31 72 17 F8`), `1/12` (`$ED`/`2A AA AA AB`).
- Direct references: `$36F9` (the reused original tail, ending in the
  original's own `RET`).

**Verification:** 407 test values, worst relative error 6.854e-06
(matching the 6.99e-6 prediction). `exp(0)=1` exactly. `exp(100)`
correctly raises through the *reused* original error path (confirming
new calculator code can correctly jump into old raw-Z80 code and have
its error handling behave identically). Speed: 1.43x faster -- more
modest than other patches, since the original `EXP` already used the
same cheap exponent-adjustment trick this one also relies on; the win
here is purely a shorter calculator program.

---

## Patch 12 — `CIRCLE` (Midpoint Circle Algorithm)

**New territory for this project: screen graphics, not calculator
arithmetic.**

**Original:** approximates the circle as a polygon of straight chords,
each computed via full floating-point trigonometry through the
calculator -- the original's own comments state a large circle checks
free memory (and therefore runs calculator code) over 1300 times,
specifically because it needs BREAK-key interruption checks at that
frequency.

**Exact boundary with `DRAW`, traced before touching anything:**
`CIRCLE` and `DRAW` share real infrastructure -- `CD-PRMS1` (parameter
setup, confirmed called from both commands directly in source) and the
arc/line-drawing loop `DRAW-LINE`/`DRW-STEPS`. `CIRCLE`'s own unique
code -- after `abs`/`re-stack`/`end-calc` confirm `r` as a positive
float -- runs from `$2331` to a `JP L2420` at `$237F` (forwarding into
the shared arc loop), landing exactly at `$2382` where `DRAW`'s own
entry point begins, zero gap, zero overlap. This is the only span
replaced.

**Trampoline:** `$2331`-`$2381`, **81 bytes**. `$2331` confirmed reached
as fresh Z80 code (immediately preceded by `end-calc`), plain `JP`
sufficient.

**Two deliberate behavior changes, confirmed with the user before
building anything:**
1. Not pixel-identical to the original (different algorithm).
2. Out-of-bounds points are silently skipped, not errored -- the
   original raises "Integer out of range" partway through drawing; this
   one just doesn't plot points outside `[0,255]x[0,175]` and continues.
   No BREAK-key interruption checks either (deliberately omitted).

**Core routines (after a post-delivery refactor -- see below), all in
one contiguous block at `$3A96`, 328 bytes total:**

| Routine | Purpose |
|---|---|
| `CHECK_AND_PLOT` | Given `(px,py)` in `HL`/`DE`, bounds-checks and tail-calls `PLOT-SUB` |
| `PLOT4` | Given `(a,b)` in scratch RAM, plots the 4 sign-combinations `(cx+-a,cy+-b)` with duplicate-avoidance |
| `NEW_CIRCLE_CORE` | Entry point: float-to-int conversion, radius<=0 special case, Bresenham main loop |

**RAM state, in `PRBUFF`-owned space, `$5B0C`-`$5B19`, 14 bytes:**
`CIRCLE_CX`, `CIRCLE_CY`, `CIRCLE_X`, `CIRCLE_Y`, `CIRCLE_ERR` (2 bytes
each, signed 16-bit), plus `CIRCLE_A`/`CIRCLE_B` (2 bytes each, added
by the `PLOT4` refactor).

**Design details:**
- `FP-TO-BC` (`$2DA2`, unmodified) converts `x`, `y`, `r` from float to
  signed 16-bit (round-to-nearest). Centre coordinates are deliberately
  NOT range-clamped here -- the centre can legitimately sit off-screen
  with part of a large circle still visible; bounds are checked
  per-pixel at plot time instead.
- Standard Bresenham/midpoint circle recurrence, signed 16-bit
  throughout.
- 8-way point symmetry via two calls to `PLOT4` (`(a,b)=(x,y)`, then
  `(a,b)=(y,x)` unless `x=y`), with explicit duplicate-avoidance
  conditions verified exhaustively in Python across radii 0-59 before
  writing any assembly -- zero duplicates, zero missed points, in every
  case (this matters specifically for `OVER 1`: plotting the same pixel
  twice in one circle would toggle it back off).
- `PLOT-SUB` (`$22E5`, unmodified) does the actual pixel setting,
  preserving `OVER`/`INVERSE`/attribute handling exactly.

**Post-delivery revision -- 422 bytes to 328 bytes:** the first version
unrolled all 8 symmetric points into 8 separate inline blocks (~25
bytes each), flagged as excessive given it drained free ROM space from
618 to 196 bytes. Refactored into the shared `PLOT4` subroutine
(described above), called twice instead of unrolled 8 times -- same
verified duplicate-avoidance logic, same behavior, 94 fewer bytes (22%
reduction). Reverified from scratch against the identical test suite
(not assumed equivalent from the design alone): same result every time.

**Three bugs caught during verification, all in the TEST HARNESS, not
the routine -- each traced to a specific cause, not just retried:**
1. Loaded a relocatable standalone-assembled routine at a different
   address than it was assembled for; its own internal `CALL`/`JP`
   instructions still pointed at the old address (zero pixels plotted,
   no crash -- a strong clue, since it wasn't a hard failure).
2. A hand-derived Python reimplementation of `PIXEL-ADD`'s bit-twiddling
   had its own bug -- caught because the RIGHT NUMBER of pixels were
   being plotted, just not at the expected addresses (fixed by calling
   the real ROM's own `PIXEL-ADD` as ground truth instead of
   re-deriving it by hand).
3. `PLOT-SUB` reads `P_FLAG` via `IY`-relative addressing (`IY+$57`),
   not a fixed absolute address -- with `IY=$5C3A` that's `$5C91`, not
   the address the test initially poked, so an `OVER 1` test was
   silently testing `OVER 0`.

**Final verification, against the real deliverable, through its actual
trampoline entry point:** 7 test circles (various centres/radii
including `r=1`, `r=0.4`, and centres near screen edges) matched
pixel-for-pixel against a Python reference using the real ROM's
`PIXEL-ADD` as ground truth -- 0 missing pixels, exact bit-count match,
every case. `OVER 1` drawn twice returns the screen to exactly blank.
Fully off-screen circles plot nothing; partially off-screen circles
clip to exactly the expected visible subset. `DRAW` confirmed
completely untouched -- every byte difference in the shared region
traced and found to already be accounted for by two separate, earlier,
documented patches (`ADDR`'s hook, `RND`'s trampoline), zero new or
unexplained differences.

---

## Confirmed dead code, not yet reclaimed

**`n-mod-m`** (`$36A0`-`$36AE`, 15 bytes, calculator literal `$32`) --
documented in the ROM's own source as "only used internally by the RND
function." Confirmed genuinely dead, not just per the comment: grepped
the entire ROM for every reference to its address (exactly two -- the
calculator's own dispatch table entry, and its own definition -- both
of which leave the routine itself untouched and still technically
dispatchable via `RST 28H; DEFB $32` by any user machine-code program,
which is why this is "dead" in the sense of no ROM-internal caller
remaining, not "unreachable in every sense"). Checked every new routine
in this project for accidental use of literal `$32`: none. Not
reclaimed -- logged as an already-verified option if more space is
needed later.

**Checked and ruled out as NOT dead:** original `SQR` was just `stk-half;
fall through into to-power` -- `to-power` remains fully alive (it's
what makes `^` work). `TAN`/`ATN`/`LN`/`EXP`'s own preprocessing/
reduction logic was kept in every case, so nothing upstream of any
trampoline was ever exclusively used by the part that got replaced.
`CIRCLE`'s only external calls (`CD-PRMS1`, `PLOT-SUB`) are both still
fully alive, shared with `DRAW` or used throughout the ROM generally.

---

## Consolidated ROM memory map (Checkpoint Alpha)

| Address range | Contents | Size (bytes) |
|---|---|---:|
| `$0DDA`-`$0DDB` | `CL-SET` operand, retargeted (Patch 7) | 2 |
| `$0DFC`-`$0DFD` | `CL-SET-2` operand, retargeted (Patch 7) | 2 |
| `$0ECF`-`$0ED0` | `COPY-BUFF` operand, retargeted (Patch 7) | 2 |
| `$0EE0`-`$0EE1` | `CLEAR-PRB` operand, retargeted (Patch 7) | 2 |
| `$0F15` | `COPY-L-3`: `INC HL` to `NOP` (Patch 7) | 1 |
| `$128C`-`$128D` | Boot-time init operand, retargeted (Patch 8) | 2 |
| `$1E5A`-`$1E5E` | `RANDOMIZE` tail, retargeted (Patch 8) | 5 |
| `$2331`-`$2381` | `CIRCLE` trampoline (Patch 12) | 81 |
| `$2508`-`$2509` | `S-LOOP-1` hook, retargeted to `ADDR_DISPATCH` (Patch 5) | 2 |
| `$25F8`-`$2626` | `S-RND` trampoline (Patch 8) | 47 |
| `$36C4`-`$36F8` | `EXP` calculator-portion trampoline (Patch 11) | 53 |
| `$374A`-`$3782` | `LN` tail trampoline (Patch 10) | 57 |
| `$37AA`-`$37B4` | `COS` trampoline (Patch 3) | 11 |
| `$37B7`-`$37D9` | `SIN`/`COS`-shared `C-ENT` trampoline (Patch 2) | 35 |
| `$37DA`-`$37E1` | `TAN` trampoline (Patch 4) | 8 |
| `$37FA`-`$3832` | `ATN` `CASES` trampoline (Patch 6) | 57 |
| `$386E`-`$38A6` | `NEW_SQR` (Patch 1) | 57 |
| `$38A7`-`$38CA` | `NEW_SIN_CORE` (Patch 2) | 36 |
| `$38CB`-`$38E8` | `NEW_COS_CORE` (Patch 3) | 30 |
| `$38E9`-`$392B` | `NEW_TAN_CORE` (Patch 4) | 67 |
| `$392C`-`$3968` | `ADDR_DISPATCH`/`S-ADDR` (Patch 5) | 61 |
| `$3969`-`$399B` | `NEW_ATN_CORE` (Patch 6) | 51 |
| `$399C` | `SAFE_ZERO_BYTE` (Patch 7) | 1 |
| `$399D`-`$39A8` | `CL_PRINTER_FIX` (Patch 7) | 12 |
| `$39A9`-`$39D5` | `NEW_RND_BYTE` (Patch 8) | 45 |
| `$39D6`-`$39E8` | `RND_BOOT_SEED` (Patch 8) | 19 |
| `$39E9`-`$39F0` | `RND_DEFAULT_TABLE` (Patch 8) | 8 |
| `$39F1`-`$3A19` | `RND_RESEED` (Patch 8) | 41 |
| `$3A1A`-`$3A36` | `NEW_S_RND` (Patch 8) | 29 |
| `$3A37`-`$3A3D` | `RND_BOOT_STUB` (Patch 8) | 7 |
| `$3A3E`-`$3A61` | `NEW_LN_TAIL` (Patch 10) | 36 |
| `$3A62`-`$3A95` | `NEW_EXP_CORE` (Patch 11) | 52 |
| `$3A96`-`$3BDD` | `CHECK_AND_PLOT`/`PLOT4`/`NEW_CIRCLE_CORE` (Patch 12, refactored) | 328 |
| `$3BDE`-`$3CFF` | **Free ROM: 290 bytes** | 290 |
| `$3D00`-... | Character bitmap table -- untouched | -- |

**RAM (`PRBUFF`-owned, `$5B00`-`$5BFF`, since Patch 7):**

| Address range | Contents | Size (bytes) |
|---|---|---:|
| `$5B00`-`$5B0B` | `RND` state (Patch 8) | 12 |
| `$5B0C`-`$5B19` | `CIRCLE` working variables (Patch 12) | 14 |
| `$5B1A`-`$5BFF` | **Free RAM: 230 bytes** | 230 |

**Total ROM free space budget consumed so far:** 1170 bytes (the
original free block at `$386E`-`$3CFF`) down to 290 bytes -- 880 bytes
used across all twelve patches, plus 15 bytes of confirmed-dead,
not-yet-reclaimed `n-mod-m`.
