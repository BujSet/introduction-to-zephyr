# Intstruction for Following the Build and run Zephyr applications Tutorial in arm FVP simulator with executorch
https://learn.arm.com/learning-paths/embedded-and-microcontrollers/zephyr/zephyr/

# Hardware Requirements:

The docker image used here now supports running on both arm64 and amd64 architectures for Linux.

# Accesssing Docker Image:

The docker image can be accessed by either building the image directly from the Dockerfile source file, or by pulling the image down (recommended). In either case, the docker image requires *at least* 40 GB of disk space. If using windows, it's highly recommended to manage your docker artifacts with Docker Desktop.

## Building the image locally

```
docker build -t rselagam/zephyr-armfvp:v1 -f Dockerfile.armfvp_zephyr  .
```

## (Recommended) Pulling the image from Docker hub

```
docker pull rselagam/zephyr-armfvp:v1
```

# Run docker image interactivately

## Linux/macOS

```
docker run --rm -it --entrypoint /bin/bash  --net=host -v "$(pwd)"/workspace:/workspace -w /workspace rselagam/zephyr-armfvp:v1
```

## Windows (PowerShell)

```
docker run --rm -it --entrypoint /bin/bash --net=host -v "${PWD}\workspace:/workspace" -w /workspace rselagam/zephyr-armfvp:v1
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
FVP_Corstone_SSE-300_Ethos-U55 -a /home/zephyruser/zephyrproject/zephyr/build/zephyr/zephyr.elf -C mps3_board.visualisation.disable-visualisation=1 -C mps3_board.telnetterminal0.start_telnet=0 -C mps3_board.uart0.out_file='-' --simlimit 30
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
docker build --build-arg BASE_IMAGE=ubuntu:22.04 --no-cache --platform linux/amd64,linux/arm64,windows/amd64 -t env-zephyr-armfvp:v6 -f Dockerfile.armfvp_zephyr  .
docker build --build-arg BASE_IMAGE=arm64v8/ubuntu:22.04 --no-cache --platform linux/arm64 -t env-zephyr-armfvp-linux-arm64:v6 -f Dockerfile.armfvp_zephyr  .
docker build --platform windows/amd64 -t env-zephyr-armfvp-win64:v3 -f Dockerfile.armfvp_zephyr  .
docker build --platform linux/arm64 -t env-zephyr-armfvp-arm64:v3 -f Dockerfile.armfvp_zephyr  .

docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
docker run --privileged --rm tonistiigi/binfmt --install all
docker buildx create --name mybuilder --driver docker-container --platform linux/amd64,linux/arm6 --use
docker buildx build --network=host --platform linux/amd64,linux/arm64 -t rselagam/zephyr-armfvp:v1 --push -f Dockerfile.armfvp_zephyr  .
```

# Building Executorch with arm Zephyr Toolchain

In the docker image, with the old venv sourced:

```
cd /home/zephyruser/executorch
source .venv/bin/activate
git switch -c arm-zphyr-eabi origin/arm-zphyr-eabi
git pull --rebase
./install_executorch.sh && \
    git submodule sync && \
    git submodule update --init --recursive && \
    mkdir cmake-out && cd cmake-out && cmake .. && cd ../ && \
    cmake --build cmake-out -j9
export PATH=${PATH}:/home/zephyruser/zephyr-sdk-0.16.0/arm-zephyr-eabi/bin
./examples/arm/setup.sh --i-agree-to-the-contained-eula --skip-toolchain-setup
source /home/zephyruser/executorch/examples/arm/ethos-u-scratch/setup_path.sh
./examples/arm/setup.sh --i-agree-to-the-contained-eula --skip-toolchain-setup
cmake --preset zephyr
cmake --build cmake-out -j10 --target executor_runner
python3 -m examples.arm.aot_arm_compiler --model_name="add" --quantize
python3 -m examples.arm.aot_arm_compiler --model_name="add"
python3 -m examples.arm.aot_arm_compiler --model_name="mv3"
FVP_Corstone_SSE-300_Ethos-U55 -a cmake-out/executor_runner -C mps3_board.visualisation.disable-visualisation=1 -C mps3_board.telnetterminal0.start_telnet=0 -C mps3_board.uart0.out_file='-' -C cpu0.CFGITCMSZ=15 -C cpu0.CFGDTCMSZ=15 -C mps3_board.FPGA_SRAM_SIZE=2 --simlimit 600
```




git config --global user.email "ranganath1000@gmail.com" && \
git config --global user.name "BujSet"

cd /home/zephyruser/
git clone https://github.com/BujSet/executorch.git
cd executorch
git switch -c arm-zphyr-eabi origin/arm-zphyr-eabi
git pull --rebase
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip && \
    ./install_requirements.sh && \
    git submodule sync && \
    git submodule update --init --recursive

export PATH=${PATH}:/home/zephyruser/zephyr-sdk-0.16.0/arm-zephyr-eabi/bin
./examples/arm/setup.sh --i-agree-to-the-contained-eula --skip-toolchain-setup
source /home/zephyruser/executorch/examples/arm/ethos-u-scratch/setup_path.sh
./examples/arm/setup.sh --i-agree-to-the-contained-eula --skip-toolchain-setup
cmake --preset zephyr
cmake --build cmake-out -j10 --target executor_runner



git config --global user.email "ranganath1000@gmail.com" && \
git config --global user.name "BujSet"

cd /home/zephyruser/
git clone https://github.com/BujSet/executorch.git
cd executorch
git switch -c arm-zphyr-eabi origin/arm-zphyr-eabi
git pull --rebase
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
./install_executorch.sh --clean
./install_executorch.sh

./examples/arm/setup.sh --i-agree-to-the-contained-eula --skip-toolchain-setup
source /home/zephyruser/executorch/examples/arm/ethos-u-scratch/setup_path.sh
./examples/arm/setup.sh --i-agree-to-the-contained-eula --skip-toolchain-setup
cmake --preset zephyr
cmake --build cmake-out -j10 --target executor_runner

cmake --preset zephyr
examples/arm/run.sh --model_name=add --no_quantize --target=ethos-u55-128
