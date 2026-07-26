# Changelog

## v1.0 (packaged from Checkpoint Gamma)

The first packaged release. Internally, this line of work went through
two development phases before packaging:

### Phase 1 (internally: Checkpoint Alpha)

- Rewrote `SQR`, `SIN`, `COS`, `TAN`, `ATN`, `LN`, `EXP` for speed.
  `ASN`, `ACS`, and `^` inherited the improvement for free.
- Added `ADDR(var)`, a new BASIC function returning a variable's
  address, reclaimed from `LPRINT`'s token.
- Replaced `RND` with Boriel BASIC's CMWC algorithm (redesigned to not
  require self-modifying code, which cannot work from ROM).
- Rewrote `CIRCLE` from floating-point trigonometry to an integer
  Midpoint Circle Algorithm.
- Neutered the printer buffer (`PRBUFF`) to reclaim 256 bytes of real
  RAM as scratch space.
- Net free ROM space: 1170 -> 290 bytes.
- Fifteen real bugs found and fixed along the way, full detail in
  `docs/DECISIONS_1_ALPHA.md`.

### Phase 2 (internally: Checkpoint Beta / packaged as Gamma)

- Zero algorithm, accuracy, or behavioral changes from Phase 1 --
  purely a space-reclamation pass.
- Relocated seven routines Phase 1 had left in the main free block
  into previously-wasted, oddly-shaped trampoline padding elsewhere in
  the ROM, freeing that space for the contiguous free block instead.
- Net free ROM space: 290 -> 563 bytes (a 94% increase over Phase 1,
  48% more than the space available at the very start, once Phase 1's
  own additions are accounted for).
- Two stale-`ORG` bugs, one hand-counting error, and one delivery
  mistake found and fixed along the way, full detail in
  `docs/DECISIONS_2_BETA.md`.

### Packaging

- Renamed and packaged as ZXROMEX v1.0. No source changes from
  Checkpoint Gamma -- this is that deliverable, verified byte-for-byte
  identical at packaging time, under a stable public name.

## v1.1 / v1.1.1 / v1.2 / v1.3

v1.2 and v1.3 were byte-for-byte identical to v1.1.1 (confirmed via
SHA256) -- no ROM source changed across those three labels, only test
tooling and documentation.

### Added

- **`EXP` accuracy fix**: Padé[2/2] -> Padé[3/3]. Worst-case relative
  error ~7e-6 -> ~2e-8.
- **`to-power` (`^`) integer-exponent fast path**: bypasses `LN`+`EXP`
  composition for whole-number exponents. Fixes `5000^2` landing ~77
  units off out of 25,000,000 under the old composed path. Also newly
  handles negative bases with integer exponents correctly (previously
  always errored via `LN`).
- **`LLIST(expr)` switch mechanism**: reuses `LLIST`'s token the same
  way `ADDR` reuses `LPRINT`'s. Dispatches by selector through a jump
  table to registered targets, each with private scratch in a 4-byte
  parameter slice the caller `POKE`s beforehand.
- **`LLIST(0)`: ZX0 Standard decompression.** Wraps the real, unmodified
  upstream `dzx0_standard` decompressor (Einar Saukas & Urusergi).
  Reads a source and destination address from the parameter slice.
- **`LLIST(1)`: GOTO/GOSUB/RETURN/`NEXT` line-address cache.** 32
  programmer-managed slots. Insert via `LLIST(1)` after `POKE`-ing a
  slot number and target line number into the slice; validates both
  before writing, returns the resolved address on success or `0` on any
  failure. Transparently accelerates all four jump mechanisms via a
  single interception point, with no syntax change to any of them.

### Fixed

- A previously-shipped, working feature (`ADDR`) had its own error path
  silently corrupted mid-session while adding the `LLIST` switch --
  extending `ADDR_DISPATCH` overran a stale hardcoded `ORG` anchor with
  zero slack. Found and fixed before delivery, along with the whole
  downstream chain of similar hardcoded anchors it could have cascaded
  through.

