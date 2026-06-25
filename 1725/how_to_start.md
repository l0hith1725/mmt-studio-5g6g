# How to Actually Use the MMT Setup and Start Research
## A Step-by-Step Practical Guide

---

## What You Are Working With

Three things run together:

```
┌─────────────────────────────────────────────────────────┐
│   Your Mac (Development Machine)                        │
│                                                         │
│  ┌──────────────┐    ┌────────────────────────────┐    │
│  │  sacore      │    │  satester                  │    │
│  │  (5G Core)   │◄──►│  (gNB + UE simulator)      │    │
│  │  :5000       │    │  :5001                     │    │
│  │  172.30.0.10 │    │  172.30.0.20               │    │
│  └──────────────┘    └────────────────────────────┘    │
│         ▲                                               │
│         │  (you will plug in here)                     │
│  ┌──────┴──────────────────────────┐                   │
│  │  Python ML Engine  :8765        │                   │
│  │  (LSTM + DQN + Guardrail)       │                   │
│  │  YOU will build this            │                   │
│  └─────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

- **sacore** = the real 5G Core (AMF, SMF, UPF, PCF, NWDAF, IMS). Written in Go.
- **satester** = gNB simulator + UE pool. Written in Python. Pretends to be a
  real base station with 128 virtual mobile phones.
- **Your ML Engine** = what you will build in Python. Talks to the core over REST.

---

## IMPORTANT: Mac Cannot Run the Full Stack

The core (sacore) uses Linux kernel features — specifically:
- **DPDK** (hugepages) for the UPF data plane
- **TUN interfaces** for GTP-U tunnels
- **SCTP** for NGAP

These do not run natively on macOS. You have **two options:**

### Option A: Use Docker Desktop (Easier, recommended for now)

Docker Desktop on Mac runs a Linux VM under the hood. The containers run
inside that VM, which has the Linux kernel. This works for the core.

**Requirement:** Docker Desktop for Mac (M1/M2/Intel), at least 8 GB RAM
assigned to Docker.

### Option B: Use a Linux VM or remote server (Better for experiments)

For actual research experiments, you will need a Linux machine:
- Ubuntu 22.04 or Debian 12
- 8+ GB RAM, 4+ CPU cores
- This is where you run your real experiments and collect data

For learning and code reading → Mac is fine.
For actual paper experiments → Linux server (college lab machine, VPS, etc.)

---

## Part 1: First Run — Get the Stack Up

### Step 1: Check Docker is installed

```bash
docker --version
docker compose version
```

You need Docker Engine 24+ and Compose v2. If missing:
- Download Docker Desktop from https://www.docker.com/products/docker-desktop/

### Step 2: Increase Docker resources

In Docker Desktop → Settings → Resources:
- Memory: at least 6 GB (8 GB preferred)
- CPUs: at least 4
- Disk: at least 20 GB

### Step 3: Navigate to orchestrate and bring up the stack

```bash
cd /Users/sai/Downloads/trash/mmt-studio-5g6g/orchestrate
./run_studio.sh
```

This will:
1. Build the Go core Docker image from `core/`
2. Build the Python tester Docker image from `tester/`
3. Start three containers: `sacore`, `satraffic`, `satester`
4. Create a Docker bridge network `mmtnet` at 172.30.0.0/24

**First build takes 5–15 minutes** (compiles Go code). Subsequent runs are fast.

### Step 4: Verify both web UIs are running

Open two browser tabs:
- **Core UI:** http://localhost:5000  ← The 5G Core dashboard
- **Tester UI:** http://localhost:5001 ← The gNB + UE controller

If you see both dashboards, the stack is running.

### Step 5: Configure the tester to point at the core

In the **Tester UI** (http://localhost:5001):
- Find gNB profile settings
- Set **AMF IP** = `172.30.0.10`
- Set **AMF SCTP Port** = `38412`
- Set **UPF IP** = `172.30.0.10`
- Set **GTP-U Port** = `2152`

---

## Part 2: Your First UE Attach — Understanding What Happens

### Attach 1 UE manually

In the Tester UI:
1. Go to UE management
2. Select 1 UE from the `embb-bulk` pool (IMSI starting with `001011234560001`)
3. Click "Register" or "Attach"
4. Watch the logs

**What happens in the code when you press Attach:**

```
Tester (satester) sends:
  → NGAP NG Setup Request to AMF (port 38412)
  → NGAP Initial UE Message (Registration Request NAS)

