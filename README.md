# Instructions on how to use/test Circuit Training (CT) in OpenLane flow

## Step 1. Software installation

### Installing CT with convertors

In order to use CT in the OpenLane flow, you need [converters](https://github.com/RustamC/circuit_training/tree/with_convertors/circuit_training/convertor)[^1]. 
Then CT version with the converters already included is installed using the following commands:
```bash
git clone https://github.com/RustamC/circuit_training.git
cd circuit_training
git fetch && git checkout -b wc origin/with_convertors
docker build --tag circuit_training:wc -f tools/docker/ubuntu_circuit_training tools/docker/
```

### Installing OpenLane

[Standard instructions](https://github.com/The-OpenROAD-Project/OpenLane#installation-the-short-version)

## Step 2. CT input preparation

### Initial LED/DEF generation

Before running CT one needs to run OpenLane flow to generate initial placement using the following commands:
```bash
cd OpenLane
./flow.tcl -design manual_macro_placement_test -tag ct -to placement
exit
```

We run the flow until placement to generate usable LED/DEF as fast as possible and the placement file contains only macros placement. 
You can change `-to placement` to `-to cts` and generate the solution containing both macros and standard cells placement.

After running these commands in `designs/manual_macro_placement_test/runs/ct` you will find the required files:
1) DEF file: `results/tmp/placement/5-global.macro_placement.def` (with std cells: `results/tmp/placement/9-global.def`),
2) LEF file: `results/tmp/merged.nom.lef`,
3) LIB file: `results/tmp/synthesis/1-sky130_fd_sc_hd_tt_025C_1v80.no_pg.lib`

### LEF/DEF conversion into Protobuf

## Step 3. Running CT

## Step 4. Conversion from CT placement (\*.plc) into OpenLane macro placement (\*.cfg)

## Step 5. Running OpenLane with CT macro placement

[^1]: The LEF/DEF to ProtoBuf converter and the ProtoBuf parser are taken from the [MacroPlacement](https://github.com/TILOS-AI-Institute/MacroPlacement) project with slight modifications.
