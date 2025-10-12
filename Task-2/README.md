# Static Timing Analysis (STA) and OpenSTA

## Introduction

**Static Timing Analysis (STA)** is a key technique for verifying the timing of digital designs. It checks whether circuits meet required timing constraints, ensuring reliable and correct operation at the intended clock frequency.

STA differs from **timing simulation**, which verifies both functionality and timing by applying stimulus vectors to the design and observing responses. In contrast, STA is *static*—it analyzes all possible timing paths in the design without depending on specific input data values.

> **Timing Analysis**: Refers to both static timing analysis and simulation-based timing analysis; both analyze designs for timing issues.
>
<img width="1164" height="658" alt="Screenshot 2025-10-12 103304" src="https://github.com/user-attachments/assets/142aab30-8c82-4bd1-a74a-a96fb6c6c97f" />

---

## STA in the Digital Design Flow

STA is performed at multiple stages in a CMOS digital design flow:

<img width="648" height="633" alt="Screenshot 2025-10-12 103313" src="https://github.com/user-attachments/assets/18168dee-9f9e-41f3-8a06-1d23412e8264" />



---

## OpenSTA

**OpenSTA** is an open-source static timing analyzer for gate-level digital circuit analysis. It uses a TCL command interpreter for reading designs, specifying timing constraints, and generating timing reports.

<img width="1266" height="434" alt="Screenshot 2025-10-12 103328" src="https://github.com/user-attachments/assets/6ca173e4-a319-425c-a0d1-5536dd9997f3" />


### Input Files

- `*.v` : Gate-level Verilog Netlist  
- `*.lib` : Liberty Timing Libraries  
- `*.sdc` : Synopsys Design Constraints (clocks, delays, false paths)  
- `*.sdf` : Annotated Delay File (optional)  
- `*.spef`: Parasitics (RC extraction)  
- `*.vcd` / `*.saif` : Switching Activity for Power Analysis 

---

## Key Features

### Clock Modeling

- **Generated Clocks**: Clocks derived from existing clocks  
- **Latency**: Clock propagation delay  
- **Source Latency**: Delay from clock source to input  
- **Uncertainty**: Jitter or skew margins  
- **Propagated vs. Ideal**: Real vs. ideal clock network  
- **Gated Clock Checks**: For conditionally enabled clocks  
- **Multi-Frequency Clocks**: Multiple clock domain support

### Timing Exceptions

Refine analysis for real behavior:

- `set_false_path`: Ignore invalid paths  
- `set_multicycle_path`: Allow multiple clock cycles  
- `set_max_delay` / `set_min_delay`: Set custom timing limits

### Delay Calculation

- **Integrated Dartu/Menezes/Pileggi RC algorithm**:  
  Models effective capacitance for RC networks to compute realistic gate/net delays.

- **External Delay Calculator API**:  
  Allows integration of custom delay models (e.g., layout-aware or temperature-adaptive models).

---

## Timing Analysis and Reporting

OpenSTA provides commands for analyzing timing paths, delays, and setup/hold checks:

- `report_checks`: Reports timing violations across paths.

  ```tcl
  report_checks -from [get_pins U1/Q] -to [get_pins U2/D]
  ```

---

## Timing Paths

**Timing Paths** are logical routes that signals take through a digital circuit, from their source (start point) to their destination (end point), passing through combinational and sequential elements. STA analyzes these to determine delays and timing margins.

- **Start Point**: Where a signal originates (input port or register's clock pin).
- **End Point**: Where a signal terminates (register's data input pin or output port).
- **Combinational Logic**: Gates and logic through which signals pass.

### Types of Timing Paths

<img width="807" height="661" alt="Screenshot 2025-10-12 103339" src="https://github.com/user-attachments/assets/399d7f30-3fa5-4f86-a0ee-7bccc008960a" />


1. **Input to Register (in2reg)**
2. **Register to Register (reg2reg)**
3. **Register to Output (reg2out)**
4. **Input to Output (in2out)**

---

## Setup and Hold Checks

- **Setup Check**:  
  The data must be stable for a minimum time *before* the clock edge.  
  If not, it results in a setup violation and incorrect data capture.

- **Hold Check**:  
  The data must remain stable for a minimum time *after* the clock edge.  
  If not, it results in a hold violation and possible data corruption.

---

## Slack Calculation

Slack quantifies the timing margin for setup and hold checks.

- **Setup Slack** = Data required time − Data arrival time  
- **Hold Slack** = Data arrival time − Data required time

> - **Data Arrival Time**: Time for a signal to travel from start to end point.
> - **Data Required Time**: Time determined by clock path traversal.
> - **Slack**: Difference between required and actual arrival times.
>     - **Positive Slack**: Design meets timing.
>     - **Zero Slack**: Design is at timing limit.
>     - **Negative Slack**: Timing violation exists.

---

## Common SDC Constraints

**Synopsys Design Constraints (SDC)** define timing, environmental, and power requirements for STA tools.

| Category                | Commands                                                                                                 |
|-------------------------|----------------------------------------------------------------------------------------------------------|
| **Operating Conditions**  | `set_operating_conditions`                                                                              |
| **Wire-load Models**      | `set_wire_load_mode`<br>`set_wire_load_model`<br>`set_wire_load_selection_group`                        |
| **Environmental**         | `set_drive`<br>`set_driving_cell`<br>`set_load`<br>`set_fanout_load`<br>`set_input_transition`<br>`set_port_fanout_number` |
| **Design Rules**          | `set_max_capacitance`<br>`set_max_fanout`<br>`set_max_transition`                                      |
| **Timing**                | `create_clock`<br>`create_generated_clock`<br>`set_clock_latency`<br>`set_clock_transition`<br>`set_disable_timing`<br>`set_propagated_clock`<br>`set_clock_uncertainty`<br>`set_input_delay`<br>`set_output_delay` |
| **Exceptions**            | `set_false_path`<br>`set_max_delay`<br>`set_multicycle_path`                                           |
| **Power**                 | `set_max_dynamic_power`<br>`set_max_leakage_power`                                                     |

### Constraint Purposes

- **Operating Conditions**: Set process-voltage-temperature (PVT) corners.
- **Wire-load Models**: Estimate interconnect parasitics pre-layout.
- **Environmental Constraints**: Model I/O drive and load characteristics.
- **Design Rule Constraints**: Enforce physical limitations (fanout, capacitance, transition).
- **Timing Constraints**: Define clocks, clock properties, and timing relationships.
- **Timing Exceptions**: Specify non-functional or multi-cycle paths.
- **Power Constraints**: Limit dynamic and leakage power.

---

## Further Reading

- [OpenSTA GitHub](https://github.com/The-OpenROAD-Project/OpenSTA)
- [SDC Syntax Reference](https://www.synopsys.com/support/training/rtl-synthesis/static-timing-analysis.html)
- [Liberty File Format (Timing Libraries)](https://www.synopsys.com/implementation-and-signoff/design-compiler/dc-ultra.html)

---

## License

This documentation is provided for educational purposes and is not affiliated with or endorsed by Synopsys or The OpenROAD Project.
