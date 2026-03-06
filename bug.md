# CTC Bidirectional Transfer Bug – Top 5 Potential Causes

---

## 📑 Table of Contents

| # | Theory | Jump to |
|---|--------|---------|
| 1 | Block Control Byte (BCB) Desynchronization | [→ Theory 1](#-theory-1-block-control-byte-bcb-desynchronization) |
| 2 | Shared Buffer Pool Exhaustion | [→ Theory 2](#-theory-2-shared-buffer-pool-exhaustion) |
| 3 | Function Control Sequence (FCS) Collision | [→ Theory 3](#-theory-3-function-control-sequence-fcs-collision) |
| 4 | ACK/Response Handling Race | [→ Theory 4](#-theory-4-ackresponse-handling-race) |
| 5 | CTC Read/Write CCW & CBUFFER Ownership | [→ Theory 5](#-theory-5-ctc-readwrite-ccw--cbuffer-ownership) |

**Quick links:** [Bug Summary](#-bug-summary) · [Debug Aids](#-debug-aids-already-present) · [Investigation Order](#-recommended-investigation-order)

---

## 🐛 Bug Summary

> **When:** Sending files in **both directions simultaneously** over a CTCNJE connection  
> **Result:** CTC driver enters an endless loop  
> **Works:** Single-direction transfers  
> **Where:** COMSUP (communications supervisor) interrupt handler — CYCLE/CYCLE1 loop (ACKs, NAKs, incoming data)

---

## 🔴 Theory 1: Block Control Byte (BCB) Desynchronization

<a id="theory-1"></a>

| Field | Value |
|-------|-------|
| **Location** | `CBCBCNTO` / `CBCBCNTI` in CINBUF and CBCBCHEK |
| **Listing** | DMTXJC pages 127–128 |
| **Offsets** | 002932–00293E |

**Code:**

```asm
002932 D200 9CF8 D009 04CF8 00009 4827 MVC CBCB(1),BUFBCB GET BCB COUNT
002938 D500 9CCD 9CF8 04CCD 04CF8 4828 CLC CBCBCNTI(1),CBCB DOES RECEIVED MATCH EXPECTED
00293E 4770 A53E 029B6 4829 BNE CBCBCHEK BR IF NO
```

**Why it matters:** BCB correlates sent and received blocks. With bidirectional traffic:
- Both sides send blocks and expect matching BCB sequences
- Single pair of counters (`CBCBCNTO`, `CBCBCNTI`) may not track correctly when:
  - Side A sends block N while Side B sends block M
  - CTC delivers them interleaved
  - BCB for "our" receive gets overwritten or confused with "their" send

> ⚠️ **Symptom:** Mismatch in `CBCBCHEK` → retries or wrong branching → loop

---

## 🟠 Theory 2: Shared Buffer Pool Exhaustion

<a id="theory-2"></a>

| Field | Value |
|-------|-------|
| **Location** | `$BUFPOOL`, `$INBUF`, `$OUTBUF`, GASSIGN / GSWITCH |
| **Listing** | DMTXJC pages 110–111, 117–118 |
| **Offsets** | 00230C–002362, 0025B6–0025DE |

**Code:**

```asm
00231E 5860 9634 04634 4168 L R6,$BUFPOOL GET FIRST BUFFER ADDR
002322 D503 6000 AC28 00000 030A0 4169 CLC 0(4,R6),=F'0' ONLY ONE LEFT????
002328 4780 BE7C 022F4 4170 BE ORETURN YEP...BETTER NOT USE IT
...
0025B6 94DF 9D0C 04D0C 4461 NI BUFSYNSW,255-GDQBUFS ALLOW DEQUEUING TRY
0025BA D203 D058 6000 00058 00000 4462 MVC TCTBUFER,0(R6) UPDATE CHAIN
0025C0 45E0 A16A 025E2 4463 BAL R14,GASSIGN GO RE-ASSIGN BUFFER
```

**Why it matters:** With simultaneous RX and TX:
- Reader (RCT) and Printer/Punch (PCT) both consume buffers
- `$TPGET` dequeues from `$INBUF` → GASSIGN → TCTs
- `$TPPUT` gets buffers from `$BUFPOOL`
- Buffers not returned to the right chain, or `TCTBUFER` / `TCTBUFLM` / `TCTBUFCT` out of sync → spin waiting for buffers stuck on wrong queue

> ⚠️ **Symptom:** `OGETBUF` or `GWAIT` repeatedly failing, or `GDQ` looping without progress

---

## 🟡 Theory 3: Function Control Sequence (FCS) Collision

<a id="theory-3"></a>

| Field | Value |
|-------|-------|
| **Location** | `$FCSIN`, `$FCSOUT`, `CFCSTEMP`, CYCLE1 FCS test |
| **Listing** | DMTXJC pages 125–126 |
| **Offsets** | 00283E–002854 |

**Code:**

```asm
00283E D201 9CC4 9646 04CC4 04646 4753 MVC CFCSTEMP(2),$FCSIN MOVE RECEIVED TO TEMP AREA
002844 D401 9CC4 9CC2 04CC4 04CC2 4754 NC CFCSTEMP(2),CFCSSTD SETUP FOR TEST
00284A D701 9CC4 9CC2 04CC4 04CC2 4755 XC CFCSTEMP(2),CFCSSTD TEST AGAINST STANDARD
002850 4740 A424 0289C 4756 BC 4,CWAITBIT STREAM IS NEGATED
002854 4757 CYCLE1 EQU *
```

**Why it matters:** FCS controls which streams (Reader, Printer, Punch, Console) may send:
- Both sides negotiate FCS
- `$FCSIN` updated by incoming block while processing outgoing → FCS test fails or forces `CWAITBIT`
- `CWAITBIT` sets `$TPPNONE` → stops buffering; other side keeps sending → deadlock or tight loop

> ⚠️ **Symptom:** Repeated branch to `CWAITBIT`, or oscillation between CYCLE and CWAITBIT

---

## 🟢 Theory 4: ACK/Response Handling Race

<a id="theory-4"></a>

| Field | Value |
|-------|-------|
| **Location** | CACKED, CWRTOK, CWRTNEXT, CYCLE loop |
| **Listing** | DMTXJC pages 123–125 |
| **Offsets** | 0027D0–002836 |

**Code:**

```asm
0027D0 4718 CACKED DS 0H ACKNOWLEDGEMENT WAS ACK
...
0027E4 4724 CWRTOK DS 0H
...
002822 4743 CWRTNEXT EQU * ENTRY TO START NEXT WRITE
002822 9104 9D0C 04D0C 4744 TM BUFSYNSW,CACKSW WAS AN ACK RECEIVED?
002826 4710 A3BA 02832 4745 BO CNOD YES
00282A 45F0 A644 02ABC 4746 BAL R15,CSETCOM MAKE A COMMUTATOR PASS
00282E 45F0 A644 02ABC 4747 BAL R15,CSETCOM AND ANOTHER
02832 4748 CNOD EQU *
002832 94FB 9D0C 04D0C 4749 NI BUFSYNSW,255-CACKSW RESET SWITCH
02836 4750 CYCLE EQU * COMMUTATOR CYCLE POINT
```

**Why it matters:** When both sides send:
- Interrupt may signal **read** completion while processing **ACK** for write
- `CBUFFER` and `BUFSTAT` may point to wrong buffer (read vs write)
- Assumed sequence: write → ACK → next write; interleaved traffic can treat read buffer as ACK'd write
- `CSETCOM` called twice after ACK; inconsistent commutator state → loop without progress

> ⚠️ **Symptom:** Endless CYCLE → CACKED/CWRTOK → CWRTNEXT → CYCLE, no buffer sent or released

---

## 🟣 Theory 5: CTC Read/Write CCW & CBUFFER Ownership

<a id="theory-5"></a>

| Field | Value |
|-------|-------|
| **Location** | `CBUFFER`, `$ENDREAD`, CINBUF path |
| **Listing** | DMTXJC pages 122–123, 127 |
| **Offsets** | 00272E, 002776, 0028A4 |

**Code:**

```asm
00272E 58D0 9CBC 04CBC 4661 L R13,CBUFFER GET CURRENT BUFFER ADDR
...
002776 4687 $ENDREAD DS 0H EXTERNAL ENTRY POINT
...
0028A4 4789 CINBUF DS 0H *
0028A4 9101 D006 00006 4790 TM BUFSTAT,BUFFAKE IS THIS A DUMMY BUFFER?
```

**Why it matters:** CTC is full-duplex on one device. Interrupt can signal:
- **Write** completion (our send)
- **Read** completion (their send)

`CBUFFER` should point to the buffer for the completed I/O. With overlapping reads and writes:
- Read completion leaves `CBUFFER` at read buffer while logic expects write buffer (or vice versa)
- `BUFSTAT`/`BUFFAKE` and CINBUF vs CACKED branching depend on correct buffer identity
- Wrong `CBUFFER` → repeated processing of same buffer or wrong type → loop

> ⚠️ **Symptom:** Wrong branch between CINBUF and CACKED, or repeated handling of same buffer

---

## 🔧 Debug Aids Already Present

<a id="debug-aids"></a>

| Item | Details |
|------|---------|
| **LOOPCT** | Offset 045CC in DMTXJCA; cleared in `$SIO` (RSIO) and SIGNOFF |
| **XC LOOPCT,LOOPCT** | RSIO entry ~002DBE, SIGNOFF ~002FCE |

Loop is in the I/O / COMSUP path. Instrument CYCLE, CACKED, CINBUF, and CBCBCHEK (e.g., increment LOOPCT and exit after 100 iterations) to narrow down the cause.

---

## 📋 Recommended Investigation Order

<a id="investigation-order"></a>

| Priority | Theory | Action |
|:--------:|--------|--------|
| **1** | [Theory 5](#-theory-5-ctc-readwrite-ccw--cbuffer-ownership) | Verify `CBUFFER` and buffer type (read vs write) at each interrupt |
| **2** | [Theory 4](#-theory-4-ackresponse-handling-race) | Trace `CACKSW` and `BUFSYNSW` through CACKED → CWRTNEXT → CYCLE |
| **3** | [Theory 1](#-theory-1-block-control-byte-bcb-desynchronization) | Log `CBCBCNTO` / `CBCBCNTI` / `CBCB` on each block exchange |
| **4** | [Theory 3](#-theory-3-function-control-sequence-fcs-collision) | Log `$FCSIN` and CYCLE/CWAITBIT path |
| **5** | [Theory 2](#-theory-2-shared-buffer-pool-exhaustion) | Trace buffer chains (`$BUFPOOL`, `$INBUF`, `$OUTBUF`) and TCT counts during bidirectional transfer |

---

*[↑ Back to top](#ctc-bidirectional-transfer-bug--top-5-potential-causes)*
