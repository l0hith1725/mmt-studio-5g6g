# What Can the System Actually Control?
## Deep Dive: PCF Controls, UE Tiers, and Congestion Blocking

---

## Short Answer: No, it is NOT just AMBR throttling.

The PCF controls **five different things simultaneously** for each UE.
AMBR is only one of them. Here is the full picture.

---

## The 5 Knobs the PCF Controls Per UE

| Knob | What it does | Struct Field in Code |
|------|-------------|---------------------|
| **AMBR** | Max total bandwidth for the UE's entire PDU session | `SmPolicyDecision.SessionAMBRUL/DL` |
| **PCC Rules (allow/block)** | Which services the UE is ALLOWED to use at all | `SmPolicyDecision.PccRules` |
| **5QI** | Priority class — how the gNB scheduler treats this UE's packets | `PCCRule.FiveQI` |
| **ARP** | Who gets kicked off when the network is full | `PCCRule.ArpPriority` |
| **GBR/MBR** | Guaranteed and max bandwidth PER SERVICE (not total) | `PCCRule.GBRULKbps`, `PCCRule.MBRULKbps` |

Look at `core/nf/pcf/pcf.go` line 61–72:
```go
type PCCRule struct {
    ServiceName  string
    FiveQI       int         // ← Priority class for THIS service
    ResourceType string      // "GBR" or "NonGBR"
    ArpPriority  int         // ← Who gets pre-empted
    GBRULKbps    int         // ← Guaranteed bandwidth for THIS service
    GBRDLKbps    int
    MBRULKbps    int         // ← Max bandwidth for THIS service
    MBRDLKbps    int
    IsDefault    bool
}
```

And look at `mapMediaToServices()` in pcf.go line 587:
```go
case "audio":
    out = append(out, "conv_voice")  // 5QI 1 — voice service
case "video":
    out = append(out, "conv_video")  // 5QI 2 — video service
```

**This is the key insight:** The PCF already handles audio and video as
**separate PCC rules**. You can activate or deactivate each one independently.
This is exactly how you implement your three UE tiers.

---

## Your 3-Tier UE Classification System

### How the tiers work in 3GPP terms

| UE Tier | Who | Allowed Services | 5QI | ARP | AMBR |
|---------|-----|-----------------|-----|-----|------|
| **Tier 1: Emergency** | Ambulance, police, fire | Video + Audio + SMS | 1 (highest) | Priority 1, cannot be pre-empted | Full (20 Mbps) |
| **Tier 2: Semi-priority** | Hospital staff, critical infra | Audio + SMS (NO video) | 5 | Priority 3, protected | Medium (5 Mbps) |
| **Tier 3: Public** | Everyone else | SMS only (NO audio, NO video) | 9 (lowest) | Priority 15, can be pre-empted | Low (512 kbps) |

### How each tier is enforced — the mechanism

**Tier 1 (Emergency):**
- PCF activates PCC rules for: `conv_voice` (5QI 1) + `conv_video` (5QI 2) + `default_data` (SMS)
- All rules have ARP priority=1, pre-emption capability=YES, vulnerability=NO
- AMBR is never reduced

**Tier 2 (Semi-priority):**
- PCF activates: `conv_voice` (5QI 5) + `default_data` (SMS)
- PCF does NOT activate: `conv_video` — so video calls are simply BLOCKED
- ARP priority=3
- AMBR capped at 5 Mbps

**Tier 3 (Public):**
- PCF activates: ONLY `default_data` (SMS/data)
- `conv_voice` and `conv_video` are DEACTIVATED → audio and video calls BLOCKED
- ARP priority=15 → can be pre-empted to make room for Tier 1
- AMBR capped at 512 kbps

### The blocking mechanism in the code

When a UE tries to make a video call, the IMS sends a SIP INVITE.
The AF (in `services/ims/af.go`) calls `HandleAARequest()` in `pcf.go`.

Look at `HandleAARequest()` (line 546):
```go
func HandleAARequest(imsi string, mediaTypes []string, ...) bool {
    services := mapMediaToServices(mediaTypes)  // "video" → "conv_video"
    activated := DefaultPccRuleManager.ActivateRules(imsi, dnn, services)
    if len(activated) == 0 {
        // ← THIS IS WHERE YOU RETURN "DENIED"
        // If conv_video is not in the allowed list for this UE tier,
        // ActivateRules returns nothing, and the call is rejected
        return false
    }
    ...
}
```

