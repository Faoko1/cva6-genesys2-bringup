# cva6-genesys2-bringup
Reproducible notes for building OpenHW CVA6 (v5.3) on Digilent Genesys-2: bit/mcs build, flashing, boot, and troubleshooting

# CVA6 (OpenHW v5.3.0) on Digilent Genesys-2 — Build & Bring-Up

This repository documents my **reproducible process** to synthesize the OpenHW **CVA6** APU, generate a **Genesys-2** (Kintex-7 XC7K325T) bitstream/MCS, flash it, and boot via UART. It’s written so reviewers/employers can quickly see a clean flow and the issues I solved.

> Upstream CVA6: https://github.com/openhwgroup/cva6 (tag `v5.3.0`)  
> Board: Digilent Genesys-2 (XC7K325T)

---

## Contents
- [Prerequisites](#prerequisites)  
- [Clone CVA6 (v5.3.0) with submodules](#clone-cva6-v530-with-submodules)  
- [Environment (example)](#environment-example)  
- [Build bitstream + MCS (Genesys-2)](#build-bitstream--mcs-genesys-2)  
- [Flash / Program in Vivado](#flash--program-in-vivado)  
- [Console / Boot](#console--boot)  
- [Troubleshooting I actually hit](#troubleshooting-i-actually-hit)  
- [Notes](#notes)  
- [License](#license)

---

## Prerequisites
- **Xilinx Vivado** — upstream validated Genesys-2 flow with **2018.2**. Newer versions (e.g., 2024.x) can work but may require IP regeneration and minor tweaks.  
- **RISC-V toolchain** — the `RISCV` environment variable should point to your RISC-V toolchain prefix (for software/bootrom).  
- **Git with submodules** — you must clone with submodules and keep them in sync.  

Optional but helpful: `screen` or PuTTY for serial, `nproc` for threads, a fast temp directory (e.g., `/scratch` or `/dev/shm`).

---

## Clone CVA6 (v5.3.0) with submodules
```bash
# Choose a working dir
mkdir -p ~/work/cva6 && cd ~/work/cva6

# Clone the exact tag with submodules
git clone --branch v5.3.0 --depth 1 --recurse-submodules https://github.com/openhwgroup/cva6
cd cva6

# Make sure submodules match the meta-commit
git submodule sync --recursive
git submodule update --init --recursive --checkout

--- 





## Environment (example)

## From the CVA6 repo root:
```bash
# paths
export CVA6_REPO_DIR=$PWD
export RISCV=/opt/riscv                              # <-- edit for your machine
export XILINX_VIVADO=/opt/Xilinx/Vivado/2018.2       # <-- edit for your machine

# HPDcache: use the pinned submodule + known-good config
export HPDCACHE_DIR=$CVA6_REPO_DIR/core/cache_subsystem/hpdcache
export HPDCACHE_TARGET_CFG=$CVA6_REPO_DIR/core/include/cv64a6_imafdc_sv39_hpdcache_config_pkg.sv

# threads (optional)
export NTHREADS="${NTHREADS:-$(nproc)}"
export MAKEFLAGS="-j${NTHREADS}"

# Vivado environment (if needed)
[ -f "$XILINX_VIVADO/settings64.sh" ] && source "$XILINX_VIVADO/settings64.sh"


