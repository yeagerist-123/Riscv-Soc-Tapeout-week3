# Part 3 – Generate Timing Graphs with OpenSTA
---

## Objective
- The purpose of this README is to document the **OpenSTA timing analysis workflow**, including installation, running example analyses, and comparing timing slacks for different designs.  
- This task focuses on understanding **timing behavior, slack calculation, and critical path evaluation** using OpenSTA.

---
## Installation Guide

### CUDD Installation

```bash
cd
git clone https://github.com/ivmai/cudd.git
cd cudd
autoreconf -I
mkdir build
cd build
../configure --prefix=$HOME/cudd
make
make install
```
![Cudd installation terminal output](Screenshots/cudd_terminal.jpg)

---

### Installation of OpenSTA
```bash
sudo apt-get update
sudo apt-get install build-essential tcl-dev tk-dev cmake git libeigen3-dev autoconf m4 perl automake 

git clone https://github.com/The-OpenROAD-Project/OpenSTA.git
cd OpenSTA
cd build
cmake .. -DUSE_CUDD=ON -DCUDD_DIR=$HOME/cudd
make
sudo make install
sta
```
![OpenSTA installation terminal output](Screenshots/OpenSTA_terminal_op.jpg)

---
## Example Timing Analysis

### Step 1 : Verilog Module
- This module is given by OpenSTA as `example1.v` 

```verilog
module top (in1, in2, clk1, clk2, clk3, out);
  input in1, in2, clk1, clk2, clk3;
  output out;
  wire r1q, r2q, u1z, u2z;

  DFF_X1 r1 (.D(in1), .CK(clk1), .Q(r1q));
  DFF_X1 r2 (.D(in2), .CK(clk2), .Q(r2q));
  BUF_X1 u1 (.A(r2q), .Z(u1z));
  AND2_X1 u2 (.A1(r1q), .A2(u1z), .ZN(u2z));
  DFF_X1 r3 (.D(u2z), .CK(clk3), .Q(out));
endmodule 
```
---
### Step 2: Synthesis

```bash
cd /OpenSTA/examples/
yosys
read_liberty -lib nangate45_slow.lib.gz
read_verilog example1.v
synth -top top
show
```
**Screenshot** : Synthesizing terminal output

![Synthesis terminal op](Screenshots/synthesis_terminal_op1.jpg)

---

**Screenshot** : Stats of cells used 

![Synthesis terminal op](Screenshots/synthesis_terminal_op2.jpg)

---
### Step 3 : Timing analysis using OpenSTA

```bash
 sta
```
- After launching the OpenSTA interactive shell (denoted by the `%` prompt), you can run the following commands to carry out a basic static timing analysis:

```bash
# Load the standard cell timing library (Liberty format)
read_liberty ./OpenSTA/examples/nangate45_slow.lib.gz

# Load the gate-level Verilog netlist
read_verilog ./OpenSTA/examples/example1.v

# Link the top-level module with the loaded timing library
link_design top

# Define a 10 ns clock named 'clk' for inputs clk1, clk2, and clk3
create_clock -name clk -period 10 {clk1 clk2 clk3}

# Set input delay of 0 ns for signals in1 and in2 relative to clock 'clk'
set_input_delay -clock clk 0 {in1 in2}

# Generate a timing check report for the design (By default it do setup/max checks)
report_checks

# Generate a timing check report for min check 
report_checks -path_delay min
```
**Screenshot**: Terminal Output of STA Commands

![Cmd Terminal](Screenshots/cmd_Terminal.jpg)

---

**Screenshot** : Max path timing report

![Max path check](Screenshots/max_path_check.jpg)

---
**Screenshot** : Min path timing report

![Min path check](Screenshots/min_path_check.jpg)

---
**Screenshot** : Synthesized design

![Show](Screenshots/show.jpg)
 
---

## Analysis of Timing :

