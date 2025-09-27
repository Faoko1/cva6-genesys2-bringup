# cva6-genesys2-bringup
Reproducible notes for building OpenHW CVA6 (v5.3) on Digilent Genesys-2: bit/mcs build, flashing, boot, and troubleshooting


# CVA6 (OpenHW v5.3.0) on Digilent Genesys-2 — Build & Bring-Up

This repo documents my **repeatable** steps to synthesize the OpenHW **CVA6** APU, generate a **Genesys-2** bitstream/MCS, flash it, and boot via UART. It’s meant for reviewers and employers to see a clean, professional process.

> Upstream CVA6: https://github.com/openhwgroup/cva6 (tag `v5.3.0`)  
> Board: Digilent Genesys-2 (XC7K325T)

---

## Quick Start

1. **Clone CVA6 (v5.3.0) with submodules**
   ```bash
   git clone --branch v5.3.0 --depth 1 --recurse-submodules https://github.com/openhwgroup/cva6
   cd cva6
   git submodule sync --recursive
   git submodule update --init --recursive --checkout

