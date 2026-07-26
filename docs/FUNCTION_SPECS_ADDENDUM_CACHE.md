# ZXROMEX_v1.5_FUNCTION_SPECS.md — addendum

Continues `ZXROMEX_FUNCTION_SPECS.md`. Covers only what changed or was
added in v1.1 through v1.3; everything from Checkpoint Alpha/Beta/v1.0
is unchanged. Every address below was read from a real `pasmo` assembly
of the current source (`.sym` output), not hand-computed.

## Modified: `EXP` trampoline

| | |
|---|---|
| Trampoline slot | `$36C4`–`$36F8` (53 bytes, unchanged) |
| Now contains | `JP NEW_EXP_CORE_V2` + zero padding |
| `NEW_EXP_CORE_V2` | `$3AD2`, 85 bytes |
| Algorithm | Padé[3/3] (was Padé[2/2]) |
| Accuracy | ~2e-8 worst-case relative error (was ~7e-6) |

## Modified: `to-power` (`^`) head

| | |
|---|---|
| Head | `$3851`–`$385C` (12 bytes, unchanged size) |
| Now contains | `JP TO_POWER_V2` + zero padding |
| `TO_POWER_V2` | `$3B27` |
| `TO_POWER_BLOCK_END` | `$3BB0` (self-correcting marker, not a literal) |
| Behavior | integer-exponent fast path; falls back to `LN`+`EXP` via
  `NEW_EXP_CORE_V2` for non-integer exponents |

## Modified: `ADDR_DISPATCH`

| | |
|---|---|
| Address | `$38CB` (unchanged) |
| Size | 13 bytes (was 8) — extended to also recognize `LLIST_TOKEN` |
| `ADDR_BLOCK_END` | `$390D` (self-correcting marker; the whole
  downstream `ADDR`→`RND`→`CIRCLE`→free-block-start chain now computed
  from real assembled output, not hardcoded literals — see Decisions
  Entry 2 for why this mattered) |

## New: `S_LLIST_SWITCH` and the `LLIST` dispatch table

| | |
|---|---|
| `S_LLIST_SWITCH` | `$3BB0` |
| `SLS_SYNTAX` | `$3BEA` — safe `BC=0` placeholder during interactive
  syntax-checking |
| `SLS_BADSEL` | out-of-range selector → `REPORT_C` |
| `SLS_RETURN` | `$3BE4` — `CALL STACK_BC; JP S_NUMERIC` |
| `LLIST_TABLE` | 2 entries currently (`LLIST_TABLE_COUNT = 2`) |
| `LLIST_SLICE` | `$5B1A`–`$5B1D`, 4 bytes, in `PRBUFF` |
| `LLIST_SWITCH_BLOCK_END` | `$3CF8` |

Parameter slice convention: little-endian byte order throughout (low
byte first), matching how BASIC programs typically `POKE` a 16-bit
value. No cross-target convention beyond that — each target interprets
the slice's contents entirely on its own terms, and the slice carries no
persistence guarantee across calls.

## New: `LLIST(0)` — ZX0 decompression

| | |
|---|---|
| `ZX0_DECOMPRESS` | `$3BFA` |
| `DZX0_STANDARD` | `$3C08`, 68 bytes |
| Source | Einar Saukas & Urusergi, confirmed byte-identical to upstream
  (see Decisions Entry 3) |
| Slice layout | `slice[0..1]` = source address (LE), `slice[2..3]` =
  destination address (LE) |