| **Parameter** | **Description**      | **Value (ns)** |
|:--------------|:---------------------|:--------------:|
| **r2/Q**      | Clock-to-Q delay      | 0.23 |
| **u1**        | Buffer delay          | 0.08 |
| **u2**        | AND2 delay            | 0.10 |
| **Clock Period** | Clock period                  | 10.00 |
| **Library Setup Time** | Setup time           | -0.16 |

---

## Timing Calculations
---
## Set Up Time Analysis

**Data Arrival Time (tarrival):**
```bash 
tarrival = tclk_q + tbuf + tand
tarrival = 0.23 + 0.08 + 0.10 = 0.41 ns
```

**Data Required Time (trequired):**
```bash
trequired = Tclk - tsetup
trequired = 10 - 0.16 = 9.84 ns
```
**Slack Calculation**
```bash
Slacksetup = trequired - tarrival
Slacksetup = 9.84 - 0.41 = 9.43 ns (Positive → MET)
```

###  Observation

- The **setup slack is positive (9.43 ns)** → **timing is met**.  
- There is a **large timing margin**, meaning the circuit can tolerate extra delay.  
- The **negative setup time (-0.16 ns)** effectively increases the available time, aiding timing closure.

---

##  Hold Time Analysis

For the **hold check**, we consider the **shortest data path**.

**Data Arrival Time (tarrival_min):** 
```bash 
tarrival_min = tclk_q + tcomb_min
tarrival_min = 0.00 ns
```

**Data Required Time (trequired_hold):**  
```bash
trequired_hold = thold
trequired_hold = 0.01 ns
```
**Slack Calculation**
```bash
Slackhold = tarrival_min − trequired_hold
Slackhold = 0.00 − 0.01
Slackhold = −0.01 ns (VIOLATED)
```
### Observation
- The **hold slack is negative (−0.01 ns)** → **Hold timing is violated**.  
- This indicates that data is arriving **too early** at the capture flip-flop before it becomes stable for the next clock edge.  
- The path therefore **fails the hold requirement**, causing a potential data corruption risk.

**Screenshot** : The timing report were analysed and Verified.

![Delay](Screenshots/Delay.jpg)

---

## SPEF-Based Timing Analysis

**Definition:**  
**SPEF** is a standardized textual file format that provides detailed parasitic information (resistance, capacitance, and sometimes inductance) of a chip’s routing interconnects, which is used for **accurate post-layout timing analysis (STA)**.

### Key Points:
- Captures **RC parasitics** of nets after placement and routing.  
- Used by **Static Timing Analysis (STA) tools** to compute **delay, slew, and crosstalk effects**.  
- Ensures **timing closure** by reflecting realistic interconnect delays.  
- Comes in two main types:  
  - **Unit SPEF:** resistances in ohms, capacitances in farads.  
  - **Scaled SPEF:** values scaled by a factor for readability.  

**Summary:**  
SPEF bridges the gap between layout parasitics and accurate timing analysis.

---
### Steps to do SPEF Based Timing Analysis
```bash
# Change to the directory containing OpenSTA examples
cd OpenSTA/examples

# Invoke the OpenSTA tool
sta

# Load the standard cell timing library (Liberty format)
read_liberty ./nangate45_slow.lib.gz

# Load the gate-level Verilog netlist for analysis
read_verilog ./example1.v

# Link the top-level module in the Verilog netlist with the loaded timing library
link_design top

# Load the parasitic SPEF file for accurate delay calculation
read_spef ./example1.dspef

# Define a 10 ns clock named 'clk' for signals clk1, clk2, and clk3
create_clock -name clk -period 10 {clk1 clk2 clk3}

# Set input delay of 0 ns for signals in1 and in2 relative to the clock 'clk'
set_input_delay -clock clk 0 {in1 in2}

# Generate timing report for max check
report_checks

# Generate timing report for min check
report_checks -path_delay min
```
**Screenshot** : Terminal output of the tcl script

