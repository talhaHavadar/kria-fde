# How Yocto Builds the aarch64-xilinx-elf Toolchain

## Overview

Yocto doesn't download a pre-built toolchain from Xilinx. Instead, it builds the entire toolchain from source using standard GCC, binutils, and newlib. The key is in the configuration - Yocto uses specific target triplets and build flags to create a baremetal/standalone toolchain compatible with Xilinx tools.

## The Build Process

### 1. Target System Configuration

When building for standalone/baremetal targets, Yocto uses:
- **DISTRO**: `xilinx-standalone`
- **TARGET_VENDOR**: `-xilinx`
- **TCLIBC**: `newlib` (instead of glibc for Linux)
- **Target Triplet**: `aarch64-xilinx-elf`

This is configured in `meta-xilinx/meta-xilinx-standalone/conf/distro/xilinx-standalone.inc`:
```
TARGET_VENDOR = "-xilinx"
TCLIBC = "newlib"
```

### 2. GCC Cross-Compiler Build

Yocto builds GCC from the standard upstream sources (GCC 13.3 in this case) with these key configurations:

```bash
./configure \
    --build=x86_64-linux \
    --host=x86_64-linux \
    --target=aarch64-xilinx-elf \
    --program-prefix=aarch64-xilinx-elf- \
    --with-newlib \
    --disable-threads \
    --disable-shared \
    --enable-languages=c,c++,fortran \
    --with-arch=armv8-a \
    --disable-libssp \
    --disable-libgomp \
    --disable-libmudflap \
    --disable-libquadmath \
    --disable-libquadmath-support
```

Key points:
- `--target=aarch64-xilinx-elf`: Creates the xilinx-elf target
- `--with-newlib`: Uses newlib instead of glibc
- `--disable-threads`: No threading support (baremetal)
- `--disable-shared`: Static libraries only

### 3. Binutils Build

Similarly, binutils (assembler, linker, etc.) is built from standard sources targeting `aarch64-xilinx-elf`.

### 4. Newlib C Library

Instead of glibc, Yocto builds newlib - a lightweight C library suitable for embedded systems:

```bash
./configure \
    --host=aarch64-xilinx-elf \
    --enable-newlib-io-long-long \
    --enable-newlib-io-c99-formats \
    --disable-newlib-multithread
```

### 5. The Build Sequence

The toolchain is built in multiple stages:

1. **Stage 1**: Build a minimal cross-compiler (gcc-cross-initial)
2. **Stage 2**: Build newlib using the stage 1 compiler
3. **Stage 3**: Build the full cross-compiler (gcc-cross) with newlib support
4. **Stage 4**: Build additional libraries (libgcc, libstdc++)

## Recipes Involved

The main recipes that build the toolchain:

1. **gcc-cross_13.3.bb**: The cross-compiler
   - Located in: `poky/meta/recipes-devtools/gcc/`
   - Modified by xilinx-standalone distro settings

2. **newlib_4.4.0.bb**: The C library
   - Located in: `meta-arm/meta-arm-toolchain/recipes-devtools/newlib/`
   
3. **binutils-cross_2.42.bb**: Assembler, linker, etc.
   - Located in: `poky/meta/recipes-devtools/binutils/`

## Building Your Own Toolchain

To build just the toolchain using Yocto:

```bash
# Set up for standalone/baremetal build
MACHINE=k26-sm-sdt DISTRO=xilinx-standalone bitbake meta-toolchain

# Or build individual components
MACHINE=k26-sm-sdt DISTRO=xilinx-standalone bitbake gcc-cross-aarch64
MACHINE=k26-sm-sdt DISTRO=xilinx-standalone bitbake newlib
```

The resulting toolchain will be in:
- `tmp-k26-sm-sdt-cortexa53-fsbl/sysroots/x86_64-linux/usr/bin/aarch64-xilinx-elf/`

## Why Not Use Upstream aarch64-none-elf?

You might wonder why not use the standard ARM bare-metal toolchain (aarch64-none-elf). The xilinx-elf variant:

1. Uses specific optimization flags optimized for Xilinx devices
2. Includes patches for Xilinx-specific features
3. Maintains compatibility with Xilinx SDK/Vitis expectations
4. Uses the same specs files as Xilinx tools

## Manual Toolchain Build (Without Yocto)

If you want to build the toolchain manually without Yocto:

```bash
# 1. Download sources
wget https://ftp.gnu.org/gnu/gcc/gcc-13.3.0/gcc-13.3.0.tar.xz
wget https://ftp.gnu.org/gnu/binutils/binutils-2.42.tar.xz
wget ftp://sourceware.org/pub/newlib/newlib-4.4.0.tar.gz

# 2. Build binutils
tar xf binutils-2.42.tar.xz
mkdir build-binutils && cd build-binutils
../binutils-2.42/configure \
    --target=aarch64-xilinx-elf \
    --prefix=/opt/xilinx-toolchain \
    --disable-nls
make -j$(nproc)
make install

# 3. Build GCC (stage 1)
tar xf gcc-13.3.0.tar.xz
cd gcc-13.3.0
./contrib/download_prerequisites
cd ..
mkdir build-gcc && cd build-gcc
../gcc-13.3.0/configure \
    --target=aarch64-xilinx-elf \
    --prefix=/opt/xilinx-toolchain \
    --enable-languages=c,c++ \
    --with-newlib \
    --without-headers \
    --disable-shared \
    --disable-threads \
    --disable-libssp \
    --disable-libgomp \
    --disable-libmudflap \
    --disable-libquadmath \
    --disable-libquadmath-support
make -j$(nproc) all-gcc
make install-gcc

# 4. Build newlib
export PATH=/opt/xilinx-toolchain/bin:$PATH
tar xf newlib-4.4.0.tar.gz
mkdir build-newlib && cd build-newlib
../newlib-4.4.0/configure \
    --target=aarch64-xilinx-elf \
    --prefix=/opt/xilinx-toolchain \
    --enable-newlib-io-long-long \
    --enable-newlib-io-c99-formats \
    --disable-newlib-multithread
make -j$(nproc)
make install

# 5. Build GCC (final)
cd ../build-gcc
make -j$(nproc)
make install
```

## Conclusion

Yocto builds the xilinx-elf toolchain from standard open-source components (GCC, binutils, newlib) with specific configuration flags. The "magic" is in:
1. The target triplet configuration (aarch64-xilinx-elf)
2. Using newlib instead of glibc
3. Disabling features not needed for baremetal (threads, shared libs)
4. Applying any Xilinx-specific patches through the meta-xilinx layer

This approach gives you a fully open-source toolchain that's compatible with Xilinx's tools and hardware, without requiring proprietary downloads.