AMF (core/nf/amf/) receives Registration Request:
  → Authenticates UE via AUSF/UDM (5G-AKA)
  → Sends Registration Accept back

SMF (core/nf/smf/) sets up PDU session:
  → Calls PCF: smpolicy.Create() in smpolicy.go
  → PCF returns SmPolicyDecision (AMBR=200Mbps, 5QI=9)
  → SMF configures UPF tunnel

UE is now attached! It has an IP address from 10.45.0.0/16 pool.
```

Watch the core logs to see this:
```bash
cd /Users/sai/Downloads/trash/mmt-studio-5g6g/orchestrate
./run_studio.sh logs
```

### Check the NWDAF is collecting data

After attaching the UE, wait 30 seconds, then call the NWDAF API:
```bash
curl http://localhost:5000/api/nwdaf/analytics?analyticsId=UE_MOBILITY
```

You should see JSON with `current_ues: 1`. This is the data your LSTM will
train on.

---

## Part 3: Run the Existing NWDAF Test Suite

This is the most important step before writing any new code.
Run the existing NWDAF tests to see what already works:

```bash
cd /Users/sai/Downloads/trash/mmt-studio-5g6g/orchestrate
./run_studio.sh logs &   # run logs in background

# In the tester UI, or via CLI:
# Run the NWDAF test suite: 28_nwdaf.robot
```

The NWDAF test suite (at `tester/robot/suites/policy_charging/28_nwdaf.robot`)
has 5 test cases:
- **TC-NWDAF-001:** Data collection (registration + PDU session events)
- **TC-NWDAF-002:** Anomaly detection
- **TC-NWDAF-003:** Analytics subscription via REST API
- **TC-NWDAF-010:** NF load analytics
- **TC-NWDAF-011:** UE mobility analytics

**Run them. Look at which ones PASS and which FAIL.**

The ones that pass tell you what is already wired.
The ones that fail tell you what you need to fix or extend.

---

## Part 4: Run the Stress Test — Your Disaster Baseline

This is the existing congestion test. It is the closest thing to your
"disaster scenario" that already exists:

**Test file:** `tester/robot/suites/diagnostics/04_stress.robot`

This test registers many UEs rapidly (simulating a traffic spike).

Run it, watch the NWDAF logs, and observe:
1. Does the NWDAF's `UE_MOBILITY` analytics show the UE count spike?
2. Does the `NF_LOAD` analytics show increasing load?
3 Does the PCF change any policy? (Spoiler: NO. Not yet. That's what you build.)

**Record the baseline numbers from this test.** This is Row 1 of your
paper's results table: "Static 3GPP Baseline (no ML)."

---

## Part 5: Understanding the NWDAF API — Your Research Interface

The NWDAF exposes a REST API at `http://localhost:5000/api/nwdaf/`.

These are the endpoints you will call from your Python ML engine:

```bash
# Get current network load
curl http://localhost:5000/api/nwdaf/analytics?analyticsId=NF_LOAD

# Get UE mobility stats
curl http://localhost:5000/api/nwdaf/analytics?analyticsId=UE_MOBILITY

# Get QoS drop rate
curl http://localhost:5000/api/nwdaf/analytics?analyticsId=QOS_SUSTAINABILITY

# Subscribe to continuous notifications
curl -X POST http://localhost:5000/api/nwdaf/subscriptions \
  -H "Content-Type: application/json" \
  -d '{
    "analytics_id": "NF_LOAD",
    "callback_url": "http://localhost:8765/nwdaf-callback",
    "interval_sec": 30
  }'
```

The response looks like:
```json
{
  "analytics_id": "NF_LOAD",
  "result": {
    "load_level": "low",
    "trend": "increasing",
    "avg_registration_rate": 2.5,
    "avg_session_rate": 1.8
  },
  "confidence": 0.75,
  "computed_at": 1751234567
}
```

**These JSON fields are your LSTM input features.**

---

## Part 6: The UE Configuration — How to Add Your 3 Tiers

Look at `tester/config/baseline.yaml`. It defines 128 UEs in 4 buckets:
- `embb-bulk`: 100 UEs (IMSI 001..100) → these become your **Tier 3 (Public)**
- `urllc-pool`: 8 UEs (IMSI 117..124) → these become your **Tier 1 (Emergency)**
- `miot-pool`: 16 UEs → IoT (not needed for your paper)

**You do not need to add new UEs.** You just use the IMSI prefix to classify:

```python
def classify_ue(imsi: str) -> str:
    """
    IMSI starts with 001011234560...
    Last 3 digits = UE number (001-128)
    """
    ue_number = int(imsi[-3:])
    if ue_number <= 8:
        return "emergency"    # urllc-pool UEs (117-124) — reuse these
    elif ue_number <= 24:
        return "semi_priority"
    else:
        return "public"       # embb-bulk UEs
```

In the PCF Go code, you add the same logic in `classifyUE(supi string) string`.

---

## Part 7: The PCF REST API — How Your ML Engine Talks to the Core

The core exposes OAM (Operations, Administration, Maintenance) REST APIs.
You can modify policies at runtime without restarting the core.

```bash
# See current policy for a UE
curl http://localhost:5000/api/pcf/associations

# Trigger a policy update (your ML engine calls this)
curl -X POST http://localhost:5000/api/pcf/emergency-mode \
  -H "Content-Type: application/json" \
  -d '{
    "mode": "disaster",
    "tier3_ambr_kbps": 512,
    "tier3_block_video": true,
    "tier3_block_audio": true
  }'
```

**Note:** This specific endpoint does not exist yet — you will create it.
It is the bridge between your Python ML engine and the Go PCF.

---

## Part 8: Your First Research Code — Step by Step

### Step 1: Create the ML engine directory

```bash
mkdir -p /Users/sai/Downloads/trash/mmt-studio-5g6g/ml_engine
cd /Users/sai/Downloads/trash/mmt-studio-5g6g/ml_engine
python3 -m venv .venv
source .venv/bin/activate
pip install flask requests numpy torch
```

### Step 2: Write a simple data collector (Day 1 actual code)

Create `ml_engine/collector.py`:

```python
"""
collector.py — Polls NWDAF every 30 seconds and saves data to CSV.
This is the data you will train your LSTM on.
"""

import requests
import csv
import time
from datetime import datetime

CORE_URL = "http://localhost:5000"
OUTPUT_FILE = "nwdaf_data.csv"

ANALYTICS_IDS = ["NF_LOAD", "UE_MOBILITY", "QOS_SUSTAINABILITY"]

def collect_once() -> dict:
    """Collect one round of analytics from NWDAF."""
    row = {"timestamp": datetime.now().isoformat()}

    for analytics_id in ANALYTICS_IDS:
        try:
            resp = requests.get(
                f"{CORE_URL}/api/nwdaf/analytics",
                params={"analyticsId": analytics_id},
                timeout=5
            )
            if resp.status_code == 200:
                data = resp.json()
                result = data.get("result", {})
                # Flatten the result dict into the row
                for key, val in result.items():
                    row[f"{analytics_id}_{key}"] = val
        except requests.exceptions.RequestException as e:
            print(f"  Warning: Could not reach NWDAF for {analytics_id}: {e}")

    return row

def main():
    print(f"Starting NWDAF data collection → {OUTPUT_FILE}")
    print("Press Ctrl+C to stop.\n")

    fieldnames = None
    with open(OUTPUT_FILE, "w", newline="") as f:
        writer = None

        while True:
            row = collect_once()
            print(f"[{row['timestamp']}] UEs: {row.get('UE_MOBILITY_current_ues', '?')} | "
                  f"Load: {row.get('NF_LOAD_load_level', '?')} | "
                  f"Drop rate: {row.get('QOS_SUSTAINABILITY_drop_rate', '?')}")

            if writer is None:
                fieldnames = list(row.keys())
                writer = csv.DictWriter(f, fieldnames=fieldnames)
                writer.writeheader()

            writer.writerow(row)
            f.flush()  # write to disk immediately

            time.sleep(30)  # poll every 30 seconds

if __name__ == "__main__":
    main()
```

Run it while the stack is running:
```bash
python ml_engine/collector.py
```

After 10 minutes you will have your first dataset.

### Step 3: Write the disaster trigger (test script)

Create `ml_engine/disaster_sim.py`:

```python
"""
disaster_sim.py — Simulates a disaster by instructing the tester
to rapidly register many UEs, then measures the NWDAF response.
"""

import requests
import time

TESTER_URL = "http://localhost:5001"
CORE_URL   = "http://localhost:5000"

def register_ue_batch(count: int, ue_class: str):
    """Tell the tester to register a batch of UEs."""
    print(f"Registering {count} {ue_class} UEs...")
    # This calls the tester's REST API to attach UEs
    resp = requests.post(f"{TESTER_URL}/api/ue/batch-register", json={
        "count": count,
        "ue_class": ue_class,
    })
    return resp.json()

def get_nwdaf_snapshot() -> dict:
    """Get current NWDAF analytics."""
    resp = requests.get(f"{CORE_URL}/api/nwdaf/analytics",
                        params={"analyticsId": "UE_MOBILITY"})
    return resp.json()

def main():
    print("=== Disaster Simulation ===")
    print("T=0: Baseline state (20 public UEs)")
    print("T=60s: DISASTER — inject 80 public + 10 emergency UEs")
    print("T=360s: Measure recovery\n")

    # Phase 1: Baseline
    print("[Phase 1] Baseline — 20 public UEs attached")
    time.sleep(60)
    baseline = get_nwdaf_snapshot()
    print(f"  NWDAF: {baseline['result']}")

    # Phase 2: Disaster injection
    print("\n[Phase 2] DISASTER — injecting 80 public + 10 emergency UEs")
    register_ue_batch(80, "public")
    register_ue_batch(10, "emergency")

    # Phase 3: Monitor for 5 minutes
    print("\n[Phase 3] Monitoring for 5 minutes...")
    for i in range(10):
        time.sleep(30)
        snapshot = get_nwdaf_snapshot()
        ue_count = snapshot.get("result", {}).get("current_ues", "?")
        print(f"  T+{(i+1)*30}s: UEs={ue_count}")

if __name__ == "__main__":
    main()
```

---

## Part 9: The Logs — What to Watch

The most important logs for your research:

```bash
# Watch everything
cd orchestrate && ./run_studio.sh logs

# Watch only core (PCF, NWDAF, AMF logs)
docker logs sacore -f 2>&1 | grep -E "(pcf|nwdaf|amf)"

# Watch only NWDAF
docker logs sacore -f 2>&1 | grep "nwdaf"

# Watch only PCF policy decisions
docker logs sacore -f 2>&1 | grep "pcf.smpolicy"
```

When a UE attaches, you will see in the logs:
```
[pcf.smpolicy] SM Policy Association created ctxRef=smpolicy-001011234560001-1-1
               dnn=internet sst=1 rules=1 defaultQFI=1 charging=offline
```

This line shows exactly what policy the PCF gave the UE.
For your paper, you want to see this line change when your ML engine acts.

---

## Part 10: The 3-Day Quick Start Research Plan

### Today (Day 0): Get the stack running

```bash
# 1. Start the stack
cd orchestrate && ./run_studio.sh

# 2. Open browser: http://localhost:5000 and http://localhost:5001

# 3. Attach 1 UE from tester UI and watch core logs

# 4. Call the NWDAF API
curl http://localhost:5000/api/nwdaf/analytics?analyticsId=UE_MOBILITY

# Goal: See the UE count go from 0 to 1 in the API response.
```

### Tomorrow (Day 1): Collect your first dataset

```bash
# 1. Run the data collector
python ml_engine/collector.py &

# 2. Run the stress test from tester UI (suite 04_stress.robot)
# This creates traffic spikes the NWDAF will record

# 3. After 30 minutes, open nwdaf_data.csv
# This is your first LSTM training dataset

# Goal: Have a CSV with 60+ rows of NWDAF data.
```

### Day After Tomorrow (Day 2): First LSTM prototype

```bash
# 1. Load the CSV into a Jupyter notebook or Python script
# 2. Plot UE_MOBILITY_current_ues over time
# 3. Train a simple LSTM on it
# 4. Goal: predict whether UE count will exceed 80 in the next 30 seconds
```

---

## Summary: The Research Flow

```
Stack running → Attach UEs → Watch NWDAF collect data
     ↓
Run stress test → Record baseline numbers
     ↓
Build ML engine → Train LSTM on NWDAF data
     ↓
Wire LSTM to NWDAF → Predictions flow to DQN
     ↓
Wire DQN to PCF → Policy changes automatically
     ↓
Run experiment again → Compare with baseline
     ↓
That delta = your IEEE paper
```

---

## Quick Reference: Ports and Endpoints

| Service | URL | What it does |
|---------|-----|-------------|
| Core Web UI | http://localhost:5000 | 5G Core dashboard |
| Tester Web UI | http://localhost:5001 | gNB + UE controller |
| NWDAF Analytics | http://localhost:5000/api/nwdaf/analytics | Get analytics |
| NWDAF Subscribe | http://localhost:5000/api/nwdaf/subscriptions | Subscribe to alerts |
| PCF Associations | http://localhost:5000/api/pcf/associations | See UE policies |
| Your ML Engine | http://localhost:8765 | You will build this |

---

*Written for mmt-studio-5g6g project, Summer Capstone 2026.*
