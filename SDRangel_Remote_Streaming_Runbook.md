# SDRangel Remote Streaming (WSL) — Setup & Operation Runbook

**Audience:** Public users and anyone reading this guide in the SDRangel folder  
**Purpose:** Provide a repeatable procedure to run **SDRangelSrv** in **WSL (Ubuntu)** and stream IQ data to **SDRangel GUI** using **RemoteSink (UDP)** → **RemoteInput** on `localhost`.

This runbook covers:
- Concepts and data-flow overview
- Environment prerequisites and installation
- Server start (SDRangelSrv)
- Server configuration (TestSource → RemoteSink UDP)
- Verification (API + tcpdump)
- GUI setup (RemoteInput)
- Operational checklist, performance expectations, troubleshooting, and full reset

---

## 1) What “Done” Looks Like (Acceptance Criteria)

You have met the requirement when all of the following are true:

1. **WebAPI reachable**  
   `http://127.0.0.1:8091/sdrangel` returns JSON and includes `"appname": "SDRangelSrv"`.

2. **Server has an active RX deviceset and a RemoteSink channel**  
   `/sdrangel` shows `devicesetcount: 1` and `channelcount: 1` for deviceset 0.

3. **IQ stream is actively flowing via UDP**  
   `tcpdump` shows continuous UDP packets to `127.0.0.1:9090`.

4. **Client GUI receives stream**  
   SDRangel GUI “RemoteInput” stays **ON** (does not stop after 1–2 seconds) and spectrum/waterfall shows activity.

5. **No active error state**  
   `GET /sdrangel/deviceset/0/device/run` returns a state that is not `"error"`.

---

## 2) Concepts and Data Flow (Quick Context)

### 2.1 Terms used in this runbook

- **SDRangelSrv:** Headless server mode (WebAPI + DSP/device engine) in WSL.
- **SDRangel GUI:** Desktop client that visualizes and demodulates received IQ.
- **TestSource:** Virtual IQ source (stable and hardware-independent).
- **RemoteSink:** Server-side channel that sends IQ over UDP.
- **RemoteInput:** Client-side device that receives IQ over UDP.

### 2.2 Data-flow diagram

```text
WSL (Server)                                        GUI (Client)
-----------------------------                      ---------------------------
TestSource (IQ generator)
        |
        v
RemoteSink channel  --UDP 127.0.0.1:9090-->  RemoteInput device --> Spectrum/Waterfall/Demod
        |
        +-- controlled via WebAPI 127.0.0.1:8091
```

---

## 3) Environment & Assumptions

- **Host OS:** Windows
- **Linux:** WSL (Ubuntu 24.04 recommended)
- **Server runtime:** `sdrangelsrv` inside WSL
- **Client runtime:** SDRangel GUI (WSLg or native Windows build)
- **Ports:**
  - WebAPI control: `127.0.0.1:8091`
  - IQ UDP stream: `127.0.0.1:9090`
- **Streaming mode:** Localhost loopback for repeatable baseline test

---

## 4) Prerequisites (WSL / Ubuntu)

### 4.1 Install required utilities

```bash
sudo apt update
sudo apt install -y curl tcpdump lsof jq
```

### 4.2 Ensure SDRangel server binary is available

```bash
command -v sdrangelsrv
sdrangelsrv -h | head
```

Expected:
- A valid executable path is printed (example: `/usr/bin/sdrangelsrv`)
- Help text starts with usage/options output

### 4.3 If `sdrangelsrv` is missing

Use one of these:

1. Install from package source available in your environment, or
2. Build from source using the repository build instructions and ensure the produced `sdrangelsrv` is in PATH (or call it via full path).

Validation command after installation/build:

```bash
sdrangelsrv -h | head -n 5
```

---

## 5) Start the Server (SDRangelSrv)

### 5.1 Ensure port 8091 is free

```bash
sudo lsof -iTCP:8091 -sTCP:LISTEN || true
```

If a PID is listed:

```bash
sudo kill -9 <PID>
```

Re-check:

```bash
sudo lsof -iTCP:8091 -sTCP:LISTEN || true
```

### 5.2 Start SDRangelSrv in clean state and save logs

Open **WSL Terminal #1**:

```bash
rm -f ~/sdrangelsrv.log
sdrangelsrv -a 127.0.0.1 -p 8091 --scratch 2>&1 | tee -a ~/sdrangelsrv.log
```

Keep this terminal running.

### 5.3 Verify WebAPI is reachable

Open **WSL Terminal #2**:

```bash
curl -s http://127.0.0.1:8091/sdrangel | jq '{appname, architecture, dspRxBits, dspTxBits}'
```

Expected output contains:

```json
{
  "appname": "SDRangelSrv"
}
```

---

## 6) Configure Streaming: TestSource → RemoteSink (UDP 9090)

This section builds server pipeline:

**TestSource (IQ generator)** → **RemoteSink (UDP streamer)**

### 6.1 Create RX deviceset 0

```bash
curl -s http://127.0.0.1:8091/sdrangel | jq '.devicesetlist.devicesetcount'
```

