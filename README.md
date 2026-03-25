# soc-design-and-planning-vsd
RTL-to-GDSII physical design flow using OpenLANE &amp; Sky130 PDK | VSD SoC Design and Planning Workshop.
# Digital VLSI SoC Design and Planning — RTL to GDSII

> A 2-week hands-on workshop on complete RTL-to-GDSII flow for digital VLSI SoC design,
> organised by **VSD (VLSI System Design)** in collaboration with **NASSCOM**.
> This repository documents my learning, lab outputs, and key takeaways from each day.

## Day 1 — Inception of Open-Source EDA, OpenLANE & Sky130 PDK

#### Understanding the Chip Package

When we look at any embedded board and point to what we call the "chip," we're actually looking at the **package** — a protective casing around the actual silicon die. The real chip sits in the centre of this package and communicates with the outside world via **wire bonding** — tiny wires that connect the chip's pads to the package pins.

#### Inside the Chip: Core, Pads, and Die

Zooming into the chip itself, all signals between the chip and the external world pass through **pads** placed around the periphery. The region enclosed by the pads is called the **core** — this is where all the actual digital logic lives. Together, the core and the pads form the **die**, which is the fundamental unit of chip manufacturing.

- **Foundry** — the place where chips are physically manufactured
- **Foundry IPs** — IP blocks that require specialized process knowledge to implement (e.g., PLLs, SRAMs)
- **Macros** — reusable, purely digital logic blocks

#### From Software to Silicon — The ISA Bridge

A C program running on a chip goes through a multi-layer transformation:

1. The C code is compiled into **RISC-V assembly** (or another ISA)
2. The assembler converts it to **binary machine code (0s and 1s)**
3. This binary pattern needs an **RTL implementation** of the ISA
4. The RTL gets synthesized and goes through the full **PnR (Place and Route)** flow to become a physical layout

The system software stack (OS → Compiler → Assembler) acts as the bridge between what the programmer writes and what the hardware executes.

#### Why Open-Source EDA Matters

For a fully open-source ASIC design flow, three things are needed:

1. **RTL Designs** (e.g., from opencores.org)
2. **EDA Tools** (synthesis, P&R, verification)
3. **PDK Data** (process-specific design rules, standard cell libraries)

Historically, PDKs were proprietary and distributed only under NDAs, making chip design inaccessible to most people. This changed in **June 2020**, when Google collaborated with SkyWater Technology to release the **Sky130 PDK** as the world's first open-source process design kit — a massive milestone for the VLSI community.

#### OpenLANE and the Automated RTL to GDSII Flow

**OpenLANE** is an open-source flow built on top of multiple EDA tools that automates the journey from an RTL netlist all the way to the final GDSII layout file. It uses:

| Stage | Tool(s) Used |
|---|---|
| Synthesis | Yosys, ABC |
| Floorplan & PDN | OpenROAD |
| Placement | OpenROAD |
| CTS | TritonCTS |
| Routing | FastRoute, TritonRoute |
| SPEF Extraction | OpenRCX |
| GDS Streaming | Magic, KLayout |
| Timing Analysis | OpenSTA |
| DRC & LVS | Magic, Netgen |

### Lab — Running OpenLANE for `picorv32a`

#### Setting Up and Invoking OpenLANE

The very first step is to navigate to the OpenLANE working directory and launch the tool in **interactive mode**, which lets us run each stage step-by-step.

```bash
cd /home/vscode/Desktop/OpenLane
make mount
./flow.tcl -interactive
package require openlane 1.0.2
```

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/f83c45ec488f339de18b7ae7c4b3e916195f4a10/images/01_syn.png)

#### Preparing the Design

Before running synthesis, we prepare the design to merge the cell LEF and technology LEF files, and set up the run directory.

```tcl
prep -design picorv32a
```

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/02_syn1.png) -->

#### Running Synthesis

```tcl
run_synthesis
```
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/03_syn2.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/04_syn3.png)

After synthesis completes, we can calculate the **flop ratio** — a useful sanity check:

```
Flop Ratio = (No. of D Flip-Flops) / (Total No. of Cells)
           = 1613 / 15762
           ≈ 0.1023  →  ~10.23%
```
## Day 2 — Floorplanning and Introduction to Library Cells

#### Chip Floorplanning — Core Area and Utilisation

Floorplanning is about deciding where everything goes on the chip. Two key parameters drive this:

