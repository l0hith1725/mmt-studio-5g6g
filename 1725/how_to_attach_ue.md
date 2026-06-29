# How to Attach a UE — Complete Step-by-Step Guide
## For Remote Linux Machine Setup

---

## The Problem

You cannot just click "Register UE" and expect it to work.
There is a **mandatory 4-step sequence** that must happen in order.
If any step is skipped, the next one will fail.

```
Step 1: Verify both containers are running
Step 2: Create + connect the gNB (simulated base station)
Step 3: Load UEs into the tester's memory
Step 4: Register (attach) a UE
Step 5: (optional) Establish a PDU session (data connection)
```

---

## Access the Remote Machine

Replace `<REMOTE_IP>` with your Linux server's IP address everywhere below.

### Option A: SSH Port Forwarding (Recommended — Access from your Mac)

Run this on your Mac. It tunnels the remote ports to your local machine:

```bash
ssh -L 5000:<REMOTE_IP>:5000 \
    -L 5001:<REMOTE_IP>:5001 \
    user@<REMOTE_IP>
```

After this, on your Mac:
- Core UI → http://localhost:5000
- Tester UI → http://localhost:5001
- All curl commands below use `localhost`

### Option B: Use curl directly on the remote machine

SSH into the machine and run all the curl commands there:
```bash
ssh user@<REMOTE_IP>
# Then run curl commands using localhost
```

---

## Step 0: Verify the Stack is Running

```bash
# On the remote machine:
docker ps
```

You should see THREE containers:
```
CONTAINER ID   IMAGE                    STATUS     NAMES
xxxxxxxxxxxx   mmt-studio-core:dev      Up X min   sacore
xxxxxxxxxxxx   mmt-studio-tester:dev    Up X min   satraffic
xxxxxxxxxxxx   mmt-studio-tester:dev    Up X min   satester
```

**If you see nothing or errors:**
```bash
cd /path/to/mmt-studio-5g6g/orchestrate
./run_studio.sh
```

Wait 2–3 minutes for the build. Then `docker ps` again.

**Verify the tester API is responding:**
```bash
curl http://localhost:5001/api/gnbs
# Expected: {"items": []}   ← empty list is fine, means tester is alive
```

**Verify the core API is responding:**
```bash
curl http://localhost:5000/api/status
# Expected: some JSON with NF status
```

If either curl fails with "connection refused" → the container is not running.

---

## Step 1: Create and Connect the gNB

The gNB (base station) is a simulated piece of radio hardware.
It must be created and connected to the AMF BEFORE any UE can attach.

### Via the Tester Web UI (http://localhost:5001)

1. Open the Tester UI
2. Find the **gNB** section or tab
3. Look for a button: **"Apply Profile"** or **"Add gNB"**
4. Select the profile named `tester-gnb-00`
5. Click **Connect** — you should see the state change to **READY**

### Via curl (if UI is not accessible)

```bash
# Step 1a: Apply the saved gNB profile
curl -X POST http://localhost:5001/api/gnb-config/tester-gnb-00/apply

# Expected response:
# {"ok": true, "gnb": {"gnb_name": "tester-gnb-00", "state": "IDLE", ...}}
```

```bash
# Step 1b: Connect the gNB to the AMF (NG Setup procedure)
curl -X POST http://localhost:5001/api/gnbs/0/connect

# Expected response:
# {"ok": true, "state": "READY"}
```

**If you get `"state": "FAILED"` or `"state": "IDLE"` instead of `"READY"`:**
- The gNB could not reach the AMF.
- Check the core is running: `docker logs sacore --tail 50`
- Check the network bridge: `docker network ls | grep mmtnet`

**Verify gNB is READY:**
```bash
curl http://localhost:5001/api/gnbs
# Look for: "state": "READY"
```

---

## Step 2: Load UEs into the Tester

The tester stores UE SIM credentials (IMSI, K, OPc) in a database.
They must be loaded into memory before you can attach them.

