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

Consider that installation path is `/home/$USER/src`.

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

You can just copy [test_macro](https://github.com/RustamC/OpenLane/tree/fpi/designs/test_macro) into your `/home/$USER/src/OpenLane/designs` directory.

### Initial LED/DEF generation

Before running CT one needs to run OpenLane flow to generate initial placement using the following commands:
```bash
cd ~/src/OpenLane
make mount
./flow.tcl -design test_macro -tag ct
exit
```

One can run the flow until placement stage to generate usable LED/DEF as fast as possible so the placement file contains only macros placement. 

After running these commands in `/home/$USER/src/OpenLane/designs/test_macro/runs/ct` you will find the required files:
1) DEF file: `tmp/placement/7-macros_placed.def`,
2) LEF file: `tmp/merged.nom.lef`,
3) LIB file: `tmp/synthesis/1-sky130_fd_sc_hd_tt_025C_1v80.no_pg.lib`,
4) Verilog gate-level netlist: `results/synthesis/test_macro.v`

## Step 3. LEF/DEF conversion into Protobuf

### Copy files

Copy all LEF/DEF/LIB/Verilog files mentioned earlier into a new directory `/home/$USER/src/circuit_training/circuit_training/convertor/test_macro`:
1) DEF file `tmp/placement/7-macros_placed.def` -> `test_macro.def`,
2) LEF file `tmp/merged.nom.lef` -> `test_macro.lef`,
3) LIB file `tmp/synthesis/1-sky130_fd_sc_hd_tt_025C_1v80.no_pg.lib` -> `test_macro.lib`,
4) Verilog `results/synthesis/test_macro.v` -> `test_macro.v`

```bash
cd ~/src/OpenLane
mkdir ../circuit_training/circuit_training/convertor/test_macro
cp designs/test_macro/runs/ct/tmp/placement/7-macros_placed.def ../circuit_training/circuit_training/convertor/test_macro/test_macro.def
cp designs/test_macro/runs/ct/tmp/merged.nom.lef ../circuit_training/circuit_training/convertor/test_macro/test_macro.lef
cp designs/test_macro/runs/ct/tmp/synthesis/1-sky130_fd_sc_hd__tt_025C_1v80.no_pg.lib  ../circuit_training/circuit_training/convertor/test_macro/test_macro.lib
cp designs/test_macro/runs/ct/results/synthesis/test_macro.v ../circuit_training/circuit_training/convertor/test_macro/
```

### Prepare config file

Create `config.json` in `/home/$USER/src/circuit_training/circuit_training/convertor`:
```json
{
    "CONVERTOR": "d2p",
    "DESIGN": "test_macro",
    "NETLIST": "./test_macro/test_macro.v",

    "DEF": "./test_macro/test_macro.def",
    "LEFS": ["./test_macro/test_macro.lef"],
    "LIBS": ["./test_macro/test_macro.lib"],
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
cd ~/src/circuit_training
docker run -it --rm -v $(pwd):/workspace --workdir /workspace circuit_training:wc bash
cd circuit_training/convertor
python3 convertor.py config.json
```

<details>
  <summary>Command output:</summary>

  ```bash
    root@88403868560e:/workspace/circuit_training/convertor# python3 convertor.py config.json
    OpenROAD v2.0-5083-ga783d1b9c
    This program is licensed under the BSD-3 license. See the LICENSE file for details.
    Components of this program may be licensed under more restrictive licenses which must be honored.
    [INFO ODB-0222] Reading LEF file: ./test_macro/test_macro.lef
    [WARNING ODB-0220] WARNING (LEFPARS-2036): SOURCE statement is obsolete in version 5.6 and later.
    The LEF parser will ignore this statement.
    To avoid this warning in the future, remove this statement from the LEF file with version 5.6 or later. See file ./test_macro/test_macro.lef at line 68353.

    [INFO ODB-0223]     Created 13 technology layers
    [INFO ODB-0224]     Created 25 technology vias
    [INFO ODB-0225]     Created 442 library cells
    [INFO ODB-0226] Finished LEF file:  ./test_macro/test_macro.lef
    [INFO ODB-0127] Reading DEF file: ./test_macro/test_macro.def
    [INFO ODB-0128] Design: test_macro
    [INFO ODB-0130]     Created 315 pins.
    [INFO ODB-0131]     Created 10 components and 380 component-terminals.
    [INFO ODB-0133]     Created 324 nets and 360 connections.
    [INFO ODB-0134] Finished DEF file: ./test_macro/test_macro.def
    Output netlist: ./test_macro/netlist.pb.txt
    Output netlist: ./test_macro/netlist.plc
    root@88403868560e:/workspace/circuit_training/convertor#
  ```  
</details>


