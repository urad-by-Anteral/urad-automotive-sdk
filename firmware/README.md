# Firmware

Prebuilt firmware binaries for the uRAD Automotive (AWR1843AoP) are published
as assets of this repository's [Releases](../../../releases) — they are not
stored in the git history.

| Binary | Application | Notes |
|---|---|---|
| `out_of_box_1843_aop.bin` | Out-of-box demo | Point cloud streaming; used with [urad-mmwave](https://github.com/urad-by-Anteral/urad-mmwave-core) |
| `uRAD_LevelSensing_AWR1843AoP.bin` | Level sensing | Data UART at 921600 baud (default) |
| `uRAD_LevelSensing_AWR1843AoP_115200_br.bin` | Level sensing | Data UART at 115200 baud (for Arduino/slow hosts) |

## Flashing

1. Install [TI UniFlash](https://www.ti.com/tool/UNIFLASH).
2. Put the board in flashing mode (see the
   [user manual](../docs/user-manual-en.pdf) for the SOP jumper settings).
3. Load the `.bin` as *meta image* and flash.
4. Restore the functional mode jumpers and power-cycle the board.

The firmware images are built from the Texas Instruments mmWave SDK and are
redistributed for use with uRAD hardware, subject to the applicable TI
license terms. The Level Sensing binaries are built by Anteral from the TI
High Accuracy Level Sensing demo, with the distance correction applied on
the device (see
[applications/level_sensing](../applications/level_sensing/README.md)); the
two variants differ only in the data UART baud rate (921600 standard and
115200).
