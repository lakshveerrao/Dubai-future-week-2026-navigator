# Future Week Navigator: Technical Reference

*Architecture, hardware, firmware, positioning, data formats and deployment, for builders*

## 0. Status and Scope

Version 1 runs on the CGOTCHI (ESP32-S3) as a MicroPixel guest app with the real Dubai Future Week venue map. It provides touch-driven destination selection, a Mode 7 tilted 3D map with route overlay, accelerometer-based pedestrian dead reckoning (PDR) constrained to the route, home-zone resets on arrival and a spoken arrival cue. It has been pitched to the Dubai Future Week organisers.

This document describes the v1 architecture and the engineering plan for v2, which adds radio-based absolute positioning (Google Geolocation API, on-device Wi-Fi fingerprinting, Wi-Fi FTM, BLE), heading sensing and docking-station fleet management.

### 0.1 Second implementation: Waveshare ESP32-S3-Touch-AMOLED-1.8

A second working build runs on the Waveshare ESP32-S3-Touch-AMOLED-1.8, built and flashed with Claude Code. It delivers the same visitor flow on a 1.8" AMOLED display and adds Google Geolocation API fixes over the venue Wi-Fi, combined with step counting between fixes. Unlike the CGOTCHI build, it is native firmware rather than a sandboxed MicroPixel guest, so it has direct access to the Wi-Fi radio and HTTPS. The Google Geolocation request format, coordinate conversion and fusion described in sections 9.2 and 9.6 apply to this build today rather than as future work; section 19 documents the build as tested in full.

| Subsystem | CGOTCHI build | Waveshare build |
| --- | --- | --- |
| SoC / memory | ESP32-S3R8, 16 MB flash, 8 MB PSRAM | ESP32-S3R8, 16 MB flash, 8 MB PSRAM |
| Display | 2.01" IPS 296x240, JD9853 SPI | 1.8" AMOLED 368x448, QSPI (SH8601 on V1, CO5300 on V2) |
| Touch | CST810 | FT3168 on V1, CST820 on V2 |
| Motion | Accelerometer only (SC7A20 / LIS2DH12) | QMI8658 6-axis (accelerometer + gyroscope) |
| Audio | ES8311 + NS4150B, 3 W speaker, 2 mics | ES8311, onboard speaker and microphone |
| Power | 600 mAh, PFC161 latch | AXP2101 PMU, 350 to 400 mAh cell on MX1.25 |
| Extras | BQ27220 fuel gauge | PCF85063 RTC, TCA9554 I/O expander, TF card slot |
| Software | MicroPixel guest app (C++ to WASM AOT) | Native ESP32-S3 firmware |
| Positioning | Route-constrained PDR + home-zone resets, offline | Google Geolocation via venue Wi-Fi + step counting |
| Network at runtime | None | Venue Wi-Fi |

The Waveshare build's API key should move from firmware to a proxy service before wider deployment (section 9.2), and its gyroscope is available for relative heading in a later release.

## 1. System Architecture

```
+--------------------------- CGOTCHI (ESP32-S3) ---------------------------+
|  MicroPixel host firmware (ESP-IDF 6.1)                                   |
|   drivers: JD9853 LCD (SPI2) | CST810 touch (I2C) | ES8311+NS4150B (I2S)  |
|            SC7A20/LIS2DH12 accel (I2C) | buttons | Wi-Fi/BLE radio        |
|   WAMR AOT runtime  <-- host API (display, input, audio, sensors) -->     |
|  +---------------------- Guest app (C++ -> WASM AOT) -------------------+ |
|  |  UI state machine  | Router (A*) | Positioning (PDR + resets)        | |
|  |  Mode 7 renderer   | Instruction generator | Audio cues              | |
|  |  Venue bundle (graph, POIs, polygons, texture, voice clips)          | |
|  +----------------------------------------------------------------------+ |
+---------------------------------------------------------------------------+
        | USB-C (app install, venue bundle, charging)
+-------v-------------+        v2: HTTPS (Wi-Fi)       +----------------------+
| Dock / fleet host   |  ---------------------------->  | Location proxy       |
| (USB hub + SBC,     |                                 | (holds Google API    |
|  micropixel CLI)    |                                 |  key, rate limiting) |
+---------------------+                                 +----------------------+
```

v1 is fully offline. The device never needs network access at runtime. v2 radio positioning is additive and degrades gracefully back to v1 behaviour when no radio fix is available.

## 2. Hardware Platform

### 2.1 Bill of materials (CGOTCHI, board revision OSTB\_XIAOZHI\_V1.2)

| Subsystem | Part | Bus / address | Notes |
| --- | --- | --- | --- |
| SoC | ESP32-S3R8 | n/a | Dual-core Xtensa LX7 up to 240 MHz, 2.4 GHz Wi-Fi 4, BLE 5, native USB |
| Memory | 16 MB flash, 8 MB octal PSRAM | n/a | PSRAM holds the map texture and route buffers |
| Display | JD9853, 2.01" IPS, 240x296 portrait | SPI2 @ 40 MHz | Presented by MicroPixel as 296x240 landscape |
| Touch | CST810 | I2C 0x15 | Polled, no interrupt line; shares reset with LCD |
| Audio out | ES8311 codec + NS4150B class-D amp | I2C 0x18 + I2S0 | 3 W speaker |
| Audio in | ES7210 ADC, 2 mics | I2C 0x40 | Not yet supported by MicroPixel |
| Motion | SC7A20 / LIS2DH12-class accelerometer | I2C 0x19 | Mounted on the rear, axes rotated in firmware; no gyroscope, no magnetometer |
| Fuel gauge | BQ27220 | I2C 0x55 | Battery level not yet exposed by MicroPixel |
| Power | 600 mAh Li-ion, PFC161 power controller | GPIO2 latch | Roughly 3 to 4 h runtime, USB-C charging |
| Enclosure | 48 x 41 x 26 mm cube | n/a | Three top buttons |

### 2.2 Pin map

From the official board profile (`firmware/espressif/main/platform/boards/cheeko-gotchi/board_config.hpp`).

| Signal | GPIO |
| --- | --- |
| Power latch (drive low on battery) | 2 |
| I2C SCL / SDA | 11 / 12 |
| LCD SCLK / MOSI / CS / DC | 9 / 10 / 14 / 8 |
| LCD reset (also CST810 reset, active low) | 17 |
| Backlight (active high) | 13 |
| I2S MCLK / BCLK / WS / DOUT | 5 / 15 / 16 / 6 |
| NS4150B amplifier enable (active high) | 4 |
| BOOT / VOL- / VOL+ buttons | 0 / 39 / 40 |

In the MicroPixel input model the three buttons map to Confirm, Left and Right.

### 2.3 Hardware constraints that shape the design

The accelerometer gives step events and step energy but no orientation in the horizontal plane, so v1 cannot measure heading. Position is therefore tracked as one-dimensional progress along the active route rather than as a free 2D position. Absolute heading in v2 requires an external magnetometer or IMU (section 8).

Runtime of 3 to 4 h is shorter than an event day. Fleet rotation through docks is part of the system design, not an accessory.

## 3. Firmware and Runtime

### 3.1 MicroPixel split

The device runs MicroPixel host firmware built on ESP-IDF 6.1. Applications are C++ guest programs compiled with the WASI SDK to WebAssembly and then ahead-of-time compiled with the MicroPixel fork of `wamrc` (AOT v6). The host owns all drivers and exposes a constrained API to the guest. Installing an app over USB replaces only the guest bundle; the host firmware is untouched.

Guest SDK headers used by the navigator, all under `guest/sdk/`:

| Header | Used for |
| --- | --- |
| `display.hpp`, `graphics.hpp`, `ui/` | Frame composition, text, tiles, overlays |
| `input.hpp` | Touch events and the three buttons |
| `audio.hpp` | `AudioClip` / `Playback` for Ogg Opus clips, `Tone` and `ToneSequencer` fallback |
| `sensors.hpp` | `Accelerometer` (`Sensor<Acceleration>`), sample interval control |
| `mode7_plane.hpp` | `Mode7Plane` / `Mode7PlaneConfig` for the perspective ground plane |
| `storage.hpp`, `resources.hpp` | Venue bundle and assets |
| `timer.hpp`, `clock.hpp` | Frame and sensor scheduling |

### 3.2 Toolchain

```sh
# one-time
export PATH="$SDK_ROOT:$PATH"
export WASI_SDK_PATH=/path/to/wasi-sdk
export WAMRC=/abs/path/to/micropixel-wamrc
python3 -m pip install "pyserial>=3.5,<4"

# per project
micropixel --transport usb device status
micropixel init future-week-nav --app-id com.lakshveer.futureweeknav --title "Future Week Navigator"
cd future-week-nav
micropixel --transport usb run            # build, install, start, follow logs
micropixel --transport usb run --no-follow
```

Windows uses the managed installation described in `guest/sdk/AI.md`. With several devices attached, pass `--port <port>` before `run`.

### 3.3 Host firmware build (only needed for v2 host extensions)

```sh
git submodule update --init --recursive
source /path/to/esp-idf-v6.1/export.sh
python3 -m pip install -r requirements-dev.txt
bash tools/s3.sh build-host gotchi
python3 tools/fetch_release_app_store.py
bash tools/s3.sh flash-host gotchi <port>
bash tools/s3.sh flash-apps gotchi <port>
```

