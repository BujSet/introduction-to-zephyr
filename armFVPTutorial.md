# Intstruction for Following the Executorch arm-ethos-u FVP Tutorial


# Build Docker Image:

```
docker build -t env-zephyr-armfvp -f Dockerfile.armfvp .
```

# Run docker image interactivately

```
docker run --rm -it --entrypoint /bin/bash  -p 3333:3333 -p 2222:22 -p 8800:8800 -v "$(pwd)"/workspace:/workspace -w /workspace env-zephyr-armfvp
```

# Using arm FVP in the Docker image


## Preliminaries

On start, you should be prompted with something that looks like:

```
(venv) root@7c26f549e7f3:/workspace#
```

Running an `ls` command should show you the following output:

```
apps  boards  executorch  modules  west.yml
```

Now, you can run the following to setup the paths for the tutorial:

```
source  executorch/examples/arm/ethos-u-scratch/setup_path.sh
cd executorch/
```

Since we are developing in a Docker contianer, you'll need to specify you github credentials before running the setup script:

```
git config --global user.email "you@example.com"
git config --global user.name "Your Name"
```

## Running the FVP Setup Script

After that, you should finally be able to run the setup command:

```
./examples/arm/setup.sh --i-agree-to-the-contained-eula
```

This will likely take quite a while to run... (mayb 15-20 min)

For some reason, this produces a package dependency error, but is resolved if you just re-run:

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
/workspace/executorch/examples/arm/ethos-u-scratch/arm-gnu-toolchain-13.3.rel1-x86_64-arm-none-eabi/bin/arm-none-eabi-gcc
```





