# PROJECT BIBLE
## Predictive Adaptive QoS Control for 5G Emergency Communications
### Using NWDAF · LSTM · DQN · Guardrail on a Real 5G Core

> **This is the single source of truth for the entire project.**  
> If you are new, read this top to bottom before touching any code.  
> If something changes, update this document first.

---

## Table of Contents

1. [What We Are Building](#1-what-we-are-building)
2. [The Problem](#2-the-problem)
3. [Our Solution](#3-our-solution)
4. [Why This Is Novel](#4-why-this-is-novel)
5. [System Architecture](#5-system-architecture)
6. [The Codebase](#6-the-codebase)
7. [What We Add to the Code](#7-what-we-add-to-the-code)
8. [IEEE Paper Requirements](#8-ieee-paper-requirements)
9. [The 20-Day Execution Plan](#9-the-20-day-execution-plan)
10. [Experiments and Metrics](#10-experiments-and-metrics)
11. [Paper Structure](#11-paper-structure)
12. [Glossary](#12-glossary)

---

## 1. What We Are Building

**One sentence:**  
> A system that uses AI to protect emergency phone calls during disasters on a real 5G network — before the network gets congested, not after.

**The demo:**  
1. Start the 5G core (`mmt-studio-5g6g`) with ~20 normal users connected.
2. Simulate a disaster: suddenly inject 200 users + 10 emergency responders.
3. **Without our system:** The network chokes. Emergency calls drop.  
   **With our system:** The AI predicted the spike 30 seconds early, already throttled public users, and emergency calls never notice anything wrong.

**Target publication:** IEEE VTC / WCNC / ICC — 6–8 page conference paper.

---

## 2. The Problem

### Real-world context
During mass casualty events (earthquakes, terrorist attacks, wildfires), cellular networks experience **300–500% traffic spikes**. This causes:
- Emergency responder calls to fail
- 911 dispatch connections to drop
- Coordinated rescue operations to break down

This has been documented in real disasters (cite in paper: 2011 Japan Earthquake, 2013 Boston Marathon bombing, 2018 California wildfires).

### The technical gap
Current 5G networks use **static rules** defined in the PCF (Policy Control Function):  
- Everyone gets the same QoS until someone manually intervenes
- The network reacts *after* congestion already happened
- No automatic differentiation between an emergency call and someone streaming YouTube

**The gap we fill:** The NWDAF (analytics function) collects data, but there is no closed loop from analytics → policy change → enforcement. We build that loop.

---

## 3. Our Solution

### Four components working together

```
                ┌─────────────────────────────────────────────────────┐
                │              5G Core (mmt-studio-5g6g)               │
                │                                                       │
                │  [UEs] → [gNB] → [AMF] → [SMF] → [UPF]             │
                │                     ↓                                │
                │               [NWDAF]  ←── collects metrics          │
                │                  ↓                                   │
                │         HTTP POST /action                            │
                │                  ↓                                   │
                │        [Python ML Engine]                            │
                │         ├── LSTM: predicts congestion 30s ahead      │
                │         ├── DQN: decides what policy action to take  │
                │         └── Guardrail: hard safety rules             │
                │                  ↓                                   │
                │         HTTP POST /nwdaf-alert                       │
                │                  ↓                                   │
                │               [PCF]  ←── applies new QoS rules       │
                │                  ↓                                   │
                │         [SMF enforces on UPF]                        │
                └─────────────────────────────────────────────────────┘
```

### Component 1 — NWDAF (existing code, slightly extended)
- Collects: UE count, registration rate, packet drop rate every 30 seconds
- We add: a call to the ML engine after each collection cycle

### Component 2 — LSTM Predictor (new Python code)
- Input: last 10 data points (10 × 30s = 5 minutes of history)
- Output: probability of congestion in the next 30 seconds
- Trained on: the data we collect during Day 7 baseline experiments

### Component 3 — DQN Agent (new Python code)
- Input: current UE count, LSTM prediction, emergency UE count, current AMBR
- Output: one of 5 actions (no_change / mild_throttle / severe_throttle / block_video / restore)
- Learns: to maximize emergency call success rate while being fair to public users

### Component 4 — Guardrail (new Python code)
- Sits between DQN and PCF
- Hard rules the DQN CANNOT override:
  - Emergency UEs are NEVER throttled (3GPP ARP rule)
  - AMBR never goes below 64 kbps (SMS minimum)
  - If emergency success rate < 95%, only protective actions allowed

### Component 5 — PCF (existing code, heavily extended)
- 3-tier UE classification: emergency / public / restricted
- Static baseline controller (Days 5–7)
- Dynamic AMBR update from ML signals (Days 13–14)
- Admission control: block new connections when congested

---

## 4. Why This Is Novel

### What existing papers do
| Paper type | What they do | Problem |
|-----------|-------------|---------|
| Emergency QoS papers | Propose 5QI/ARP priority schemes | Simulated, not on real 5G core |
| NWDAF papers | Analytics and reporting only | No closed loop to PCF |
| DQN network papers | Resource allocation in slicing | ns-3 simulation, not real hardware |
| LSTM congestion papers | Predict congestion | No enforcement mechanism |

### What we do that is new
1. **LSTM predicts before congestion** — not reactive, predictive
2. **DQN learns the optimal throttle level** — not a fixed threshold
3. **Guardrail enforces formal 3GPP safety rules** — ML cannot violate spec
4. **Implemented on a real, running 5G Core** — not a simulation

**The combination of all four on a real system = novelty claim.**

One sentence for the paper abstract:
> "We present the first implementation of a predictive, safe, ML-driven QoS controller for emergency communications, combining LSTM congestion prediction, DQN policy optimization, and formal safety guardrails on a real 3GPP-aligned 5G Core."

---

## 5. System Architecture

### Network functions involved

| NF | Role | Where in code |
|----|------|---------------|
| AMF | Authenticates UEs, manages registration | `core/nf/amf/` |
| SMF | Creates PDU sessions, enforces QoS | `core/nf/smf/` |
| UPF | Forwards packets, applies AMBR | `core/nf/upf/` |
| PCF | Sets QoS policy rules | `core/nf/pcf/` ← **our main work** |
| NWDAF | Collects analytics, detects load | `core/nf/nwdaf/` |
| ML Engine | Predicts + decides policy actions | `core/ai_engine/ml_engine/` ← **new** |

### Data flow during a disaster (step by step)

```
T=0s:   Normal operation, 20 public UEs connected
T=60s:  200 UEs try to connect simultaneously
T=60s:  NWDAF collection loop fires — sees UE count spike
T=60s:  NWDAF calls ML engine: POST /action
T=60s:  LSTM says: "90% probability of congestion in 30s"  ← predictive
T=60s:  DQN says: "action = throttle_public_video_severe"
T=60s:  Guardrail checks: safe? Yes → forward to PCF
T=61s:  PCF receives POST /nwdaf-alert
T=61s:  PCF walks all active sessions, updates AMBR for public UEs
T=61s:  Emergency UEs: untouched, still 20 Mbps, 5QI=1, ARP=1
T=61s:  Public UEs: AMBR dropped to 512 kbps, video blocked
T=90s:  Emergency UEs register and make calls — 100% success rate
```

Without ML (baseline):
```
T=60s:  UEs connect, count crosses 150 → hardcoded throttle kicks in
T=90s:  Emergency UEs try to connect — but throttle happened too late,
        some connections already failed during the spike
```

---

## 6. The Codebase

### Repository: `mmt-studio-5g6g`

```
mmt-studio-5g6g/
├── core/                    ← 5G Core (Go)
│   ├── nf/
│   │   ├── amf/             ← Access & Mobility Management
│   │   ├── smf/             ← Session Management
│   │   ├── upf/             ← User Plane (packet forwarding)
│   │   ├── pcf/             ← Policy Control ← OUR MAIN CODE
│   │   │   ├── pcf.go       ← ClassifyUE(), PCC rules, policy engine
│   │   │   ├── api.go       ← Dashboard/panel API
│   │   │   └── smpolicy/
│   │   │       ├── smpolicy.go          ← Create/Update/Delete sessions
│   │   │       ├── baseline_controller.go  ← NEW: 3-tier static policy
│   │   │       └── ml_action.go         ← NEW: /nwdaf-alert + ApplyMLAction
│   │   └── nwdaf/           ← Analytics (we add ML callback here)
│   ├── db/
│   │   └── seed/
│   │       ├── baseline.yaml  ← UE roster (add emergency/restricted buckets)
│   │       └── services.go    ← QoS service catalog (conv_voice, conv_video, etc.)
│   └── ai_engine/
│       └── ml_engine/       ← NEW: Python ML microservice
│           ├── app.py       ← Flask REST API
│           ├── lstm_predictor.py
│           ├── dqn_agent.py
│           └── guardrail.py
├── tester/                  ← 5G network tester (Python)
│   ├── config/
│   │   └── baseline.yaml    ← MUST match core's baseline.yaml
│   └── src/
│       └── baseline.py      ← Reads baseline.yaml, exposes IMSI lists
└── 1725/                    ← Our working folder (notes, plans, results)
    ├── PROJECT_BIBLE.md     ← This file
    ├── days5_7_plan.md      ← PCF baseline code guide
    ├── 20day_paper_plan.md  ← Original full plan
    └── baseline_results.csv ← Results from Day 7 experiment (fill in)
```

### Key existing structs (Go)

**`PCCRule`** — one QoS rule for one service:
```go
type PCCRule struct {
    ServiceName  string  // "conv_voice", "conv_video", "ims_signalling"
    FiveQI       int     // QoS class (1=voice, 2=video, 5=signalling, 9=best-effort)
    ResourceType string  // "GBR" or "NonGBR"
    ArpPriority  int     // 1=highest (never dropped), 15=lowest (dropped first)
    GBRULKbps    int     // guaranteed bandwidth floor (upload)
    GBRDLKbps    int     // guaranteed bandwidth floor (download)
    MBRULKbps    int     // maximum bandwidth ceiling (upload)
    MBRDLKbps    int     // maximum bandwidth ceiling (download)
    IsDefault    bool    // is this the always-on default flow?
}
```

**`SmPolicyDecision`** — full policy for one PDU session:
```go
type SmPolicyDecision struct {
    PccRules      []PCCRule  // which services are allowed
    DefaultQFI    uint8      // QoS Flow Identifier for the default flow
    Default5QI    int        // default QoS class
    SessionAMBRUL int        // max total upload bandwidth (kbps)
    SessionAMBRDL int        // max total download bandwidth (kbps)
    ChargingMethod string
    RevalidationTime time.Time
    SmPolicyCtxRef string
}
```

### Existing QoS service catalog (`db/seed/services.go`)

| Service Name | 5QI | Type | ARP | GBR UL | GBR DL | MBR UL | MBR DL |
|---|---|---|---|---|---|---|---|
| `conv_voice` | 1 | GBR | 1 | 64 kbps | 64 kbps | 128 kbps | 128 kbps |
| `conv_video` | 2 | GBR | 2 | 1000 kbps | 1000 kbps | 4000 kbps | 4000 kbps |
| `ims_signalling` | 5 | NonGBR | 1 | — | — | — | — |
| `default_data` | 9 | NonGBR | 9 | — | — | — | — |

### Existing UE roster (`tester/config/baseline.yaml`)

| Bucket | Count | IMSI Start | DNN | Purpose |
|--------|-------|-----------|-----|---------|
| `embb-bulk` | 100 | `001011234560001` | internet, ims | Regular users |
| `miot-pool` | 16 | `001011234560101` | iot | IoT devices |
| `urllc-pool` | 8 | `001011234560117` | internet | Low-latency |
| `multi-slice` | 4 | `001011234560125` | internet, ims, mcx | Multi-slice |

**We add:**

| Bucket | Count | IMSI Start | Tier |
|--------|-------|-----------|------|
| `emergency-pool` | 10 | `001010000000001` | emergency |
| `restricted-pool` | 20 | `001019999990001` | restricted |

---

## 7. What We Add to the Code

### File 1: `core/nf/pcf/pcf.go` — ADD `ClassifyUE()`

```go
func ClassifyUE(supi string) string {
    imsi := supi
    if strings.HasPrefix(supi, "imsi-") { imsi = supi[5:] }
    // Emergency: IMSI 001010000000001 – 001010000000010
    if strings.HasPrefix(imsi, "0010100000000") {
        tail := imsi[len("0010100000000"):]
        if tail >= "01" && tail <= "10" { return "emergency" }
    }
    // Restricted: IMSI 001019999990001 – 001019999990020
    if strings.HasPrefix(imsi, "00101999999000") { return "restricted" }
    return "public"
}
```

### File 2: `core/nf/pcf/smpolicy/baseline_controller.go` — NEW

Contains:
- `activeUECount` — atomic counter of live sessions
- `TrackUECreate()` / `TrackUEDelete()` — increment/decrement counter
- `AdmissionAllowed(supi)` — returns false when network is congested
- `BaselineDecision(supi)` — returns full `SmPolicyDecision` with PCC rules

**3-tier policy:**
| Tier | Services allowed | AMBR | ARP | Admission |
|------|----------------|------|-----|-----------|
| Emergency | ims_signalling + conv_voice + conv_video | 20 Mbps | 1 | Always |
| Public | ims_signalling + conv_voice | 512 kbps | 8 | Block if UE > 200 |
| Restricted | ims_signalling only (SMS) | 64 kbps | 14 | Block if UE > 150 |

### File 3: `core/nf/pcf/smpolicy/ml_action.go` — NEW

Contains:
- `MLAction` struct — JSON the ML engine sends
- `ApplyMLAction(action)` — walks all active sessions, updates AMBR via `PushNotify`
- `NWDAFAlertHandler(w, r)` — HTTP POST `/nwdaf-alert` endpoint

### File 4: `core/nf/nwdaf/nwdaf.go` — MODIFY (Day 12, after NWDAF fixed)

Add `queryMLEngine()` — after each collection cycle, POST analytics to Python.

### File 5: `core/ai_engine/ml_engine/app.py` — NEW

Flask REST API:
- `POST /predict` → LSTM returns congestion probability
- `POST /action` → DQN + Guardrail returns throttle action

### File 6: `core/ai_engine/ml_engine/lstm_predictor.py` — NEW

PyTorch LSTM:
- Input: sequence of (ue_count, registration_rate, drop_rate) × 10 steps
- Output: float 0.0–1.0 (probability of congestion in 30 seconds)

### File 7: `core/ai_engine/ml_engine/dqn_agent.py` — NEW

DQN with 5 actions:
- 0: no_change
- 1: throttle_public_mild (5 Mbps)
- 2: throttle_public_severe (512 kbps)
- 3: block_public_video
- 4: restore_normal

Reward function:
```python
reward = 10 × emergency_success_rate
       - 2  × public_drop_rate
       - 1  × unnecessary_throttle_events
```

### File 8: `core/ai_engine/ml_engine/guardrail.py` — NEW

Hard rules:
1. Emergency UEs never throttled (3GPP TS 23.501 §5.7.2.2 ARP)
2. AMBR never below 64 kbps (SMS minimum)
3. If emergency success < 95%, only allow protective actions

---

## 8. IEEE Paper Requirements

### Conference target options
| Conference | Deadline (check each year) | Pages | URL |
|-----------|--------------------------|-------|-----|
| IEEE VTC Spring | ~November | 5 | vtc.ieee.org |
| IEEE VTC Fall | ~March | 5 | vtc.ieee.org |
| IEEE WCNC | ~September | 6 | wcnc.ieee.org |
| IEEE ICC | ~October | 6 | icc.ieee.org |

Check current deadlines at: **cfp.ieee.org**

### What IEEE requires

1. **Original contribution** — must be unpublished, not under review elsewhere
2. **Related work section** — cite at least 8–10 relevant papers
3. **Evaluation with real metrics** — tables + graphs from actual experiments
4. **IEEE template** — use the official `.docx` or LaTeX template
5. **6–8 pages** for most conferences (2-column format)
6. **PDF submission** via EDAS or IEEE CMS portal

### What your paper must prove

| Claim | What proves it |
|-------|---------------|
| "We implemented on a real 5G Core" | Cite mmt-studio-5g6g, show Go code in paper |
| "LSTM predicts 30s ahead" | Graph: detection latency vs reactive system |
| "DQN is fairer than greedy" | Graph: unnecessary throttle rate |
| "Emergency calls protected" | Table: success rate ≥ 95% at all times |
| "Guardrail is formally safe" | List the 3 invariants + show they hold |

### Minimum results needed for acceptance

| Metric | Minimum to claim novelty |
|--------|-------------------------|
| Emergency call success rate improvement | +10% over baseline |
| Congestion detection time advantage | At least 20s earlier than reactive |
| Unnecessary throttle reduction vs greedy | At least 30% fewer events |

---

## 9. The 20-Day Execution Plan

### Overview

| Phase | Days | Goal |
|-------|------|------|
| Understand | 1–4 | Read the codebase, understand data flow |
| Baseline | 5–7 | Write PCF 3-tier policy, measure static baseline |
| ML Engine | 8–12 | Build Python LSTM + DQN + Guardrail |
| Integration | 13–15 | Wire ML → PCF, end-to-end test |
| Experiments | 16–18 | Run 3 experiments, collect graphs |
| Paper | 19–20 | Write and submit |

### Day-by-day

**Day 1** — Read `core/nf/nwdaf/nwdaf.go`. Understand `collectionLoop()` and what data it collects. Write down what fields are available for ML features.

**Day 2** — Read `core/nf/pcf/smpolicy/smpolicy.go`. Understand `SmPolicyDecision`. Know which fields control bandwidth. Answer: "which field throttles a user?"

**Day 3** — Read `nwdaf.go processSubscriptions()`. Find the gap: NWDAF notifies subscribers, but PCF doesn't subscribe. This is what you are building.

**Day 4** — Run the tester. Register 20 UEs. Take a screenshot. This is Figure 1 in your paper.

**Day 5** — Add `ClassifyUE()` to `core/nf/pcf/pcf.go`.

**Day 6** — Create `core/nf/pcf/smpolicy/baseline_controller.go`. Full 3-tier policy with PCC rules, GBR/MBR, ARP, admission control. Run `go build` — must pass.

**Day 7** — Run disaster scenario with tester. 20 UEs → spike 130 UEs at T=60s. Record metrics for 300 seconds. Save as `1725/baseline_results.csv`. These numbers = Table II in your paper.

**Day 8** — Create `core/ai_engine/ml_engine/`. Set up Python venv. Write `app.py` skeleton with two endpoints.

**Day 9** — Write `lstm_predictor.py`. Train it on Day 7 data. Test: does it predict congestion?

**Day 10** — Write `dqn_agent.py`. Define state space, action space, reward function.

**Day 11** — Write `guardrail.py`. Three hard rules. Test: can you make the guardrail block an unsafe DQN action?

**Day 12** — Modify `core/nf/nwdaf/nwdaf.go`. Add `queryMLEngine()` that POSTs to Python `/action`.

**Day 13** — Create `core/nf/pcf/smpolicy/ml_action.go`. The `/nwdaf-alert` endpoint + `ApplyMLAction()`.

**Day 14** — Wire everything together. Check: NWDAF → ML → PCF chain builds and compiles.

**Day 15** — End-to-end integration test. Start Python + 5G core + tester. Inject 200 UEs. Watch the full chain fire in logs. Fix bugs.

**Day 16** — Experiment 1: full disaster scenario. 3 runs, average results. Save CSV + 3 graphs.

**Day 17** — Experiment 2: prediction horizon ablation (reactive vs 15s vs 30s predictive).

**Day 18** — Experiment 3: DQN vs greedy vs static baseline.

**Day 19** — Write paper Sections 1–4 in Overleaf.

**Day 20** — Write Sections 5–7, format, submit.

---

## 10. Experiments and Metrics

### Experiment 1 — Disaster Scenario (Day 16)

**Setup:**
- T=0: 20 public UEs
- T=60s: inject 100 public + 10 emergency + 20 restricted simultaneously
- Run 300 seconds. Repeat 3 times.

**Measure every 10 seconds:**
| Metric | How to measure |
|--------|---------------|
| Emergency call success rate (%) | Did emergency UEs get PDU session + IMS registration? |
| Emergency call setup time (ms) | Time from IMSI attach to IMS REGISTER complete |
| Public avg AMBR (kbps) | Read from PCF logs |
| UPF packet drop rate (%) | NWDAF QOS_SUSTAINABILITY metric |
| Time to detect + respond (s) | NWDAF collection time → PCF update time |

**Expected outcome:**
- ML system: emergency success ≥ 95% at all times
- Baseline: emergency success drops to ~75% during the spike window

### Experiment 2 — Prediction Horizon (Day 17)

Three variants:
- **Reactive:** Act only when `drop_rate > 0.05` (congestion already happened)
- **15s predictive:** LSTM horizon = 15 seconds
- **30s predictive:** LSTM horizon = 30 seconds (default)

**Metric:** Emergency call setup time during T=60–90s window.

### Experiment 3 — DQN vs Greedy vs Static (Day 18)

Three variants:
- **Static baseline (Day 7):** Fixed threshold throttle
- **Greedy:** Always throttle to maximum when congestion detected
- **DQN + Guardrail:** Learned optimal action

**Metric:** Unnecessary throttle events (public UEs throttled when congestion wasn't severe).

---

## 11. Paper Structure

### Section 1 — Introduction (~1 page)
- Hook: disaster + network failure statistic
- Problem: static 5G QoS reacts too late
- Contributions (4 bullet points, one per pillar)
- Paper organization

### Section 2 — Background (~1 page)
- 3GPP 5G QoS: 5QI, ARP, GBR, AMBR, AMBR (from your `gptQos.md`)
- NWDAF: what it collects, subscription mechanism
- PCF: N7 interface, SmPolicyDecision

### Section 3 — Related Work (~0.5 pages)
- 5–8 papers, 1 sentence each
- End with: "Unlike prior work, we combine X + Y + Z on a real 5G Core testbed."

**Search terms for Google Scholar:**
- "NWDAF emergency 5G QoS machine learning"
- "DQN network slicing resource management 5G"
- "LSTM 5G congestion prediction"
- "emergency communication network slicing"

### Section 4 — System Design (~1.5 pages)
- Architecture diagram (from Section 5 of this document)
- LSTM design: input features, architecture, training
- DQN design: state space, action space, reward
- Guardrail: 3 invariants mapped to 3GPP spec citations
- PCF integration: 3-tier classification, admission control

### Section 5 — Implementation (~0.5 pages)
- "Implemented on mmt-studio-5g6g, a 3GPP TS 23.288-aligned open-source 5G Core"
- Go for core NFs, Python + PyTorch for ML, REST for integration
- Lines of code per component

### Section 6 — Evaluation (~2 pages)
- 3 figures:
  - **Fig 1:** Emergency call success rate over time (baseline vs ML system)
  - **Fig 2:** Congestion detection latency (reactive vs predictive)
  - **Fig 3:** Unnecessary throttle events (static vs greedy vs DQN)
- 1 table: overall performance comparison
- 1 paragraph per figure explaining the result

### Section 7 — Conclusion (~0.25 pages)
- "We proposed... We implemented... Results show X% improvement..."
- Future work: multi-cell, real radio

---

## 12. Glossary

| Term | What it means | Code location |
|------|--------------|---------------|
| **5QI** | QoS class number (1=voice, 2=video, 9=best-effort) | `PCCRule.FiveQI` |
| **ARP** | Who gets kicked when network is full (1=protected, 15=first to go) | `PCCRule.ArpPriority` |
| **AMBR** | Maximum total bandwidth a UE can use | `SmPolicyDecision.SessionAMBRUL/DL` |
| **GBR** | Guaranteed minimum bandwidth floor for one service | `PCCRule.GBRULKbps` |
| **MBR** | Maximum bandwidth ceiling for one service | `PCCRule.MBRULKbps` |
| **QFI** | ID number on a packet flow (so UPF knows which rule to apply) | `SmPolicyDecision.DefaultQFI` |
| **PCC Rule** | One policy entry: which service + what QoS | `PCCRule` struct |
| **PDU Session** | A UE's data connection (like a phone call channel for data) | Created by SMF |
| **NWDAF** | The analytics brain — observes network load | `core/nf/nwdaf/` |
| **PCF** | The policy brain — decides what QoS each UE gets | `core/nf/pcf/` |
| **SMF** | Session manager — creates sessions and enforces PCF rules | `core/nf/smf/` |
| **UPF** | Packet forwarder — the actual bandwidth enforcement point | `core/nf/upf/` |
| **LSTM** | ML model that reads a time sequence and predicts the future | `ml_engine/lstm_predictor.py` |
| **DQN** | ML agent that learns the best action to take | `ml_engine/dqn_agent.py` |
| **Guardrail** | Safety layer between DQN and PCF — enforces 3GPP rules | `ml_engine/guardrail.py` |
| **N7** | The interface between PCF and SMF (TS 29.512) | `smpolicy.go` |
| **GBR flow** | A QoS flow with a guaranteed minimum rate (like a dedicated lane) | 5QI 1 and 2 |
| **NonGBR flow** | A best-effort flow, no guarantee (shares available capacity) | 5QI 5 and 9 |
| **Pre-emption** | When ARP=1 UE can kick ARP=14 UE off the network | Admission control |
| **SUPI** | Subscriber ID (IMSI with prefix "imsi-XXXXXXXXX") | `SmPolicyContextData.SUPI` |

---

## Current Status

| Item | Status |
|------|--------|
| Days 1–4: Understand codebase | ✅ Done |
| Day 5: ClassifyUE() | 🔲 Write in pcf.go |
| Day 6: baseline_controller.go | 🔲 New file |
| Day 6: Wire into smpolicy.go | 🔲 Modify |
| Day 7: Run baseline experiment | 🔲 Need tester running |
| Days 8–12: Python ML engine | 🔲 After NWDAF fixed |
| Days 13–14: PCF ML integration | 🔲 ml_action.go |
| Day 15: End-to-end test | 🔲 |
| Days 16–18: Experiments | 🔲 |
| Days 19–20: Write paper | 🔲 |

---

*Last updated: 2026-06-29*  
*Target: IEEE VTC / WCNC / ICC 2026*  
*Stack: mmt-studio-5g6g (Go) + Python ML microservice*
