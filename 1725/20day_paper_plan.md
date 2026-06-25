# 20-Day IEEE Paper Plan
## "Predictive Adaptive QoS Control for 5G Emergency Communications Using NWDAF, LSTM, DQN, and Guardrail"

**Target Venue:** IEEE VTC / WCNC / ICC (conference proceedings, 6–8 pages)
**Stack:** mmt-studio-5g6g (Go 5G Core) + Python ML microservice

---

## What You Are Building in One Sentence

> A system where the NWDAF collects network stats, your LSTM predicts congestion
> before it peaks, your DQN decides what policy to apply (throttle public users,
> protect emergency users), and the Guardrail ensures the ML never violates
> 3GPP safety rules — all demonstrated on a real running 5G core.

---

## The Four Pillars of Your Paper

| Pillar | What it does | Where in the code |
|--------|-------------|-------------------|
| **LSTM** | Predicts congestion N seconds ahead | Python microservice |
| **DQN** | Learns optimal policy action | Python microservice |
| **Guardrail** | Hard-blocks unsafe actions | Python microservice |
| **PCF Integration** | Applies decisions to real UE flows | `core/nf/pcf/` |

---

## Days 1–4 : Understand the Existing Code (No new code yet)

### Day 1 — Understand NWDAF Data Collection
**File:** `core/nf/nwdaf/nwdaf.go` + `core/nf/nwdaf/analytics/analytics.go`

**Tasks:**
- Read `collectionLoop()` in `nwdaf.go` (line 303). This runs every 30 seconds,
  calls `collectors.CollectAll()`, and stores data into `nwdaf_data_points` table.
- Read `ComputeAnalytics()` in `analytics.go` (line 78). Notice it supports:
  - `NF_LOAD` — registration/session rates from AMF/SMF
  - `UE_MOBILITY` — how many UEs are connected
  - `QOS_SUSTAINABILITY` — packet drop rate from UPF
  - `SLICE_LOAD` — session count per network slice
- Understand `AnalyticsResult` struct (line 63). The `Result` map is the payload
  your ML engine will receive.
- **Write down:** What fields inside `NF_LOAD` and `UE_MOBILITY` can serve as
  your congestion features? (hint: `avg_registration_rate`, `peak_ues`,
  `drop_rate` from QOS_SUSTAINABILITY)
- **End of Day Goal:** You can explain what data the NWDAF currently produces
  and why it is useful for predicting congestion.

### Day 2 — Understand PCF Policy Structures
**File:** `core/nf/pcf/smpolicy/smpolicy.go`

**Tasks:**
- Find `SmPolicyDecision` struct (around line 60–90). These are the fields
  your DQN will modify:
  - `SessionAMBRUL` / `SessionAMBRDL` — per-user bandwidth cap in kbps
  - `Default5QI` — the QoS priority class
  - `DefaultQFI` — identifies the QoS flow
- Find `SmPolicyContextData` struct. This is the UE context the PCF receives
  from the SMF when a session starts. It tells you who the UE is (SUPI/IMSI)
  and what service they are requesting.
- Read `core/nf/pcf/pcf.go` — find where `SmPolicyDecision` is returned to
  the SMF. This is the enforcement point.
- **Write down:** If you wanted to throttle a public user from 20 Mbps to 2 Mbps,
  which field do you change and to what value?
- **End of Day Goal:** You understand exactly which Go struct fields map to
  which QoS concepts (AMBR, 5QI, ARP).

### Day 3 — Understand the NWDAF → PCF Notification Path
**Files:** `core/nf/nwdaf/nwdaf.go` (processSubscriptions, sendNotification)

**Tasks:**
- Read `processSubscriptions()` (line 359). The NWDAF fires a callback HTTP POST
  to any NF that subscribed. Currently the PCF does NOT subscribe — there is no
  wired NWDAF→PCF notification in the existing code.
- This is your most important finding: **the NWDAF→PCF closed loop is not
  wired yet.** You need to build it. This is legitimate research work.
- Read `sendNotification()` (line 417). It sends a JSON payload with
  `analytics_id`, `result`, and `timestamp` to `callback_url`.
- **Write down:** What URL would the PCF need to expose to receive NWDAF alerts?
  Design the JSON schema for the alert your ML engine will send.
