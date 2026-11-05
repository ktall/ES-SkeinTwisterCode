# Firmware notes

First read, 2026-11-05. Version 2.01, ~700 lines of C++ (really C style), bare
AVR (no Arduino), Atmel Studio project. Nothing run or flashed yet.

## What it does

The knob does **not** set speed. The motor always ramps to full power. The
knob picks the mode and, in the middle, the **twist time**:

| Knob position | Mode | Behaviour |
|---|---|---|
| Fully left (ADC < cal value, ~15) | Press & hold | Runs while the pedal is held. No time limit |
| Middle | Automatic | Tap to start, stops after 1.2 to 7.8 s (knob), tap again to stop early |
| Fully right (ADC > cal value, ~1010) | Tap on / tap off | Tap to start, tap to stop, 30 s safety cutoff ("the cat pushed the foot pedal") |

Start is a soft ramp: PWM 5 to 50 in steps of 5, every 50 ms (~0.5 s). Tap
on/off mode blips the motor first to say "this one won't stop by itself".

## How it's built

- **Clock**: internal 8 MHz RC divided by 8, so 1 MHz (fuses in the header comment). Slow but plenty.
- **Timer0**: 1 ms tick, a home-made Arduino-style `millis()`.
- **Timer1**: motor PWM on the MOSFET gate (pin PB1). TOP = 50, so 0 to 50 = 0 to 100 %.
- **INT0**: foot pedal interrupt on falling edge, 75 ms debounce checked in the main loop.
- **ADC**: reads the knob pot, 10 bits.
- **UART, 9600 baud**: service menu on the serial connector (J5), only during the first 5 minutes after power-on: `c` calibrates the knob ends, `d` dumps usage stats, `e` erases the EEPROM.
- **EEPROM**: knob calibration + usage log (number of runs, total run time). Proper wear leveling: moves to a fresh block every 65 535 writes, below the 100 000 write endurance of a cell.
- **Prebuilt `Debug/SkeinTwister.hex`** is in the repo: we can flash the stock firmware without building anything.
- **Flashing**: AVRDude with a USBtiny programmer over ISP (J1, the Hirose DF13 connector). Fuses for both ATmega328P and 328PB are given, so the PB variant is a drop-in alternative if the P is hard to find.

## Quality

Small, readable, well commented, single main loop with a state machine. Classic
embedded style (globals, busy waits). Nothing to be ashamed of; easy to own.

## Bench plan: the existing unit as a dev board

It's the wife's machine: back everything up before touching it, and ask the
owner before any flash.

1. **Check it runs**: power on, LED blinks every 0.5 s = firmware alive. Try the three modes.
2. **Read it without changing anything**: USB-serial adapter (5 V, FTDI style) on J5, 9600 baud, send `d` (in the first 5 minutes): version + usage stats (number of runs, total time). Real usage data for free.
3. **Back up the chip**: ISP programmer on J1, AVRDude reads flash, EEPROM and fuses to files. Flash may be lock-protected; EEPROM and fuses should still read.
4. **Measure the limits**: current in a shunt or clamp meter on the motor line: running, tight skein, full stall. Tells us how far finding 1 really goes.
5. **Only then flash** our builds. The backup (or the repo's .hex) gets it back to stock.

To buy: ISP programmer (USBasp or USBtinyISP), USB-serial adapter, Hirose DF13 cables (6-pin for J1, 3-pin for J5; pinouts from the schematic).

## Findings

1. **No stall protection.** When the skein is tight the motor stalls and keeps drawing stall current until the timer ends: up to 7.8 s in automatic, 30 s in tap mode, unlimited in press & hold. Heat for the motor, the MOSFET and the power supply. Nothing on the board measures current, so firmware alone can't fix it.
2. **No watchdog.** If the firmware ever hangs with the motor on, nothing stops it. Cheap to add (a few lines), and it matters for the CE risk assessment.
3. **The power supply is the real torque limiter.** Carrie's note says a 5 A supply "might cut out early": most likely the supply's current limit sags the 12 V, the 5 V regulator drops out, the brown-out detector (4.7 V) resets the chip and the motor stops. Works, but by accident rather than by design.
4. **PWM timer prescaler is changed by `motor()`** (sets/clears CS10 on top of CS11 from init): running PWM is ~156 Hz instead of the 1.25 kHz the comments say, and "stops timer" doesn't stop it. Harmless (the motor runs at 100 % after the ramp), but comments and code disagree.
5. **Interrupts are off during EEPROM writes** (~3.4 ms per byte, 6 bytes per run): `millis()` loses ~20 ms each run. Harmless.
6. **Windows-only toolchain** (Atmel Studio). Porting to plain avr-gcc + Makefile, or PlatformIO, would let us build on the Mac.
