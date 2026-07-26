# KEYWORD_ADDR — a new `ADDR(var)` BASIC function for the Sinclair 48K ROM

This document is a complete, self-contained specification for adding an
`ADDR(variable)` function to a **standard, unmodified 48K Spectrum ROM** —
no dependency on any other project or hardware. It gives a program the
address of a variable's actual data in memory, so it can be manipulated
directly (`PEEK`/`POKE`, `OUT`, or any other memory-aware code) without
having to guess or hand-compute where the interpreter put it.

Everything below was empirically verified — assembled, patched into a real
ROM image, and run end-to-end in a Z80 emulator — not just reasoned about.
Addresses quoted are for the standard 48K ROM; if you're working from a
different ROM revision, re-verify each one against your own build rather
than trusting the numbers here (see "Verification method").

---

## 1. What it is

`ADDR` is used like a built-in function, inside an expression:

```basic
LET A = ADDR(B)
```

It always returns a plain number (0-65535): the address of the **first
byte of the variable's actual data** — for a numeric variable, that's the
first byte of its 5-byte floating-point value; for an array, either a
specific element's float or the array's own base address.

### Full syntax

| Syntax | Result |
|---|---|
| `ADDR(x)` — simple numeric variable, e.g. `ADDR(B)` | address of the 5-byte float |
| `ADDR(name)` — long-named numeric variable, e.g. `ADDR(COUNT)` | address of the 5-byte float |
| `ADDR(arr(i,j,...))` — specific array element, e.g. `ADDR(Q(2,3))` | address of that element's 5-byte float |
| `ADDR(arr(1,1,...,1))` — all subscripts = 1, matching the array's real dimensionality | the array's own base address (first byte of its flat data) |

```basic
10 LET B = 42
20 DIM Q(2,2,2,2)
30 PRINT ADDR(B)             : REM address of B's float
40 PRINT ADDR(Q(1,2,1,2))    : REM address of that one element
50 PRINT ADDR(Q(1,1,1,1))    : REM address of the array's own base
```

Array subscript count must exactly match how the array was `DIM`'d — same
rule as any normal array reference. A bare array name with no subscript,
`ADDR(Q)`, does **not** give the array base — with no `(...)`, the ROM's
own variable lookup treats it as a request for a *simple* variable named
`Q`, a different variable entirely from the array `Q()`. Use
`ADDR(Q(1,1,...,1))` for the base.

### Errors

| Input | Result |
|---|---|
| `ADDR(Z)`, `Z` never assigned/`DIM`'d | `REPORT-2` — "Variable not found" |
| `ADDR B` (missing parentheses) | `REPORT-C` — "Nonsense in BASIC" |

### Not implemented — errors cleanly rather than misbehaving

`ADDR(a$)` and `ADDR(a$(i))` (string scalars and string array elements)
currently raise `REPORT-C` via a deliberate placeholder in the code, not a
crash or silent wrong answer. String addressing needs its own separate
verification pass (different storage layout — length-prefixed, variable
size) before it could be added; treat it as a known gap, not an oversight.

### Untested edge case

`ADDR(i)` where `i` is a live `FOR...NEXT` control variable has never been
explicitly tested. Reasoning from the ROM's own storage-format table
suggests it would likely resolve correctly (a `FOR...NEXT` record starts
with the same 5-byte float layout as a simple scalar), but this hasn't
been traced or verified — don't assume it works.

---

## 2. The addressing formula

This is the actual logic, and the two things that make it correct rather
than approximately-correct:

```
CALL LOOK-VARS          ; resolves a variable name from source text
JP   C, REPORT-2        ; not found -> "Variable not found"
CALL Z, STK-VAR         ; only fires for array/subscript syntax -- resolves
                         ; the subscript list to a specific element's address
INC  HL                 ; the one correction that fixes BOTH paths (see below)
```

**Why `INC HL` fixes both paths, not just one.** `LOOK-VARS` (for a plain
scalar) returns `HL` pointing at the *last byte of the variable's name* —
for a single-letter variable that's trivially the only byte; for a
long name, `LOOK-VARS`'s own matching loop walks the whole name and lands
on its last byte either way. The data starts one byte later — hence
`INC HL`. Separately, `STK-VAR` (for an array element) returns `HL` one
byte *short* of the true element address — confirmed by tracing its
`SV-MULT` loop directly and cross-checking against the array's own
self-declared header length. The *same* single `INC HL`, applied
unconditionally after either path, corrects both. This isn't a
coincidence: it's copied directly from how the ROM's own `S-LETTER`
routine (which resolves any plain variable reference inside an expression)
does exactly this, for exactly this reason.

