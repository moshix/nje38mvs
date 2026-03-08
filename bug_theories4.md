# Bug Theories 4: The Real Reason NJE38 Loops on Bidirectional Traffic

## Preface

This document supersedes `bug_theories3.md`. Fix 1 (opening `$TPGETCM` at `MREPUTZ`) does not
eliminate the loop because it addresses the wrong symptom. The root cause is not buffer-pool
exhaustion. It is **FCS stream negation**, a BSC-era flow-control mechanism that fires too
aggressively at CTC speeds and permanently disables output buffering (`$TPPNONE`) while both
sides are simultaneously active.

---

## 1. Key Data: CFCSSTD Is Identical in Both Drivers

Before building the hypothesis the two listings were compared directly:

| Symbol    | DMTXJC (NJE38)      | DMTYJC (Peter's driver)  |
|-----------|---------------------|--------------------------|
| `CFCSSTD` | `X'88C1'` — 04CC2   | `X'88C1'` — 003A82       |
| `CFCSTEMP`| 04CC4               | 003A84                   |
| `CFCSOUT` | 04CC0               | 003A80                   |
| `CFCSSTD` stmt | DMTXJCA stmt ~5469 | DMTYJC stmt 6059    |

**Both drivers require the same FCS bit pattern.** Therefore the loop cannot be explained by a
difference in the standard value. The difference must lie in what $FCSOUT *each system sends*,
and why one sender's $FCSOUT triggers CWAITBIT on the other side while DMTYJC's does not.

---

## 2. What `$TPPNONE` Is and How It Gets Set

### 2.1 The CYCLE FCS check (DMTXJC1, offset 002836)

Every time `COMSUP` finishes reading a CTC buffer, it falls into `CWRTNEXT`, which calls `CYCLE`:

```
002836  9140 9646  04646    TM   $FCSIN,X'40'       WAIT-A-BIT explicit signal?
00283A  4710 A424  0289C    BO   CWAITBIT            yes → stop buffering
00283E  D201 9CC4  9646     MVC  CFCSTEMP(2),$FCSIN  copy received FCS
002844  D401 9CC4  9CC2     NC   CFCSTEMP(2),CFCSSTD mask against X'88C1'
00284A  D701 9CC4  9CC2     XC   CFCSTEMP(2),CFCSSTD XOR against X'88C1'
002850  4740 A424  0289C    BC   4,CWAITBIT          non-zero → stream negated
002854  947F 9D0C  04D0C    NI   BUFSYNSW,255-$TPPNONE   (CYCLE1) clear stop flag
```

The `NC` + `XC` pair is a "both-bits-must-be-set" test:
- `NC` masks $FCSIN against `X'88C1'` → result is the subset of required bits that are present.
- `XC` XORs that result against `X'88C1'` → result is zero **only** if all required bits were present.
- If any required bit was absent, `XC` leaves a non-zero result → `BC 4,CWAITBIT` fires.

`CFCSSTD = X'88C1'` requires bit `X'0001'` (low byte, bit 0) to be present in every received
buffer. In NJE terms, `X'0001'` is `PCTFCS` — the printer-TCT flow bit.

### 2.2 CWAITBIT (DMTXJC1, offset 00289C)

```
00289C  9680 9D0C  04D0C    OI   BUFSYNSW,$TPPNONE   set X'80': STOP ALL BUFFERING
0028A0  47F0 A596  02A0E    B    CRESPOND            send a null buffer
```

Once `$TPPNONE` is set, **no new TP buffer can be allocated** anywhere in the system until
`CYCLE1` clears it. `CYCLE1` is only reached if a subsequent `CYCLE` check passes.

### 2.3 OGETBUF test 1 (DMTXJC1, offset 00230C)

```
00230C  9180 9D0C  04D0C    TM   BUFSYNSW,$TPPNONE   stop flag set?
002310  4710 AE9C  02334    BO   OGETBUF1            yes → return failure (condition cc≠0)
```

`MPUT` calls `$TPPUT` → `OREENT` → `OGETBUF`. If `$TPPNONE` is set, `OGETBUF` returns to
`$TPPUT` with a non-zero condition code. `$TPPUT` passes that cc back to `MPUT`. `MPUT` patches
`$CCOMM1` to `MREPUT` and returns. `MREPUT` re-tries the same call. Infinite loop.

---

## 3. Why PCTFCS Gets Cleared: GASSIGN and TCT Buffer Limits

### 3.1 GASSIGN/GASIT2 (DMTXJC1, offset 002686–0026A4)

Every incoming data buffer is assigned to a TCT by `GASSIGN`. When the buffer is assigned to the
printer TCT, `GASIT2` checks the running count against the configured limit:

```
002690  D500 D01C  D006  CLC  TCTBUFCT,TCTBUFLM    count ≥ limit?
002694  4740 A628  025A0 BL   GASRET               no → leave FCS bit set, return
002698  D401 9648  D01E  OC   $FCSOUT,TCTFCS       yes → mask in the bit …
00269E  D401 9648  D01E  XC   $FCSOUT,TCTFCS       … then XOR it away (clears it)
0026A4  GASRET: return
```

`TCTBUFLM` for the printer TCT is `PCTBUFLM`, defined in DMTXJCA:

```
004236  02     PCTBUFLM  DC  AL1(2)    MAX NUM OF BUFFERS ASSIGNABLE TO PRINTER TCT
```

`TCTFCS` for the printer TCT is `PCTFCS = X'0001'`.

**As soon as 2 incoming data buffers accumulate in the printer TCT, `GASSIGN` clears `X'0001'`
from `$FCSOUT`.** The cleared `$FCSOUT` is then transmitted inside every outgoing CTC buffer as
`BUFFCS`.

When the opposite NJE38 reads that buffer and runs `CYCLE`:
- `$FCSIN` now has `X'88C0'` instead of `X'88C1'` (bit `X'0001'` is gone).
- `NC` against `X'88C1'` → `X'88C0'`.
- `XC` against `X'88C1'` → `X'0001'` ≠ 0.
- `BC 4,CWAITBIT` fires.
- `$TPPNONE` is set on the receiving side.

`GSWITCH` re-adds the bit when the printer processor consumes a buffer from the printer TCT:

```
0025D8  OC   $FCSOUT,TCTFCS    restore stream bit (one buffer freed)
```

But at CTC speeds (file 71 arrives in ~3 seconds), the printer TCT refills faster than the
printer processor drains it. `PCTFCS` stays cleared in `$FCSOUT` for extended periods.

---

## 4. The Complete Loop Mechanism

### Timeline (both sides are NJE38):

```
T1  NJE38-MVS begins sending file 71 (reader processor → $TPPUT → $OUTBUF)
T2  NJE38-VM  begins receiving file 71 (GASSIGN → printer TCT)
T3  NJE38-VM  printer TCT reaches 2 buffers → GASIT2 clears PCTFCS from $FCSOUT
T4  NJE38-VM  sends a CTC buffer carrying $FCSOUT = X'88C0'  ← bit X'0001' missing
T5  NJE38-MVS reads that buffer → COMSUP → CYCLE
        $FCSIN = X'88C0'
        NC X'88C0', X'88C1' = X'88C0'
        XC X'88C0', X'88C1' = X'0001' ≠ 0
        → BC 4,CWAITBIT fires
        → OI BUFSYNSW,$TPPNONE      $TPPNONE = 1 on MVS side
        → B CRESPOND (send null buffer to VM)
T6  NJE38-VM  enqueues file 471 to send → $CONTROL builds MC1
        MPUT → $TPPUT → OREENT → OGETBUF
        TM BUFSYNSW,$TPPNONE  ← $TPPNONE is set (from T5)
        BO OGETBUF1           ← failure returned immediately
        MPUT patches $CCOMM1 → MREPUT
T7  MREPUT → BAL $TPREPUT → OREENT → OGETBUF → $TPPNONE still set → fail
        BZ MREPUTZ → OI $TPGETCM+1,OPEN → B $CCOMM1+4 → NOPs → B CMDECK
T8  CMDECK: was ADAECB posted? If yes → COMSUP → CWRTNEXT
        NJE38-VM printer TCT still has ≥ 2 buffers (printer hasn't drained yet)
        → GASSIGN still clears PCTFCS → $FCSOUT still = X'88C0'
        → CTC null buffer from CRESPOND completes → COMSUP → CYCLE → $TPPNONE set AGAIN
T9  $CCOMM1 still = MREPUT → MREPUT again → OGETBUF → $TPPNONE → fail → MREPUTZ → ...
        ↑_____________________________________________________|
                        INFINITE LOOP
```

`LOOPCT` (the user's debug counter at `CMDECK`) increments every pass. No `$SIO` is ever
initiated by MREPUT, so LOOPCT is never cleared. After 100 increments → forced 0C1 ABEND.

### Why `$TPPNONE` is never cleared during the loop:

`CYCLE1` (`NI BUFSYNSW,255-$TPPNONE`) is reached only when the FCS check at `CYCLE` passes.
The FCS check passes only when NJE38-VM's `$FCSOUT` has `X'0001'` restored. That only happens
when NJE38-VM's printer processor consumes a buffer from the printer TCT (`GSWITCH`).

During the loop, NJE38-MVS is sending null buffers (via `CRESPOND`). NJE38-VM receives them,
does nothing with $OUTBUF (null), and responds. But NJE38-MVS is NOT sending file 71 data
during the loop (reader processor is blocked because $CCOMM1 is stuck at MREPUT, not at
$CONTROL/$RDREAD). However: NJE38-VM's printer TCT already contains 2+ buffers from the
file 71 data that was sent BEFORE the loop started (T1–T3). The printer processor on
NJE38-VM must drain those buffers before PCTFCS is restored. But printer spool I/O is slow
relative to the CTC null-buffer cycling speed. The CYCLE check fires again (via COMSUP) before
the printer drains even one buffer. Result: `$TPPNONE` is re-set on every COMSUP invocation.

### Why Fix 1 (`OI $TPGETCM+1,OPEN` at MREPUTZ) does not help:

`$TPGETCM` is the commutator gate that allows `$TPGET` to run. Opening it allows incoming
buffers already in `$INBUF` to be processed. That can free the *MC1 tank* back to `$TANKPOL`.
However, it does **nothing** to clear `$TPPNONE`. `OGETBUF`'s first test (`TM BUFSYNSW,$TPPNONE`)
fires before any buffer pool check. With `$TPPNONE` set, `OGETBUF` returns failure regardless of
how many buffers are in `$BUFPOOL`. Opening `$TPGETCM` is therefore irrelevant to this loop.

---

## 5. Why the Loop Does NOT Occur with DMTYJC on the VM Side

DMTYJC (Peter's driver) has a **delay mechanism** added as a `BSC2CTC2` change, absent from
NJE38's DMTXJC:

```
; DMTYJC, CINBUF path (003160–003168):
003160  CH   R15,=H'14'   More than len(null buffer)?   BSC2CTC2
003164  BNH  *+8          Skip reset of delay counter   BSC2CTC2
003168  MVI  DELAYCNT,9   Received data, don't delay    BSC2CTC2

; Data:
003B5C  09   DELAYCNT  DC  X'9'   Delay count            BSC2CTC2
003B5D       TMRCMD1   DC  C'MSG * * Delay timer scheduled'
003B7A       TMRCMD2   DC  C'MSG * * Delay timer expired'
```

When DMTYJC receives a CTC buffer containing real data (longer than a null buffer, i.e.
`length > 14`), it resets `DELAYCNT` to 9. When DMTYJC has nothing to send, instead of
immediately issuing a null buffer, it arms a delay timer (via `DLYECB`). CTC cycling pauses for
the delay period. During this pause:

- NJE38-MVS is NOT being driven by incoming CTC data. It is in `DMTWAT` (waiting for `ADAECB`).
- Meanwhile, other ECBs in NJE38-MVS's `ECBLIST` can fire — in particular, `RDEVSYNC`
  (the spool device completion ECB for the printer).
- `RDEVSYNC` posts → `CMDECK` opens the printer commutator → printer processor runs → reads
  one buffer from the printer TCT → `GSWITCH` re-adds `PCTFCS` to NJE38-MVS's `$FCSOUT`.
- When DMTYJC's delay expires and the next CTC buffer arrives, NJE38-MVS's `CYCLE` check sees
  `$FCSIN` with `PCTFCS` restored → XOR = 0 → `CYCLE1` → `$TPPNONE` cleared.

The delay gives the printer processor exactly the time window it needs. NJE38's DMTXJC
**has no such delay**. It sends null buffers immediately, giving the printer zero slack time
between CTC cycles. `$TPPNONE` is set and re-set with every cycle; the printer never gets a
chance to drain.

### Summary of the difference:

| Behavior                          | NJE38/DMTXJC (looping) | DMTYJC (working) |
|-----------------------------------|------------------------|------------------|
| Null-buffer delay after real data | None                   | DELAYCNT=9 cycles|
| PCTBUFLM (printer TCT limit)      | 2                      | unknown, ≥2      |
| Printer drains between CTC cycles | Rarely                 | Reliably         |
| $FCSOUT X'0001' cleared frequency | High (fast CTC)        | Low (delay gap)  |
| Receiving NJE38 CWAITBIT fires    | Persistently           | Rarely/never     |
| $TPPNONE state when MC1 arrives   | SET → MPUT fails       | CLEAR → MPUT OK  |

---

## 6. CSECT and Offset Reference

| Component    | CSECT    | Offset  | Stmt  | Role                                           |
|--------------|----------|---------|-------|------------------------------------------------|
| `PCTBUFLM`   | DMTXJCA  | 004236  | 6758  | Printer TCT buffer limit = 2 → triggers stream negation |
| `GASIT2`     | DMTXJC1  | 002690  | 4463  | Clears PCTFCS from $FCSOUT when count ≥ PCTBUFLM |
| `GSWITCH`    | DMTXJC1  | 0025D8  | 4432  | Restores PCTFCS to $FCSOUT when printer drains buffer |
| `CFCSSTD`    | DMTXJCA  | 04CC2   | ~5469 | X'88C1' — requires PCTFCS (X'0001') in every $FCSIN |
| `CYCLE`      | DMTXJC1  | 002836  | 4751  | NC/XC FCS test; missing bit → branch to CWAITBIT |
| `CYCLE1`     | DMTXJC1  | 002854  | 4758  | NI BUFSYNSW,255-$TPPNONE — only clearing path  |
| `CWAITBIT`   | DMTXJC1  | 00289C  | 4785  | Sets $TPPNONE; branches to CRESPOND (null buf)  |
| `BUFSYNSW`   | DMTXJCA  | 04D0C   | —     | Contains $TPPNONE (X'80')                       |
| `OGETBUF`    | DMTXJC1  | 00230C  | 4160  | Test 1: TM $TPPNONE → failure if set            |
| `MREPUT`     | DMTXJC1  | 0009BE  | 1338  | Retry loop entry; spins when MPUT keeps failing  |
| `MREPUTZ`    | DMTXJC1  | 0009EC  | 1353  | User's fix1 entry; opens $TPGETCM (insufficient) |
| `$CCOMM1`    | DMTXJCA  | 04E50   | —     | Commutator; patched to MREPUT when MPUT fails    |

---

## 7. Proposed Fixes (in order of invasiveness)

### Fix A — Increase `PCTBUFLM` (lowest risk, try first)

Only three TCTs have a `BUFLM` / `BUFCT` field pair. The reader TCT (`RTCT`) and job TCT (`JTCT`)
use a different layout and have no `BUFLM` field. The three that do are:

| Symbol     | Offset  | Value | TCT              | FCS mask cleared |
|------------|---------|-------|------------------|------------------|
| `CCTBUFLM` | 0040AE  | 5     | Control (CTCT)   | `CCTFCS = X'0000'` (no bit) |
| `WCTBUFLM` | 004116  | 3     | Writer (WTCT)    | `WCTFCS = X'0040'` |
| `PCTBUFLM` | 004236  | 2     | Printer (PTCT)   | `PCTFCS = X'0001'` |

`CFCSSTD = X'88C1'` requires both `X'0040'` (WCTFCS) and `X'0001'` (PCTFCS) to be present in
every received buffer. The printer TCT is the bottleneck here: its limit of 2 is reached almost
immediately during fast file reception. `WCTBUFLM = 3` could also contribute if the writer is
busy, but during a pure file transfer it is unlikely to be the primary trigger.

In `DMTXJCA`, change:

```asm
; Before:
PCTBUFLM  DC  AL1(2)

; After:
PCTBUFLM  DC  AL1(6)    ; 6 of 8 TP buffers before signaling backpressure
```

`WCTBUFLM` can also be raised if needed (e.g., from 3 to 6), but the printer TCT is the
primary cause in a receive-while-sending scenario.

**Note:** This does not disable backpressure. It only raises the threshold. If the system is
under extreme load, the loop could still theoretically occur — just far less often.

### Fix B — Port the BSC2CTC2 delay mechanism from DMTYJC

Add `DELAYCNT` and a delay timer (armed via `DLYECB`) to `CRESPOND` / `CWRTNEXT`. When the
output queue is empty, decrement `DELAYCNT`. When `DELAYCNT` > 0, arm the timer and WAIT instead
of sending a null buffer immediately. Reset `DELAYCNT` to 9 whenever actual data is received
(equivalent to DMTYJC's `BSC2CTC2` change at offset 003168).

This is the correct BSC-to-CTC adaptation: CTC round-trip latency is microseconds, so
null-buffer cycling saturates the protocol. DMTYJC's delay breaks the cycle, giving the printer
processor a guaranteed window to drain the TCT between rounds.

### Fix C — Clear `$TPPNONE` in `MREPUTZ` (quick workaround)

Add `NI BUFSYNSW,255-$TPPNONE` at `MREPUTZ` before the `B $CCOMM1+4`:

```asm
MREPUTZ  EQU  *
         OI   $TPGETCM+1,OPEN          ; Fix 1 (already in place)
         NI   BUFSYNSW,255-$TPPNONE    ; Clear the buffering-stop flag
         B    $CCOMM1+4                ; Continue fall-through
```

On the next MREPUT iteration, if `ADAECB` has not yet fired (so COMSUP has not run and
re-set `$TPPNONE`), `OGETBUF` test 1 passes → buffer allocated → MC2 built → protocol
resumes.

**Risk:** This bypasses BSC stream-negation flow control for the duration of MREPUT retries.
In pure CTC mode this is acceptable: CTC's actual flow control is via ATTN (the remote will
not issue CTRL unless it is ready to receive). However, if the printer TCT has accumulated many
buffers, clearing `$TPPNONE` allows MPUT to succeed while the printer is still backlogged.
The printer will drain at spool speed; the TCT queue may grow momentarily but will not overflow
(GASSIGN still chains buffers correctly regardless of `$TPPNONE`).

**Fix C alone may not be sufficient** if COMSUP fires and re-sets `$TPPNONE` between
`MREPUTZ` and the next MREPUT call. Combining Fix C with Fix A or Fix B is recommended.

### Fix D — Change `CFCSSTD` to `X'0000'` (most aggressive)

```asm
; Before:
CFCSSTD  DC  X'88C1'   STANDARD FCS

; After:
CFCSSTD  DC  X'0000'   STANDARD FCS (disabled for CTC mode)
```

With `CFCSSTD = X'0000'`, the `NC`/`XC` test always produces zero. `BC 4,CWAITBIT` never
fires from the stream-negation path. `CWAITBIT` can still be reached via the explicit WAB
check (`TM $FCSIN,X'40'`), which fires when the remote sends a dummy buffer (`BUFFAKE`) and
sets `X'40'` in `BUFFCS`. This preserves protection against actual buffer exhaustion while
removing the overly aggressive stream-negation trigger.

**Tradeoff:** Stream-bit backpressure (the OC/XC dance in GASSIGN/GSWITCH) continues to
maintain `$FCSOUT` correctly, so the remote side still sees stream bits cleared when TCT limits
are reached. It just means the local side ignores those signals. In a two-NJE38 environment
where both sides have the same bug, disabling the check on both sides removes the deadlock.

---

## 8. Recommended Action

1. **Immediately**: Apply Fix C (clear `$TPPNONE` in `MREPUTZ`) for the quickest test. This
   should eliminate the loop in most cases by ensuring at least one MREPUT iteration succeeds
   before COMSUP can re-arm `$TPPNONE`.

2. **Short term**: Apply Fix A (increase `PCTBUFLM` from 2 to 6, and optionally raise
   `WCTBUFLM` from 3 to 6). Combine with Fix C. This reduces the frequency of `CWAITBIT`
   and makes Fix C's window of opportunity large enough to reliably break the deadlock.

3. **Proper BSC2CTC fix**: Port Fix B (the `DELAYCNT` delay mechanism) from DMTYJC. This is
   what Peter's driver does and it is the architecturally correct adaptation of the BSC null-
   buffer cycle to CTC mode. It directly addresses the timing gap problem.
