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
![OpenSTA installation terminal output]
<img width="1920" height="923" alt="w311" src="https://github.com/user-attachments/assets/d4c2f374-2a71-4445-81b6-fcff16d5f5d6" />


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

<img width="1920" height="923" alt="w3-2" src="https://github.com/user-attachments/assets/17b214bf-652c-4600-b61c-95892d11c4c9" />
<img width="1920" height="923" alt="w3-3" src="https://github.com/user-attachments/assets/f509df03-adce-40b9-af3b-1851fa2b9449" />


---


---
### Step 3 : Timing analysis using OpenSTA

```bash
 sta
```
- After launching the OpenSTA interactive shell (denoted by the `%` prompt), you can run the following commands to carry out a basic static timing analysis:

```bash
# Load the standard cell timing library (Liberty format)
read_liberty ./nangate45_slow.lib.gz

# Load the gate-level Verilog netlist
read_verilog ./example1.v

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

<img width="1920" height="923" alt="w3-12" src="https://github.com/user-attachments/assets/817a4a93-c532-4231-b95a-b889c1488060" />
<img width="1920" height="923" alt="w3-13" src="https://github.com/user-attachments/assets/29a9af5d-94ef-42a1-880c-b1caff12c9ea" />


---
**Screenshot** : Synthesized design

<img width="1920" height="923" alt="w3-4" src="https://github.com/user-attachments/assets/a382583a-cd6d-4204-bade-02e46038dce1" />
 
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
<img width="1920" height="923" alt="w33" src="https://github.com/user-attachments/assets/bb866f2f-a617-4689-b97a-6c94d70d30ca" />
<img width="1920" height="923" alt="w44" src="https://github.com/user-attachments/assets/c4c1d794-cdd1-4477-bd07-6533dc068896" />


 ---
## Capacitance Analysis
- generate a report of parasitic or electrical checks with a focus on capacitances 
```bash
report_checks -digits 4 -fields capacitance
```
<img width="1920" height="923" alt="w55" src="https://github.com/user-attachments/assets/565b7269-0cc0-4625-b27a-b63f7621604c" />


 ---
 ```bash
report_checks -digits 4 -fields [list capacitance slew input_pins fanout]
```
<img width="1920" height="923" alt="w66" src="https://github.com/user-attachments/assets/e4cbb698-662d-41fd-a6fc-0da39de2aaf1" />


---
## Power Report Analysis
- Generates a **power analysis report** for your design.
- Computes **dynamic, static, and total power** for the circuit.

**Command:**
```bash
report_power
```
<img width="469" height="151" alt="Screenshot 2025-11-22 233241" src="https://github.com/user-attachments/assets/f8ed8931-655e-44eb-8b4c-b2e098c4d019" />


---

## Pulse Width Checks

- Checks the **pulse width of signals** in your design.
- Ensures that **short glitches or narrow pulses** do not violate timing constraints.
- Important for **setup/hold integrity** and **avoiding false switching**.

**Command:**
```bash
report_pulse_width_checks
```
<img width="448" height="131" alt="Screenshot 2025-11-22 233249" src="https://github.com/user-attachments/assets/4d5fa682-366c-4cbc-be2d-05d13bff0fdc" />


---
## Report Units
- Displays the **units of measurement** currently used in the STA tool.

**Command:**
```bash
report_units
```
<img width="186" height="99" alt="Screenshot 2025-11-22 233532" src="https://github.com/user-attachments/assets/a5e297e8-5630-48aa-acb9-bb0ef8f0b788" />


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

# Timing Analysis of VSDBabySoc
---
## Steps to do Timing analysis of VSDBabySoC

```bash
cd Desktop/vlsi/VSDBabySoC
sta

# Load Liberty Libraries (standard cell + IPs)
read_liberty  ./src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
read_liberty  ./src/lib/avsdpll.lib
read_liberty ./src/lib/avsddac.lib

# Read Synthesized Netlist
read_verilog ./src/module/vsdbabysoc.synth.v

# Link the Top-Level Design
link_design vsdbabysoc

# Apply SDC Constraints
read_sdc ./src/sdc/vsdbabysoc_synthesis.sdc
 
#SDC Constraints
set_units -time ns
create_clock [get_pins {pll/CLK}] -name clk -period 11