### Via Web UI

Find the **UE** section → click **"Load SIMs"** or **"Load UEs from DB"**

### Via curl

```bash
# Load all UEs from the sim database into memory
curl -X POST http://localhost:5001/api/ues/load-sims

# Expected response:
# {"ok": true, "loaded": 128, "total": 128}
```

**Verify UEs are loaded:**
```bash
curl http://localhost:5001/api/ues
# You should see a list of 128 UEs
```

**If `"loaded": 0`** — the SIM database is empty. 
You need to provision UEs from the core:
```bash
# Sync UEs from the core's database to the tester
curl -X POST http://localhost:5001/api/core/sync-ues
```

---

## Step 3: Register (Attach) a UE

Now you can finally attach a UE.

The first UE in the baseline config has IMSI: `001011234560001`

### Via Web UI

1. In the UE list, find IMSI `001011234560001`
2. Click **"Register"** or **"Attach"**
3. Watch the state change: `IDLE` → `REGISTERED`

### Via curl

```bash
# Attach UE with IMSI 001011234560001
curl -X POST http://localhost:5001/api/ues/001011234560001/register

# Expected response:
# {"ok": true, "state": "REGISTERED"}
```

**If you get `"UE not found"`:**
- Run Step 2 again (load-sims)

**If you get `"No gNBs"`:**
- Run Step 1 again (create + connect gNB)

**If you get `"gNB not ready (IDLE)"` or `"gNB not ready (FAILED)"`:**
- The NG Setup between gNB and AMF failed
- Check: `docker logs sacore --tail 100 | grep -i "ngap\|amf\|setup"`

---

## Step 4: Establish a PDU Session (Data Connection)

After registration, the UE needs a PDU Session to have a data connection
(this is what gets an IP address and is what the PCF applies policies to).

### Via curl

```bash
# Establish PDU session for the registered UE
curl -X POST http://localhost:5001/api/ues/001011234560001/pdu-session \
  -H "Content-Type: application/json" \
  -d '{"dnn": "internet", "sst": 1, "psi": 1}'

# Expected response:
# {"ok": true}
```

**After this**, the PCF in the core will have an active `SmPolicyAssociation`
for this UE. You can verify in the core logs:

```bash
docker logs sacore --tail 50 | grep "pcf.smpolicy"
# You should see:
# SM Policy Association created ctxRef=smpolicy-001011234560001-1-1
```

---

## Full One-Shot Script: Attach a UE From Scratch

Save this as `attach_ue.sh` on the remote machine and run it:

```bash
#!/bin/bash
# attach_ue.sh — Attach one UE from scratch on the MMT stack.
# Run from the remote Linux machine (or with SSH port forwarding).

TESTER="http://localhost:5001"
IMSI="001011234560001"

echo "=== MMT UE Attach Script ==="
echo ""

# Step 1: Apply gNB profile
echo "[1/5] Applying gNB profile..."
gnb=$(curl -s -X POST "$TESTER/api/gnb-config/tester-gnb-00/apply")
echo "      $gnb"

# Step 2: Connect gNB to AMF
echo "[2/5] Connecting gNB to AMF (NG Setup)..."
sleep 1
conn=$(curl -s -X POST "$TESTER/api/gnbs/0/connect")
echo "      $conn"

gnb_state=$(echo "$conn" | python3 -c "import sys,json; print(json.load(sys.stdin).get('state','?'))" 2>/dev/null)
if [ "$gnb_state" != "READY" ]; then
  echo ""
  echo "ERROR: gNB state is '$gnb_state', not READY."
  echo "       Check: docker logs sacore --tail 50 | grep -i ngap"
  exit 1
fi
echo "      gNB is READY ✓"

# Step 3: Load SIMs
echo "[3/5] Loading UE SIMs into memory..."
sims=$(curl -s -X POST "$TESTER/api/ues/load-sims")
echo "      $sims"

# Step 4: Register UE
echo "[4/5] Registering UE $IMSI..."
sleep 1
reg=$(curl -s -X POST "$TESTER/api/ues/$IMSI/register")
echo "      $reg"

ue_state=$(echo "$reg" | python3 -c "import sys,json; print(json.load(sys.stdin).get('state','?'))" 2>/dev/null)
if [ "$ue_state" != "REGISTERED" ]; then
  echo ""
  echo "ERROR: UE state is '$ue_state', not REGISTERED."
  echo "       Check: docker logs sacore --tail 100 | grep -i 'amf\|auth\|nas'"
  exit 1
fi
echo "      UE is REGISTERED ✓"

# Step 5: PDU Session
echo "[5/5] Establishing PDU Session (internet, SST=1)..."
sleep 1
pdu=$(curl -s -X POST "$TESTER/api/ues/$IMSI/pdu-session" \
  -H "Content-Type: application/json" \
  -d '{"dnn": "internet", "sst": 1, "psi": 1}')
echo "      $pdu"

echo ""
echo "=== Done! ==="
echo ""
echo "Verify in core logs:"
echo "  docker logs sacore --tail 30 | grep pcf.smpolicy"
echo ""
echo "Verify NWDAF sees the UE:"
echo "  curl http://localhost:5000/api/nwdaf/analytics?analyticsId=UE_MOBILITY"
```