**So for Tier 3 public users:** you set `conv_voice` and `conv_video` as
permanently INACTIVE_GATED in the PCCRuleManager. When they try to call,
`ActivateRules()` returns nothing, the AF returns false, and the SIP INVITE
is rejected with a "403 Forbidden" or "503 Service Unavailable".

---

## Blocking New UE Registrations (AMF-level control)

This is a **completely different control plane** — it operates at the **AMF**,
not the PCF.

### What happens during registration

When a new UE wants to connect:
1. UE sends **Registration Request** to gNB
2. gNB forwards it to **AMF** (NGAP)
3. AMF decides: accept or reject
4. If rejected, AMF sends **Registration Reject** with a cause code

The AMF can reject with cause codes including:
- `#22 — Congestion` (standard 3GPP TS 24.501 §9.11.3.2)
- `#72 — Not authorized in this PLMN`

### How to implement it

The AMF needs to:
1. Check the current congestion level (from NWDAF)
2. Check the UE's priority class
3. If congestion is predicted AND the new UE is Tier 3:
   → Send Registration Reject with cause `#22`
   → The UE will back off and retry after a timer

**Mathematically:** This is called **Access Class Barring (ACB)** or
**Extended Access Barring (EAB)** in 3GPP (TS 22.011).

The gNB can broadcast barring factors over System Information Block (SIB2):
- "10% of Tier 3 UEs are allowed to register" (during mild congestion)
- "0% of Tier 3 UEs" (during severe congestion)
- "100% of Tier 1 UEs always allowed"

### What your ML system does for registration blocking

```
NWDAF predicts: congestion in 30 seconds
DQN action: "block_tier3_registration"
Guardrail checks: "Is this a Tier 1 UE?" → No → Allow block

AMF receives the action:
  → Sets barring factor = 0% for Tier 3
  → Sets barring factor = 100% for Tier 1 (always)
  → Any new Tier 3 UE that tries to attach gets Registration Reject
```

---

## Complete Decision Table: What Happens at Each Congestion Level

This is the core of your DQN's action space.

| Congestion Level | LSTM Prediction | Tier 1 Emergency | Tier 2 Semi-priority | Tier 3 Public |
|-----------------|----------------|-----------------|---------------------|--------------|
| **None (normal)** | — | Video+Audio+SMS, 20 Mbps | Video+Audio+SMS, 20 Mbps | Video+Audio+SMS, 20 Mbps |
| **Low (25–50%)** | Optional | Video+Audio+SMS, 20 Mbps | Audio+SMS, 10 Mbps | Audio+SMS, 5 Mbps |
| **Medium (50–75%)** | 30s ahead | Video+Audio+SMS, 20 Mbps | Audio+SMS, 5 Mbps | SMS only, 1 Mbps |
| **High (75–90%)** | 30s ahead | Video+Audio+SMS, 20 Mbps | Audio+SMS, 2 Mbps | SMS only, 512 kbps |
| **Severe (>90%)** | Immediate | Video+Audio+SMS, 20 Mbps | Audio+SMS, 1 Mbps, GBR | SMS only, BLOCK new registration |
| **Critical** | Immediate | Video+Audio+SMS, 20 Mbps + GBR | Block video, Audio+SMS | Block ALL, kick out existing Tier 3 via ARP pre-emption |

---

## ARP Pre-emption: The Nuclear Option

When congestion is critical and emergency users still cannot connect,
the PCF can tell the AMF/SMF to **pre-empt** (forcibly disconnect)
existing Tier 3 sessions.

This is already supported in the code. In `pcf.go` the `PCCRule` struct has
`ArpPriority`. When an emergency UE cannot get resources:
1. AMF/SMF checks all existing sessions
2. Finds the lowest ARP priority session (highest number = lowest priority)
3. Sends a PDU Session Modification to release that session
4. Allocates those resources to the emergency UE

**In plain English:** A public user watching YouTube gets disconnected so that
an ambulance can make a video call. This is legal in 3GPP (TS 23.501 §5.7.2.2).

---