The ESP-IDF commit is pinned in `tools/ci/firmware-sources.json`, and the build path must not contain spaces. If the board does not enumerate, hold BOOT while powering on to enter the ROM download mode.

### 3.4 Sandbox boundary

The guest API exposes no Wi-Fi scan, socket or HTTP primitives. Every v2 feature that needs the radio (scanning, FTM, BLE, network calls) must be implemented in the host firmware and surfaced to the guest through a new host API, or run on a companion MCU. Section 9 assumes a host extension that returns scan results and position fixes to the guest.

## 4. Application Architecture

### 4.1 State machine

```
          tap tile / Confirm                     progress >= L - r_arrive
 SELECT ------------------------> NAVIGATE ------------------------------> ARRIVED
   ^                                 |  \   "I arrived" (snap s = L)         |
   |        X / cancel               |   \------------------------------->   |
   +---------------------------------+                                       |
   +-------------------------- "Pick next place" ----------------------------+
 DOCKED (USB power detected) -> reset home zone to Entrance -> SELECT
```

Entering ARRIVED sets the home zone to the destination node, clears the PDR accumulator, plays the arrival clip and recomputes distances on the SELECT grid from the new origin.

### 4.2 Modules

| Module | Responsibility |
| --- | --- |
| VenueStore | Loads the venue bundle, exposes nodes, edges, POIs, polygons and the pre-rendered floor texture |
| Router | A\* over the navigation graph, returns the node path and a polyline in venue metres |
| Navigator | Holds the active route, progress `s`, remaining distance, ETA and the next manoeuvre |
| StepDetector | Turns raw acceleration into step events and per-step length estimates |
| Positioning | v1: route-constrained PDR. v2: fuses radio fixes (section 9.6) |
| Renderer | Mode 7 ground plane, route overlay, puck, destination label, HUD card |
| AudioCues | Arrival clip, turn chime, tone fallback |
| UiController | Tile grid, touch hit-testing, button focus, screen transitions |

### 4.3 Frame loop

Rendering runs at the display refresh budget while sensors are sampled on their own interval. Per frame: drain input events, drain accelerometer samples into StepDetector, update Navigator, compute the camera pose from the puck position, render, then emit audio cues raised during the update. Heavy work such as A\* runs once per destination change, never per frame.

## 5. Venue Data Model

### 5.1 Coordinate frame

All geometry is stored in a local Cartesian frame in metres. The origin is the main entrance, +x points along the venue's primary axis and +y is 90 degrees counter-clockwise from it. A georeference block allows conversion to and from WGS84, which v2 needs to consume Google Geolocation fixes.

### 5.2 Bundle schema (`venue.json`)

```json
{
  "version": "2026.11.1",
  "frame": { "units": "m", "origin_node": "entrance" },
  "georef": { "lat0": 0.0, "lng0": 0.0, "rotation_deg": 0.0, "scale": 1.0 },
  "floors": [ { "id": "G", "texture": "floor_G.png", "px_per_m": 4 } ],
  "nodes": [
    { "id": "entrance", "x": 0, "y": 0, "floor": "G" },
    { "id": "n12", "x": 34.5, "y": 0, "floor": "G" }
  ],
  "edges": [
    { "a": "entrance", "b": "n12", "len": 34.5, "cost": 1.0, "accessible": true }
  ],
  "pois": [
    { "id": "main_stage", "name": "Main Stage", "node": "n40", "category": "stage", "arrive_radius": 4 }
  ],
  "polygons": [
    { "id": "hall_a", "label": "Hall A", "height": 3, "pts": [[10,20],[40,20],[40,50],[10,50]] }
  ],
  "audio": { "arrived": "arrived.ogg" }
}
```

Nodes sit on walkable corridor centrelines and at every decision point. Edge `cost` is a multiplier on length that lets crowded or slow corridors be penalised without changing geometry. POIs attach to the nearest graph node and carry their own arrival radius.

### 5.3 Authoring pipeline

1. Obtain the official floor plan as vector (PDF, SVG or CAD export) or a high-resolution raster.
2. Scale it to metres using at least two surveyed reference distances, and record the residual error.
3. Trace walkable corridors and place graph nodes at junctions, doors and POI frontages; trace hall footprints as polygons.
4. Georeference with at least three control points matched to satellite imagery or surveyed GPS outdoors; solve a similarity transform (translation, rotation, uniform scale) by least squares.
5. Rasterise the floor texture from the polygons at a fixed `px_per_m` in the dark theme palette.
6. Run validation: graph connectivity from the entrance to every POI, no zero-length edges, every POI node reachable, texture bounds covering all nodes.
7. Package `venue.json`, textures and audio into the app bundle and bump `version`.

## 6. Routing and Guidance

### 6.1 Path search

A\* over the node graph with Euclidean distance as the admissible heuristic and `len x cost` as the edge weight. Graph sizes for a single venue (hundreds of nodes) make this trivially fast on the S3. The resulting node list is converted into a polyline `P = [p0 ... pn]` with cumulative arc length `S_i`, total length `L`.

### 6.2 Manoeuvre generation

At each interior vertex the signed turn angle is computed from the incoming and outgoing segment bearings.

| Turn angle | Instruction |
| --- | --- |
| under 30 degrees | Continue (vertex suppressed) |
| 30 to 135 degrees | Turn left / right |
| over 135 degrees | Turn around |

The HUD shows the next non-suppressed manoeuvre and the distance to it, `S_k - s`. The prompt switches to the following manoeuvre once `s` passes `S_k`. ETA uses a walking speed of 1.2 m/s, overridable by the measured cadence times step length.

## 7. Positioning v1: Route-Constrained PDR

### 7.1 Step detection

The accelerometer is sampled at 50 Hz. The magnitude `|a| = sqrt(ax^2 + ay^2 + az^2)` in g is low-pass filtered (first-order IIR, cutoff about 3 Hz) to remove handling jitter, then a peak-valley detector emits a step when a peak above the upper threshold follows a valley below the lower threshold and the minimum interval since the last step has elapsed.

| Parameter | Starting value | Purpose |
| --- | --- | --- |
| Sample rate | 50 Hz | Nyquist margin for walking cadence (1.5 to 2.5 Hz) |
| LPF cutoff | 3 Hz | Removes hand shake |
| Peak threshold | 1.15 g | Heel-strike detection |
| Valley threshold | 0.90 g | Reject non-cyclic motion |
| Minimum step interval | 300 ms | Rejects double counting |
| Idle timeout | 2 s | Marks the user as stationary |

```cpp
struct StepDetector {
  float lp = 1.0f, alpha = 0.27f;          // alpha for ~3 Hz at 50 Hz
  float peak = 0, valley = 2;
  bool armed = false;
  uint64_t last_us = 0;

  // returns step length in metres, or 0 when no step
  float push(float ax, float ay, float az, uint64_t t_us) {
    float m = sqrtf(ax*ax + ay*ay + az*az);
    lp += alpha * (m - lp);
    if (lp < 0.90f) { armed = true; valley = fminf(valley, lp); }
    if (armed && lp > 1.15f) peak = fmaxf(peak, lp);
    if (armed && peak > 0 && lp < peak - 0.05f && t_us - last_us > 300000) {
      float len = K * powf(peak - valley, 0.25f);   // Weinberg model
      last_us = t_us; armed = false; peak = 0; valley = 2;
      return len;
    }
    return 0.0f;
  }
  float K = 0.48f;                            // calibrated per device
};
```

Step length uses the Weinberg model `L = K x (a_max - a_min)^(1/4)`, with `K` calibrated by walking a measured 50 m line and fitting to the observed step count. A fixed 0.7 m step is the fallback if calibration is skipped.

### 7.2 Route constraint

Without heading, PDR integrates distance only. The user is assumed to follow the active route, so position is the scalar arc length `s` along `P`, advanced by each step length and clamped to `[0, L]`. The rendered position is `P(s)`, interpolated on the current segment, and the camera yaw follows the segment bearing. This removes lateral drift entirely and reduces error to along-track scale error, which grows linearly with distance walked.

### 7.3 Home-zone resets

On every arrival `s` is reset and the origin node becomes the destination node, so accumulated along-track error is bounded by the length of a single leg instead of the whole visit. The "I arrived" control forces `s = L`. Automatic arrival fires when `s >= L - arrive_radius`.

### 7.4 Error budget

With a 5 percent along-track scale error, a 120 m leg ends within about 6 m of the true destination, which is inside the arrival radius of large POIs but not of individual stalls. Calibrated `K` and shorter legs tighten this; v2 radio fixes remove it.

## 8. Heading (v2)

Absolute heading needs a magnetometer, ideally fused with a gyroscope. Candidate modules are a BMM150 or QMC5883L magnetometer, or a BNO085 9-axis IMU with on-chip fusion that outputs a rotation vector directly. Integration depends on whether the enclosure exposes the I2C bus (GPIO11/12); otherwise it requires a hardware revision.

Indoor magnetic fields are distorted by steel structure, escalators and display hardware. Mitigations are hard-iron and soft-iron calibration on the device, rejection of readings whose field magnitude deviates from the local reference, and fusing heading with the route bearing so that a reliable route bearing dominates while the user is on-route. With heading available the positioning state becomes 2D and can be map-matched to corridors (section 9.6).

## 9. Positioning v2: Radio Fixes

### 9.1 Options compared

