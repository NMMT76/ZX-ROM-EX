# ZXROMEX_v1.3_Decisions.md — v1.1 through v1.3

Continues the chronological decision-log convention from
`checkpoint_BETA_Decisions.md`. Picks up after ZXROMEX v1.0 (packaged
Checkpoint Gamma). All addresses below were read directly from a real
assembled build (`pasmo`, verified via its own `.sym` output), not
hand-computed — this project's own standing rule, restated because it
was violated and caught mid-session (see Entry 4).

---

## Entry 1 — EXP Padé[3/3] + `to-power` integer fast path

**Motivation:** v1.0's `EXP` used Padé[2/2] (~6.99e-6 worst-case relative
error). Composed through `^` (`LN`+`EXP`), this made `5000^2` land ~77
units off out of 25,000,000 — visibly wrong for a simple integer square.

**`NEW_EXP_CORE_V2`** (`$3AD2`, 85 bytes) — Padé[3/3] replaces Padé[2/2]
at the `EXP` trampoline (`$36C4`–`$36F8`, now `JP NEW_EXP_CORE_V2` +
padding). `EXP(1) = 2.7182818353` vs true `2.71828182` (~2e-8 error,
down from ~7e-6).

**`TO_POWER_V2`** (`$3B27`) — new integer-exponent fast path for `^`,
bypassing `LN`+`EXP` entirely when the exponent is a whole number.
`5000^2 = 25000000.0` exactly. Also correctly handles negative bases
with integer exponents (`(-2)^3 = -8`), which the original always
errored on via `LN`.

**Four real bugs found via emulator regression, all fixed:**
1. Boolean tests via `LD A,(HL); AND A` on calculator booleans always
   false-positive (byte 0 of a calculator boolean is always `$00`).
   Fixed: `jump-true` throughout, matching the ROM's own convention.
2. `BC` clobbered by `RST 28H` (the calculator dispatcher uses it as
   scratch). Fixed: `PUSH BC`/`POP BC` around every such call.
3. Stale hand-computed `ORG` silently truncated real code after the
   boolean fix shrank the block. Fixed: self-correcting marker label
   (`TO_POWER_BLOCK_END`) instead of a literal.
4. `FP_TO_BC`'s real calling convention is `BC`=unsigned magnitude,
   carry=overflow, zero=sign — not the assumed signed two's-complement
   `BC`. All range/sign-detection code rewritten accordingly.

---

## Entry 2 — `LLIST(expr)` switch mechanism

**Design:** `LLIST(selector)` dispatches through a jump table
(`LLIST_TABLE`, currently 2 entries) to registered target routines,
mirroring `ADDR`'s reuse of `LPRINT`'s token. Parameters for a target
live in a 4-byte slice (`LLIST_SLICE`, `$5B1A`–`$5B1D`) that the caller
`POKE`s before calling — the selector itself is the only thing passed as
a real BASIC argument.

**Governing decisions (apply to every current and future target):**
- The slice has **no persistence guarantee** before, during, or after a
  call. Each target owns it as private scratch; no shared convention
  between targets.
- **Not interrupt-safe by design.** BASIC never runs from inside the
  ISR, so BASIC-only entry is both the intent and the safety guarantee.
- Bad selector (out of table range) is sanitized — structural control
  flow, hard error (`REPORT_C`). Bad slice *data* is the caller's
  responsibility — same "tough luck" tier as a bad `USR` address.
- Return value: `BC`, same `USR`/`STACK-BC` convention.
- `S_LLIST_SWITCH` (`$3BB0`) correctly returns a safe `BC=0` placeholder
  during interactive syntax-checking (`SLS_SYNTAX`, `$3BEA`) rather than
  attempting real dispatch — required, matching `S_ADDR`'s own proven
  precedent for the same reason. (`SYNTAX_Z`'s polarity was
  double-checked against ROM ground truth during Entry 5's
  investigation and confirmed correct here — see that entry.)

**A real, separate bug found and fixed while integrating this:**
extending `ADDR_DISPATCH` (`$38CB`) by 5 bytes to add the `LLIST` token
check overran a hardcoded `ORG $3908` anchor with **zero actual slack**,
silently truncating `ADDR`'s own error-handling `JP NZ,REPORT_C` — its
target address's high byte got overwritten by the next thing after it,
corrupting a previously-shipped, working feature. Caught only by
chasing down an "unaccounted" diff byte, not by the clean assembly
succeeding. The whole downstream chain of hardcoded sequential `ORG`
anchors this cascaded through (`ADDR` → `RND` → `CIRCLE` → free-block
start) was converted to self-correcting marker labels, same pattern as
Entry 1's fix, closing this whole class of bug rather than patching one
instance.

