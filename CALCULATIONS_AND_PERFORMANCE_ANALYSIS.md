# 12T SRAM Calculations and Performance Analysis

This document explains the calculations, the actual butterfly-curve screenshot in this repository, and the Cadence workflow to obtain further results.

## 1. Reading the uploaded butterfly graph

The graph below is the actual image currently stored in this repository.

![12T SRAM butterfly / DC transfer plot](Images/BUTTERFLY%20GRAPH.jpeg)

### What can be read directly from this screenshot

- The plot is labelled as a DC analysis.
- The swept voltage is shown from **0 V to 1.8 V**.
- Both horizontal and vertical axes are voltage axes, in volts.
- Two transfer characteristics are plotted, labelled in the legend as v vs y and y vs x.
- The curves show the voltage-transfer behaviour of the two coupled inverter paths.

### How SNM is calculated from the graph

1. Export the DC sweep data for both transfer curves from Cadence (CSV or text).
2. Confirm which curve is the first inverter transfer function and which is the mirrored second transfer function.
3. Put both curves on the same voltage axes and ensure the second characteristic is mirrored correctly.
4. Find the largest square that fits inside each butterfly lobe.
5. The side length of the limiting largest square is the SNM.

$$SNM = \text{side length of the largest square inside the butterfly}$$

## 2. Parameters to collect from the actual Cadence design

| Parameter | Symbol | Where to find it |
|---|---|---|
| Supply voltage | VDD | Testbench voltage source |
| Ground voltage | VSS | Testbench |
| Process corner and temperature | — | ADE simulation setup / PDK |
| NMOS width and length | Wn, Ln | Schematic instance properties |
| PMOS width and length | Wp, Lp | Schematic instance properties |
| Output / bit-line load | CL | Testbench |
| Input rise and fall times | tr, tf | vpulse properties |
| Switching frequency | f | Testbench pulse period |
| Measurement interval | T | Transient analysis setup |

## 3. MOSFET hand calculations

For a long-channel MOSFET in saturation, the first-order square-law approximation is:

$$I_D = \frac{1}{2}\mu C_{ox}\frac{W}{L}(V_{GS}-V_{TH})^2$$

Overdrive voltage:

$$V_{OV}=V_{GS}-V_{TH}$$

Transconductance:

$$g_m=\frac{2I_D}{V_{OV}}$$

Approximate output resistance with channel-length modulation:

$$r_o\approx\frac{1}{\lambda I_D}$$

Here, μ is carrier mobility, Cox is oxide capacitance per unit area, W/L is the transistor aspect ratio, VTH is threshold voltage, and λ is the channel-length modulation parameter.

These are first-order hand calculations. Modern PDK compact models can differ, particularly for short-channel devices. Use Cadence operating-point results and the actual PDK model for final values.

## 4. Propagation delay: tPHL and tPLH

Measure delay between a clearly defined input and output on a defined circuit path. For rail-to-rail signals with VSS = 0:

$$t_{PHL}=t(V_{out}\text{ crosses }0.5V_{DD}\text{ falling})-t(V_{in}\text{ crosses }0.5V_{DD}\text{ rising})$$

$$t_{PLH}=t(V_{out}\text{ crosses }0.5V_{DD}\text{ rising})-t(V_{in}\text{ crosses }0.5V_{DD}\text{ falling})$$

Average propagation delay:

$$t_p=\frac{t_{PHL}+t_{PLH}}{2}$$

For a non-zero VSS, use the midpoint of the signal swing, or the appropriate threshold defined for the measurement.

**SRAM caution:** State which path is measured: word-line-to-read-output, bit-line-to-output, write-control-to-stored-node, or a separate inverter path. Do not label an arbitrary inverter delay as SRAM read access time.

### Cadence measurement steps

1. Open the correct testbench and identify the input and output nets.
2. Set transient analysis long enough to capture the transition.
3. Run the simulation and plot both signals on the same time axis.
4. Determine the 50% voltage crossing of the input and output.
5. Measure the time difference using waveform cursors or ADE calculator functions.
6. Repeat for the opposite transition.
7. Record supply, load, temperature, process corner and input slew.

## 5. Read and write access time

A common read-access definition is:

$$t_{read}=t(\text{valid read output})-t(\text{word line active})$$

