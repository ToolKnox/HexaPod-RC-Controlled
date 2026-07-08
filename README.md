# HexaPod-RC-Controlled

Firmware for an 18-servo hexapod walker, running on a custom STM32H750 controller board and driven from a hobby RC transmitter.

> **Status — early, documentation in progress.** The PCB and mechanical files are done and the firmware boots and walks; install instructions, pin tables, and the RC binding walkthrough below still have TODOs that the maker is filling in. Build at your own risk and expect to read the source.

Project page (renders, BOM, print files, build photos): <https://www.printables.com/model/1443108>

---

## What this is

- **MCU:** STM32H750VBT6 (Cortex-M7 @ 480 MHz) on a custom carrier PCB. The WeAct Studio H750 module is the reference; any generic H750 board with **≥ 8 MB / 64 Mbit QSPI flash** works. The H750 has only 128 KB of on-chip flash, so the application image lives in external QSPI — a board with too-small or missing QSPI flash will not run this firmware.
- **RTOS:** FreeRTOS, with separate tasks for leg control, IMU/DMP, and status LED.
- **IMU:** MPU6050 over I²C, using the InvenSense DMP driver vendored under `SourceCode/MDK-ARM/USER/DMP/`.
- **Actuation:** 18× LX-224 serial-bus servos (half-duplex UART, daisy-chained), three per leg.
- **Input:** hobby RC receiver (Spektrum or FrSky — see *RC receiver* below for the protocol the firmware currently parses).

## Repository layout

```
SourceCode/
├── Core/                       # CubeMX-generated init: clocks, GPIO, I2C, UART, DMA
│   ├── Inc/                    # main.h, FreeRTOSConfig.h, peripheral headers
│   └── Src/                    # main.c, freertos.c, peripheral .c
├── Drivers/                    # Vendored ST HAL + CMSIS — do not regenerate over the top
│   ├── STM32H7xx_HAL_Driver/
│   └── CMSIS/
└── MDK-ARM/                    # Keil μVision project
    ├── *.uvprojx               # Open this in Keil MDK-ARM
    ├── JLinkSettings.ini       # J-Link debugger settings
    ├── RTE/                    # Two build configurations:
    │   ├── _Hexapod/           #   release firmware
    │   └── _app_test/          #   bring-up / diagnostic build
    └── USER/
        ├── APP/                # bsp, leg, arm, Servo, gait_prg, remote, mpu6050, my_math
        ├── TASK/               # LegControl_task, MPU_task, led_task
        └── DMP/                # InvenSense MPU6050 DMP driver
```

## Hardware required

