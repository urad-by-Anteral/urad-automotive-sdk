# Medium Range Radar (MRR)

Long-range detection and tracking of vehicles and obstacles for the uRAD
Automotive radar, using the TI Medium Range Radar firmware
(`xwr18xx_mrr_demo.bin` ([Releases](../../../../releases)), from the
TI Radar Toolbox 4.00.00.05).

The firmware time-multiplexes **two radar modes in one frame** ("advanced
frame" with two subframes), so a single sensor simultaneously provides:

| Subframe | Mode | Max range | Range resolution | Max velocity | Outputs |
|---|---|---|---|---|---|
| 0 — **MRR** (Medium Range Radar) | TX beamforming, velocity disambiguation | ~120 m | ~0.7 m | ~150 km/h | Detected points + tracked objects (EKF) |
| 1 — **USRR** (Ultra Short Range Radar) | TDM-MIMO, 12 virtual antennas | ~30 m | ~4.3 cm | ~36 km/h | Dense point cloud + DBSCAN clusters + parking assist border |

Typical use cases: forward collision awareness and adaptive-cruise-style
object tracking (MRR subframe), cross-traffic and vulnerable road user
detection, automated parking and obstacle proximity mapping (USRR
subframe).

> **⚠ Antenna caveat — AWR1843AoP vs. AWR1843BOOST:** TI designed this demo
> for the AWR1843BOOST evaluation board (PCB-etched ISK-style antennas).
> The uRAD Automotive is based on the AWR1843**AoP** (antenna-on-package),
> whose antenna array geometry, element spacing and fields of view differ.
> Range and velocity measurements are unaffected, but **angle estimation
> (azimuth/elevation → X-Y-Z positions) and the MRR TX beamforming pattern
> are calibrated for the BOOST layout** and may be inaccurate on this
> product until validated. The firmware also has hardcoded range bias and
> RX phase compensation values measured on a BOOST board (see the "Range
> Bias and Rx Channel Gain/Offset Measurement" section of the TI user
> guide for the recalibration procedure, which requires rebuilding the
> firmware). Treat angular positions as experimental on this hardware.

## How it works

Processing chain (runs on the on-chip C674x DSP; summary of the TI user
guide):

1. **Range FFT** per chirp, per RX antenna (1D processing).
2. **Doppler FFT** per range bin (2D processing) during the inter-chirp
   idle time, producing the range-velocity matrix.
3. **CFAR detection** in both the Doppler and range directions, plus peak
   grouping and SNR/magnitude pruning against ground clutter.
4. **Direction of arrival** (azimuth, and elevation in the USRR subframe)
   to place each detection in X-Y-Z.
5. **Velocity disambiguation** (MRR subframe only): the subframe contains
   *fast* and *slow* chirps with slightly different repetition periods;
   combining both velocity estimates through the Chinese remainder theorem
   extends the maximum unambiguous velocity to ~150 km/h.
6. **Clustering** (DBSCAN) groups the detected points into rectangles
   (center + half-extents). USRR clusters are streamed out; in cross-traffic
   scenes they outline vehicles crossing the field of view.
7. **Tracking** (MRR subframe): an Extended Kalman Filter with states
   [x, y, vx, vy] and measurements [range, radial velocity, sin(azimuth)],
   fed by the strongest object of each cluster. Tracked objects are
   streamed out with position, velocity vector and cluster size.
8. **Parking assist** (USRR subframe): the nearest obstruction distance is
   computed for 32 azimuth sectors (bins of sin(azimuth) across the full
   field of view, clamped to 20 m), forming a "parking border" arc.

## Required firmware

Flash `xwr18xx_mrr_demo.bin` ([Releases](../../../../releases))
(unmodified TI prebuilt binary) with TI **UniFlash**:

1. Put the board in **flashing mode** with the DIP switches (see chapter 3
   of the [uRAD Automotive User Manual](../../docs/user-manual-en.pdf) for
   the switch positions and connector pinout of this product).
2. In UniFlash select the AWR1843 device, load the `.bin` in the *Program*
   tab and set the **application/user UART COM port** in *Settings &
   Utilities*, then power-cycle and *Load Images*.
3. Put the board back in **functional mode** and power-cycle again.

Unlike most TI demos, this firmware **configures itself and starts
streaming immediately on boot**: it accepts **no UART commands at all**
(the CLI is compiled out of the prebuilt binary), performs its RF
configuration internally through the "advanced frame" API, and transmits
detection data continuously on the data UART until powered off.

## Chirp configuration

**There is no `.cfg` chirp file for this application.** The RF
configuration is compiled into the firmware. The prebuilt binary uses:

| Compiled-in choice | Value | Source (TI Radar Toolbox 4.00.00.05) |
|---|---|---|
| Operation mode | MRR + USRR subframes (2 subframes per frame) | `SUBFRAME_CONF_MRR_USRR` in `src/common/mrr_config_consts.h` |
| MRR chirp design | 120 m variant | `MRR_RANGE_120m` → `mrr_config_chirp_design_MRR120.h` |
| USRR chirp design | 30 m variant | `mrr_config_chirp_design_USRR30.h` |
| Frame periodicity | 30 ms MRR + 30 ms USRR (~16.7 Hz per subframe) | chirp design headers |

To change any of these (MRR-only or USRR-only modes, the 80 m MRR design,
the USRR20/USRR50 designs, detection thresholds), edit the headers under
`source/ti/examples/Automotive_ADAS_and_Parking/medium_range_radar/src/`
of the TI Radar Toolbox and rebuild the firmware in Code Composer Studio —
see the "Demo Configuration" and "Building from Source Code" sections of
the TI user guide. There is nothing to tune at runtime.

## Running the client

The Python client is part of the shared
[urad-mmwave](https://github.com/urad-by-Anteral/urad-mmwave-core) SDK — no
code in this folder is needed:

```bash
pip install git+https://github.com/urad-by-Anteral/urad-mmwave-core.git
```

Because the firmware needs no configuration, the client only opens the
**data serial port** (the auxiliary data COM port, 921600 baud). Windows
example:

```bash
urad-mrr --data-port COM7
```

Raspberry Pi (single-UART adapter; the optional GPIO reset restarts the
stream from frame 0):

```bash
urad-mrr --data-port /dev/serial0 --gpio-reset-pin 6
```

### CLI reference

| Flag | Meaning |
|---|---|
| `-c, --config PATH` | Optional JSON profile ([`config_radar.json`](config_radar.json)); only the `data_serial` section is used. The `control_serial` section is required by the shared schema but ignored — this firmware has no control channel. |
| `--data-port PORT` | Data serial port (e.g. `COM7`, `/dev/serial0`). Required unless `--config` provides it. |
| `--baudrate N` | Data UART baud rate (default 921600, fixed by the firmware — change only if you rebuilt the firmware). |
| `--output-dir DIR` | Directory for the output text files (default `./output`). |
| `--no-save` | Disable all file output. |
| `--gui` | Live top-view window (requires `pip install urad-mmwave[gui]`). |
| `--duration SECONDS` | Stop after this many seconds. |
| `--max-frames N` | Stop after N frames (each subframe counts as one frame). |
| `--gpio-reset-pin PIN` | BCM pin to reset the chip before reading (Raspberry Pi). |
| `-v, --verbose` | Debug logging. |
| `--version` | Print the client version. |

There is **no `--chirp` flag** for this application: the configuration is
compiled into the firmware (see above).

For programmatic use, decode packets with
`urad_mmwave.apps.medium_range_radar.parse_frame` or iterate
`urad_mmwave.apps.medium_range_radar.stream("COM7")` — see the module
docstring for the frame model.

## GUI

`--gui` opens a live top view (X-Y plane, radar at the origin looking up
the Y axis) drawing both subframes together:

- **Green dots** — MRR subframe detected points (long range).
- **Blue dots** — USRR subframe detected points (short range, dense).
- **Gray rectangles** — USRR DBSCAN clusters (center ± half-extents).
- **Orange squares** — MRR tracked objects, each with an orange **velocity
  vector** (the line covers the distance the object travels in 1 s) and a
  dashed box showing the associated cluster size.
- **Red arc** — parking assist border: the nearest obstruction per azimuth
  sector; sectors at exactly 20 m are free space (that is the firmware's
  clamp value).

The window title shows the live counts. Points, clusters and tracked
objects refresh at the subframe rate (~16 Hz each).

## Output

### Console

One line per received frame:

```
MRR   points:  12  trackers: 2
USRR  points: 240  clusters: 5
```

### Files (disable with `--no-save`)

Appended in `--output-dir`, one line per frame with the host epoch
timestamp as the last value (frames with no data are skipped):

| File | Line content |
|---|---|
| `PointCloud.txt` | `subframe` (0 = MRR, 1 = USRR) then `x y z doppler peak_val` per point, then timestamp |
| `Trackers.txt` | `x y vx vy x_size y_size` per tracked object (MRR frames), then timestamp |
| `Clusters.txt` | `x_center y_center x_size y_size` per cluster (USRR frames), then timestamp |
| `ParkingAssist.txt` | 32 ranges in meters, one per azimuth bin (USRR frames), then timestamp |

Units: meters and m/s; negative doppler = approaching the radar.

### UART protocol (for integrators)

Little-endian binary stream on the data UART at **921600 baud**. Each
packet: 8-byte sync word `01 02 03 04 05 06 07 08`, then a 32-byte header,
then the TLVs, then `0x0F` padding up to a 32-byte multiple
(`total_packet_len` includes the padding).

Header after the sync word (`uint32` each): `version`, `total_packet_len`,
`platform`, `frame_number`, `time_cpu_cycles`, `num_detected_obj`,
`num_tlvs`, `subframe_number`. **`subframe_number` selects the content: 0 =
MRR subframe, 1 = USRR subframe.**

Each TLV: `uint32 type`, `uint32 length` (length excludes this 8-byte
header), then a 4-byte descriptor `uint16 num_elements`, `uint16 q_format`
(always 7: divide fixed-point values by 2^7 = 128), then the elements:

| Type | Name | Subframe | Element (little-endian) | Element size |
|---|---|---|---|---|
| 1 | Detected points | both | `int16 speed` (m/s·2^q), `uint16 peak_val`, `int16 x, y, z` (m·2^q) | 10 B |
| 2 | Clusters | USRR | `int16 x_center, y_center, x_size, y_size` (m·2^q; sizes are half-extents) | 8 B |
| 3 | Tracked objects | MRR | `int16 x, y, vx, vy` (m·2^q, m/s·2^q), `int16 x_size, y_size` (m·2^q) | 12 B |
| 4 | Parking assist | USRR | `uint16 range` (m·2^q) per azimuth bin, 32 bins | 2 B |

Parking assist bin layout: bins quantize sin(azimuth) over [-1, 1); bins
0…15 cover sin(azimuth) ∈ [0, 1) (boresight to the right), bins 16…31
cover [-1, 0) (left of boresight). A bin equal to 20 m (the maximum) means
no obstruction in that sector.

A TLV is present only when it has content (e.g. no tracked-objects TLV
when nothing is tracked); a frame can carry zero TLVs. Note that TLV
types 2–4 collide with the numbering of the standard out-of-box demo
(range profile, noise profile, azimuth heatmap) — **do not** decode this
stream with an out-of-box parser.

## ⚠ One configuration per boot / continuous streaming

TI Radar Toolbox firmwares accept a single configuration per boot; for
most applications this means you must reset the radar between runs. This
firmware is the extreme case: it **configures itself on boot and streams
forever** — it cannot be stopped, reconfigured or restarted over UART.
Stopping the client simply closes the port; the sensor keeps transmitting.
To restart the stream (or recover from a fault), power-cycle the board:
unplug and replug the USB cable, press the reset button, or drive the
RESET pin (on Raspberry Pi setups `--gpio-reset-pin` does this before
reading).

## Troubleshooting

- **No packets / timeout after ~30 s** — you are probably on the wrong COM
  port: the data streams on the *auxiliary data* port, not the
  *application/user* port used for flashing. Check both candidates in the
  device manager. Also confirm the board is in functional mode (DIP
  switches) and was power-cycled after flashing.
- **Client reports resynchronization warnings or implausible lengths** —
  baud rate mismatch (the firmware transmits at 921600) or a host UART
  that cannot sustain 921600 baud reliably (some USB-serial adapters and
  long cables drop bytes; the parser resynchronizes on the next frame).
- **Frames arrive but positions look angularly wrong** — expected on this
  product until validated: see the AoP antenna caveat at the top.
- **Nothing detected at long range** — the MRR subframe is tuned for
  vehicle-sized targets (~10 m² RCS at 120 m); pedestrians and small
  objects are only visible much closer. Indoor tests mostly exercise the
  USRR subframe.
- **LEDs** — a steady power LED with no data usually means flashing mode
  is still selected; see chapter 3 of the User Manual.

Links: [TI Medium Range Radar user guide](https://dev.ti.com) (Radar
Toolbox → Automotive_ADAS_and_Parking → medium_range_radar),
[TI Radar Toolbox](https://www.ti.com/tool/RADAR-TOOLBOX),
[uRAD Automotive User Manual](../../docs/user-manual-en.pdf),
[urad-mmwave-core](https://github.com/urad-by-Anteral/urad-mmwave-core).

## Credits

Based on the **Medium Range Radar** example of the TI Radar Toolbox
**4.00.00.05** (Texas Instruments). The firmware binary is redistributed
unmodified with TI's authorization and remains subject to the TI license.
The uRAD client code (`urad-mrr` in
[urad-mmwave-core](https://github.com/urad-by-Anteral/urad-mmwave-core)) is
released under the MIT License by Anteral.