![Terminal_op](Screenshots/terminal_op.jpg)

**Screenshot**: SPEF Based Max path Check 

![Max SPEF Analysis](Screenshots/max_spef.jpg)

---
**Screenshot**: SPEF Based Min path Check 
 
![Min SPEF Analysis](Screenshots/min_spef.jpg)
 ---
## Capacitance Analysis
- generate a report of parasitic or electrical checks with a focus on capacitances 
```bash
report_checks -digits 4 -fields capacitance
```
 ![capacitance_check](Screenshots/capacitance_check.jpg)

 ---
 ```bash
report_checks -digits 4 -fields [list capacitance slew input_pins fanout]
```
![cap](Screenshots/cap.jpg)

---
## Power Report Analysis
- Generates a **power analysis report** for your design.
- Computes **dynamic, static, and total power** for the circuit.

**Command:**
```bash
report_power
```
![power analysis](Screenshots/power_analysis.jpg)

---

## Pulse Width Checks

- Checks the **pulse width of signals** in your design.
- Ensures that **short glitches or narrow pulses** do not violate timing constraints.
- Important for **setup/hold integrity** and **avoiding false switching**.

**Command:**
```bash
report_pulse_width_checks
```
![pulse_width](Screenshots/pulse_width.jpg)

---
## Report Units
- Displays the **units of measurement** currently used in the STA tool.

**Command:**
```bash
report_units
```
![unit](Screenshots/unit.jpg)

---
## Timing Report Comparison: Without SPEF vs With SPEF

| Node / Signal                  | Without SPEF Delay / Time (ns) | With SPEF Delay / Time (ns) |
|--------------------------------|-------------------------------|-----------------------------|
| clock clk (rise edge)           | 0.00 / 0.00                   | 0.00 / 0.00                 |
| clock network delay (ideal)     | 0.00 / 0.00                   | 0.00 / 0.00                 |
| ^ r2/CK (DFF_X1)               | 0.00 / 0.00                   | 0.00 / 0.00                 |
| r2/Q (DFF_X1)                  | 0.23 / 0.23                   | 2.58 / 2.58                 |
| u1/Z (BUF_X1)                  | 0.08 / 0.31                   | 2.58 / 5.16                 |
| u2/ZN (AND2_X1)                | 0.10 / 0.41                   | 2.75 / 7.91                 |
| r3/D (DFF_X1)                  | 0.00 / 0.41                   | 0.00 / 7.92                 |
| **Data Arrival Time**           | 0.41                           | 7.92                         |
| clock clk (rise edge)           | 10.00 / 10.00                  | 10.00 / 10.00                |
| clock network delay (ideal)     | 0.00 / 10.00                   | 0.00 / 10.00                 |
| clock reconvergence pessimism   | 0.00 / 10.00                   | 0.00 / 10.00                 |
| r3/CK (DFF_X1)                 | 10.00 / 10.00                  | 10.00 / 10.00                |
| Library Setup Time              | -0.16 / 9.84                   | -0.57 / 9.43                 |
| **Data Required Time**          | 9.84                            | 9.43                         |
| **Slack**                       | ✅ 9.43 (MET)                  | ⚠️ 1.52 (MET)                |

### Observation

- Including **SPEF parasitics** significantly increases the **delays** along the data path.  
- **Data arrival time** at `r3/D` increases from **0.41 ns → 7.92 ns**.  
- **Slack decreases** from **9.43 ns → 1.52 ns**, though it still meets the setup requirement.  
- The increase in delay is mainly due to **interconnect capacitance and resistance** captured in the SPEF file.  
- This highlights the importance of considering **post-layout parasitics** for accurate timing analysis.
---

## Note:
- The `Timing analysis of VSDBabySoc` and `Multi-Corner PVT` analysis are done in [View VSDBabySoC Timing Analysis](VSDBabySoC_Timing_Analysis.md)

---
