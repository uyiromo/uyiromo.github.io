---
title: vc707nvmm
layout: default
---

[**Go to Top Page**](https://uyiromo.github.io)

# Introduction
- This page describes the RISC-V emulator employing NVMM performance emulation based on SiFive Freedom SoC
  - [Freedom SoC](https://github.com/sifive/freedom)

# Requirement
## Hardware
- Our emulator uses the folowing FPGA board
  - [Xilinx Virtex-7 FPGA VC707 Evaluation Kit](https://www.xilinx.com/products/boards-and-kits/ek-v7-vc707-g.html)
- 4 GB DIMM
  - TBD
- Ethernet Adapter over PCIe
  - TBD

## Software
- Vivado (recommend: >= 2018.3)
  - We tested by using **Vivado 2018.3.**
  - If you use the newer version, some modification may be required.
- Operating System
  - We tested by using **Ubuntu 18.04.5 LTS**

# Build
## 1. Install required packages
```shell
sudo apt install bimfmt-support libncurses5-dev libncursesw5-dev libpopt-dev uuid-dev
```
- Other packages may be required. If so, also install them.

## 2. Build Freedom SoC
1. Clone vc707nvmm repository and setup it

```shell
git clone https://github.com/uyiromo/freedom.git freedom-devel
cd freedom-devel
git checkout ebb795ff7192ae76fd43a43217cf651a63d8eb00
git submodule update --init --recursive
```

2. Build
```shell
make -j`nproc` -f Makefile.vc707-u500devkit mcs
```
- You can specify the number of Rocket core by option `NUM_CORES=<N>`
  - Default value is 4
  - `NUM_CORES` sets followings:
    - The number of cores in SoC: `new WithNBigCores(N)` @src/main/scala/unleashed/DevKitConfigs.scala
    - The number of cores in ZSBL: `NUM_CORES N` @bootrom/freedom-u540-c000-bootloader/sifive/platform_fpga.h
    - The number of cores in FSBL: `NM_CORES N` @bootrom/freedom-u540-c000-bootloader/fsbl/main.c
- You may face with IP version errors
  - such as `Requested IP 'xilinx.com:ip:axi_pcie:2.8' cannot be created`
  - Replace IP version with that of your Vivado IP library

```git
diff --git a/fpga-shells/src/main/scala/ip/xilinx/vc707axi_to_pcie_x1/vc707axi_to_pcie_x1.scala b/fpga-shells/src/main/scala/ip/xilinx/vc707axi_to_pcie_x1/vc707axi_to_pcie_x1.scala
index 1884238..a2901bf 100644
--- a/fpga-shells/src/main/scala/ip/xilinx/vc707axi_to_pcie_x1/vc707axi_to_pcie_x1.scala
+++ b/fpga-shells/src/main/scala/ip/xilinx/vc707axi_to_pcie_x1/vc707axi_to_pcie_x1.scala
@@ -401,7 +401,7 @@ class VC707AXIToPCIeX1(implicit p:Parameters) extends LazyModule
   ElaborationArtefacts.add(
     "vc707axi_to_pcie_x1.vivado.tcl",
     """ 
-      create_ip -vendor xilinx.com -library ip -version 2.8 -name axi_pcie -module_name vc707axi_to_pcie_x1 -dir $ipdir -force
+      create_ip -vendor xilinx.com -library ip -version 2.9 -name axi_pcie -module_name vc707axi_to_pcie_x1 -dir $ipdir -force
       set_property -dict [list \
       CONFIG.AXIBAR2PCIEBAR_0             {0x40000000} \
       CONFIG.AXIBAR2PCIEBAR_1             {0x00000000} \
```

3. When compilation finishes successfuly, the bitstream is placed at `builds/vc707-u500devkit/obj/VC707PCIeShell.bit`


## 3. Build Linux kernel with OpenSBI
- Using the keystone repository.
1. Clone vc707nvmm repository and setup it
```shell
git clone https://github.com/uyiromo/keystone.git keystone-devel
cd keystone-devel
git checkout vc707nvmm
git submodule update --init --recursive
```

2. Build RISC-V toolchains
```shell
cd riscv-gnu-toolchain
./configure --prefix=$(pwd)/../riscv
make               # newlib gcc (riscv64-unknown-elf-*)
make linux         #    gnu gcc (riscv64-unknown-linux-gnu-*)
export RISCV=`pwd`/../riscv
export PATH=$RISCV/bin:$PATH
```

3. Build Keystone SDK
```shell
cd sdk
mkdir build
cd build
export KEYSTONE_SDK_DIR=`pwd`
cmake ..
make
make install
```

4. Build Linux kernal and fw_payload.bin
```
mkdir build
cd build
cmake .. -DLINUX_VC707=y -DNO_BR2=y
make
make image
```

5. When compilation finishes successfuly, the `fw_payload.bin` is placed at `build/sm.build/platform/generic/firmware/fw_payload.bin`
- It is an OpenSBI firmware and containts the Linux kernel as payload

# TBD





