---

## Entry 3 — ZX0 Standard decompression as `LLIST(0)`

A flood-fill implementation was designed, fully built, and fully
verified first (rectangle containment, no-op on already-set seed,
edge-touching fills, worst-case whole-screen fill, Y-clamping) but
didn't fit the remaining free ROM block alongside everything else
(needed 466 bytes, 341 available at the time). Deferred rather than cut
down — full design, code, and test results preserved in
`MightImplement.md` for a future checkpoint with more space.

**`ZX0_DECOMPRESS`** (`$3BFA`) wraps **`DZX0_STANDARD`** (`$3C08`, 68
bytes) — the real, unmodified upstream decompressor by Einar Saukas &
Urusergi. Confirmed byte-for-byte identical to the authoritative
upstream source (the user's own copy of `dzx0_standard.asm`, assembled
standalone and diffed against the embedded copy — the only differences
were the 4 internal `CALL` targets' addresses, all shifting by the
identical relocation delta, i.e. pure address relocation with zero
content divergence). Round-trip verified against a real, known-good
compressed test vector (not one generated by this project's own tooling
— an earlier attempt using a third-party `zx0` PyPI package produced a
compressed stream that failed to decompress correctly, later determined
to be an incompatibility in that package, not in `DZX0_STANDARD`).

Reads a 2-byte source address and 2-byte destination address from
`LLIST_SLICE`. Always returns `BC=0` (ZX0's own format has no
length/error signal to report). Same "tough luck" contract as `USR` —
a bad destination address can corrupt memory or crash; that's the
caller's responsibility, same as any other raw-address mechanism in this
project.

---

## Entry 4 — `LLIST(1)`: GOTO/GOSUB/RETURN/`NEXT` line-address cache

**Investigation first:** all four of `GO TO`, `GO SUB`, `RETURN`, and
`NEXT` were confirmed to funnel through exactly one choke point —
`LINE-NEW` (`$1B9E`), reached from `STMT-RET` after every statement,
making the single call to `LINE-ADDR` (`$196E`) that does the actual
O(n) linear scan from `PROG`. None of the four call `LINE-ADDR`
directly; they all just populate `NEWPPC`/`NSPPC` and let `LINE-NEW`
do the work. This meant one interception point could transparently
accelerate all four.

Register-flow tracing (not assumed) confirmed `LINE-ADDR`'s `DE` output
is dead by the time it matters — `LINE-USE` (`$1BBF`) recomputes its own
`DE` from scratch by reading the found line's own record, never
consuming what `LINE-ADDR` left there. A cache hit therefore only needs
to produce `HL` (address) and the `Z` flag (exact match) to be a fully
transparent substitute.

**Storage:** 32 slots × 4 bytes (line number + address) in the tail 128
bytes of `PRBUFF` (`CACHE_BASE = $5B80`–`$5BFF`), leaving `$5B1F`–`$5B7F`
free for future use, per explicit instruction rather than using all
remaining space. Empty slot = line-number field `$0000` (line 0 is never
a valid BASIC line number). Linear search, not hash-indexed (explicit
choice over a direct-mapped design, given a bounded 32-slot scan is
cheap and avoids collision-driven "silently no faster" behavior).

