# FUNCTION_SPECS.md — Checkpoint Gamma ("Nothing left on the table")

Structural reference for `checkpoint_gamma_Spectrum48_patched.asm`:
exact current addresses, sizes, and layout for every routine touched
across Checkpoint Alpha and the Beta repacking effort. For algorithm
derivations, accuracy figures, and design rationale, see
`checkpoint_alpha_FUNCTION_SPECS.md` (unchanged — no algorithm moved or
changed in Beta, only *location*). For the narrative of how each
relocation was done and verified, see `checkpoint_BETA_Decisions.md`.

Every address below was cross-checked two ways before being written
here: read directly from the deliverable's own `ORG` statements, and
independently confirmed by chained arithmetic (`start + size = next
start`) across the entire main-block sequence — see the table at the
end of this document for that full chain, verified with no gaps and no
overlaps.

---

## Trampoline-hosted routines (relocated in Beta)

Seven routines now live inside trampoline slots elsewhere in the ROM
instead of the main free block. Three of the seven sit inside their
*own* trampoline (self-fit, no `JP` needed at all — execution falls
straight through); the other four sit inside a *different* function's
spare trampoline padding, reached via a separate `JP` from their own
dispatch point.

| Routine | Size | Host slot | Slot address | Slot total | Local slack |
|---|---:|---|---|---:|---:|
| `NEW_EXP_CORE` | 52 | own (`EXP`) | `$36C4` | 53 | 1 |
| `NEW_ATN_CORE` | 51 | own (`ATN` `CASES`) | `$37FA` (+1 `end-calc`) | 57 | 5 |
| `NEW_LN_TAIL` | 36 | own (`LN` `GRE.8`) | `$374A` (+1 `end-calc`) | 57 | 20 |
| `NEW_S_RND` | 29 | own (`S-RND`) | `$25F8` | 47 | 18 |
| `NEW_COS_CORE` | 30 | `C-ENT` (`SIN`/`COS` shared) | `$37BB` (host header `$37B7`, 4B) | 35 | 1 |
| `NEW_TAN_CORE` | 67 | `CIRCLE` | `$2334` (host header `$2331`, 3B) | 81 | 11 |
| `RND_DEFAULT_TABLE` | 8 | `COS` | `$37AD` (host header `$37AA`, 3B) | 11 | **0** |

### Detail per relocated routine

**`NEW_EXP_CORE`** (`$36C4`–`$36F7`, 52 bytes) — entry point for the
`exp` calculator literal itself; no `JP` needed, the dispatch table
entry lands directly on this code. Ends with `JP $36F9`, the reused
original overflow-handling tail (unmoved). 1 byte of `$00` slack at
`$36F8`.

**`NEW_ATN_CORE`** (`$37FB`–`$382D`, 51 bytes) — reached via
`end-calc` at `$37FA` (required: this slot is entered mid-calculator-
literal-stream, both by falling through from `SMALL` and via a
calculator-level `jump`/`jump-true` from the `BIG` case), then falls
straight into the routine's own fresh `RST 28H`. 5 bytes of `$00`
slack at `$382E`–`$3832`.

**`NEW_LN_TAIL`** (`$374B`–`$376E`, 36 bytes) — same `end-calc`-first
pattern as `NEW_ATN_CORE`, required for the same reason (mid-literal-
stream entry via `GRE.8`). 20 bytes of `$00` slack at `$376F`–`$3782`
— the widest local margin of any self-fit relocation.

**`NEW_S_RND`** (`$25F8`–`$2614`, 29 bytes) — entry point for `S-RND`
itself (fresh Z80 code via the scan-func dispatch table, first
instruction a plain `CALL`); no `end-calc` ever needed here. Ends with
`JP $2630` (`S-PI-END`, unmoved). 18 bytes of `$00` slack at
`$2615`–`$2626`.

**`NEW_COS_CORE`** (`$37BB`–`$37D8`, 30 bytes) — hosted inside `C-ENT`'s
own slot, *not* `COS`'s own trampoline (`COS`'s slot is only 11 bytes
total, far too small). `C-ENT`'s own required content — `end-calc`
(`$37B7`) + `JP NEW_SIN_CORE` (`$37B8`–`$37BA`) — is unrelated and
unaffected; `NEW_COS_CORE` occupies the padding that follows, reached
only via a separate `JP` from `COS`'s own trampoline at `$37AA`. 1 byte
of `$00` slack at `$37D9`.

**`NEW_TAN_CORE`** (`$2334`–`$2376`, 67 bytes) — hosted inside
`CIRCLE`'s own trampoline slot at `$2331`. `CIRCLE`'s own required
content — `JP NEW_CIRCLE_CORE` (`$2331`–`$2333`) — is unrelated and
unaffected; `NEW_TAN_CORE` occupies the padding that follows, reached
only via a separate `JP` from `TAN`'s own trampoline at `$37DA`. 11
bytes of `$00` slack at `$2377`–`$2381` — the tightest cross-slot fit
in the project.