| Returns | `BC=0` always (ZX0's format carries no length/error signal) |

## New: `LLIST(1)` — GOTO/GOSUB/RETURN/`NEXT` line-address cache

| | |
|---|---|
| `CACHE_INSERT` | `$3C4C` |
| `CACHE_SLOT_ADDR` (shared helper) | `$3C99` |
| `CACHE_CLEAR` | `$3CA3` |
| `CACHE_LOOKUP_OR_REAL` | `$3CB3` |
| `CACHE_MISS` | `$3CF2` (was `$3CED` before the v1.5 line-0 guard; shifted
  +5 bytes, see below) |
| `CACHE_BASE` | `$5B80`, 128 bytes (tail of `PRBUFF`) |
| `CACHE_SLOTS` | 32 |
| Slot layout | 4 bytes: `line_hi, line_lo, addr_hi, addr_lo`
  (big-endian, matching `LINE-ADDR`'s own `H`=high/`L`=low convention —
  **not** the same byte order as the parameter slice, which is
  little-endian; these are two independent, intentionally different
  conventions, don't assume one from the other) |
| Empty slot marker | line-number field `= $0000` |
| `LINE-NEW` interception | `$1B9E`, `CALL CACHE_LOOKUP_OR_REAL`
  (replaces `CALL LINE-ADDR` in place, same 3-byte instruction size) |
| Total mechanism size | 172 bytes (v1.3) → 177 bytes (v1.5, +5 for the
  line-0 guard below) |
| Free space remaining after integration | 3 bytes, at
  `LLIST_SWITCH_BLOCK_END = $3CFD` through `$3D00` (was 8 bytes at
  `$3CF8` before v1.5's fix) |

### v1.5: line-0 / empty-slot sentinel collision fix

`CACHE_LOOKUP_OR_REAL`'s empty-slot marker (line-number field `$0000`)
collided with a genuine lookup for target line `0` — which is exactly
what a bare `RUN` (no line argument) asks the interpreter to locate,
via `RUN`'s `USE-ZERO` default and `LINE-NEW`'s call into this routine,
*before* the program's own first statement executes. This caused a
false "hit" against any empty slot's stale, uninitialized address
bytes on every `RUN`. See `DECISIONS_4_CACHE_FIX.md` for the full
narrative and reproduction.

Fix: 5 bytes prepended to `CACHE_LOOKUP_OR_REAL`'s entry —

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

`CACHE_LOOKUP_OR_REAL`'s own entry address is unchanged (`$3CB3`); only
`CACHE_MISS` and `LLIST_SWITCH_BLOCK_END` shifted, both self-correcting
via symbolic `ORG` (no other addresses affected — confirmed by full
binary diff against the pre-fix build, contained entirely to
`$3CB4`–`$3CFC`, character-set table at `$3D00`+ byte-identical).

### `CACHE_INSERT` contract

Entry: `LLIST_SLICE[0]` = slot (0–31), `LLIST_SLICE[1..2]` = target line
number (LE). Exit: `BC` = resolved line address on success, `0` on any
failure (bad slot, line not found) or on a `slot=0,line=0` clear-all
request (which also returns `0` — no distinct success value for that
case, deliberately).

### `CACHE_LOOKUP_OR_REAL` contract

Entry: `HL` = target line number (`H`=high, `L`=low — same contract as
`LINE-ADDR` itself). Exit: `HL` = address, `Z` flag = exact match. Fully
substitutable for a direct `CALL LINE-ADDR` — a cache miss tail-jumps
(`JP`, not `CALL`) into the real, unmodified `LINE-ADDR`, so its `RET`
returns directly to whatever called `CACHE_LOOKUP_OR_REAL`. `LINE-ADDR`'s
`DE` output is not reproduced on a cache hit — confirmed dead by the
time it would matter downstream (see Decisions Entry 4).

## Space accounting

| | Bytes |
|---|---:|
| Free block budget at `NEW_CIRCLE_CORE_END` through `$3D00` | 558 |
| `NEW_EXP_CORE_V2` + `TO_POWER_V2` | 222 |
| `S_LLIST_SWITCH` + `LLIST_TABLE` + `ZX0_DECOMPRESS` +
  `DZX0_STANDARD` | 154 |
| `CACHE_INSERT` + `CACHE_SLOT_ADDR` + `CACHE_CLEAR` +
  `CACHE_LOOKUP_OR_REAL` (v1.5, incl. line-0 guard) | 177 |
| **Remaining free** | **3** |

Effectively no room left in this block for anything further without
either a real space-reclamation pass (matching the discipline of
Checkpoint Beta's own repacking effort) or deferring more work the way
flood fill already was.
