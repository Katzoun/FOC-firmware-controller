# Project notes: hoverboard mainboard as a UGV drive

Working notes for this fork of hoverboard-firmware-hack-FOC. The goal is to use the mainboard and motors from a disassembled hoverboard as the drive of a UGV.

Status as of 2026-11-12. Everything about firmware behaviour below was read from the code and has not been verified on the board yet (the firmware has not been flashed).

## Hardware

- Hoverboard mainboard with a single MCU, **GD32F103RCT6**.
- Both motors connected (the left one on an extended cable).
- ST-Link programmer on the SWD header (SWDIO, SWCLK, GND; do not connect the 3.3 V pin).
- Powered from the hoverboard battery (10S).

## Current state

- The chip is **unlocked** (Read Out Protection disabled with STM32 ST-LINK Utility). The original firmware is erased.
- The firmware is built as `VARIANT_USART` with no configuration changes: `.pio\build\VARIANT_USART\firmware.bin` (about 29 kB).
- The firmware is **not flashed yet**.
- The device that will send commands is not decided yet (see "Open items").

## Building

PlatformIO is installed as a VS Code extension; it downloaded the toolchain on the first build.

- In VS Code: PlatformIO → environment `VARIANT_USART` → Build.
- From the command line: `%USERPROFILE%\.platformio\penv\Scripts\pio.exe run -e VARIANT_USART`

There is no need to uncomment a variant in `platformio.ini` or `Inc/config.h`; selecting the environment is enough.

## Flashing with STM32 ST-LINK Utility

Keep the power button pressed the whole time (or bridge it). With no firmware running, the board powers off as soon as the button is released. Keep the wheels off the ground.

1. **Target → Program & Verify…** (Ctrl+P).
2. File path: `firmware.bin` from the path above.
3. Start address: `0x08000000`.
4. Tick **Verify after programming** and **Reset after programming**, then **Start**.
5. The log should end with `Verification...OK`.
6. **Target → Disconnect**, release the button, then power the board on again with a short press.

If the connection fails: in Target → Settings lower the SWD frequency (480 kHz or less) and select Connect Under Reset.

## What the firmware does after power-on

1. Waits 0.5 s, then latches its own power and plays the startup melody (`Src/main.c`, `STARTUP_DELAY_MS`, `poweronMelody()`).
2. Waits until the power button is released.
3. Two short beeps: motors enabled (the input command is zero).
4. If nothing arrives on the serial line: three low beeps repeating, commands zeroed, motors in `OPEN_MODE` with no power (`Src/util.c`, "In case of timeout bring the system to a Safe State").

It behaves the same way while driving if commands stop arriving for 0.8 s (`SERIAL_TIMEOUT 160`).

## Default control settings (`Inc/config.h`)

| Setting | Value | Meaning |
|---|---|---|
| `CTRL_TYP_SEL` | `FOC_CTRL` | field oriented control |
| `CTRL_MOD_REQ` | `VLT_MODE` | the command sets motor voltage, not speed |
| `I_MOT_MAX` | 15 A | current limit per motor |
| `N_MOT_MAX` | 1000 rpm | speed limit |
| `FIELD_WEAK_ENA` | 0 | field weakening off |
| `DIAG_ENA` | 1 | motor diagnostics on |
| `INACTIVITY_TIMEOUT` | 8 min | auto power-off when idle |

- **Both motors must be connected.** A hall sensor error on one motor stops both (`Src/bldc.c`, `enableFin`). The only way around it is disabling diagnostics (`DIAG_ENA 0`), which is acceptable for a bench test at most.
- For the UGV, `SPD_MODE` (holds speed under load) and `TANK_STEERING` (each input drives one wheel) will probably fit better.
- Hall sensor odometry is enabled with `ENABLE_ODOMETRY`; it adds the `wheelR_cnt` and `wheelL_cnt` counters to the feedback frame.

## Serial control

- In `VARIANT_USART` the default is **USART2 = the left sensor board cable** (`CONTROL_SERIAL_USART2`, `FEEDBACK_SERIAL_USART2`). This port is **not 5 V tolerant**.
- The right cable (USART3) is 5 V tolerant. To switch: in the `VARIANT_USART` section of `Inc/config.h` comment out the USART2 pair, uncomment the USART3 pair, and rebuild.
- According to the upstream project documentation, each sensor board cable carries a wire at about 15 V. This is not verified in this repository: measure the wires before connecting anything and leave the 15 V wire unconnected.
- Line settings: 115200 baud, 8N1.
- Command frame: 8 bytes, four 16-bit little-endian values: start `0xABCD`, `steer`, `speed`, checksum (XOR of the previous three). `steer` and `speed` range from −1000 to 1000.
- Mixing: `speed` has weight 1.0, `steer` 0.5. Commands pass through a rate limiter and a low-pass filter.
- Feedback: every 10 ms the board sends `cmd1`, `cmd2`, right and left wheel speed, battery voltage, board temperature and LED state.
- Example counterpart: `Arduino/hoverserial/hoverserial.ino`.

## Power button and beeps

| Signal | Meaning |
|---|---|
| short button press | power off |
| press longer than 5 s | input calibration (not needed for serial control) |
| 1 low beep repeating | motor error, typically hall sensors |
| 2 low beeps repeating | ADC input timeout |
| 3 low beeps repeating | serial timeout (no commands arriving) |
| 4 low beeps repeating | general timeout (PPM, PWM, Nunchuk) |
| fast medium-pitch beeping | low battery (below 3.5 V per cell) |
| fast high-pitch beeping | driving backward |

Below 3.37 V per cell the board powers off when standing still. Battery voltage measurement is not calibrated for this board (`BAT_CALIB_REAL_VOLTAGE`, `BAT_CALIB_ADC`). Temperature protection is off; it requires calibration.

## Open items

- Decide what will send the commands (a 3.3 V device on the left cable, or a 5 V Arduino on the right one) and adjust the config if needed.
- Flash the firmware and verify the power-on behaviour.
- First motor spin, check rotation direction (`INVERT_L_DIRECTION`, `INVERT_R_DIRECTION`).
- Calibrate battery voltage measurement.
- Later: `SPD_MODE`, `TANK_STEERING`, `ENABLE_ODOMETRY`.
