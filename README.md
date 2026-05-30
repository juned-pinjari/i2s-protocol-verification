# Circuit-Level Modelling and Verification of I2S Protocol with Gate-Level SIPO Receiver

Published on FOSSEE eSim Circuit Simulation Repository, IIT Bombay  
[View on FOSSEE](https://esim.fossee.in/circuit-simulation-project/esim-circuit-simulation-run/766)  
**Contributor:** Juned Pinjari | Government College of Engineering, Nagpur  
**Tools:** eSim 2.5 · Ngspice · KiCad 8.0

---

## Overview

This project implements and verifies the I2S (Inter-IC Sound) serial audio protocol at the circuit level using a mixed-signal SPICE simulation. The I2S protocol (developed by Philips Semiconductors) is the standard interface used in audio ICs, DACs, ADCs, and DSPs across the semiconductor industry.

The simulation models a complete stereo audio receiver:
- 8-bit SIPO (Serial-In Parallel-Out) shift register built from XSPICE `d_dff` primitives
- PWL-based transmitter generating Left and Right channel audio frames
- Full Philips I2S specification compliance including WS-framing and MSB-first bit ordering

---

## Verification Results

| Parameter | Requirement | Observed | Status |
|---|---|---|---|
| SCK Frequency | 1 MHz | 1 MHz | Pass |
| WS Frequency | 62.5 kHz | 62.5 kHz | Pass |
| Left Channel Payload | `10101010` | `10101010` | Pass |
| Right Channel Payload | `11001100` | `11001100` | Pass |
| Gate-Level Propagation Delay | < 10 ns | 3.65 ns | Pass |

---

## Circuit Architecture

### Schematic — Logical Architecture (KiCad 8.0)

![Schematic](docs/schematic_logical_architecture.png)

Three `adc_bridge_1` converters translate SCK, WS, and SD into the digital domain. Eight cascaded `d_dff` primitives form the SIPO shift register (U4–U11). Eight `dac_bridge_1` converters translate digital outputs back to analog for Ngspice plotting.

### Signal Roles

| Signal | Description |
|---|---|
| SCK | Master Serial Clock — 1 MHz PULSE, 5 V, 50% duty cycle |
| WS | Word Select (LR Clock) — 62.5 kHz PULSE, WS=0: Left Ch, WS=1: Right Ch |
| SD | Serial Data — PWL source encoding `10101010` (Left) and `11001100` (Right) |
| SCK_DIG / WS_DIG / SD_DIG | Digital-domain equivalents via `adc_bridge_1` converters |
| OUT_0 – OUT_7 | Parallel SIPO outputs (MSB = OUT_7, LSB = OUT_0) |
| VOUT_0 – VOUT_7 | Analog-domain equivalents via `dac_bridge_1` for Ngspice plotting |

### Design Decisions

**Why a hybrid PWL + XSPICE testbench?**  
Hardware latching using WS-derived gated clocks causes XSPICE convergence failures in mixed-signal Ngspice. Channel separation is instead validated by temporal sampling of SIPO outputs at WS frame boundaries — functionally equivalent and convergence-safe.

**Why manually authored `.cir` instead of KiCad netlist export?**  
KiCad 8.0 strips XSPICE symbols (`adc_bridge_1`, `d_dff`) during netlist export due to node-validation rule changes. The KiCad schematic serves as the logical architecture diagram; the `.cir` testbench is hand-authored with XSPICE instances injected directly.

---

## Waveforms

### Input Stimulus and I2S Timing Alignment
WS transitions exactly one SCK cycle before the MSB — per Philips I2S specification.

![Input Stimulus](results/waveforms/fig2_input_stimulus.png)

### Full 16-bit Stereo Frame (0–20 µs)
Left Channel (WS=0, t=0–8 µs) and Right Channel (WS=1, t=8–16 µs).

![Full Frame](results/waveforms/fig3_full_frame.png)

### Left Channel Data Extraction (at t = 8.0 µs)
SIPO parallel outputs stable at `10101010` at the WS=0 window close.

![Left Channel](results/waveforms/fig4_left_channel.png)

### Right Channel Data Extraction (at t = 16.0 µs)
SIPO parallel outputs stable at `11001100` at the WS=1 window close.

![Right Channel](results/waveforms/fig5_right_channel.png)

### Gate-Level Propagation Delay: 3.65 ns
Measured from 50% threshold of SCK rising edge to VOUT_0 output transition.

![Propagation Delay](results/waveforms/fig6_propagation_delay.png)

---

## Repository Structure

```
i2s-protocol-verification/
├── simulation/
│   ├── I2S_Protocol_Simulation_tb.cir     <- Runnable Ngspice testbench (use this to simulate)
│   ├── I2S_Protocol_Simulation.cir        <- eSim-generated stub (for reference)
│   ├── I2S_Protocol_Simulation.kicad_sch  <- Logical architecture schematic (KiCad 8.0)
│   ├── I2S_Protocol_Simulation.net        <- KiCad netlist export (partial, see note below)
│   └── I2S_Protocol_Simulation.proj       <- eSim project file
├── results/
│   ├── plot_data_v.txt                    <- Voltage data from Ngspice (print allv)
│   ├── plot_data_i.txt                    <- Current data from Ngspice (print alli)
│   └── waveforms/                         <- eSim Python plots from Ngspice simulation
│       ├── fig2_input_stimulus.png
│       ├── fig3_full_frame.png
│       ├── fig4_left_channel.png
│       ├── fig5_right_channel.png
│       └── fig6_propagation_delay.png
└── docs/
    ├── schematic_logical_architecture.png
    └── I2S_Abstract_Juned_Pinjari.pdf
```

**Note on `.net` file:** The KiCad-exported netlist only contains R and PULSE source components. XSPICE primitives (`d_dff`, `adc_bridge_1`, `dac_bridge_1`) are stripped by KiCad 8.0's node-validation rules. The complete simulation netlist is in `I2S_Protocol_Simulation_tb.cir`.

---

## How to Run

**Requirements:** eSim 2.5 with Ngspice backend  
Ubuntu 24.04+ users: see [this port](https://github.com/juned-pinjari/eSim) for GCC 14 / Python 3.13 compatibility fixes.

```bash
# Run directly with Ngspice
ngspice simulation/I2S_Protocol_Simulation_tb.cir

# Or via eSim GUI:
# File -> Open Project -> simulation/I2S_Protocol_Simulation.proj
# Run Ngspice -> select I2S_Protocol_Simulation_tb.cir
```

After simulation, `plot_data_v.txt` and `plot_data_i.txt` are generated in the working directory. Use GTKWave or matplotlib to plot signals.

---

## References

1. NXP Semiconductors. UM11732 — I2S Bus Specification, Rev. 3.0, February 2022. https://www.nxp.com/docs/en/user-manual/UM11732.pdf
2. FOSSEE Team, IIT Bombay. eSim User Manual v2.5. https://esim.fossee.in
3. Wikipedia. Inter-IC Sound (I2S). https://en.wikipedia.org/wiki/I2S

---

## About

Developed as part of the [FOSSEE Circuit Simulation Project](https://esim.fossee.in/circuit-simulation-project), IIT Bombay. Demonstrates mixed-signal protocol verification using open-source EDA tools.

**Skills demonstrated:** SPICE netlist authoring · XSPICE mixed-signal simulation · Digital protocol verification · eSim/KiCad toolchain · Ngspice waveform analysis

---

## License

Copyright transferred to the FOSSEE Project, IIT Bombay.  
Released under [Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/).  
Original contributor: Juned Pinjari, Government College of Engineering, Nagpur.