Confirm deviceset count:

```bash
curl -s http://127.0.0.1:8091/sdrangel | jq '.devicesetlist.devicesetcount'
```

Expected: `1`

### 6.2 Set device to TestSource

```bash
curl -s -X PUT "http://127.0.0.1:8091/sdrangel/deviceset/0/device" \
  -H "Content-Type: application/json" \
  -d '{"hwType":"TestSource","direction":0}' | jq '{hwType, direction}'
```

Expected output includes `"hwType": "TestSource"`.

### 6.3 Add RemoteSink channel

```bash
curl -s -X POST "http://127.0.0.1:8091/sdrangel/deviceset/0/channel" \
  -H "Content-Type: application/json" \
  -d '{"channelType":"RemoteSink","direction":0}' | jq '{channelType, direction}'
```

Expected output includes `"channelType": "RemoteSink"`.

### 6.4 Configure RemoteSink output to localhost:9090 (FEC disabled)

```bash
curl -s -X PATCH "http://127.0.0.1:8091/sdrangel/deviceset/0/channel/0/settings" \
  -H "Content-Type: application/json" \
  -d '{
    "RemoteSinkSettings": {
      "dataAddress": "127.0.0.1",
      "dataPort": 9090,
      "nbFECBlocks": 0
    },
    "channelType": "RemoteSink",
    "direction": 0
  }' | jq '.RemoteSinkSettings | {dataAddress, dataPort, nbFECBlocks}'
```

Expected:

```json
{
  "dataAddress": "127.0.0.1",
  "dataPort": 9090,
  "nbFECBlocks": 0
}
```

### 6.5 Start device and verify run-state

```bash
curl -s -X POST "http://127.0.0.1:8091/sdrangel/deviceset/0/device/run" | jq
curl -s "http://127.0.0.1:8091/sdrangel/deviceset/0/device/run" | jq
```

Expected:
- `state` is typically `"running"`
- `state` must not be `"error"`

---

## 7) Validate Streaming (Server-Side Proof)

### 7.1 Confirm UDP packets flow to port 9090

```bash
sudo tcpdump -i lo -n udp port 9090 -c 10
```

Expected repeated packets:

```text
127.0.0.1:<ephemeral_port> > 127.0.0.1.9090: UDP, length ...
```

This is the strongest server-side evidence that IQ stream is active.

### 7.2 Confirm deviceset/channel counts via API

```bash
curl -s http://127.0.0.1:8091/sdrangel | jq '.devicesetlist | {devicesetcount, devicesetfocus}'
curl -s http://127.0.0.1:8091/sdrangel | jq '.devicesetlist.deviceSets[0].channelcount'
```

Expected:
- `devicesetcount = 1`
- `channelcount = 1`

### 7.3 Fast health-check (single command)

```bash
echo "APP=$(curl -s http://127.0.0.1:8091/sdrangel | jq -r '.appname')"; \
echo "DEVCOUNT=$(curl -s http://127.0.0.1:8091/sdrangel | jq -r '.devicesetlist.devicesetcount')"; \
echo "RUNSTATE=$(curl -s http://127.0.0.1:8091/sdrangel/deviceset/0/device/run | jq -r '.state')"
```

---

## 8) Configure SDRangel GUI (Client) — RemoteInput

### 8.1 Add Rx device: RemoteInput

In SDRangel GUI:

1. Click **Add Rx device**
2. Select **RemoteInput**
3. Set:
   - **Remote (API):** `127.0.0.1` port `8091`
   - **Data (UDP):** `127.0.0.1` port `9090`
   - **FEC:** disabled / `0`
4. Click **Set** (if present)
5. Click **Play (▶)** on RemoteInput

Expected:
- RemoteInput stays **ON** (no auto-stop after 1–2 seconds)
- Spectrum/waterfall shows activity

If RemoteInput stops quickly, immediately re-check Section **7.1** (`tcpdump`).

---

## 9) Add Channels / Demodulators (Optional)

Once RemoteInput is running:

1. Click **Add channel**
2. Choose demodulator (NFM/WFM/AM/SSB)
3. Tune **Δf (delta frequency)** by clicking signal peak in spectrum

### Note on Δf limits

Manual Δf is bounded by Nyquist: approximately **± sampleRate/2**.  
Example: sample rate `48 kHz` → max tunable Δf around `±24 kHz`.

---

## 10) Performance Expectations (Localhost Baseline)

Typical expected baseline on localhost (`127.0.0.1`) for this demo:

- **Startup time:** API ready in a few seconds
- **Transport loss:** near zero on localhost
- **Client behavior:** RemoteInput remains stable ON
- **Latency:** low and visually near-real-time in spectrum/waterfall

If your system behaves significantly worse, inspect CPU load, server logs, and UDP flow (Section **7.1**).

---

## 11) Troubleshooting (Ranked by Probability)

### 11.1 RemoteInput stops after 1–2 seconds (Most Common)

**Likely cause #1:** No UDP data reaches `127.0.0.1:9090`.

Check:

```bash
sudo tcpdump -i lo -n udp port 9090 -c 10
```