- **End of Day Goal:** You have a clear picture of the gap you are filling.

### Day 4 — Study the Tester (Simulation Environment)
**Directory:** `tester/`

**Tasks:**
- Read `tester/README.md`. Understand how to launch UE pools and configure
  UE count.
- Find how to configure UE priority/type (Emergency vs Public). You will need
  to add a UE "class" field if it doesn't exist.
- Understand how to run a "disaster scenario": spawn 200 public UEs suddenly,
  plus 10 emergency UEs, and measure call success rate.
- **End of Day Goal:** You can run the tester and see UEs attach to the 5G
  core. Take a screenshot — this is figure 1 in your paper.

---

## Days 5–7 : Build the Baseline (Static Controller — No ML)

> **Why:** A paper without a baseline is just a demo. The baseline is the
> thing your ML system has to beat. Without it, reviewers will reject the paper.

### Day 5 — Add UE Classification to PCF
**File:** `core/nf/pcf/pcf.go` and `smpolicy/smpolicy.go`

**Tasks:**
- Add a simple classification rule in the PCF's `SmPolicyDecision` creation:
  - If SUPI starts with a specific prefix → Emergency UE
  - Otherwise → Public UE
- In the tester, configure Emergency UEs with a known IMSI prefix (e.g.,
  `001010000000001` – `001010000000010`).
- **Write code:** A function `classifyUE(supi string) string` that returns
  `"emergency"`, `"responder"`, or `"public"`.

### Day 6 — Build the Static Baseline Controller
**File:** `core/nf/pcf/smpolicy/baseline_controller.go` (NEW FILE)

**What to write:**
```go
// baselineDecision applies static 3GPP-aligned QoS rules.
// This is the comparison baseline for the ML system.
func baselineDecision(ueClass string, congestionLevel string) SmPolicyDecision {
    switch ueClass {
    case "emergency":
        // Never throttled, highest ARP, GBR guaranteed
        return SmPolicyDecision{
            Default5QI:    1,      // Highest priority 5QI
            SessionAMBRUL: 20000, // 20 Mbps — never reduced
            SessionAMBRDL: 20000,
        }
    case "public":
        if congestionLevel == "high" {
            return SmPolicyDecision{
                Default5QI:    9,    // Best-effort
                SessionAMBRUL: 2000, // Throttled to 2 Mbps
                SessionAMBRDL: 2000,
            }
        }
        return SmPolicyDecision{
            Default5QI:    9,
            SessionAMBRUL: 20000,
            SessionAMBRDL: 20000,
        }
    }
    // Default
    return SmPolicyDecision{Default5QI: 9, SessionAMBRUL: 5000, SessionAMBRDL: 5000}
}
```

**The static congestion level is determined by a hardcoded threshold:**
- if `current_ues > 150` → "high"
- else → "normal"

### Day 7 — Measure the Baseline
**Tasks:**
- Run the tester: start with 20 UEs, then at T=60s inject 200 more public UEs
  and 10 emergency UEs simultaneously (simulate the disaster).
- Record for 5 minutes:
  - Emergency call success rate (%)
  - Emergency call setup time (ms)
  - Public user average throughput (kbps)
  - NWDAF `QOS_SUSTAINABILITY` drop_rate
- **Save these numbers.** They are the "Static 3GPP Baseline" row in your
  results table. This is the most important output of this week.

---

## Days 8–12 : Build the ML Engine (Python Microservice)

### Day 8 — Create the Python ML Project
**Location:** Create a new folder `core/ai_engine/ml_engine/`

**Setup:**
```bash
cd core/ai_engine/ml_engine
python3 -m venv .venv
source .venv/bin/activate
pip install torch numpy flask scikit-learn
```

**Create `app.py` skeleton:**
```python
from flask import Flask, request, jsonify
app = Flask(__name__)

@app.route('/predict', methods=['POST'])
def predict():
    # NWDAF data comes in here
    # Returns: {"congestion_predicted": true/false, "horizon_seconds": 30}
    pass

@app.route('/action', methods=['POST'])
def action():
    # DQN decides policy action
    # Returns: {"action": "throttle_public_video", "ambr_ul": 2000, "ambr_dl": 2000}
    pass

if __name__ == '__main__':
    app.run(port=8765)
```

