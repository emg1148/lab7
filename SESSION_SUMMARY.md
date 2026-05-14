# Lab 7 LiveKit ESP32 Session Summary

## Session Overview
Date: May 12-14, 2026  
Project: LiveKit minimal_video example adapted for bird feeder camera setup  
Target: ESP32-P4 with OV5647 camera + ES8311 codec

---

## Initial Problem

### CMake Version Constraint Error
```
CMake Error at build.cmake:629:
  ERROR: Version solving failed:
  - no versions of livekit/livekit match 0.3.7
  - project depends on livekit/livekit (0.3.7)
```

**Root Cause**: The component manifest had an exact version constraint (0.3.7) for a local override path, but the version wasn't in the public registry.

**Solution**: Updated `main/idf_component.yml` to use flexible versioning:
```yaml
livekit/livekit:
  version: ">=0.3.7"  # Changed from "0.3.7" to ">=0.3.7"
  override_path: ../../../
```

---

## Architecture Decision: SDK Pivot

### Original Plan (Rejected)
- Use WHIP ingress with esp_peer component
- Complex setup requiring manual ingress creation and token management

### Final Approach (Adopted)
- **Official LiveKit ESP32 SDK** (developer preview, v0.3.6+)
- First-party support for ESP32-P4
- Native H.264 video publishing
- Opus audio with AEC
- Native room connection via JWT (no WHIP indirection)

**Rationale**: Cleaner, first-party maintained SDK with direct LiveKit integration.

---

## Project Configuration

### 1. Board & Camera Setup

**Hardware**: ESP32-P4-Function-EV-Board with OV5647 MIPI CSI camera + ES8311 codec

**Source**: Adapted from Lab 6 doorbell_demo  
- Codec board: `ESP32_P4_EYE`
- Camera: OV5647 (MIPI RAW10 1280×960 binning @ 45fps native output)

**Files Modified**:
- `sdkconfig.defaults`: Set camera to OV5647, codec board type
- `main/Kconfig.projbuild`: Hardcoded codec board default
- `main/main.c`: Added hardcoded WiFi credentials

### 2. WiFi Configuration

**Problem**: `livekit_example_utils` component expected WiFi credentials but menuconfig approach didn't work.

**Solution**: Hardcoded credentials directly in `main/main.c`:
```c
#define WIFI_SSID     "iPhone (59)"
#define WIFI_PASSWORD "Erica123"
```

The `lk_example_network_connect()` utility function reads these `#define` values.

### 3. LiveKit Room Connection

**Token**: Generated from LiveKit Cloud dashboard  
- Room: `birdfeeder`
- Participant identity: `esp32-cam`
- Grants: room join, publish (audio+video), subscribe (audio)
- Format: JWT (starts with `eyJhbGc...`)

**Configuration Method**: Pregenerated token (Option B in example)
- Updated `sdkconfig.defaults`:
  ```
  CONFIG_LK_EXAMPLE_USE_PREGENERATED=y
  ```
- Hardcoded in `main/example.c`:
  ```c
  livekit_room_connect(room_handle,
      "wss://lab7-ifvw37bc.livekit.cloud",
      "<JWT token>");
  ```

---

## Video Streaming Issues & Resolution

### Issue 1: WiFi Connection Failures
**Error**: `E (1610) network_connect: WiFi SSID is empty`  
**Resolution**: Added WiFi `#define` to main.c for external component to find.

### Issue 2: One Frame Then Freeze
**Symptom**: Stream showed one image then stopped updating  
**Investigation**: 
- Room connected successfully (logs confirmed)
- Audio encoder initialized
- Video encoder initialized  
- But frames weren't flowing

**Initial Hypothesis**: Video encoder not being triggered  
**Test**: Attempted explicit `esp_capture_start()` call  
**Result**: Warning "Already started" — capture was auto-starting on init

### Issue 3: Resolution Mismatch
**Problem**: Attempting to resize camera output from native 1280×960 to lower resolution failed  
**Finding**: Camera only outputs natively at 1280×960 (OV5647_MIPI_RAW10_1280X960_BINNING_45FPS)  
**Solution**: Keep resolution at 1280×960, reduce FPS instead

### Issue 4: Slow/Frozen Video Stream
**Symptom**: Updated very slowly or froze on single frame even after successful room connection  
**Cause**: 1280×960 H.264 @ 15fps was saturating ESP32-P4 video encoder memory/bandwidth  
**Root**: Encoder couldn't keep up with frame requests at high resolution/FPS

**Final Solution**: Reduce FPS from 15 to 10
```c
.video_encode = {
    .codec = LIVEKIT_VIDEO_CODEC_H264,
    .width = 1280,
    .height = 960,
    .fps = 10      // Reduced from 15
},
```

