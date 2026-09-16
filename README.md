# libvmaf-cuda-windows-guide
Step-by-step guide to run GPU-accelerated VMAF (libvmaf_cuda) on Windows using WSL2, Docker, and the NVIDIA Container Toolkit.

**TL;DR:** Running GPU-accelerated VMAF on Windows is surprisingly painful. After spending hours fighting broken builds and undocumented errors, the only reliable path I found is **WSL2 + Docker Desktop + NVIDIA Container Toolkit + a CUDA-enabled FFmpeg**. This guide walks you through the whole setup using the [easyVmaf](https://github.com/gdavila/easyVmaf) project, so you can skip the trial-and-error I went through.

This guide uses **easyVmaf**, a project that ships a `Dockerfile.cuda` specifically built for this purpose. We'll run it inside WSL2, with Docker Desktop and the NVIDIA Container Toolkit handling the GPU passthrough.

**The result:** VMAF running at ~15x real-time speed on an RTX 3060 Mobile. A 46-minute video gets analyzed in about 3 minutes. On CPU, the same task would take like an hour.

> ⚠️ **AMD and Intel GPUs will not work with this guide.** `libvmaf_cuda` is NVIDIA-exclusive.

## Table of contents

- [The Problem](#the-problem)
- [Prerequisites](#prerequisites)
- [Part 1 — Setting up the environment](#part-1--setting-up-the-environment)
  - [1.1 Install WSL2](#11-install-wsl2)
  - [1.2 Accessing Windows drives from WSL](#12-accessing-windows-drives-from-wsl)
  - [1.3 Verify GPU access from WSL](#13-verify-gpu-access-from-wsl)
  - [1.4 Install Docker Desktop](#14-install-docker-desktop)
  - [1.5 Install the NVIDIA Container Toolkit](#15-install-the-nvidia-container-toolkit)
- [Part 2 — Building the easyVmaf image](#part-2--building-the-easyvmaf-image)
  - [2.1 Clone the repository](#21-clone-the-repository)
  - [2.2 Critical fix: pin nv-codec-headers](#22-critical-fix-pin-nv-codec-headers)
  - [2.3 Build the image](#23-build-the-image)
  - [2.4 Verify the libvmaf_cuda filter](#24-verify-the-libvmaf_cuda-filter)
- [Part 3 — Running VMAF with GPU acceleration](#part-3--running-vmaf-with-gpu-acceleration)
  - [3.1 capabilities=video](#31-capabilitiesvideo)
  - [3.2 Full analysis command](#32-full-analysis-command)
- [Troubleshooting](#troubleshooting)
- [Conclusion](#conclusion)

## The Problem

If you've ever tried to calculate VMAF on Windows, you already know that:

- The standard `libvmaf` filter works, but it runs on the **CPU** (slow).
- `libvmaf_cuda` **does not exist in any prebuilt Windows binary** (not in gyan.dev, not in BtbN, not anywhere *that I know*).
- Most information online is scattered, outdated, or simply missing.

## Prerequisites

Before starting, make sure you have:

| Component | Minimum requirement |
|---|---|
| **Windows** | Windows 10 (version 2004+) or Windows 11 |
| **NVIDIA GPU** | Any CUDA-capable GPU (I'm using an RTX 3060 Mobile) |
| **NVIDIA drivers** | Version 525+ (for CUDA 12.x) — [download here](https://www.nvidia.com/Download/index.aspx) |
| **Disk space** | ~30 GB free (WSL + Docker + images) |
| **RAM** | 16 GB recommended (WSL2 is memory-hungry) |

> ⚠️ **Do not install NVIDIA drivers inside WSL.** WSL automatically uses the drivers from Windows. Installing Linux drivers inside WSL will break GPU passthrough.

## Part 1 — Setting up the environment

This part covers everything needed to get a working Linux + Docker + GPU stack: WSL2, Docker Desktop, and the NVIDIA Container Toolkit.

### 1.1 Install WSL2


Open **PowerShell as Administrator** and run

```powershell
wsl --install
```

This command will:
- download and install WSL2 kernel
- install Ubuntu distro by default

**Restart Windows** when prompted.

After the restart, open PowerShell again and verify:

```powershell
wsl --list --verbose
```

Expected output:

```text
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

Make sure `VERSION` is `2`. If it says `1`, upgrade with:

```powershell
wsl --set-version Ubuntu 2
wsl --set-default-version 2
```

#### Ubuntu update

Open your Ubuntu terminal either by typing in PowerShell

```powershell
ubuntu
```
or 

```powershell
wsl
```

Once in Ubuntu terminal enter:

```bash
sudo apt update && sudo apt upgrade -y
```

### 1.2 Accessing Windows drives from WSL

If you're new to Linux, one of the first things to understand is that WSL doesn't use `C:\`, `D:\`, etc. Instead, it mounts every Windows drive under `/mnt/`.

| Windows path | WSL path |
|---|---|
| `C:\Users\YourName\Videos` | `/mnt/c/Users/YourName/Videos` |
| `D:\Movies` | `/mnt/d/Movies` |
| `E:\Backups\2026` | `/mnt/e/Backups/2026` |

### 1.3 Verify GPU access from WSL

if you already have installed NVIDIA drivers on Windows, run:

```bash
nvidia-smi
``` 
You should see the `nvidia-smi` table with your GPU listed. If it doesn't work, your Windows drivers are outdated or WSL isn't configured properly.

```bash
Wed Sep 16 09:30:35 2026
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 615.71.08              KMD Version: 616.92        CUDA UMD Version: 13.4     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3060 ...    On  |   00000000:01:00.0  On |                  N/A |
| N/A   55C    P8             14W /  115W |    1085MiB /   6144MiB |      6%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
``` 

### 1.4 Install Docker Desktop

Download Docker Desktop from: https://www.docker.com/products/docker-desktop/

During installation, make sure to check **"Use WSL 2 instead of Hyper-V"**.

#### Configure WSL2 integration

Once installed, open Docker Desktop and go to Settings 

![docker_desktop_settings](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/rbljfannb88p1c6avmd8.png)

**General tab:**
- Check **Use the WSL 2 based engine**
- Click **Apply & Restart**

![docker_desktop_WSL_integration_backend](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/0iw77q5pw4c1fcxv3gad.png)

**Resources → WSL Integration:**
- Check **Enable integration with my default WSL distro**
- Turn on the Ubuntu toggle

![docker_desktop_WSL_integration](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/nhu8cz5wialai51ybwop.png)

**Resources → Advanced (optional):**
- Uncheck **"Enable Resource Saver"**

![docker_desktop_settings_resources](https://dev-to-uploads.s3.us-east-2.amazonaws.com/uploads/articles/lst6dufyc6q65wj8t65a.png)

> If "Resource Saver" is enabled, Docker will suspend WSL2 after inactivity, and you'll have to restart it manually.

#### Verify Docker works in WSL

In your Ubuntu terminal:

```bash
docker --version
```

Expected output:

```text
Docker version 27.3.1, build ce12230
```

> ⚠️ If you get `var/run/docker.sock: connect: permission denied.` jump to the [Troubleshooting](#varrundockersock-connect-permission-denied) section.

### 1.5 Install the NVIDIA Container Toolkit

This is the component that allows Docker containers to access the GPU.

#### Install

In your WSL Ubuntu terminal:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg && \
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit
```

#### Configure the Docker runtime

```bash
sudo nvidia-ctk runtime configure --runtime=docker
```

Expected output:

```text
INFO[0000] Config file does not exist; using empty config
INFO[0000] Wrote updated config to /etc/docker/daemon.json
INFO[0000] It is recommended that docker daemon be restarted.
```

#### Restart Docker

Restart **Docker Desktop** from Windows (either via the restart icon or by quitting and reopening it).

#### Verify Docker can see the GPU

Once Docker Desktop is running again, execute in your WSL terminal:

```bash
docker run --rm --gpus all nvidia/cuda:12.3.2-base-ubuntu22.04 nvidia-smi
```

You should see the `nvidia-smi` table with your GPU listed.

```bash
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 615.71.08              KMD Version: 616.92        CUDA UMD Version: 13.4     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 3060 ...    On  |   00000000:01:00.0  On |                  N/A |
| N/A   55C    P8             14W /  115W |    1085MiB /   6144MiB |      6%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

## Part 2 — Building the easyVmaf image

We'll use the [easyVmaf](https://github.com/gdavila/easyVmaf) project, which ships a `Dockerfile.cuda` configured for GPU-accelerated VMAF.

### 2.1 Clone the repository

```bash
cd ~
git clone --depth 1 https://github.com/gdavila/easyVmaf.git
cd easyVmaf
```

### 2.2 Critical fix: pin nv-codec-headers

> 💡 **This step is undocumented anywhere else.** I figured it out after spending hours debugging the compilation error with the help of AI. Skip it and your build will fail.

The original `Dockerfile.cuda` clones the **latest** version of `nv-codec-headers`, which is **incompatible with FFmpeg 8.1**. You'll get this error during the build:

```text
libavcodec/nvenc.c:2529:42: error: 'NV_ENC_CLOCK_TIMESTAMP_SET' 
has no member named 'countingType'; did you mean 'countingTypeLSB'?
```

In recent versions, NVIDIA renamed `countingType` to `countingTypeLSB/countingTypeMSB`, but FFmpeg 8.1 still uses `countingType`.


**The fix:** Pin `nv-codec-headers` to version `n12.1.14.0`, which still uses `countingType` **and** already includes the modern CUDA functions (`cuStreamCreateWithPriority`, `cuMemHostAlloc`, etc.) that `libvmaf_cuda` needs.

Edit the Dockerfile:

```bash
nano Dockerfile.cuda
```

- Press `CTRL+W`, type `nv-codec-headers`, press `ENTER`. You'll land on this line:

```dockerfile
RUN git clone --depth 1 https://git.videolan.org/git/ffmpeg/nv-codec-headers.git && \
    cd nv-codec-headers && \
    make install
```

Change it by adding `n12.1.14.0` before --depth:

```dockerfile
RUN git clone --branch n12.1.14.0 --depth 1 https://git.videolan.org/git/ffmpeg/nv-codec-headers.git && \
    cd nv-codec-headers && \
    make install
```

Save with `CTRL+X`, then `Y`, then `ENTER`.

### 2.3 Build the image

```bash
docker build -f Dockerfile.cuda -t easyvmaf:cuda .
```

### 2.4 Verify the `libvmaf_cuda` filter

> ⚠️ The `easyvmaf:cuda` image defines `easyVmaf` (its own CLI) as the `ENTRYPOINT`. To run raw `ffmpeg`, you must override the entrypoint.

```bash
docker run --rm --gpus all --entrypoint ffmpeg easyvmaf:cuda -filters | grep -E "libvmaf|scale_cuda"
```

Expected output:

```text
.. libvmaf           VV->V      Calculate the VMAF between two video streams.
.. libvmaf_cuda      VV->V      Calculate the VMAF between two video streams.
.. scale_cuda        V->V       GPU accelerated video resizer
```
**If you see `libvmaf_cuda`, you're done with the setup.**

## Part 3 — Running VMAF with GPU acceleration

### 3.1 capabilities=video

By default, `--gpus all` alone only grants the compute and utility capabilities. It does not mount the video decode/encode libraries `libnvcuvid.so.1` (NVDEC) and `libnvidia-encode.so.1` (NVENC). Since we tell FFmpeg to decode with `-hwaccel cuda`, it needs those libraries. Without them, FFmpeg fails with:

```text
Cannot load libnvcuvid.so.1
Failed loading nvcuvid.
Failed setup for format cuda: hwaccel initialisation returned error.
```

**The fix:** explicitly request the `video` capability:

```bash
--gpus all,capabilities=video
```

Now Docker mounts `libcuda.so.1` (CUDA compute), `libnvcuvid.so.1` (NVDEC), and `libnvidia-encode.so.1` (NVENC). FFmpeg can then decode, filter, and analyze entirely on the GPU.

### 3.2 Full analysis command

```bash
docker run --gpus all,capabilities=video --rm --entrypoint ffmpeg \
  -v "/path/to/your/videos":/videos \
  easyvmaf:cuda \
  -hwaccel cuda -hwaccel_output_format cuda \
  -i "/videos/distorted.mkv" \
  -hwaccel cuda -hwaccel_output_format cuda \
  -i "/videos/reference.mkv" \
  -filter_complex "[0:v]scale_cuda=format=yuv420p[dis];[1:v]scale_cuda=format=yuv420p[ref];[dis][ref]libvmaf_cuda=log_fmt=json:log_path=/videos/vmaf_full.json" \
  -f null -
```


#### Real-world result

Here's the output from a test on a 46-minute video file:

```text
[Parsed_libvmaf_cuda_2 @ 0x760b44004f80] VMAF score: 93.558906
speed=14.9x elapsed=0:03:05.05
[out#0/null @ 0x5ef225e22140] video:27438KiB audio:2072848KiB subtitle:0KiB
frame=66265 fps=350 q=-0.0 Lsize=N/A time=00:46:03.79 bitrate=N/A speed=14.6x elapsed=0:03:09.43
```
**3 minutes for a 46-minute video at ~15x real-time speed.** That's the whole point of using `libvmaf_cuda`.

## Troubleshooting

### `var/run/docker.sock: connect: permission denied`

```text
docker: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Head "http://%2Fvar%2Frun%2Fdocker.sock/_ping": dial unix /var/run/docker.sock: connect: permission denied.
```

Your user doesn't belong to the `docker` group, which owns `/var/run/docker.sock`.

fix: 

enter this command in your WSL Ubuntu terminal

```bash
sudo usermod -aG docker $USER
```

This adds your user to the `docker` group. Then **close and reopen WSL** (group changes only apply to new sessions), and verify:

```bash
groups
```

Expected output:

```text
youruser adm cdrom sudo dip plugdev users docker
```


### `Cannot load libnvcuvid.so.1`

Missing `,capabilities=video` in the `--gpus` flag. See [section 3.1](#31-capabilitiesvideo).


### `NV_ENC_CLOCK_TIMESTAMP_SET has no member named 'countingType'`

You didn't pin `nv-codec-headers` to `n12.1.14.0`. See [section 2.2](#22-critical-fix-pin-nv-codec-headers).

### WSL2 hangs after inactivity

Edit `C:\Users\YOUR_USER\.wslconfig`:

```ini
[wsl2]
vmIdleTimeout=-1
```

Then run `wsl --shutdown` in PowerShell. Also disable "Resource Saver" in Docker Desktop (see [section 1.3](#14-install-docker-desktop)).

## Conclusion

Setting up `libvmaf_cuda` on Windows was a long journey. At the start, I couldn't find much information about it — most guides either stop at "use `libvmaf` on CPU" or assume you're on Linux. Even though this isn't 100% native to Windows (it runs through WSL2 + Docker), it's a solid alternative that is absolutely worth the effort.

Once it's working, you get VMAF analysis at **15x real-time speed**, which completely changes what's practical for video quality workflows.

## References

- [easyVmaf](https://github.com/gdavila/easyVmaf) — the project powering this guide
- [NVIDIA Container Toolkit docs](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/index.html)
- [WSL2 GPU support](https://learn.microsoft.com/en-us/windows/wsl/tutorials/gpu-compute)
- [Netflix VMAF](https://github.com/Netflix/vmaf)