# Generate Timing Report (by default max path)
report_checks

# Generate Timing Report for min path
report_checks -path_delay min
```
**Screenshot**: Terminal Output
 
<img width="1920" height="923" alt="w3-50" src="https://github.com/user-attachments/assets/ecaf181d-cf2c-48b0-8fc5-c356c3c5ef24" />

<img width="1920" height="923" alt="w3-51" src="https://github.com/user-attachments/assets/72aba42c-89cf-4b23-84a1-881aa203fae9" />

---
# Multi-PVT Corner Timing Analysis of VSDBabySoC using OpenSTA
---
## Multi-PVT Corners in STA

**Definition:**  
In **Static Timing Analysis (STA)**, **Multi-PVT Corners** refer to evaluating the design under multiple combinations of **Process, Voltage, and Temperature (PVT)** conditions. These corners help ensure that the design **meets timing constraints across all expected operating conditions**.

### Components of PVT:

1. **Process (P):**  
   - Variations in manufacturing, e.g., **slow (SS), typical (TT), fast (FF)** transistors.
   - Accounts for **chip-to-chip variability**.

2. **Voltage (V):**  
   - Variations in supply voltage, e.g., **nominal, high, low**.
   - Ensures timing is robust against **power supply fluctuations**.

3. **Temperature (T):**  
   - Variations in operating temperature, e.g., **-40°C, 25°C, 125°C**.
   - Models the effect of **thermal conditions** on transistor speed.

### Purpose of Multi-PVT Corners:

- STA at a **single corner** is insufficient for real-world conditions.  
- Multi-PVT analysis ensures the **design meets timing and functional requirements** under all expected scenarios:  
  - **Fast Process + High Voltage + Low Temperature** → fastest circuits, check **hold violations**.  
  - **Slow Process + Low Voltage + High Temperature** → slowest circuits, check **setup violations**.  

  **Example Scenarios:**

- **Fast Corner:** FF transistors at **-40 °C, 1.95 V** → circuits are faster; checks **hold violations** (data may arrive too early).  
- **Slow Corner:** SS transistors at **100 °C, 1.40 V** → circuits are slower; checks **setup violations** (data may arrive too late). 
---
## Timing Libraries for Multi-PVT Analysis

The timing libraries required for this analysis can be downloaded from the **SkyWater PDK**:

- **SkyWater PDK – sky130_fd_sc_hd Timing Libraries**  
  - These libraries provide **process-, voltage-, and temperature-specific timing models** needed for accurate STA.  
  - You can download them from the official [SkyWater PDK repository](https://github.com/google/skywater-pdk) or the timing library package link provided by the PDK.
---
 
## Using the Multi-PVT TCL Script for STA

We are going to use the `multi_pvt_corners.tcl` script to **automate Static Timing Analysis (STA) across multiple PVT corners**. This allows us to verify that the design meets timing constraints under all **process, voltage, and temperature conditions** without manually switching libraries or rerunning analyses.

### Steps Performed by the Script:

1. **Load PVT-specific timing libraries**  
   - Example: `sky130_fd_sc_hd__ss_100C_1v40.lib` for a **slow-slow, high-temperature, low-voltage** corner.  

2. **Link the synthesized netlist**  
   - Ensures the same RTL design is used for all corners.  

3. **Apply SDC constraints**  
   - Clocks, input/output delays, and timing exceptions are applied consistently for each corner.  

4. **Run timing checks**  
   - Includes **setup, hold, worst negative slack (WNS), total negative slack (TNS)** for each corner.  

5. **Save detailed reports**  
   - Generates a **separate report for each PVT corner** under `./sta_outputs/` for analysis.  

### Benefits:

- Provides **comprehensive timing validation** across all operating conditions.  
- Automates repetitive STA tasks, saving **time and effort**.  
- Identifies **worst-case paths** for setup and hold, ensuring reliable chip operation.  

---

## Script to run Static Timing Analysis for all corners

```bash
#---------------------------------------------
#  Multi-corner STA Automation Script (OpenSTA)
#---------------------------------------------