- **Utilisation Factor** = (Area occupied by Netlist) / (Total Core Area)
  - A utilisation of 0.5–0.6 is typical — you want room for buffers, routing, etc.
- **Aspect Ratio** = Height / Width of the core
  - A ratio of 1 means a square; anything else is a rectangle.

#### Pre-Placed Cells and Decoupling Capacitors

**Pre-placed cells** (like memories, PLLs, and complex IP blocks) are fixed in position before automated placement runs. Their location is determined manually based on connectivity and power intent.

**Decoupling capacitors** are placed around pre-placed cells to act as local charge reservoirs — they compensate for voltage drops caused by switching activity and ensure these blocks see clean power.

#### Power Planning — Mesh vs Ring

A good power grid uses both **power rings** around the core and a **power mesh** across the chip. Multiple VDD and VSS rails are distributed in both metal layers so that every standard cell has a nearby power tap, minimising IR drop and electromigration risk.

#### Pin Placement and Logical Cell Blockage

Input and output pins are placed along the chip boundary. The relative placement of pins is guided by connectivity — a pin that drives logic deep in the core should be closer to that logic. The area between the core and the die boundary (I/O ring area) is blocked from automated cell placement to reserve it for pin buffers and ESD cells.

### Lab — Floorplan and Placement

#### Running Floorplan

```tcl
run_floorplan
```
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/05_floorplan.png)

After this completes, we can inspect the DEF file that was generated:

```bash
cd results/floorplan/
less picorv32a.def
```

#### Viewing the Floorplan in Magic

```bash
magic -T /home/vsduser/Desktop/OpenLane/designs/picorv32a/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.nom.lef \
      def read picorv32a.def &
```
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/06_floorplan1.png)

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/07_floorplan2.png)

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/08_floorplan3.png)

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/11_floorpln.png)

#### Running Placement

```tcl
run_placement
```
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/12_placement.png)

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/13_placement1.png)

Standard cells legally placed

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/14_placement3.png)

## Day 3 — Design and Characterisation of Library Cells using Magic & ngspice

#### CMOS Inverter — SPICE Deck

To characterise a standard cell, we write a SPICE netlist describing the PMOS and NMOS transistors along with their W/L ratios, supply voltage, input stimulus, and load capacitance.

Key parameters we extract from simulation:

- **Rise time** — 20% to 80% of output rising edge
- **Fall time** — 80% to 20% of output falling edge
- **Propagation delay** — 50% input to 50% output

#### 16-Mask CMOS Fabrication Process (Brief Overview)

The chip fabrication follows a sequence of about 16 mask steps:

1. Substrate selection (p-type, high resistivity)
2. Active region creation (field oxidation + Si3N4 mask)
3. N-well and P-well formation (ion implantation)
4. Gate oxide growth
5. Polysilicon gate deposition
6. Source/Drain implantation (LDD + halo)
7. Contacts and metal layers
8. Final passivation

### Lab — Cloning and Characterising a Custom Inverter Cell

#### Cloning the Standard Cell Repository

```bash
git clone https://github.com/nickson-jose/vsdstdcelldesign.git
```

```bash
magic -T sky130A.tech sky130_inv.mag &
```
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/15_day3.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/16_inv.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/17_inv2.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/18_inv3.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/19_inv4.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/20_inv5.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/21_inv6.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/22_inv7.png)

#### Extracting SPICE Netlist from Magic

Inside the tkcon console:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/24_inv9.png)
Screenshot of created spice file
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/25_inv10.png)

Editing the spice model file for analysis through simulation.

Measuring unit distance in layout grid
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/26_inv11.png)
Final edited spice file ready for ngspice simulation
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/27_inv12.png)
#### Running ngspice Simulation

```bash
ngspice sky130_inv.spice
```

```ngspice
plot y vs time a
```
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/28_inv13.png)
Screenshot of generated plot
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/29_inv14.png)
From the waveform, measure rise time, fall time, and propagation delay values.
Rise transition time calculation

Rise transition time = Time taken for output to rise to 80% - Time taken for output to rise to 20%

20% of output = 660 mV

80% of output = 2.64 V
Fall transition time calculation

Fall transition time = Time taken for output to fall to 20% - Time taken for output to fall to 80%

20% of output = 660 mV

80% of output = 2.64 V

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/30_inv15.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/31_inv16.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/32_inv17.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/33_inv18.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/34_inv19.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/35_inv20.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/36_inv21.png)

