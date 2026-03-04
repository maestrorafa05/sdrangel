# BladeRF2 Remote Control with SDRangel

**Purpose:** Control BladeRF2 via SDRangelSrv WebAPI and stream IQ data to remote SDRangel GUI

## 1. Start SDRangelSrv with BladeRF2

### Check available devices:

```cmd
sdrangelsrv --list-devices
```

Expected output shows:

```
Available devices:
 HWType: BladeRF2 Serial: 7cd55b47332f496cbb6e807f7377d6f0
 HWType: BladeRF2 Serial: 7cd55b47332f496cbb6e807f7377d6f0
```

### Start server:

```cmd
sdrangelsrv -a <REMOTE_IP> -p <REMOTE_PORT> --scratch
```

## 2. Configure BladeRF2 Device via WebAPI

### Verify server connection:

```bash
curl -s http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel | jq '{appname, architecture}'
```

### Query available devices remotely:

```bash
# Check main API info (might contain device info)
curl -s http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel | jq

# Try to get available hardware types
curl -s http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/devices | jq

# Check if there's a hardware enumeration endpoint
curl -s http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/hardware | jq

# Most likely working endpoint for hardware info
curl -s http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/hardwareinfo | jq

# Get current devicesets (shows configured devices)
curl -s http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel | jq '.devicesetlist'
```

### Access API documentation:

```bash
# Access Swagger UI in browser or get the spec
curl -s http://<REMOTE_IP>:<REMOTE_PORT>/swagger-ui
# or
curl -s http://<REMOTE_IP>:<REMOTE_PORT>/api-docs | jq
```

**Note:** The exact endpoint for device enumeration may vary by SDRangel version. Use Swagger documentation to find all available endpoints.

### Create deviceset:

```bash
curl -s -X POST "http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/deviceset?direction=0"
```

### Set device to BladeRF2:

```bash
curl -s -X PUT "http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/deviceset/0/device" \
  -H "Content-Type: application/json" \
  -d '{"hwType":"BladeRF2","direction":0}' | jq
```

## 3. Configure BladeRF2 Settings

### Check current settings structure:

```bash
curl -s "http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/deviceset/0/device/settings" | jq
```

### Configure for WiFi Channel 1 (2.412 GHz):

```bash
curl -s -X PUT "http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/deviceset/0/device/settings" \
  -H "Content-Type: application/json" \
  -d '{
    "bladeRF2InputSettings": {
      "LOppmTenths": 0,
      "bandwidth": 20000000,
      "biasTee": 0,
      "centerFrequency": 2412000000,
      "dcBlock": 0,
      "devSampleRate": 20000000,
      "fcPos": 0,
      "gainMode": 0,
      "globalGain": 50,
      "iqCorrection": 0,
      "iqOrder": 1,
      "log2Decim": 0,
      "reverseAPIAddress": "127.0.0.1",
      "reverseAPIDeviceIndex": 0,
      "reverseAPIPort": 8888,
      "transverterDeltaFrequency": 0,
      "transverterMode": 0,
      "useReverseAPI": 0
    },
    "deviceHwType": "BladeRF2",
    "direction": 0
  }' | jq
```

**Key WiFi Channel 1 Settings:**

- **centerFrequency**: `2412000000` (2.412 GHz)
- **devSampleRate**: `20000000` (20 MS/s)
- **bandwidth**: `20000000` (20 MHz)
- **globalGain**: `50` (50 dB)

## 4. Control Data Acquisition

### Start data acquisition:

```bash
curl -s -X POST "http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/deviceset/0/device/run" | jq
```

### Stop data acquisition:

```bash
curl -s -X DELETE "http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/deviceset/0/device/run" | jq
```

### Check device status:

```bash
curl -s "http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/deviceset/0/device/run" | jq '.state'
```

## 5. Setup Remote Streaming

### Add RemoteSink channel for streaming IQ data:

```bash
curl -s -X POST "http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/deviceset/0/channel" \
  -H "Content-Type: application/json" \
  -d '{"channelType":"RemoteSink","direction":0}' | jq
```

### Configure RemoteSink to stream to remote client:

```bash
curl -s -X PUT "http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/deviceset/0/channel/0/settings" \
  -H "Content-Type: application/json" \
  -d '{
    "RemoteSinkSettings": {
      "dataAddress": "<LOCAL_IP>",
      "dataPort": <LOCAL_PORT>,
      "nbFECBlocks": 0
    },
    "channelType": "RemoteSink",
    "direction": 0
  }' | jq
```

**Replace `<LOCAL_IP>`** with the actual IP address of your local SDRangel GUI.

## 6. Configure Remote SDRangel GUI

### In SDRangel GUI:

1. **Add Rx device** → **RemoteInput**
2. Configure RemoteInput:
   - **API Address**: `<REMOTE_IP>`
   - **API Port**: `<REMOTE_PORT>`
   - **Data Address**: `<REMOTE_IP>`
   - **Data Port**: `<LOCAL_PORT>`
   - **Device Index**: `0`
3. Click **Apply**
4. Click **Play ▶** to start receiving

## 7. Verification Commands

### Check streaming configuration:

```bash
# Verify RemoteSink settings
curl -s "http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/deviceset/0/channel/0/settings" | jq '.RemoteSinkSettings'

# Check device status
curl -s "http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel/deviceset/0/device/run" | jq '.state'

# Check deviceset info
curl -s http://<REMOTE_IP>:<REMOTE_PORT>/sdrangel | jq '.devicesetlist'
```

## 8. Data Flow Summary

```
Remote Server (<REMOTE_IP>):
BladeRF2 → SDRangelSrv → RemoteSink → UDP:<LOCAL_PORT>
                                         ↓
Local Client (<LOCAL_IP>):               ↓
SDRangel GUI ← RemoteInput ← UDP:<LOCAL_PORT> ←──┘
```

## Notes

- **Use PUT instead of PATCH** for device settings
- **Complete JSON object required** - partial updates may fail
- **Case sensitive**: Use `bladeRF2InputSettings` (lowercase 'b')
- **Field names**: Use `devSampleRate` not `sampleRate`
- **Stop device before major setting changes** for best results

This setup provides full remote control of BladeRF2 with real-time IQ streaming to remote clients.