**Why `ADDR(arr(1,1,...,1))` gives the array's base, with no special
case.** `STK-VAR`'s own subscript arithmetic computes, for each dimension,
`(subscript - 1) * stride`, accumulated across all dimensions (confirmed
by reading its `SV-MULT` loop, which explicitly comments `"adjust returned
result from 1-x to 0-x"`). If every subscript is `1`, every term is zero
— unconditionally, regardless of how many dimensions there are or how big
they are. So `(1,1,...,1)` is *mathematically forced* to compute offset
zero. No separate "give me the whole array" syntax was needed; this
formula already does it for free.

---

## 3. The routine (verified, exact source)

This is the actual assembled and tested code (`pasmo` syntax). It needs to
sit somewhere in ROM with enough contiguous free space (52 bytes total,
contiguous, no padding) — see section 5 for how to find that space on a
ROM that doesn't already have some set aside.

```asm
; ---- fixed addresses this depends on (standard 48K ROM) ----
LOOK_VARS   EQU $28B2
STK_VAR     EQU $2996
STACK_BC    EQU $2D2B
S_ALPHNUM   EQU $2684   ; the ROM's normal "not a known operator/function"
                         ; fallback inside expression scanning
S_NUMERIC   EQU $26C3   ; shared continuation that finishes any
                         ; numeric-result expression term
REPORT_C    EQU $1C8A   ; "Nonsense in BASIC"
REPORT_2    EQU $1C2E   ; "Variable not found"

; ---- the token you're reclaiming (pick one -- see section 6) ----
ADDR_TOKEN  EQU $xx     ; <-- set this to your chosen keyword's token byte

;; ADDR-DISPATCH-STUB
; Hooked in place of the ROM's own "not found in primary function table"
; fallback jump (see section 4). A still holds the character that failed
; to match, untouched by CP, so falling through to the real S-ALPHNUM for
; anything that isn't our token is exactly as safe as the original code.
ADDR_DISPATCH:
        CP      ADDR_TOKEN
        JP      NZ,S_ALPHNUM
        JP      S_ADDR

;; S-ADDR
; Entered with CH_ADD pointing AT the token itself (the ROM's own
; character-lookahead never consumes it), matching the convention every
; other expression-scanning function uses.
S_ADDR:
        RST     $20             ; consume the token
        CP      $28             ; expect '('
        JP      NZ,REPORT_C
        RST     $20             ; consume '(' -> CH_ADD at the variable ref

        CALL    LOOK_VARS
        JP      C,REPORT_2      ; "Variable not found"
        CALL    Z,STK_VAR       ; only fires for array/subscript syntax

        LD      A,($5C3B)       ; FLAGS
        CP      $C0
        JR      C,ADDR_STRING   ; string result -- not implemented, see below
        INC     HL              ; the single correction, see section 2
        LD      B,H
        LD      C,L
        CALL    STACK_BC        ; stacks HL as a numeric result, same
                                 ; helper USR/PEEK already use
        JR      ADDR_DONE

ADDR_STRING:
        ; Deliberate placeholder -- string addressing isn't implemented.
        ; Errors cleanly rather than returning a wrong or garbage address.
        JP      REPORT_C

ADDR_DONE:
        RST     $18             ; GET-CHAR
        CP      $29             ; expect ')'
        JP      NZ,REPORT_C
        RST     $20             ; consume ')'
        JP      S_NUMERIC       ; rejoin the ROM's own shared continuation
```

**Size: 52 bytes total, contiguous, no padding required.** Confirmed by
assembling this block standalone.

`JP` is used throughout rather than `JR` because the block will typically
sit far away (in address terms) from the fixed ROM addresses it calls —
`JR`'s range (±127 bytes) usually isn't enough. If your placement happens
to be close enough, `JR` would work for the branches that don't leave the
block, but there's no benefit to relying on that.

---

## 4. Where it plugs into the ROM's own dispatch

The ROM has a small table (`scan-func`, at `$2596` on the standard 48K
ROM) used while scanning an expression, to recognize a handful of special
characters and functions (`RND`, `PI`, `INKEY$`, `SCREEN$`, `ATTR`,
`POINT`, brackets, operators). If a character doesn't match anything in
that table, control falls through to a fixed instruction:

```asm
;; S-LOOP-1                       (at $24FF on the standard 48K ROM)
        LD      C,A
        LD      HL,$2596          ; scan-func
        CALL    $16DC             ; INDEXER
        LD      A,C
        JP      NC,$2684          ; <-- this is the instruction to edit
        ...
```

**This `JP NC,nn` is the entire hook point.** Change its 2-byte operand
from `$2684` (the real `S-ALPHNUM`) to your new `ADDR_DISPATCH` stub's
address. That's a `JP cc,nn` instruction either way — **same 3-byte shape,
zero size change** — so nothing else in the ROM needs to move, and no
other `ORG` anchor anywhere in the ROM is disturbed by this edit alone.

