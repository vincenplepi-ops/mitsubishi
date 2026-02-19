# Mitsubishi Ladder Logic Quick Start

This note provides a compact starter reference for Mitsubishi PLC ladder programming (GX Works / MELSEC style).

## 1) Basic Ladder Elements

- **NO contact** (`| |`) is true when the addressed bit is ON.
- **NC contact** (`|/|`) is true when the addressed bit is OFF.
- **Coil** (`( )`) writes ON/OFF to a bit output.
- **SET/RST** latch and unlatch a bit.
- **Timers/Counters** provide delay and count-based control.

Common device prefixes:

- `X` = physical inputs
- `Y` = physical outputs
- `M` = internal relays (memory bits)
- `D` = data registers
- `T` = timers
- `C` = counters

## 2) Typical Start/Stop Motor Circuit

Classic self-hold rung:

1. `X0` = START (momentary NO)
2. `X1` = STOP (momentary NC wiring, often represented as NC condition in logic)
3. `Y0` = MOTOR output

Logic intention:

- Press START (`X0`) -> turn on `Y0`
- `Y0` contact seals itself in parallel with START
- Press STOP (`X1` condition breaks) -> `Y0` turns off

## 3) Interlock Pattern

When two outputs must never be ON together (e.g., forward/reverse):

- Forward coil rung includes NC condition of reverse output.
- Reverse coil rung includes NC condition of forward output.

This creates software interlocking.

## 4) Timer Example (On-Delay)

- Use input `X2` to enable timer `T0` with preset.
- When elapsed, use `T0` done contact to energize `Y1`.

Useful for delayed starts, debounce, and sequencing.

## 5) Good Practices

- Use meaningful labels/comments for each device.
- Separate **I/O mapping**, **logic**, and **alarms** by section.
- Add interlocks in both hardware (where needed) and software.
- Use simulation/monitor mode before commissioning.
- Keep an I/O list and revision history with every change.

## 6) Safety Reminder

PLC software examples are educational only. Validate all logic against machine safety requirements, applicable standards, and your site lockout/commissioning procedures.