### Day 9 — Build the LSTM Congestion Predictor
**File:** `core/ai_engine/ml_engine/lstm_predictor.py`

**What to build:**
```python
import torch
import torch.nn as nn

class CongestionLSTM(nn.Module):
    """
    Input:  sequence of (current_ues, registration_rate, drop_rate) — last T=10 steps
    Output: probability of congestion in next 30 seconds
    """
    def __init__(self, input_size=3, hidden_size=64, num_layers=2):
        super().__init__()
        self.lstm = nn.LSTM(input_size, hidden_size, num_layers, batch_first=True)
        self.fc = nn.Linear(hidden_size, 1)
        self.sigmoid = nn.Sigmoid()

    def forward(self, x):
        # x shape: (batch, seq_len, input_size)
        out, _ = self.lstm(x)
        out = self.fc(out[:, -1, :])   # Last timestep
        return self.sigmoid(out)
```

**Training data:** Use the NWDAF `nwdaf_data_points` database from Day 7's
experiment. Extract `(current_ues, registration_rate, drop_rate)` at each
30-second interval. Label any window where `drop_rate > 0.05` as "congested".

**Key insight for JavaScript learners:** Python lists and JS arrays are similar,
but Python tensors (via PyTorch) are like typed arrays that run on GPU. The
`nn.LSTM` layer is doing the same thing as a for-loop over time, but in a
mathematically optimized way that "remembers" patterns.

### Day 10 — Build the DQN Agent
**File:** `core/ai_engine/ml_engine/dqn_agent.py`

**State space (what the agent observes):**
- `current_ue_count` (normalized 0–1 by dividing by max capacity)
- `congestion_probability` (0–1, from LSTM output)
- `emergency_ue_count` (how many emergency UEs are active)
- `current_public_ambr` (current throttle level, normalized)

**Action space (what the agent can do):**
```python
ACTIONS = {
    0: "no_change",
    1: "throttle_public_video_mild",   # AMBR → 5 Mbps
    2: "throttle_public_video_severe", # AMBR → 2 Mbps
    3: "block_public_video",           # AMBR → 512 kbps
    4: "restore_normal",               # Return to 20 Mbps
}
```

**Reward function (teach the agent what "good" means):**
```python
def compute_reward(emergency_success_rate, public_drop_rate, unnecessary_throttle):
    reward = 0
    reward += 10 * emergency_success_rate   # Strongly reward protecting emergency UEs
    reward -= 2 * public_drop_rate          # Penalize dropping public UEs unnecessarily
    reward -= 1 * unnecessary_throttle      # Penalize throttling when not needed
    return reward
```

### Day 11 — Build the Guardrail Engine
**File:** `core/ai_engine/ml_engine/guardrail.py`

**This is critical for the paper's novelty claim #3.**

```python
class GuardrailEngine:
    """
    Formal safety constraints that the DQN CANNOT violate.
    These map directly to 3GPP ARP pre-emption vulnerability rules.
    """

    HARD_RULES = [
        # Rule 1: Emergency UEs can NEVER be throttled (ARP §7.6.8 TS 23.501)
        lambda state, action: not (
            action in ["throttle_public_video_severe", "block_public_video"]
            and state["emergency_ue_count"] == 0  # Only apply to public
        ),
        # Rule 2: If emergency call success rate drops below 95%, only action=0 allowed
        lambda state, action: not (
            state.get("emergency_success_rate", 1.0) < 0.95
            and action == "no_change"
        ),
        # Rule 3: AMBR can never go below 64 kbps (minimum for SMS/signalling)
        lambda state, action: not (
            action == "block_public_video"
            and state.get("public_ambr_current", 1000) < 128
        ),
    ]

    def is_safe(self, state: dict, action: str) -> bool:
        for rule in self.HARD_RULES:
            if not rule(state, action):
                return False
        return True

    def safe_action(self, state: dict, dqn_action: str) -> str:
        if self.is_safe(state, dqn_action):
            return dqn_action
        # Fall back to the most conservative safe action
        return "no_change"
```

### Day 12 — Wire the Python Service to the NWDAF (Go)
**File:** `core/nf/nwdaf/nwdaf.go` — add ML callback