### Running CT for clustered ProtoBuf
(in the same Docker container)
```bash
cd /workspace/
export OUTPUT_DIR=/workspace/circuit_training/convertor/test_macro/out
export BLOCK_NAME=test_macro
export NETLIST_FILE=/workspace/circuit_training/convertor/test_macro/netlist.pb.txt
export HMETIS_DIR=/usr/local/bin
export PLC_WRAPPER_MAIN=/usr/local/bin/plc_wrapper_main
python3 circuit_training/grouping/grouper_main.py \
--output_dir=$OUTPUT_DIR \
--netlist_file=$NETLIST_FILE \
--block_name=$BLOCK_NAME \
--hmetis_dir=$HMETIS_DIR \
--plc_wrapper_main=$PLC_WRAPPER_MAIN
```

In `/home/user/circuit_training/convertor/test_macro/out` you will find nested directories that contain `initial.plc` file.
The full path is also displayed during clustering execution (`Paritioned netlist: <full path to initial.plc>`).

#### If you got an error

Sometimes you can get an error like this:
```bash
FileNotFoundError: [Errno 2] No such file or directory: '/workspace/circuit_training/convertor/test_macro/out/19cols_28rows/g500_ub5_nruns10_c5_r3_v3_rc1/metis_input.part.593'
```

This happens because CT uses hmetis for clustering and hmetis generates output file, which name containes the running params & number of clusters. And CT uses this fact to search for output file. However, sometimes hmetis cannot generate clustered netlist with the given number of clusters and CT clustering fails. 

You can:
1) try to rerun clustering until hmetis generates the required clustering, or
2) just reduce the number of std cells clusters in [grouper.py](https://github.com/RustamC/circuit_training/blob/80fe78fc795d9390048aa795ce6feb3edf2f6706/circuit_training/grouping/grouper.py#L57) and rerun clustering.

In out example, you can jest set the number of std cells clusters to 10 and it should work.

<details>
  <summary>Output:</summary>

  ```bash
    I0522 09:13:16.771213 140228645574464 grouper.py:183] writing netlist: /workspace/circuit_training/convertor/test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/netlist.pb.txt
W0522 09:13:17.045428 140228645574464 placement_util.py:229] block_name is not set. Please add the block_name in:
/workspace/circuit_training/convertor/test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/netlist.pb.txt
or in:
None
I0522 09:13:17.267102 140228645574464 grouper.py:207] Placement file : /workspace/circuit_training/convertor/test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/initial.plc, WL: 219765.413113, cong: 0.738833}
I0522 09:13:17.273428 140228645574464 grouper.py:442] Partitioned netlist: /workspace/circuit_training/convertor/test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/netlist.pb.txt, plc file: /workspace/circuit_training/convertor/test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/initial.plc
I0522 09:13:17.274006 140228645574464 grouper.py:444] Previous costs:
I0522 09:13:17.275111 140228645574464 grouper.py:348] Wirelength: 220204.710008
I0522 09:13:17.275449 140228645574464 grouper.py:349] Wirelength cost: 0.427315
I0522 09:13:17.275783 140228645574464 grouper.py:350] Congestion cost: 0.740039
I0522 09:13:17.276026 140228645574464 grouper.py:351] Overlap cost: 0.000000
I0522 09:13:17.395770 140228645574464 grouper.py:450] Costs for partitioned netlist:
I0522 09:13:17.396512 140228645574464 grouper.py:348] Wirelength: 219765.415001
I0522 09:13:17.396742 140228645574464 grouper.py:349] Wirelength cost: 0.426462
I0522 09:13:17.397081 140228645574464 grouper.py:350] Congestion cost: 0.738833
I0522 09:13:17.397278 140228645574464 grouper.py:351] Overlap cost: 0.000000
x/y displacement: dx = -2.157368421052624, dy = -1.3914285714284915, macro: spm_01
x/y displacement: dx = 2.657368421052638, dy = 1.231428571428637, macro: spm_02
x/y displacement: dx = -2.157368421052624, dy = -0.32285714285717404, macro: spm_03
x/y displacement: dx = 2.657368421052638, dy = 0.8428571428571843, macro: spm_04
x/y displacement: dx = 2.657368421052638, dy = 0.4542857142857173, macro: spm_05
x/y displacement: dx = 7.49210526315801, dy = 0.4542857142857173, macro: spm_06
x/y displacement: dx = -2.157368421052624, dy = 0.06571428571430715, macro: spm_07
x/y displacement: dx = 2.657368421052638, dy = 1.6200000000000045, macro: spm_08
x/y displacement: dx = -7.672105263157903, dy = 0.06571428571430715, macro: spm_end_0
x/y displacement: dx = -2.157368421052624, dy = -1.7800000000000296, macro: spm_start_0
Total macro displacement: 42.65172932330853, avg: 4.265172932330853
I0522 09:13:17.862271 140228645574464 grouper.py:460] Saved legalized placement : /workspace/circuit_training/convertor/test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/legalized.plc
I0522 09:13:17.862604 140228645574464 grouper.py:461] Costs after legalization:
I0522 09:13:17.862900 140228645574464 grouper.py:348] Wirelength: 220070.918158
I0522 09:13:17.863093 140228645574464 grouper.py:349] Wirelength cost: 0.427055
I0522 09:13:17.863287 140228645574464 grouper.py:350] Congestion cost: 0.738375
I0522 09:13:17.863477 140228645574464 grouper.py:351] Overlap cost: 0.000000
Saved legalized placement : /workspace/circuit_training/convertor/test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/legalized.plc
  ```  
</details>

## Step 4. Running CT

Now you are ready for running CT. Paths to the clustered netlist & initial placement could be found in the clustering output:
```bash
Partitioned netlist: /workspace/circuit_training/convertor/test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/netlist.pb.txt, 
plc file: /workspace/circuit_training/convertor/test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/initial.plc
```

It's a little bit different from running the standard flow:
1) `SEQUENCE_LENGTH` param is set to number of macros + 1.
2) `NUM_ITERATIONS` is also set to speed up the placement process.