**Rationale**: 
- Keeps native camera resolution (no costly resizing)
- Reduces encoder workload by ~33%
- 10fps acceptable for surveillance/bird feeder (minimal motion)
- Smooth streaming achieved

---

## Final Configuration Summary

### sdkconfig.defaults (Key Settings)
```ini
CONFIG_IDF_TARGET="esp32p4"
CONFIG_LK_EXAMPLE_CODEC_BOARD_TYPE="ESP32_P4_EYE"
CONFIG_LK_EXAMPLE_USE_PREGENERATED=y
CONFIG_CAMERA_OV5647=y
CONFIG_CAMERA_OV5647_MIPI_RAW10_1280X960_BINNING_45FPS=y
CONFIG_CAMERA_XCLK_USE_ESP_CLOCK_ROUTER=y
CONFIG_LK_EXAMPLE_SPEAKER_VOLUME=85
```

### main/main.c (WiFi Defines)
```c
#define WIFI_SSID     "iPhone (59)"
#define WIFI_PASSWORD "Erica123"
```

### main/example.c (Video Encoder)
```c
.video_encode = {
    .codec = LIVEKIT_VIDEO_CODEC_H264,
    .width = 1280,
    .height = 960,
    .fps = 10
}
```

### main/example.c (Room Connection)
```c
livekit_room_connect(room_handle,
    "wss://lab7-ifvw37bc.livekit.cloud",
    "<your-jwt-token>");
```

---

## Build & Deployment

### Build Command
```bash
ESP-IDF: Build (or Full Clean Build if config changed)
```

### Flash Command
```bash
ESP-IDF: Flash
```

### Monitor Command
```bash
ESP-IDF: Monitor
```

### Expected Boot Sequence
1. WiFi connects to "iPhone (59)"
2. NTP syncs time via Google/pool.ntp.org
3. Logs: `I (12499) livekit_example: Room state changed: Connected`
4. Audio encoder initializes
5. Video encoder initializes (1280×960 @ 10fps H.264)
6. ESP joins LiveKit room as participant `esp32-cam`
7. Camera frames stream to LiveKit Cloud at ~10fps

---

## Verification in LiveKit Cloud Dashboard

1. Navigate to: https://cloud.livekit.io → lab7 project
2. Click Sessions
3. Should see room named `birdfeeder` with participant `esp32-cam`
4. Click participant to preview live video stream
5. Bitrate meter should show steady video bitrate (~500-800kbps at 1280×960 10fps H.264)

---

## Key Learnings

| Topic | Learning |
|-------|----------|
| **Component Versioning** | Local override paths can use flexible version constraints (>=X.Y.Z) |
| **LiveKit SDK** | Official ESP32 SDK is cleaner than WHIP + esp_peer for embedded cases |
| **Hardcoded Credentials** | External utility components sometimes expect `#define` values over Kconfig |
| **Camera Resolution** | Native resolution often outperforms software resizing; reduce FPS instead |
| **Video Encoding** | 1280×960 H.264 requires careful FPS tuning for embedded hardware (10fps is sweet spot) |
| **Capture Pipeline** | Auto-starts on init; explicit `media_start()` not always needed but can be explicit |

---

## AWS Resources (Still Needed - Phase 1)

Not yet created:
- S3 bucket for recordings
- IoT Core thing + certificates
- IoT endpoint configuration
- IAM credentials for LiveKit egress

These are needed for Phase 6 (recording integration) but not required for basic streaming.

---

## Next Steps (Recommended)

1. ✅ **Streaming working** — Verify 10fps is smooth in LiveKit dashboard
2. ⏳ **Audio two-way** — Test speaker output (if needed for two-way audio)
3. ⏳ **Runtime config** — Move hardcoded JWT to NVS/config file (not committed to git)
4. ⏳ **AWS integration** — Set up S3 bucket + IoT thing for Phase 6
5. ⏳ **Recordings** — Enable LiveKit egress to S3

---

## Appendix: File Changes Summary

| File | Change | Reason |
|------|--------|--------|
| `main/idf_component.yml` | version: ">=0.3.7" | Allow flexible local override |
| `sdkconfig.defaults` | Camera: OV5647; Codec: P4_EYE | Hardware config |
| `main/Kconfig.projbuild` | default "ESP32_P4_EYE" | Codec board selection |
| `main/main.c` | Added WIFI_SSID/PASSWORD #define | External util component compatibility |
| `main/example.c` | width/height: 1280×960, fps: 10 | Encoder tuning for stability |
| `main/example.c` | Hardcoded JWT + server URL | Room connection credentials |

---

**Status**: ✅ **Video streaming functional at 1280×960 10fps H.264 + Opus audio**
