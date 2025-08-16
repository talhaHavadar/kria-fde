# Comprehensive Guide: Building BOOT.bin for K26-SM-SDT Without Yocto

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Obtaining the Toolchain](#obtaining-the-toolchain)
4. [Setting Up the Build Environment](#setting-up-the-build-environment)
5. [Building Components](#building-components)
6. [Assembling BOOT.bin](#assembling-bootbin)
7. [Troubleshooting](#troubleshooting)

## Overview

This guide provides step-by-step instructions for building the BOOT.bin file for Xilinx K26-SM-SDT platform without using Yocto. The BOOT.bin contains:

- FSBL (First Stage Boot Loader)
- PMU Firmware
- ARM Trusted Firmware (ATF/BL31)
- U-Boot
- Device Tree Blob

## Prerequisites

### Hardware

- Host machine running Linux (Ubuntu 20.04 or later recommended)
- At least 16GB RAM
- 50GB free disk space

### Software Requirements

- Git
- CMake (3.16 or later)
- Ninja build system
- Python 3.8+
- Device tree compiler (dtc)
- Standard build tools (gcc, make, etc.)

```bash
# Install basic dependencies on Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y \
    git cmake ninja-build python3 python3-pip \
    build-essential device-tree-compiler \
    libssl-dev libncurses5-dev bc flex bison \
    wget curl unzip
```

## Obtaining the Toolchain

### Option 1: Vitis Installation (Recommended)

The aarch64-xilinx-elf toolchain comes with AMD/Xilinx Vitis installation.

1. Download Vitis from AMD website:

   - Visit: https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vitis.html
   - Download Vitis 2025.1 (or matching your design version)
   - You need a free AMD account to download

2. Install Vitis:

```bash
chmod +x Xilinx_Unified_2025.1_*.bin
./Xilinx_Unified_2025.1_*.bin
# Follow installer, select Vitis and ensure "Install devices for Zynq UltraScale+ MPSoCs" is checked
```

3. Source Vitis settings:

```bash
source /tools/Xilinx/Vitis/2025.1/settings64.sh
```

The toolchain will be available at:

- `/tools/Xilinx/Vitis/2025.1/gnu/aarch64/lin/aarch64-xilinx-elf/`

### Option 2: PetaLinux SDK

PetaLinux also includes the required toolchain.

1. Download PetaLinux from AMD website
2. Install PetaLinux:

```bash
./petalinux-v2025.1-final-installer.run --dir /opt/petalinux
```

3. Source PetaLinux settings:

```bash
source /opt/petalinux/settings.sh
```

### Option 3: Build from Yocto (Advanced)

You can build the toolchain SDK from your existing Yocto setup:

```bash
MACHINE=k26-sm-sdt bitbake meta-toolchain
# or for standalone toolchain:
MACHINE=k26-sm-sdt DISTRO=xilinx-standalone bitbake meta-xilinx-toolchain
```

## Setting Up the Build Environment

Create a working directory structure:

```bash
mkdir -p ~/k26-boot-build/{sources,build,output}
cd ~/k26-boot-build
export WORK_DIR=$PWD
```

Set up environment variables:

```bash
# Adjust path based on your Vitis installation
export XILINX_VITIS=/tools/Xilinx/Vitis/2025.1
export PATH=$XILINX_VITIS/gnu/aarch64/lin/aarch64-xilinx-elf/bin:$PATH
export PATH=$XILINX_VITIS/bin:$PATH

# Verify toolchain is accessible
which aarch64-xilinx-elf-gcc
which bootgen
```

## Building Components

### Step 1: Obtain Board-Specific Files

You need hardware-specific files from your Yocto build or hardware design:

```bash
# Copy from your Yocto build
cp /path/to/yocto/build/tmp-k26-sm-sdt-cortexa53-fsbl/deploy/images/k26-sm-sdt/devicetree/k26-sm-sdt-cortexa53-fsbl.dtb $WORK_DIR/sources/
cp /path/to/yocto/build/tmp-k26-sm-sdt-cortexa53-fsbl/work/*/fsbl-firmware/*/recipe-sysroot/usr/share/sdt/k26-sm-sdt/psu_init.* $WORK_DIR/sources/
```

### Step 2: Clone Required Repositories

```bash
cd $WORK_DIR/sources

# Xilinx embeddedsw (for FSBL and PMU firmware)
git clone https://github.com/Xilinx/embeddedsw.git
cd embeddedsw && git checkout xlnx_rel_v2025.1 && cd ..

# ARM Trusted Firmware
git clone https://github.com/Xilinx/arm-trusted-firmware.git && cd arm-trusted-firmware && git checkout xlnx_rebase_v2.10_2025.1 && cd ..

# U-Boot
git clone https://github.com/Xilinx/u-boot-xlnx.git && cd u-boot-xlnx && git checkout xlnx_rebase_v2024.01_2025.1 && cd ..

# Device Tree
git clone https://github.com/Xilinx/device-tree-xlnx.git
cd device-tree-xlnx
git checkout xlnx_rel_v2025.1
cd ..
```

### Step 3: Build FSBL

```bash
cd $WORK_DIR/build
mkdir fsbl && cd fsbl

# Copy board-specific files
cp $WORK_DIR/sources/psu_init.c $WORK_DIR/sources/embeddedsw/lib/sw_apps/zynqmp_fsbl/src/
cp $WORK_DIR/sources/psu_init.h $WORK_DIR/sources/embeddedsw/lib/sw_apps/zynqmp_fsbl/src/

# Configure FSBL build
cmake -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE=$WORK_DIR/sources/embeddedsw/cmake/toolchainfiles/toolchain_aarch64.cmake \
  -DCMAKE_C_COMPILER=aarch64-xilinx-elf-gcc \
  -DCMAKE_CXX_COMPILER=aarch64-xilinx-elf-g++ \
  -DCMAKE_C_FLAGS="-mcpu=cortex-a53+crc -Os -flto -ffat-lto-objects -DARMA53_64" \
  -DCMAKE_BUILD_TYPE=Release \
  -DBOARD=k26i \
  -DYOCTO=ON \
  $WORK_DIR/sources/embeddedsw/lib/sw_apps/zynqmp_fsbl/src

# Build FSBL
ninja

# Copy output
cp executable.elf $WORK_DIR/output/fsbl.elf
```

### Step 4: Build PMU Firmware

```bash
cd $WORK_DIR/build
mkdir pmufw && cd pmufw

# PMU firmware uses microblaze compiler
export MB_GNU=$XILINX_VITIS/gnu/microblaze/lin

cmake -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE=$WORK_DIR/sources/embeddedsw/cmake/toolchainfiles/toolchain_microblaze.cmake \
  -DCMAKE_C_COMPILER=mb-gcc \
  -DCMAKE_CXX_COMPILER=mb-g++ \
  -DCMAKE_C_FLAGS="-mlittle-endian -mhard-float -mxl-soft-mul -mcpu=v11.0" \
  -DCMAKE_BUILD_TYPE=Release \
  -DBOARD=k26i \
  -DYOCTO=ON \
  $WORK_DIR/sources/embeddedsw/lib/sw_apps/zynqmp_pmufw/src

# Build PMU firmware
ninja

# Copy output
cp executable.elf $WORK_DIR/output/pmufw.elf
```

### Step 5: Build ARM Trusted Firmware

```bash
cd $WORK_DIR/sources/arm-trusted-firmware

# Clean any previous builds
make distclean

# Build ATF for ZynqMP
make CROSS_COMPILE=aarch64-xilinx-elf- \
     PLAT=zynqmp \
     RESET_TO_BL31=1 \
     ZYNQMP_CONSOLE=cadence1 \
     ZYNQMP_BL32_MEM_SIZE=0x1000000 \
     bl31

# Copy output
cp build/zynqmp/release/bl31/bl31.elf $WORK_DIR/output/arm-trusted-firmware.elf
```

### Step 6: Build U-Boot

```bash
cd $WORK_DIR/sources/u-boot-xlnx

# Use the appropriate defconfig for K26
make CROSS_COMPILE=aarch64-xilinx-elf- xilinx_zynqmp_virt_defconfig

# Configure for K26 specifics (you may need to adjust based on your hardware)
cat >> .config << EOF
CONFIG_DEFAULT_DEVICE_TREE="zynqmp-sm-k26-revA"
CONFIG_OF_BOARD_SETUP=y
CONFIG_ZYNQMP_SPL_PM_CFG_OBJ_FILE=""
EOF

# Build U-Boot
make CROSS_COMPILE=aarch64-xilinx-elf- -j$(nproc)

# Copy outputs
cp u-boot.elf $WORK_DIR/output/
cp u-boot.dtb $WORK_DIR/output/
```

### Step 7: Prepare Device Tree Blob

If you need a FIT blob for device tree:

```bash
cd $WORK_DIR/build

# Create FIT source
cat > fit-dtb.its << 'EOF'
/dts-v1/;

/ {
    description = "DTB image for K26";
    #address-cells = <1>;

    images {
        fdt-1 {
            description = "zynqmp-sm-k26";
            data = /incbin/("../output/u-boot.dtb");
            type = "flat_dt";
            arch = "arm64";
            compression = "none";
            load = <0x100000>;
            hash-1 {
                algo = "sha1";
            };
        };
    };

    configurations {
        default = "config-1";
        config-1 {
            description = "zynqmp-sm-k26";
            fdt = "fdt-1";
        };
    };
};
EOF

# Build FIT image
mkimage -f fit-dtb.its $WORK_DIR/output/fit-dtb.blob
```

## Assembling BOOT.bin

### Step 1: Create BIF File

```bash
cd $WORK_DIR/output

cat > bootgen.bif << 'EOF'
the_ROM_image:
{
    [bootloader, destination_cpu=a53-0] fsbl.elf
    [destination_cpu=pmu] pmufw.elf
    [destination_cpu=a53-0, exception_level=el-3, trustzone] arm-trusted-firmware.elf
    [destination_cpu=a53-0, load=0x100000] fit-dtb.blob
    [destination_cpu=a53-0, exception_level=el-2] u-boot.elf
}
EOF
```

### Step 2: Generate BOOT.bin

```bash
bootgen -image bootgen.bif -arch zynqmp -w -o BOOT.bin

# Verify the output
ls -la BOOT.bin
```

## Verification

To verify your BOOT.bin is correctly built:

```bash
# Check BOOT.bin contents
bootgen -arch zynqmp -image bootgen.bif -w -o BOOT.bin -log trace

# Extract components for verification
bootgen -arch zynqmp -image BOOT.bin -dump

# Check file size (should be similar to Yocto-built version)
ls -lh BOOT.bin
```

## Troubleshooting

### Common Issues

1. **Toolchain not found**

   - Ensure Vitis/PetaLinux is properly sourced
   - Check PATH variable includes toolchain location

2. **CMake configuration fails**

   - Verify all prerequisite packages are installed
   - Check CMake version (minimum 3.16)
   - Ensure toolchain file paths are correct

3. **Build failures**

   - Check compiler flags match your hardware variant
   - Verify board-specific files (psu_init.\*) are correct
   - Ensure git repositories are on correct branches

4. **Bootgen errors**
   - Verify all ELF files exist and are valid
   - Check BIF syntax is correct
   - Ensure bootgen version matches your Vitis version

### Debug Tips

1. Enable verbose output:

```bash
# For CMake
cmake -DCMAKE_VERBOSE_MAKEFILE=ON ...

# For Make
make V=1 ...

# For bootgen
bootgen -log trace ...
```

2. Compare with Yocto build:

```bash
# Compare file sizes
ls -la $YOCTO_DEPLOY_DIR/*.elf
ls -la $WORK_DIR/output/*.elf

# Compare bootgen BIF
cat $YOCTO_WORK_DIR/bootgen.bif
```

3. Check dependencies:

```bash
aarch64-xilinx-elf-objdump -x fsbl.elf | grep NEEDED
```

## Additional Resources

- [AMD/Xilinx Bootgen User Guide](https://docs.xilinx.com/r/en-US/ug1283-bootgen-user-guide)
- [Zynq UltraScale+ MPSoC Software Developer Guide](https://docs.xilinx.com/r/en-US/ug1137-zynq-ultrascale-mpsoc-swdev)
- [Xilinx Embeddedsw Repository](https://github.com/Xilinx/embeddedsw)
- [Building FSBL from Source](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18842019/Zynq+UltraScale+FSBL)

## Notes

- This guide assumes you have access to board-specific files (psu_init.c/h) from your hardware design
- The exact git commits and branches should match your target Yocto version
- Some configurations may require additional patches or modifications based on your specific hardware
- Always refer to AMD/Xilinx official documentation for the most up-to-date information