<details>
    <summary><b>Running instructions:</b></summary>
    
```bash
# Sets the environment variables needed by each job. These variables are
# inherited by the tmux sessions created in the next step.
# Sequence length = NUM_TOTAL_HARD_MACROS + 1
export ROOT_DIR=./logs/run_00
export REVERB_PORT=8008
export REVERB_SERVER="127.0.0.1:${REVERB_PORT}"
export NETLIST_FILE=/workspace/circuit_training/convertor/test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/netlist.pb.txt
export INIT_PLACEMENT=/workspace/circuit_training/convertor/test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/initial.plc
export SEQUENCE_LENGTH=11
export NUM_ITERATIONS=50

# Creates all the tmux sessions that will be used.
tmux new-session -d -s reverb_server && \
   tmux new-session -d -s collect_job_00 && \
   tmux new-session -d -s collect_job_01 && \
   tmux new-session -d -s collect_job_02 && \
   tmux new-session -d -s train_job && \
   tmux new-session -d -s eval_job

# Starts the Replay Buffer (Reverb) Job
tmux attach -t reverb_server
python3 -m circuit_training.learning.ppo_reverb_server \
   --root_dir=${ROOT_DIR}  --port=${REVERB_PORT}

# Starts the Training job
# Change to the tmux session `train_job`.
# `ctrl + b` followed by `s`
python3 -m circuit_training.learning.train_ppo \
  --root_dir=${ROOT_DIR} \
  --replay_buffer_server_address=${REVERB_SERVER} \
  --variable_container_server_address=${REVERB_SERVER} \
  --num_episodes_per_iteration=16 \
  --global_batch_size=64 \
  --netlist_file=${NETLIST_FILE} \
  --init_placement=${INIT_PLACEMENT} \
  --sequence_length=${SEQUENCE_LENGTH} \
  --num_iterations=${NUM_ITERATIONS}

# Starts the Collect job
# Change to the tmux session `collect_job_00`.
# `ctrl + b` followed by `s`
python3 -m circuit_training.learning.ppo_collect \
  --root_dir=${ROOT_DIR} \
  --replay_buffer_server_address=${REVERB_SERVER} \
  --variable_container_server_address=${REVERB_SERVER} \
  --task_id=0 \
  --netlist_file=${NETLIST_FILE} \
  --init_placement=${INIT_PLACEMENT} \
  --max_sequence_length=${SEQUENCE_LENGTH}

# Starts the Eval job
# Change to the tmux session `eval_job`.
# `ctrl + b` followed by `s`
python3 -m circuit_training.learning.eval \
  --root_dir=${ROOT_DIR} \
  --variable_container_server_address=${REVERB_SERVER} \
  --netlist_file=${NETLIST_FILE} \
  --init_placement=${INIT_PLACEMENT}

# <Optional>: Starts 2 more collect jobs to speed up training.
# Change to the tmux session `collect_job_01`.
# `ctrl + b` followed by `s`
python3 -m circuit_training.learning.ppo_collect \
  --root_dir=${ROOT_DIR} \
  --replay_buffer_server_address=${REVERB_SERVER} \
  --variable_container_server_address=${REVERB_SERVER} \
  --task_id=1 \
  --netlist_file=${NETLIST_FILE} \
  --init_placement=${INIT_PLACEMENT} \
  --max_sequence_length=${SEQUENCE_LENGTH}

# Change to the tmux session `collect_job_02`.
# `ctrl + b` followed by `s`
python3 -m circuit_training.learning.ppo_collect \
  --root_dir=${ROOT_DIR} \
  --replay_buffer_server_address=${REVERB_SERVER} \
  --variable_container_server_address=${REVERB_SERVER} \
  --task_id=2 \
  --netlist_file=${NETLIST_FILE} \
  --init_placement=${INIT_PLACEMENT} \
  --max_sequence_length=${SEQUENCE_LENGTH}

```
</details>
    
