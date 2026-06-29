# Days 5–7: Corrected 2-Hour Execution Plan
## What Was Wrong + Complete Rewrite

---

## ❌ What Was Wrong in My Previous Plan

### Wrong 1: Only 2 UE classes, you need 3
My plan had `"emergency"` and `"public"` only.  
You need:
- **Tier 1 — Emergency**: video + audio + SMS (first responders)
- **Tier 2 — Public**: audio + SMS only (regular public during disaster)
- **Tier 3 — Restricted**: SMS only (lowest-privilege users, e.g., tourists/data-only)

### Wrong 2: `BaselineDecision` only set AMBR — no real PCC rules
I set `SessionAMBRUL/DL` and `Default5QI` only.  
The real enforcement is per-service **PCC rules** — the PCF must explicitly
allow/deny each service (video, voice, SMS) per UE tier.  
`SmPolicyDecision.PccRules []pcf.PCCRule` exists and is the right field to use.

### Wrong 3: No GBR/MBR or ARP per flow
The `PCCRule` struct has `GBRULKbps`, `GBRDLKbps`, `MBRULKbps`, `MBRDLKbps`,
`ArpPriority` — all left at 0/empty in my plan. Wrong.

### Wrong 4: No admission control
You said: "if congestion > expected, stop new UE connect."  
My plan had zero code for this. It needs to be a check at the start of
`smpolicy.Create()`.

### Wrong 5: No "Restricted" IMSI bucket in baseline.yaml
Only emergency was added. Need a third bucket for restricted UEs.

---

## The Real QoS Values (from `db/seed/services.go`)

These service names already exist in the database — use them directly in PCC rules.

| Service Name | 5QI | Type | ARP | GBR UL/DL | MBR UL/DL | What it enables |
|---|---|---|---|---|---|---|
| `conv_voice` | 1 | GBR | 1 | 64/64 kbps | 128/128 kbps | Audio call |
| `conv_video` | 2 | GBR | 2 | 1000/1000 kbps | 4000/4000 kbps | Video call |
| `ims_signalling` | 5 | NonGBR | 1 | - | - | SMS + SIP signalling |
| `default_data` | 9 | NonGBR | 9 | - | - | Internet/data |

> ARP 1 = highest priority (never pre-empted).  
> ARP 9 = lowest (first to be pre-empted when network is full).

### QFI (QoS Flow Identifier) — which number goes to which service
QFI is a number 1–63 that the SMF assigns to a flow.  
The convention in this codebase: `DefaultQFI` is set from the rule position in
the list (see `pcf.go` line 160: `defaultQFI = uint8(i + 1)`).  
For your baseline, use:
- QFI 1 → `ims_signalling` (always-on default flow)
- QFI 2 → `conv_voice`
- QFI 3 → `conv_video`

---

## 3-Tier Disaster Policy Table

| Metric | Tier 1: Emergency | Tier 2: Public | Tier 3: Restricted |
|--------|------------------|----------------|-------------------|
| Video call (`conv_video`) | ✅ Allowed | ❌ Blocked | ❌ Blocked |
| Audio call (`conv_voice`) | ✅ Allowed | ✅ Allowed | ❌ Blocked |
| SMS / IMS signalling | ✅ Allowed | ✅ Allowed | ✅ Allowed |
| Session AMBR UL/DL | 20 Mbps | 512 kbps | 64 kbps |
| Default 5QI | 1 (voice GBR) | 1 (voice GBR) | 5 (signalling) |
| ARP Priority | 1 (protected) | 8 (normal) | 14 (pre-emptable) |
| New connections allowed? | Always | Yes, if UE < 200 | Blocked if UE > 150 |

---

## IMSI Ranges to Use

| Bucket | IMSI Range | Class |
|--------|-----------|-------|
| `emergency-pool` | `001010000000001` – `001010000000010` (10 UEs) | `"emergency"` |
| `embb-bulk` | `001011234560001` – `001011234560100` (100 UEs) | `"public"` |
| `restricted-pool` | `001019999990001` – `001019999990020` (20 UEs) | `"restricted"` |

---

## ⏱ Hour 1 — Write the Code

### Step 1 — Fix `ClassifyUE()` in `pcf.go` (10 min)

**File:** `core/nf/pcf/pcf.go`  
Replace whatever you had with:

```go
// ClassifyUE returns the disaster-tier for a subscriber.
//
//   "emergency"  — first responders. IMSI prefix 0010100000000 (01–10).
//   "restricted" — low-priority UEs. IMSI prefix 0010199999900.
//   "public"     — everyone else.
func ClassifyUE(supi string) string {
    imsi := supi
    if strings.HasPrefix(supi, "imsi-") {
        imsi = supi[5:]
    }
    // Emergency: 001010000000001 – 001010000000010
    if strings.HasPrefix(imsi, "0010100000000") {
        tail := imsi[len("0010100000000"):]
        if tail >= "01" && tail <= "10" {
            return "emergency"
        }
    }
    // Restricted: 001019999990001 – 001019999990020
    if strings.HasPrefix(imsi, "00101999999000") {
        return "restricted"
    }
    return "public"
}
```

---

### Step 2 — Rewrite `baseline_controller.go` (30 min)

**File:** `core/nf/pcf/smpolicy/baseline_controller.go`

This is the complete rewrite with real PCC rules:

```go
// Copyright (c) 2026 MakeMyTechnology. All rights reserved.
//
// baseline_controller.go — Static 3-tier disaster QoS baseline.
//
// Tier 1 (emergency): video + audio + SMS. Full GBR protection. ARP=1.
// Tier 2 (public):    audio + SMS only.  No video. ARP=8.
// Tier 3 (restricted): SMS only.         No calls. ARP=14.
// Admission control: block restricted at >150 UEs, public at >200 UEs.
package smpolicy

import (
    "sync/atomic"

    "github.com/mmt/mmt-studio-core/nf/pcf"
    "github.com/mmt/mmt-studio-core/oam/logger"
)

var activeUECount atomic.Int64

func TrackUECreate() { activeUECount.Add(1) }
func TrackUEDelete() { activeUECount.Add(-1) }
func CurrentUECount() int64 { return activeUECount.Load() }

// AdmissionAllowed returns false if the network is too congested to admit
// this UE class. Emergency UEs are ALWAYS admitted.
//
// Thresholds (hardcoded baseline — no ML):
//   > 200 UEs → block public UEs from new connections
//   > 150 UEs → block restricted UEs from new connections
//   Emergency → never blocked
func AdmissionAllowed(supi string) bool {
    count := activeUECount.Load()
    switch pcf.ClassifyUE(supi) {
    case "emergency":
        return true                  // never blocked
    case "restricted":
        return count <= 150          // blocked first
    default: // "public"
        return count <= 200          // blocked at higher threshold
    }
}

// BaselineDecision builds the full PCC rule set for a UE during disaster.
// Returns a SmPolicyDecision with per-service PCC rules, GBR/MBR,
// ARP priority, and session AMBR — all 3GPP-aligned.
func BaselineDecision(supi string) SmPolicyDecision {
    log := logger.Get("pcf.baseline")
    ueClass := pcf.ClassifyUE(supi)

    // ims_signalling is always allowed — needed for SMS and SIP registration.
    // It is the default Non-GBR flow (QFI=1 by convention).
    imsSignalling := pcf.PCCRule{
        ServiceName:  "ims_signalling",
        FiveQI:       5,         // IMS signalling — TS 23.501 Table 5.7.4-1
        ResourceType: "NonGBR",
        ArpPriority:  1,         // Signalling always ARP=1
        IsDefault:    true,
    }

    switch ueClass {
    case "emergency":
        // TIER 1: video + audio + SMS. All GBR flows. ARP=1 (highest).
        // Never pre-empted. Can pre-empt others (pcap=1).
        log.Infof("baseline: emergency UE %s → tier1 full services", supi)
        return SmPolicyDecision{
            DefaultQFI:    1,
            Default5QI:    1,          // conv_voice as the default flow
            SessionAMBRUL: 20_000,     // 20 Mbps
            SessionAMBRDL: 20_000,
            PccRules: []pcf.PCCRule{
                imsSignalling,
                {
                    ServiceName:  "conv_voice",
                    FiveQI:       1,
                    ResourceType: "GBR",
                    ArpPriority:  1,   // highest — never pre-empted
                    GBRULKbps:    64,  // guaranteed 64kbps voice floor
                    GBRDLKbps:    64,
                    MBRULKbps:    128,
                    MBRDLKbps:    128,
                },
                {
                    ServiceName:  "conv_video",
                    FiveQI:       2,
                    ResourceType: "GBR",
                    ArpPriority:  1,
                    GBRULKbps:    1000, // 1 Mbps video floor
                    GBRDLKbps:    1000,
                    MBRULKbps:    4000, // 4 Mbps video peak
                    MBRDLKbps:    4000,
                },
            },
        }

    case "public":
        // TIER 2: audio + SMS only. No video. ARP=8 (pre-emptable by emergency).
        log.Infof("baseline: public UE %s → tier2 audio+sms only", supi)
        return SmPolicyDecision{
            DefaultQFI:    1,
            Default5QI:    1,
            SessionAMBRUL: 512,    // 512 kbps — enough for audio, not video
            SessionAMBRDL: 512,
            PccRules: []pcf.PCCRule{
                {
                    ServiceName:  "ims_signalling",
                    FiveQI:       5,
                    ResourceType: "NonGBR",
                    ArpPriority:  8,   // lower than emergency
                    IsDefault:    true,
                },
                {
                    ServiceName:  "conv_voice",
                    FiveQI:       1,
                    ResourceType: "GBR",
                    ArpPriority:  8,
                    GBRULKbps:    64,
                    GBRDLKbps:    64,
                    MBRULKbps:    128,
                    MBRDLKbps:    128,
                },
                // conv_video intentionally NOT included → video blocked
            },
        }

    default: // "restricted"
        // TIER 3: SMS (IMS signalling) only. No calls. ARP=14 (lowest useful).
        log.Infof("baseline: restricted UE %s → tier3 sms only", supi)
        return SmPolicyDecision{
            DefaultQFI:    1,
            Default5QI:    5,          // signalling flow only
            SessionAMBRUL: 64,         // 64 kbps — SMS + keep-alive only
            SessionAMBRDL: 64,
            PccRules: []pcf.PCCRule{
                {
                    ServiceName:  "ims_signalling",
                    FiveQI:       5,
                    ResourceType: "NonGBR",
                    ArpPriority:  14,  // very low — first to be dropped
                    IsDefault:    true,
                },
                // conv_voice NOT included → no calls
                // conv_video NOT included → no video
            },
        }
    }
}
```

