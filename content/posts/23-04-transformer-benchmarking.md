---
title: "Experimenting with Transformers"
date: 2023-04-21T15:05:55-04:00
draft: true
tags: ["transformers", "benchmarking"]
---

## Install FT natively
- TL;DR: I cannot successfully run the FT benchmarking with native installation. Kept on running into this error: https://github.com/NVIDIA/FasterTransformer/issues/177

- Environment: A6000 (Ampere, SM86, 48GB memory), CUDA 11.7, PyTorch (1.13), g++ (Ubuntu 9.4.0-1ubuntu1~20.04.1) 9.4.0

- Build from source. 

```bash
$ export INSTALL_PATH=/work/zhang-capra/users/sx233/install

# Build with PyTorch support on
$ cmake -DSM=86 -DCMAKE_BUILD_TYPE=Release -DBUILD_PYT=ON -DBUILD_MULTI_GPU=ON -DCMAKE_PREFIX_PATH=$INSTALL_PATH ..
$ git submodule init && git submodule update
$ make -j32
```

- Additional dependencies: huggingface transformers, PyO3, Rust nightly (required for PyO3 binding)

```bash
# A nightly build of Rust is required for PyO3
# https://github.com/huggingface/transformers/issues/2831#issuecomment-600141935
$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
$ source $HOME/.cargo/env
$ pip install transformers==2.5.1
```

### Changes made to compile FT
- As there is no nvidia-docker installed on the server, I installed the whole thing in a native way. 

- DO NOT use conda to install pytroch & MKL. It leads to lots of undefined reference errors to MKL APIs

```bash
pip install torch==1.13.1+cu117 torchvision==0.14.1+cu117 --extra-index-url https://download.pytorch.org/whl/cu117
```

- Patch: link `mpi_utils` with `-lmpi_cxx`.To fix the error `undefined reference to MPI::Datatype::Datatype()` in linking process

```bash
if (BUILD_MULTI_GPU)
    target_link_libraries(mpi_utils PUBLIC -lmpi -lmpi_cxx logger)
endif()
```

- Patch: install PyTorch from source with MPI support. Otherwise, `torch.distributed` is not available. https://discuss.pytorch.org/t/distributed-pytorch-with-mpi/77106/4

```bash
$ python ../examples/pytorch/bert/bert_example.py 1 12 32 12 64 --data_type fp16 --time
[INFO] MPI is not available in this PyTorch build.
[FT][WARNING] Skip NCCL initialization since requested tensor/pipeline parallel sizes are equals to 1.
```


## Install FT with Nvidia docker

- No enough space on `zhang-x3`: Environment: 2080Ti x8 (Turing, 11GB DDR6), Ubuntu 20.04 (g++ 9.4.0), CMake 3.16. 

```bash
docker: write /var/lib/docker/tmp/GetImageBlob677064158: no space left on device.
```

- Private work station (3060Ti x2, 3070 x2, Ampere, 8GB GDDR6, SM86): Ubuntu 22.04 (g++ 11.3.0), CUDA 11.5

```bash
# GPG key adding to apt-key
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
      && curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
      && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
            sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
            sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# https://cnvrg.io/how-to-setup-docker-and-nvidia-docker-2-0-on-ubuntu-18-04/
sudo apt-get install -y nvidia-docker2


# inside the docker container (TF)
cmake -DSM=86 -DCMAKE_BUILD_TYPE=Release -DBUILD_TF=ON -DTF_PATH=/usr/local/lib/python3.8/dist-packages/tensorflow_core/ -DBUILD_MULTI_GPU=ON ..
```

- Unfortunately, no peer access on my private work station. As the other 3 GPUs are connected with PCIe 3.0 x1 slot, which is sharing the DMI to CPU, and far away from GPU0. 

```bash
root@1ba1b6c90879:/workspace/FasterTransformer/build# ./bin/multi_gpu_gpt_example
Total ranks: 1.
P0 is running with 0 GPU.
Device NVIDIA GeForce RTX 3070
[FT][WARNING] Device 0 peer access Device 1 is not available.
[FT][WARNING] Device 0 peer access Device 2 is not available.
[FT][WARNING] Device 0 peer access Device 3 is not available.
```


## Some notes
- AITemplate: talked with one developer with 