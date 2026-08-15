# Moonraker Timelapse — Cinematic & Dynamic Bed Parking Enhancement

A feature-rich extension for `moonraker-timelapse` that enhances 3D printer timelapses with **Dynamic Toolhead Bed Parking** (physical camera dolly slide) and **Cinematic Virtual Camera Motion** (Ken Burns effect, temporal frame rate interpolation, and exposure stabilization) without changing frame capture workflows or compromising printer performance.

---

## Key Features

1. **Automatic Layer Change Detection (`auto_layer` Mode)**:
   - Automatically tracks physical Z-height layer increases in real time via Klipper status updates (`gcode_move`).
   - Requires **ZERO Slicer setup** — no need to insert `TIMELAPSE_TAKE_FRAME` into layer change G-Code!
   - Triggers clean toolhead parking (static or dynamic dolly slide) automatically at every physical layer.

2. **Dynamic Toolhead Bed Parking (Physical Camera Dolly Slide)**:
   - Moves the print bed progressively along the Y-axis across layers from `park_dynamic_y_min` to `park_dynamic_y_max`.
   - Creates a physical camera dolly slide illusion in the timelapse video without adding print time.
   - Clamped within safe physical boundaries (`y_min` to `y_max`) to protect bed cables and endstops.
   - Automatically falls back to static parking if layer count is unknown.

2. **Duration-Preserving 30 FPS / 60 FPS Output**:
   - Separates the source frame timeline rate (e.g. 10 FPS) from the video output framerate (e.g. 30 FPS or 60 FPS).
   - Generates smooth intermediate frames without shortening the final timelapse duration (`Duration = total_frames / source_timeline_fps`).

3. **Ken Burns Virtual Camera Effect**:
   - Adds smooth, continuous virtual camera zoom and pan across the entire timelapse.
   - Smooth Cosine Easing prevents sudden camera movements at the start or end.
   - Dynamic boundary calculation ensures zero black borders or empty frame margins.

4. **Configurable Target Zoom Focus Position**:
   - `kenburns_target_x` (0% to 100%, default 50%): Horizontal zoom focus point.
   - `kenburns_target_y` (0% to 100%, default 50%): Vertical zoom focus point.
   - Allows zooming directly into off-center 3D prints on the print bed.

5. **Exposure Deflickering**:
   - Uses FFmpeg `deflicker` to remove frame-to-frame webcam brightness variations.

6. **Resource-Isolated High-Performance Pipeline**:
   - Single FFmpeg filtergraph: zero intermediate JPEGs saved to disk.
   - Optimized filter order (`zoompan` at 10 FPS source before upsampling) reduces CPU scaling workload by 66%.
   - Uses H.264 `superfast` encoder preset and bounded memory buffers (`size=5`) to prevent SWAP memory thrashing.
   - Runs under `nice -n 19` with CPU thread limits to protect Klipper print execution priority.
   - Automatic fallback to standard rendering if any filter fails.

---

## Configuration & Usage

Add or modify the following options in your `moonraker.conf` file under the `[timelapse]` section:

```ini
[timelapse]
output_path: ~/printer_data/timelapse/
frame_path: ~/printer_data/timelapse/frame
snapshoturl: http://localhost:1984/api/frame.jpeg?src=printer

# ----------------------------------------------------------------------
# Dynamic Toolhead Bed Parking Options (Linear Progressive Slide)
# ----------------------------------------------------------------------

# Enable/disable progressive Y-axis bed parking move across layers (True / False)
# Range: True, False | Default: False
park_dynamic_enabled: False

# Minimum Y-axis bed parking position in mm (start of print)
# Range: 0.0 to 220.0 mm | Default: 30.0
park_dynamic_y_min: 30.0

# Maximum Y-axis bed parking position in mm (end of print)
# Range: 0.0 to 220.0 mm | Default: 180.0
park_dynamic_y_max: 180.0

# ----------------------------------------------------------------------
# Cinematic Render Enhancement Options
# ----------------------------------------------------------------------

# Master switch for cinematic rendering pipeline (True / False)
# Range: True, False | Default: True
cinematic_enabled: True

# Enable smooth Ken Burns virtual camera zoom and pan (True / False)
# Range: True, False | Default: True
kenburns_enabled: True

# Target zoom factor ratio at the end of the timelapse
# Range: 1.0 (no zoom) to 2.0 (2x zoom) | Recommended: 1.05 to 1.15 | Default: 1.06
kenburns_zoom: 1.06

# Horizontal target focus point on camera POV in percentage (0% left, 50% center, 100% right)
# Range: 0.0 to 100.0 | Default: 50.0
kenburns_target_x: 50.0

# Vertical target focus point on camera POV in percentage (0% top, 50% center, 100% bottom)
# Range: 0.0 to 100.0 | Default: 50.0
kenburns_target_y: 50.0

# Enable temporal brightness stabilization to reduce webcam flicker (True / False)
# Range: True, False | Default: True
exposure_stabilization: True

# Enable temporal frame rate interpolation (True / False)
# Range: True, False | Default: True
temporal_interpolation: True

# Source timeline framerate equivalent (speed of print construction)
# Range: 1 to 60 FPS | Default: 10
source_timeline_fps: 10

# Final output video framerate
# Range: 10, 30, 60 FPS | Default: 30
cinematic_output_fps: 30
```

---

## Implementation & Architecture

### 1. Klipper Macro (`klipper_macro/timelapse.cfg`)
- **`TIMELAPSE_TAKE_FRAME`**: When `park_dynamic_enabled` is active, queries `current_layer` and `total_layer` from Klipper `print_stats`.
- Calculates progressive target Y position:
  `target_y = y_min + (current_layer - 1) / (total_layers - 1) * (y_max - y_min)`
- Clamps `target_y` safely within `[y_min, y_max]`.
- Moves the toolhead/bed to the calculated position before taking the snapshot.

### 2. Moonraker Component (`component/timelapse.py`)
- **`__init__(self, confighelper)`**:
  Registers configuration defaults for cinematic rendering and dynamic bed parking.
- **`webrequest_settings(self, webrequest)`**:
  Handles API requests at `/machine/timelapse/settings` and persists settings to Moonraker DB.
- **`setgcodevariables(self)`**:
  Transmits dynamic parking parameters (`PARK_DYNAMIC_ENABLE`, `PARK_DYNAMIC_Y_MIN`, `PARK_DYNAMIC_Y_MAX`) to Klipper via `_SET_TIMELAPSE_SETUP`.
- **`render(self, webrequest=None)`**:
  Executes high-performance FFmpeg rendering pipeline with `nice -n 19`, `-threads 2`, `-preset superfast`, and fallback error handling.

---

## Update Manager Setup in `moonraker.conf`

To track update changes from your GitHub fork without Mainsail update warnings:

```ini
[update_manager timelapse]
type: git_repo
primary_branch: feature/cinematic-v3
path: ~/moonraker-timelapse
origin: https://github.com/gustiarto/moonraker-timelapse.git
managed_services: klipper moonraker
```

---

## License

GNU General Public License v3.0 (GPLv3).