| Technique | Typical indoor accuracy | Infrastructure | Network at runtime | Cost driver |
| --- | --- | --- | --- | --- |
| Google Geolocation API (Wi-Fi) | Tens of metres, venue-dependent | None (uses existing APs) | Yes | Per-request billing, connectivity |
| On-device Wi-Fi fingerprinting | 3 to 5 m | Existing APs plus a site survey | No | Survey labour, re-survey on AP changes |
| Wi-Fi FTM (802.11mc RTT) | 1 to 2 m line-of-sight | FTM responders (ESP32 anchors) | No | Anchor hardware and power |
| BLE beacon RSSI | 2 to 5 m | Beacons every 5 to 10 m | No | Beacon hardware, batteries |
| UWB | Under 0.5 m | UWB anchors plus a UWB tag | No | Needs extra radio hardware on the device |

### 9.2 Google Geolocation API

The Geolocation API returns a WGS84 position and an accuracy radius from a list of observed Wi-Fi access points (and optionally cell towers).

```http
POST https://www.googleapis.com/geolocation/v1/geolocate?key=API_KEY
Content-Type: application/json

{
  "considerIp": false,
  "wifiAccessPoints": [
    { "macAddress": "3c:37:86:5d:75:d4", "signalStrength": -43, "channel": 11 },
    { "macAddress": "30:86:2d:c4:29:d0", "signalStrength": -58, "channel": 6 },
    { "macAddress": "a4:2b:b0:11:9e:02", "signalStrength": -71, "channel": 1 }
  ]
}
```

```json
{ "location": { "lat": 25.0000, "lng": 55.0000 }, "accuracy": 35.0 }
```

At least two access points are required. `considerIp` must be `false`, otherwise a failed Wi-Fi match silently falls back to an IP-based city-level position. Access points whose SSID ends in `_nomap` are opted out of Google's database and should be filtered out. A `404 notFound` means no match; the client must then fall back to PDR. Usage is billed per request against a Google Cloud billing account.

On the device this requires the host firmware to scan and to make an HTTPS call:

```c
wifi_scan_config_t cfg = {
  .show_hidden = true,
  .scan_type = WIFI_SCAN_TYPE_ACTIVE,
  .scan_time.active = { .min = 60, .max = 120 },
};
ESP_ERROR_CHECK(esp_wifi_scan_start(&cfg, true));   // blocking scan
uint16_t n = 32;
wifi_ap_record_t recs[32];
ESP_ERROR_CHECK(esp_wifi_scan_get_ap_records(&n, recs));
// recs[i].bssid, recs[i].rssi, recs[i].primary -> JSON body
// POST with esp_http_client to the location proxy (not to Google directly)
```

The API key must never be embedded in device firmware. The device posts scan results to a small proxy service that holds a restricted key, enforces per-device rate limits and caches recent results. The device also needs an internet route at runtime, either venue Wi-Fi without a captive portal or a dedicated SSID provided by the organisers.

The fix is converted into the venue frame through the georeference. For a venue a few hundred metres across, an equirectangular approximation is sufficient:

```
dx = (lng - lng0) * cos(lat0) * 111320
dy = (lat - lat0) * 110540
[x, y] = R(-rotation) * [dx, dy] / scale
```

Google's indoor accuracy depends on how well its crowd-sourced database covers the venue's access points, and temporary event APs are usually unknown to it. In practice the API is useful as a coarse cold-start fix (which hall or wing) and for sanity-checking PDR, not for stall-level guidance. Always use the returned `accuracy` as the measurement variance in fusion.

### 9.3 On-device Wi-Fi fingerprinting

An offline radio map is built by surveying reference points on a grid of about 3 to 5 m along all corridors. At each point, 10 to 20 scans are recorded while rotating the device, and the mean and standard deviation of RSSI per BSSID are stored.

```json
{
  "ap_index": ["3c:37:86:5d:75:d4", "30:86:2d:c4:29:d0"],
  "refs": [
    { "x": 12.0, "y": 4.0, "mean": [-48, -71], "std": [3.1, 4.4], "n": 16 }
  ]
}
```

Online, the device scans, keeps BSSIDs present in `ap_index`, substitutes -100 dBm for missing entries, and ranks reference points by Euclidean distance in signal space. Position is the inverse-distance-weighted mean of the k nearest references (k = 3 to 5). Excluding locally administered (randomised) MACs removes phone hotspots that move with visitors. Survey on a setup day under realistic load where possible, because dense crowds attenuate 2.4 GHz signals by several dB. A store of 300 reference points over 60 APs is well under 100 KB.

A full active scan of all 2.4 GHz channels takes on the order of a second and is the dominant radio energy cost, so scans run every 3 to 5 s while navigating, and only when a step has been detected since the last scan.

### 9.4 Wi-Fi FTM (802.11mc RTT)

The ESP32-S3 supports FTM in ESP-IDF; the station side acts as initiator and a SoftAP as responder. Distance is `RTT x c / 2`, averaged over a burst of frames, with a per-device zero-distance offset calibrated once. Venue access points rarely support FTM responder mode, so this approach means deploying ESP32 anchors as responders at surveyed positions. With three or more anchor ranges, position is solved by linearised least squares. Espressif recommends line-of-sight, RSSI of -70 dBm or better and 20 MHz bandwidth for usable ranges. Note that the station must disconnect from or coordinate with any AP it is associated with while ranging.

### 9.5 BLE beacons

The ESP32-S3 can passively scan iBeacon or Eddystone advertisements. Range is estimated with a log-distance path-loss model, `d = 10^((TxPower_1m - RSSI) / (10 n))` with `n` around 2 to 3 indoors, after smoothing RSSI with a median filter. Beacons are simple to deploy along corridors and at POI entrances, and the nearest-beacon rule alone gives reliable zone-level arrival detection.

### 9.6 Fusion

In v1 the state is one-dimensional, so fusion is a scalar Kalman filter on `s`. Each step is a prediction (`s += L_step`, process variance proportional to step length), and each radio fix projected onto the route polyline is a measurement with variance equal to the fix's accuracy squared. Fixes whose perpendicular distance from the route exceeds a gate (for example 15 m) indicate the user has left the route and trigger re-routing from the nearest graph node.

Once heading is available, the state becomes 2D and a particle filter (a few hundred particles) is the better fit. Particles propagate by step length and heading with noise, particles that cross walls in the polygon layer are discarded, and radio fixes reweight them. This map-matching gives corridor-level accuracy even with weak radio fixes.

## 10. Rendering

The floor is drawn with `Mode7Plane`: a pre-rendered top-down texture of the venue projected onto a tilted ground plane, rendered row by row from farthest to nearest. The camera is placed behind and above the puck, looking along the current segment bearing, so the route always points up-screen. Hall blocks are drawn as extruded labels on top, the route polyline is projected per vertex and drawn with a glow stroke, and the puck is fixed at bottom centre. The HUD card, destination pill and "I arrived" control are drawn in screen space.

The texture lives in PSRAM. At 4 px/m a 300 m x 200 m venue is 1200 x 800 px, about 1.9 MB at 16 bpp, which fits; larger venues are tiled per floor or wing. The camera pose is smoothed with a critically damped spring so turns read as a pan rather than a snap.

### 10.1 Visual system

| Token | Value |
| --- | --- |
| Background | #0B0E13 |
| Surface | #131820 |
| Primary text | #F4F7FA |
| Secondary text | #9AA4B2 |
| Accent | #5EF2D6 (alternatives #7AA8FF, #FFB547) |
| Display font | Space Grotesk 700 |
| UI font | Inter 400 to 600 |
| Minimum touch target | 44 px |

## 11. Audio

The arrival cue is a short mono clip produced with an offline TTS engine and encoded as Ogg Opus:

```sh
ffmpeg -i arrived.wav -ac 1 -ar 48000 -c:a libopus -b:a 24k arrived.ogg
```

It is loaded once as an `AudioClip` and played as a new `Playback` on each arrival. If clip playback is unavailable, a short `ToneSequencer` arpeggio is played instead. Per-destination announcements ("You've arrived at Main Stage") can be shipped as one clip per POI, at roughly 3 to 5 KB per second at 24 kbps.

## 12. Input

Touch is the primary input, with the CST810 polled by the host. The SELECT grid is a 2x2 page of 44 px-plus tiles with swipe or Left/Right paging. Buttons provide a full non-touch path: Left and Right move focus, Confirm activates. During NAVIGATE, Confirm is bound to "I arrived" and a long press on Confirm cancels the route.

## 13. Power

The backlight and the radio dominate consumption. v1 keeps the radio off entirely, which is why it reaches the 3 to 4 h envelope. In v2 the scan interval is the main control: scanning only while moving, backing off to 10 s or more when stationary, and dimming the backlight after 20 s of no movement extend runtime. Battery percentage needs the BQ27220 exposed through a host API before the UI or the dock can show it.

## 14. Docking and Fleet Management

Each dock bay supplies USB-C power, and data runs through a powered USB hub to a single-board computer running the `micropixel` CLI. When a device is inserted the host identifies it by serial number, checks the installed app and venue bundle versions, pushes updates if needed, signals the app to reset its home zone to the entrance, and lights the bay LED green when charged and current, red while charging or updating. Because MicroPixel does not yet expose battery level, charge state is inferred from charge time or read once the gauge is exposed.

A fleet table (device ID, bundle version, last dock, check-out and check-in time, dock location) supports operations reporting and loss tracking. Returns can be enforced with a refundable deposit, an ID or ticket scan at check-out, or open trust, and the check-out record is the hook for whichever policy the organisers choose.

