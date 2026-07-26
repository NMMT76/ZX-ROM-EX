# ZXROMEX_v1.5_Decisions.md — the line-0 cache bug, start to finish

Continues `ZXROMEX_v1.3_Decisions.md`. Picks up from `UnderInvestigation.md`
(v1.3-era, now resolved and folded into this log) and closes it out. All
addresses below were read from a real `pasmo` assembly (`.sym` output), not
hand-computed.

---

## Entry 1 — Root cause: line 0 collides with the cache's own empty-slot sentinel

**Symptom:** `LLIST(1)` (the GOTO/GOSUB/RETURN/NEXT cache) worked on a
program's first `RUN`, but any subsequent `RUN` in the same session — with
the program completely unchanged — crashed immediately with
`Nonsense in BASIC, 0:1`, before the program's own first line ever executed.

**Diagnosis:** `CACHE_LOOKUP_OR_REAL`'s "slot is empty" marker is a
line-number field of `$0000` — chosen because line 0 is never a valid
BASIC line. But `$0000` is *also* exactly the line number a bare `RUN`
(no line argument) asks `LINE-NEW` to locate: `RUN`'s syntax class
defaults the missing argument to `0` (`USE-ZERO`), and `GO-TO` stores
that `0` straight into `NEWPPC`. `LINE-NEW` then calls
`CACHE_LOOKUP_OR_REAL` with `HL=0` — before line 1 of the program has
had any chance to run, since this happens as part of `RUN`'s own
dispatch, not the program's logic.

`CACHE_SCAN_LOOP` scans for a slot whose line field matches the target.
Any never-used or `CACHE_CLEAR`ed slot has a line field of `$0000` —
identical to the target. Result: a spurious "exact match" against the
first empty slot, returning whatever garbage happens to sit in that
slot's *address* bytes — `CACHE_CLEAR` zeroes only the line fields,
leaving address bytes untouched (previously assumed harmless, since
they're "only read when the line field is nonzero" — an assumption
this bug violated). `LINE-NEW` treats that garbage as a genuine line
pointer and sets `PPC` from whatever's there.

**Why run 1 worked and run 2 didn't:** on a fresh load, the whole cache
block is pristine zero — line *and* address bytes — so the false hit
resolves to address `$0000`, which happened not to visibly break.
After a real run populates real slots with real addresses, the *next*
run's line-0 lookup — which fires before that run's own cache-clear
line gets to execute — sees leftover, non-trivial garbage in an empty
slot's address bytes instead, and that's what broke.

**Fix** (`CACHE_LOOKUP_OR_REAL`, 5 bytes prepended):

```asm
CACHE_LOOKUP_OR_REAL:
        LD      A,H
        OR      L
        JP      Z,LINE_ADDR     ; target line 0 is never a valid cached
                                 ; entry (also the empty-slot sentinel) --
                                 ; always defer to the real search
        LD      A,H
        LD      (CACHE_TARGET_HI),A
        ...
```

Line 0 can never legitimately be cached anyway (the code's own
invariant), so refusing to consult the cache for it costs nothing and
closes the collision entirely. `HL` is already the target (`0`), which
is exactly `LINE_ADDR`'s own entry contract — no setup needed before
the tail jump.

**Verification:** reproduced directly (poisoned an empty slot's address
bytes with a recognizable value, called `CACHE_LOOKUP_OR_REAL` with
`HL=0`, confirmed the false hit on the pre-fix binary and its absence
post-fix) plus regression checks confirming real cache hits, real
misses, and nonexistent-line lookups are all unaffected. Full binary
diff against the pre-fix build: 72 bytes differ, entirely contained to
`$3CB4`–`$3CFC` (the cache routine itself); character-set table at
`$3D00`+ byte-identical. Confirmed on real hardware: `RUN` now repeats
cleanly, any number of times.

---

## Entry 2 — Test-suite bugs uncovered once Entry 1 stopped masking them

Fixing Entry 1 let the stress test (`zxromex_v1.3_stress_test.bas`) run
far enough to expose several pre-existing bugs in the *test listing
itself* — none of them ROM issues. Recorded here because they cost real
debugging time and are exactly the kind of thing worth not
rediscovering.

### 2a. Multi-character `FOR` loop variables

`FOR slotn=0 TO 31` / `FOR passn=1 TO 5` — invalid on real Sinclair
BASIC. Confirmed directly against the ROM's own `F-REORDER` (`FOR`'s
setup): it unconditionally treats the single byte immediately before
the loop variable's value as *the* name byte via `DEC HL`, with no
handling for a multi-character variable's longer descriptor format.
`F-LOOP` (used by `NEXT` to find its matching `FOR`) likewise reads and
compares exactly one character after the `NEXT` token. Neither errors
at parse time (BASIC's syntax checker doesn't restrict variable name
length in general); the failure is memory corruption at `FOR`-execution
time, surfacing later as unrelated-looking `Nonsense in BASIC` errors
wherever the corrupted bytes get reinterpreted.

