# Tank Batch Control — TIA Portal + Factory I/O (SCL)

![TIA Portal](https://img.shields.io/badge/TIA%20Portal-V15.1%2B-0098A1)
![PLC](https://img.shields.io/badge/PLC-S7--1200%2F1500-B5121B)
![Language](https://img.shields.io/badge/language-SCL-2E8B57)
![Factory I/O](https://img.shields.io/badge/Factory%20I%2FO-Level%20Control-E67E22)

A PLC project written in **SCL** (Structured Control Language) for a Siemens S7‑1200/S7‑1500 controller, performing **analog** liquid‑level control in the **Level Control** scene of the **Factory I/O** simulator. The program runs a batch sequence — fill → hold → drain — repeated a set number of times, driven by a state machine.

Built as a coursework project for *Advanced PLC*, it demonstrates a full set of required SCL constructs (see *Grading criteria*).

## Requirements

- **TIA Portal V15.1** or newer (the code uses S7‑1200/1500 instructions: `TON_TIME`, `CTU_INT`, `R_TRIG`, `NORM_X`, `SCALE_X`)
- **Factory I/O** with a valid license (edition supporting Siemens drivers)
- Controller: **S7‑1200** or **S7‑1500**

> Note: this repository contains the project files, not the software itself. You need your own TIA Portal and Factory I/O licenses to run it.

## Repository structure

```
.
├── README.md
├── tia/
│   ├── TankBatchControl.zap15_1     # archived TIA project (use Retrieve to restore)
│   └── src/
│       └── TankBatch.scl            # SCL source (FB + FC) — readable / importable
└── factoryio/
    └── TankLevelControl.factoryio   # saved scene including the I/O mapping

```

## Getting started

### 1. TIA project
1. In TIA Portal: **Project → Retrieve**, select `tia/TankBatchControl.zap15_1`, choose a target folder.
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
| Level Meter | %IW64 | Int (WORD) | tagLevelRaw | input |
| Fill Valve | %QW64 | Int (WORD) | tagFillValve | output |
| Discharge Valve | %QW66 | Int (WORD) | tagDrainValve | output |

## How it works

The program runs as a state machine (`statState`):

| State | Name | Description |
|---|---|---|
| 0 | IDLE | waiting for Start |
| 1 | FILLING | proportional fill‑valve opening until `statSetpoint` is reached |
| 2 | HOLDING | hold/mix for `HOLD_TIME` |
| 3 | DRAINING | drain to the empty threshold + extra drain time (`DRAIN_HOLD`) |
| 4 | DONE | increment counter, decide: next batch or finish |
| 5 | ALARM | fill timeout (watchdog) |

The level is read from an analog meter, scaled with `NORM_X`/`SCALE_X`, and filtered with a moving average. The sequence repeats `BATCH_TARGET` times, then returns to IDLE. Key parameters (`statSetpoint`, timers, batch count) can be tuned in the `FbTankBatch` block.

## Grading criteria

| # | Criterion | Where in the code |
|---|---|---|
| 1 | Logic functions | section 8: `AND`/`OR`/`NOT` on the lamps |
| 2 | Arithmetic | P‑controller and moving average |
| 3 | IF/ELSE conditionals | valve clamp, state transitions, FC guard |
| 4 | Loops FOR / CASE / WHILE | filter, state machine, buffer clearing |
| 5 | Timers + CONTINUE/EXIT/RETURN | 3× `TON_TIME`, FOR loop, `FcScaleLevel` |
| 6 | Counters | `CTU_INT` — batch counter |
| 7 | Analog signals | `NORM_X`/`SCALE_X`, analog input and outputs |
| 8 | Edge detection | 2× `R_TRIG` on the buttons |

## Author

Marcel Kwiatkowski, Eng.
