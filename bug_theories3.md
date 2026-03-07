# DMTXJC Endless Loop — Definitive Code Trace Analysis

> This document supersedes `bug_theories2.md` on the loop question.  
> Every claim is verified directly against the assembled listing
> (`BOB ZW02 DMTXJC 468 06 MAR 26 12.04.20 PM.pdf`).  
> Statement numbers, offsets, and instruction sequences are cited for every step.  
> No I/O activity occurs during the loop. No WAIT is issued. No ECBs are posted.

---

## Table of Contents

1. [The Exact Loop Path — Step by Step](#1-the-exact-loop-path--step-by-step)
2. [Step 1 — CMDECK and LOOPCT](#2-step-1--cmdeck-and-loopct-csect-dmtxjc1)
3. [Step 2 — ECB Dispatch Checks](#3-step-2--ecb-dispatch-checks-csect-dmtxjc1)
4. [Step 3 — ALLCHK: CLC Not Equal → Branch to $START](#4-step-3--allchk-clc-not-equal--branch-to-start-csect-dmtxjc1)
5. [Step 4 — $CCOMM1 → MREPUT](#5-step-4--ccomm1--mreput-csects-dmtxjca--dmtxjc1)
6. [Step 5 — $TPREPUT → OREENT → OGETBUF → ORETURN](#6-step-5--tpreput--oreent--ogetbuf--oreturn-csect-dmtxjc1)
7. [Step 6 — Back in MREPUT: BZ → $CCOMM1+4](#7-step-6--back-in-mreput-bz--ccomm14)
8. [Step 7 — NOPs Fall Through → B CMDECK](#8-step-7--nops-fall-through--b-cmdeck-csect-dmtxjca)
9. [Why OGETBUF Refuses the Last Buffer](#9-why-ogetbuf-refuses-the-last-buffer)
10. [Root Cause: How $BUFPOOL Is Depleted in Bidirectional Mode](#10-root-cause-how-bufpool-is-depleted-in-bidirectional-mode)
11. [Why the Loop Is Inescapable Once Started](#11-why-the-loop-is-inescapable-once-started)
12. [LOOPCT and the Debug Abend](#12-loopct-and-the-debug-abend)
13. [Summary of All Code Locations Involved](#13-summary-of-all-code-locations-involved)
14. [Recommended Fixes](#14-recommended-fixes)

---

## 1. The Exact Loop Path — Step by Step

The loop is a **pure, uninterrupted code loop**. No I/O operations are initiated, no
MVS WAIT is executed, and no ECBs are posted during any iteration. The path visits
code in two CSECTs on every cycle:

```
CMDECK (DMTXJC1, 000790)
  └─ ECB checks: CMDECB, MSGECB, RDEVSYNC, MASTERSW/JOB,
                 MASTERSW/SYSOUT, ADAECB — all zero, branches taken (no action)
       └─ ALLCHK (DMTXJC1, 0008C2)
            CLC $START,=$ALLOFF — NOT equal ($CCOMM1 is OPEN)
            BNE $START → branch into commutator table (DMTXJCA, 04E4C)
              └─ $CCOMM1 (DMTXJCA, 04E4C) — BC 15,MREPUT (gate OPEN)
                   → MREPUT (DMTXJC1, 0009BE)
                        └─ BAL R14,$TPREPUT (DMTXJC1, 0023AA)
                               └─ B OREENT (DMTXJC1, 0022AE)
                                    └─ L R6,OBUFPTR → LTR → BE OGETBUF
                                         └─ OGETBUF (DMTXJC1, 00230C)
                                              tests 1, 2 fail
                                              test 3: CLC 0(4,R6),=F'0' → EQUAL
                                              BE ORETURN (00231A taken)
                                                └─ ORETURN (DMTXJC1, 0022F4)
                                                     L R8,R6,R5,R14 → BR R14
                                                         [returns with CC=0]
                                    ← $TPREPUT returns CC=0
                        └─ BZ $CCOMM1+4 (0009C6) — BZ taken (CC=0)
                               → $CCOMM1+4 (DMTXJCA, 04E50)
                                    NOP (BC 0,$TPGET)   — falls through
                                    NOP (BC 0,$PCOM1)   — falls through
                                    NOP (BC 0,$RCOM1)   — falls through
                                    NOP (BC 0,$JCOM1)   — falls through
                                    NOP (BC 0,$WCOM1)   — falls through
                                    NOP (BC 0,CMDPROC)  — falls through
                                    NOP (BC 0,MSGPROC)  — falls through
                                    NOP (BC 0,$COMSUP)  — falls through
                                    NOP (BC 0,$INTRUPT) — falls through
                                         └─ B CMDECK ← hard branch at bottom
← repeat indefinitely
```

---

## 2. Step 1 — CMDECK and LOOPCT (CSECT DMTXJC1)

```
Offset  Object Code      Stmt   Source
000790  5880 95CC        1091   L   R8,LOOPCT       load debug counter
000794  4180 8001        1092   LA  R8,1(,R8)        increment by 1
000798  5080 95CC        1093   ST  R8,LOOPCT        store back
00079C  5980 AC2C        1094   C   R8,=F'100'       threshold check
0007A0  4740 C350        1095   BL  SKIPPY           < 100: continue
```

LOOPCT (at offset 045CC in DMTXJCA) is a **debug counter** added by the user to detect
the loop. It increments by one each time CMDECK is entered. If it reaches 100 the code
issues a 30-second STIMER WAIT and then abends (`DC AL4(0)` = S0C1 at offset 0007BA,
stmt 1104). LOOPCT is cleared **only** at two points:

| Location | Stmt | Offset | When |
|---|---|---|---|
| ISIO (init) | 824 | 00050A | Link startup |
| $SIO | 5313 | 002DBE | Immediately after every successful `BALR R14,R15` (EXCP) |

Because the loop **never issues any I/O**, $SIO is never called, LOOPCT is never
cleared, and after 100 iterations the abend fires.

---

## 3. Step 2 — ECB Dispatch Checks (CSECT DMTXJC1)

After SKIPPY (000C8), the code checks each ECB/flag in turn. In the loop state,
**every single check takes the "not posted / not active" branch**:

| Label | Offset | Instruction | Condition in loop | Branch taken |
|---|---|---|---|---|
| SKIPPY | 0007C8 | TM CMDECB,X'40' | CMDECB=0 | BZ MSGECK |
| MSGECK | 0007DA | TM MSGECB,X'40' | MSGECB=0 | BZ RDRECBCK |
| RDRECBCK | 0007EC | TM RDEVSYNC,X'40' | RDEVSYNC=0 | BNO JOBECBCK |
| JOBECBCK | 0007FE | TM MASTERSW,JOB | JOB bit=0 | BNO PRTECBCK |
| PRTECBCK | 000826 | TM MASTERSW,SYSOUT | SYSOUT bit=0 | BNO ADAECBCK |
| ADAECBCK | 00084E | TM ADAECB,X'40' | ADAECB=0 | BNO ALLCHK |

All ECBs are zero. No reader, no printer, no CTC adapter I/O has completed. The code
falls straight through to ALLCHK.

---

## 4. Step 3 — ALLCHK: CLC Not Equal → Branch to $START (CSECT DMTXJC1)

```
Offset   Object Code                    Stmt   Source
0008C2   D527 9E4C C470   04E4C 008E8   1181   CLC $START($COMEND-$START),$ALLOFF
0008C8   4770 9E4C         04E4C        1182   BNE $START    IF NOT EQUAL → dispatch
```

`$ALLOFF` (at DMTXJC1 offset 0008E8) is the **all-NOP template** — a fixed copy of what
the commutator table looks like when every gate is closed:

```
0008E8  4700 C4B4  NOP $CONTROL   ← $CCOMM1 template entry
0008EC  4700 BF4E  NOP $TPGET
0008F0  4700 91D6  NOP $PCOM1
0008F4  4700 9256  NOP $RCOM1
0008F8  4700 939E  NOP $JCOM1
0008FC  4700 90B6  NOP $WCOM1
000900  4700 B704  NOP CMDPROC
000904  4700 BB60  NOP MSGPROC
000908  4700 A654  NOP $COMSUP
00090C  4700 A2A6  NOP $INTRUPT
```

The **live commutator table** lives in DMTXJCA at `$START` (= `$CCOMM1`, offset 04E4C).
Gates are opened by changing the mask byte of a `BC 0,addr` NOP into `BC 15,addr`
(opcode stays `47`, second byte changes from `00` to `F0`). Gates are closed by
reverting the mask byte to `00`.

The CLC at ALLCHK compares 40 bytes (`$COMEND-$START` = 0x28 bytes, D527) of the live
table against the all-NOP template. **If they are NOT equal, at least one gate is open.**

In the loop: the CLC is NOT equal because `$CCOMM1`'s mask byte is `F0` (OPEN),
not `00`. Therefore `BNE $START` is taken, branching into the live commutator table
in DMTXJCA at 04E4C.

---

## 5. Step 4 — $CCOMM1 → MREPUT (CSECTs DMTXJCA + DMTXJC1)

At `$START` (= `$CCOMM1` in DMTXJCA at 04E4C), the first entry is a **OPEN branch**
instruction:

```
04E4C  47F0 ????   BC 15,MREPUT    ← mask=F0 (OPEN), branches to MREPUT
```

Normally this entry would read `47F0 C4B4` (BC 15,$CONTROL) when the control-record
processor is open. It was changed to point to `MREPUT` by:

```
Offset   Object Code           Stmt   Source
0009B0   D201 9E4E C576        1334   MVC $CCOMM1+2(2),MREPUTA   SET COMUTATOR RE-ENTRY
0009B6   5080 9648             1335   ST  R8,MTANK               save tank addr
0009BA   47F0 9054             1336   B   CCTRTN                  exit to commutator
```

This code is in `MPUT` (stmt 1331, offset 0009A8). MPUT is called when the code needs
to **send a control-record response** (e.g., a START FUNCTION REQUEST / MC1, or signon
response) and the first `$TPPUT` attempt fails (returns CC=0). When $TPPUT fails:

1. `$CCOMM1+2` is patched from `$CONTROL` to `MREPUTA` (= address of MREPUT)
2. The tank address is saved in `MTANK`
3. The code exits to the commutator (`B CCTRTN`)

On the next CMDECK pass, the open `$CCOMM1` (now pointing to MREPUT) causes the
dispatch above, entering `MREPUT` at 0009BE:

```
Offset   Object Code       Stmt   Source
0009BE   5880 9648         1339   L   R8,MTANK          restore tank addr
0009C2   45E0 BF32 023AA   1340   BAL R14,$TPREPUT      TRY IT
0009C6   4780 9E50  04E50  1341   BZ  $CCOMM1+4         CYCLE IF STILL NOT ACCEPTED
```

---

## 6. Step 5 — $TPREPUT → OREENT → OGETBUF → ORETURN (CSECT DMTXJC1)

### $TPREPUT (offset 0023AA, stmt 4230)

```
0023AA  5080 9A58    4231   ST  R8,OINADD      save input tank addr
0023AE  50E0 9A54    4232   ST  R14,OSAVR14    save return address
0023B2  5050 9A50    4233   ST  R5,OSAVR5
0023B6  5060 9A4C    4234   ST  R6,OSAVR6
0023BA  41F0 0001    4235   LA  R15,1          constant
0023BE  4850 8000    4236   LH  R5,TANKCHN     compressed count from tank header
0023C2  47F0 BE36 022AE  4237  B  OREENT       ENTER FLOW
```

### OREENT (offset 0022AE, stmt 4132)

```
0022AE  5860 9A64    4133   L   R6,OBUFPTR     get addr of active output buffer
0022B2  1266         4134   LTR R6,R6          is it zero?
0022B4  4780 BE94 0230C  4135  BE  OGETBUF      YES — no active buffer, get one
```

`OBUFPTR` (at DMTXJCA offset 04A64) holds the current write position within the
active output TP buffer. It is set to 0 when the last output buffer was flushed to
`$OUTBUF` (in OBUFFULL, stmt 4204: `SR R6,R6 → ST R6,OBUFPTR`). In the loop state,
`OBUFPTR = 0`: there is no active output buffer. The `BE OGETBUF` branch is taken.

### OGETBUF (offset 00230C, stmt 4163)

Three sequential tests:

**Test 1** — Stop-buffering flag (stmt 4164–4165):
```
00230C  9180 9D0C    4164   TM  BUFSYNSW,$TPPNONE     stop buffering?
002310  4710 BEE0 02358  4165  BO  OGETBUF1             yes → set CC=0, return
```
Flag NOT set. Fall through.

**Test 2 (first CLC)** — Pool completely empty (stmt 4166–4167):
```
002314  D503 9634 AC28  4166  CLC $BUFPOOL,=F'0'      is pool empty?
00231A  4780 BE7C 022F4  4167  BE  ORETURN              yes → return CC=0
```
Pool is NOT empty — there is one buffer. Fall through.

**Test 3 (second CLC — the one the user identified)** — Only one buffer left (stmts 4168–4170):
```
00231E  5860 9634  4168   L   R6,$BUFPOOL              get first free buffer addr
002322  D503 6000 AC28  4169  CLC 0(4,R6),=F'0'       is its forward chain = 0?
002328  4780 BE7C 022F4  4170  BE  ORETURN  YEP...BETTER NOT USE IT @VA03301
```

`0(R6)` is the **forward chain pointer** of the first (and only) buffer in `$BUFPOOL`.
If this pointer is zero, the buffer has no successor — it is the **last remaining buffer**
in the pool. The code deliberately refuses to take it ("BETTER NOT USE IT").
`BE ORETURN` is taken. **Condition code = 0 (equal).**

### ORETURN (offset 0022F4, stmt 4154)

```
0022F4  5880 9A58    4155   L   R8,OINADD      restore tank addr
0022F8  5860 9A4C    4156   L   R6,OSAVR6
0022FC  5850 9A50    4157   L   R5,OSAVR5
002300  58E0 9A54    4158   L   R14,OSAVR14    restore return addr (= 0009C6)
002304  07FE         4159   BR  R14            return to MREPUT
```

ORETURN restores registers and returns to the instruction after the BAL in MREPUT
(offset 0009C6). **The condition code is still 0 (equal), set by the CLC at 002322.**

---

## 7. Step 6 — Back in MREPUT: BZ → $CCOMM1+4

Back at MREPUT offset 0009C6:

```
0009C6  4780 9E50   1341   BZ  $CCOMM1+4    CYCLE IF STILL NOT ACCEPTED
```

`BZ` = Branch if Zero condition code (CC=0). The CLC at 002322 left CC=0.
**BZ is taken**. Control jumps to `$CCOMM1+4` = DMTXJCA offset 04E50.

This is **not** returning through MEXIT (which would free the tank, reset `$CCOMM1+2`
back to `$CONTROL`, and close the gate). MREPUT's `BNZ` path (would go to MEXIT at
stmt 1342) is NOT taken. The gate stays open, pointing to MREPUT.

---

## 8. Step 7 — NOPs Fall Through → B CMDECK (CSECT DMTXJCA)

The live commutator table in DMTXJCA, starting at `$CCOMM1+4` (04E50):

```
04E50  4700 xxxx   BC 0,$TPGET     mask=00 — TRUE NOP — fall through
04E54  4700 xxxx   BC 0,$PCOM1     mask=00 — TRUE NOP — fall through
04E58  4700 xxxx   BC 0,$RCOM1     mask=00 — TRUE NOP — fall through
04E5C  4700 xxxx   BC 0,$JCOM1     mask=00 — TRUE NOP — fall through
04E60  4700 xxxx   BC 0,$WCOM1     mask=00 — TRUE NOP — fall through
04E64  4700 xxxx   BC 0,CMDPROC    mask=00 — TRUE NOP — fall through
04E68  4700 xxxx   BC 0,MSGPROC    mask=00 — TRUE NOP — fall through
04E6C  4700 xxxx   BC 0,$COMSUP    mask=00 — TRUE NOP — fall through
04E70  4700 xxxx   BC 0,$INTRUPT   mask=00 — TRUE NOP — fall through
04E74  47F0 C318   B CMDECK        hard branch back to 000790
```

Every entry from `$CCOMM1+4` to the end of the table has its mask byte = `00`,
making each a TRUE NOP (BC 0,addr branches on no condition code, i.e., never).
The code falls completely through to the unconditional `B CMDECK` at the bottom.

**One loop iteration is complete. Repeat from Step 1.**

---

## 9. Why OGETBUF Refuses the Last Buffer

The check at stmt 4169-4170 (offsets 002322–002328) is a **deliberate deadlock-
prevention guard** from the original BSC code:

```asm
002322  CLC 0(4,R6),=F'0'   ONLY ONE LEFT???? @VA03301
002328  BE  ORETURN          YEP...BETTER NOT USE IT @VA03301
```

The comment `@VA03301` traces to an old RSCS VM/370 APAR fix. The rationale: if only
one buffer remains in the pool, do not use it for output. Keep it as a reserve so the
CTC adapter can always get at least one buffer for its next receive operation.

In the original BSC environment this was safe: BSC always had timer-driven
retransmissions and line-turnarounds that would eventually post an ECB, free a
tank-decompressed buffer back to `$BUFPOOL`, and allow the retry to succeed.

**In the CTC environment with bidirectional transfers, this assumption breaks down**
(see §10 and §11 below).

---

## 10. Root Cause: How $BUFPOOL Is Depleted in Bidirectional Mode

### Buffer lifecycle in a single CTC I/O cycle

Each buffer travels through this chain:

```
$BUFPOOL (free)
  → OBUFPTR (being filled by $TPPUT with compressed outgoing data)
  → $OUTBUF  (full buffer waiting to be transmitted)
  → CBUFFER  (CTC WRITE uses buffer; then CTC READ overwrites same buffer
               with incoming data from the remote side)
  → $INBUF   (received data, waiting for $TPGET to process)
  → TCT buffer chain (TCTBUFER, being decompressed by $TPGET into tanks)
  → $BUFPOOL (freed via GASSIGN→GIGNORIT when buffer is exhausted)
```

### Unidirectional (A→B only) — works fine

- A fills a buffer with data records.
- CTC WRITE sends A's data. CTC READ receives B's response.
- B's response is a short ACK (a few bytes).
- $TPGET quickly decompresses the short ACK → one tank → printer/control.
- The buffer is freed via GIGNORIT almost immediately.
- $BUFPOOL stays healthy (3–4 buffers available).

### Bidirectional (A→B and B→A simultaneously) — fails

- A fills a buffer with A's data records (large, full buffer).
- CTC WRITE sends A's data. CTC READ receives **B's full data block** (not just an ACK).
- B's full data block is a **large payload** requiring many tank decompression passes.
- $TPGET assigns the buffer to a TCT chain (e.g., PRINT TCT).
- The PRINT processor ($PRTN1 / $WRTN1) consumes tanks one by one and writes to
  local spool. **This takes real time.**
- While PRINT is consuming tanks, A's reader ($TPPUT) needs **another buffer from
  $BUFPOOL** to fill with the next outgoing block.
- A second CTC cycle runs: another received full buffer goes into another TCT chain.
- $BUFPOOL is now feeding two consumers simultaneously: the outgoing sender AND the
  incoming TCT chains.
- After a few cycles, $BUFPOOL reaches 1 buffer.

### The trigger: MPUT is called at this moment

A control record must be sent at the inter-file boundary. In NJE, after one file is
transmitted, the sender issues a **START FUNCTION REQUEST** (`MC1`, type 001) to
obtain permission to start the next file. This goes through `MPUT`:

```
000944  47F0 C4D8     1267   B   MPROCESS
...
000A10  000A10        MC1    -- process START FUNCTION REQUEST
000A54  47F0 C530     1404   B   MPUT        AND SEND IT
```

MPUT calls `$TPPUT`. $TPPUT calls OREENT → OGETBUF. With only 1 buffer in $BUFPOOL,
OGETBUF fires the "BETTER NOT USE IT" guard and returns CC=0. MPUT sets $CCOMM1 to
MREPUT. **The loop begins.**

---

## 11. Why the Loop Is Inescapable Once Started

Once the MREPUT loop is running, three conditions must be simultaneously satisfied
to exit:

**Condition A** — $BUFPOOL must grow above 1 (requires a buffer to return via
GIGNORIT/GSWITCH, which requires $TPGET to run and exhaust a TCT buffer).

**Condition B** — $TPGET must run (requires its commutator gate `$TPGETCM` to
be opened, which requires either ADAECB to fire from CTC I/O completion, or a
processor to call `$GETTNK` which calls `MVI $TPGETCM+1,OPEN` at stmt 4569).

**Condition C** — A processor must run (requires its commutator gate to be open,
which requires tanks in its TCT queue, which requires $TPGET to run — circular).

**The deadlock:**

```
$BUFPOOL=1 → $TPPUT fails → MREPUT loop → no new CTC I/O issued
           → ADAECB never posted → $TPGETCM never opened
           → $TPGET never runs → TCT buffers never freed
           → $BUFPOOL stays at 1 → loop continues
```

- `ADAECB` is not posted because no CTC I/O has been started.
- CTC I/O is not started because `$OUTBUF` is empty: the last buffer in it was
  consumed by the last CTC write, and no new output buffer was filled (MREPUT is
  looping and never successfully fills one).
- `$TPGETCM` stays closed: no ADAECB, no processor $GETTNK calls.
- TCT buffer chains remain populated with unconsumed records.
- $BUFPOOL stays at exactly 1.

The only escape would be an external ECB posting (a reader spool I/O completing,
a console command, a message, a delay timer expiring), but the user's trace confirms
all of those ECBs are also clear (RDEVSYNC, CMDECB, MSGECB all zero, MASTERSW=0).

---

## 12. LOOPCT and the Debug Abend

LOOPCT (DMTXJCA offset 045CC) increments by 1 at CMDECK on every pass.

During the loop, LOOPCT is NOT cleared (no I/O → $SIO never executes → no
`XC LOOPCT,LOOPCT` at stmt 5313).

After 100 iterations, CMDECK's `C R8,=F'100'` (stmt 1094) fails the `BL SKIPPY`
branch. The code executes:

```
0007A4  STM R14,R1,SAE1    save regs for dump
0007A8  STIMER WAIT,30 sec  give operator time to grab trace dataset
0007B6  LM  R14,R1,SAE1    restore regs
0007BA  DC  AL4(0)          → S0C1 abend (program check)
```

The 30-second STIMER WAIT is intentional: it gives the operator console time to act
(grab a trace dataset, etc.) before the abend kills the task. The NOP instructions in
the commutator table all have second byte `00`, confirming they are true NOPs and not
misassembled branches.

---

## 13. Summary of All Code Locations Involved

### CSECT DMTXJC1 (base address 000478 in ESD, length 002CE8)

| Label | Abs. Offset | Stmt | Role in loop |
|---|---|---|---|
| CMDECK | 000790 | 1090–1095 | Loop restart; increments LOOPCT |
| SKIPPY | 0007C8 | 1107–1108 | CMDECB check |
| MSGECK | 0007DA | 1112–1113 | MSGECB check |
| RDRECBCK | 0007EC | 1118–1119 | RDEVSYNC check |
| JOBECBCK | 0007FE | 1124–1125 | MASTERSW,JOB check |
| PRTECBCK | 000826 | 1135–1136 | MASTERSW,SYSOUT check |
| ADAECBCK | 00084E | 1146–1148 | ADAECB check → ALLCHK |
| ALLCHK | 0008C2 | 1181–1182 | CLC live table vs all-NOP template; BNE $START |
| $ALLOFF | 0008E8 | 1194–1203 | All-NOP reference template (10 entries × 4 bytes) |
| MPUT | 0009A8 | 1331–1336 | Called to put control record; on $TPPUT fail, sets $CCOMM1+2→MREPUTA |
| MREPUT | 0009BE | 1338–1341 | Retry loop entry; BAL $TPREPUT; BZ $CCOMM1+4 |
| $TPREPUT | 0023AA | 4230–4237 | Re-entry to $TPPUT; saves regs; B OREENT |
| OREENT | 0022AE | 4132–4135 | L R6,OBUFPTR; LTR; BE OGETBUF (OBUFPTR=0) |
| OBUFOK | 0022C2 | 4139+ | Normal path (skipped in loop) |
| OGETBUF | 00230C | 4163–4170 | Test 3 fires: only 1 buffer in $BUFPOOL → ORETURN |
| OGETBUF1 | 002358 | 4183–4185 | SR R6,R6 → ORETURN (CC=0 path for stop-buffering) |
| ORETURN | 0022F4 | 4154–4159 | Restore regs, BR R14 back to MREPUT+4 (0009C6) |

### CSECT DMTXJCA (base address 004000 in ESD, length 001000)

| Symbol | Abs. Offset | Role in loop |
|---|---|---|
| $START / $CCOMM1 | 04E4C | Live commutator table entry 0 (OPEN → BC 15,MREPUT) |
| $CCOMM1+4 … +36 | 04E50–04E70 | 9 entries, all BC 0,addr (NOPs, mask=00) |
| B CMDECK | 04E74 | Hard branch ending the commutator table fall-through |
| OBUFPTR | 04A64 | Active output buffer write pointer (=0 in loop) |
| $BUFPOOL | 04634 | Free buffer pool anchor (contains 1 entry in loop) |
| LOOPCT | 045CC | Debug iteration counter (never cleared during loop) |

---

## 14. Recommended Fixes

### Fix 1 — Break the MREPUT deadlock when stuck (primary fix)

The loop is perpetuated because `BZ $CCOMM1+4` (stmt 1341) goes into the NOP table
with no mechanism to re-post an ECB or free a buffer. Add a "stuck" detector:

```asm
; In MREPUT, after BZ $CCOMM1+4 is taken (buffer still not available):
; Before falling through the NOPs to CMDECK, open $TPGETCM explicitly.
; This forces $TPGET to run on the next CMDECK pass, which will process
; any TCT-chain buffers and potentially free one back to $BUFPOOL.

0009C6  BZ   MREPUTZ               ; branch to new label instead of $CCOMM1+4

MREPUTZ:
        OI   $TPGETCM+1,OPEN       ; force $TPGET commutator open
        B    $CCOMM1+4             ; continue with commutator fall-through
```

This ensures that even if the loop fires, $TPGET will run on the next CMDECK pass,
process any pending TCT buffers, and eventually free a buffer to $BUFPOOL. Once
$BUFPOOL grows above 1, OGETBUF will succeed and MREPUT will exit normally.

### Fix 2 — Allow using the last buffer when the CTC adapter is idle

The guard at OGETBUF stmt 4170 (`YEP...BETTER NOT USE IT`) was designed for a BSC
environment where a free buffer must always be available for the adapter's receive
path. In CTC mode, the adapter does not autonomously post receives; it only receives
as part of an explicit CCW chain (WRITE→CTRL→READ). If the adapter is idle (ADAECB
not posted), the "last buffer reserve" is unnecessary. Add a check:

```asm
; In OGETBUF, before the "BETTER NOT USE IT" branch:
002322  CLC  0(4,R6),=F'0'         ; only one left?
002328  BNE  OGETBUF2              ; no, not the last — normal path
        TM   ADAECB,X'40'          ; is CTC adapter currently busy?
        BO   ORETURN               ; yes: keep reserve (BSC-era behavior)
        ; Adapter idle: safe to use last buffer
```

### Fix 3 — Increase the buffer pool size

The simplest operational fix: increase the number of TP buffers allocated at link
startup. With more buffers, the bidirectional scenario requires more CTC cycles before
$BUFPOOL reaches 1. A pool of 8–10 buffers should be sufficient for typical 50,000-
record bidirectional transfers without hitting the one-buffer floor.

The buffer count is configured via `TNUMBUFS` (DMTXJCA offset 04474), currently set
during initialization. Increase this value.

### Fix priority summary

| Fix | Effort | Addresses |
|---|---|---|
| Fix 1: Open $TPGETCM in MREPUTZ | 2–3 instructions | Breaks the infinite loop immediately |
| Fix 2: Skip reserve when CTC idle | 2 instructions | Eliminates spurious refusal |
| Fix 3: Increase buffer pool | Configuration change | Raises the threshold before loop triggers |

**Apply Fix 1 first.** It directly addresses the confirmed loop mechanism: $TPGET
is not running during the loop, and forcing it open on each MREPUT retry will drain
the TCT buffer backlog and replenish $BUFPOOL.

---

## Appendix: Condition Code Path Through OGETBUF to MREPUT

```
OGETBUF entry:
  TM  BUFSYNSW,$TPPNONE → CC set by TM
  BO  OGETBUF1          → not taken (flag clear)
  CLC $BUFPOOL,=F'0'    → CC = not equal (1 buffer in pool)
  BE  ORETURN           → not taken
  L   R6,$BUFPOOL       → R6 = address of sole buffer (CC unchanged by L)
  CLC 0(4,R6),=F'0'     → CC = EQUAL (chain pointer = 0 = only buffer)
  BE  ORETURN           → TAKEN (CC=0)

ORETURN:
  L R8,OINADD           → CC unchanged
  L R6,OSAVR6           → CC unchanged
  L R5,OSAVR5           → CC unchanged
  L R14,OSAVR14         → CC unchanged
  BR R14                → returns to MREPUT+4 with CC=0 still set

MREPUT+4 (0009C6):
  BZ $CCOMM1+4          → CC=0 → BZ TAKEN
```

The condition code is set once (by `CLC 0(4,R6),=F'0'` at 002322), is never
modified through ORETURN's load instructions, and arrives at `BZ $CCOMM1+4`
with value 0 (equal). This is why BZ is always taken in the loop state.


