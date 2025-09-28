# cva6-genesys2-bringup  
Reproducible notes for building OpenHW CVA6 (v5.3) on Digilent Genesys-2: bit / mcs build, flashing, boot, and troubleshooting

# CVA6 (OpenHW v5.3.0) on Digilent Genesys-2 — Build & Bring-Up

This repository documents my **reproducible process** to synthesize the OpenHW **CVA6** APU, generate a **Genesys-2** (Kintex-7 XC7K325T) bitstream/MCS, flash it, and boot via UART — so reviewers, collaborators, or maintainers can follow exactly, spot issues, or reproduce reliably.

Upstream: https://github.com/openhwgroup/cva6 (tag `v5.3.0`)  
Target board: Digilent Genesys-2 (XC7K325T)

---

## Prerequisites & Environment Setup

- **Vivado / Xilinx toolchain**  
  The upstream Genesys-2 FPGA flow is validated on **Vivado 2018.2**; newer versions may work but may require IP regeneration or tweaks.  
- **RISC-V cross toolchain**  
  Set your `RISCV` environment variable to your RISC-V toolchain prefix.  
- **Git + submodules**  
  Clone with submodules (or init them) so all supporting IP is present.  
- **Useful extras**  
  - Use fast local temporary storage (e.g. `/scratch`, `/dev/shm`) for Vivado build artifacts  
  - Use `screen` or PuTTY for serial logs  
  - Use `nproc` / parallel make for faster builds

Example setup (place in an `env.sh` or run manually before builds):

~~~bash
export CVA6_REPO_DIR=$PWD
export RISCV=/opt/riscv
export XILINX_VIVADO=/opt/Xilinx/Vivado/2018.2

export HPDCACHE_DIR=$CVA6_REPO_DIR/core/cache_subsystem/hpdcache
export HPDCACHE_TARGET_CFG=$CVA6_REPO_DIR/core/include/cv64a6_imafdc_sv39_hpdcache_config_pkg.sv

export NTHREADS="${NTHREADS:-$(nproc)}"
export MAKEFLAGS="-j${NTHREADS}"

[ -f "$XILINX_VIVADO/settings64.sh" ] && source "$XILINX_VIVADO/settings64.sh"

if [ -d "/scratch" ]; then
  FAST="/scratch/${USER}/vivado_fast"
else
  FAST="/dev/shm/${USER}/vivado_fast"
fi
mkdir -p "$FAST/tmp" "$FAST/.Xil"
export TMPDIR="$FAST/tmp"
export XILINX_LOCAL_USER_DATA="$FAST/.Xil"
export XILINX_LOCAL_CACHE_DIR="$FAST/.Xil"
~~~

Run the above from **inside the CVA6 repo root** so relative paths work.

---

## Clone CVA6 (v5.3.0) & Submodules

~~~bash
mkdir -p ~/work/cva6 && cd ~/work/cva6
git clone --branch v5.3.0 --depth 1 --recurse-submodules https://github.com/openhwgroup/cva6
cd cva6
git submodule sync --recursive
git submodule update --init --recursive --checkout
~~~

Ensure all submodules (especially IP, hpdcache, tools) are correct before building.

---

## Build Bitstream & MCS

From the CVA6 root directory:

~~~bash
make fpga BOARD=genesys2
~~~

This should produce:

- `corev_apu/fpga/work-fpga/ariane_xilinx.bit`  
- `corev_apu/fpga/work-fpga/ariane_xilinx.mcs`

These files are your FPGA configuration outputs: `.bit` for volatile JTAG loads, `.mcs` for flash.

---

## FPGA Flashing & Bitstream Programming (Vivado)

### Flash the `.mcs` into QSPI flash (persistent boot)

1. Set **JP5 = JTAG**, power on the board.  
2. In Vivado: **Hardware Manager → Open Target → Auto Connect**  
3. Add configuration memory device:  
   - Right-click FPGA → **Add Configuration Memory Device…**  
   - Select `s25fl256sxxxxxx0-spi-x1_x2_x4` (the 256 Mbit QSPI flash memory)  
   - Confirm  
4. Program the flash:  
   - Right-click the flash device → **Program Configuration Memory Device…**  
   - Set **Configuration file** to `ariane_xilinx.mcs`  
   - Leave **PRM file** blank (unless you created one)  
   - Under options: enable **Erase**, **Program**, **Verify** (optionally **Blank Check**)  
   - Choose **Configuration File Only**  
   - Use **Pull-down** for non-config I/O pin state  
   - Click **OK / Program**  
5. After successful programming, set **JP5 = QSPI**, then **power-cycle** the board. Now the FPGA should auto-load from QSPI at each boot.  
6. (Optional) Press **PROG** to force reload from flash manually.

> Note: “Configuration File Only” restricts operations to only the flash region covered by the `.mcs`, making the process safer and faster.  
> Vivado uses an indirect configuration mechanism: it loads a mini SPI writer into the FPGA using JTAG, then streams your `.mcs` into flash.

### Program `.bit` over JTAG (for fast iteration)

- Keep **JP5 = JTAG**  
- In Vivado Hardware Manager → **Program Device…**  
- Select `ariane_xilinx.bit`, click **Program**  
- This is volatile and won't persist after reboot, but is great during development.

---

## Boot / Console

- Connect the USB-UART cable (FTDI) to your host machine  
- Use PuTTY (Windows) or `screen /dev/ttyUSB* 115200` (Linux/macOS) at **115200** baud  
- If your OS is on an SD card, insert it  
- On power-up (with JP5 = QSPI), you should see: FPGA configuring → U-Boot → OpenSBI → Linux logs on the serial console

---

## Persistent Boot: Always Load OS

To make sure the board auto-boots Linux every time:

1. Ensure the FPGA is set to auto load (JP5 = QSPI, `.mcs` programmed)  
2. Configure U-Boot environment so it automatically loads kernel + DTB  
3. Save that environment so it persists across boots



## Troubleshooting & Lessons Learned

- **“sbi_trap_error: hart0: trap handler failed (error –2)”**  
  This occurs **after** “Starting kernel …” and usually means there is a mismatch between the DTB, memory map, OpenSBI, or kernel entry address — not typically a jumper or flash configuration issue.

- **No or garbled UART output**  
  - Verify that baud is set to **115200**  
  - Ensure the UART base address in the DTB matches your hardware  
  - If `earlycon=` is not enabled and early console fails, the kernel might crash before console initialization

- **FPGA does not load bitstream from flash**  
  - Check that **JP5** is set to **QSPI**  
  - Make sure your `.mcs` was successfully written into the flash device  
  - Press the **PROG** button to force a reconfiguration from flash manually

- **Vivado cannot find the correct flash part**  
  - Use the search box in the “Add Configuration Memory Device” dialog to filter by “s25fl256”  
  - Pick the variant ending in **0** that supports SPI x1, x2, and x4

- **Process gets stuck on "Copying Boot Image Done"**  
  - Flash the sd card properly or use a different one.
    
---

## Suggestions / To-Do

- Document **jumper settings** (JP5, JP4) and when to use each position  
- Insert your exact working `extlinux.conf` or `bootcmd` snippet that reliably boots Linux  
- Add a “lessons learned” summary explaining how you resolved traps, what pitfalls to watch out for, etc.  
- Include your Vivado version and submodule commit hashes for reproducibility  
- (Optional) Describe how to embed the Linux payload inside OpenSBI (using `FW_PAYLOAD`) to bypass U-Boot entirely  
