# GLS OF BABYSOC
## POST-SYNTHESIS SIMULATION

### Purpose of GLS:
Gate-Level Simulation is used to verify the functionality of a design after the synthesis process. Unlike behavioral or RTL (Register Transfer Level) simulations, which are performed at a higher level of abstraction, GLS works on the netlist generated post-synthesis. This netlist includes the actual gates and connections used to implement the design.

### Key Aspects of GLS for BabySoC:
1. **Verification with Timing Information:**
   - GLS is performed using Standard Delay Format (SDF) files to ensure timing correctness.
   - This checks if the SoC behaves as expected under real-world timing constraints.

2. **Design Validation Post-Synthesis:**
   - Confirms that the design's logical behavior remains correct after mapping it to the gate-level representation.
   - Ensures that the design is free from issues like metastability or glitches.

3. **Simulation Tools:**
   - Tools like Icarus Verilog or a similar simulator can be used for compiling and running the gate-level netlist.
   - Waveforms are typically analyzed using GTKWave.

4. **Importance for BabySoC:**
   - BabySoC consists of multiple modules like the RISC-V processor, PLL, and DAC. GLS ensures that these modules interact correctly and meet the timing requirements in the synthesized design.


Here is the step-by-step execution plan for running the  commands manually:
---
### **Step 1: Load the Top-Level Design and Supporting Modules**
```bash
yosys
```

<img width="1920" height="922" alt="w3-1" src="https://github.com/user-attachments/assets/9d6092ca-1b4e-4d8a-9e0c-a776eb05a4d3" />



Inside the Yosys shell, run:
```yosys
read_verilog src/module/vsdbabysoc.v
read_verilog -I src/include src/module/rvmyth.v
read_verilog -I src/include src/module/clk_gate.v

```
<img width="1920" height="922" alt="w3-2" src="https://github.com/user-attachments/assets/7422858d-6df3-4d83-98aa-df5954aa781a" />


---

### **Step 2: Load the Liberty Files for Synthesis**
Inside the same Yosys shell, run:
```yosys
read_liberty -lib src/lib/avsdpll.lib
read_liberty -lib src/lib/avsddac.lib
read_liberty -lib src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```
<img width="1920" height="922" alt="w3-3" src="https://github.com/user-attachments/assets/cfd03612-38a0-48a4-b6d9-40698edb74fd" />


---

### **Step 3: Run Synthesis Targeting `vsdbabysoc`**
```yosys
synth -top vsdbabysoc
```

<img width="1920" height="922" alt="w3-4" src="https://github.com/user-attachments/assets/90b56053-e87c-4d5c-ad94-6a856d6de66f" />


---

### **Step 4: Map D Flip-Flops to Standard Cells**
```yosys
dfflibmap -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib
```
<img width="1920" height="922" alt="w3-5" src="https://github.com/user-attachments/assets/2efdae06-3b7c-4e20-8346-32b716db4a7c" />


---

### **Step 5: Perform Optimization and Technology Mapping**
```yosys
opt
abc -liberty src/lib/sky130_fd_sc_hd__tt_025C_1v80.lib -script +strash;scorr;ifraig;retime;{D};strash;dch,-f;map,-M,1,{D}
```
<img width="1920" height="922" alt="w3-6" src="https://github.com/user-attachments/assets/12f7c45f-e556-4540-89b7-8f2fbb71af72" />
<img width="1920" height="922" alt="w3-7" src="https://github.com/user-attachments/assets/089d894e-c1a4-4d2f-bd8c-ddd2cb2ff128" />
<img width="1920" height="922" alt="w3-8" src="https://github.com/user-attachments/assets/4791bfe2-cfe8-463c-b05c-311570663c63" />
<img width="1920" height="922" alt="w3-9" src="https://github.com/user-attachments/assets/f881afdf-4869-4c1f-8af8-567f6b4a2145" />

---

### **Step 6: Perform Final Clean-Up and Renaming**
```yosys
flatten
setundef -zero
clean -purge
rename -enumerate
```
<img width="1920" height="922" alt="w3-10" src="https://github.com/user-attachments/assets/647842c9-36b8-49e8-86e4-c7e867e124d3" />

---

### **Step 7: Check Statistics**
```yosys
stat
```
<img width="1920" height="922" alt="w3-11" src="https://github.com/user-attachments/assets/c7ebc41c-b2fa-404e-a284-9f76ea8738cc" />
<img width="1920" height="922" alt="w3-12" src="https://github.com/user-attachments/assets/c7cd50cc-31c9-465a-88c9-2cc850beffd6" />
<img width="1920" height="922" alt="w3-13" src="https://github.com/user-attachments/assets/3b1e07aa-b896-475a-a9bf-044547714303" />

---


### **Step 8: Write the Synthesized Netlist**
```yosys
write_verilog -noattr output/post_synth_sim/vsdbabysoc.synth.v
```

<img width="1920" height="269" alt="w313" src="https://github.com/user-attachments/assets/39bd9e8e-d959-466a-8578-f42007a2e330" />

---

## POST_SYNTHESIS SIMULATION AND WAVEFORMS
---

### **Step 1: Compile the Testbench**
Run the following `iverilog` command to compile the testbench:
```bash
iverilog -o /home/mohan/Desktop/vlsi/VSDBabySoC/output/post_synth_sim/post_synth_sim.out \
-DPOST_SYNTH_SIM -DFUNCTIONAL -DUNIT_DELAY=#1 \
-I /home/mohan/Desktop/vlsi/VSDBabySoC/src/include \
-I /home/mohan/Desktop/vlsi/VSDBabySoC/src/module \
-I /home/mohan/Desktop/vlsi/VSDBabySoC/src/gls_model \
/home/mohan/Desktop/vlsi/VSDBabySoC/src/module/testbench.v

```
---
### **Step 2: Navigate to the Post-Synthesis Simulation Output Directory**
```bash
cd output/post_synth_sim/
```
---
### **Step 3: Run the Simulation**

```bash
./post_synth_sim.out
```
---
### **Step 4: View the Waveforms in GTKWave**

```bash
gtkwave post_synth_sim.vcd
```
<img width="1920" height="922" alt="w314" src="https://github.com/user-attachments/assets/866b1d27-d5df-4890-84e2-9f9e14b0d55d" />
<img width="1920" height="922" alt="ps gtk" src="https://github.com/user-attachments/assets/75d3c6d4-4d5d-4299-88d3-341255611cbe" />

---