**`CACHE_INSERT`** (`$3C4C`, LLIST(1)) — reads slot (`slice[0]`, 0-31)
and line number (`slice[1..2]`, little-endian) from the slice.
`slot=0,line=0` clears the whole cache, returning `0` (no special-cased
success value — deliberately, "the caller knows what they called").
Slot out of range, or line not found via the real unmodified
`LINE-ADDR`, refuses the write entirely and returns `0`. A valid insert
returns the resolved **address**, not just `1` — lets the caller use the
result directly without a second lookup. Insertion validates slot range
and line existence up front, unlike the address-trusting convention
elsewhere in this project (ZX0's raw addresses, `USR`) — a deliberate
exception: an unchecked slot number here is a raw array index that could
write past the cache block into adjacent memory, a structural safety
issue rather than a data-sanity one.

**`CACHE_LOOKUP_OR_REAL`** (`$3CB3`) — the `LINE-NEW` interception.
Linear scan across all 32 slots; hit returns the cached address with `Z`
forced set; miss tail-jumps (`JP`, not `CALL`) into the real
`LINE-ADDR`, so its own `RET` returns straight to `LINE-NEW` — fully
transparent, no extra stack bookkeeping. `LINE-NEW`'s own
`CALL LINE-ADDR` was replaced with `CALL CACHE_LOOKUP_OR_REAL` — same
3-byte instruction size either way, so nothing downstream needed
re-anchoring.

**Space:** the whole mechanism (`CACHE_INSERT` + `CACHE_SLOT_ADDR` +
`CACHE_CLEAR` + `CACHE_LOOKUP_OR_REAL`) is 172 bytes assembled, fitting
182 available at integration time (fits with margin; earlier drafts
needed mechanical trimming — shared slot-address-computation subroutine,
`JR` instead of `JP` where in range, an OR-chain instead of three
separate reload-and-compare sequences for the clear-all check — to close
an initial 26-byte gap without cutting functionality).

**Correctness (isolated, high confidence):** 11+ unit tests against
`CACHE_INSERT`/`CACHE_LOOKUP_OR_REAL` directly, including multi-slot
correctness, slot=0 with a genuinely nonzero line (must not be
misread as clear-all), the upper boundary slot 31, overwrite/eviction
behavior, and — specifically checked after being raised as a concern —
all 32 slots populated simultaneously, confirming the scan terminates on
a bounded counter reaching 32 rather than any sentinel-based mechanism,
so a full cache correctly falls through to the real scan for any
genuinely uncached line rather than hanging or false-matching.

**Correctness under full, real BASIC execution: unresolved as of v1.3.**
See Entry 5 and `UnderInvestigation.md`.

---

## Entry 5 — `LLIST(1)` real-world bug report: investigated, not resolved

A concrete, deterministic bug was reported against the shipped v1.1.1
build: `Nonsense in BASIC at line 50` on every run of
`zxromex_cache_test.tap`, both loaded from tape and after manually
retyping the line fresh — ruling out a tape-tokenization discrepancy.
Extensive investigation (full detail, including every hypothesis tried
and ruled out, in `UnderInvestigation.md`) found:

- **A real bug, but in this project's own test tooling, not the ROM**:
  `rom_harness.py`'s `setup_basic_environment()` had `FLAGS=$00` labeled
  "runtime, not syntax-checking" — backwards, per the ROM's own ground
  truth (`SET 7,(IY+$01)` = running program, `RES 7,(IY+$01)` = checking
  syntax, confirmed via `grep` across the whole source). This had been
  silently running every test using that harness function in
  syntax-check mode instead of real runtime for the whole session.
  **Fixed** in the harness (now `$80`).
- `SYNTAX_Z`'s polarity in `S_LLIST_SWITCH` itself, initially suspected,
  was checked against this same ground truth and against `S_ADDR`'s
  proven precedent and confirmed **correct**, not a bug.
- `CACHE_INSERT`'s core logic and the full post-`SCANNING` dispatch path
  (selector parsing, table lookup, `Z`-flag handling) were proven
  correct in isolation, including with the exact slot/line values from
  the failing test.
- The tokenized bytes for `(1)` were confirmed byte-for-byte structurally
  identical to the already-proven-working `(0)`.
- A **control test** — the identical test methodology applied to the
  known-good `LLIST(0)` — also produced the same false error, proving
  the emulator test methodology itself (not `LLIST(1)`) was unreliable
  for this class of test. No further conclusions were drawn from that
  methodology after this was discovered.
- A more faithful reproduction attempt (real tokenized program bytes
  loaded into `PROG`, driven through the genuine `STMT-RET` execution
  loop rather than any synthetic stub) still hit an incomplete-system-
  state wall (`WAIT-KEY`), confirming the project's emulator harness
  cannot currently drive a full, real BASIC program through genuine
  statement-by-statement execution. This is a real, still-open tooling
  gap, not specific to this bug.

**Then, separately and still unexplained:** the user reported that a
rebuilt binary (labeled v1.2, produced purely to check versioning) then
passed the same cache test suite that v1.1.1 had failed. **The two ROM
binaries are byte-for-byte identical (SHA256 match)** — confirmed before
and after this report. Since the ROM did not change, this cannot be a
ROM code difference; the most likely explanations are stale
machine/emulator state from the extensive interactive editing done
while diagnosing, or a different file having actually been loaded — but
neither was confirmed before the investigation was closed out. **This is
the single most important open thread for whoever picks this up next.**
Full detail, priority-ordered next steps, and a purpose-built stress/
performance test (`zxromex_v1.3_stress_test.tap`, includes running the
whole suite twice via `GOSUB` specifically to probe for state-dependent
behavior) are in `UnderInvestigation.md`.

**Status: ZXROMEX v1.2 and v1.3 are byte-identical to v1.1.1.** No ROM
code changed across these three labels — only test tooling and
documentation. Do not treat the version number increase as implying a
functional change.
