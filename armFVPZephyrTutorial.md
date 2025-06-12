# Intstruction for Following the Build and run Zephyr applications Tutorial in arm FVP simulator with executorch
https://learn.arm.com/learning-paths/embedded-and-microcontrollers/zephyr/zephyr/

# Hardware Requirements:

This tutorial was written based running on a Windows machine, but should work for others.

# Accesssing Docker Image:

The docker image can be accessed by either building the image directly from the Dockerfile source file, or by pulling the image down (recommended). In either case, the docker image requires *at least* 40 GB of disk space. If using windows, it's highly recommended to manage your docker artifacts with Docker Desktop.

## Building the image locally

```
docker build -t env-zephyr-armfvp:v5 -f Dockerfile.armfvp_zephyr_win64  .
```

## (Recommended) Pulling the image from Docker hub

```
docker pull rselagam/env-zephyr-armfvp:v5
```

# Run docker image interactivately

## Linux/macOS

```
docker run --rm -it --entrypoint /bin/bash  --net=host -v "$(pwd)"/workspace:/workspace -w /workspace rselagam/env-zephyr-armfvp:v5
```

## Windows (PowerShell)

```
docker run --rm -it --entrypoint /bin/bash --net=host -v "${PWD}\workspace:/workspace" -w /workspace rselagam/env-zephyr-armfvp:v5
```

## (Optional) Test your executorch installation:

Generate the example pte file:
```
cd /home/zephyruser/executorch
source .venv/bin/activate
python -m examples.portable.scripts.export --model_name="add"
```

You should see the following output:

```
WARNING:torchao.kernel.intmm:Warning: Detected no triton, on systems without Triton certain kernels will not work
/home/zephyruser/executorch/.venv/lib/python3.10/site-packages/executorch/exir/dialects/edge/_ops.py:9: UserWarning: pkg_resources is deprecated as an API. See https://setuptools.pypa.io/en/latest/pkg_resources.html. The pkg_resources package is slated for removal as early as 2025-11-30. Refrain from using this package or pin to Setuptools<81.
  import pkg_resources
```

and `add.pte` will be created under `/home/zephyruser/executorch/`

```
./cmake-out/executor_runner --model_path add.pte
```

Which should produce:

```
I 00:00:00.003987 executorch:executor_runner.cpp:166] Model file add.pte is loaded.
I 00:00:00.004058 executorch:executor_runner.cpp:175] Using method forward
I 00:00:00.004429 executorch:executor_runner.cpp:226] Setting up planned buffer 0, size 48.
I 00:00:00.005262 executorch:executor_runner.cpp:251] Method loaded.
I 00:00:00.006576 executorch:executor_runner.cpp:284] Model executed successfully 1 time(s) in 0.691078 ms.
I 00:00:00.006604 executorch:executor_runner.cpp:293] 1 outputs:
Output 0: tensor(sizes=[1], [2.])
```

# Setup arm FVP in the Docker image

```
cd /home/zephyruser/executorch
source  examples/arm/ethos-u-scratch/setup_path.sh
```
## Verify Arm toolcahin:

Run

```
which arm-none-eabi-gcc
```

which should output something like:

```
/executorch/examples/arm/ethos-u-scratch/arm-gnu-toolchain-13.3.rel1-x86_64-arm-none-eabi/bin/arm-none-eabi-gcc
```

## Build the models from the tutorial:

```
python3 -m examples.arm.aot_arm_compiler --model_name="softmax"
python3 -m examples.arm.aot_arm_compiler --model_name="add" --delegate
```

There is some error with the quatization flag, for now we'll not use that model, but it's a future TODO: `python3 -m examples.arm.aot_arm_compiler --model_name="mv2" --delegate --quantize`

You should now see the following `.pte` files in the `/executorch/` directory:

```
add_arm_delegate_ethos-u55-128.pte
softmax_arm_ethos-u55-128.pte
```

## Run the models

### Running the add model

```
./examples/arm/run.sh --model_name=add --target=ethos-u85-128
```

Produces the following output:

```
I [executorch:arm_executor_runner.cpp:675] Model executed successfully.
I [executorch:arm_executor_runner.cpp:679] 1 outputs:
Output[0][0]: (int) 2
Output[0][1]: (int) 2
Output[0][2]: (int) 2
Output[0][3]: (int) 2
Output[0][4]: (int) 2
I [executorch:arm_executor_runner.cpp:783] Program complete, exiting.
I [executorch:arm_executor_runner.cpp:787]
Info: /OSCI/SystemC: Simulation stopped by user.
[backends/arm/scripts/run_fvp.sh] Simulation complete, 0
Checking for problems in log:
No problems found!
+ set +x
```

### Running the softmax model:

```
./examples/arm/run.sh --model_name=softmax --target=ethos-u85-128
```

```
I [executorch:arm_executor_runner.cpp:675] Model executed successfully.
I [executorch:arm_executor_runner.cpp:679] 1 outputs:
Output[0][0]: (float) 0.225432
Output[0][1]: (float) 0.225432
Output[0][2]: (float) 0.225432
Output[0][3]: (float) 0.225432
I [executorch:arm_executor_runner.cpp:783] Program complete, exiting.
I [executorch:arm_executor_runner.cpp:787]
Info: /OSCI/SystemC: Simulation stopped by user.
[backends/arm/scripts/run_fvp.sh] Simulation complete, 0
Checking for problems in log:
No problems found!
+ set +x
```


# Running the Zephyr hello world

## (Optional) Building the elf
The project elf should already be built, but you can rebuild with the following if necessary:

```
cd /home/zephyruser/zephyrproject/zephyr
rm -rf build
west build -p auto -b mps3/corstone300/an547 samples/hello_world
```

## Flashing the FVP Simulator

```
cd /home/zephyruser/zephyrproject/zephyr
FVP_Corstone_SSE-300_Ethos-U55 -a build/zephyr/zephyr.elf -C mps3_board.visualisation.disable-visualisation=1 -C mps3_board.telnetterminal0.start_telnet=0 -C mps3_board.uart0.out_file='-' --simlimit 30
```

You should see output that looks like:

```
telnetterminal0: Listening for serial connection on port 5000
telnetterminal1: Listening for serial connection on port 5001
telnetterminal2: Listening for serial connection on port 5002
telnetterminal5: Listening for serial connection on port 5003

    Ethos-U rev 136b7d75 --- Apr 12 2023 13:44:01
    (C) COPYRIGHT 2019-2023 Arm Limited
    ALL RIGHTS RESERVED

*** Booting Zephyr OS build v4.1.0-5715-gd2a5c1ca82f0 ***
Hello World! mps3/corstone300/an547
xterm: xterm: Xt error: Can't open display:
Xt error: Can't open display:
xterm: DISPLAY is not set
xterm: DISPLAY is not set

Info: Simulation is stopping. Reason: Simulated time has been exceeded.

Info: /OSCI/SystemC: Simulation stopped by user.
[warning ][main@0][01 ns] Simulation stopped by user
```

# TODOs
Need to build the image to work on different platforms, following commands may help

```
docker build --platform linux/amd64,linux/arm64,windows/amd64 -t env-zephyr-armfvp:v3 -f Dockerfile.armfvp_zephyr  .
docker build --platform windows/amd64 -t env-zephyr-armfvp-win64:v3 -f Dockerfile.armfvp_zephyr_win64  .
docker build --platform linux/arm64 -t env-zephyr-armfvp-arm64:v3 -f Dockerfile.armfvp_zephyr_arm64  .
```