## 15. Testing and Validation

| Test | Method | Target |
| --- | --- | --- |
| Step count | 200 steps on a counted course, five walkers | Error under 3 percent |
| Step length calibration | 50 m measured line, fit K | Along-track error under 5 percent |
| Route coverage | Every POI from every POI, automated A\* | 100 percent reachable |
| Guidance | Walk every leg on site, record arrival error | Within arrival radius |
| Radio fix accuracy (v2) | Surveyed checkpoints, log fixes | Report CEP50 and CEP95 |
| Battery | Continuous navigation loop at full brightness | At least 3 h |
| Crowd conditions | Repeat radio tests during peak hours | Degradation documented |

## 16. Known Limitations and Risks

The device cannot sense heading, so it assumes the user follows the route and cannot detect a wrong turn until a radio fix or the user corrects it. Guest apps cannot use the radio without host firmware changes. Google Geolocation needs runtime internet, billing and good database coverage of the venue's access points, and event-specific APs are often unknown to it. Fingerprints go stale when the organisers move or replace access points, so a re-survey is part of each event's setup. Battery runtime is below a full event day, so dock rotation is required. MicroPixel does not yet expose microphones, battery level or the power key.

## 17. Roadmap

| Version | Scope |
| --- | --- |
| v1 (shipped) | Real venue map, touch UI, Mode 7 3D map, route-constrained PDR, home-zone resets, spoken arrival |
| v1.1 | Per-POI voice announcements, step-length calibration screen, accessible-route option |
| v2 | Host firmware extension for Wi-Fi scanning, fingerprinting with offline radio map, Kalman fusion, off-route detection and re-routing |
| v2.1 | Google Geolocation via proxy as cold-start and fallback fix |
| v3 | Magnetometer or 9-axis IMU, 2D particle filter with wall constraints, dock fleet automation with battery reporting |

## 18. Key Problems Faced During the Build

The problems below came up between the first idea and the working v1. Each row records what broke or blocked progress, what it would have cost, and how it was resolved or contained.

| # | Problem | Impact | Resolution |
| --- | --- | --- | --- |
| 1 | GPS is unusable inside exhibition halls | No absolute position source indoors | Dropped GPS entirely; position comes from accelerometer PDR, with radio fixes planned for v2 |
| 2 | Accelerometer dead reckoning drifts, and along-track error grows linearly with distance walked | Over a multi-stop visit the puck would end far from the user | Home-zone reset on every arrival bounds error to a single leg (section 7.3) |
| 3 | Without absolute position the device cannot know when the user has reached a POI | No reliable arrival trigger | Explicit "I arrived" control plus automatic arrival at s >= L - arrive\_radius |
| 4 | The CGOTCHI has no gyroscope or magnetometer | Free 2D PDR is impossible and the arrow cannot rotate with the user | Position modelled as 1D progress along the route, camera yaw taken from the segment bearing; magnetometer or 9-axis IMU planned for v3 |
| 5 | The retail listing gave no chip-level specs, and the Amazon page blocked automated access | Display, touch, audio and sensor parts and pins were unknown | Traced the product to the vendor site and the official MicroPixel repository; parts and pins taken from the board profile `board_config.hpp` |
| 6 | A community firmware for the same board never found the speaker amplifier enable line, leaving its speaker silent | The spoken arrival cue would produce no sound | The official board profile identifies the NS4150B enable on GPIO4, active high, driven by the MicroPixel host |
| 7 | The JD9853 panel is 240x296 portrait while the enclosure is used landscape | Risk of rotated rendering and mismatched touch coordinates | Designed against MicroPixel's 296x240 landscape presentation, which handles rotation and touch mapping in the host |
| 8 | The first UI concept was a tall, phone-shaped device with a compass arrow | All screens had to be reworked once the target became a square cube | Rebuilt every screen for a square display, switched the list to a 2x2 tile grid and replaced the compass with a Mode 7 3D map |
| 9 | The UI initially relied on an on-screen button row before touch capability was confirmed | Wasted screen area on a 2" display | Removed the on-screen row once the touchscreen was confirmed; the three physical buttons remain as a non-touch path |
| 10 | MicroPixel guest apps have no Wi-Fi scan, socket or HTTP API | Wi-Fi positioning and Google Geolocation cannot run inside the app | Shipped v1 fully offline; v2 radio work is scoped as a host firmware extension or a companion MCU (section 3.4) |
| 11 | Google Geolocation needs runtime internet, an API key and billing, and has weak coverage of temporary event access points | Unreliable as a primary indoor fix, and a key in firmware would leak | Treated as a coarse cold-start fix only, called through a proxy that holds a restricted key; fingerprinting chosen as the primary v2 method |
| 12 | Wi-Fi fingerprints are fragile: dense crowds attenuate 2.4 GHz, and organisers move or replace access points | Event-day fixes can disagree with the survey | Survey under load where possible, re-survey per event, keep PDR and "I arrived" as the fallback |
| 13 | The real floor plan has no metric scale, routing graph or georeference | It cannot be used directly for routing or for converting Google fixes | Authoring pipeline: scale from reference distances, trace a corridor graph, georeference with control points, rasterise the texture, validate connectivity (section 5.3) |
| 14 | Battery runtime is 3 to 4 h, shorter than an event day | Devices die mid-visit | Radio kept off in v1, dock rotation built into operations, backlight dimming when stationary |
| 15 | MicroPixel does not yet expose battery level, microphones or the power key | The dock cannot report charge and voice input is unavailable | Charge state inferred at the dock for now; battery reporting depends on a host API for the BQ27220 |
| 16 | Code could not be flashed from the cloud planning session, and a full-firmware flash risked bricking the device | Deployment needed a local, safe path | Guest app installed over USB from a laptop with `micropixel --transport usb run`, leaving host firmware untouched; Windows uses the managed SDK install |
| 17 | Shared devices can be lost or not returned | Fleet shrinkage during the event | Dock check-out and check-in records; deposit, ID scan or trust policy to be chosen with the organisers |

## 19. Version 2 As Built: Waveshare ESP32-S3-Touch-AMOLED-1.8 with Google Wi-Fi Geolocation

### 19.1 Status

Version 2 has been built, flashed and tested end to end on hardware. It runs the same user flow as Version 1 against the real Dubai Future Week map: pick a destination on the touchscreen, follow the tilted 3D map with its route line and arrow, and hear "You've arrived!" from the speaker. The difference is positioning. Version 1 relies only on route-constrained step counting. Version 2 joins the venue Wi-Fi, scans nearby access points, sends the scan to the Google Geolocation API over HTTPS, and merges the returned fixes with accelerometer step counting between fixes.

Version 2 was not part of the original plan. The plan placed Wi-Fi positioning as future work on the CGOTCHI; it was instead delivered on a different board after testing showed the CGOTCHI guest sandbox has no radio API (section 19.12).

No positioning accuracy, fix latency, battery runtime or API usage has been measured yet, and this section reports no such figures. This reference does not record whether the firmware was built with the Arduino core or with ESP-IDF, or which board revision (V1 or V2, see 19.2) was used. The code below is a reference implementation written against ESP32 Arduino core 3.x APIs, with ESP-IDF equivalents named beside it. It is not the as-built source, and its constants are starting values.

### 19.2 Hardware

Waveshare sells two revisions under one product name, differing only in the display and touch controllers. Waveshare's documentation says V1 is discontinued and shipments switch to V2 from 30 May 2026. The revisions can be told apart by the label on the back of the board, or at runtime by probing I2C address 0x15 (V2 touch) and 0x38 (V1 touch), which is how Waveshare's own `board_variant.c` does it.

| Part | Function | Bus / address | Notes |
| --- | --- | --- | --- |
| ESP32-S3R8 | MCU | n/a | Dual-core LX7 up to 240 MHz, 8 MB PSRAM in package, 16 MB external NOR flash |
| SH8601 (V1) | AMOLED controller | QSPI | 368 x 448; Arduino driver `Arduino_SH8601`, no column offset |
| CO5300 (V2) | AMOLED controller | QSPI | 368 x 448; Waveshare V2 sketches pass a column offset of 16 to `Arduino_CO5300` |
| FT3168 (V1) | Capacitive touch | I2C 0x38 | `Arduino_FT3x68` in Arduino\_DriveBus |
| CST820 (V2) | Capacitive touch | I2C 0x15 | Driven through `Arduino_CST816x` |
| QMI8658 | 6-axis IMU | I2C 0x6B | `QMI8658_L_SLAVE_ADDRESS` in SensorLib (0x6A alternate) |
| AXP2101 | PMU, charger, fuel gauge | I2C 0x34 | `AXP2101_SLAVE_ADDRESS` in XPowersLib |
| PCF85063A | RTC | I2C 0x51 | Backup-battery pad keeps time across battery swaps |
| ES8311 | Audio codec (DAC + mic ADC) | I2C 0x18 control, I2S data | 0x18 with CE low (`ES8311_ADDRESS_0`) |
| TCA9554 | 8-bit I/O expander | I2C 0x20 | LCD reset, touch reset, a power enable, SD CS, PWR key input (19.3) |
| Speaker amplifier | Drives the onboard speaker | GPIO46 enable | Amplifier part number not published |
| Microphone | Analog mic | ES8311 ADC | Not used by the navigator |
| microSD | Storage | SDMMC 1-bit | Usable for map tiles or audio clips |
| Battery | 3.7 V Li-ion/LiPo | MX1.25 2-pin | Wiki recommends a 400 mAh cell |
| BOOT key | Download mode / user key | GPIO0 | Active low |
| PWR key | Power on/off / user key | EXIO4 | Hold 6 s to power off |

