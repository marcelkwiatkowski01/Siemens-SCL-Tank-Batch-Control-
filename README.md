# Tank Batch Control — TIA Portal + Factory I/O (SCL)

![TIA Portal](https://img.shields.io/badge/TIA%20Portal-V18%2B-0098A1)
![PLC](https://img.shields.io/badge/PLC-S7--1200%2F1500-B5121B)
![Language](https://img.shields.io/badge/language-SCL-2E8B57)
![Factory I/O](https://img.shields.io/badge/Factory%20I%2FO-Level%20Control-E67E22)

A PLC project written in **SCL** (Structured Control Language) for a Siemens S7‑1200/S7‑1500 controller, performing **analog** liquid‑level control in the **Level Control** scene of the **Factory I/O** simulator. The program runs a batch sequence — **fill → hold → drain** — repeated a set number of times, driven by a state machine.

Built as a coursework project for *Advanced PLC*, it deliberately exercises the full set of required SCL constructs (logic and arithmetic functions, edge detection, conditionals, `FOR`/`WHILE`/`CASE` loops, control statements, timers, counters and analog scaling). See *[SCL constructs & grading criteria](#scl-constructs--grading-criteria)* for the mapping.

## Contents

- [Overview](#overview)
- [The controlled process](#the-controlled-process)
- [Requirements](#requirements)
- [Repository structure](#repository-structure)
- [Getting started](#getting-started)
- [I/O table](#io-table)
- [Program architecture](#program-architecture)
- [How it works](#how-it-works)
- [Process parameters](#process-parameters)
- [SCL constructs & grading criteria](#scl-constructs--grading-criteria)
- [Author](#author)

## Overview

Key features of the program:

- **Complete batch cycle** — fill → hold → drain, automatically repeated `BATCH_TARGET` times, then back to idle.
- **State‑machine design** — the whole sequence is a `CASE` state machine with an explicit `ALARM`/safe state for unexpected conditions.
- **Proportional (P) valve control** — the analog fill valve is driven by a P‑controller toward an adjustable setpoint.
- **Filtered measurement** — the analog level is smoothed with a moving average over a circular sample buffer.
- **Modular analog scaling** — raw‑to‑engineering conversion (`NORM_X`/`SCALE_X`) is isolated in a reusable function with an input guard.
- **Online tuning** — the fill setpoint lives in the instance data block and can be changed at runtime without recompiling.
- **Safety supervision** — a watchdog timer guards the fill phase; a counter tracks completed batches.

## The controlled process

The object is a **vertical tank** fitted with an analog‑controlled **fill valve** (bottom), a **drain valve** and an analog **level meter**. An operator box on the frame carries **Start / Stop / Reset** push‑buttons with matching indicator lamps.

The plant is modeled in **Factory I/O**, and the link between the soft‑PLC and the simulator is provided by the **ComDrvS7** driver.

## Requirements

- **TIA Portal V18** or newer (the code uses S7‑1200/1500 instructions: `TON_TIME`, `CTU_INT`, `R_TRIG`, `NORM_X`, `SCALE_X`)
- **Factory I/O** with a valid license (edition supporting Siemens drivers)
- Controller: **S7‑1200** or **S7‑1500**

> Note: this repository contains the project files, not the software itself. You need your own TIA Portal and Factory I/O licenses to run it.

## Repository structure

```
.
├── README.md
├── tia/
│   └── TankBatchControl.zap18        # archived TIA project (use Retrieve to restore)
└── factoryio/
    └── TankLevelControl.factoryio    # saved scene including the I/O mapping

```

## Getting started

### 1. TIA project
1. In TIA Portal: **Project → Retrieve**, select `tia/TankBatchControl.zap18`, choose a target folder.
2. Compile the controller (right‑click the CPU → **Compile → Hardware and software**).
3. Download the program to the controller and set it to **RUN**.

### 2. Factory I/O
1. Open the scene `factoryio/TankLevelControl.factoryio` (**File → Open**). The scene already stores the Siemens driver and the I/O mapping.
2. In *Drivers → CONFIGURATION*, verify that analog encoding is **WORD** and the controller model matches the project.
3. Click **CONNECT** (green icon = connected), then start the scene with **RUN**.
4. Press **Start** on the panel.

> To rebuild the mapping manually (e.g. on a different scene), use `docs/io-table.md`. Addresses in TIA and in the Factory I/O driver must be identical.

## I/O table

| Factory I/O part | PLC address | Type | TIA tag | Direction |
|---|---|---|---|---|
| Start | %I0.0 | Bool | tagStart | input |
| Stop | %I0.1 | Bool | tagStop | input |
| Reset | %I0.2 | Bool | tagReset | input |
| Start Light | %Q0.0 | Bool | tagLampRun | output |
| Stop Light | %Q0.1 | Bool | tagLampStop | output |
| Reset Light | %Q0.2 | Bool | tagLampAlarm | output |
| Level Meter | %IW30 | Int (WORD) | tagLevelRaw | input |
| Fill Valve | %QW64 | Int (WORD) | tagFillValve | output |
| Discharge Valve | %QW66 | Int (WORD) | tagDrainValve | output |

## Program architecture

The program is built modularly from three blocks:

| Block | Type | Role |
|---|---|---|
| `OB1` (Main) | Organization block (SCL) | Cyclic entry point. Contains only the call to the function block, wiring the interface variables to the I/O tags. |
| `FbTankBatch` (+ `FbTankBatch_DB`) | Function block + instance DB | Holds the entire cycle logic and all state memory (state machine, filtered level, setpoint, batch count, sample buffer, timer/counter/edge instances). |
| `FcScaleLevel` | Function | Reusable analog scaling of the raw level value to engineering units (percent), with a divide‑by‑zero guard. |

The setpoint and other static values persist in `FbTankBatch_DB`, so the operating point (`statSetpoint`) can be adjusted online without recompiling.

## How it works

### State machine (`CASE`)

The cycle is a `CASE`‑based state machine (`statState`). The `ELSE` branch handles the alarm and any unexpected state by driving both valves to a safe position.

| State | Name | Description |
|---|---|---|
| 0 | IDLE | waiting for Start; valves closed |
| 1 | FILLING | proportional fill‑valve opening until `statSetpoint` is reached |
| 2 | HOLDING | hold/mix for `HOLD_TIME` |
| 3 | DRAINING | drain to the empty threshold + extra settle time (`DRAIN_HOLD`) |
| 4 | BATCH DONE | increment the counter, then decide: next batch or finish |
| 5 | ALARM | fill timeout (watchdog); valves forced to safe position |

### Analog scaling — `FcScaleLevel`

The function converts the raw transducer value into engineering units (level percent). It starts with an **input guard**: if `rawMax <= 0` it returns `0.0` and exits immediately via `RETURN`, protecting against a divide‑by‑zero and a bad configuration. The conversion itself uses `NORM_X` (normalize the raw value to `0.0…1.0`) followed by `SCALE_X` (scale to the engineering range). This `NORM_X`/`SCALE_X` pair is the required way of handling an analog variable.

### Proportional fill control

During `FILLING` the analog valve is driven by a P‑controller: the error `tempErr = statSetpoint − statLevelPct` is multiplied by the gain (`tempCmd = tempErr · KP`), clamped to the **50–100 %** range with an `IF…ELSIF…END_IF`, and finally converted back to a raw analog word for the valve. State transitions are likewise driven by `IF…ELSIF` (setpoint reached → HOLDING, fill timeout → ALARM).

### Measurement filtering (moving average)

Each cycle the scaled level is written into a **circular sample buffer**. A `FOR` loop then averages the buffer and illustrates two control statements: `CONTINUE` skips invalid (negative) samples without breaking the loop, while `EXIT` aborts the loop immediately on a corrupted value (above 200 %). The sum divided by the sample count yields the filtered level `statLevelPct`.

### Edge detection, timers & counter

- **Edge detection** — rising edges of the Start and Stop buttons are captured with `R_TRIG` instances, so a held button produces a single pulse.
- **Timers** — three `TON_TIME` instances: `instFillTon` (fill‑time watchdog), `instHoldTon` (hold time), and `instDrainTon` (drain settle; its `IN` combines the draining state with the empty‑threshold condition through a logical `AND`).
- **Counter** — a `CTU_INT` (`instBatchCtu`) counts completed batches and signals when `BATCH_TARGET` is reached.

### Stop / Reset handling

A supervisory section runs above the state machine: a **Stop** pulse zeroes the state machine, while **Reset** additionally clears the sample buffer. Buffer clearing is implemented with a `WHILE…DO` loop that iterates over every array cell until the terminating condition is met.

### Signaling (logic functions)

Lamp control is built from logic functions. The helper `tempRunning` is the logical **OR** of the working states 1, 2 and 3. The run lamp lights when `tempRunning AND NOT tempAllDone`, the stop lamp reflects state 0, and the alarm lamp reflects state 5 — using the `AND`, `OR` and `NOT` operators.

## Process parameters

Technological parameters are declared in the `Constant` section of `FbTankBatch` and can be tuned to the plant:

| Parameter | Value | Meaning |
|---|---|---|
| `KP` | 8.0 | proportional gain of the fill controller |
| `RAW_MAX` | 27648 | full‑scale analog value (Siemens) |
| `LEVEL_MAX` | 100.0 | upper engineering range of the level [%] |
| `SAMPLE_COUNT` | 5 | number of samples in the moving‑average filter |
| `FILL_TIMEOUT` | T#30s | fill watchdog time limit |
| `HOLD_TIME` | T#5s | hold / mix duration |
| `DRAIN_HOLD` | T#4s | drain settle (run‑out) time |
| `BATCH_TARGET` | 3 | number of batches per run |
| `EMPTY_LEVEL` | 2.0 | "tank empty" threshold [%] |
| `statSetpoint` | ~55 % (default, editable online) | target fill level |

## SCL constructs & grading criteria

| # | Criterion | Where in the code |
|---|---|---|
| 1 | Logic functions (`AND`/`OR`/`NOT`) | signaling section (lamp control) |
| 2 | Arithmetic functions | P‑controller and moving average |
| 3 | Edge detection (`R_TRIG`) | Start / Stop buttons |
| 4 | Conditionals `IF…ELSIF…ELSE` | valve clamp, state transitions, FC guard |
| 5 | Loops `FOR…BY…DO` / `CASE…ELSE` | averaging filter and the state machine |
| 6 | Loops `WHILE…DO` | sample‑buffer clearing on Reset |
| 7 | Control statements `CONTINUE` / `EXIT` / `RETURN` | filter loop and `FcScaleLevel` guard |
| 8 | Timers | 3× `TON_TIME` (fill watchdog, hold, drain settle) |
| 9 | Counters | `CTU_INT` — batch counter |
| 10 | Analog variable `NORM` & `SCALE` | `FcScaleLevel` (`NORM_X`/`SCALE_X`), analog input and outputs |

## Author

Marcel Kwiatkowski, Eng.