## Step 4. Conversion from CT placement (\*.plc) into OpenLane macro placement (\*.cfg)

After CT is finished, you will find final placement file `rl_opt_placement.plc` in `/home/$USER/src/circuit_training/logs/run_00/111/`.
To convert the final macro placement into OpenLane macro placement (\*.cfg), we should update `config.json` in `/home/$USER/src/circuit_training/circuit_training/convertor`:
```json
{
    "CONVERTOR": "p2m",
    "DESIGN": "test_macro",
    "NETLIST": "./test_macro/test_macro.v",

    "DEF": "./test_macro/test_macro.def",
    "LEFS": ["./test_macro/test_macro.lef"],
    "LIBS": ["./test_macro/test_macro.lib"],
    "PB_FILE": "./test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/netlist.pb.txt",
    "PLC_FILE": "/workspace/logs/run_00/111/rl_opt_placement.plc",
    "OPENROAD_EXE": "./utils/openroad",

    "MACRO_CFG": "./test_macro/test_macro.cfg"
}
```

As you can see:
1) `PB_FILE` now is pointing to clustered netlist,
2) `PLC_FILE` now is pointing to CT placement,
3) `CONVERTOR` value has changed.
    
To run the convertor you should first exit tmux (`ctrl + b` followed by `d`) & from the same Docker container run:
```bash
tmux kill-server
cd circuit_training/convertor
python3 convertor.py config.json
exit
```

<details>
  <summary>Output:</summary>

  ```bash
  root@88403868560e:/workspace# tmux kill-server
root@88403868560e:/workspace# cd circuit_training/convertor
root@88403868560e:/workspace/circuit_training/convertor# python3 convertor.py config.json
#[INFO] Reading from ./test_macro/out/19cols_28rows/g10_ub5_nruns10_c5_r3_v3_rc1/netlist.pb.txt
OpenROAD v2.0-5083-ga783d1b9c
This program is licensed under the BSD-3 license. See the LICENSE file for details.
Components of this program may be licensed under more restrictive licenses which must be honored.
[INFO ODB-0222] Reading LEF file: ./test_macro/test_macro.lef
[WARNING ODB-0220] WARNING (LEFPARS-2036): SOURCE statement is obsolete in version 5.6 and later.
The LEF parser will ignore this statement.
To avoid this warning in the future, remove this statement from the LEF file with version 5.6 or later. See file ./test_macro/test_macro.lef at line 68353.

[INFO ODB-0223]     Created 13 technology layers
[INFO ODB-0224]     Created 25 technology vias
[INFO ODB-0225]     Created 442 library cells
[INFO ODB-0226] Finished LEF file:  ./test_macro/test_macro.lef
[INFO ODB-0127] Reading DEF file: ./test_macro/test_macro.def
[INFO ODB-0128] Design: test_macro
[INFO ODB-0130]     Created 315 pins.
[INFO ODB-0131]     Created 10 components and 380 component-terminals.
[INFO ODB-0133]     Created 324 nets and 360 connections.
[INFO ODB-0134] Finished DEF file: ./test_macro/test_macro.def
Output cfg: ./test_macro/test_macro.cfg
root@88403868560e:/workspace/circuit_training/convertor#  
  ```
</details>
    
After conversion you will find `test_macro.cfg` in `/home/$USER/src/circuit_training/circuit_training/convertor/test_macro`.
We should copy it to OpenLane design folder:
```bash
cp ./circuit_training/convertor/test_macro/test_macro.cfg ../OpenLane/designs/test_macro
```

## Step 5. Running OpenLane with CT macro placement

Now in this final step we should update design configuration in `config.json` in `/home/$USER/src/OpenLane/designs/test_macro` and set `MACRO_PLACEMENT_CFG` (path to CT macro placement file):
```json
{
    "PDK": "sky130A",
    ...
    "MACRO_PLACEMENT_CFG": "dir::test_macro.cfg",
    ...
}
```
And then run:
```bash
cd ../OpenLane
make mount
./flow.tcl -design test_macro -run after_ct
```

## The end

![The end](https://i.pinimg.com/236x/7f/ad/fe/7fadfe1460f434e256e012136d6c81de.jpg?nii=t)
    
[^1]: The LEF/DEF to ProtoBuf converter and the ProtoBuf parser are taken from the [MacroPlacement](https://github.com/TILOS-AI-Institute/MacroPlacement) project with slight modifications.