All I2C devices (touch, IMU, PMU, RTC, codec, expander) share one bus, so the firmware needs a single bus owner or a mutex (19.4). The two Arduino library sets in Waveshare's repo differ: `examples/arduino-v2/libraries` has no XPowersLib, so a V2 build that reads the AXP2101 must take it from the original set or the Library Manager.

| Item | Value | Source |
| --- | --- | --- |
| Arduino core | esp32 by Espressif; CI uses 3.3.11, wiki says 3.0.6 or later | repo `docs/CI.md`, wiki |
| CI FQBN | `esp32:esp32:esp32s3:FlashSize=16M,PartitionScheme=app3M_fat9M_16MB` | `.github/workflows/examples.yml` |
| PSRAM setting | OPI PSRAM, inferred from the R8 (octal) die; CI does not set it | inference |
| GFX Library for Arduino | 1.6.4 bundled (wiki lists 1.4.9) | bundled `library.properties` |
| SensorLib | 0.3.3 bundled (wiki lists 0.2.1) | bundled |
| XPowersLib | 0.2.6 (original set only) | bundled |
| Arduino\_DriveBus | 1.0.1 | bundled |
| Adafruit\_XCA9554 + BusIO 1.17.4 | 1.0.0 (wiki lists ESP32\_IO\_Expander 0.0.3) | bundled, wiki |
| LVGL | 8.4.0 | bundled |
| ESP-IDF BSP | `waveshare/esp32_s3_touch_amoled_1_8` ^2.0.3, IDF 5.5 to before 6.1 | `docs/GETTING_STARTED.md` |
| ES8311 driver (Arduino) | `es8311.c/.h` copied into the sketch folder | `examples/arduino*/examples/15_ES8311/` |

### 19.3 Pin map

GPIO numbers come from `examples/arduino/libraries/Mylibrary/pin_config.h` and `examples/arduino-v2/libraries/Mylibrary/pin_config.h` in `waveshareteam/ESP32-S3-Touch-AMOLED-1.8` (the two files are identical). Expander bit assignments come from `examples/esp-idf/9x_*/components/board_variant/board_variant.c`, and button and SD notes from the Waveshare wiki. No board schematic has been published, so none of this has been checked against one.

| Signal | GPIO / pin | Macro | Notes |
| --- | --- | --- | --- |
| LCD QSPI CS | GPIO12 | `LCD_CS` |  |
| LCD QSPI SCK | GPIO11 | `LCD_SCLK` |  |
| LCD QSPI D0–D3 | GPIO4, 5, 6, 7 | `LCD_SDIO0..3` |  |
| LCD reset | TCA9554 EXIO0 | `IO_EXPANDER_LCD_RST` | No GPIO reset; sketches pass `GFX_NOT_DEFINED` |
| Power enable | TCA9554 EXIO1 | `IO_EXPANDER_DSI_PWR_EN` | Name carried over from the ESP32-P4 port; load undocumented |
| Touch reset | TCA9554 EXIO2 | `IO_EXPANDER_TOUCH_RST` |  |
| PWR key input | TCA9554 EXIO4 | n/a | Per wiki |
| SD CS | TCA9554 EXIO7 | `IO_EXPANDER_SD_CS` | Hold high in SDMMC mode |
| I2C SDA / SCL | GPIO15 / GPIO14 | `IIC_SDA`, `IIC_SCL` | 400 kHz in IDF examples |
| Touch INT | GPIO21 | `TP_INT` |  |
| I2S MCLK | GPIO16 | `I2S_MCK_IO` | MCLK = 256 x fs |
| I2S BCLK | GPIO9 | `I2S_BCK_IO` |  |
| I2S WS (LRCK) | GPIO45 | `I2S_WS_IO` | ESP32-S3 strapping pin |
| I2S DOUT (ESP32 to codec) | GPIO8 | `I2S_DO_IO` |  |
| I2S DIN (codec to ESP32) | GPIO10 | `I2S_DI_IO` |  |
| Speaker PA enable | GPIO46 | `PA` | Active high; ESP32-S3 strapping pin |
| SDMMC CLK / CMD / D0 | GPIO2 / GPIO1 / GPIO3 | `SDMMC_*` | Wiki lists the same pins as SPI SCK / MOSI / MISO |
| BOOT key | GPIO0 | n/a |  |

`pin_config.h` also defines `DOPIN 10` and `DIPIN 8`, the reverse of `I2S_DO_IO 8` and `I2S_DI_IO 10`. The working ES8311 sketch calls `i2s.setPins(BCK, WS, I2S_DO_IO, I2S_DI_IO, MCK)`, so ESP32 data out is GPIO8; the `DOPIN`/`DIPIN` pair appears to be named from the codec's side.

Before touching the display or touch controller, firmware must run the expander reset sequence: set EXIO0–2 as outputs, drive them low, wait 20 ms, drive them high, and only then call `begin()` on the touch and display drivers. The IDF `board_variant.c` writes the config register 0x03 and output register 0x01 directly, sets SD CS first, waits 20 ms and then releases all resets.

### 19.4 Firmware architecture

The firmware runs natively on the ESP32-S3 with full access to the Wi-Fi radio. Whether it uses Arduino `setup()/loop()` with FreeRTOS tasks or plain ESP-IDF tasks is not recorded; the decomposition below works in either.

```
                 GPIO21 TP_INT                       QMI8658 @0x6B
                      |                                   |
          +-----------v-----------+          +------------v-----------+
          | UI / touch            |          | IMU task  (50-100 Hz)  |
          | FT3168 0x38 | CST820  |          | |a| filter, peak/valley|
          | 0x15, dest picker,    |          | step detect, Weinberg  |
          | "I arrived" button    |          | step length L_k        |
          +-----------+-----------+          +------------+-----------+
                      | events                            | step(L_k, t)
          +-----------v-----------+   s, P   +------------v-----------+
          | App state machine     |<-------->| Fusion                 |
          | SELECT -> NAVIGATE -> |          | scalar Kalman on route |
          | ARRIVED -> home reset |          | arc length s, gating   |
          | A* on venue graph     |          +------------^-----------+
          +-----+-----------+-----+                       | fix(lat,lng,acc,t_scan)
                |           |                 +-----------+------------+
      +---------v---+ +-----v--------+        | Geolocation client     |
      | Renderer    | | Audio        |        | async scan -> filter   |
      | PSRAM fb,   | | ES8311 @0x18 |        | -> JSON -> HTTPS POST  |
      | tilted map, | | I2S, PA      |        | -> parse / backoff     |
      | QSPI DMA    | | GPIO46       |        +-----------^------------+
      +-------------+ +--------------+                    | link state
                                              +-----------+------------+
   Shared I2C bus GPIO15/14 (touch, IMU,      | Wi-Fi manager          |
   PMU, RTC, codec, TCA9554) behind one mutex | join, captive probe,   |
                                              | reconnect, NTP -> RTC  |
                                              +------------------------+
```

| Task | Suggested core / priority | Period | Notes |
| --- | --- | --- | --- |
| IMU sampler + step detector | core 1, high | 50–100 Hz | Same algorithm as Version 1, reads QMI8658 over the shared bus |
| Touch / UI | core 1, medium | INT-driven | INT line reduces contention on the I2C mutex |
| Renderer | core 1, medium | 10–30 fps target, dirty-driven | Never blocks on the network |
| Fusion + state machine | core 1, medium | on step / on fix | Pure computation, owns `s` and `P` |
| Wi-Fi manager | core 0, low | event-driven | Arduino and IDF run the Wi-Fi stack on core 0 by default |
| Geolocation client | core 0, low | 10–20 s while walking | Blocking HTTPS stays on this task only |
| Audio | core 0 or 1, low | on arrival | Short DMA-driven I2S write |

The network path is advisory. If the Wi-Fi link, a captive portal, the TLS handshake or the API fails, the device carries on exactly like Version 1: step counting along the route, home-zone reset on arrival and the "I arrived" button.

### 19.5 Network bring-up

The device joins the venue SSID in station mode using credentials stored in NVS, not compiled into the image. Event guest Wi-Fi often sits behind a captive portal that an ESP32 cannot click through; while it is in place, HTTPS requests to googleapis.com fail or are redirected. After getting an IP address the firmware probes `http://connectivitycheck.gstatic.com/generate_204` and treats any response other than 204 as associated without internet. The three practical fixes all need venue IT: a dedicated PSK SSID for the devices, MAC allow-listing on the portal, or WPA2-Enterprise (`esp_eap_client` in ESP-IDF). The access point scan itself does not need association; only the HTTPS request does.

