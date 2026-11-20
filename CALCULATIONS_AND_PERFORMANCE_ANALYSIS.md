# 12T SRAM Calculations and Performance Analysis

This document explains the equations and Cadence workflow for calculating SRAM device parameters, stability, timing, power, and layout parasitics.

> **Important project-status statement:** This is a calculation guide. It does not mean that all listed measurements have already been completed. Enter numerical values only after obtaining them from the actual Virtuoso schematic, PDK models, transient waveforms, and extraction reports. Do not report illustrative values as project results.

## 1. Design parameters to collect

| Parameter | Symbol | Source |
|---|---|---|
| Supply voltage | VDD | Testbench voltage source |
| Ground | VSS | Testbench |
| Process corner and temperature | — | PDK / simulation setup |
| NMOS width and length | Wn, Ln | Schematic properties |
| PMOS width and length | Wp, Lp | Schematic properties |
| Output / bit-line load | CL | Testbench |
| Input rise/fall time | tr, tf | Pulse source |
| Switching frequency | f | Testbench |
| Transient measurement interval | T | ADE transient setup |

The exact 12T topology and signal names must be read from the actual schematic. Different 12T SRAM designs do not necessarily use the same transistor grouping or read/write path.

## 2. MOSFET hand calculations

For a long-channel MOSFET operating in saturation, the first-order square-law model is:

\[
I_D=\frac{1}{2}\mu C_{ox}\frac{W}{L}(V_{GS}-V_{TH})^2
\]

Overdrive voltage:

\[
V_{OV}=V_{GS}-V_{TH}
\]

Transconductance:

\[
g_m=\frac{2I_D}{V_{OV}}
\]

Approximate output resistance with channel-length modulation:

\[
r_o\approx\frac{1}{\lambda I_D}
\]

Here, μ is carrier mobility, Cox is oxide capacitance per unit area, W/L is aspect ratio, VTH is threshold voltage, and λ is the channel-length modulation parameter.

**Limitation:** These are first-order hand equations. Modern PDK compact models, especially for short-channel devices, are more complex. Use Cadence operating-point values and the actual model for final results.

## 3. SRAM butterfly curve and SNM

A butterfly curve is obtained by plotting the voltage-transfer characteristics of the two cross-coupled inverters, with one characteristic mirrored.

Static Noise Margin (SNM) is the side length of the largest square that can fit between the two butterfly lobes:

\[
SNM=\text{side length of the largest square}
\]

A larger SNM generally means better immunity to static noise. Clearly state the bias condition:
- **Hold SNM:** word line inactive.
- **Read SNM:** word line active under the read condition.
- **Write margin:** ability of the write circuitry to change the stored state.

A butterfly plot alone does not prove dynamic read/write functionality or access time. Record the bias conditions and axes used.

## 4. Propagation delay: tPHL, tPLH and average delay

Measure delay between a clearly defined input and output on a clearly defined circuit path.

- **tPHL:** input rising transition crosses 50% of its swing, then output falling transition crosses 50%.
- **tPLH:** input falling transition crosses 50% of its swing, then output rising transition crosses 50%.

For a rail-to-rail signal with VSS = 0:

\[
t_{PHL}=t(V_{out}=0.5V_{DD},\ falling)-t(V_{in}=0.5V_{DD},\ rising)
\]

\[
t_{PLH}=t(V_{out}=0.5V_{DD},\ rising)-t(V_{in}=0.5V_{DD},\ falling)
\]

Average propagation delay:

\[
t_p=\frac{t_{PHL}+t_{PLH}}{2}
\]

For non-zero VSS, use the midpoint \((V_{DD}+V_{SS})/2\), or the midpoint of each signal's actual swing if it is not rail-to-rail.

### SRAM-specific caution

State exactly which path is measured:
- Word-line activation to valid read output.
- Bit-line change to output.
- Write-data / write-control activation to the stored node changing.
- A separate inverter path (label this only as inverter characterization).

Do not call an arbitrary inverter delay “SRAM read access time.”

### Cadence procedure

1. Build a transient testbench for one defined read or write operation.
2. Apply the required precharge, word-line, bit-line and data conditions for the actual topology.
3. Run transient analysis for the complete operation.
4. Plot input and output on the same time axis.
5. Use waveform cursors or calculator measurements to find the 50% crossings.
6. Subtract the input crossing time from the output crossing time.
7. Repeat for the opposite transition and required operating cases.
8. Record load, supply, temperature, process corner and input slew.

## 5. Read and write access time

Read access time should use a defined start event and output-valid event. A common definition is:

\[
t_{read}=t(\text{valid read output})-t(\text{word line active})
\]

Specify the output-valid threshold (for example, 50% of the output swing or a defined logic-level criterion).

Write time can be measured as:

\[
t_{write}=t(\text{stored node reaches required new logic level})-t(\text{write begins})
\]