Incorrectly implemented poly.9 simple rule correction

Screenshot of poly rules
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/37_sky.png)
Incorrectly implemented poly.9 rule no drc violation even though spacing < 0.48u
Find problem in the DRC section of the old magic tech file for the skywater process and fix them.

Link to Sky130 Periphery rules: https://skywater-pdk.readthedocs.io/en/main/rules/periphery.html

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/38_sky1.png)

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/39_sky2.png)

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/40_sky4.png)

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/41_sky5.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/42_sky6.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/43_sky7.png)

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/44_sky8.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/45_sky9.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/46_sky10.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/47_sky11.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/48_sky12.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/49_sky13.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/50_sky14.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/51_sky15.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/54_sky18.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/53_sky17.png)

## Day 4 — Pre-Layout Timing Analysis and Clock Tree Synthesis

#### LEF Files and Guidelines for Standard Cell Ports

Before a custom cell can be used inside OpenLANE, it needs a proper **LEF file** describing its physical boundary, pin locations, and metal layer information. Port definitions must follow two important rules:

- All input and output ports must lie on the **intersection of horizontal and vertical routing tracks**
- The cell **width must be an odd multiple** of the track pitch, and height must be an odd multiple of the vertical track pitch

#### Static Timing Analysis (STA) Concepts

**Setup slack** = Data Required Time − Data Arrival Time (must be ≥ 0)

Key sources of uncertainty accounted for in STA:

- **OCV (On-Chip Variation)** — process/voltage/temperature variation modelled using derate factors
- **Clock Uncertainty** — jitter and skew margins added to timing paths
- **CRPR (Clock Reconvergence Pessimism Removal)** — removes artificial pessimism when launch and capture paths share clock buffers

#### Clock Tree Synthesis (CTS)

CTS builds a balanced tree of clock buffers to distribute the clock signal across the chip with minimal skew. After CTS:

- Hold timing must be re-checked (CTS inserts buffers that add real delay)
- Setup timing should be re-verified post-CTS as clock paths have changed

### Lab — Custom Cell Integration and STA with OpenSTA
Screenshot of tracks.info of sky130_fd_sc_hd
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/55_day4.1.png)
#### Get syntax for grid command
help grid

#### Set grid values accordingly
grid 0.46um 0.34um 0.23um 0.17um
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/57_day4.4.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/56_day4.2.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/58_day4.5.png)
Generate lef from the layout.

Command for tkcon window to write lef
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/59_day4.6.png)

Screenshot of newly created lef file
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/60_day4.7.png)

Copy the newly generated lef and associated required lib files to 'picorv32a' design 'src' directory.

Commands to copy necessary files to 'picorv32a' design 'src' directory
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/61_day4.8.png)

#### Editing `config.tcl` to Include Custom Cell