**Modify `sendNotification()` to also call the Python ML service:**
```go
// After sending to regular subscribers, call ML engine
func (s *Service) queryMLEngine(result analytics.AnalyticsResult) {
    mlURL := "http://localhost:8765/action"
    payload, _ := json.Marshal(map[string]any{
        "analytics_id": result.AnalyticsID,
        "result":       result.Result,
        "confidence":   result.Confidence,
    })
    req, _ := http.NewRequest("POST", mlURL, strings.NewReader(string(payload)))
    req.Header.Set("Content-Type", "application/json")
    client := &http.Client{Timeout: 3 * time.Second}
    resp, err := client.Do(req)
    if err != nil {
        log.Debugf("ML engine query failed: %v", err)
        return
    }
    defer resp.Body.Close()
    // Parse action and forward to PCF
    var action map[string]any
    json.NewDecoder(resp.Body).Decode(&action)
    s.applyMLAction(action) // calls PCF update
}
```

---

## Days 13–15 : Integration and PCF Policy Update

### Day 13 — Add PCF Subscription to NWDAF
**File:** `core/nf/pcf/pcf.go`

**What to add:**
- On PCF startup, call `NWDAF.Subscribe("PCF", "NF_LOAD", ...)` with a
  callback URL pointing to a new HTTP endpoint `/nwdaf-alert` on the PCF.
- The PCF's `/nwdaf-alert` handler receives the NWDAF notification and
  triggers a policy re-evaluation for all active PDU sessions.

### Day 14 — Implement Dynamic AMBR Update in PCF
**File:** `core/nf/pcf/smpolicy/smpolicy.go`

**What to add:**
A function that the PCF calls when it receives an ML action:
```go
// ApplyMLAction updates QoS for all active sessions based on ML decision.
// Called when NWDAF delivers a congestion alert + ML action.
func ApplyMLAction(action map[string]any, sessionStore SessionStore) {
    ambrUL := int(action["ambr_ul"].(float64))  // kbps from ML engine
    ambrDL := int(action["ambr_dl"].(float64))
    ueClass := action["target_ue_class"].(string)  // "public" or "all"

    for _, session := range sessionStore.All() {
        if classifyUE(session.SUPI) == ueClass {
            // Trigger SmPolicy update toward SMF
            session.Decision.SessionAMBRUL = ambrUL
            session.Decision.SessionAMBRDL = ambrDL
            sessionStore.NotifySMF(session)
        }
    }
}
```

### Day 15 — End-to-End Integration Test
**Tasks:**
- Start the Python ML service: `python app.py`
- Start the 5G core: `cd orchestrate && ./run_studio.sh`
- Open two terminals: watch NWDAF logs and PCF logs simultaneously
- Simulate disaster: inject 200 UEs via the tester
- **Verify the chain works:**
  1. NWDAF collects UE spike data
  2. NWDAF calls Python ML service `/action`
  3. Python returns throttle action
  4. NWDAF calls PCF `/nwdaf-alert`
  5. PCF updates AMBR for public UEs
  6. Emergency UEs maintain connectivity
- **If anything breaks, fix it today.** This is the most likely day to hit bugs.

---

## Days 16–18 : Experiments and Results

> These three days are the most important for the paper. The graphs you
> produce here ARE the paper.

### Day 16 — Run Experiment 1: The Disaster Scenario
**Setup:**
- 20 base UEs (normal load)
- At T=60s: inject 200 public UEs + 10 emergency UEs simultaneously
- Run for 300 seconds (5 minutes)
- Repeat 3 times and average

**Measure (every 10 seconds):**
| Metric | Baseline (Day 7) | Your System |
|--------|-----------------|-------------|
| Emergency call success rate (%) | __ | __ |
| Emergency call setup time (ms) | __ | __ |
| Public user avg throughput (kbps) | __ | __ |
| Time to detect + respond to congestion (s) | __ | __ |
| UPF packet drop rate | __ | __ |

**Save raw CSV data.** You will plot these.

### Day 17 — Run Experiment 2: Prediction Horizon Ablation
This experiment tests whether LSTM's *predictive* nature matters.

**Variants to test:**
- **Reactive (no prediction):** Act only when `drop_rate > 0.05` already (congestion happened)
- **Predictive 15s:** Act when LSTM predicts congestion in 15s
- **Predictive 30s:** Act when LSTM predicts congestion in 30s (your main system)

