# High Accuracy Level Sensing

Millimeter-accuracy distance measurement (12–150 m) for the uRAD Automotive
radar, using the dedicated Level Sensing firmware
(`uRAD_LevelSensing_AWR1843AoP*.bin`, see [Releases](../../../../releases)).

## Firmware variants

Unlike the other applications, these binaries are **built by Anteral** from
the TI High Accuracy Level Sensing demo: TI's original firmware reported a
raw distance that their visualizer corrected with a multiplication factor
in software; the uRAD firmware applies that correction on the device, so
the distances reported over UART are real distances and no factor must be
applied by the client.

The two variants differ only in the data UART baud rate — pick the one
your host supports:

| Firmware ([Releases](../../../../releases)) | Data UART baud rate | Use case |
|---|---|---|
| `uRAD_LevelSensing_AWR1843AoP_921600_br.bin` | 921600 (standard) | Default; used by the Python client |
| `uRAD_LevelSensing_AWR1843AoP_115200_br.bin` | 115200 | Arduino and hosts without high-speed UART |

The control UART is always 115200 baud.

## Python

The Python client is part of the shared
[urad-mmwave](https://github.com/urad-by-Anteral/urad-mmwave-core) SDK — no code in
this folder is needed:

```bash
pip install git+https://github.com/urad-by-Anteral/urad-mmwave-core.git
urad-level-sensing --model AWR --control-port COM8 --data-port COM7 --max-distance 12 --range 0 10 -n 5
```

Add `--gui` (requires `pip install urad-mmwave[gui]`) for a live plot of
the three ranges over time, or `--spectrum` for the full FFT range
profile computed from the raw ADC samples with the detected peaks and
their amplitudes marked. For Raspberry Pi single-UART setups add
`--gpio-reset-pin 6` and omit `--data-port`. Run
`urad-level-sensing --help` for all options, or use the API directly
(`urad_mmwave.apps.level_sensing.measure`, or `.stream` for continuous
readings).

## Other platforms

- [`cpp/level_sensing_UART.cpp`](cpp/level_sensing_UART.cpp) — portable
  desktop C++17 reference client (Windows and Linux, no dependencies;
  dual UART). Build with
  `g++ -std=c++17 -O2 -o level_sensing level_sensing_UART.cpp` or
  `cl /std:c++17 /O2 /EHsc level_sensing_UART.cpp`, then run
  `level_sensing COM5 COM4` (`--baud` must match the firmware variant).
- [`arduino/uRAD_LevelSensing.ino`](arduino/uRAD_LevelSensing.ino) — Arduino
  sketch (single UART: configures at 115200, then reopens for data).
  Set the `RadarDataBaudRate` define to the flashed firmware variant;
  this product's slow variant is the `115200_br` binary, so set it to
  115200.

## Documentation

[`docs/`](docs) contains the user manual (EN/ES), application notes (EN/ES)
and the performance report (EN).
