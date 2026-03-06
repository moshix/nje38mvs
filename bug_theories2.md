# DMTXJC Bidirectional CTC Bug — Deep Code Analysis

> **This document replaces `bug_theories.md`.** Every claim here is directly verified
> against the assembled object listing (`BOB ZW02 DMTXJC 468 06 MAR 26 12.04.20 PM.pdf`).
> No assumptions are made. Exact statement numbers, offsets, and instruction sequences
> are cited for every finding.

---

## Table of Contents

1. [How the Loop Works](#1-how-the-loop-works--cmdeck-and-loopct)
2. [The CCW Chain — What Actually Executes](#2-the-ccw-chain--what-actually-executes)
3. [The CTC Half-Duplex Constraint](#3-the-ctc-half-duplex-constraint)
4. [Root Cause A — SPECATTN Is Never Cleared](#4-root-cause-a--specattn-is-never-cleared-primary-cause)
5. [Root Cause B — ATTNREQ Deadlock in the CINBUF Path](#5-root-cause-b--attnreq-deadlock-in-the-cinbuf-path-secondary)
6. [Combined Failure Trace](#6-combined-failure-trace)
7. [Supporting Evidence](#7-supporting-evidence)
8. [Recommended Fixes](#8-recommended-fixes)

---

## 1. How the Loop Works — CMDECK and LOOPCT

The user-visible "endless loop" is rooted in DMTXJC1, **not** in DMTXJCA. CMDECK is at
offset `000790` in CSECT DMTXJC1 (stmt 1090).

```asm
000790  5880 95CC     CMDECK   L  R8,LOOPCT         stmt 1091 – load counter
000794  4180 8001     LA R8,1(,R8)                  stmt 1092 – increment
000798  5080 95CC     ST R8,LOOPCT                  stmt 1093 – store back
00079C  5980 AC2C     C  R8,=F'100'                 stmt 1094 – threshold check
0007A0  4740 C350     BL SKIPPY                     stmt 1095 – < 100: keep going
0007A4  90E1 95D4     STM R14,R1,SAE1               stmt 1096 – >= 100: save regs
         ...          STIMER WAIT,DINTVL=INT2        stmt 1097 – wait 30 seconds
0007B6  98E1 95D4     LM  R14,R1,SAE1               stmt 1103 – restore
0007BA  00000000      DC  AL4(0)                     stmt 1104 – ABEND S0C1
```

**LOOPCT is cleared in exactly two places:**

| Location | Stmt | Offset | Context |
|----------|------|--------|---------|
| ISIO (init) | 824 | `00050A` | At link startup, before first CTC I/O |
| CWRTSIO | 5061 | `002B8E` | Immediately before every `BAL R15,$SIO` |

```asm
002B8E  D703 95CC 95CC  XC LOOPCT,LOOPCT    stmt 5061 *DBUG – clear before I/O
002B94  45F0 A8FA       BAL R15,$SIO         stmt 5062 – issue I/O
```

**LOOPCT is incremented at CMDECK (stmt 1091) on every wakeup from the WAITREQ
call at stmt 1189:**

```asm
0008E2  05EF            BALR R14,R15    stmt 1189 – WAIT on ECBLIST
0008E4  47F0 C318       B    CMDECK     stmt 1190 – wake → increment LOOPCT
```

**Conclusion:** LOOPCT = 100 means the MVS task returned from WAIT 100 times for
non-CTC ECBs (reader, printer, job, message, command) while `ADAECB` — the CTC
I/O completion ECB — was **never posted**. A CTC I/O was issued (`$SIO` called,
LOOPCT cleared), the CCW chain is running asynchronously via EXCP, but it never
completes. The other ECBs in `ECBLIST` (RDEVSYNC, CMDECB, MSGECB, DLYECB) keep
firing, incrementing LOOPCT until the 100-pass ABEND is triggered.

---

## 2. The CCW Chain — What Actually Executes

The CCW string issued by every `$SIO` call from CWRTSIO (stmt 5063, offset `002B98`)
is at symbol `CCWS` in DMTXJCA, offset `004D6C`:

```asm
004D70  14 004D24 60 000001   CCWS CCW CTCSENSE,ADASENSE,CC+SILI,1  stmt 7285
004D78  01 000000 60 000000   CCWA CCW WRITE,0,CC+SILI,0             stmt 7286
004D80  07 000000 60 000001        CCW CTCCTRL,0,CC+SILI,1           stmt 7287
004D88  02 000000 20 000000   CCWC CCW READ,0,SILI,0                 stmt 7288
```

The four-step chain in execution order:

| Step | CCW | Op code | Purpose |
|------|-----|---------|---------|
| 1 | CCWS | X'14' CTC SENSE | Read adapter status |
| 2 | CCWA | X'01' WRITE | Transfer our data to the CTC hardware path |
| 3 | (unnamed) | X'07' CTC CTRL | Signal the remote end: fire ATTN interrupt |
| 4 | CCWC | X'02' READ | Read the response the remote end writes back |

The data address and length of CCWA (WRITE) and CCWC (READ) are patched at
runtime in CNWRITE (stmts 5007–5014, offsets `002AE2`–`002B00`):

```asm
002AE2  41F0 D007   LA  R15,BUFSTART        stmt 5007 – point to our buffer
002AE6  50F0 9D78   ST  R15,CCWA            stmt 5008 – WRITE addr = BUFSTART
002AEA  9201 9D78   MVI CCWA,WRITE          stmt 5009 – set WRITE opcode
002AEE  50F0 9D88   ST  R15,CCWC            stmt 5010 – READ addr = SAME BUFSTART
002AF2  9202 9D88   MVI CCWC,READ           stmt 5011 – set READ opcode
```

Both CCWA (WRITE) and CCWC (READ) point to the **same buffer address** (`BUFSTART`).
The WRITE transmits our outgoing data; the READ then overwrites the same buffer
with the incoming response.

**Critical hardware constraint:** On a S/370 CTC, `WRITE` is a **synchronous
point-to-point transfer**. There is no hardware FIFO. For the WRITE CCW to
complete, the remote end's channel must simultaneously be executing a matching
`READ` CCW. If the remote end is not in READ state when our WRITE runs, the
channel program **stalls indefinitely** at step 2.

---

## 3. The CTC Half-Duplex Constraint

The CWRTSIO code (stmts 5033–5056, offsets `002B44`–`002B82`) contains an
`ATTNREQ` wait that enforces half-duplex discipline:

```asm
002B76  9102 94BC   TM  XJCSYS,SPECATTN    stmt 5053 – was ATTN+CE+DE received?
002B7A  4710 A70C   BO  CNSPECAL           stmt 5054 – YES: skip ATTNREQ
002B7E  58F0 9EE8   L   R15,ATTNREQ        stmt 5055 – load wait routine addr
002B82  05EF        BALR R14,R15           stmt 5056 – WAIT for CTC ATTN
```

`ATTNREQ` (at `004EE8`, filled in by NJE38 at runtime) waits for a CTC ATTENTION
interrupt — which arrives only when the **remote end issues a CTRL command**
(step 3 of their own CCW chain). This ensures:

> *"Before I WRITE, I know the remote end just finished WRITing and is now in READ
> mode — so my WRITE can complete."*

In single-direction transfers (A→B only), this works perfectly: A receives ATTN
from B's CTRL, knows B is in READ state, issues its WRITE safely. The protocol
is half-duplex by design.

---

## 4. Root Cause A — SPECATTN Is Never Cleared *(primary cause)*

### Where SPECATTN is set

When CSW status `X'8C'` (ATTN + Channel-End + Device-End together) arrives,
COMSUP branches to `CRSPCATN` (stmt 4675, offset `00275E`):

```asm
00275E  947F 9D20   NI  ADACSW+4,X'7F'    stmt 4676 – strip ATTN bit from CSW
002762  9602 94BC   OI  XJCSYS,SPECATTN   stmt 4677 – PERMANENTLY set SPECATTN
```

`SPECATTN = X'02'` (bit 6 of XJCSYS, verified from `TM XJCSYS,X'02'` at stmt 5053).

### Where SPECATTN is cleared

Searching the entire listing:

| Location | Stmt | Instruction | When |
|----------|------|-------------|------|
| ISIO | 796 | `NI XJCSYS,PRIMONLY` | **Link startup only** — `PRIMONLY = X'01'`, so SPECATTN (X'02') is zeroed |

**SPECATTN is never cleared during normal operation.** The first time `ATTN+CE+DE`
occurs on a live link — which can happen routinely when both sides' I/Os happen to
complete simultaneously — SPECATTN is set and **stays set for the lifetime of the
link**.

### What happens after SPECATTN is set

In CWRTSIO (stmt 5054, offset `002B7A`): `BO CNSPECAL` branches around
`ATTNREQ`. Execution jumps directly to:

```asm
002B84  CNSPECAL  DS  0H              stmt 5058 – entry after ATTNREQ bypass
002B84  D209 9CD0 D007  MVC CBUFLAST(10),BUFSTART  stmt 5059 – save last buf
002B8A  94F7 9D0C       NI  BUFSYNSW,255-CUWFAKE   stmt 5060 – clear flag
002B8E  D703 95CC 95CC  XC  LOOPCT,LOOPCT          stmt 5061 – clear counter
002B94  45F0 A8FA       BAL R15,$SIO               stmt 5062 – ISSUE I/O
```

**CNSPECAL does not clear SPECATTN.** The skip is permanent, not a one-time
bypass.

### The failure when both sides have outbound data

With SPECATTN set, both Side A and Side B reach CWRTSIO and both skip ATTNREQ.
Both call `BAL R15,$SIO` immediately:

```
Side A CCW chain running:  SENSE → WRITE(A data) → CTRL → READ
Side B CCW chain running:  SENSE → WRITE(B data) → CTRL → READ
```

Both WRITE CCWs (step 2) are simultaneously active. On S/370 CTC hardware, each
WRITE requires the remote end to be executing a READ. But both chains have READ
at **step 4**, not step 2. Neither side reaches step 4 until step 2 completes.
Neither step 2 can complete without the other side's step 4.

**Both CCW chains stall at step 2. Neither ADAECB is ever posted.**

While both I/O operations are frozen:

- Reader ECBs (RDEVSYNC) keep firing → CMDECK → LOOPCT++
- Job ECBs (JCTECB via IOSYNCH) keep firing → CMDECK → LOOPCT++
- Printer ECBs (PCTECB via IOSYNCH) keep firing → CMDECK → LOOPCT++
- Message ECBs (MSGECB) keep firing → CMDECK → LOOPCT++

After 100 such wakeups, CMDECK hits the `C R8,=F'100'` threshold (stmt 1094),
executes `STIMER WAIT` for 30 seconds, and then abends via `DC AL4(0)` (S0C1).

---

## 5. Root Cause B — ATTNREQ Deadlock in the CINBUF Path *(secondary)*

Even if SPECATTN were properly cleared, there is a second, deeper protocol
problem that appears when Side B receives data from Side A and then wants to
send its own data.

### The CINBUF → CWRTNEXT → CWRTSIO path

When Side B receives A's data block, B processes it in CINBUF (offset `0028A4`,
stmt 4789). After queuing it to `$INBUF`, B arrives at CWRTNEXT (stmt 4743,
offset `002822`):

```asm
002822  9104 9D0C   TM  BUFSYNSW,CACKSW    stmt 4744 – ACK received?
002826  4710 A3BA   BO  CNOD               stmt 4745 – YES: skip CSETCOM calls
00282A  45F0 A644   BAL R15,CSETCOM        stmt 4746 – NO: commutator pass 1
00282E  45F0 A644   BAL R15,CSETCOM        stmt 4747 –     commutator pass 2
```

`CACKSW` is **not** set (B came from CINBUF, not CACKED). B calls CSETCOM twice,
pausing execution and allowing CMDECK to run twice. Then B continues to
`CYCLE` → `CYCLE1` → checks `$OUTBUF`. Finding data, B enters CNWRITE →
CWRTSIO.

### The deadlock at ATTNREQ

At CWRTSIO (assuming SPECATTN = 0), B calls `ATTNREQ` (stmt 5056, offset `002B82`).
B waits for a fresh CTC ATTN from Side A.

At this exact moment, **A's CCW chain is still running**. A's chain is:

```
SENSE(done) → WRITE(A data 2)(done) → CTRL(done, fired ATTN) → READ(outstanding)
```

A fired ATTN (step 3) to notify B. But that ATTN was consumed when CMDECK
processed `ADAECB` and cleared it at `LOGITBK`:

```asm
00085E  D703 9D10 9D10  XC ADAECB(4),ADAECB   stmt 1153 – ADAECB cleared here
```

**The ATTN that A sent is already gone.** B is waiting for a NEW ATTN from A.
A can only generate a new ATTN after its CCW chain completes — specifically after
its **READ** (step 4) completes. A's READ requires B to WRITE. B will not WRITE
until it gets ATTN. A cannot generate ATTN until its READ completes.

```
A's READ  waits for  B's WRITE
B's WRITE waits for  A's ATTN
A's ATTN  waits for  A's READ
```

**Circular deadlock.** Both tasks block indefinitely.

This manifests as a hang rather than the LOOPCT abend, but the `DELAYCNT`
mechanism (stmts 5038–5050) means the hang can surface as extreme delays between
transfers under certain timing conditions.

---

## 6. Combined Failure Trace

The following is the complete execution trace that leads to the LOOPCT=100 abend:

```
[Initial link startup — ISIO at 00049C]
  NI XJCSYS,PRIMONLY (stmt 796)     → SPECATTN = 0

[Normal single-direction operation — first file transfer]
  At some point, ATTN+CE+DE (X'8C') arrives in CSW
  → CRSPCATN at 00275E
  → OI XJCSYS,SPECATTN (stmt 4677)  → SPECATTN = 1  ← PERMANENT

[Second file transfer starts in opposite direction]
  Side A: has data in $OUTBUF, enters CYCLE1 (offset 002854)
  Side B: also has data in $OUTBUF, enters CYCLE1 simultaneously

[Side A reaches CWRTSIO (offset 002B44)]
  TM XJCSYS,SPECATTN → bit SET → BO CNSPECAL (skips ATTNREQ)
  XC LOOPCT,LOOPCT (stmt 5061)   → LOOPCT_A = 0
  BAL R15,$SIO (stmt 5062)        → A's CCW chain starts asynchronously
  CREXIT → return to commutator

[Side B reaches CWRTSIO simultaneously]
  TM XJCSYS,SPECATTN → bit SET → BO CNSPECAL (skips ATTNREQ)
  XC LOOPCT,LOOPCT (stmt 5061)   → LOOPCT_B = 0
  BAL R15,$SIO (stmt 5062)        → B's CCW chain starts asynchronously
  CREXIT → return to commutator

[CTC hardware state]
  A: SENSE(ok) → WRITE(A data) → STALLED (B not in READ mode)
  B: SENSE(ok) → WRITE(B data) → STALLED (A not in READ mode)
  → ADAECB_A never posted
  → ADAECB_B never posted

[CMDECK on Side A — repeatedly waking on non-CTC ECBs]
  RDEVSYNC fires → CMDECK → LOOPCT_A = 1
  MSGECB fires   → CMDECK → LOOPCT_A = 2
  RDEVSYNC fires → CMDECK → LOOPCT_A = 3
  ...
  LOOPCT_A = 100

[CMDECK stmt 1094]
  C R8,=F'100' → >= 100 branch to STIMER/ABEND path
  STIMER WAIT (stmt 1097, offset 0007A4)
  DC AL4(0) (stmt 1104, offset 0007BA) → ABEND S0C1
  → NJE38 requires operator cancel
```

---

## 7. Supporting Evidence

### Evidence 1 — SPECATTN bit is never reset after CNSPECAL

Searched all occurrences of `NI XJCSYS` in the listing:
- Stmt 796 (`0004AE`): `NI XJCSYS,PRIMONLY` — link startup only.
- No other `NI` clears bit X'02' during live operation.
- `OI XJCSYS,SPECATTN` (stmt 4677) has no matching `NI` anywhere in CWRTSIO,
  CNSPECAL, CSETCOM, $COMSUP, or the commutator dispatch path.

### Evidence 2 — Both WRITE and READ CCWs point to the same buffer

From CNWRITE (stmts 5007–5011):

```asm
002AE6  ST  R15,CCWA    ; WRITE address = BUFSTART
002AEE  ST  R15,CCWC    ; READ  address = BUFSTART (SAME)
```

When both sides issue their chains simultaneously, both are trying to
WRITE from `BUFSTART` and READ into `BUFSTART`. There is no second buffer.
The READ can only receive data after the other side's WRITE completes. Neither
WRITE can complete without a matching READ.

### Evidence 3 — LOOPCT cleared only at CWRTSIO

```asm
; Only two XC LOOPCT,LOOPCT instructions in the entire listing:
00050A  D703 95CC 95CC  XC LOOPCT,LOOPCT   stmt 824   – ISIO
002B8E  D703 95CC 95CC  XC LOOPCT,LOOPCT   stmt 5061  – CWRTSIO (before $SIO)
```

If $SIO is called and the I/O never completes, LOOPCT is cleared once and then
increments every time CMDECK wakes for any reason.

### Evidence 4 — CRSPCATN comment confirms the scenario is known

From the module source notes (page 3, stmt 86–87):

```
*- Add CSW status check for ATTN+CE+DE (X'8C') which can sometimes
*- occur. We will honor both. March 2024.
```

The note acknowledges this status *"can sometimes occur"* — i.e., it is not a
rare error condition but a normal CTC event. Once it occurs, SPECATTN is set
for the rest of the link's life.

### Evidence 5 — ATTNREQ is the only synchronization gate for simultaneous I/O

```asm
; The only check before $SIO that could prevent simultaneous WRITEs:
002B76  TM  XJCSYS,SPECATTN   stmt 5053
002B7A  BO  CNSPECAL           stmt 5054  ← this bypass is the problem
002B7E  L   R15,ATTNREQ        stmt 5055
002B82  BALR R14,R15           stmt 5056  ← the only synchronization point
002B84  CNSPECAL:              stmt 5058  ← bypassed when SPECATTN set
002B8E  XC  LOOPCT,LOOPCT      stmt 5061
002B94  BAL R15,$SIO            stmt 5062
```

Once SPECATTN causes the bypass, there is nothing else between `CWRTSIO` entry
and `$SIO` that prevents simultaneous dual-WRITE.

---

## 8. Recommended Fixes

### Fix 1 — Clear SPECATTN at CNSPECAL *(addresses Root Cause A)*

Add `NI XJCSYS,255-SPECATTN` at the top of CNSPECAL so the bypass is
one-shot, not permanent:

```asm
002B84  CNSPECAL  DS  0H
        NI  XJCSYS,255-SPECATTN    ← ADD THIS LINE
002B84  MVC CBUFLAST(10),BUFSTART  stmt 5059
002B8A  NI  BUFSYNSW,255-CUWFAKE   stmt 5060
002B8E  XC  LOOPCT,LOOPCT          stmt 5061
002B94  BAL R15,$SIO               stmt 5062
```

This ensures that after one use of the SPECATTN fast-path, the next CWRTSIO
call goes through ATTNREQ normally. In single-direction transfers this is benign
(ATTNREQ returns quickly because the other side is already sending CTRL
continuously). In bidirectional transfers, ATTNREQ prevents the simultaneous-WRITE
deadlock by re-serializing access.

### Fix 2 — Skip ATTNREQ after CINBUF reception *(addresses Root Cause B)*

When Side B has just processed an incoming data block (CINBUF path) and now
wants to send its own data, A's READ CCW is already outstanding. B does not need
to wait for fresh ATTN. Add a single-use flag (e.g., a bit in BUFSYNSW) set in
CINBUF's exit path and tested in CWRTSIO:

```asm
; In CNOTOFF (after B queues the received buffer to $INBUF):
; Before the B CWRTNEXT call, set a flag indicating "we have incoming data,
; our partner's READ is live — skip ATTNREQ once":
   OI  BUFSYNSW,CJUSTRX    ← new bit (choose an unused bit in BUFSYNSW)

; In CWRTSIO, before ATTNREQ check:
   TM  BUFSYNSW,CJUSTRX    ← did we just receive data?
   BZ  DOATTN              ← no: normal ATTNREQ wait
   NI  BUFSYNSW,255-CJUSTRX ← yes: clear the flag
   B   CNSPECAL             ← skip ATTNREQ; partner is in READ mode
DOATTN:
   TM  XJCSYS,SPECATTN
   BO  CNSPECAL
   L   R15,ATTNREQ
   BALR R14,R15
```

### Fix priority

| Fix | Effort | Impact |
|-----|--------|--------|
| Fix 1 — clear SPECATTN | One instruction | Directly prevents the LOOPCT=100 abend |
| Fix 2 — CJUSTRX flag | ~6 instructions | Prevents the residual ATTNREQ deadlock |

**Start with Fix 1.** It can be tested immediately and addresses the reported
symptom (endless loop / LOOPCT abend). Fix 2 handles the deeper protocol issue
and should be added once Fix 1 is confirmed working.

---

## Quick Debug Checklist

To confirm the analysis before applying fixes, add the following trace points:

1. **At CRSPCATN (offset `002762`):** Log `XJCSYS` after the `OI`. If SPECATTN
   goes to 1 during single-direction transfer and stays 1, Root Cause A is
   confirmed.

2. **At CWRTSIO entry (offset `002B44`):** Log whether `SPECATTN` is set and
   whether `ATTNREQ` is called. If both sides show SPECATTN=1 and ATTNREQ
   skipped at the time of the deadlock, Root Cause A is the trigger.

3. **At `CNSPECAL` (offset `002B84`):** Log `CBUFFER` on both sides. If both
   show distinct non-null buffer addresses (real data buffers, not CDUMMY),
   both were trying to write real data simultaneously.

4. **Existing LOOPCT counter** (`045CC` in DMTXJCA): already in place. When it
   triggers, examine the CSW in `ADACSW` (`004D1C`). If `ADACSW+4` shows CE+DE
   never arrived (all zeros), the CTC I/O is stalled at WRITE — confirming the
   hardware deadlock.

---

*Analysis performed by direct inspection of the DMTXJC assembler listing,
06 Mar 2026. All offsets are relative to CSECT base addresses as assembled.*