**Metric:** Emergency call setup time during the disaster injection window.

**Expected result:** The 30s predictive variant should have the lowest setup time
because the policy was already adjusted before calls started dropping.

### Day 18 — Run Experiment 3: DQN vs Greedy vs Static
**Variants:**
- **Static (Day 7):** Fixed threshold, fixed throttle
- **Greedy:** Always pick the maximum throttle when congestion is detected (no learned balance)
- **DQN + Guardrail (your system):** Learned optimal trade-off

**Metric:** Unnecessary throttle rate — how often public users were throttled
when congestion wasn't actually severe. Lower is better (fairness).

**Expected result:** DQN should throttle less than greedy while maintaining
the same emergency success rate.

---

## Days 19–20 : Write the Paper

### Day 19 — Write Sections 1–4

**Use your existing documents as source material:**
- `gptQos.md` → Section 2 (Background: ARP, 5QI, GBR, AMBR, NWDAF)
- `novelty.md` → Section 1 (Introduction, Problem Statement, Contributions)

**Section 1 — Introduction (1 page)**
```
Opening hook: "During the [disaster name], cellular networks experienced
a 400% surge in traffic, causing emergency communications to fail..."
Problem: Static 3GPP rules react after congestion. This paper prevents it.
Contributions (use novelty.md comparison table directly):
  1. LSTM-based predictive congestion detection
  2. DQN policy optimization with formal guardrail
  3. Implementation on real 5G Core (mmt-studio-5g6g NWDAF+PCF)
  4. Evaluation showing X% improvement in emergency success rate
```

**Section 2 — Background (1 page)**
Directly copy and adapt from `gptQos.md`:
- 5QI, ARP, GBR, AMBR tables are already written
- Add one paragraph on NWDAF subscription mechanism

**Section 3 — Related Work (0.5 pages)**
Find 5–8 papers on Google Scholar:
- Search: "NWDAF emergency 5G QoS machine learning"
- Search: "DQN network slicing resource management"
- Search: "LSTM 5G congestion prediction"
- Write 1 sentence on each, ending with: "Unlike prior work, we combine
  predictive LSTM + DQN + formal guardrail on a real 5G Core testbed."

**Section 4 — System Design (1.5 pages)**
Draw the architecture diagram (see below). Explain each component.

### Day 20 — Write Sections 5–7 + Submit

**Section 5 — Implementation (0.5 pages)**
- State: "We implemented on mmt-studio-5g6g, a 3GPP TS 23.288-aligned open
  source 5G Core, extending the NWDAF analytics engine and PCF policy
  decision function."
- Mention: Go for core NFs, Python + PyTorch for ML engine, REST for integration

**Section 6 — Evaluation (2 pages)**
- 3 figures from Day 16–18 experiments:
  - Fig 2: Emergency call success rate over time (Baseline vs Your System)
  - Fig 3: Congestion detection latency (Reactive vs 15s vs 30s predictive)
  - Fig 4: Unnecessary throttle events (Static vs Greedy vs DQN)
- 1 table: Overall performance comparison
- Write 1 paragraph per figure explaining what it shows and why it matters

**Section 7 — Conclusion (0.25 pages)**
- "We proposed... We implemented... Results show X% improvement..."
- Future work: multi-cell coordination, real radio hardware

**Submission Steps:**
1. Format using IEEE conference template (download from ieee.org/conferences)
2. Use Overleaf (free, browser-based LaTeX) — no installation needed
3. Submit to: IEEE VTC (Spring or Fall) or IEEE WCNC
4. Check submission deadline at: `cfp.ieee.org`

---

## Architecture Diagram for Your Paper