- Custom HexaPod controller PCB (Gerbers / order link on the [Printables page](https://www.printables.com/model/1443108); recommended to order from PCBWay **with assembly** for the SMD work).
- STM32H750VBT6 module with ≥ 8 MB QSPI flash (WeAct Studio reference, generic equivalents OK).
- 18× LX-224 (or compatible Lobot/HiWonder serial bus) servos.
- MPU6050 IMU (on-board on the controller PCB).
- Compatible RC receiver — see *RC receiver* below.
- Battery / regulator setup per the BOM.
- ST-Link V2 or **J-Link** debug probe (the project ships with `JLinkSettings.ini`, so J-Link is the expected default).

## Toolchain

This is a **Keil MDK-ARM (μVision) project**. There is no PlatformIO/CMake/Makefile build today.

- Keil MDK-ARM 5.x with the **Keil.STM32H7xx_DFP** device family pack installed.
- ARM Compiler 6 (bundled with MDK).
- J-Link tools (or ST-Link if you adapt the debug settings).

> Porting to STM32CubeIDE / CMake is on the wish list but not done — if you take a stab at it, a PR is very welcome.

## Build & flash

1. Clone the repo.
2. Open `SourceCode/MDK-ARM/*.uvprojx` in Keil μVision.
3. Select the **Hexapod** target in the configuration dropdown (the `app_test` target is for bring-up only — see below).
4. Build (F7).
5. Connect a J-Link to the SWD header on the controller PCB.
6. Flash & run (F8 / Download).

Because the H750 boots from external QSPI, the linker / Keil project is configured to program the QSPI flash via the MCU's flash loader. If the download fails with a "QSPI flash loader" error, double-check:
- The QSPI chip on your H750 module is at least 8 MB / 64 Mbit.
- Power to the QSPI VCC line is clean (a brown-out during programming corrupts the loader).

### `app_test` target

The `RTE/_app_test/` configuration is a smaller diagnostic build for bringing up a single subsystem (servo bus, IMU, RC receiver) without the full leg/gait stack. Use it when something on a freshly-soldered board doesn't respond.

> TODO (maker): document exactly what `app_test` exercises and how to interpret its output.

## Pin assignments

Pin assignments are defined in `SourceCode/MDK-ARM/USER/APP/bsp.h` and `SourceCode/Core/Inc/{gpio,i2c,usart}.h`. They are the single source of truth — if the table below disagrees with the headers, **the headers win.**

> TODO (maker): fill in the table from `bsp.h`. Suggested rows:
>
> | Function | Peripheral | Pin(s) | Notes |
> |---|---|---|---|
> | Servo bus TX/RX (LX-224) | UART_? | PA?/PA? | Half-duplex, single-wire to the daisy chain |
> | RC receiver | UART_? | PA?/PA? | Inverted? |
> | MPU6050 | I2C? | PB?/PB? | INT pin to PB? |
> | Status LED | GPIO | PC? | |
> | Debug UART | UART_? | PA?/PA? | 115200 8N1 |

## RC receiver

> TODO (maker): confirm and replace this section with the actual protocol parsed by `remote.c`.
>
> The maker's project description mentions **Spektrum (DX6e)** and **FrSky (X8R)** transmitters, which use different receiver-side wire protocols (DSM2/DSMX vs SBUS/FPort). The firmware decodes one of them today; document which, on which UART, at which baud, and whether the line needs hardware inversion. Channel-to-function mapping (throttle / yaw / mode switch / etc.) belongs here too.

## First-run checklist

1. Assign IDs **1 through 18** to the LX-224 servos one at a time using the HiWonder/Lobot servo programmer **before** chaining them — once they're on the same bus they all answer to ID conflicts at once.
2. Power the logic side from USB/SWD first, leave the servo rail off, and confirm the IMU comes up over the debug UART.
3. Re-power with the servo rail on; the legs should home to a neutral stance.
4. Bind your RC receiver per its own manual, then power the hexapod and verify channels are read on the debug UART before letting it walk.

## Troubleshooting

- **Flash download fails / "no flash loader"** — your H750 module's QSPI is too small or not wired to the standard pins. Confirm ≥ 64 Mbit and the QSPI pin map matches the WeAct reference.
- **Servos twitch / don't respond** — bus collision from duplicate IDs, or the half-duplex direction-control line isn't being toggled. Try one servo at a time on the bus first.
- **IMU never initialises** — DMP firmware upload failure. Power-cycle; check I²C pull-ups are populated; verify the MPU6050 I²C address matches what's set in `mpu6050.h`.
- **No RC channels seen** — wrong receiver protocol for what `remote.c` expects, or the UART line needs hardware inversion (SBUS) and isn't getting it.

## Contributing

Issues and PRs welcome — especially:
- A `bsp.h` → README pin table extraction.
- An STM32CubeIDE / CMake port.
- More gait modes in `gait_prg.cpp`.

## License

> TODO (maker): add a `LICENSE` file. CC-BY-NC 4.0 is reasonable for the CAD/PCB design files but isn't recommended for source code — consider a software license (MIT / GPL-3.0 / similar) for `SourceCode/`. Until a license is committed, all rights are reserved by default and others legally cannot reuse this code.

## Credits

- ST Microelectronics — STM32H7xx HAL & CMSIS (vendored under `SourceCode/Drivers/`).
- InvenSense — MPU6050 DMP motion driver (vendored under `SourceCode/MDK-ARM/USER/DMP/`).
- HiWonder/Lobot — LX-224 servo bus protocol.
- The Printables club at <https://www.printables.com/model/1443108> for build files and discussion.

## Build resources

- [Bill of Material](https://toolknox.github.io/HexaPod-RC-Controlled/bill-of-material.html)