If no packets:
- Re-run Section **6** exactly
- Confirm run state:

```bash
curl -s http://127.0.0.1:8091/sdrangel/deviceset/0/device/run | jq
```

### 11.2 API reachable but no streaming

**Likely cause #2:** RemoteSink exists but settings are not applied.

Check:

```bash
curl -s http://127.0.0.1:8091/sdrangel/deviceset/0/channel/0/settings | jq '.RemoteSinkSettings'
```

Expected:
- `dataAddress = "127.0.0.1"`
- `dataPort = 9090`
- `nbFECBlocks = 0`

### 11.3 Server cannot bind to port 8091

Symptom:
- `Cannot bind on port 8091: The bound address is already in use`

Fix:

```bash
sudo lsof -iTCP:8091 -sTCP:LISTEN
sudo kill -9 <PID>
```

Restart server (Section **5.2**).

### 11.4 GUI connected to wrong endpoint

Verify GUI values:
- API host/port = `127.0.0.1:8091`
- UDP host/port = `127.0.0.1:9090`
- FEC disabled (`0`) if server uses `nbFECBlocks: 0`

### 11.5 DBus / Positioning warning in WSL

Example warning:
- `org.freedesktop.DBus.Error.NoReply`

For this demo scenario, it is usually non-blocking.

---

## 12) Quick Restart Procedure (Daily Use)

1) Free API port if needed:

```bash
sudo lsof -iTCP:8091 -sTCP:LISTEN
sudo kill -9 <PID>
```

2) Start server:

```bash
sdrangelsrv -a 127.0.0.1 -p 8091 --scratch
```

3) Re-run Section **6** (pipeline config) and Section **7** (validation).

4) Start RemoteInput in GUI (Section **8**).

---

## 13) Full Reset Procedure (When Stuck)

Use this only if normal restart does not resolve issues.

1) Stop server process:

```bash
pkill -9 sdrangelsrv || true
```

2) Ensure no listener remains on 8091:

```bash
sudo lsof -iTCP:8091 -sTCP:LISTEN || true
```

3) Optional: clear only this runbook log artifact:

```bash
rm -f ~/sdrangelsrv.log
```

4) Start clean again from Section **5**.

> Optional destructive reset (not required for normal operation): clearing all SDRangel user config will remove saved preferences/sessions.

---

## 14) Operational Evidence Checklist (What to Share)

### 14.1 Server-side evidence (required)

- API reachable:

```bash
curl -s http://127.0.0.1:8091/sdrangel | jq '.appname'
```

- Active deviceset and channel:

```bash
curl -s http://127.0.0.1:8091/sdrangel | jq '.devicesetlist.devicesetcount, .devicesetlist.deviceset[0].channelcount'
```

- Active UDP flow proof:

```bash
sudo tcpdump -i lo -n udp port 9090 -c 10
```

### 14.2 Client-side evidence (recommended)

- Screenshot: RemoteInput running (Play ON) with visible spectrum/waterfall activity

---

## Appendix A) One-Line Sanity Checks

- API + device count:

```bash
curl -s http://127.0.0.1:8091/sdrangel | jq '.appname, .devicesetlist.devicesetcount'
```

- Device run state:

```bash
curl -s http://127.0.0.1:8091/sdrangel/deviceset/0/device/run | jq '.state'
```

- RemoteSink key settings:

```bash
curl -s http://127.0.0.1:8091/sdrangel/deviceset/0/channel/0/settings | jq '.RemoteSinkSettings | {dataAddress, dataPort, nbFECBlocks}'
```

---

## Appendix B) Known Good Minimal Sequence 

```bash
pkill -9 sdrangelsrv || true
sdrangelsrv -a 127.0.0.1 -p 8091 --scratch >/tmp/sdrangelsrv.log 2>&1 &

# wait until WebAPI is ready (max 30s)
timeout 30 bash -c 'until curl -fsS http://127.0.0.1:8091/sdrangel >/dev/null; do sleep 1; done'

curl -s -X POST "http://127.0.0.1:8091/sdrangel/deviceset?direction=0" >/dev/null
curl -s -X PUT  "http://127.0.0.1:8091/sdrangel/deviceset/0/device" \
  -H "Content-Type: application/json" \
  -d '{"hwType":"TestSource","direction":0}' >/dev/null

curl -s -X POST "http://127.0.0.1:8091/sdrangel/deviceset/0/channel" \
  -H "Content-Type: application/json" \
  -d '{"channelType":"RemoteSink","direction":0}' >/dev/null

curl -s -X PATCH "http://127.0.0.1:8091/sdrangel/deviceset/0/channel/0/settings" \
  -H "Content-Type: application/json" \
  -d '{"RemoteSinkSettings":{"dataAddress":"127.0.0.1","dataPort":9090,"nbFECBlocks":0},"channelType":"RemoteSink","direction":0}' >/dev/null

curl -s -X POST "http://127.0.0.1:8091/sdrangel/deviceset/0/device/run" | jq
sudo tcpdump -i lo -n udp port 9090 -c 5
```