The start event and required final logic level must be defined. Read/write access time is not automatically the same as inverter tPHL/tPLH.

## 6. Power calculations

### 6.1 Dynamic switching power

A common first-order estimate is:

\[
P_{dynamic}=\alpha C_{eff}V_{DD}^{2}f
\]

where α is switching activity, Ceff is effective switched capacitance, VDD is supply voltage and f is switching frequency. State the activity-factor convention used; conventions differ on whether a full charge/discharge cycle is counted as one event or two transitions.

For one full 0-to-VDD charge event, energy drawn from the supply is approximately:

\[
E_{charge}=C_{eff}V_{DD}^{2}
\]

The capacitor stores \(\frac{1}{2}C_{eff}V_{DD}^{2}\); the other half is dissipated in the ideal charging resistance.

### 6.2 Average power from transient supply current

The general expression is:

\[
P_{avg}=\frac{1}{T}\int_{t_0}^{t_0+T}V_{DD}(t)I_{DD}(t)\,dt
\]

For a constant supply:

\[
P_{avg}=V_{DD}\times I_{DD,avg}
\]

Check the simulator's current sign convention. If current delivered by the source appears negative, use its magnitude or reverse the sign consistently.

### 6.3 Static and total power

Approximate leakage power:

\[
P_{static}\approx V_{DD}I_{leak}
\]

Total average power can include dynamic, short-circuit and leakage components:

\[
P_{total}=P_{dynamic}+P_{short-circuit}+P_{static}
\]

For a measured result, integrate the supply current over a stated interval. Clarify whether the measurement includes one cell or the entire 4×1 array, precharge circuits, write drivers and output load.

## 7. RC parasitic extraction and Elmore delay

Physical interconnects, contacts and vias add parasitic resistance and capacitance. A simple lumped first-order time constant is:

\[
\tau=RC
\]

For an RC tree, the Elmore delay to a selected node is:

\[
t_{Elmore}=\sum_i R_iC_{downstream,i}
\]

Each resistance is multiplied by the capacitance downstream of that resistance along the path to the destination. This is an approximation for an RC tree, not a replacement for extracted-view simulation.

### Extraction workflow

1. Complete the physical layout.
2. Run DRC and resolve design-rule violations.
3. Run LVS and resolve schematic/layout mismatches.
4. Confirm that the correct PDK technology files and parasitic extraction tool are installed and configured.
5. Run parasitic extraction using the flow supported by the lab's Virtuoso/PDK version.
6. Generate the extracted view or extracted netlist.
7. Simulate the extracted design with the same testbench used for schematic simulation.
8. Compare the two sets of results.

A layout screenshot by itself is not evidence that RC extraction has been completed. DRC/LVS success and extraction-tool availability must be confirmed.

## 8. Schematic versus extracted comparison

| Metric | Schematic result | Extracted result | Difference |
|---|---:|---:|---:|
| Read access time | Measure | Measure | Calculate |
| Write time | Measure | Measure | Calculate |
| tPHL | Measure | Measure | Calculate |
| tPLH | Measure | Measure | Calculate |
| Average power | Measure | Measure | Calculate |

Percentage change:

\[
\%\,change=\frac{X_{extracted}-X_{schematic}}{X_{schematic}}\times100
\]

Use the same supply, corner, temperature, input slew, load and measurement definitions for both runs. Do not calculate percentage change if the schematic baseline is zero.

## 9. Result record sheet

For every reported result, record:
- Metric and exact start/end events.
- Formula, substituted values and units.
- VDD, temperature, process corner and model/PDK.
- Device dimensions and load.
- Testbench and waveform screenshot.
- Cursor or calculator readings.
- Whether it is hand-calculated, schematic-simulated or post-layout extracted.

| Item | Current evidence/status |
|---|---|
| 12T cell schematic | Included in repository |
| 4×1 organization | Included in repository |
| Butterfly plot | Included; verify bias condition and extract numeric SNM |
| Functional 4×1 read waveform | Not yet documented as verified |
| Functional 4×1 write waveform | Not yet documented as verified |
| tPHL / tPLH | Pending defined path and transient measurements |
| Read / write access time | Pending functional transient measurements |
| Average power | Pending supply-current measurement |
| RC extraction | Pending DRC/LVS status and tool availability |
| Post-layout comparison | Pending extracted-view simulation |

## 10. Evidence to add before claiming measured performance

Add screenshots/reports for:
1. Actual 12T cell and 4×1 schematics with signal names.
2. Read and write transient waveforms with testbench conditions.
3. Delay measurements showing input/output 50% crossings.
4. Supply-current waveform and average-power calculation.
5. DRC and LVS reports showing the actual final status.
6. Parasitic extraction report and extracted-view simulation.

**Final note:** This file is a learning and calculation reference. Replace every “measure” or “pending” entry with project-specific values only after you have generated and checked the corresponding Cadence result.