A write-time definition is:

$$t_{write}=t(\text{stored node reaches required new logic level})-t(\text{write begins})$$

Define the exact start event, output-valid threshold and final stored-node threshold in the testbench. These operation-level measurements are not automatically equal to inverter tPHL/tPLH.

## 6. Power calculations

First-order dynamic switching power:

$$P_{dynamic}=\alpha C_{eff}V_{DD}^{2}f$$

where α is switching activity, Ceff is effective switched capacitance, VDD is supply voltage and f is switching frequency. State the activity-factor convention used.

For one full 0-to-VDD charging event, energy drawn from the supply is approximately:

$$E_{charge}=C_{eff}V_{DD}^{2}$$

The capacitor stores one-half of this energy; the other half is dissipated in the ideal charging resistance.

Average power from transient supply current:

$$P_{avg}=\frac{1}{T}\int_{t_0}^{t_0+T}V_{DD}(t)I_{DD}(t)\,dt$$

For constant supply voltage:

$$P_{avg}=V_{DD}\times I_{DD,avg}$$

Check the simulator's current sign convention. If current delivered by the source appears negative, use its magnitude or reverse the sign consistently.

Approximate leakage power:

$$P_{static}\approx V_{DD}I_{leak}$$

Total average power can include dynamic, short-circuit and leakage components:

$$P_{total}=P_{dynamic}+P_{short-circuit}+P_{static}$$

For the measured result, integrate the supply current over a stated interval. Clarify whether the measurement includes one cell or the complete 4×1 array, precharge circuitry, write drivers and output load.

## 7. RC parasitic extraction and Elmore delay

Physical interconnects, contacts and vias add resistance and capacitance. A simple lumped time constant is:

$$\tau=RC$$

For an RC tree, the Elmore delay to a selected node is:

$$t_{Elmore}=\sum_i R_iC_{downstream,i}$$

Each resistance is multiplied by the capacitance downstream of that resistance along the path to the destination. This is an approximation for an RC tree.

### Extraction workflow

1. Complete the physical layout.
2. Run DRC and resolve design-rule violations.
3. Run LVS and resolve schematic/layout mismatches.
4. Confirm that the correct PDK technology files and parasitic extraction tool are available.
5. Run extraction using the flow supported by the lab's Virtuoso and PDK versions.
6. Generate the extracted view or extracted netlist.
7. Simulate the extracted design with the same testbench as the schematic run.
8. Compare the schematic and extracted results.

A layout screenshot alone does not prove RC extraction has been completed. DRC/LVS status and extraction-tool availability must be confirmed.

## 8. Schematic versus extracted comparison

| Metric | Schematic result | Extracted result | Difference |
|---|---:|---:|---:|
| Read access time | To measure | To measure | Calculate |
| Write time | To measure | To measure | Calculate |
| tPHL | To measure | To measure | Calculate |
| tPLH | To measure | To measure | Calculate |
| Average power | To measure | To measure | Calculate |

Percentage change:

$$\%\text{ change}=\frac{X_{extracted}-X_{schematic}}{X_{schematic}}\times100$$

Use identical supply, corner, temperature, input slew, load and measurement definitions for both runs.

## 9. What is currently evidenced in this repository

| Item | Status based on currently documented material |
|---|---|
| 12T cell schematic | Image included |
| 4×1 organization | Image included |
| DC transfer / butterfly-style plot | Image included; numerical SNM not established from screenshot |
| Functional 4×1 read waveform | Not yet documented as verified |
| Functional 4×1 write waveform | Not yet documented as verified |
| tPHL / tPLH | Requires defined path and transient measurement |
| Read / write access time | Requires functional transient measurement |
| Average power | Requires supply-current measurement |
| RC extraction | Requires confirmed DRC/LVS status and extraction flow |
| Post-layout comparison | Requires extracted-view simulation |

## 10. What to show the professor next

Prepare these evidence items in order:
1. The 12T schematic with transistor names and W/L values.
2. The 4×1 testbench with VDD, input sources and loads visible.
3. The DC transfer plot and exported DC data for numerical SNM.
4. Separate read and write transient waveforms.
5. 50% crossing measurements for delay.
6. Supply-current waveform and average-power calculation.
7. DRC/LVS reports, extraction report and extracted-view simulation if the lab supports them.

