# ZXROMEX v1.5

ZXROMEX is a "test project with a purpose". Its intended to replace
some math functions in the ZX Spectrum with faster (and slightly less
accurate) versions. The test part is that it was a test of Claude Code's
capabilities. While i architected the works, Claude wrote all the code
as i can not code in Z80 asm. That said, it performed well and managed to
overcome the challenges working within my set restrictions.

# The ROM itself:

A patched Sinclair ZX Spectrum 48K ROM: every slow math primitive
rewritten, the random-number generator replaced with a real algorithm,
`CIRCLE` rewritten from floating-point to integer math, a new `ADDR`
BASIC function, ZX0 decompression callable from BASIC, and a
programmer-managed line-address cache that measurably speeds up
`GO TO`/`GO SUB`/`RETURN`/`NEXT` in real programs — all on a ROM that
is a drop-in replacement for the stock 48K ROM, with every unmodified
routine, every system variable address, and every standard keyword
behaving exactly as it does on a real, unmodified machine.

This supersedes v1.0 through v1.3. Nothing in this package is a
work-in-progress: the one open issue carried forward from earlier
checkpoints (a real, reproducible `LLIST(1)` cache bug) is root-caused
and fixed in this release — see `docs/DECISIONS_4_CACHE_FIX.md` for the
full story.

## What's actually different from a stock 48K ROM

| Area | What changed |
|---|---|
| `SQR`, `SIN`, `COS`, `TAN`, `ATN` | Rewritten from scratch for speed. `ASN`, `ACS`, and `^` (power) inherited the improvement for free. |
| `LN`, `EXP` | Rewritten from scratch for speed. `EXP` accuracy improved again in v1.1 (Padé[2/2] -> Padé[3/3], worst-case relative error ~7e-6 -> ~2e-8). |
| `^` (to-power) | Integer-exponent fast path added in v1.1: bypasses `LN`+`EXP` composition for whole-number exponents, and correctly handles negative bases with integer exponents (previously always errored). |
| `LPRINT(var)` | A BASIC function returning a variable's address in memory — reclaimed from `LPRINT`'s token via context-sensitive interception during expression scanning (see "How the reclaimed keywords work" below). |
| `RND` | Replaced with Boriel BASIC's CMWC (complementary multiply-with-carry) algorithm, redesigned to not rely on self-modifying code (impossible from ROM) while preserving the original algorithm's actual statistical behavior. |
| `CIRCLE` | Rewritten from the original's slow floating-point trigonometry to an integer Midpoint Circle Algorithm implementation, pixel-verified against a Python reference model using the ROM's own `PIXEL-ADD` as ground truth. |
| `LLIST(expr)` | `LLIST`'s token reclaimed the same way `ADDR` reclaims `LPRINT`'s, dispatching by selector to registered targets. |
| `LLIST(0)` | ZX0 Standard decompression, callable from BASIC. Wraps the real, unmodified upstream `dzx0_standard` decompressor (Einar Saukas & Urusergi). |
| `LLIST(1)` | A 32-slot, programmer-managed `GO TO`/`GO SUB`/`RETURN`/`NEXT` line-address cache. See "The line-address cache" below. |
| Printer buffer (`PRBUFF`) | Neutered and reclaimed as real, usable RAM scratch space — the actual physical printer channel this occupied is not used by anything on a real 48K machine without a peripheral attached. |
| Free ROM space | 1170 bytes (stock) -> 3 bytes. See table below — the honest number, explained. |

### Free space, honestly

| | ROM free space (`$386E`-`$3CFF`) | RAM free (`PRBUFF`, `$5B00`-`$5BFF`) |
|---|---:|---:|
| Stock 48K ROM | 1170 bytes | 0 bytes (real printer buffer) |
| ZXROMEX v1.0 | 563 bytes | 230 bytes (of 256 reclaimed) |
| **ZXROMEX v1.5** | **3 bytes** | **91 bytes** (of 256 reclaimed) |

v1.1 through v1.5 spent most of v1.0's remaining ROM and RAM headroom
on the `LLIST` switch mechanism, ZX0 decompression, and the 32-slot
cache (128 bytes of `PRBUFF` alone). What's left is genuinely tight —
see `docs/FUNCTION_SPECS_ADDENDUM_CACHE.md`'s space-accounting table
for the exact breakdown. Any further ROM-resident feature needs a real
space-reclamation pass first.

## How the reclaimed keywords work

`LPRINT` isn't replaced outright, and `LLIST(0)`/`LLIST(1)`
don't replace `LLIST` outright — each shares its host keyword's token,
disambiguated by where the token appears. If the token shows up
somewhere a function/value is expected (e.g. `LET X = LPRINT(Y)` or
`LET R = LLIST(1)`), it's intercepted during expression scanning and
redirected to the new logic. If it shows up as a full statement on its
own, it still calls the genuine, untouched original routine (`LPRINT`
or `LLIST`, printer output). **`LIST`ing a program will always display
these tokens' text as `LPRINT`/`LLIST`**, even when they're
functionally being used as the new behavior — a known, accepted
display-level quirk, not a bug.

## The line-address cache (`LLIST(1)`)

A real Sinclair BASIC program resolves every `GO TO`, `GO SUB`,
`RETURN`, and `NEXT` target by linearly scanning the tokenized program
from the top until it finds (or passes) the target line — every single
time, for every jump, no matter how many times the same target has
been reached before. `LLIST(1)` lets a program tell the ROM, once, "the
address for line N is right here" — up to 32 such entries, entirely
under program control.

**Priming a slot:**

```basic
10 POKE 23322, slot   : REM 0-31
20 POKE 23323, target-line MOD 256
30 POKE 23324, target-line / 256 (rounded down)
40 LET r = LLIST(1)   : REM r = resolved address on success, 0 on failure
```

**Clearing everything:** `POKE 23322,0: POKE 23323,0: POKE 23324,0: LET r=LLIST(1)`
resets all 32 slots.

The cache is **programmer-managed, not automatic** — nothing
invalidates a slot if the program text is edited after inserting it.
Re-clear and re-insert after editing a cached line's position.

**What it's worth in practice:** on a real-world benchmark (a sphere
raytracer, cache-primed with its 13 hottest jump targets, frequency-
ranked from an exact dynamic count — see `docs/DECISIONS_4_CACHE_FIX.md`,
Entry 3), total render time dropped ~4.7% from priming alone, with zero
changes to the program's own logic or line numbers. On jump-dense
synthetic loops with cheap loop bodies, the effect is much larger
(roughly 2x for `FOR`/`NEXT` and `GO TO` in this project's own stress
test) — the cache removes a fixed per-jump cost, so its relative payoff
scales with how much of each iteration was jump dispatch to begin with,
versus real work like floating-point arithmetic.

## Building

```bash
pasmo ZXROMEX_v1.5.asm ZXROMEX_v1.5.bin
```

A pre-built `ZXROMEX_v1.5.bin` (16384 bytes) is included in this
package, built and verified exactly this way — checked byte-for-byte
against the source at packaging time.

## Loading

`ZXROMEX_v1.5.bin` is a standard 16KB 48K ROM image, loadable in any
emulator that accepts a custom ROM image (Fuse, etc.) or writable to
real ROM-replacement hardware. It is a drop-in replacement for the
stock 48K ROM.
