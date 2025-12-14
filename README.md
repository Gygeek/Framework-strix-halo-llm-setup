# Framework-strix-halo-llm-setup
Complete guide to running large language models locally on AMD Ryzen AI Max+ 395 (Strix Halo) with 128GB unified memory. Covers BIOS config, kernel setup, ROCm installation, and llama.cpp deployment. Run 70B+ parameter models on a single APU.

# Running Local LLMs on AMD Ryzen AI Max+ 395 (Strix Halo)

A complete guide to deploying large language models on AMD's Ryzen AI Max+ 395 APU with 128GB unified memory.

## Acknowledgments

This guide builds on the excellent work of others in the Strix Halo community:

- **[kyuz0/amd-strix-halo-toolboxes](https://github.com/kyuz0/amd-strix-halo-toolboxes)** â€” Pre-built container images with ROCm and rocWMMA optimizations
- **[pablo-ross/strix-halo-gmktec-evo-x2](https://github.com/pablo-ross/strix-halo-gmktec-evo-x2)** â€” Ubuntu-optimized setup guide for GMKTEC hardware

## Tested Hardware

This guide has been tested on systems with the AMD Ryzen AI Max+ 395:

| System | Notes |
|--------|-------|
| **Framework Desktop** | Framework's Strix Halo desktop system |
| **GMKTEC EVO-X2** | Compact mini PC with Strix Halo APU |

The configuration should work on any system using the Ryzen AI Max+ 395 (or similar Strix Halo APUs), though BIOS menu locations may vary by manufacturer.

## Overview

The AMD Ryzen AI Max+ 395 (codenamed "Strix Halo") is a powerful APU that can run large language models locally using its integrated Radeon 8060S GPU and up to 128GB of shared system memory. This guide walks you through the complete setup process on Ubuntu.

### What Makes This Hardware Special

- **128GB Unified Memory:** Unlike discrete GPUs with fixed VRAM, the Strix Halo shares system RAM with the GPU via GTT (Graphics Translation Table)
- **~115-120GB available for LLM inference:** After configuration, you can load models that would require multiple high-end GPUs
- **gfx1151 Architecture:** RDNA 3.5 with 40 Compute Units and rocWMMA support for fast matrix operations

### Performance Expectations

| Model | Tokens/Second | Notes |
|-------|---------------|-------|
| Llama-2-7B | ~50-52 t/s | Comparable to Apple M4 Max |
| 70B models (Q4) | ~15-20 t/s | Fits entirely in memory |
| Long context (8K+) | Excellent | ROCm rocWMMA excels here |

---

## Prerequisites

- **Hardware:** System with AMD Ryzen AI Max+ 395 (e.g., Framework Desktop, GMKTEC EVO-X2)
- **OS:** Ubuntu 24.04 LTS or later (this guide tested on Ubuntu 25.10)
- **BIOS Access:** Required for initial configuration

---

## Phase 1: BIOS Configuration

Before installing or configuring the OS, set these BIOS options:

### 1.1 Set VRAM to 512MB

Navigate to: `Integrated Graphics` â†’ `UMA Frame Buffer Size` â†’ `512MB`

> **Note:** This is just the display framebuffer. The actual compute memory (GTT) is configured separately and can use most of your system RAM.

### 1.2 Disable IOMMU

Find the IOMMU setting and set to `Disabled`.

- Provides ~6% memory read improvement
- Only enable if you need VFIO/GPU passthrough later

### 1.3 Set Power Mode (Optional)

Configure TDP to 85W for optimal performance/efficiency balance.

---

## Phase 2: Kernel and Boot Configuration

### 2.1 Verify Kernel Version

Kernel 6.16.9 or later is required for full memory access:

```bash
uname -r
```

Ubuntu 24.04+ with HWE kernel or Ubuntu 25.10 should meet this requirement. If not, install a newer kernel:

```bash
sudo add-apt-repository ppa:cappelikan/ppa -y
sudo apt update
sudo apt install mainline -y
sudo mainline --install 6.16.9
```

### 2.2 Configure GRUB Boot Parameters

Edit the GRUB configuration:

```bash
sudo nano /etc/default/grub
```

Find the line starting with `GRUB_CMDLINE_LINUX_DEFAULT` and modify it:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_iommu=off amdgpu.gttsize=117760"
```

**Parameter explanation:**
- `amd_iommu=off` â€” Disables IOMMU for reduced latency
- `amdgpu.gttsize=117760` â€” Sets GTT to ~115GB (adjust based on your RAM)

Apply changes:

```bash
sudo update-grub
sudo reboot
```

### 2.3 Verify Configuration After Reboot

```bash
# Check boot parameters were applied
cat /proc/cmdline | grep -o "amd_iommu=off\|amdgpu.gttsize=[0-9]*"

# Check GTT memory allocation (should show ~115-120GB in bytes)
cat /sys/class/drm/card*/device/mem_info_gtt_total
```

---

## Phase 3: GPU Access Configuration

### 3.1 Create udev Rules

This ensures proper permissions for GPU access:

```bash
sudo bash -c 'cat > /etc/udev/rules.d/99-amd-kfd.rules << EOF
SUBSYSTEM=="kfd", GROUP="render", MODE="0666"
SUBSYSTEM=="drm", KERNEL=="card[0-9]*", GROUP="render", MODE="0666"
SUBSYSTEM=="drm", KERNEL=="renderD[0-9]*", GROUP="render", MODE="0666"
EOF'
```

Reload rules:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

### 3.2 Add User to GPU Groups

```bash
sudo usermod -aG video,render $USER
```

Log out and back in for group changes to take effect.

---

## Phase 4: ROCm Installation

### Option A: Install ROCm from AMD Repository

```bash
# Add AMD ROCm repository
wget https://repo.radeon.com/rocm/rocm.gpg.key -O - | sudo apt-key add -
echo 'deb [arch=amd64] https://repo.radeon.com/rocm/apt/latest ubuntu main' | sudo tee /etc/apt/sources.list.d/rocm.list

sudo apt update
sudo apt install rocm-hip-runtime rocm-hip-sdk -y
```

### Option B: Use Container with Pre-configured ROCm (Recommended)

The container approach isolates ROCm and provides optimized builds with rocWMMA support.

Install Podman and Distrobox:

```bash
sudo apt install podman -y
curl -s https://raw.githubusercontent.com/89luca89/distrobox/main/install | sudo sh
```

Create the LLM container:

```bash
distrobox create llama-rocm \
  --image docker.io/kyuz0/amd-strix-halo-toolboxes:rocm-6.4.4-rocwmma \
  --additional-flags "--device /dev/dri --device /dev/kfd --group-add video --group-add render"
```

### 4.1 Add ROCm to PATH

Whether using host ROCm or container, add to your `.bashrc`:

```bash
echo 'export PATH=$PATH:/opt/rocm/bin' >> ~/.bashrc
source ~/.bashrc
```

### 4.2 Verify ROCm Installation

```bash
rocminfo | grep -E "Agent|Name:|Marketing"
```

Expected output should show both CPU (Agent 1) and GPU (Agent 2: gfx1151).

---

## Phase 5: llama.cpp Setup

### If Using Container (Option B)

Enter the container:

```bash
distrobox enter llama-rocm
```

The kyuz0 container images include a pre-built llama.cpp. Verify:

```bash
llama-cli --version
```

### If Building from Source

```bash
# Install dependencies (Ubuntu)
sudo apt install cmake gcc g++ git libcurl4-openssl-dev -y

# Or in Fedora-based container
sudo dnf install cmake gcc-c++ git libcurl-devel -y

# Clone and build
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp

cmake -B build -S . \
  -DGGML_HIP=ON \
  -DAMDGPU_TARGETS="gfx1151"

cmake --build build --config Release -j$(nproc)
```

---

## Phase 6: Download Models and Run Inference

### 6.1 Set Up Model Directory

```bash
mkdir -p ~/models
cd ~/models
```

### 6.2 Download a Model

```bash
# Install HuggingFace CLI
pip install "huggingface-hub[cli]"

# Download a model (example: Llama-2-7B for testing)
huggingface-cli download TheBloke/Llama-2-7B-GGUF llama-2-7b.Q4_K_M.gguf --local-dir ~/models
```

### 6.3 Run Inference

```bash
llama-cli \
  -m ~/models/llama-2-7b.Q4_K_M.gguf \
  --no-mmap \
  -ngl 99 \
  -p "Explain quantum computing in simple terms:" \
  -n 256
```

**Critical flags:**
- `--no-mmap` â€” Required for GPU backends on Strix Halo
- `-ngl 99` â€” Offload all layers to GPU (use high number)

### 6.4 Run Benchmark

```bash
llama-bench \
  -m ~/models/llama-2-7b.Q4_K_M.gguf \
  -mmp 0 \
  -ngl 99 \
  -p 512 \
  -n 128
```

> **Note:** `llama-bench` uses `-mmp 0` instead of `--no-mmap`

---

## Monitoring

### Watch GPU Usage

```bash
watch -n 1 rocm-smi
```

### Check Memory Allocation

```bash
# GTT total
cat /sys/class/drm/card*/device/mem_info_gtt_total

# GTT used
cat /sys/class/drm/card*/device/mem_info_gtt_used
```

---

## Troubleshooting

### "HSA_STATUS_ERROR_OUT_OF_RESOURCES" when running rocminfo

**Cause:** Missing udev rule for renderD devices.

**Fix:** Ensure `/etc/udev/rules.d/99-amd-kfd.rules` includes the `renderD[0-9]*` rule (see Phase 3), then:

```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

### rocminfo/rocm-smi: command not found

**Fix:** Add ROCm to PATH:

```bash
export PATH=$PATH:/opt/rocm/bin
```

Make permanent by adding to `~/.bashrc`.

### Only ~15GB VRAM visible

**Cause:** Kernel too old.

**Fix:** Upgrade to kernel 6.16.9 or later.

### Slow model loading

**Fix:** Add `--no-mmap` flag to llama-cli.

### Container can't access GPU

**Fix:** Verify device permissions:

```bash
ls -la /dev/kfd /dev/dri/
```

All devices should show `crw-rw-rw-` (mode 0666). If not, check udev rules.

### "cmake: command not found" in container

The ROCm container is Fedora-based. Install build tools with:

```bash
sudo dnf install cmake gcc-c++ git libcurl-devel -y
```

### llama-bench error: "invalid parameter for argument: --no-mmap"

**Fix:** Use `-mmp 0` instead of `--no-mmap` for llama-bench.

---

## GTT Memory Reference

| GTT Size (MB) | Bytes | Usable for Models |
|---------------|-------|-------------------|
| 117760 | ~123 billion | ~115 GB |
| 122880 | ~128 billion | ~120 GB |
| 131072 | ~137 billion | ~128 GB |

Adjust `amdgpu.gttsize` in GRUB based on your total RAM and desired allocation.

---

## Understanding APU Memory Architecture

Unlike discrete GPUs with dedicated VRAM, the Ryzen AI Max uses **unified memory**:

```
â”Œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”
â”‚           128GB System RAM              â”‚
â”œâ”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”¤
â”‚  CPU Available  â”‚  GTT (GPU Compute)    â”‚
â”‚    ~8-13 GB     â”‚     ~115-120 GB       â”‚
â””â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”˜
```

- **VRAM** shown in tools (~1GB) is just the display framebuffer
- **GTT** is what matters for LLM inference
- Memory is shared dynamically â€” unused GTT is available to CPU

---

## Useful Resources

- [llama.cpp GitHub](https://github.com/ggerganov/llama.cpp)
- [ROCm Documentation](https://rocm.docs.amd.com/)
- [kyuz0/amd-strix-halo-toolboxes](https://github.com/kyuz0/amd-strix-halo-toolboxes) â€” Pre-built container images with rocWMMA
- [pablo-ross/strix-halo-gmktec-evo-x2](https://github.com/pablo-ross/strix-halo-gmktec-evo-x2) â€” Ubuntu-optimized setup for GMKTEC
- [AMD ROCm GitHub](https://github.com/ROCm/ROCm)

---

## System Info Script

Save this script to quickly gather your system specifications:

```bash
#!/bin/bash
echo "=== LLM Server Quick Status ==="
echo "Kernel: $(uname -r)"
echo "ROCm: $(cat /opt/rocm/.info/version 2>/dev/null || echo 'Not found')"
echo "GTT: $(echo "scale=1; $(cat /sys/class/drm/card*/device/mem_info_gtt_total 2>/dev/null | head -1) / 1024^3" | bc) GB"
echo "GPU: $(rocminfo 2>/dev/null | grep "Marketing Name:" | grep -v CPU | head -1 | cut -d: -f2 | xargs)"
echo "Containers: $(distrobox list 2>/dev/null | tail -n +2 | wc -l)"
```
