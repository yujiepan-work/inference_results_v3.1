## Setup Instructions

### Anaconda and Conda Environment
+ Download and install conda

   Skip this step if you have installed conda already.
```
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O miniconda.sh
bash miniconda.sh
```

**Make sure your conda is installed at `$HOME/miniconda3`, otherwise you need to modify in the following scripts.**

+ For INT8
```
dnf install -y gcc-toolset-11-gcc gcc-toolset-11-gcc-c++ # cannot use this without root. See the workaround in prepare_env.sh
source </path/to/gcc-11/>enable
bash prepare_env.sh gpt-j-env 2>&1 | tee prepare_env.log
conda activate gpt-j-env
```
if ipex not installed successfully (e.g., on diamond, chances are some libs are missing), please install the wheel from [my compiled wheel from dgx1]. One issue is that, the wheel is compiled with transformers=4.31 rather than 4.28 (as required in `prepare_env.sh`), but i don't want to build that again. I think using 4.31 should be fine...

```bash
pip install https://github.com/intel-sandbox/yujiepan.debug.23h2.mlperf3.1/raw/main/intel_extension_for_pytorch-2.2.0%2Bgit880fda9-cp39-cp39-linux_x86_64.whl
pip install transformers==4.31
```


+ For INT4 - Please note that you will use a different environment from that for calibration
```
bash prepare_env_int4.sh gpt-j-env-int4 2>&1 | tee prepare_env_int4.log
conda activate gpt-j-env-int4
```

Again, you might want to check if ipex is installed correctly. Otherwise, use this wheel [here](https://github.com/intel-sandbox/yujiepan.debug.23h2.mlperf3.1/raw/main/intel_extension_for_pytorch-2.2.0%2Bgit880fda9-cp39-cp39-linux_x86_64.whl)

### Download and Prepare Dataset
```
CUR_DIR=$(pwd)
export WORKLOAD_DATA=${CUR_DIR}/data
mkdir -p ${WORKLOAD_DATA}
```

+ Download cnn-dailymail calibration set
```
python download-calibration-dataset.py --calibration-list-file calibration-list.txt --output-dir ${WORKLOAD_DATA}/calibration-data
```

+ Download cnn-dailymail validation set
```
python download-dataset.py --split validation --output-dir ${WORKLOAD_DATA}/validation-data
```

### Download and prepare model
+ Get finetuned checkpoint

   If you did not run calibration for INT4:
```
CHECKPOINT_DIR=${WORKLOAD_DATA}/gpt-j-checkpoint
wget https://cloud.mlcommons.org/index.php/s/QAZ2oM94MkFtbQx/download -O gpt-j-checkpoint.zip
unzip gpt-j-checkpoint.zip
mv gpt-j/checkpoint-final/ ${CHECKPOINT_DIR}
```
   If you have run calibration for INT4:
```
ln -sf $(pwd)/../../../calibration/gpt-j/pytorch-cpu/INT4/data/gpt-j-checkpoint ${WORKLOAD_DATA}/gpt-j-checkpoint
```
+ For INT8 model
```
source setup_env.sh
bash run_quantization.sh
```

+ For INT4 - Please go to `$(pwd)/../../../calibration/gpt-j/pytorch-cpu/INT4/` and follow the `README.md` in that directory to finish calibration first. Then return to continue from below
```
bash run_quantization_int4.sh
```

### Run Benchmarks with Int8
+ Offline (Performance)
```
bash run_offline.sh
bash run_offline_int4.sh # for INT4
```

+ Offline (Accuracy)
```
bash run_offline_accuracy.sh
bash run_offline_accuracy_int4.sh # for INT4
```

+ Server (Performance)
```
bash run_server.sh
bash run_server_int4.sh # for INT4
```

+ Server (Accuracy)
```
bash run_server_accuracy.sh
bash run_server_accuracy_int4.sh # for INT4
```

## Docker Instructions
Using the dockerfile provided in `docker/Dockerfile` and `docker/build_gpt-j_container.sh`, user can build and run the benchmarks following the instructions below

### Building the container

+ Since the Docker build step copies the `gpt-j` directory and its subdirectories, the `${WORKLOAD_DATA}` and `gpt-j-env` directories (which can be very large) have to be moved if they're present in the current folder. Skip this if not applicable
```
mv ${WORKLOAD_DATA} ../../gpt-j-data [Or your selected path]
mv gpt-j-env ../../gpt-j-env
```

+ Now build the docker image
```
cd docker
bash build_gpt-j_container.sh
cd ..
```
+ Start the container
```
ln -s ../../gpt-j-data data # Create softlink to the moved workload data
source setup_env.sh
docker run --name intel_gptj --privileged -itd --net=host --ipc=host -v ${WORKLOAD_DATA}:/opt/workdir/code/gptj-99/pytorch-cpu/data mlperf_inference_gptj:3.1
docker exec -it intel_gptj bash
```

 ### Start benchmarks

+ Offline (Performance)
```
bash run_offline.sh
```


+ Offline (Accuracy)
```
bash run_offline_accuracy.sh
```

+ Server (Performance)
```
bash run_server.sh
```

+ Server (Accuracy)
```
bash run_server_accuracy.sh
```
