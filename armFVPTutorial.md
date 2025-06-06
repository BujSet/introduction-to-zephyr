# Intstruction for Following the Executorch arm-ethos-u FVP Tutorial

# Hardware Requirements:

This tutorial was written based running on a Windows machine using WSL2. You need to provision at least 16GB of memory for WSL to be able to run the FVP simulator.

## Checking WSL Memory Provision

TODO steps here

check if .wslconfig file exists,

if not create it and set memory to 16GB

# Build Docker Image:

```
docker build -t env-zephyr-armfvp -f Dockerfile.armfvp .
```

# Run docker image interactivately

```
docker run --rm -it --entrypoint /bin/bash  -p 3333:3333 -p 2222:22 -p 8800:8800 -v "$(pwd)"/workspace:/workspace -w /workspace env-zephyr-armfvp
```

# Building Executorch inside the Docker image

```
cd /executorch/
```

```
python3 -m venv .venv && source .venv/bin/activate && pip install --upgrade pip
git submodule sync
git submodule update --init --recursive
mkdir cmake-out && cd cmake-out && cmake .. && cd ../
cmake --build cmake-out -j9
```

## (Optional) Test your executorch installation:

Generate the example pte file:
```
python -m examples.portable.scripts.export --model_name="add"
```

Should see output that looks like:

```
[INFO 2025-06-06 18:57:58,011 utils.py:50] Core ATen graph:
graph():
    %x : [num_users=1] = placeholder[target=x]
    %y : [num_users=1] = placeholder[target=y]
    %add : [num_users=1] = call_function[target=torch.ops.aten.add.Tensor](args = (%x, %y), kwargs = {})
    return (add,)
[INFO 2025-06-06 18:57:58,137 utils.py:70] Exported graph:
ExportedProgram:
    class GraphModule(torch.nn.Module):
        def forward(self, x: "f32[1]", y: "f32[1]"):
             # File: /executorch/examples/models/toy_model/model.py:47 in forward, code: z = x + y
            aten_add_tensor: "f32[1]" = executorch_exir_dialects_edge__ops_aten_add_Tensor(x, y);  x = y = None
            return (aten_add_tensor,)

Graph signature: ExportGraphSignature(input_specs=[InputSpec(kind=<InputKind.USER_INPUT: 1>, arg=TensorArgument(name='x'), target=None, persistent=None), InputSpec(kind=<InputKind.USER_INPUT: 1>, arg=TensorArgument(name='y'), target=None, persistent=None)], output_specs=[OutputSpec(kind=<OutputKind.USER_OUTPUT: 1>, arg=TensorArgument(name='aten_add_tensor'), target=None)])
Range constraints: {}

[INFO 2025-06-06 18:57:58,180 utils.py:141] Saved exported program to ./add.pte
```

and `add.pte` will be created under `/executorch/`

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

# Setup  arm FVP in the Docker image

```
cd /executorch
```

Now run

```
./examples/arm/setup.sh --i-agree-to-the-contained-eula
```

This will fail with the error below:

```
ERROR: pip's dependency resolver does not currently take into account all the packages that are installed. This behaviour is the source of the following dependency conflicts.
tosa-tools 0.80.2.dev1+g70ed0b4 requires jsonschema, which is not installed.
tosa-tools 0.80.2.dev1+g70ed0b4 requires flatbuffers==23.5.26, but you have flatbuffers 24.12.23 which is incompatible.
tosa-tools 0.80.2.dev1+g70ed0b4 requires numpy<2, but you have numpy 2.2.6 which is incompatible.
```

But now the path setup script is available, so you can run the follwing commands to comeplete the setup process:

```
source  examples/arm/ethos-u-scratch/setup_path.sh
./examples/arm/setup.sh --i-agree-to-the-contained-eula
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
