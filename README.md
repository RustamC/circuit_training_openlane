# Instructions on how to use/test Circuit Training (CT) in OpenLane flow

## Main steps overview
1. Install CT with convertors and OpenLane,
2. Run OpenLane with your design,
3. Convert LED/DEF into ProtoBuf,
    1. copy LEF/DEF/LIB/Verilog files into CT directory,
    2. create config file for convertor,
    3. run convertor and get flattened netlist in ProtoBuf format,
    4. run CT grouping to generate clustered netlist in ProtoBuf format.
4. Run CT with clustered netlist and get placement file (*.plc),
5. Convert placement (\*.plc) into macro placement for OpenLane (*.cfg).

## Step 1. Software installation

Consider that installation path is `/home/user/src`.

### Installing CT with convertors

In order to use CT in the OpenLane flow, you need [converters](https://github.com/RustamC/circuit_training/tree/with_convertors/circuit_training/convertor)[^1]. 
You can install CT with converters included using the following commands:
```bash
cd ~/src
git clone https://github.com/RustamC/circuit_training.git
cd circuit_training
git fetch && git checkout -b wc origin/with_convertors
docker build --tag circuit_training:wc -f tools/docker/ubuntu_circuit_training tools/docker/
```

### Installing OpenLane

[Standard instructions](https://github.com/The-OpenROAD-Project/OpenLane#installation-the-short-version)

## Step 2. CT input preparation

### Testing circuit

For our experiments we will be using `manual_macro_placement_test` with some [modifications](https://github.com/RustamC/OpenLane/tree/fpi/designs/test_macro):
1) design is renamed to `test_macro`,
2) added more macros,
3) manual macro placement disabled by default.

You can just copy [test_macro](https://github.com/RustamC/OpenLane/tree/fpi/designs/test_macro) into your `/home/user/src/OpenLane/designs` directory.

### Initial LED/DEF generation

Before running CT one needs to run OpenLane flow to generate initial placement using the following commands:
```bash
cd ~/src/OpenLane
./flow.tcl -design test_macro -tag ct -to placement
exit
```

We run the flow until placement to generate usable LED/DEF as fast as possible so the placement file contains only macros placement. 

After running these commands in `/home/user/src/OpenLane/designs/test_macro/runs/ct` you will find the required files:
1) DEF file: `results/tmp/placement/5-global.macro_placement.def`,
2) LEF file: `results/tmp/merged.nom.lef`,
3) LIB file: `results/tmp/synthesis/1-sky130_fd_sc_hd_tt_025C_1v80.no_pg.lib`,
4) Verilog gate-level netlist: `results/synthesis/test_macro.v`

## Step 3. LEF/DEF conversion into Protobuf

### Copy files

Copy all LEF/DEF/LIB/Verilog files mentioned earlier into a new directory: `/home/user/src/circuit_training/convertor/test_macro`.

### Prepare config file

Create `config.json` in `/home/user/src/circuit_training/convertor`:
```json
{
    "CONVERTOR": "d2p",
    "DESIGN": "test_macro",
    "NETLIST": "./test_macro/design/test_macro.v",

    "DEF": "./test_macro/design/test_macro.def",
    "LEFS": ["./test_macro/lefs/test_macro.lef"],
    "LIBS": ["./test_macro/libs/test_macro.lib"],
    "PB_FILE": "./test_macro/netlist.pb.txt",
    "PLC_FILE": "./test_macro/netlist.plc",
    "OPENROAD_EXE": "./utils/openroad",

    "MACRO_CFG": "./test_macro/test_macro.cfg"
}
```

JSON variables description:

* `CONVERTOR` - choose the convertor. Possible values:
  * `d2p` - converts LED/DEF into ProtoBuf,
  * `p2m` - converts ProtoBuf into macro placement for OpenLane (*.cfg),
  * `p2d` - converts ProtoBuf into LEF/DEF (actually, just updates coordinates in the original DEF). **WARNING**: Doesn't work at this moment.
* `DESIGN` - top module name.
* `NETLIST` - path to the gate-level Verilog.
* `DEF` - DEF file path.
* `LEF` - LEF file path.
* `LIBS` - array of LIB files paths.
* `PB_FILE` - ProtoBuf file path.
* `PLC_FILE` - placement file path.
* `OPENROAD_EXE` - OpenROAD executable path.
* `MACRO_CFG` - output macro placement file for OpenLane. Required only when `"CONVERTOR": "p2m"`

### Running convertor for flattened ProtoBuf

```bash
cd ~/circuit_training
docker run -it --rm -v $(pwd):/workspace --workdir /workspace circuit_training:wc bash
cd circuit_training/convertor
python3 circuit_training/convertor/convertor.py circuit_training/convertor/config.json
```

### Running CT for clustered ProtoBuf
(in the same Docker container)
```bash
export OUTPUT_DIR=/workspace/circuit_training/convertor/test_macro/out
export BLOCK_NAME=test_macro
export NETLIST_FILE=/workspace/circuit_training/convertor/netlist.pb.txt
export HMETIS_DIR=/usr/local/bin
export PLC_WRAPPER_MAIN=/usr/local/bin/plc_wrapper_main
python circuit_training/grouping/grouper_main.py \
--output_dir=$OUTPUT_DIR \
--netlist_file=$NETLIST_FILE \
--block_name=$BLOCK_NAME \
--hmetis_dir=$HMETIS_DIR \
--plc_wrapper_main=$PLC_WRAPPER_MAIN
```

In `/home/user/circuit_training/convertor/test_macro/out` you will find nested directories that contain `initial.plc` file.
The full path is also displayed during clustering execution (`Paritioned netlist: <full path to initial.plc>`).

P.S. CT uses hmetis for clustering. hmetis generates output file with name that containes the execution params & number of clusters. And CT uses this fact to search for output file. However, sometimes hmetis cannot generate clustered netlist with the given number of clusters and CT clustering fails. You can try to rerun clustering until hmetis generates the required clustering. Or just reduce the number of std cells clusters in [grouper.py](https://github.com/RustamC/circuit_training/blob/80fe78fc795d9390048aa795ce6feb3edf2f6706/circuit_training/grouping/grouper.py#L57) and rerun clustering.

## Step 4. Running CT

Now you are ready for running CT. It's a little bit different from running the standard flow. 

## Step 4. Conversion from CT placement (\*.plc) into OpenLane macro placement (\*.cfg)

## Step 5. Running OpenLane with CT macro placement

[^1]: The LEF/DEF to ProtoBuf converter and the ProtoBuf parser are taken from the [MacroPlacement](https://github.com/TILOS-AI-Institute/MacroPlacement) project with slight modifications.
