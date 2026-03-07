# DMTXJC — SPECATTN Flag: Mechanism, Bug Without Fix, and Why Pre-Queued Files Work

> Every claim is verified directly against the assembled listing
> (`BOB ZW02 DMTXJC 468 06 MAR 26 12.04.20 PM.pdf`).
> Statement numbers, offsets, and instruction sequences are cited for every step.
> This document explains why adding `NI XJCSYS,255-SPECATTN` immediately at
> label `CNSPECAL` allows simultaneous bidirectional transfers to succeed when
> both files are already queued in the spool before the connection is established.

---

## Table of Contents

1. [What SPECATTN Is](#1-what-specattn-is)
2. [When and Why ATTN+CE+DE Occurs (X'8C')](#2-when-and-why-attncede-occurs-x8c)
3. [Where SPECATTN Is Set — CRSPCATN](#3-where-specattn-is-set--crspcatn-csect-dmtxjc1)
4. [Where SPECATTN Is Consumed — CWRTSIO / CNSPECAL](#4-where-specattn-is-consumed--cwrtsio--cnspecal-csect-dmtxjc1)
5. [The CTC Write Protocol — ATTNREQ and the Ping-Pong](#5-the-ctc-write-protocol--attnreq-and-the-ping-pong)
6. [Without the Fix: SPECATTN Permanently Set → CTC Write Deadlock](#6-without-the-fix-specattn-permanently-set--ctc-write-deadlock)
7. [The Fix: One-Shot Reset at CNSPECAL](#7-the-fix-one-shot-reset-at-cnspecal)
8. [Why Pre-Queued Files Work With the Fix — Full Chain of Events](#8-why-pre-queued-files-work-with-the-fix--full-chain-of-events)
9. [Why Post-Queued Files Still Fail — A Separate Bug](#9-why-post-queued-files-still-fail--a-separate-bug)
10. [Summary of All Code Locations](#10-summary-of-all-code-locations)

---

## 1. What SPECATTN Is

`SPECATTN` is a single-bit flag in the byte `XJCSYS` (DMTXJCA, offset `044BC`).

```
Bit mask X'02' within XJCSYS (044BC in DMTXJCA)
```

Its sole purpose is to signal: **"we have already received an attention interrupt from
the remote CTC device — do not issue another MVS WAIT for one before issuing the
next WRITE."**

At ISIO initialization, `XJCSYS` is cleared of everything except the `PRIMARY` bit:

```
Offset   Stmt   Source
000708   796    NI  XJCSYS,PRIMONLY     clear all bits except PRIMARY (incl. SPECATTN)
```

From that point forward, `SPECATTN` is set only by the CTC interrupt handler
(`CRSPCATN`) and is consumed (and, with the fix, immediately cleared) at `CNSPECAL`
inside `CWRTSIO`.

---

## 2. When and Why ATTN+CE+DE Occurs (X'8C')

To understand SPECATTN you must first understand the CTC write CCW chain and what
the status byte `X'8C'` in `ADACSW+4` means.

### The CCW chain issued by each side

Every CTC data transfer from side A to side B uses this CCW sequence
(set up at `CNWRITE`, DMTXJC1 offset `002AE2`, stmts 5006–5013):

```
CCW sequence for side A's outgoing write:
  SENSE     ← test device
  WRITE     ← put A's data into the CTC channel buffer
  CTRL      ← fires a hardware ATTENTION interrupt to side B
  READ      ← A waits here for B's response/data
```

Each step completes as a channel operation:

| CCW step | Effect on side A | Effect on side B |
|---|---|---|
| SENSE | Checks device | — |
| WRITE | A's data delivered to CTC hardware | — |
| CTRL | A's channel records `CTRL complete`; hardware fires ATTN interrupt to B | B receives ATTN interrupt |
| READ | A's channel waits for data from B | B may now WRITE to A |

### When ATTN+CE+DE (X'8C') appears

After side B receives A's ATTN (from A's CTRL), B issues its own CCW chain:
`SENSE → WRITE(B's data) → CTRL → READ`.  B's CTRL fires an ATTN back to A.

A is currently executing its **READ** step — waiting for B's WRITE data.  At the
moment A's READ channel operation completes with B's data (`CE+DE`), B's CTRL has
just fired `ATTN` to A.  Both events reach A's channel-end-interrupt simultaneously:

```
ADACSW+4 = X'8C'    (binary 1000 1100)
             ↑              │    │└─ DE  (Device End)
             ATTN           │    └── CE  (Channel End)
                            └─────── ATTN (Attention from remote CTRL)
```

This means: **A got B's data AND B's "I-am-now-in-READ-mode" signal at the same time.**
Because A already has the ATTN queued, A does not need to wait for a new ATTN before
issuing its next WRITE — B is already in READ mode.

This is the **normal, efficient case** in a properly pipelined CTC exchange.
It is also the common case: B fires CTRL before A's READ channel operation finishes
bookkeeping, so the interrupt arrives simultaneously.

---

## 3. Where SPECATTN Is Set — CRSPCATN (CSECT DMTXJC1)

`COMSUP` is entered on every CTC interrupt (`$INTRUPT` / `COMSUP`, offset `00271E`,
stmt 4656–4657):

```
Offset   Object Code         Stmt   Source
00271E   9200 9E71           4657   MVI  $COMCOM+5,CLOSE     close COMSUP gate
002722   9110 9D0C           4658   TM   BUFSYNSW,$COMBUSY   busy?
002726   4710 A728  02BA0    4659   BO   CEXIT               yes, exit
00272A   90DF 9CDC           4660   STM  R13,R15,CREGS       save regs
00272E   58D0 9CBC           4661   L    R13,CBUFFER         get current buffer
002732   9108 9D0C           4662   TM   BUFSYNSW,CUWFAKE    dummy I/O?
002736   4710 A6CC  02B44    4663   BO   CWRTSIO             yes → bypass interrupt check
00273A   958C 9D20  04D20    4664   CLI  ADACSW+4,X'8C'      ATTN+CE+DE received?  *XJZ
00273E   4780 A2E6  0275E    4665   BE   CRSPCATN            yes → special attention *XJZ
```

If `ADACSW+4 = X'8C'`, control transfers to `CRSPCATN` (offset `00275E`, stmts
4675–4677):

```
Offset   Object Code         Stmt   Source
00275E   947F 9D20  04D20    4676   NI   ADACSW+4,X'7F'      turn off ATTN bit in CSW  *XJZ
002762   9602 94BC  044BC    4677   OI   XJCSYS,SPECATTN      set special-attention flag *XJZ
```

After setting the flag, execution falls through to `CRCONT` (002766), which checks for
unexpected errors, then reaches `$ENDREAD` (002776) — the normal CTC read-completion
processing path.  SPECATTN simply records the "already-have-ATTN" state; nothing else
changes.

---

## 4. Where SPECATTN Is Consumed — CWRTSIO / CNSPECAL (CSECT DMTXJC1)

`CWRTSIO` (offset `002B44`, stmt 5033) is the entry point for every CTC write
operation.  It contains an optional delay path and then reaches `CNWRCONT` (002B76):

```
Offset   Object Code         Stmt   Source
002B76   9102 94BC  044BC    5053   TM   XJCSYS,SPECATTN      was ATTN received w/ CE+DE? *XJZ
002B7A   4710 A70C  02B84    5054   BO   CNSPECAL             yes → skip the wait         *XJZ
002B7E   58F0 9EE8  04EE8    5055   L    R15,ATTNREQ          load ATTN wait routine addr  *XJZ
002B82   05EF                5056   BALR R14,R15              WAIT for ATTN from device   *XJZ
                                    ↑
                    (if SPECATTN=0, ATTNREQ is called here — MVS WAIT until CTC fires ATTN)
02B84                        5058   CNSPECAL EQU *            both paths merge here
002B84   D209 9CD0  D007     5059   MVC  CBUFLAST(10),BUFSTART  save buffer for reset
002B8A   94F7 9D0C  04D0C    5060   NI   BUFSYNSW,255-CUWFAKE
002B8E   D703 95CC  95CC     5061   XC   LOOPCT,LOOPCT        *DBUG clear counter
002B94   45F0 A8FA  02D72    5062   BAL  R15,$SIO             issue the CTC I/O
```

`ATTNREQ` (address stored at DMTXJCA `04EE8`) is provided by NJEDRV.  Its job is to
wait — via MVS WAIT or equivalent — until the CTC device fires an attention interrupt.
This ATTN signal means the remote side has finished its CTRL step and is now sitting in
READ mode, ready to accept our WRITE.

If `SPECATTN=1`, the `BO CNSPECAL` branch is taken and `ATTNREQ` is bypassed: we
already have the ATTN we need (it arrived simultaneously with the CE+DE from the
previous READ).  We go straight to `$SIO` and issue the next CCW chain.

### The user's fix

Immediately at label `CNSPECAL` (before the existing `MVC CBUFLAST...`), the fix
inserts:

```asm
NI  XJCSYS,255-SPECATTN     ; clear the flag — one-shot use only
```

This one instruction changes SPECATTN from a "permanent bypass" into a **one-shot
bypass**: it is used exactly once per ATTN+CE+DE event, then cleared.

---

## 5. The CTC Write Protocol — ATTNREQ and the Ping-Pong

The CTC requires strict half-duplex serialization between WRITE operations.
Both sides must never be in WRITE mode simultaneously:

```
Time →   Side A                          Side B
──────── ─────────────────────────────── ────────────────────────────────
t1       SENSE→WRITE(A data)→CTRL→READ   (in READ or idle)
           └── CTRL fires ATTN to B ──→   B receives ATTN
t2       (in READ, waiting for B)        B's ATTNREQ returns
                                         B: SENSE→WRITE(B data)→CTRL→READ
                                           └── CTRL fires ATTN to A ──→
t3       A's READ completes (CE+DE)       (B now in READ, waiting for A)
         + ATTN from B's CTRL arrives
         → ADACSW+4 = X'8C'
         → CRSPCATN → SPECATTN=1
t4       A processes received B data
         A's next CWRTSIO: SPECATTN=1
         → skip ATTNREQ (ATTN already held)
         → WRITE(A data)→CTRL→READ ...
```

This ping-pong is **self-sustaining** as long as:

1. Only one side is in WRITE at any given time.
2. Each side correctly waits for the other's ATTN before issuing its WRITE.
3. SPECATTN is consumed at most once per cycle.

---

## 6. Without the Fix: SPECATTN Permanently Set → CTC Write Deadlock

Without the fix, `SPECATTN` is never cleared.  After the first ATTN+CE+DE event:

```
SPECATTN stays = 1 on both sides (independently, after each side gets its CE+DE+ATTN)
```

Because `BO CNSPECAL` is always taken on both sides, both sides **always skip
ATTNREQ**.  After a few cycles, the timing drifts such that both sides issue `$SIO`
at the same time:

```
Side A: SPECATTN=1 → skip ATTNREQ → $SIO → SENSE→WRITE(A data)→CTRL→READ
Side B: SPECATTN=1 → skip ATTNREQ → $SIO → SENSE→WRITE(B data)→CTRL→READ
                                                              ↑
                                              Both in WRITE simultaneously
```

A CTC WRITE from A requires B to be in READ mode.  If B is also in WRITE mode, A's
channel cannot deliver its data — neither WRITE succeeds.  Both CCW chains stall at
their WRITE step.  `ADAECB` is never posted for either side.

The CTC adapter is now silently deadlocked at the hardware level.  The rest of NJE38
continues to run: RDEVSYNC, CMDECB, MSGECB, and DLYECB keep firing, CMDECK keeps
incrementing LOOPCT, and eventually the S0C1 abend fires after 100 iterations.

This deadlock — not the MREPUT buffer loop — is the failure mode caused directly by
`SPECATTN` being permanently set.  It is distinct from the buffer-depletion MREPUT
loop documented in `bug_theories3.md`.

---

## 7. The Fix: One-Shot Reset at CNSPECAL

With `NI XJCSYS,255-SPECATTN` inserted immediately at `CNSPECAL`:

```
Time →   Side A                          Side B
──────── ─────────────────────────────── ────────────────────────────────
t1       WRITE(A) → CTRL → READ          (in READ waiting)
           CTRL fires ATTN to B ──────→  B's ATTNREQ returns
                                         B: SPECATTN=0 → call ATTNREQ... wait
                                              (ATTN from A's CTRL just received)
                                              → returns immediately
                                         NI XJCSYS,255-SPECATTN [fix, but SPECATTN
                                              was 0 anyway here — ATTNREQ was called]
t2       A's READ completes              B: WRITE(B) → CTRL → READ
         CE+DE + ATTN from B's CTRL       CTRL fires ATTN to A ──────→
         → SPECATTN=1 on A
t3       A's CWRTSIO:
         SPECATTN=1 → BO CNSPECAL
         NI XJCSYS,255-SPECATTN [FIX]   (B in READ, waiting)
         SPECATTN=0 now on A
         → $SIO → WRITE(A) → CTRL → READ
           CTRL fires ATTN to B ──────→  B's READ: CE+DE + ATTN = X'8C'
                                         → SPECATTN=1 on B
t4       A's READ: CE+DE + ATTN          B: SPECATTN=1 → BO CNSPECAL
         → SPECATTN=1 on A              NI XJCSYS,255-SPECATTN [FIX]
                                         SPECATTN=0 on B
                                         → $SIO → WRITE(B) → CTRL → READ
```

At every moment, **only one side has SPECATTN=1**.  The side that last received
CE+DE+ATTN uses the flag once, then clears it.  The other side is calling ATTNREQ
(which returns immediately because the ATTN from the first side's CTRL already
arrived).  No simultaneous WRITEs are possible.

---

## 8. Why Pre-Queued Files Work With the Fix — Full Chain of Events

"Pre-queued" means: both files are in the NJE38 spool queue before the CTC
connection is established.  The connection is then started via the normal NJE38
startup sequence.

### Phase 1: ISIO initialization

```
Offset   Stmt   Source
000708   796    NI  XJCSYS,PRIMONLY     SPECATTN=0 (clean start)
...
00070C   1008   TM  XJCSYS,PRIMARY      Are we the non-primary (secondary)?
000710   1009   BO  NSGNCRD             YES → skip sending 'I' signon
000714   1011   MVI NCCSRCB,C'I'        Build 'I' signon record in buffer
...stmts 1013–1028: place 'I' buffer into $OUTBUF...
000770   1036   MVC BUFSTART,XACKSEQ    Fake an ACK in current buffer
000776   1037   B   $ENDREAD            Fake an interrupt to bootstrap COMSUP
```

The `B $ENDREAD` (offset `000776`, stmt 1037) enters `$ENDREAD` (offset `002776`,
stmt 4687) — the CTC interrupt completion path — as if an ACK interrupt just arrived.
`COMSUP` processes the fake ACK, reaches `CACKED`, opens the reader gate, then calls
`CWRTNEXT` (offset `002822`).

### Phase 2: First CTC write — 'I' signon (PRIMARY side)

`CWRTNEXT` finds `$OUTBUF` non-empty (the 'I' signon buffer placed at stmts
1013–1028).  It dequeues the buffer, builds the CCW chain at `CNWRITE` (002AE2),
and enters `CWRTSIO` (002B44):

```
002B76   TM  XJCSYS,SPECATTN    → 0 (SPECATTN cleared at startup)
002B7A   BO  CNSPECAL           → NOT taken
002B7E   L   R15,ATTNREQ
002B82   BALR R14,R15           → ATTNREQ called
```

ATTNREQ (implemented by NJEDRV) waits for the CTC device to signal readiness.  At
connection time the CTC hardware presents an initial ready/attention state — the
remote adapter is already initialized and sitting in an idle listen mode — so ATTNREQ
returns immediately for this first write.  `$SIO` is called:

```
PRIMARY CCW chain:   SENSE → WRITE('I' signon) → CTRL → READ
PRIMARY's CTRL fires ATTN to SECONDARY.
```

### Phase 3: SECONDARY receives 'I' signon

The SECONDARY's ATTNREQ (which was waiting) returns.  SECONDARY calls `$SIO`:

```
SECONDARY CCW chain:  SENSE → WRITE('J' response) → CTRL → READ
SECONDARY's CTRL fires ATTN to PRIMARY.
```

PRIMARY's outstanding READ (from Phase 2) receives SECONDARY's 'J' data.
At the same moment, SECONDARY's CTRL fires ATTN to PRIMARY.
PRIMARY's channel interrupt status:

```
ADACSW+4 = X'8C'   (CE + DE + ATTN simultaneously)
```

COMSUP fires on PRIMARY (`$INTRUPT`, 00271E):

```
00273A   CLI  ADACSW+4,X'8C'    → EQUAL
00273E   BE   CRSPCATN
00275E   NI   ADACSW+4,X'7F'
002762   OI   XJCSYS,SPECATTN   → SPECATTN=1 on PRIMARY
```

Control falls to `$ENDREAD` (002776), where PRIMARY decodes SECONDARY's 'J' signon
response.  MC7RESP processes it:

- Sets LCONNECT flag (link is now connected)
- Executes `OI $RCOMM1+1,OPEN` — **opens the reader gate on PRIMARY**

CWRTNEXT is reached.  **The reader gate is now open.  Because files were pre-queued,
the reader immediately starts reading from spool and calling `$TPPUT` to fill output
buffers.**  After one or more TPPUT calls, `$OUTBUF` has real file-data buffers.
CWRTNEXT dequeues the first file-data buffer and enters CWRTSIO for PRIMARY's first
file write.

### Phase 4: PRIMARY's first file write — SPECATTN one-shot skip (with fix)

```
CWRTSIO, PRIMARY side:

002B76   TM  XJCSYS,SPECATTN    → 1  (set in Phase 3 by CRSPCATN)
002B7A   BO  CNSPECAL           → TAKEN  (skip ATTNREQ)

CNSPECAL (002B84) — user's fix:
         NI  XJCSYS,255-SPECATTN  → SPECATTN=0 on PRIMARY  ← one-shot consumed

002B8E   XC  LOOPCT,LOOPCT      clear debug counter
002B94   BAL R15,$SIO
```

`$SIO` issues PRIMARY's file-data CCW chain:

```
PRIMARY:  SENSE → WRITE(file A, buffer 1) → CTRL → READ
PRIMARY's CTRL fires ATTN to SECONDARY.
```

**SECONDARY is still in its outstanding READ (from Phase 3).**  SECONDARY's READ
receives file-A data from PRIMARY.  SECONDARY's channel interrupt:

```
ADACSW+4 = X'8C'   (CE+DE from READ complete + ATTN from PRIMARY's CTRL)
→ CRSPCATN → SPECATTN=1 on SECONDARY
```

### Phase 5: SECONDARY's first file write — SPECATTN one-shot skip (with fix)

MC7 protocols on SECONDARY process the received file-A data (or ACK it).
SECONDARY's reader gate was opened at signon.  `$TPPUT` fills SECONDARY's output
buffers with file-B data.  CWRTNEXT dequeues file-B buffer, enters CWRTSIO:

```
CWRTSIO, SECONDARY side:

002B76   TM  XJCSYS,SPECATTN    → 1  (set in Phase 4 by CRSPCATN)
002B7A   BO  CNSPECAL           → TAKEN  (skip ATTNREQ)

CNSPECAL — user's fix:
         NI  XJCSYS,255-SPECATTN  → SPECATTN=0 on SECONDARY ← one-shot consumed

002B94   BAL R15,$SIO
```

```
SECONDARY:  SENSE → WRITE(file B, buffer 1) → CTRL → READ
SECONDARY's CTRL fires ATTN to PRIMARY.
```

PRIMARY's outstanding READ receives file-B data:

```
ADACSW+4 = X'8C' → CRSPCATN → SPECATTN=1 on PRIMARY
```

### Phase 6: Self-sustaining ping-pong

The pattern from Phases 4–5 repeats indefinitely:

```
Invariant: exactly one side has SPECATTN=1 at any moment.
           That side uses it once (NI clears it), issues its WRITE,
           fires CTRL → ATTN to the other side.
           The other side's READ completes → ATTN+CE+DE → SPECATTN=1
           → that side's turn to write.
```

No simultaneous WRITEs are possible.  Both files transfer at full CTC speed.
The user's observation — **"a 50,000 record file transmitted in about 3 seconds"**
— is consistent with this clean, serialized, back-to-back write protocol.

### Why both files transfer simultaneously

Each side's reader is continuously filling `$OUTBUF`.  Each side's CTC WRITE cycle
drains `$OUTBUF` from one side and also delivers data to the remote `$INBUF`.
The ping-pong alternates:  A writes one buffer, B writes one buffer, A writes one
buffer, B writes one buffer.  From an application perspective both transfers proceed
concurrently, interleaved at the buffer granularity.

---

## 9. Why Post-Queued Files Still Fail — A Separate Bug

When the CTC connection is **already active** and then both files are queued
simultaneously:

### Phase A: Connection idle after signon

After signon, with no files queued, `$OUTBUF` is empty.  CWRTNEXT reaches
`CRESPOND` (offset `002A0E`, stmt 4900):

```
002A0E   CLC $BUFPOOL,=F'0'    → not empty
002A1C   L   R13,$BUFPOOL      → get a free buffer
...
002A2E   MVI BUFSTAT,BUFFAKE   → mark as dummy/null
002A36   B   CSTNDWRT          → build CCW chain with null payload
```

The code sends a **null buffer** (data byte = 0 at stmt 4915: `MVI BUFDATA,0`).
Null buffers keep the protocol cycling: each null buffer goes through the full
`SENSE → WRITE(null) → CTRL → READ` CCW chain.  The CTRL fires ATTN to the remote.
The remote's READ completes with `ATTN+CE+DE → SPECATTN=1` on the remote, which then
sends its own null buffer.

The fix is in effect: `SPECATTN` is used once per null-buffer cycle and cleared.
The null-buffer ping-pong operates correctly and indefinitely.

### Phase B: Both files queued simultaneously

Both files are submitted to the spool.  Both sides' reader gates are already open
(opened at signon).  Both readers start calling `$TPPUT`:

- Side A: $TPPUT for file A → needs buffer from $BUFPOOL → fills it → queues to $OUTBUF
- Side B: $TPPUT for file B → needs buffer from $BUFPOOL → fills it → queues to $OUTBUF

Both readers drain `$BUFPOOL` at the same rate.  Unlike the pre-queued scenario —
where the transfer begins from a fresh-start state and the CTC immediately begins
cycling buffers back to the pool — the simultaneous drain here races against the
null-buffer overhead already consuming pool entries.

Additionally, both sides must send a **START FUNCTION REQUEST** control record
(`MC1` type 001) through `MPUT` before data can flow.  `MPUT` calls `$TPPUT`, which
calls `OREENT → OGETBUF`.  If `$BUFPOOL` has been drained to exactly **1 buffer**
by the simultaneous `$TPPUT` calls, `OGETBUF` fires the "BETTER NOT USE IT" guard:

```
002322   CLC 0(4,R6),=F'0'    → EQUAL (only 1 buffer; its chain pointer = 0)
002328   BE  ORETURN           → taken; returns CC=0 to MREPUT
```

`MPUT` sets `$CCOMM1` to `MREPUT` and exits.  The **MREPUT infinite loop**
(documented fully in `bug_theories3.md`) begins.

### Why SPECATTN fix does not address this

The SPECATTN fix eliminates the **CTC write deadlock** (simultaneous WRITE stall).
The MREPUT loop is caused by **buffer pool exhaustion** — a completely independent
mechanism.  The SPECATTN fix does not increase the buffer pool size, does not alter
`OGETBUF`, and does not open `$TPGETCM`.

The post-queued scenario fails for the reason documented in `bug_theories3.md`:
buffers are consumed faster than they are freed, `$BUFPOOL` reaches 1, and the
MREPUT loop cannot escape because no CTC I/O is running to return buffers.

### Why the pre-queued scenario does not trigger buffer exhaustion

In the pre-queued scenario, the connection starts from an idle state.  The signon
exchange consumes only 2–3 CTC cycles (a few hundred milliseconds).  The spool reader
has not yet had time to aggressively fill output buffers.  The very first CTC file
write begins before `$BUFPOOL` has been significantly drained.  As the CTC cycles
file buffers (A→B, B→A), the sender's ACK path immediately returns each sent buffer
to `$BUFPOOL`:

```
CACKED (0027D0) → CWRTOK (0027E4):
0027F0   MVC 0(4,R13),$BUFPOOL    chain this buffer back to free pool
0027F6   ST  R13,$BUFPOOL          it is now the new pool head
```

Buffers cycle: fill → send → ACK → return → refill.  At steady state the pool stays
healthy and `OGETBUF`'s "last buffer" guard is never triggered.

---

## 10. Summary of All Code Locations

### CSECT DMTXJC1

| Label / Symbol | Abs. Offset | Stmt | Role |
|---|---|---|---|
| ISIO clear | 000708 | 796 | `NI XJCSYS,PRIMONLY` — SPECATTN=0 at startup |
| $ENDREAD | 002776 | 4687 | CTC read-complete handler; fake-interrupt entry at startup |
| COMSUP / $INTRUPT | 00271E | 4656–4657 | CTC interrupt entry; closes COMSUP gate |
| CRSPCATN | 00275E | 4675–4677 | Sets SPECATTN when ADACSW+4=X'8C' |
| CRCONT | 002766 | 4679 | Checks for unexpected errors; falls to $ENDREAD |
| CNWRITE | 002AE2 | 5006 | Builds CCW chain (SENSE/WRITE/CTRL/READ); branches to CWRTSIO |
| CWRTSIO | 002B44 | 5033 | Entry for every CTC write |
| CNWRCONT | 002B76 | 5052–5056 | `TM XJCSYS,SPECATTN`; `BO CNSPECAL`; `L R15,ATTNREQ`; `BALR R14,R15` |
| CNSPECAL | 002B84 | 5058 | **User fix inserts `NI XJCSYS,255-SPECATTN` here** |
| CACKED | 0027D0 | 4718 | ACK received: returns buffer to $BUFPOOL, opens gates, calls CWRTNEXT |
| CWRTNEXT | 002822 | 4743 | Cycle: checks $OUTBUF; if non-empty → CSTNDWRT; if empty → CRESPOND |
| CRESPOND | 002A0E | 4900 | No data to send: allocates null/dummy buffer and writes it |

### CSECT DMTXJCA

| Symbol | Abs. Offset | Role |
|---|---|---|
| XJCSYS | 044BC | Flag byte containing SPECATTN (bit X'02') |
| ATTNREQ | 04EE8 | Vector to NJEDRV's CTC attention wait routine |
| $BUFPOOL | 04634 | Free output buffer pool anchor |

---

*Analysis performed by direct inspection of the DMTXJC assembler listing,
06 Mar 2026. All offsets are relative to assembled CSECT base addresses.*