---

### Step 3 — Wire into `smpolicy.go` (15 min)

**File:** `core/nf/pcf/smpolicy/smpolicy.go`

**Change 1 — Admission control at top of `Create()`:**  
Right after the SUPI/PDUSessionID validation check (around line 169), add:

```go
// Admission control: reject new sessions when congestion is too high.
// Emergency UEs bypass this check (AdmissionAllowed always returns true for them).
if !AdmissionAllowed(ctx.SUPI) {
    return SmPolicyDecision{}, fmt.Errorf(
        "pcf.smpolicy: admission denied for %s — network congested (active_ues=%d)",
        ctx.SUPI, CurrentUECount())
}
```

**Change 2 — Replace hardcoded AMBR with BaselineDecision:**  
In `Create()`, replace the `decision := SmPolicyDecision{...}` block.  
Currently (lines 202–217) it hardcodes `SessionAMBRUL: 200_000`.  
Replace the entire block:

```go
// Build the baseline QoS decision for this UE's tier.
// BaselineDecision handles 3-tier classification + full PCC rule set.
baseline := BaselineDecision(ctx.SUPI)

// Merge: use baseline QoS as the session policy.
// PCC rules from the baseline override the DB-driven ruleSet for
// the disaster scenario. In steady-state (no disaster), you'd use
// the ruleSet from pcf.CreatePolicy — switch on a disaster flag.
decision := SmPolicyDecision{
    PccRules:         baseline.PccRules,
    DefaultQFI:       baseline.DefaultQFI,
    Default5QI:       baseline.Default5QI,
    SessionAMBRUL:    baseline.SessionAMBRUL,
    SessionAMBRDL:    baseline.SessionAMBRDL,
    ChargingMethod:   ruleSet.ChargingMethod,
    SmPolicyCtxRef:   nextCtxRef(k),
    RevalidationTime: time.Now().Add(DefaultRevalidationInterval),
}
```

**Change 3 — Track UE count:**  
After `pm.Inc(pm.PCFSmPolicyCreate, 1)` in `Create()`:
```go
TrackUECreate()
```
After `pm.Inc(pm.PCFSmPolicyDelete, 1)` in `Delete()`:
```go
TrackUEDelete()
```

---

### Step 4 — Build check (5 min)

```bash
go build ./core/nf/pcf/...
```

