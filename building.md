# Building the GPU C library

This document presents instructions to build the LLVM C library targeting NVIDIA GPUs. These instructions were tested on a machine running Windows 11 Home with:  

* AMD Ryzen AI 9 HX (12 cores)  
* 32 GB RAM
* NVIDIA GeForce RTX 4070 Laptop GPU

Newer or older hardware should also work; adjust the parallelism accordingly (a flag like `-j4` in `ninja` commands).

WSL 2 and a recent NVIDIA Windows GPU driver must already be enabled and installed.

## 1. Install Ubuntu 24.04 on WSL 2

Open the Windows PowerShell and run:

```powershell
wsl --install Ubuntu-24.04
```

Launch the new distribution, choose a UNIX username and password, and execute the next steps.

## 2. Install the CUDA Toolkit for WSL 2

Follow the [CUDA on WSL User Guide](https://docs.nvidia.com/cuda/wsl-user-guide/index.html#cuda-support-for-wsl-2) and use the WSL-Ubuntu meta-package so you do not overwrite the Windows host driver.

For example, the commands to install the CUDA Toolkit 12.9 are:

```bash
sudo apt-key del 7fa2af80
wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-wsl-ubuntu.pin
sudo mv cuda-wsl-ubuntu.pin /etc/apt/preferences.d/cuda-repository-pin-600
wget https://developer.download.nvidia.com/compute/cuda/12.9.0/local_installers/cuda-repo-wsl-ubuntu-12-9-local_12.9.0-1_amd64.deb
sudo dpkg -i cuda-repo-wsl-ubuntu-12-9-local_12.9.0-1_amd64.deb
sudo cp /var/cuda-repo-wsl-ubuntu-12-9-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt update
sudo apt -y install cuda-toolkit-12-9
```

## 3. Set up environment variables

Append the following to `~/.bashrc` (use Visual Studio Code or any editor):

```bash
export LLVM_HOME="$HOME/opt/llvm"
export PATH="$LLVM_HOME/bin:$PATH"
```

Activate the change and confirm:

```bash
source ~/.bashrc
echo "$LLVM_HOME"
```

## 4. Install prerequisites

Install build prerequisites:

```bash
sudo apt update
sudo apt -y install build-essential git cmake ninja-build gcc-multilib
```

## 5. Check out the LLVM project

A shallow clone is fast and sufficient for users who only build:

```bash
git clone --depth=1 https://github.com/llvm/llvm-project.git
```

## 6. Configure the build

Below is the list of commands for a simple recipe to build the GPU C library in release mode.

```bash
cd llvm-project
mkdir build && cd build
cmake ../llvm -G Ninja \
  -DLLVM_ENABLE_PROJECTS="clang;lld" \
  -DLLVM_ENABLE_RUNTIMES="openmp;offload;libc" \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX="$LLVM_HOME" \
  -DRUNTIMES_nvptx64-nvidia-cuda_LLVM_ENABLE_RUNTIMES=libc \
  -DLLVM_RUNTIME_TARGETS="default;nvptx64-nvidia-cuda"
```

We need an up-to-date `clang` to build the GPU C library and an up-to-date `lld` to link GPU executables, so we enable them in `LLVM_ENABLE_PROJECTS`. We add `openmp`, `offload`, and `libc` to `LLVM_ENABLE_RUNTIMES` so they are built for the default target. We then set `RUNTIMES_nvptx64-nvidia-cuda_LLVM_ENABLE_RUNTIMES` to enable `libc` for NVIDIA GPUs. The `LLVM_RUNTIME_TARGETS` sets the enabled targets to be built; in this case we want the default target and NVIDIA GPUs.

## 7. Build and test

After configuring the build with the above `cmake` command, one can build the GPU C library with the following command:

```bash
ninja install -j4
```

Try the `clang` compiler:

```bash
clang --version
```

Then run all supported GPU tests:

```bash
ninja check-libc-nvptx64-nvidia-cuda -j4
```