Fix: renamed to single-character variables (`slotn`→`n`, `passn`→`p`),
checked against every other variable in the file for collisions.

### 2b. Uninitialized variable read

`1560 LET x=x+1` with `x` never previously assigned — a genuine runtime
bug (`2 Variable not found` on real hardware), not just a `bas2tap`
static-check complaint. `bas2tap -n` suppresses the *warning*; it does
not fix the *program*. Fixed by initializing `LET x=0` before the first
loop that uses it.

### 2c. Mistargeted performance-test cache inserts

The `P1` (FOR/NEXT) and `P2` (GO TO) sections of the performance
benchmark each time two loops — one uncached, one with a cache entry
inserted beforehand — and compare. Both inserts targeted the *first*
loop's line number instead of the *second* (the one actually being
timed): `P1` cached line 1560 instead of 1620; `P2` cached line 1690
instead of 1760. Every "cached" iteration was therefore a genuine cache
**miss** — full 32-slot scan, then fall-through to the real search —
strictly worse than the uncached baseline, which never touches the
cache at all. This is what produced the initial, misleading result
("cached marginally *slower*"). `P3` (GO SUB) was unaffected — both its
timed loops call the same target (`GO SUB 2100`) — which is exactly why
it was the only section showing a real speedup before this fix.

Fix: corrected both `POKE` sequences to the real second-loop targets.
Post-fix, `FOR`/`NEXT` and `GO TO` both roughly doubled in speed under
the cache; `GO SUB` unchanged (~35% faster, as before) — consistent
with jump-dense tight loops amortizing the cache's fixed per-lookup
saving more often than a subroutine call that fires once per several
statements.

---

## Entry 3 — Real-world benchmark: sphere raytracer

Used as a "generic benchmark" (a user-supplied raytracer, not written
for this project) to check the cache's benefit on realistic, non-synthetic
code, with the additional constraint of leaving the program's original
line numbers/entry points untouched.

**Frequency analysis, not guesswork:** rather than trying to drive a
full 704-block real BASIC run through the emulator harness (see
`ZXROMEX_v1_3_Decisions.md`'s notes on that harness's inability to get
a genuine multi-line program through statement-by-statement execution
without stalling), the program's exact control flow — same loop bounds,
same `IF` conditions, same hardcoded sphere data — was ported to Python
and every dynamic jump event counted by target line. Since the scene is
fully deterministic, this gives exact counts, not estimates:

| line | count | what it is |
|---|---:|---|
| 1102 | 28,251 | `NEXT S` — sphere loop inside the ray/sphere test |
| 1500 | 24,341 | `GO TO` on a sphere miss (`D<0`) |
| 1000 | 9,417 | `GO SUB` into the ray/sphere routine |
| 130 / 311 | 6,160 each | `NEXT V`, pixel-collection / drawing loops |
| 125 / 202 / 208 / 310 | 770 each | `NEXT U` / `NEXT C` loops |
| 25 | 672 | `NEXT Y` |
| 500 | 594 | `GO TO`, fast-path (uniform-block) skip |
| 160 | 440 | `GO TO`, corner-color-reuse shortcut |
| 20 | 31 | `NEXT X` |

All 13 fixed jump targets in the program fit in the cache's 32 slots
with room to spare, ordered by frequency into slots 0–12 (hottest in
slot 0, minimizing average linear-scan cost on `CACHE_SCAN_LOOP`'s
side, since a genuine miss always scans all 32 regardless of how many
are populated, but a hit's cost is exactly the scanned-slot index).

**Result:** 6520 frames → 6213 frames, ~4.7% faster, entirely from a
one-time cache-priming subroutine (`GO SUB 4000`, 13 `POKE`+`LLIST(1)`
pairs) with zero changes to the program's own logic. Smaller than the
stress test's synthetic ~2x, for a real reason: the hottest loop here
(`NEXT S`/`GO TO 1500`, the ray/sphere intersection test) does
substantial floating-point work per iteration — several multiplies, a
division, and for surviving spheres a `SQR` call — which dominates
per-iteration cost far more than the line-address lookup the cache
replaces ever did. The cache is doing real, correct work (avoiding over
52,000 linear `PROG` scans just from slots 0–1 alone); the workload
just isn't jump-dispatch-bound the way a tight `LET x=x+1` loop is.
Conclusion: for numerically-heavy BASIC, the cache is a genuine, free
win, but not the dominant lever — that would be calculator-stack
arithmetic cost, which is out of scope for a same-entry-points patch.

---

## Status

All items in `UnderInvestigation.md` are now resolved; that file's
content is superseded by this entry and is not carried forward into the
v1.5 public documentation set. `MightImplement.md`'s deferred-feature
list is unaffected by this session's work and remains open for future
consideration.