# Define list of timing libraries (corners)
set list_of_lib_files {
    sky130_fd_sc_hd__ff_n40C_1v95.lib
    sky130_fd_sc_hd__ff_100C_1v65.lib
    sky130_fd_sc_hd__ff_100C_1v95.lib
    sky130_fd_sc_hd__ff_n40C_1v56.lib
    sky130_fd_sc_hd__ff_n40C_1v65.lib
    sky130_fd_sc_hd__ff_n40C_1v76.lib
    sky130_fd_sc_hd__ss_100C_1v40.lib
    sky130_fd_sc_hd__ss_100C_1v60.lib
    sky130_fd_sc_hd__ss_n40C_1v28.lib
    sky130_fd_sc_hd__ss_n40C_1v35.lib
    sky130_fd_sc_hd__ss_n40C_1v40.lib
    sky130_fd_sc_hd__ss_n40C_1v44.lib
    sky130_fd_sc_hd__ss_n40C_1v76.lib
    sky130_fd_sc_hd__ss_n40C_1v60.lib
    sky130_fd_sc_hd__tt_025C_1v80.lib
    sky130_fd_sc_hd__tt_100C_1v80.lib
}

#---------------------------------------------
#  Load base cell libraries and design files
#---------------------------------------------
read_liberty ./src/lib/avsdpll.lib
read_liberty ./src/lib/avsddac.lib

#---------------------------------------------
#  Create output folder
#---------------------------------------------
file mkdir sta_outputs

#---------------------------------------------
#  Loop through each .lib file (corner)
#---------------------------------------------
set i 1
foreach lib_file $list_of_lib_files {

    puts "\n=== Running STA for corner: $lib_file ==="

    # Load corner-specific library
    read_liberty ./src/lib/$lib_file

    # Read design and constraints
    read_verilog ./src/module/vsdbabysoc.synth.v
    link_design vsdbabysoc
    current_design vsdbabysoc
    read_sdc ./src/sdc/updated_synth.sdc

    # Perform timing checks
    check_setup -verbose

    #-----------------------------------------
    # Generate detailed reports
    #-----------------------------------------
    report_checks \
        -path_delay min_max \
        -fields {nets cap slew input_pins fanout} \
        -digits 4 \
        > ./sta_outputs/min_max_$lib_file.txt

    #-----------------------------------------
    # Save key metrics (WNS, TNS)
    #-----------------------------------------
    exec echo "$lib_file" >> ./sta_outputs/sta_worst_max_slack.txt
    report_worst_slack -max -digits 4 >> ./sta_outputs/sta_worst_max_slack.txt

    exec echo "$lib_file" >> ./sta_outputs/sta_worst_min_slack.txt
    report_worst_slack -min -digits 4 >> ./sta_outputs/sta_worst_min_slack.txt

    exec echo "$lib_file" >> ./sta_outputs/sta_tns.txt
    report_tns -digits 4 >> ./sta_outputs/sta_tns.txt

    exec echo "$lib_file" >> ./sta_outputs/sta_wns.txt
    report_wns -digits 4 >> ./sta_outputs/sta_wns.txt

    incr i
}
puts "\n All corners analysed. Reports saved in ./sta_outputs/"
``` 
---
## SDC Constraints For VSDBabySoC

```bash
# =============================================================================
# SDC Constraints for vsdbabysoc Module (synthesised netlist)
# Generated for OpenSTA Static Timing Analysis
# Clock period: 11 ns (~90.9 MHz)
# =============================================================================

set_units -time ns

# Clock definition
create_clock -name clk -period 11 [get_pins pll/CLK]

set_clock_latency -source 2 [get_clocks clk]
set_clock_latency 1 [get_clocks clk]
set_clock_uncertainty -setup 0.5 [get_clocks clk]
set_clock_uncertainty -hold 0.5 [get_clocks clk]

# Design constraints
set_max_area 8000
set_max_fanout 5 vsdbabysoc
set_max_transition 10 vsdbabysoc

# Input constraints
set_input_delay -clock clk -max 4 [get_ports {reset VCO_IN ENb_CP ENb_VCO REF VREFH}]
set_input_delay -clock clk -min 1 [get_ports {reset VCO_IN ENb_CP ENb_VCO REF VREFH}]
set_input_transition -max 0.4 [get_ports {reset VCO_IN ENb_CP ENb_VCO REF VREFH}]
set_input_transition -min 0.1 [get_ports {reset VCO_IN ENb_CP ENb_VCO REF VREFH}]