**`RND_DEFAULT_TABLE`** (`$37AD`–`$37B4`, 8 bytes) — pure data
(`DEFB 82,97,120,111,102,116,20,12`), hosted inside `COS`'s own
trampoline slot at `$37AA`. `COS`'s own required content — `JP
NEW_COS_CORE` (`$37AA`–`$37AC`) — is unrelated and unaffected;
the table occupies the padding that follows. **Zero bytes of slack** —
the only perfect zero-waste fit in the project. Referenced via
`LD HL,RND_DEFAULT_TABLE` from `RND_BOOT_SEED` and `RND_RESEED`.

---

## Trampolines still holding only their own routine (unmoved)

| Trampoline | Address | Total size | Contents | Local slack |
|---|---|---:|---|---:|
| `SQR` | `$384A` | 7 | `JP NEW_SQR` | 4 |
| `TAN` | `$37DA` | 8 | `JP NEW_TAN_CORE` (now targets `$2334`) | 5 |
| `RANDOMIZE` tail | `$1E5A` | 5 | `JP RND_RESEED` | 2 |

These three residual pockets (11 bytes total) are confirmed too small
for any routine in the ROM and are not worth reclaiming — see Beta
`Decisions.md` Entry 2's original analysis, unchanged by anything done
since.

---

## Main free-block chain (routines still resident there, in address order)

Every entry below flows immediately into the next with **zero gap**,
confirmed both by the file's own `ORG` statements and by independent
chained arithmetic (`start_address + size = next_start_address`) run
across the whole sequence.

| Address range | Contents | Size (bytes) |
|---|---|---:|
| `$386E`–`$38A6` | `NEW_SQR` (Newton-Raphson `SQR`, Alpha Patch 1) | 57 |
| `$38A7`–`$38CA` | `NEW_SIN_CORE` (Bhaskara I `SIN`, Alpha Patch 2) | 36 |
| `$38CB`–`$3907` | `ADDR_DISPATCH`/`S-ADDR` (`ADDR(var)`, Alpha Patch 5) | 61 |
| `$3908` | `SAFE_ZERO_BYTE` (Alpha Patch 7) | 1 |
| `$3909`–`$3914` | `CL_PRINTER_FIX` (Alpha Patch 7) | 12 |
| `$3915`–`$3941` | `NEW_RND_BYTE` (Alpha Patch 8) | 45 |
| `$3942`–`$3954` | `RND_BOOT_SEED` (Alpha Patch 8) | 19 |
| `$3955`–`$397D` | `RND_RESEED` (Alpha Patch 8) | 41 |
| `$397E`–`$3984` | `RND_BOOT_STUB` (Alpha Patch 8) | 7 |
| `$3985`–`$3ACC` | `CHECK_AND_PLOT`/`PLOT4`/`NEW_CIRCLE_CORE` (Alpha Patch 12) | 328 |
| `$3ACD`–`$3CFF` | **Free ROM: 563 bytes** | 563 |
| `$3D00`–... | Character bitmap table — untouched | — |

**RAM** (`PRBUFF`-owned, `$5B00`–`$5BFF`, since Alpha Patch 7) is
completely unaffected by Beta — no RAM layout changed, only ROM code
addresses:

| Address range | Contents | Size (bytes) |
|---|---|---:|
| `$5B00`–`$5B0B` | `RND` state (Alpha Patch 8) | 12 |
| `$5B0C`–`$5B19` | `CIRCLE` working variables (Alpha Patch 12) | 14 |
| `$5B1A`–`$5BFF` | Free RAM: 230 bytes (unchanged since Alpha) | 230 |

---

## Confirmed dead code, still not reclaimed

**`n-mod-m`** (`$36A0`–`$36AE`, 15 bytes, calculator literal `$32`) —
unchanged from Checkpoint Alpha: still confirmed dead (no ROM-internal
caller since `RND` was replaced in Alpha Patch 8), still deliberately
not reclaimed, since it remains a technically-available direct-dispatch
calculator literal for any user machine-code program. See Alpha
`Decisions.md` for the full reasoning; nothing done in Beta changes
this tradeoff or this recommendation.

---

## Full-chain verification

The entire main-block sequence above was independently checked with
chained arithmetic before this document was written:

```
$386E + 57  = $38A7   (NEW_SQR -> NEW_SIN_CORE)
$38A7 + 36  = $38CB   (NEW_SIN_CORE -> ADDR_DISPATCH)
$38CB + 61  = $3908   (ADDR_DISPATCH -> SAFE_ZERO_BYTE)
$3908 + 1   = $3909   (SAFE_ZERO_BYTE -> CL_PRINTER_FIX)
$3909 + 12  = $3915   (CL_PRINTER_FIX -> NEW_RND_BYTE)
$3915 + 45  = $3942   (NEW_RND_BYTE -> RND_BOOT_SEED)
$3942 + 19  = $3955   (RND_BOOT_SEED -> RND_RESEED)
$3955 + 41  = $397E   (RND_RESEED -> RND_BOOT_STUB)
$397E + 7   = $3985   (RND_BOOT_STUB -> CHECK_AND_PLOT/PLOT4/NEW_CIRCLE_CORE)
$3985 + 328 = $3ACD   (CIRCLE core -> free space)
$3ACD + 563 = $3D00   (free space -> character bitmap table)
```

Every step matches the deliverable's own `ORG` statements exactly —
zero gaps, zero overlaps, zero unexplained bytes anywhere in the chain.