**Why the table itself can't just be extended, and doesn't need to be.**
The `scan-func` table has no slack — its own end marker is immediately
followed by real code, so growing it in place would shift everything
after it. Worse, its entries use a single **unsigned, forward-only** byte
offset (confirmed by decoding the real table and reproducing its own
addressing arithmetic), so even relocating a copy of the whole table
somewhere else wouldn't work — a copy living at a higher address could
never point backward to the original entries' targets. None of this
matters here, because the fallback-hook approach above doesn't touch the
table at all.

Bracket the edit defensively even though it's a zero-size change:

```asm
        ORG $2507          ; wherever the JP NC instruction actually starts
                            ; on YOUR assembled ROM -- verify, don't assume
        JP      NC,ADDR_DISPATCH
        ORG $250A           ; wherever the NEXT instruction must resume --
                            ; again, verify against your own build
```

`pasmo` (and most Z80 assemblers) will **silently** seek backward and
overwrite already-emitted bytes if an `ORG` is ever wrong — no error, no
warning, exit code 0. The anchors above don't prevent that by themselves;
what actually catches it is comparing a full binary diff of your patched
ROM against an unpatched baseline afterward (see section 7) and confirming
only the bytes you meant to change actually changed.

---

## 5. Placement — the pristine ROM already has room, at `$386E`

**Correction to an earlier draft of this document, which wrongly claimed a
stock ROM has zero free bytes.** That was checked incorrectly. Verified
properly this time, directly against a freshly assembled, completely
unmodified 48K ROM: there is a genuine, pre-existing block of `$FF`
padding from **`$386E` to `$3CFF`** — **1170 bytes**, sitting between the
end of the ROM's real code and the fixed start of the character-set
bitmap table at `$3D00`. This is inherent to the standard Sinclair ROM
itself, not something you need to create. 52 bytes fits with enormous
headroom to spare — no need to reclaim a dead stub, replace a routine, or
extend the ROM at all.

```asm
        ORG $386E          ; the genuine start of free space on a standard,
                            ; unmodified 48K ROM -- confirmed directly
                            ; against the assembled binary, not assumed
ADDR_DISPATCH:
        ...
S_ADDR:
        ...
        ORG $386E+52       ; self-check: confirms the block really is
                            ; exactly 52 bytes, catches a silent overrun
                            ; if it isn't (see section 7)
```

Don't just trust `$386E` as a label or a number quoted here, either —
before relying on it, confirm it against your own assembled ROM the same
way: scan forward from a plausible starting point for a run of `$FF`
bytes ending exactly at `$3D00` (the character-set table's fixed start),
and confirm the byte immediately before your intended placement is really
part of that padding, not the tail end of real code. If you're working
from a ROM revision other than the standard 48K issue 3 ROM, re-verify
the exact boundary — it's extremely unlikely to have moved, but "extremely
unlikely" isn't "verified," and this project has been burned by exactly
that gap before.

If you ever need more than 1170 bytes for something else built on top of
this, the same three options as before remain available as a fallback:
reclaiming a dead stub's body (`CAT`/`FORMAT`/`MOVE`/`ERASE`, a few bytes
only), replacing an existing routine with something smaller, or extending
the ROM outright if your target format allows it.

---

## 6. Choosing which keyword to sacrifice

`ADDR` needs an existing token byte to attach to — reusing one keyword's
token rather than inventing a new one, because every byte value $A5-$FF is
already assigned and there's no free slot. The mechanism in section 4
works identically for *any* command-range token ($CE-$FF) — none of them
are usable inside an expression today (confirmed: none fall inside
`scan-func`'s table or the ROM's separate arithmetic-mapped range for
single-argument functions), so hooking any one of them mid-expression
costs nothing structurally. The choice is about tradeoffs, not mechanics.

**Two things change when you pick a token — both fully reversible and
independent of each other:**

1. **The mid-expression meaning** (via the hook in section 4) — this is
   additive only. The keyword's original statement behavior (typing it at
   the start of a line) is **never touched or disabled** by this patch,
   regardless of which token you pick — proven by full-binary-diff twice
   over, for two structurally different keywords (a dead stub and a fully
   alive one).
2. **The displayed text** (optional, see section 8) — purely cosmetic,
   independent of #1, and doesn't need to be done at all if having the new
   function display under its old keyword's name is acceptable.

**What actually matters is: does anything external care about this
keyword's original statement meaning still being easy to invoke and
recognize under its original name?** Two real considerations, from
research into what actually depends on these commands:

- **`CAT`, `FORMAT`, `MOVE`, `ERASE`, `OPEN #`, `SAVE`, `LOAD`, `VERIFY`,
  `MERGE`, `CLS`, `CLEAR`** are specifically watched by genuine Interface 1
  hardware and its modern clones/replacements (still actively made) — they
  intercept via the error-raising mechanism these commands use, then do
  their own independent re-check of the statement to recognize their own
  keywords. Since the statement path is never touched, this still works
  functionally either way — but a renamed display (section 8) on any of
  these would be genuinely misleading to a user who also has such hardware
  attached, especially `SAVE`/`LOAD`/`CLS`/`CLEAR`, which are also just
  extremely commonly typed regardless of hardware.
- **`LPRINT`, `LLIST`, `COPY`** are printer-only — no drive/storage
  hardware cares about them. Real printer hardware for the Spectrum was
  rare even at the time and is rarer now, making these the lowest-risk
  category if renaming the display matters to you.
- Fundamental language keywords (`LET`, `IF`, `PRINT`, `FOR`, `GO TO`,
  etc.) have no known hardware dependency at all, but are typed and read
  constantly — a poor choice purely on disruption grounds, independent of
  any compatibility question.

**Recommendation, absent a specific reason to choose otherwise:**
`LPRINT`, `LLIST`, or `COPY` — all in the lowest-disruption category, none
flagged by any hardware research, and rare enough in ordinary use that a
renamed display (if you choose to rename at all) won't be confusing.

---

## 7. Verification method (don't skip this)

Whatever token and placement you choose, verify with a **full binary diff
against your own unpatched baseline** — this is the only reliable way to
catch a silently-wrong `ORG` (assemblers do not warn about this; confirmed
directly: exit code 0, zero output, on both a backward-seeking `ORG` and a
genuine overrun into the next anchored block).

1. Assemble your ROM source, unmodified, to get a baseline binary.
2. Apply the patch (sections 4-6), assemble again.
3. Diff the two binaries byte-for-byte. Categorize every differing byte —
   you should be able to explain each one (the dispatch operand, the new
   routine's bytes, and nothing else, unless you also did the optional
   rename in section 8). Zero unexplained differences is the bar, not "it
   assembled without errors."
4. Beyond the diff, actually **run** the patched ROM in a Z80 emulator and
   exercise `ADDR` for real — a scalar, an array element, the array-base
   trick, and both error paths — rather than trusting the diff alone to
   prove correctness of behavior, only of byte-placement.

If you don't already have Z80-emulation tooling for this kind of
verification, `z80js` (npm) plus any Z80 cross-assembler (`pasmo` is
freely available and was used throughout this project) is enough to
reproduce every check described here.

---

## 8. Optional: renaming the display text

Not required — `ADDR` works identically whether or not you also make
`LIST` show `ADDR` instead of your chosen keyword's original name. If you
do want the display to match:

**The keyword-spelling table (`TKN-TABLE`, `$0095` on the standard 48K
ROM) is walked purely by counting word-end markers, not by fixed
per-entry byte offsets** — confirmed by reading the actual display routine
directly. This means a replacement word's *length* doesn't need to match
the original at all. Separately, **keyboard entry never consults this
table** — a keyword is typed via a direct keycode-to-token-value lookup or
arithmetic mapping, confirmed by reading that code directly too. So
renaming the display text has zero effect on how the keyword is typed.

Each entry in the table is stored as plain ASCII, with the **last**
letter's high bit set to mark the word's end, e.g.:
```asm
        DEFM    "LPRIN"
        DEFB    'T'+$80
```
Replace with your new 4-letter name:
```asm
        DEFM    "ADD"
        DEFB    'R'+$80
```

If the replacement is a **different length** than the original (as above:
`ADDR` is 4 letters, `LPRINT` is 6), every entry *after* this one in the
table shifts by the size difference — harmless to the table's own
function (it doesn't care about individual entry positions), but it will
shift the address of everything that follows the table in ROM unless
compensated. Add filler bytes (`DEFB $FF`) totaling exactly the size
difference, positioned after the table's own last real entry (`COPY`, the
highest-valued token, on the standard 48K ROM) — this is provably inert,
since nothing ever walks past `COPY`'s own entry. Bracket both ends:

```asm
        ORG $xxxx          ; the entry's real start address on your build
        DEFM    "ADD"
        DEFB    'R'+$80
        ; ... rest of the table, unedited ...
        DEFB    $FF, $FF   ; exactly (old length - new length) filler bytes
        ORG $xxxx           ; wherever the table's real end / next section
                            ; must resume -- verify against your own build,
                            ; and confirm via the diff method in section 7
                            ; that it lands exactly where the unpatched
                            ; ROM had it
```

Verify the rename actually works by running the ROM's own display routine
for real (not just diffing the table) — feed it the token number
(`token_value - $A5`, since the table starts at `RND`) and confirm it
prints your new name, and separately confirm the *next* entry in the
table (whatever keyword originally followed yours) still prints correctly
too, proving the compensation didn't corrupt anything downstream.