## What Your DQN Agent Actually Controls: Expanded Action Space

From the 20-day plan, the original action space was only AMBR throttling.
Now you know the full picture:

```python
ACTIONS = {
    # ── Tier 3 (Public) controls ──
    0:  "no_change",
    1:  "tier3_throttle_ambr_mild",      # AMBR 20M → 5M, all services allowed
    2:  "tier3_throttle_ambr_severe",    # AMBR 5M → 1M
    3:  "tier3_block_video",             # Deactivate conv_video PCC rule
    4:  "tier3_block_audio_and_video",   # Deactivate conv_voice + conv_video
    5:  "tier3_block_registration",      # AMF: reject new Tier 3 registrations
    6:  "tier3_preempt_existing",        # Release existing Tier 3 sessions via ARP

    # ── Tier 2 (Semi-priority) controls ──
    7:  "tier2_block_video",             # Deactivate conv_video for Tier 2
    8:  "tier2_throttle_ambr",           # Reduce Tier 2 AMBR
    9:  "tier2_block_registration",      # Reject new Tier 2 registrations

    # ── Recovery actions ──
    10: "restore_tier3_audio",           # Re-enable audio for Tier 3
    11: "restore_all_normal",            # Full restoration after congestion ends
}
```

**The DQN learns:** Start with action 1 when LSTM predicts 25% congestion.
If the congestion keeps increasing, escalate to 3, then 4, then 5. If it drops,
de-escalate. This learned escalation is what static rules cannot do.

---

## What Happens Inside the Code: The Full Chain

```
1. [Python LSTM]
   reads NWDAF data → predicts: "congestion in 30s, 87% confidence"

2. [Python DQN]
   state = {ue_count: 180, congestion_prob: 0.87, emergency_ues: 8}
   → selects action: "tier3_block_audio_and_video"

3. [Python Guardrail]
   checks: "is any Tier 1 UE being blocked?" → No → SAFE

4. [NWDAF Go code] calls PCF alert endpoint

5. [PCF Go code - pcf.go]
   for each Tier 3 UE:
       DefaultPccRuleManager.DeactivateRulesByName(imsi, "ims",
           []string{"conv_voice", "conv_video"})
       → those UEs' video/audio calls get a SIP 503

6. [PCF Go code - smpolicy.go]
   calls PushNotify() → OnUpdateNotify fires → SMF gets new SmPolicyDecision
   → SMF releases the GBR bearer for video/audio on Tier 3 UEs
   → gNB frees those radio resources

7. [AMF Go code - if blocking registration]
   AMF starts rejecting Registration Requests from Tier 3 UEs
   with cause #22 (Congestion)

8. Emergency UEs connect successfully.
```

---

## Is This a Good Idea? (Honest Assessment)

**Yes, and here is why this is a real, published approach:**

1. **Real disasters validate it:** The 2011 Japan earthquake caused cellular
   networks to fail because public users flooded the network. Japan's MIC
   mandated priority communications (JPY 2013 regulations) after this.

2. **3GPP already anticipated this:** The whole ARP + 5QI + pre-emption
   system was designed for exactly this scenario. You are not inventing new
   mechanisms — you are adding *intelligence* to when and how they are applied.

3. **The ML novelty is real:** Current deployments apply these rules manually
   or with static thresholds. Your LSTM predicts BEFORE it happens. Your DQN
   learns the optimal escalation sequence. This is not static spec-reading.

**The one real concern:** Fairness. When you block a public user from making
an audio call during a disaster, that person might be trying to find their
family. The paper should acknowledge this tradeoff and discuss it in the
conclusion as future work (e.g., a "missing person lookup" exemption).

---

## Summary: The 3 Layers of Control

```
Layer 1: PCC Rules (PCF)       — Which services are ALLOWED per UE tier
                                  (video, audio, SMS — per-service on/off)

Layer 2: AMBR / GBR / MBR      — How much bandwidth each UE and
          (PCF → SMF → UPF)      service gets (rate limiting)

Layer 3: Registration Barring   — Whether new UEs can even connect
          (AMF)                   (access class barring, cause #22)
```

Your ML system controls all three layers.
That is three points of control, not one.
That is also three novel contributions for your paper.

---

*Written for mmt-studio-5g6g project, Summer Capstone 2026.*