### Deferred

- **Flood fill**: fully designed, implemented, and verified in
  isolation. Didn't fit the remaining free ROM space. Full design,
  code, and test results preserved for a future checkpoint (see
  `MightImplement.md`, internal).

### Known open issue at the time (resolved in v1.5, see below)

- A real, deterministic `LLIST(1)` bug was present: any `RUN` after the
  first, in the same session, with the program unchanged, crashed
  immediately. Root cause not found at the time this checkpoint closed.

## v1.5

### Fixed

- **The v1.1–v1.3 `LLIST(1)` bug, root-caused and fixed.**
  `CACHE_LOOKUP_OR_REAL`'s empty-slot sentinel (line-number field
  `$0000`) collided with a genuine lookup for line `0` -- which is
  exactly what a bare `RUN` (no line argument) asks the interpreter to
  locate, before the program's own first statement ever executes. Every
  `RUN` after the cache held any real leftover state produced a false
  cache hit against stale, garbage address bytes. Fixed with a 5-byte
  guard: `CACHE_LOOKUP_OR_REAL` now defers straight to the real
  `LINE_ADDR` search whenever the target is line 0, since line 0 can
  never legitimately be cached in the first place. Full narrative,
  reproduction, and verification in `docs/DECISIONS_4_CACHE_FIX.md`.
- Three unrelated bugs in the v1.3 stress test itself, found once the
  above stopped masking them: multi-character `FOR`/`NEXT` loop
  variables (invalid on real Sinclair BASIC -- confirmed against the
  ROM's own `F-REORDER`/`F-LOOP`, which only ever handle a
  single-character loop-variable name), an uninitialized-variable read,
  and two mistargeted cache-insert `POKE`s in the performance-comparison
  section that made the "cached" `FOR`/`NEXT` and `GO TO` loops silently
  benchmark a permanent cache *miss* instead of a hit. All fixed; see
  `docs/DECISIONS_4_CACHE_FIX.md`, Entry 2.

### ROM space

- Free block: 8 bytes -> 3 bytes (the line-0 fix's 5 bytes came out of
  this project's last remaining slack). No further work fits in this
  block without a real space-reclamation pass.

### New/updated test artifacts

- `tests/stress_test/zxromex_v1_3_stress_test.bas`/`.tap` -- corrected
  per the three bugs above; now passes cleanly end-to-end on real
  hardware, `RUN` any number of times, including the full performance
  comparison section.
- `tests/benchmarks/__gg_rt_03_cache_primed.bas`/`.tap` -- a real-world
  (non-synthetic) sphere raytracer, cache-primed via an exact,
  frequency-ranked slot assignment (see `docs/DECISIONS_4_CACHE_FIX.md`,
  Entry 3): ~4.7% faster end to end from cache priming alone, with the
  program's original line numbers and entry points left untouched.

## Known limitations, honestly stated (unchanged since v1.0)

- `ADDR` shares `LPRINT`'s token, and `LLIST(0)`/`LLIST(1)` share
  `LLIST`'s; `LIST`ing a program always displays the token's stock
  keyword text, regardless of which behavior is actually in use at
  that point. See the README's "How the reclaimed keywords work"
  section.
- 15 bytes (`n-mod-m`, confirmed dead code since before this project
  began) and 11 bytes of residual trampoline slack (`SQR` 4B, `TAN`'s
  own trampoline 5B, `RANDOMIZE`'s tail 2B) remain deliberately
  unreclaimed -- each individually too small to be worth the risk for
  the space saved. Not defects; documented, considered decisions.
- The cache (`LLIST(1)`) is programmer-managed, not automatic: nothing
  invalidates a slot if the program is edited after inserting it.
  Re-`CACHE_CLEAR`/re-insert after any edit to a cached line's
  position.
