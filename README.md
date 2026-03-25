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

<!-- SCREENSHOT: Add terminal showing OpenLANE launch and interactive mode here -->

#### Preparing the Design

Before running synthesis, we prepare the design to merge the cell LEF and technology LEF files, and set up the run directory.

```tcl
prep -design picorv32a
```

<!-- SCREENSHOT: Add terminal output after prep command here -->

#### Running Synthesis

```tcl
run_synthesis
```
<!-- SCREENSHOT: Add synthesis completion terminal output here -->
<!-- SCREENSHOT: Add yosys synthesis report showing cell counts here -->

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

After this completes, we can inspect the DEF file that was generated:

```bash
cd results/floorplan/
less picorv32a.floorplan.def
```

#### Viewing the Floorplan in Magic

```bash
magic -T /home/vsduser/Desktop/work/tools/openlane_working_dir/pdks/sky130A/libs.tech/magic/sky130A.tech \
      lef read ../../tmp/merged.lef \
      def read picorv32a.floorplan.def &
```

<!-- SCREENSHOT: Add Magic layout view of floorplan here -->
<!-- SCREENSHOT: Add zoomed-in view showing standard cell rows and pin placement here -->

#### Running Placement

```tcl
run_placement
```

<!-- SCREENSHOT: Add Magic view of post-placement layout here -->

---

## Day 3 — Design and Characterisation of Library Cells using Magic & ngspice

### Theory

<details>
<summary><b>Click to expand</b></summary>

#### CMOS Inverter — SPICE Deck

To characterise a standard cell, we write a SPICE netlist describing the PMOS and NMOS transistors along with their W/L ratios, supply voltage, input stimulus, and load capacitance.

Key parameters we extract from simulation:

- **Rise time** — 20% to 80% of output rising edge
- **Fall time** — 80% to 20% of output falling edge
- **Propagation delay** — 50% input to 50% output

<!-- SCREENSHOT: Add ngspice simulation waveform for CMOS inverter here -->

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

<!-- SCREENSHOT: Add Magic view of sky130_inv inverter layout here -->

#### Extracting SPICE Netlist from Magic

Inside the tkcon console:

```tcl
extract all
ext2spice cthresh 0 rthresh 0
ext2spice
```

<!-- SCREENSHOT: Add extracted SPICE file content here -->

#### Running ngspice Simulation

```bash
ngspice sky130_inv.spice
```

```ngspice
plot y vs time a
```

<!-- SCREENSHOT: Add ngspice transient waveform plot here -->

From the waveform, measure rise time, fall time, and propagation delay values.

---

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

<!-- SCREENSHOT: Add timing path diagram (setup/hold) here -->

#### Clock Tree Synthesis (CTS)

CTS builds a balanced tree of clock buffers to distribute the clock signal across the chip with minimal skew. After CTS:

- Hold timing must be re-checked (CTS inserts buffers that add real delay)
- Setup timing should be re-verified post-CTS as clock paths have changed

### Lab — Custom Cell Integration and STA with OpenSTA

#### Editing `config.tcl` to Include Custom Cell

```tcl
set ::env(LIB_SYNTH)      "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(LIB_FASTEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__fast.lib"
set ::env(LIB_SLOWEST)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__slow.lib"
set ::env(LIB_TYPICAL)    "$::env(OPENLANE_ROOT)/designs/picorv32a/src/sky130_fd_sc_hd__typical.lib"
set ::env(EXTRA_LEFS)     [glob $::env(OPENLANE_ROOT)/designs/$::env(DESIGN_NAME)/src/*.lef]
```

#### Running OpenSTA (Pre-CTS Timing)

```bash
sta pre_sta.conf
```

<!-- SCREENSHOT: Add OpenSTA output showing setup slack report here -->

#### Running CTS

```tcl
run_cts
```

<!-- SCREENSHOT: Add post-CTS timing report here -->

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

<!-- SCREENSHOT: Add PDN generation terminal output here -->
<!-- SCREENSHOT: Add Magic view of power straps and rails here -->

#### Running Routing

```tcl
run_routing
```

<!-- SCREENSHOT: Add post-routing Magic layout view here -->
<!-- SCREENSHOT: Add DRC zero violations confirmation here -->

#### Post-Route Timing Analysis

```bash
openroad
read_lef /path/to/merged.lef
read_def /path/to/picorv32a.def
read_verilog /path/to/picorv32a.v
read_liberty /path/to/sky130_fd_sc_hd__typical.lib
read_spef /path/to/picorv32a.spef
report_checks -path_delay min_max
```

<!-- SCREENSHOT: Add final STA timing report showing WNS and TNS here -->

#### Final GDSII Output

```tcl
run_magic
run_magic_spice_export
run_magic_drc
run_netgen
run_magic_antenna_check
```

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

- **Kunal Ghosh** — Co-founder, VSD (VLSI System Design)
- **Nickson Jose** — for the `vsdstdcelldesign` repository used in Day 3 labs
- **Tim Edwards** — for Magic layout tool
- **NASSCOM** — for facilitating this workshop program

---

## References

- [VSD SoC Design Workshop](https://www.vlsisystemdesign.com/)
- [OpenLANE GitHub](https://github.com/The-OpenROAD-Project/OpenLane)
- [SkyWater Sky130 PDK](https://github.com/google/skywater-pdk)
- [vsdstdcelldesign](https://github.com/nickson-jose/vsdstdcelldesign)