```cpp
// Reference implementation, ESP32 Arduino core 3.x. ESP-IDF: esp_wifi_set_config/esp_wifi_connect,
// esp_event_handler_register(WIFI_EVENT/IP_EVENT), esp_netif_sntp_init().
#include <WiFi.h>
#include <HTTPClient.h>
#include <time.h>

enum NetState { NET_DOWN, NET_ASSOC_NO_INET, NET_UP };
volatile NetState netState = NET_DOWN;

static void onWifiEvent(arduino_event_id_t e, arduino_event_info_t) {
  if (e == ARDUINO_EVENT_WIFI_STA_GOT_IP)       netState = NET_ASSOC_NO_INET;  // verify below
  if (e == ARDUINO_EVENT_WIFI_STA_DISCONNECTED) netState = NET_DOWN;           // auto-reconnect runs
}

void netBegin(const char* ssid, const char* psk) {
  WiFi.mode(WIFI_STA);
  WiFi.setAutoReconnect(true);
  WiFi.onEvent(onWifiEvent);
  WiFi.begin(ssid, psk);
  configTzTime("<+04>-4", "time.google.com", "pool.ntp.org");   // Dubai, UTC+4, no DST
}

bool probeInternet() {                       // captive-portal check
  HTTPClient h;
  h.setTimeout(4000);
  if (!h.begin("http://connectivitycheck.gstatic.com/generate_204")) return false;
  int code = h.GET();
  h.end();
  return code == 204;
}
```

Reconnects use `setAutoReconnect(true)` on Arduino or `esp_wifi_connect()` from the disconnect handler on ESP-IDF. A watchdog on the manager task should run `WiFi.disconnect(); WiFi.begin()` with exponential backoff from about 2 s to about 60 s if no IP arrives within 30 s, and re-run the probe after every reconnect.

For TLS, validate the googleapis.com chain against Google Trust Services roots rather than pinning a leaf or intermediate certificate, because Google rotates them. On Arduino use `WiFiClientSecure::setCACert()` with the roots from `https://pki.goog/roots.pem`, or the core's `setCACertBundle()` (its signature has changed between core releases). On ESP-IDF set `.crt_bundle_attach = esp_crt_bundle_attach` in `esp_http_client_config_t`. Sync NTP once per boot, write the time to the PCF85063, and seed the system clock from the RTC at the next boot so that logs, scan `age` and any date-checked TLS are correct before the network comes up.

### 19.6 Geolocation pipeline

The geolocation client runs a non-blocking active scan, filters the results, POSTs JSON to `https://www.googleapis.com/geolocation/v1/geolocate?key=API_KEY`, and parses `location.lat`, `location.lng` and `accuracy` (metres).

| Request field | Value sent | Notes |
| --- | --- | --- |
| `considerIp` | `false` | Defaults to `true`; without `false` a weak scan falls back to IP geolocation, far too coarse for a venue |
| `wifiAccessPoints[]` | 2–15 entries | Must contain two or more physically distinct, stationary access points |
| `.macAddress` | BSSID `"AA:BB:CC:DD:EE:FF"` | Required; only universally administered MACs can be located |
| `.signalStrength` | RSSI in dBm | Optional |
| `.channel` | primary channel | Optional |
| `.age` | ms since scan completed | Optional |
| `.signalToNoiseRatio` | omitted | The ESP32 scan record does not report SNR |