# Output constraints
set_load -max 0.5 [get_ports OUT]
set_load -min 0.5 [get_ports OUT]
set_output_delay -clock clk -max 0.5 -clock clk [get_ports OUT]
set_output_delay -clock clk -min 0.5 -clock clk [get_ports OUT]

# Path delay
set_max_delay 10 -from [get_clocks clk] -to [get_ports OUT]
```
## How to run Multi-PVT corner analysis
```bash
# Go to OpenSTA interactive shell (denoted by %)
sta

source multi_pvt_corners.tcl
```

<img width="1920" height="1080" alt="w3-99" src="https://github.com/user-attachments/assets/0d899367-1728-4018-aeeb-489675bdb798" />

---
## STA Output
- After successfull analysis of `multi pvt corners`, it produces 4 output files.
They are.
```bash
sta_tns.txt
sta_wns.txt
sta_worst_max_slack.txt
sta_worst_min_slack.txt
```
**Screenshot of sta_worst_max_slack.txt**

<img width="1920" height="1080" alt="wmax" src="https://github.com/user-attachments/assets/dab7c0a4-9483-446c-abc0-37254b1ad38e" />


---
**Screenshot of sta_worst_min_slack.txt**

<img width="1920" height="1080" alt="wmin" src="https://github.com/user-attachments/assets/e4e69fb1-5640-4d5e-b201-cf89d0d96cac" />


---
**Screenshot of sta_wns.txt**

<img width="1920" height="1080" alt="wns" src="https://github.com/user-attachments/assets/d3203415-ab98-4954-af2a-acf9a03d2660" />


---
**Screenshot of sta_tns.txt**

<img width="1920" height="1080" alt="tns" src="https://github.com/user-attachments/assets/d70048ba-6bb6-4a10-be3d-627b0bc9a30a" />


---
# Multi-PVT Timing Summary Report

This report summarises the **setup and hold timing performance** across multiple PVT (Process, Voltage, Temperature) corners analysed using **OpenSTA** for the vsdbabysoc design.

# Timing Summary: VSDBabySoC Post-Route Multi-PVT Analysis

## Legend

- 🟩 = Good (positive slack)
- 🟥 = Bad (negative slack → violation)

---

## ⭐ Final Timing Summary Table

| PVT Corner       | Setup Slack (worst max) | Hold Slack (worst min) | WNS       | TNS         | Observation                          |
|------------------|------------------------|------------------------|-----------|-------------|--------------------------------------|
| ff_n40C_1v95     | 🟩 +3.4517              | 🟥 −0.3125              | 🟩 0.0000 | 🟩 0.0000    | Hold slightly failing, setup excellent |
| ff_100C_1v65     | 🟩 +1.8571              | 🟥 −0.2509              | 🟩 0.0000 | 🟩 0.0000    | Minor hold issue, setup OK           |
| ff_100C_1v95     | 🟩 +3.2718              | 🟥 −0.3040              | 🟩 0.0000 | 🟩 0.0000    | Hold small violation, setup strong   |
| ff_n40C_1v56     | 🟩 +0.4674              | 🟥 −0.2085              | 🟩 0.0000 | 🟩 0.0000    | Slight hold violation                |
| ff_n40C_1v65     | 🟩 +1.4847              | 🟥 −0.2449              | 🟩 0.0000 | 🟩 0.0000    | Minor hold issue                     |
| ff_n40C_1v76     | 🟩 +2.3758              | 🟥 −0.2757              | 🟩 0.0000 | 🟩 0.0000    | Slight hold violation                |
| ss_100C_1v40     | 🟥 −13.8601             | 🟩 +0.4053              | 🟥 −13.8601| 🟥 −8230.1904 | Major setup failure                  |
| ss_100C_1v60     | 🟥 −7.0086              | 🟩 +0.1420              | 🟥 −7.0086 | 🟥 −3244.6812 | Setup fail                           |
| ss_n40C_1v28     | 🟥 −54.0971             | 🟩 +0.6461              | 🟥 −54.0971| 🟥 −37717.0312| Severe setup fail                    |
| ss_n40C_1v35     | 🟥 −34.1196             | 🟩 +0.6229              | 🟥 −34.1196| 🟥 −24030.6152| Large setup fail                     |
| ss_n40C_1v40     | 🟥 −25.4777             | 🟩 +0.6119              | 🟥 −25.4777| 🟥 −17867.7402| Large setup fail                     |
| ss_n40C_1v44     | 🟥 −20.7284             | 🟩 +0.4909              | 🟥 −20.7284| 🟥 −14272.8594| Large setup fail                     |
| ss_n40C_1v76     | 🟥 −4.7197              | 🟩 +0.0038              | 🟥 −4.7197 | 🟥 −2284.7170 | Setup fail                           |
| ss_n40C_1v60     | 🟥 −9.8793              | 🟩 +0.1628              | 🟥 −9.8793 | 🟥 −5687.9634 | Setup fail                           |
| tt_025C_1v80     | 🟩 +0.4766              | 🟥 −0.1904              | 🟩 0.0000 | 🟩 0.0000    | Tiny hold issue, good setup          |
| tt_100C_1v80     | 🟩 +0.5359              | 🟥 −0.1855              | 🟩 0.0000 | 🟩 0.0000    | Very small hold issue                |

---

## ⭐ Overall Observation

- ✔ **All FF (fast-fast) corners have:**
    - Strong setup
    - Very small hold violations (expected in fast corners)
- ✔ **All TT (typical-typical) corners are almost clean** except for tiny hold violations.
- ✔ **All SS (slow-slow) corners show:**
    - Major and severe setup violations.
    - Because slow corners → slow cells → timing fails.

---
---

## TIMING SUMMARY & VISUALISATIONS

1. **Worst Max Slack Across corners**

<img width="1920" height="1080" alt="wmax gr" src="https://github.com/user-attachments/assets/4ce382cf-24f5-4f32-9f45-de518f449c90" />


**Observation**
- The Setup Slack values for most corners are positive, indicating timing is met for those corners.
- The worst corner (ss_n40C_1v28) has the maximum setup slack , showing a significant timing violation under the slow-slow (SS) corner at low voltage and low temperature.

---

2. **Worst Min Slack Across Corners**

<img width="1920" height="1080" alt="hold gr" src="https://github.com/user-attachments/assets/ee65c18d-442f-462c-9bce-816e9c7e382b" />


**Observation**
- The Hold Slack values for most corners are positive, indicating no hold violations for those corners.
- The worst corner (ss_n40C_1v28) has the minimum hold slack.

---

3. **Worst Negative Slack Across Corners**

<img width="1920" height="1080" alt="wnsgraph" src="https://github.com/user-attachments/assets/6a825f49-66af-449f-a9a4-46549e197164" />


**Observation**
- Some corners have negative setup slack, indicating setup timing violations.
- The worst negative slack (WNS) occurs at ss_n40C_1v28 .
---

4. **Total Negative Slack Across Corners**

<img width="1920" height="1080" alt="tns gr" src="https://github.com/user-attachments/assets/6daf0a2a-4f28-46de-a61e-8f5c44fd5b50" />


**Observation**
- Total Negative Slack aggregates all negative setup slacks.
- The highest TNS occurs at ss_n40C_1v28, indicating the largest cumulative timing violations.

---
## Inference from Multi PVT Corners slack analysis

- The analysis of hold slack, setup slack, and total negative slack across all process corners reveals that the worst-case timing occurs in the ss_n40C_1v28 corner, with significant setup violations.
- Hold timing is generally safe across corners. The total negative slack highlights cumulative violations, guiding optimization priorities. 
- Focus should be on critical paths in the slowest corners to achieve timing closure and reliable operation.

---
##  Conclusion

- Fast (FF) and Typical (TT) corners meet timing comfortably; setup and hold slacks are positive.
- Slow-Slow (SS) corners, especially ss_n40C_1v28, show severe setup violations, indicating paths are too slow under low voltage and low temperature.
- Hold timing is safe across all corners; no early data capture issues observed.
- Optimization is needed for critical paths in slow corners: retiming, path restructuring, or clock adjustment.
- Overall, the design is robust in typical and fast conditions, but slow corners require attention to ensure reliable operation across all PVT scenarios.
---
