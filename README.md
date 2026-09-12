# ⚡ LOW-POWER UART COMMUNICATION SYSTEM ON ARTIX-7 FPGA

[![Platform](https://img.shields.io/badge/Platform-Xilinx%20Artix--7%20FPGA-black.svg)](https://www.xilinx.com)
[![Tool](https://img.shields.io/badge/Tool-Xilinx%20Vivado-orange.svg)](https://www.xilinx.com/products/design-tools/vivado.html)
[![HDL](https://img.shields.io/badge/HDL-Verilog%20HDL-blue.svg)](#)
[![Dept](https://img.shields.io/badge/Dept-ECE%20%7C%20DBIT-orange.svg)](http://www.dbit.co.in)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> A synthesizable, low-power Universal Asynchronous Receiver-Transmitter (UART) protocol implementation on the Xilinx Artix-7 FPGA, featuring FSM-based TX/RX architecture, configurable baud-rate generation, and low-power clock-gating strategies.

---

## 🎓 Academic Information
- **Institution**: [Don Bosco Institute of Technology (DBIT)](http://www.dbit.co.in), Bengaluru - 560074
- **Department**: Electronics and Communication Engineering
- **Domain**: VLSI Design, Digital Circuit Design & RTL Prototyping
- **Author**: **Nandini K** (`USN: 1DB23EC101`)
- **Profile**: [LinkedIn](https://www.linkedin.com/in/nandini-k-99a804340/) | [Email](mailto:krish41105@gmail.com)

---

## 📌 Table of Contents
1. [Project Overview & Abstract](#-project-overview--abstract)
2. [Key Features](#-key-features)
3. [System Block Diagram & Architecture](#-system-block-diagram--architecture)
4. [Hardware & EDA Tool Specifications](#-hardware--eda-tool-specifications)
5. [Module Descriptions & Pin Mapping](#-module-descriptions--pin-mapping)
6. [Finite State Machine (FSM) Design](#-finite-state-machine-fsm-design)
7. [Repository Structure](#-repository-structure)
8. [Simulation, Synthesis & Power Results](#-simulation-synthesis--power-results)
9. [Step-by-Step Simulation & Execution](#-step-by-step-simulation--execution)
10. [References & Citations](#-references--citations)

---

## 📖 Project Overview & Abstract
UART (Universal Asynchronous Receiver-Transmitter) is an essential asynchronous serial protocol utilized in embedded platforms and SoC communication. Traditional UART architectures incur unnecessary dynamic power loss due to high-frequency free-running clock distribution networks.

This project delivers a **synthesizable, low-power UART Core** implemented in **Verilog HDL** and mapped to the **Xilinx Artix-7 FPGA**:
- **Dynamic Power Reduction**: Implemented clock-enabling logic that suspends module clocking during idle bus states.
- **Robust Sampling**: Employs 16× oversampling in the Receiver (RX) to combat phase mismatch and false start-bit noise.
- **Configurability**: Parameterized baud-rate generator capable of standard rates (9600 to 115200 bps) from an onboard 100 MHz oscillator.

---

## ✨ Key Features
- ⚡ **Low-Power Design Strategy**: Disables internal sampling counters when the line is idle.
- ⏱️ **Configurable Baud Rate**: Integer clock division logic with parameterizable modulo counters.
- 🛡️ **Noise Rejection**: Mid-bit voting filter during the 16× oversampled RX window.
- 🔄 **Independent TX & RX Engines**: Full-duplex asynchronous communication capability.
- 📊 **Vivado Validated**: 100% verified through RTL simulation, gate-level synthesis, and implementation reports.

---

## 📊 System Block Diagram & Architecture

```mermaid
flowchart TD
    subgraph Clocking["Clock Subsystem"]
        CLK_IN["100 MHz On-Board Oscillator"]
        BAUD_GEN["Baud Rate Generator\n(Baud Tick & 16x Tick)"]
        CLK_IN --> BAUD_GEN
    end

    subgraph Transmitter["UART TX Engine"]
        TX_FIFO["TX Data Register / FIFO"]
        TX_FSM["TX Controller (FSM)\n(IDLE -> START -> DATA -> STOP)"]
        TX_SHIFT["Parallel-to-Serial Shift Reg"]
        TX_OUT["UART_TXD (Serial Out)"]
        TX_FIFO --> TX_FSM
        TX_FSM --> TX_SHIFT
        TX_SHIFT --> TX_OUT
    end

    subgraph Receiver["UART RX Engine"]
        RX_IN["UART_RXD (Serial In)"]
        RX_SAMPLER["16x Mid-Bit Sampler & Filter"]
        RX_FSM["RX Controller (FSM)\n(START -> 8 DATA BITS -> STOP)"]
        RX_FIFO["RX Output Register / Buffer"]
        RX_IN --> RX_SAMPLER
        RX_SAMPLER --> RX_FSM
        RX_FSM --> RX_FIFO
    end

    BAUD_GEN -->|Baud Tick| Transmitter
    BAUD_GEN -->|16x Oversample Tick| Receiver


Hardware & EDA Tool Specifications
Parameter	Specification
Target Device	Xilinx Artix-7 FPGA (xc7a35tcpg236-1 / Basys 3)
EDA Tool	Xilinx Vivado Design Suite (v2020.2 or later)
HDL Standard	Verilog-2001
System Clock	100 MHz
Default Baud Rate	9600 bps / 115200 bps
Data Frame	1 Start Bit, 8 Data Bits, 1 Stop Bit, No Parity (8-N-1)

🔌 Module Descriptions & Pin Mapping
Port Name	Direction	Artix-7 Pin	Description
clk	Input	W5	100 MHz Master System Clock
reset	Input	V17	Active-High Asynchronous System Reset
rx	Input	B18	Serial Data Input from External Host/PC
tx	Output	A18	Serial Data Output to External Host/PC
tx_start	Input	U18	Pushbutton trigger to initiate transmission
tx_data[7:0]	Input	V16 to V17	8-bit input switches for data payload
rx_data[7:0]	Output	U16 to V14	8 onboard LEDs reflecting received byte
rx_done	Output	L1	Interrupt LED indicating valid byte received
🔄 Finite State Machine (FSM) Design
Transmitter (TX) State Flow:
IDLE: Line stays HIGH (1'b1). Waits for tx_start.
START: Drives line LOW (1'b0) for 1 baud period.
DATA: Shifts out 8 data bits (LSB first) sequentially on each baud pulse.
STOP: Drives line HIGH (1'b1) for 1 baud period, then returns to IDLE.
📂 Repository Structure


├── rtl/
│   ├── uart_top.v            # Top-level integration module
│   ├── baud_rate_gen.v       # Modulo divider baud clock generator
│   ├── uart_tx.v             # Serializer & TX FSM
│   └── uart_rx.v             # Deserializer with 16x oversampler
├── tb/
│   ├── uart_top_tb.v         # Self-checking testbench
│   └── baud_rate_gen_tb.v    # Clock division testbench
├── constraints/
│   └── basys3_artix7.xdc     # Vivado XDC pin constraint file
├── sim/
│   └── waveforms/            # Waveform capture captures (.png)
└── docs/
    └── synthesis_report.pdf  # Power, timing, and resource utilization reports
📈 Simulation, Synthesis & Power Results
Functional Verification: Verified across corner cases (back-to-back frames, clock drift).
Resource Utilization (Artix-7):
Slice LUTs: < 1% utilization
Slice Registers (FFs): < 1% utilization
Global Clock Buffers (BUFG): 1
Power Analysis: Total On-Chip Power measured at < 0.085 W using the Vivado Report Power feature.
🚀 Step-by-Step Simulation & Execution
Clone the repository:
bash


git clone https://github.com/YOUR_USERNAME/low-power-uart-artix7.git
Open Xilinx Vivado:
Create a New Project 
→
→ Choose RTL Project.
Select part: xc7a35tcpg236-1 (or your Artix-7 board part).
Add Sources:
Add all files from /rtl as Design Sources.
Add /tb/uart_top_tb.v as Simulation Sources.
Add /constraints/basys3_artix7.xdc as Constraints.
Run Simulation:
Click Run Behavioral Simulation.
Observe UART TX output serialized data matching the test stimulus.
Synthesis & Bitstream:
Click Run Synthesis 
→
→ Run Implementation 
→
→ Generate Bitstream.
📚 References & Citations
Xilinx 7 Series FPGAs Configurable Logic Block User Guide (UG474).
Pong P. Chu, FPGA Prototyping by Verilog Examples: Xilinx Spartan-3 Version.



