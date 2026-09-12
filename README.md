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