```tcl
set ::env(LIB_SYNTH)      "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(EXTRA_LEFS)     [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/63_day4.10.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/64_day4.11.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/65_day4.15.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/66_day4.12.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/67_day4.13.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/68_day4.16.png)
Screenshot of placement def in magic
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/69_day4.17.png)
Screenshot of custom inverter inserted in placement def with proper abutment
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/70_day4.18.png)
#### Command to view internal connectivity layers
expand
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/71_day4.19.png)

#### Running OpenSTA (Pre-CTS Timing)
Newly created pre_sta.conf for STA analysis in openlane directory
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/72_day4.20.png)
Newly created my_base.sdc for STA analysis in openlane/designs/picorv32a/src directory based on the file openlane/scripts/base.sdc
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/73_day4.22.png)

```bash
sta pre_sta.conf
```
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/74_day4.21.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/75_day4.23.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/76_day4.24.png)

#### Running CTS

```tcl
run_cts
```
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/78_cts1.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/79_cts2.png)

#### Command to run OpenROAD tool
openroad

Reading lef file:
read_lef /OpenLane/designs/picorv32a/runs/24-03_10-03/tmp/merged.nom.lef

Reading def file:
read_def /OpenLane/designs/picorv32a/runs/24-03_10-03/results/cts/picorv32a.def

Creating an OpenROAD database to work with:
write_db pico_cts.db

Loading the created database in OpenROAD:
read_db pico_cts.db

Read netlist post CTS:
read_verilog /OpenLane/designs/picorv32a/runs/24-03_10-03/results/synthesis/picorv32a.v

Read library for design:
read_liberty $::env(LIB_SYNTH_COMPLETE)

Link design and library:
link_design picorv32a

Read in the custom sdc we created:
read_sdc /OpenLane/designs/picorv32a/src/my_base.sdc

Setting all cloks as propagated clocks:
set_propagated_clock [all_clocks]

Check syntax of 'report_checks' command:
help report_checks

Generating custom timing report:
report_checks -path_delay min_max -fields {slew trans net cap input_pins} -format full_clock_expanded -digits 4

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/80_cts4.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/81_cts5.png)
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/82_cts6.png)

Exit to OpenLANE flow
exit
---

## Day 5 — Final RTL to GDSII using TritonRoute & OpenSTA

#### Routing — Global vs Detailed

Routing happens in two stages:

1. **Global Routing (FastRoute)** — divides the chip into routing regions and finds approximate paths for each net, respecting layer and congestion constraints
2. **Detailed Routing (TritonRoute)** — takes the global routing guides and assigns exact wire segments, vias, and metal tracks while adhering to DRC rules

#### SPEF and Post-Route STA

After routing, parasitics (resistance and capacitance of actual wires) are extracted into a **SPEF (Standard Parasitic Exchange Format)** file. These parasitics are then back-annotated into the netlist and STA is re-run for final sign-off timing.


### Lab — Power Distribution, Routing, and GDSII Generation

#### Generating Power Distribution Network

```tcl
gen_pdn
```
#### Running Routing

```tcl
run_routing
```
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/83_rout1.png)


Screenshots of PDN def
![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/85_rout3.png)

![image alt](https://github.com/gowdanayu/soc-design-and-planning-vsd/blob/2983478f1676f67bec4ff6805d9409ff08dd0498/images/86_rout4.png)

Common violations to look out for:

- *Min spacing violations* – two wires too close on the same layer
- *Antenna violations* – long metal segments accumulating charge during etch (can damage gate oxide)
  - Fix: insert antenna diodes or use jumper vias to a higher layer

-----


### Generating GDSII

tcl
run_magic


This invokes Magic to stream out the final *GDSII file* — the actual data sent to the foundry for fabrication. You can also generate the *LEF* for the full chip and run a final *LVS* (Layout vs Schematic) check with Netgen to verify that what was routed matches the synthesized netlist.

-----

## Tools & Environment

| Tool | Purpose |
|---|---|
| **OpenLANE** | RTL-to-GDSII automation flow |
| **Yosys** | RTL synthesis |
| **OpenROAD** | Floorplan, Placement, CTS, Routing |
| **Magic** | Layout editor, DRC, LVS |
| **OpenSTA** | Static Timing Analysis |
| **ngspice** | SPICE simulation |
| **TritonRoute** | Detailed routing |
| **Netgen** | LVS (Layout vs Schematic) |
| **Sky130 PDK** | SkyWater 130nm open-source PDK |

---

## Key Learnings

- Understood how a chip moves from an idea (RTL) to a manufacturable file (GDSII) using a fully open-source toolchain
- Got hands-on with floorplanning, placement, CTS, and routing for the `picorv32a` RISC-V core
- Learned how to characterise custom standard cells and integrate them into an existing flow
- Gained practical experience with STA concepts — setup/hold slack, OCV, CRPR — using OpenSTA
- Understood how parasitics from post-route SPEF extraction affect timing sign-off

---


## Acknowledgements

A huge thank you to *Kunal Ghosh* (Co-founder, VSD Corp. Pvt. Ltd.) and *Nickson P Jose* (Physical Design Engineer, Intel) for putting together such a well-structured and genuinely practical workshop. Running a real CPU from RTL to GDSII using nothing but open-source tools is something I didn’t expect to be possible — and yet here we are.
- **Kunal Ghosh** — Co-founder, VSD (VLSI System Design)
- **Nickson Jose** — for the `vsdstdcelldesign` repository used in Day 3 labs
- **NASSCOM** — for facilitating this workshop program

---

## References

- [VSD SoC Design Workshop](https://www.vlsisystemdesign.com/)
- [OpenLANE GitHub](https://github.com/The-OpenROAD-Project/OpenLane)
- [SkyWater Sky130 PDK](https://github.com/google/skywater-pdk)
- [vsdstdcelldesign](https://github.com/nickson-jose/vsdstdcelldesign)
Documented by Nayana | VTU Electronics & Communication Engineering | VLSI & Physical Design Enthusiast