Filtering runs in four steps. Access points whose SSID ends in `_nomap` (Google's owner opt-out) are dropped. BSSIDs with the locally administered bit set (`b[0] & 0x02`), covering randomised, virtual and Wi-Fi Direct BSSIDs, and multicast BSSIDs (`b[0] & 0x01`) are dropped. Anything weaker than about −90 dBm is dropped. The rest are sorted by RSSI and the strongest 15 kept. Enterprise access points broadcast several SSIDs from BSSIDs that differ only in the low bits, so the reference code counts distinct radios by masking the low nibble of the last octet (a heuristic) and does not send a request with fewer than two.

| Scan parameter | Arduino `WiFi.scanNetworks()` | ESP-IDF `wifi_scan_config_t` | Reference value |
| --- | --- | --- | --- |
| Mode | `async=true` | `esp_wifi_scan_start(&cfg, false)` | non-blocking |
| Active / passive | `passive=false` | `scan_type = WIFI_SCAN_TYPE_ACTIVE` | active |
| Dwell per channel | `max_ms_per_chan` (default 300) | `scan_time.active.min/max` | 120 ms |
| Hidden SSIDs | `show_hidden=true` | `show_hidden = true` | include (BSSID still useful) |
| Results | `SSID(i)`, `BSSID(i)`, `BSSIDstr(i)`, `RSSI(i)`, `channel(i)` | `esp_wifi_scan_get_ap_records()` | n/a |

```cpp
// Reference implementation, ESP32 Arduino core 3.x + ArduinoJson 7.
// ESP-IDF equivalents: esp_wifi_scan_start / esp_wifi_scan_get_ap_records,
// esp_http_client (crt_bundle_attach) and cJSON.
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <vector>
#include <algorithm>

extern const char GOOGLE_ROOTS_PEM[];        // GTS roots from pki.goog/roots.pem

struct GeoFix { bool ok; int http; double lat, lng; float acc_m; uint32_t t_scan; };

static bool usableBssid(const uint8_t* b) { return b && !(b[0] & 0x02) && !(b[0] & 0x01); }

bool scanStart() {
  return WiFi.scanNetworks(/*async*/true, /*hidden*/true, /*passive*/false, /*ms/chan*/120)
         == WIFI_SCAN_RUNNING;
}

// Call when WiFi.scanComplete() >= 0. Returns number of APs placed in 'body'.
int buildRequest(String& body, uint32_t tScanDone) {
  int n = WiFi.scanComplete();
  struct Ap { int i; int rssi; };
  std::vector<Ap> v;
  for (int i = 0; i < n; ++i) {
    if (!usableBssid(WiFi.BSSID(i))) continue;
    if (WiFi.SSID(i).endsWith("_nomap")) continue;
    if (WiFi.RSSI(i) < -90) continue;
    v.push_back({i, WiFi.RSSI(i)});
  }
  std::sort(v.begin(), v.end(), [](const Ap& a, const Ap& b) { return a.rssi > b.rssi; });
  if (v.size() > 15) v.resize(15);

  std::vector<uint64_t> radios;              // heuristic distinct-radio count
  for (auto& a : v) {
    const uint8_t* b = WiFi.BSSID(a.i);
    uint64_t key = 0; for (int k = 0; k < 6; ++k) key = (key << 8) | b[k];
    key &= ~0x0FULL;
    if (std::find(radios.begin(), radios.end(), key) == radios.end()) radios.push_back(key);
  }
  if (radios.size() < 2) { WiFi.scanDelete(); return 0; }

  JsonDocument doc;
  doc["considerIp"] = false;
  JsonArray arr = doc["wifiAccessPoints"].to<JsonArray>();
  uint32_t age = millis() - tScanDone;
  for (auto& a : v) {
    JsonObject o = arr.add<JsonObject>();
    o["macAddress"]     = WiFi.BSSIDstr(a.i);
    o["signalStrength"] = a.rssi;
    o["channel"]        = WiFi.channel(a.i);
    o["age"]            = age;
  }
  WiFi.scanDelete();
  serializeJson(doc, body);
  return (int)v.size();
}

GeoFix geolocate(const String& body, const char* key, uint32_t tScanDone) {
  GeoFix f{false, 0, 0, 0, 0, tScanDone};
  WiFiClientSecure tls;
  tls.setCACert(GOOGLE_ROOTS_PEM);
  HTTPClient http;
  http.setTimeout(8000);
  String url = String("https://www.googleapis.com/geolocation/v1/geolocate?key=") + key;
  if (!http.begin(tls, url)) return f;
  http.addHeader("Content-Type", "application/json");
  f.http = http.POST(body);                   // <0 = transport/TLS error
  String resp = (f.http > 0) ? http.getString() : String();
  http.end();

  JsonDocument doc;
  if (resp.length() == 0 || deserializeJson(doc, resp)) return f;
  if (f.http == 200) {
    f.lat   = doc["location"]["lat"];
    f.lng   = doc["location"]["lng"];
    f.acc_m = doc["accuracy"];
    f.ok    = true;
  } else {
    const char* reason = doc["error"]["errors"][0]["reason"] | "unknown";
    log_w("geolocate %d %s", f.http, reason);
  }
  return f;
}
```

The response is `{"location":{"lat":..,"lng":..},"accuracy":..}`, where `accuracy` is the radius of a circle around the location in metres. Google does not state the confidence level of that circle, so treating it as 1σ in fusion (19.7) is an assumption.

| HTTP | `reason` (domain) | Meaning | Firmware action |
| --- | --- | --- | --- |
| 200 | n/a | Fix returned | Pass to fusion |
| 400 | `parseError` (global) | Body is not valid JSON | Firmware bug; log and do not retry the same body |
| 400 | `keyInvalid` (usageLimits) | Key invalid, or billing/API not enabled | Disable geolocation until reboot, show "Wi-Fi positioning off", run on step counting |
| 403 | `dailyLimitExceeded` (usageLimits) | Daily quota used up | Suspend until the quota resets |
| 403 | `userRateLimitExceeded` (usageLimits) | Console-configured limit hit | Exponential backoff, minimum 60 s |
| 404 | `notFound` (geolocation) | Valid request, no result | Normal at poorly mapped spots; keep step counting, lengthen the interval |
| < 0 | n/a | `HTTPClient` transport/TLS failure | Ask the Wi-Fi manager to re-probe, back off |
| 5xx | n/a | Not documented | Treat as transient, back off |

Google publishes no per-minute rate limit; limits come from the daily and per-user quotas set in the Cloud console. The reference policy scans only while the user is walking (a step in the last 5 s): immediately when a route starts, then every 10–20 s while moving, with a forced scan when the fusion variance `P` exceeds about (8 m)². The POST is skipped when the strongest five BSSIDs match the previous request and fewer than five steps have been taken since. An associated active scan at 120 ms per channel across 13 channels takes roughly 1.6 s plus home-channel dwell, during which link throughput drops.

### 19.7 Coordinate conversion and fusion with step counting

Each fix is converted into the venue map frame in two steps. An equirectangular projection about a venue anchor (φ₀, λ₀) gives `x_E = R cos φ₀ (λ − λ₀)` and `y_N = R (φ − φ₀)` with R = 6 371 008.8 m and angles in radians. An affine transform `[u v]ᵀ = A [x_E y_N]ᵀ + b` then maps into the metric frame used by the A\* graph and route polyline. A and b are fitted by least squares from at least three surveyed control points and checked at a fourth, because an error in A or b adds a fixed offset to every fix.

The fix is projected onto the active route polyline P₀ … P\_N with cumulative lengths S₀ = 0 … S\_N = L. For each segment the closest point, clamped to the segment ends, gives the arc length `s_fix = S_i + t |P_{i+1} − P_i|` and the perpendicular distance d⊥. Where a route doubles back, only segments within ±(30 m + 3√P) of the current estimate are searched.

| Event | Update |
| --- | --- |
| Step of length L\_k (Weinberg) | s ← min(s + L\_k, L); P ← P + q L\_k |
| Fix accepted | R = acc²; K = P / (P + R); s ← s + K (s\_fix − s\_at\_scan); P ← (1 − K) P |
| Arrival (s ≥ L − r\_arrive, or "I arrived") | Play clip; home zone = destination; new route starts at s = 0, P = P₀ |

The scan finishes a second or more before the response arrives while the user keeps walking, so the innovation is computed against `s_at_scan` and applied to the current s. A fix is rejected when d⊥ > 15 m, when |s\_fix − s\_at\_scan| > max(15 m, 3√(P + R)), or when acc > 50 m. Three consecutive d⊥ rejections raise an "Off route?" prompt that offers to re-plan with A\* from the projected fix; re-planning is not automatic, so a single bad fix cannot reroute a visitor.

```cpp
// Reference implementation (portable C++). Constants are starting values.
#include <cmath>
#include <vector>
#include <algorithm>

struct Pt { float u, v; };
struct Route { std::vector<Pt> p; std::vector<float> S; float L; };   // S[i] cumulative

struct Track {
  float s = 0, P = 1.0f;          // m, m^2
  float q = 0.05f;                // m^2 per metre walked (tune on site)
  float s_at_scan = 0;
  int   offRoute = 0;
};

static Pt toVenue(double lat, double lng, const double A[4], const double b[2],
                  double lat0, double lng0) {
  const double R = 6371008.8, d2r = M_PI / 180.0;
  double x = R * std::cos(lat0 * d2r) * (lng - lng0) * d2r;
  double y = R * (lat - lat0) * d2r;
  return { float(A[0]*x + A[1]*y + b[0]), float(A[2]*x + A[3]*y + b[1]) };
}

static bool project(const Route& r, Pt q, float sLo, float sHi, float& sOut, float& dOut) {
  bool found = false; dOut = 1e9f;
  for (size_t i = 0; i + 1 < r.p.size(); ++i) {
    if (r.S[i+1] < sLo || r.S[i] > sHi) continue;           // windowed search
    float ex = r.p[i+1].u - r.p[i].u, ey = r.p[i+1].v - r.p[i].v;
    float len2 = ex*ex + ey*ey; if (len2 < 1e-6f) continue;
    float t = ((q.u - r.p[i].u)*ex + (q.v - r.p[i].v)*ey) / len2;
    t = std::clamp(t, 0.0f, 1.0f);
    float cx = r.p[i].u + t*ex - q.u, cy = r.p[i].v + t*ey - q.v;
    float d = std::sqrt(cx*cx + cy*cy);
    if (d < dOut) { dOut = d; sOut = r.S[i] + t*std::sqrt(len2); found = true; }
  }
  return found;
}

void onStep(Track& k, const Route& r, float stepLen) {
  k.s = std::min(k.s + stepLen, r.L);
  k.P += k.q * stepLen;
}

void onScanDone(Track& k) { k.s_at_scan = k.s; }

enum FixResult { FIX_USED, FIX_REJECTED, FIX_OFF_ROUTE };

FixResult onFix(Track& k, const Route& r, Pt fix, float acc) {
  if (acc > 50.0f) return FIX_REJECTED;
  float Rv = acc * acc;
  float win = 30.0f + 3.0f * std::sqrt(k.P);
  float sFix, dPerp;
  if (!project(r, fix, k.s_at_scan - win, k.s_at_scan + win, sFix, dPerp)) return FIX_REJECTED;
  if (dPerp > 15.0f) { return (++k.offRoute >= 3) ? FIX_OFF_ROUTE : FIX_REJECTED; }
  float innov = sFix - k.s_at_scan;
  if (std::fabs(innov) > std::max(15.0f, 3.0f * std::sqrt(k.P + Rv))) return FIX_REJECTED;
  k.offRoute = 0;
  float K = k.P / (k.P + Rv);
  k.s = std::clamp(k.s + K * innov, 0.0f, r.L);
  k.P = (1.0f - K) * k.P;
  return FIX_USED;
}
```

Accuracy radii of tens of metres give a small gain K, so step counting stays in charge over short distances and fixes mainly stop drift building up over long walks. Whether q and the gating thresholds suit the real venue has not been measured.

### 19.8 Display and rendering on the 368 x 448 AMOLED

The panel is 368 x 448 portrait. One RGB565 frame is 368 x 448 x 2 = 329,728 bytes (about 322 KiB), too large to keep comfortably in internal SRAM beside the Wi-Fi and TLS buffers, so the framebuffer and the pre-rendered map texture live in the 8 MB PSRAM; the ESP32-S3's GDMA reads PSRAM directly for SPI transfers.

The bundled `Arduino_ESP32QSPI.h` defaults to 40 MHz and 1024 pixels per transaction. Four data lines at 40 MHz give 20 MB/s raw, so a full-frame push takes at least about 16.5 ms, a theoretical ceiling near 60 fps before command overhead and the perspective warp itself. Real frame rates were not measured.

The tilted map is built by inverse homography: each destination pixel in the map viewport samples the top-down map texture (pre-rasterised into PSRAM with the route drawn on it) at a precomputed coordinate. The lookup table is cached between frames and rebuilt only when the camera moves by more than a pixel. The arrow, distance text and status band are overlays pushed as dirty rectangles through the controller's address window, and the map region is redrawn only when s changes enough to move the camera. Some QSPI AMOLED controllers need even-aligned windows; this was not verified for the SH8601 or CO5300, so if partial updates show garbled edges, round each window outward to even coordinates. The V2 CO5300 needs the column offset of 16 (19.2); the firmware probes the touch address at boot and picks the driver and offset to match, so one image runs on both revisions.

The layout is portrait, not a scaled copy of the CGOTCHI landscape layout. From top to bottom: a band of about 64 px with the destination name and metres remaining, a 3D map viewport of about 368 x 300 px with the arrow anchored in its lower third, and a band of about 84 px with the next instruction and the "I arrived" button. Touch targets should be at least 48 px.

An AMOLED has no backlight and black pixels are off, so the dark map theme also saves power. `setBrightness(0–255)` writes the panel's brightness register; Waveshare's examples use 255 and the panel is rated at 350 nit, while 150–180 is a reasonable indoor start, dropping when idle. A static UI on a shared OLED device risks image retention over a multi-day event, so the status band and arrow should shift by 1–2 px every few minutes, full-white areas should be avoided, and the screen should blank to black in the dock or after 60 s idle.

### 19.9 Audio

The arrival clip plays through the ES8311 over I2S, following Waveshare's `15_ES8311` sketch: the codec is configured over I2C at 0x18 with MCLK from the MCLK pin at 256 x fs, the ESP32 is the I2S master, and the speaker amplifier is enabled on GPIO46. The clip is stored as 16 kHz 16-bit mono PCM in a flash array or the FAT partition; two seconds is 64 KB.

```cpp
// Reference implementation after examples/arduino*/examples/15_ES8311 (ESP_I2S + bundled es8311.c).
// ESP-IDF: i2s_channel_init_std_mode + esp_codec_dev (es8311_codec_new, pa_pin = BSP_POWER_AMP_IO).
#include <Wire.h>
#include "ESP_I2S.h"
#include "es8311.h"
#include "pin_config.h"

I2SClass i2s;
extern const uint8_t arrived_pcm[]; extern const size_t arrived_pcm_len;   // 16 kHz s16 mono

bool audioInit() {
  pinMode(PA, OUTPUT); digitalWrite(PA, LOW);                 // amp off when idle
  i2s.setPins(I2S_BCK_IO, I2S_WS_IO, I2S_DO_IO, I2S_DI_IO, I2S_MCK_IO);
  if (!i2s.begin(I2S_MODE_STD, 16000, I2S_DATA_BIT_WIDTH_16BIT, I2S_SLOT_MODE_MONO)) return false;
  es8311_handle_t es = es8311_create(0, ES8311_ADDRESS_0);    // I2C port 0, 0x18; Wire already begun
  if (!es) return false;
  const es8311_clock_config_t clk = { .mclk_inverted = false, .sclk_inverted = false,
      .mclk_from_mclk_pin = true, .mclk_frequency = 16000 * 256, .sample_frequency = 16000 };
  if (es8311_init(es, &clk, ES8311_RESOLUTION_16, ES8311_RESOLUTION_16) != ESP_OK) return false;
  es8311_voice_volume_set(es, 80, nullptr);                   // tune per revision
  return true;
}

void playArrived() {
  digitalWrite(PA, HIGH); delay(20);                          // let the amp settle
  i2s.write(arrived_pcm, arrived_pcm_len);
  delay(50); digitalWrite(PA, LOW);
}
```

Waveshare's IDF example defaults to speaker volume 70 on V1 and 90 on V2 and notes V2 output is quieter, so volume is chosen from the detected revision. If mono slot mode drives only one channel, use `I2S_SLOT_MODE_STEREO` with duplicated samples. GPIO45 (WS) and GPIO46 (PA) are ESP32-S3 strapping pins, so board rework must not add pull resistors to them.

### 19.10 Power

The AXP2101 charges from USB and the MX1.25 battery connector. Through XPowersLib it reports battery voltage, percentage, charging state and VBUS presence (`getBattVoltage()`, `getBatteryPercent()`, `isCharging()`, `isVbusIn()`), and supports long-press power-off. The UI shows a battery icon, and dock insertion is detected through VBUS so the device enters a black-screen idle state with Wi-Fi off while charging. Unlike the CGOTCHI build, battery level is therefore available to the dock.

The wiki quotes, for a 400 mAh cell, about 1 hour at full brightness, 3–4 hours with the display dark and about 6 hours in low-power use. These are vendor figures, not measured on the navigator, and all are shorter than an event day, which is why the docking model matters.

Wi-Fi is the largest energy consumer the navigator controls: an active scan keeps the radio receiving for over a second with probe-request bursts, and each HTTPS POST adds a TLS handshake. The scan policy in 19.6 saves power as well as API calls. Between scans the station stays associated in modem-sleep (`WIFI_PS_MIN_MODEM`, the default); when the radio is not needed at all (docked, or idle on the destination picker) `WiFi.mode(WIFI_OFF)` / `esp_wifi_stop()` saves more, at the cost of a few seconds to re-join.

### 19.11 Security and cost

An API key compiled into firmware can be recovered by anyone with physical access, since `esptool.py read_flash` dumps flash in the clear unless flash encryption is enabled, and these devices are handed to the public. The key must be treated as exposed and its reach limited.

| Measure | Effect | Notes |
| --- | --- | --- |
| Restrict key to the Geolocation API | A leaked key cannot call other Maps or Cloud APIs | Cloud console, Credentials |
| IP-address application restriction | Key works only from the venue's public egress IP | Only if the venue NAT IP is static and known |
| Per-day quota cap | Hard ceiling on cost if the key leaks | Pair with billing alerts |
| Key in NVS, not in source or git | Keeps the key out of repositories and published binaries | Provision per batch over USB |
| Flash encryption + secure boot | Makes casual extraction harder | Irreversible eFuse burns; plan before enabling |
| Proxy service (recommended) | Key never leaves the server; devices use revocable per-device tokens; proxy can rate-limit, cache, log and filter | Adds a server dependency; the device posts the same JSON to the proxy |

Each request sends the BSSIDs of nearby access points to Google. The venue organisers should be told, and no device or user identifiers are sent.

The API is billed per request. Google's Maps Platform pricing list (last updated 17 Nov 2026) shows the Geolocation SKU in the Essentials category with the first 10,000 events per month free, then USD 5.00 per 1,000 up to 100,000, with lower tiers above. Whether `notFound` responses are billed is not stated. Request volume is roughly devices x walking hours x 3600 / interval: 50 devices walking 2 hours a day at one request per 10 s is about 36,000 requests a day, which would use the free cap within the first day, so the scan policy and duplicate-skip rule in 19.6 matter. Check the pricing page before budgeting.

### 19.12 What changed from the plan

The plan listed Wi-Fi positioning through the Geolocation API as future v2 work on the CGOTCHI. It could not be built there, because the CGOTCHI guest sandbox exposes no radio API, so the app cannot scan access points or open its own HTTPS connection. Version 2 was delivered instead as native firmware on a different ESP32-S3 board, with the user flow kept the same.

| Aspect | Plan (v2 on CGOTCHI) | As built and tested (v2) |
| --- | --- | --- |
| Board | CGOTCHI (ESP32-S3) | Waveshare ESP32-S3-Touch-AMOLED-1.8 |
| Firmware model | Guest app in the CGOTCHI sandbox | Native firmware with full radio access |
| Positioning | Step counting + Wi-Fi fixes (future) | Step counting + Google Geolocation fixes, fused on route arc length |
| Network | Not available in the sandbox | Venue Wi-Fi STA, HTTPS to googleapis.com |
| Display | 2.01" 296 x 240 IPS, landscape layout | 1.8" 368 x 448 AMOLED, portrait layout |
| Touch | CST810 | FT3168 (V1) or CST820 (V2) on I2C |
| IMU | SC7A20 / LIS2DH12 accelerometer | QMI8658 6-axis @ 0x6B |
| Audio | ES8311 + NS4150B | ES8311 + amplifier on GPIO46 |
| Offline behaviour | Fully offline | Falls back to Version 1 behaviour without network or API |
| Running cost | None | Per-request API billing (19.11) |
| User flow | Pick destination, 3D route + arrow, "You've arrived!", home-zone reset, "I arrived" | Unchanged |
| Map | Real Dubai Future Week map | Unchanged, plus a georeference (A, b, anchor) for fixes |

### 19.13 Known limitations of v2

1. No accuracy, latency, battery or cost measurements exist yet; every threshold in 19.6–19.10 is a starting value.
2. Fix quality depends on Google's database containing the venue's access points. Access points installed temporarily for the event, or recently moved, may be unknown; those spots return `notFound` or coarse fixes and the device runs on step counting alone.
3. Enterprise access points often use locally administered BSSIDs for secondary SSIDs; these are filtered out because Google cannot locate them, which can leave too few in dense halls.
4. The API returns no floor or level, so the device cannot tell levels apart in a multi-level venue.
5. The confidence level of `accuracy` is undocumented; using it as 1σ for R is an assumption.
6. Each fix arrives one to several seconds after its scan. Innovation against `s_at_scan` compensates, but the latency still caps how quickly the device can correct itself.
7. Fusion constrains position to the active route. A visitor who leaves the route is detected only through the 15 m perpendicular gate, and re-planning needs the user to confirm.
8. Captive portals, WPA2-Enterprise or congested venue Wi-Fi can block the network path completely without venue IT involvement.
9. The API key in firmware is exposed (19.11) until a proxy is in place.
10. Battery life on a small cell with Wi-Fi active is short, which makes docking essential.
11. A static AMOLED UI risks image retention over a multi-day event.
12. Correct behaviour on both board revisions depends on runtime detection; Waveshare's CI checks compilation only, and no schematic is published.

### 19.14 References

- [Waveshare ESP32-S3-Touch-AMOLED-1.8 repository](https://github.com/waveshareteam/ESP32-S3-Touch-AMOLED-1.8): `pin_config.h` (both Arduino sets), `board_variant.c`, `15_ES8311`, `12_i2s_codec`, `docs/CI.md`, `docs/GETTING_STARTED.md`
- [Waveshare documentation](https://docs.waveshare.com/ESP32-S3-Touch-AMOLED-1.8) and [wiki](https://www.waveshare.com/wiki/ESP32-S3-Touch-AMOLED-1.8)
- [Geolocation API: request and response](https://developers.google.com/maps/documentation/geolocation/requests-geolocation)
- [Geolocation API: errors](https://developers.google.com/maps/documentation/geolocation/requests-geolocation-errors)
- [Geolocation API: usage and billing](https://developers.google.com/maps/documentation/geolocation/usage-and-billing)
- [Google Maps Platform pricing](https://developers.google.com/maps/billing-and-pricing/pricing)
- [Google `_nomap` opt-out](https://support.google.com/maps/answer/1725632)
- [Google Trust Services roots](https://pki.goog/roots.pem)
- [ESP-IDF certificate bundle](https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/protocols/esp_crt_bundle.html)
- [ESP32-S3 datasheet](https://documentation.espressif.com/esp32-s3_datasheet_en.html)

## 20. References

- [CGOTCHI product site](https://cgotchi.com/)
- [MicroPixel / CGOTCHI official repository](https://github.com/ALTIO-AI-PRIVATE-LIMITED/micropixel-gotchi)
- [Community map navigation project on Cheeko Gotchi V2](https://github.com/chaitanyaasati/mapnavigation)
- [Google Geolocation API: requests and responses](https://developers.google.com/maps/documentation/geolocation/requests-geolocation)
- [Google Geolocation API: usage and billing](https://developers.google.com/maps/documentation/geolocation/usage-and-billing)
- [ESP-IDF Wi-Fi FTM example](https://github.com/espressif/esp-idf/blob/dc859c1e67e/examples/wifi/ftm/README.md)
- [Future Week Navigator: Build Summary](https://claude.ai/artifact/L172vf1HEzSRvoJZoNiE9M)
- [Screen designs](https://claude.ai/artifact/7AGMn8cgRVsWMJWb9eUkzS)
- [Waveshare ESP32-S3-Touch-AMOLED-1.8 wiki](https://www.waveshare.com/wiki/ESP32-S3-Touch-AMOLED-1.8)
- [Future Week Navigator: Research Paper](https://claude.ai/artifact/XLGyCxQiZ2vvquE9DT2VH3)
