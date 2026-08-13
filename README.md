# Moonraker Timelapse — Cinematic Render Enhancement

A feature-rich extension for `moonraker-timelapse` that enhances 3D printer timelapses with virtual camera movement (Ken Burns effect), frame rate interpolation, and exposure stabilization without changing frame capture workflows or compromising printer performance.

---

## Key Features

1. **Dynamic Toolhead Bed Parking (Physical Dolly Slide)**:
   - Moves the print bed progressively along the Y-axis across layers ($Y_{\text{min}}$ to $Y_{\text{max}}$).
   - Creates a physical camera dolly slide illusion in the timelapse video without adding print time.

2. **Duration-Preserving 30 FPS / 60 FPS Output**:
   - Separates the source frame timeline rate (e.g. 10 FPS) from the video output framerate (e.g. 30 FPS or 60 FPS).
   - Generates smooth intermediate frames without shortening the final timelapse duration (`Duration = total_frames / source_timeline_fps`).

3. **Ken Burns Virtual Camera Effect**:
   - Adds smooth, continuous virtual camera zoom and pan across the entire timelapse.
   - Smooth Cosine Easing prevents sudden camera movements at the start or end.
   - Dynamic boundary calculation ensures zero black borders or empty frame margins.

3. **Configurable Target Zoom Focus Position**:
   - `kenburns_target_x` (0% to 100%, default 50%): Horizontal zoom focus point.
   - `kenburns_target_y` (0% to 100%, default 50%): Vertical zoom focus point.
   - Allows zooming directly into off-center 3D prints on the print bed.

4. **Exposure Deflickering**:
   - Uses FFmpeg `deflicker` to remove frame-to-frame webcam brightness variations.

5. **Resource-Isolated Streaming Pipeline**:
   - Single FFmpeg streaming filtergraph: zero intermediate JPEGs saved to disk.
   - Runs under `nice -n 19` with CPU thread limits to prevent interference with Klipper print execution or go2rtc webcam streaming.
   - Automatic fallback to standard rendering if any filter fails.

---

## Configuration & Usage

Add or modify the following options in your `moonraker.conf` file under the `[timelapse]` section:

```ini
[timelapse]
output_path: ~/printer_data/timelapse/
frame_path: ~/printer_data/timelapse/frame

# ----------------------------------------------------------------------
# Cinematic Render Enhancement Configuration
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

## Implementation & Code Architecture

All cinematic enhancements are encapsulated inside [`component/timelapse.py`](component/timelapse.py):

### Core Functions & Methods

1. **`__init__(self, confighelper)`**:
   - Registers configuration defaults (`cinematic_enabled`, `kenburns_enabled`, `kenburns_zoom`, `kenburns_target_x`, `kenburns_target_y`, `exposure_stabilization`, `temporal_interpolation`, `source_timeline_fps`, `cinematic_output_fps`).
   - Merges database overrides and `moonraker.conf` settings.

2. **`webrequest_settings(self, webrequest)`**:
   - Handles `GET` and `POST` HTTP requests to `/machine/timelapse/settings`.
   - Validates parameter ranges (e.g. clamping target X/Y coordinates between 0% and 100%).
   - Persists updated settings into Moonraker DB (`self.database.insert_item`).

3. **`get_frame_dimensions(self, filepath)`**:
   - Reads the JPEG file header to extract original frame width and height (`width, height`) for `zoompan` scale calculation.

4. **`render(self, webrequest=None)`**:
   - **Timeline FPS & Duration Calculation**:
     `Duration = framecount / source_timeline_fps`
   - **FFmpeg Filtergraph Assembly**:
     - `orientation_filters`: Applies `transpose`, `hflip`, `vflip`, or `rotate`.
     - `deflicker=size=10:mode=pm`: Exposure stabilization.
     - `framerate=fps=30`: Temporal frame rate upsampling.
     - `zoompan`: Eased zoom and dynamic center pan around `(target_x, target_y)`.
   - **Process Isolation**: Executes FFmpeg via `nice -n 19` and `-threads 2`.
   - **Graceful Fallback**: If cinematic filtering fails, falls back automatically to standard renderer.

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