Expected errors to fix:
- If `logger` not imported in `baseline_controller.go` — add `"github.com/mmt/mmt-studio-core/oam/logger"`
- If `pcf.ClassifyUE` not found — make sure step 1 is saved and exported (uppercase C)
- If `AdmissionAllowed` fmt needed — add `"fmt"` to imports in smpolicy.go

---

## ⏱ Hour 2 — Config + Measurement

### Step 5 — Update `baseline.yaml` (10 min)

Add **two** new UE buckets (emergency + restricted):

```yaml
ue_buckets:
  - name: emergency-pool
    count: 10
    imsi_start:   "001010000000001"
    msisdn_start: "9110000001"
    slices: [2]                     # URLLC
    dnns: [ims]
    default_dnn: ims
    ue_ambr_dl_kbps: 20000
    ue_ambr_ul_kbps: 20000

  - name: restricted-pool
    count: 20
    imsi_start:   "001019999990001"
    msisdn_start: "9120000001"
    slices: [1]                     # eMBB
    dnns: [ims]
    default_dnn: ims
    ue_ambr_dl_kbps: 64
    ue_ambr_ul_kbps: 64

  # ... keep existing embb-bulk, urllc-pool, multi-slice unchanged
```

Reset DB after this change: `POST /api/admin/remove-db-file`

---

### Step 6 — Run the disaster scenario (30 min)

**Phase 1 — Normal (T=0 to T=60s):**
- Register 20 public UEs → should get 512kbps AMBR + voice+SMS rules

**Phase 2 — Disaster injection (T=60s):**
- Inject 100 public + 10 emergency + 20 restricted simultaneously
- Total = 150 UEs → restricted UEs should hit the 150-UE wall → admission denied

**Phase 3 — Verify (T=60s to T=300s):**
- All 10 emergency UEs connected → video + audio + SMS working
- Public UEs connected → audio + SMS, video blocked
- Some restricted UEs rejected → check for admission denied log

---

### Step 7 — What to verify in logs

```
# Emergency UE getting in:
pcf.baseline: emergency UE imsi-001010000000001 → tier1 full services
pcf.smpolicy: SM Policy Association created ... AMBR=20000/20000 rules=3

# Public UE getting in:
pcf.baseline: public UE imsi-001011234560001 → tier2 audio+sms only
pcf.smpolicy: SM Policy Association created ... AMBR=512/512 rules=2

# Restricted UE being blocked:
pcf.smpolicy: admission denied for imsi-001019999990001 — network congested (active_ues=152)

# When hitting 200 UEs — public also blocked:
pcf.smpolicy: admission denied for imsi-001011234560050 — network congested (active_ues=201)
```

---

### Step 8 — Record results (10 min)

Save as `1725/baseline_results.csv`:

```csv
time_s,active_ues,emergency_connected,public_connected,restricted_admitted,avg_emergency_ambr,avg_public_ambr
0,20,0,20,0,20000,512
60,150,10,100,12,20000,512
120,150,10,100,0,20000,512
...
300,150,10,100,0,20000,512
```

The column `restricted_admitted` should drop to 0 once UE count passes 150.  
That proves your admission control works.

---

## Checklist

```
[ ] ClassifyUE() returns 3 values: emergency / public / restricted
[ ] baseline_controller.go has AdmissionAllowed() + BaselineDecision()
[ ] BaselineDecision() sets PccRules with GBR/MBR/ARP per flow
[ ] smpolicy.Create() checks AdmissionAllowed() before proceeding
[ ] TrackUECreate/Delete wired
[ ] go build ./core/nf/pcf/... passes cleanly
[ ] baseline.yaml has emergency-pool + restricted-pool
[ ] Disaster run shows: emergency gets 3 rules, public gets 2, restricted gets 1
[ ] Admission denial logged when UE count exceeds threshold
[ ] baseline_results.csv saved
```

---

## Paper Table (this is Table II in your paper)

| Metric | Tier 1: Emergency | Tier 2: Public | Tier 3: Restricted |
|--------|------------------|----------------|-------------------|
| Session AMBR | 20 Mbps | 512 kbps | 64 kbps |
| Default 5QI | 1 | 1 | 5 |
| ARP Priority | 1 | 8 | 14 |
| GBR (voice) | 64/64 kbps | 64/64 kbps | — |
| MBR (voice) | 128/128 kbps | 128/128 kbps | — |
| GBR (video) | 1000/1000 kbps | — | — |
| MBR (video) | 4000/4000 kbps | — | — |
| Admission threshold | Always | > 200 UEs → block | > 150 UEs → block |