## 11. Worked numerical examples 

The following examples show how to substitute values and calculate the result. These assumed values are for learning only. They must not be presented as measurements from this 12T SRAM project.

### 11.1 Propagation delay example

Assume a transient waveform gives these 50% crossing times:

| Event | Time |
|---|---:|
| Input rising crosses 50% VDD | 1.000 ns |
| Output falling crosses 50% VDD | 1.180 ns |
| Input falling crosses 50% VDD | 2.000 ns |
| Output rising crosses 50% VDD | 2.240 ns |

$$t_{PHL}=1.180-1.000=0.180\text{ ns}=180\text{ ps}$$

$$t_{PLH}=2.240-2.000=0.240\text{ ns}=240\text{ ps}$$

$$t_p=\frac{180+240}{2}=210\text{ ps}$$

Interpretation: the illustrative falling delay is 180 ps, rising delay is 240 ps, and average propagation delay is 210 ps. Actual values must be measured from a specified SRAM signal path in Cadence.

### 11.2 Elmore delay example

Consider a simple two-resistor RC tree:

- R1 = 1 kΩ from source to intermediate node.
- R2 = 2 kΩ from intermediate node to output.
- C1 = 10 fF from intermediate node to ground.
- C2 = 20 fF from output node to ground.

For delay to the output, R1 sees both downstream capacitors, while R2 sees only C2:

$$t_{Elmore}=R_1(C_1+C_2)+R_2C_2$$

$$t_{Elmore}=1\text{ k}\Omega(10+20)\text{ fF}+2\text{ k}\Omega(20\text{ fF})$$

$$t_{Elmore}=30\text{ ps}+40\text{ ps}=70\text{ ps}$$


### 11.3 Dynamic power example

Assume only for this example:
- Activity factor α = 0.2.
- Effective switched capacitance Ceff = 20 fF.
- Supply VDD = 1.8 V.
- Switching frequency f = 100 MHz.

$$P_{dynamic}=\alpha C_{eff}V_{DD}^{2}f$$

$$P_{dynamic}=0.2(20\times10^{-15})(1.8)^2(100\times10^6)$$

$$P_{dynamic}=1.296\times10^{-6}\text{ W}=1.296\text{ µW}$$

Energy drawn from the supply for one full charging event:

$$E_{charge}=C_{eff}V_{DD}^{2}=20\text{ fF}\times(1.8)^2=64.8\text{ fJ}$$


### 11.4 Average power from supply current example

Assume a constant supply of 1.8 V and a hypothetical measured average supply-current magnitude of 120 µA:

$$P_{avg}=V_{DD}I_{DD,avg}$$

$$P_{avg}=1.8\times120\text{ µA}=216\text{ µW}$$

The 120 µA value is hypothetical. Your actual value must come from the supply-current waveform in Cadence.

### 11.5 Static leakage power example

Assume a hypothetical standby leakage current of 2 µA at 1.8 V:

$$P_{static}=V_{DD}I_{leak}=1.8\times2\text{ µA}=3.6\text{ µW}$$

Measure actual standby current with the SRAM in a defined hold state.

### 11.6 Percentage change after extraction example

Assume a schematic delay of 200 ps and an extracted-layout delay of 230 ps:

$$\%\text{ change}=\frac{230-200}{200}\times100=15\%$$

The extracted delay is 15% higher in this comparision.

## 12. Which values can be calculated from the current GitHub graph?

From the static DC screenshot, we can identify the displayed sweep range (0 to 1.8 V) and the two plotted transfer characteristics. We cannot calculate numerical tPHL, tPLH, read/write access time, power, Elmore delay, or extracted RC from this graph.

| Requested result | Required data currently missing |
|---|---|
| Numerical SNM | Exported DC curve data or largest-square cursor measurements |
| tPHL / tPLH | Transient input/output waveforms and defined path |
| Read / write access time | Functional read/write transient waveforms |
| Dynamic power | Actual switched capacitance, activity and frequency, or transient supply-current waveform |
| Average / standby power | Supply voltage and measured average current |
| Elmore delay | Extracted RC tree: each resistance and downstream capacitance |
| Post-layout delay | Extracted view/netlist and transient simulation |
| Percentage timing change | Both schematic and extracted timing results |