Make it executable and run:
```bash
chmod +x attach_ue.sh
./attach_ue.sh
```

---

## Troubleshooting

### Error: "connection refused" on port 5001
```bash
docker ps | grep satester
# If not running:
cd orchestrate && ./run_studio.sh
```

### Error: "gNB not ready (FAILED)"
```bash
# Check if core AMF is reachable from tester
docker exec satester ping -c 2 172.30.0.10

# Check AMF is listening on NGAP port
docker exec sacore ss -lnp | grep 38412

# Check core logs
docker logs sacore --tail 100 | grep -i "amf\|ngap\|setup"
```

### Error: "UE not found" even after load-sims
```bash
# Check if sim DB has UEs
curl http://localhost:5001/api/sim-db | python3 -m json.tool | head -30

# If empty, provision from core
curl -X POST http://localhost:5001/api/core/sync-ues
```

### Error: Authentication failure in core logs
```bash
# The K/OPc in the tester SIM doesn't match what the core has
# Reset the core DB and resync
curl -X POST http://localhost:5000/api/admin/remove-db-file   # reset core DB
curl -X POST http://localhost:5001/api/core/sync-ues          # reprovision
```

### After reboot: gNB always starts IDLE
```bash
# The tester's in-memory state is lost on restart.
# You must ALWAYS run Steps 1-2 again after every restart.
# The gNB pool is NOT persisted between restarts.
```

---

## What You Should See in Core Logs When It Works

```bash
docker logs sacore -f | grep -E "(amf|pcf|smf|nwdaf)" --line-buffered
```

Successful attach looks like this sequence in the logs:
```
[amf]        Registration Request: IMSI=001011234560001
[ausf]       5G-AKA auth started: IMSI=001011234560001
[amf]        Security Mode Complete: IMSI=001011234560001  
[amf]        Registration Accept sent: IMSI=001011234560001
[smf]        PDU Session Establishment: IMSI=001011234560001 DNN=internet
[pcf.smpolicy] SM Policy Association created ctxRef=smpolicy-... AMBR=200000kbps
[upf]        GTP-U tunnel created: UE-IP=10.45.0.2
[nwdaf]      Data collected: UE_MOBILITY current_ues=1
```

The line `SM Policy Association created` is the PCF applying the first policy.
**This is the exact line your ML system will change** when you implement the research.

---

*Saved in mmt-studio-5g6g/1725/ — Summer Capstone 2026*



```
how to create a pdu-session::

curl -X POST http://localhost:5001/api/ues/001011234560001/pdu-session \
  -H "Content-Type: application/json" \
  -d '{"dnn": "internet", "sst": 1, "psi": 1}'

```