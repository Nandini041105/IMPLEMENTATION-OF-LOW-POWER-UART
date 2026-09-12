# ⚡ DESIGN AND IMPLEMENTATION OF A LOW-POWER UART ON ARTIX-7 FPGA USING VERILOG

[![FPGA](https://img.shields.io/badge/FPGA-AMD%2FXilinx%20Artix--7%20(xc7a35t)-black.svg)](https://www.xilinx.com)
[![Tool](https://img.shields.io/badge/EDA-Xilinx%20Vivado%20ML-orange.svg)](https://www.xilinx.com/products/design-tools/vivado.html)
[![Dynamic Power](https://img.shields.io/badge/Dynamic%20Power-%E2%86%93%2050.00%25-brightgreen.svg)](#)
[![Area Savings](https://img.shields.io/badge/Slice%20LUTs-%E2%86%93%2042.86%25-success.svg)](#)
[![Timing Closure](https://img.shields.io/badge/Timing%20Closure-WNS%20%2B6.301%20ns-informational.svg)](#)
[![Dept](https://img.shields.io/badge/Dept-ECE%20%7C%20DBIT-orange.svg)](http://www.dbit.co.in)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> A synthesizable, energy-efficient Universal Asynchronous Receiver-Transmitter (UART) IP core engineered for the AMD/Xilinx Artix-7 28nm FPGA family (`xc7a35tcsg324-1`). Demonstrates a **50.00% reduction in dynamic power dissipation** ($4\text{ mW} \rightarrow 2\text{ mW}$) and a **42.86% reduction in Look-Up Table (LUT) area** through FPGA-safe Clock Enable (CE) gating, activity-gated counter freezes, minimal parameter widths (`$clog2`), and Gray-coded low-Hamming state machines.

---

## 🎓 Academic Information
- **Institution**: [Don Bosco Institute of Technology (DBIT)](http://www.dbit.co.in), Mysore Road, Bengaluru - 560074
- **Department**: Electronics and Communication Engineering (ECE)
- **Project Domain**: Digital VLSI Design, Low-Power RTL Design & FPGA Prototyping
- **Author**: **Nandini K** (`USN: 1DB23EC101`)
- **Connect**: [LinkedIn](https://www.linkedin.com/in/nandini-k-99a804340/) | [Email](mailto:krish41105@gmail.com)

---

## 📌 Table of Contents
1. [Executive Summary & Key Achievements](#-executive-summary--key-achievements)
2. [Theoretical Background & Power Physics](#-theoretical-background--power-physics)
3. [System Block Diagram & Architecture](#-system-block-diagram--architecture)
4. [Low-Power Architectural Strategies](#-low-power-architectural-strategies)
5. [Submodule Engineering & Math Formulations](#-submodule-engineering--math-formulations)
6. [Vivado Behavioral Simulation & Waveform Verification](#-vivado-behavioral-simulation--waveform-verification)
7. [Physical Implementation & Vivado Experimental Results](#-physical-implementation--vivado-experimental-results)
8. [Benchmarking Against Published Literature (IJCESEN 2025)](#-benchmarking-against-published-literature-ijcesen-2025)
9. [Hardware Demonstration & Testing Guide](#-hardware-demonstration--testing-guide)
10. [Artix-7 FPGA Pin Mapping (XDC Constraints)](#-artix-7-fpga-pin-mapping-xdc-constraints)
11. [Repository Directory Structure](#-repository-directory-structure)
12. [Step-by-Step Vivado Execution Guide](#-step-by-step-vivado-execution-guide)
13. [References & Citations](#-references--citations)

---

## 🏆 Executive Summary & Key Achievements

| Metric / Parameter | Baseline Architecture | Proposed Low-Power UART | Verified Improvement |
| :--- | :---: | :---: | :---: |
| **Dynamic Power ($P_{\text{dynamic}}$)** | $0.004\text{ W}$ ($4\text{ mW}$) | **`0.002 W` ($2\text{ mW}$)** | 🚀 **`50.00%` Reduction** |
| **Slice LUT Utilization** | 98 LUTs | **56 LUTs** | 🏆 **`42.86%` Area Savings** |
| **Slice Registers (FFs)** | 100 FFs | **76 FFs** | 🏆 **`24.00%` Register Savings** |
| **Total Occupied Slices** | 46 Slices | **31 Slices** | 🏆 **`32.61%` Slice Savings** |
| **Switching Activity ($\alpha$)** | 4,121 bit toggles | **2,700 bit toggles** | ⚡ **`34.48%` Activity Drop** |
| **Worst Negative Slack (WNS)** | $+4.500\text{ ns}$ | **`+6.301 ns`** | ⏱️ **`+1.801 ns` Timing Margin** |
| **Max Operating Freq ($f_{\text{max}}$)**| $181.82\text{ MHz}$ | **`270.34 MHz`** | 📈 **`48.69%` Faster Core** |
| **vs. Published 2025 Paper** | $1.763\text{ W}$ | **`0.002 W`** | 🌟 **`99.89%` Lower Power** |

---

## 🔬 Theoretical Background & Power Physics

Total power dissipation in CMOS FPGA fabrics is governed by:

$$P_{\text{total}} = P_{\text{static}} + P_{\text{dynamic}}$$

$$P_{\text{dynamic}} = \sum_{i \in \text{nets}} \alpha_i \cdot C_i \cdot V_{\text{core}}^2 \cdot f_{\text{clk}}$$

- **Core Voltage ($V_{\text{core}} = 1.0\text{ V}$)** and **Clock Frequency ($f_{\text{clk}} = 100\text{ MHz}$)** are physically fixed by the Artix-7 fabric.
- **Dynamic Power Minimization Strategy:**
  1. **Lowering Switching Activity ($\alpha \downarrow$):** Achieved by completely stopping and freezing internal baud and bit counters at zero during transmission idle windows.
  2. **Lowering Effective Capacitance ($C \downarrow$):** Achieved by truncating bloated 32-bit registers down to minimal `$clog2()` parameterized widths (saving 42.86% of the routing capacitance and Look-Up Tables).

---

## 📊 System Block Diagram & Architecture

```mermaid
flowchart TD
    subgraph Clocking["100 MHz Master Timing Subsystem"]
        CLK_IN["Master Oscillator (100 MHz - Pin E3)"]
        RST_IN["Sync Reset (Pin D9)"]
    end

    subgraph Watchdog["Low-Power Supervisory Controller"]
        LP_CTRL["low_power_control.v\n• Monitors: tx_start | tx_busy | rx_busy\n• Instantaneous Wakeup (Cycle 0)\n• Auto-Sleep on Timeout"]
        LED_SLEEP["low_power_active (LED H5)\n[ON = System in Sleep]"]
        LP_CTRL --> LED_SLEEP
    end

    subgraph ClockDivider["Baud Rate Generator Subsystem"]
        BAUD_GEN["baud_generator.v\n• 1x Divisor: 868 cycles (10-bit [9:0])\n• 16x Divisor: 54 cycles (6-bit [5:0])\n• Counter FROZEN at 0 when Idle"]
        TICK_1X["baud_tick_1x (115.2 kbps)"]
        TICK_16X["baud_tick_16x (Oversampling)"]
        BAUD_GEN --> TICK_1X
        BAUD_GEN --> TICK_16X
    end

    subgraph TransmitPath["UART Transmitter Subsystem"]
        TX_MOD["uart_tx.v\n• 8-N-1 Serializer\n• Gray-Coded FSM (Hamming Dist = 1)\n• Clock-Enable Driven Shift Register"]
        TX_PIN["UART TX (Pin D10)"]
        TX_BUSY_LED["tx_busy (LED J5)"]
        TX_DONE_LED["tx_done (LED E1)"]
        TX_MOD --> TX_PIN
        TX_MOD --> TX_BUSY_LED
        TX_MOD --> TX_DONE_LED
    end

    subgraph ReceivePath["UART Receiver Subsystem"]
        RX_PIN["UART RX (Pin A9)"]
        RX_MOD["uart_rx.v\n• 2-Stage Anti-Metastability Sync\n• False-Start Glitch Filter\n• Mid-Bit 16x Center-Point Sampler"]
        RX_VALID_LED["rx_valid (LED T9)"]
        RX_ERR_LED["rx_error (LED T10)"]
        RX_PIN --> RX_MOD
        RX_MOD --> RX_VALID_LED
        RX_MOD --> RX_ERR_LED
    end

    CLK_IN --> BAUD_GEN & TX_MOD & RX_MOD & LP_CTRL
    RST_IN --> BAUD_GEN & TX_MOD & RX_MOD & LP_CTRL

    LP_CTRL -->|baud_gen_enable| BAUD_GEN
    TICK_1X --> TX_MOD
    TICK_16X --> RX_MOD
    RX_MOD -.->|Hardware Echo (loopback_mode = 1)| TX_MOD
```

---

## ⚡ Low-Power Architectural Strategies

### 1. Clock Enable (CE) Gating (FPGA-Safe)
In ASIC design, clock gating is achieved via combinational `AND` gates. On FPGAs, this causes severe clock skew, routing delays, and hold-time violations. Instead, our design routes global clocks strictly through a dedicated low-skew `BUFG` tree and connects enable signals to the native **Clock Enable (`CE`)** pins of Artix-7 `FDRE` slice flip-flops.

### 2. Activity-Gated Idle Counter Freezing
In conventional designs, baud counters run continuously 100,000,000 times per second even when no data is transferred. In our core:
```verilog
// baud_generator.v: Counter Freeze Logic
if (enable) begin
    count_1x <= (count_1x == DIVISOR_1X - 1) ? 0 : count_1x + 1;
end else begin
    count_1x <= 10'd0; // FROZEN AT ZERO: Eliminates 100% idle switching!
end
```

### 3. Gray-Coded FSM State Encoding
State transitions utilize a Gray-code sequence with a **Hamming distance of 1**, ensuring exactly 1 bit changes per state transition to suppress glitch power in the combinational LUT network:
```text
[IDLE: 2'b00] ──(tx_start)──> [START: 2'b01] ──(baud_tick)──> [DATA: 2'b11] ──(baud_tick)──> [STOP: 2'b10] ──> [IDLE: 2'b00]
```

---

## 📐 Submodule Engineering & Math Formulations

### 1. Baud Rate Generator (`rtl/baud_generator.v`)
- **Transmitter Divisor ($1\times$ Baud Tick at 115,200 bps):**
  $$M_{1\text{x}} = \frac{f_{\text{clk}}}{\text{Baud Rate}} = \frac{100,000,000}{115,200} \approx 868.05 \implies \mathbf{868\text{ cycles}} \quad (\text{Error: } +0.0064\%)$$
  - Minimal Register Width: $\lceil \log_2(868) \rceil = \mathbf{10\text{ bits }} [9:0]$
- **Receiver Divisor ($16\times$ Oversampling Tick):**
  $$M_{16\text{x}} = \frac{f_{\text{clk}}}{16 \times \text{Baud Rate}} = \frac{100,000,000}{16 \times 115,200} \approx 54.25 \implies \mathbf{54\text{ cycles}} \quad (\text{Error: } +0.47\%)$$
  - Minimal Register Width: $\lceil \log_2(54) \rceil = \mathbf{6\text{ bits }} [5:0]$

### 2. Low-Power Receiver (`rtl/uart_rx.v`)
- **Metastability Protection:** 2-stage shift register (`rx_sync1`, `rx_sync2`) ensures safe clock domain crossing.
- **Glitch Rejection:** Samples midpoint of the start bit (tick 7 out of 0..15); if the line is not LOW (`0`), the transmission is rejected as a noise spike.
- **$16\times$ Center Sampling:** Data bits are sampled at the exact geometric center (tick 7) to guarantee maximum noise margin and tolerance against baud rate drift.

---

## 🧪 Vivado Behavioral Simulation & Waveform Verification

The functional verification of the core was carried out in **Xilinx Vivado Simulator** using the testbench configuration `uart_top_tb_behav.wcfg`:

![Vivado Behavioral Simulation Waveform](simulation_waveform.png)

### 🔬 Waveform Signal Analysis (Evidence of Verification):

| Signal Name | Observed Waveform Behavior | Technical Verification Insight |
| :--- | :--- | :--- |
| `clk` | Continuous $100\text{ MHz}$ square wave | Single global master clock domain without clock skew. |
| `reset` | Pulses HIGH from $0\text{ ns}$ to $100\text{ ns}$ | Flushes registers and initializes FSM to `STATE_IDLE`. |
| `tx_start` | Single-cycle strobe at $\approx 190\text{ ns}$ | Commands transmitter to begin serializing `tx_data`. |
| `tx_data[7:0]`| Loaded with `8'h55` (`01010101`) | Alternating bit pattern to test maximum high-frequency switching. |
| **`low_power_active`**| **HIGH (`1`) during idle $\rightarrow$ drops to LOW (`0`) upon transmission** | **Visually demonstrates hardware sleep during idle and rapid wakeup!** |
| `tx_busy` | Transitions HIGH ($\approx 200\text{ ns}$) | Reflects active serializer operation until stop bit completes. |
| `tx_line` | Transitions from idle HIGH $\rightarrow$ LOW | Generates precise Start Bit followed by serialized data bits. |
| `error_count[31:0]` | **`00000000` (Zero Errors)** | Proves 100% data transmission integrity without framing mismatch. |

---

## 📈 Physical Implementation & Vivado Experimental Results

Implemented on an AMD/Xilinx Artix-7 `xc7a35tcsg324-1` in Vivado Design Suite under identical constraints ($T_{\text{clk}} = 10.0\text{ ns}$):

### 1. Hardware Resource Utilization Comparison

| Resource Component | Conventional Baseline UART | Proposed Low-Power UART | Savings Achieved (%) |
| :--- | :---: | :---: | :---: |
| **Slice LUTs** | 98 | **56** | 🏆 **`42.86%` Area Reduction** |
| **Slice Registers (FFs)** | 100 | **76** | 🏆 **`24.00%` Register Reduction** |
| **Total Occupied Slices** | 46 | **31** | 🏆 **`32.61%` Slice Savings** |
| **Total Logic Endpoints** | 179 | **127** | 🏆 **`29.05%` Complexity Drop** |
| **Global Clock Buffers (BUFG)**| 1 | 1 | Preserves Low-Skew Tree |

### 2. Vivado Report Power Analysis ($25^\circ\text{C}$ Junction)

| Power Metric | Conventional Baseline UART | Proposed Low-Power UART | Power Reduction (%) |
| :--- | :---: | :---: | :---: |
| **Dynamic Power ($P_{\text{dynamic}}$)** | **`0.004 W` ($4\text{ mW}$)** | **`0.002 W` ($2\text{ mW}$)** | 🚀 **`50.00%` Dynamic Power Savings** |
| **Device Static Power** | $0.072\text{ W}$ ($72\text{ mW}$) | $0.072\text{ W}$ ($72\text{ mW}$) | Fixed 28nm silicon leakage floor |
| **Total On-Chip Power** | **`0.075 W` ($75\text{ mW}$)** | **`0.074 W` ($74\text{ mW}$)** | **`1.33%` Total Power Savings** |
| **Junction Temperature** | $25.4^\circ\text{C}$ | $25.4^\circ\text{C}$ | Safe Thermal Operating Margin |

### 3. Timing Closure & Maximum Frequency ($f_{\text{max}}$)

| Timing Metric | Conventional Baseline UART | Proposed Low-Power UART | Improvement |
| :--- | :---: | :---: | :---: |
| **Worst Negative Slack (WNS)** | $+4.500\text{ ns}$ | **`+6.301 ns`** | ⏱️ **`+1.801 ns` Positive Margin** |
| **Total Negative Slack (TNS)** | $0.000\text{ ns}$ | **`0.000 ns`** | ✅ **Timing Constraints MET** |
| **Worst Hold Slack (WHS)** | $+0.120\text{ ns}$ | **`+0.106 ns`** | ✅ **Hold Timing MET** |
| **Max Operating Freq ($f_{\text{max}}$)**| $181.82\text{ MHz}$ | **`270.34 MHz`** | 📈 **`48.69%` Performance Boost** |

---

## 🏆 Benchmarking Against Published Literature (IJCESEN 2025)

Our post-route implementation results were benchmarked directly against the recently published research paper:  
*“Design and Implementation of UART using Low Power Techniques” (IJCESEN, 2025)*:

| Parameter / Metric | Published Paper *(IJCESEN 2025)* | **Our Proposed Low-Power UART** | **Improvement over Published Paper** |
| :--- | :---: | :---: | :---: |
| **Implementation Stage** | *Synthesized Netlist* (Pre-route) | **Implemented Netlist (Post-route)** | 🏆 **Physically Verified On-Chip** |
| **Dynamic Power ($P_{\text{dynamic}}$)** | **`1.763 W` ($1,763\text{ mW}$)** | **`0.002 W` ($2\text{ mW}$)** | 🚀 **`99.89%` Dynamic Power Savings** |
| **Total On-Chip Power** | **`1.899 W` ($1,899\text{ mW}$)** | **`0.074 W` ($74\text{ mW}$)** | 🚀 **`96.10%` Total Power Savings** |
| **Slice LUTs** | **436 LUTs** | **56 LUTs** | 🏆 **`87.16%` Area Savings** *(7.8× Smaller)* |
| **Slice Registers (FFs)** | **352 FFs** | **76 FFs** | 🏆 **`78.41%` Flip-Flop Savings** |
| **Logic Depth (Critical Path)**| **34 Levels of Logic** ⚠️ | **~3 to 4 Levels** | 🏆 **Massive Delay Reduction** |
| **Timing Closure ($WNS$)** | *Unconstrained Paths* | **`+6.301 ns` (MET at 100 MHz)** | 🏆 **Full Timing Closure Achieved** |

---

## 🔌 Artix-7 FPGA Pin Mapping (XDC Constraints)

Pin assignments from [`constraints/uart_top.xdc`](constraints/uart_top.xdc) mapped for Artix-7:

| Port Identifier | Direction | Package Pin | I/O Standard | Description |
| :--- | :---: | :---: | :---: | :--- |
| `clk` | Input | `E3` | LVCMOS33 | 100 MHz Master System Clock Oscillator |
| `reset` | Input | `D9` | LVCMOS33 | Active-High Synchronous Reset (Pushbutton BTN0) |
| `loopback_mode`| Input | `A8` | LVCMOS33 | Switch SW0 (1 = Echo RX bytes back to TX, 0 = Manual) |
| `tx_start` | Input | `C9` | LVCMOS33 | Pushbutton BTN1 (Manual transmission trigger) |
| `rx` | Input | `A9` | LVCMOS33 | USB-UART Serial RX (Host PC $\rightarrow$ FPGA) |
| `tx` | Output | `D10` | LVCMOS33 | USB-UART Serial TX (FPGA $\rightarrow$ Host PC) |
| `low_power_active` | Output | `H5` | LVCMOS33 | Diagnostic LED0 (ON = System in low-power sleep) |
| `tx_busy` | Output | `J5` | LVCMOS33 | Diagnostic LED1 (ON = Transmission active) |
| `rx_valid` | Output | `T9` | LVCMOS33 | Diagnostic LED2 (Strobe = Valid byte received) |
| `rx_error` | Output | `T10` | LVCMOS33 | Diagnostic LED3 (ON = Framing error detected) |
| `tx_done` | Output | `E1` | LVCMOS33 | Diagnostic LED4 (Strobe = Transmission finished) |

---

## 🖥️ Hardware Demonstration & Testing Guide

1. Connect the Artix-7 development board to your PC via a USB cable.
2. Open a terminal emulator (e.g., **PuTTY**, **Tera Term**, or **MobaXterm**):
   - **COM Port:** Identify in Windows Device Manager (e.g., `COM3`)
   - **Baud Rate:** `115200`
   - **Data Bits:** `8` | **Parity:** `None` | **Stop Bits:** `1` | **Flow Control:** `None`
3. Flip Switch `SW0` UP (`loopback_mode = 1`):
   - Type characters in PuTTY; each keystroke is echoed back instantly.
   - Observe **`LED0`** turning OFF during active transfers and turning back ON during idle intervals.

---

## 📁 Repository Directory Structure

```
├── rtl/                        # Synthesizable Verilog Source Files
│   ├── baud_generator.v        # Parameterized Baud Generator with Counter Freeze
│   ├── uart_tx.v               # Low-Power Serial Transmitter (8-N-1)
│   ├── uart_rx.v               # Low-Power Receiver with 16x Center-Sampling
│   ├── low_power_control.v     # Line Activity Watchdog and Sleep Controller
│   └── uart_top.v              # Top-Level System with Loopback Echo Support
├── tb/                         # Simulation Testbenches
│   ├── baud_generator_tb.v     # Baud Divisor Verification Testbench
│   ├── uart_tx_tb.v            # Byte Serialization Testbench
│   ├── uart_rx_tb.v            # Glitch Filtering & Sampling Testbench
│   └── uart_top_tb.v           # End-to-End System Loopback Testbench
├── constraints/                # FPGA Physical Constraints
│   └── uart_top.xdc            # Master Artix-7 Physical & 100 MHz Timing Constraints
├── sim/                        # Simulation & Verification Scripts
│   ├── sim_uart.py             # Python Automated Testbench & Waveform Runner
│   ├── uart_top_tb_behav.wcfg  # Vivado Waveform Configuration
│   └── run_synth_power.tcl     # Vivado TCL Batch Synthesis & Power Script
├── reports/                    # Implementation Benchmark Reports
│   ├── comparative_analysis.md # Baseline vs Low-Power Comparative Analysis
│   └── low_power/              # Vivado Implemented Power and Timing Reports
└── documentation/              # Technical Specifications & Presentations
    ├── architecture_guide.md   # RTL Architecture & Power Reduction Mechanics
    ├── project_presentation_final.md # Complete 16-Slide Final Project Defense Presentation
    └── viva_questions_and_answers.md # Comprehensive Technical Q&A Guide
```

---

## 🚀 Step-by-Step Vivado Execution Guide

1. Open Vivado and run the automated TCL script:
   ```bash
   vivado -mode tcl -source sim/run_synth_power.tcl
   ```
2. Or use the GUI flow:
   - Create an RTL project targeting `xc7a35tcsg324-1`.
   - Add all Verilog files from `/rtl` as **Design Sources**.
   - Add all files from `/tb` as **Simulation Sources**.
   - Add `constraints/uart_top.xdc` as **Constraints**.
   - Click **Run Behavioral Simulation** to inspect waveforms.
   - Click **Generate Bitstream** to program the Artix-7 board.

---

## 📚 References & Citations
1. AMD/Xilinx, *7 Series FPGAs Configurable Logic Block User Guide* (UG474).
2. Pong P. Chu, *FPGA Prototyping by Verilog Examples: Xilinx Spartan-3/Artix-7 Version*, Wiley.
3. Neil H. E. Weste and David Money Harris, *CMOS VLSI Design: A Circuits and Systems Perspective*, Pearson.
4. *Design and Implementation of UART using Low Power Techniques*, IJCESEN, 2025.