```
┌─────────────────────────────────────────────────────────────────┐
│                    5G Core (mmt-studio-5g6g)                    │
│                                                                 │
│  ┌──────────┐    ┌──────────────────────────────────────────┐  │
│  │  tester  │───▶│  AMF  │  SMF  │  UPF  │  IMS CSCF       │  │
│  │ (200 UEs)│    └────────────────────────┬─────────────────┘  │
│  └──────────┘                             │                    │
│                               NF metrics  │                    │
│                                           ▼                    │
│                               ┌───────────────────┐           │
│                               │  NWDAF            │           │
│                               │  ─ collectionLoop │           │
│                               │  ─ Subscribe API  │           │
│                               └────────┬──────────┘           │
│                                        │ HTTP POST /action     │
│                               ┌────────▼──────────┐           │
│                               │  Python ML Engine  │           │
│                               │  ─ LSTM Predictor  │           │
│                               │  ─ DQN Agent       │           │
│                               │  ─ Guardrail       │           │
│                               └────────┬──────────┘           │
│                                        │ HTTP POST /nwdaf-alert│
│                               ┌────────▼──────────┐           │
│                               │  PCF              │           │
│                               │  ─ SmPolicyUpdate │           │
│                               │  ─ AMBR throttle  │           │
│                               └───────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Daily Checklist

| Day | Task | Output | Status |
|-----|------|--------|--------|
| 1 | Understand NWDAF data collection | Written notes on NWDAF fields | [ ] |
| 2 | Understand PCF SmPolicyDecision | Written notes on which struct fields to modify | [ ] |
| 3 | Map NWDAF→PCF gap | Architecture gap document | [ ] |
| 4 | Run tester, attach UEs | Screenshot of UEs attaching | [ ] |
| 5 | Add UE classification to PCF | `classifyUE()` function | [ ] |
| 6 | Build static baseline controller | `baseline_controller.go` | [ ] |
| 7 | Measure baseline | CSV with baseline results | [ ] |
| 8 | Setup Python ML project | `app.py` skeleton running | [ ] |
| 9 | Build LSTM predictor | `lstm_predictor.py` with training script | [ ] |
| 10 | Build DQN agent | `dqn_agent.py` with reward function | [ ] |
| 11 | Build Guardrail engine | `guardrail.py` with 3 hard rules | [ ] |
| 12 | Wire NWDAF → ML service | Go code calling Python REST endpoint | [ ] |
| 13 | PCF subscribes to NWDAF | PCF `/nwdaf-alert` endpoint working | [ ] |
| 14 | Dynamic AMBR update in PCF | `ApplyMLAction()` working | [ ] |
| 15 | End-to-end integration test | Full chain verified in logs | [ ] |
| 16 | Experiment 1: disaster scenario | CSV + 3 graphs | [ ] |
| 17 | Experiment 2: prediction horizon | CSV + 1 graph | [ ] |
| 18 | Experiment 3: DQN vs greedy | CSV + 1 graph | [ ] |
| 19 | Write paper sections 1–4 | Draft in Overleaf/LaTeX | [ ] |
| 20 | Write sections 5–7 + submit | Final PDF submitted | [ ] |

---

## Key Terms Quick Reference

| Term | What it is | Struct field in code |
|------|------------|---------------------|
| 5QI | Priority class for a packet flow | `SmPolicyDecision.Default5QI` |
| ARP | Who gets preempted when network is full | Inside PCC rule in PCF |
| GBR | Guaranteed bandwidth (min floor) | GBR flow parameter |
| AMBR | Max total bandwidth per UE | `SmPolicyDecision.SessionAMBRUL/DL` |
| MFBR | Max bandwidth per individual flow | Per-flow parameter |
| QFI | ID tag on a packet flow | `SmPolicyDecision.DefaultQFI` |
| NWDAF | Analytics function — observes the network | `core/nf/nwdaf/` |
| PCF | Policy function — sets the rules | `core/nf/pcf/` |
| SMF | Session manager — enforces PCF rules on UPF | `core/nf/nf/smf/` |

---

## What Makes This Paper Pass Peer Review

Reviewers check three things:

1. **Is the problem real?** ✅ Emergency communication failure during disasters
   is documented in literature. Cite 2–3 real incidents.

2. **Is the solution novel?** ✅ The combination of predictive LSTM +
   DQN + formal guardrail on a real 5G Core is novel. Most prior papers
   use ns-3 simulation, not a real core.

3. **Do the results support the claims?** ✅ You need emergency call
   success rate to be measurably higher (target: +15% or more vs baseline)
   with your system. If the numbers don't show this, fix the DQN reward
   function before Day 19.

---

*Written for mmt-studio-5g6g project, Summer Capstone 2026.*
*Target: IEEE VTC / WCNC / ICC conference track.*
