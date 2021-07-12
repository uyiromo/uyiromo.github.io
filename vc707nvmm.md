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
  - SD card
    - We used 16 GB one (if >= 8GB, it should work)
  - 4 GB DIMM
    - MT8KTF51264HZ-1G9P1
  - FMC Board
    - HTG-FMC-PCIE-RC
  - 12V Power for FMC
    - HTG-PWR-12V-6A
  - Ethernet Adapter over PCIe
    - Intel Gigabit CT PCI-E Network Adapter EXPI9301CTBLK
- Setup the vc707 board according to the following pdf
  - [SiFive Freedom U500 VC707 FPGA Getting Started Guide](https://sifive.cdn.prismic.io/sifive%2Fc248fabc-5e44-4412-b1c3-6bb6aac73a2c_sifive-u500-vc707-gettingstarted-v0.2.pdf)
- In addition, replace the 1 GiB DIMM with the 4 GiB DIMM


## Software
- Vivado (recommend: >= 2018.3)
  - We tested by using **Vivado 2018.3.**
  - If you use the newer version, some modification may be required.
- Operating System
  - We tested by using **Ubuntu 18.04.5 LTS**

# Build
## 1. Install required packages
```shell
sudo apt install bimfmt-support libncurses5-dev libncursesw5-dev libpopt-dev uuid-dev  debootstrap debian-ports-archive-keyring
```
- Other packages may be required. If so, also install them.

## 2. Build Freedom SoC
### 2.1. Clone vc707nvmm repository and setup it

```shell
git clone https://github.com/uyiromo/freedom.git freedom-devel
cd freedom-devel
git checkout nvmm_emulation
git submodule update --init --recursive
```

### 2.2. Build
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

### 2.3. Get the bitstream
- When compilation finishes successfuly, the bitstream is placed at `builds/vc707-u500devkit/obj/VC707PCIeShell.bit`


## 3. Build Linux kernel with OpenSBI
- Using the keystone repository.

### 3.1. Clone vc707nvmm repository and setup it
```shell
git clone https://github.com/uyiromo/keystone.git keystone-devel
cd keystone-devel
git checkout vc707nvmm
git submodule update --init --recursive
```

### 3.2. Build RISC-V toolchains
```shell
cd riscv-gnu-toolchain
./configure --prefix=$(pwd)/../riscv
make               # newlib gcc (riscv64-unknown-elf-*)
make linux         #    gnu gcc (riscv64-unknown-linux-gnu-*)
export RISCV=`pwd`/../riscv
export PATH=$RISCV/bin:$PATH
```

### 3.3. Build Keystone SDK
```shell
cd sdk
mkdir build
cd build
export KEYSTONE_SDK_DIR=`pwd`
cmake ..
make
make install
```

### 3.4. Build Linux kernal and fw_payload.bin
```
mkdir build
cd build
cmake .. -DLINUX_VC707=y -DNO_BR2=y
make
make image
```

### 3.5. Get the firmware
- When compilation finishes successfuly, the `fw_payload.bin` is placed at `build/sm.build/platform/generic/firmware/fw_payload.bin`
- It is an OpenSBI firmware and containts the Linux kernel as payload



## 4. Build Debian File System
### 4.1 Build RISC-V QEMU
- It is required to setup the rootfs

#### 4.1.1. Clone and build
```shell
git clone https://github.com/qemu/qemu qemu
cd qemu
git checkout v5.0.0
mkdir build
cd build
../configure --target-list=riscv64-linux-user --static --disable-docs --disable-system
make -j`nproc`
```

#### 4.1.2 Place it in /usr/bin
```shell
sudo ln -s `pwd`/riscv64-linux-user/qemu-riscv64 /usr/bin/qemu-riscv64-static
```

#### 4.1.3 Configure binfmt
```shell
cat > /tmp/qemu-riscv64 <<EOF
package qemu-user-static 
type magic
offset 0
magic \x7f\x45\x4c\x46\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xf3\x00
mask \xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff
interpreter /usr/bin/qemu-riscv64-static
EOF

sudo update-binfmts --import /tmp/qemu-riscv64
```
- After this procedure, the OS call qemu automatically when executing riscv64 binaries


### 4.2 Build debootstrap
#### 4.2.1 Do debootstrap 1st stage
```shell
mkdir <rootfs-dir>
sudo debootstrap --foreign --arch riscv64 --keyring /etc/apt/trusted.gpg.d/debian-ports-archive-2019.gpg \
  --include=debian-ports-archive-keyring  unstable <rootfs-dir> https://snapshot.debian.org/archive/debian-ports/20191201T030503Z/
sudo cp `which qemu-riscv64-static` <rootfs-dir>/usr/bin
```

#### 4.2.2 Do debootstrap 2nd stage
```shel
sudo chroot <rootfs-dir> bash
> /usr/bin/groups: cannot find name for group ID 0
> I have no name!@****:/#
I have no name!@****:/# ./debootstrap/debootstrap --second-stage
```

#### 4.2.3 Enable users
```shell
I have no name!@****:/# passwd
I have no name!@****:/# su
I have no name!@****:/# apt-get update
I have no name!@****:/# adduser <uid>
I have no name!@****:/# gpasswd -a <uid> sudo
```

#### 4.2.4 Setup Package Repositories
```shell
I have no name!@****:/# cat /etc/apt/sources.list
#deb http://deb.debian.org/debian unstable main
deb http://ftp.ports.debian.org/debian-ports/ sid main
deb http://ftp.ports.debian.org/debian-ports/ unreleased main
deb-src http://ftp.ports.debian.org/debian-ports/ sid main
```

#### 4.2.5 [Optional] Change hostname
```shell
I have no name!@****:/# echo "vc707" > /etc/hostname
```


#### 4.2.6 [Optional] Install Packages
```shell
I have no name!@****:/# apt-get update
I have no name!@****:/# apt-get upgrade -y
I have no name!@****:/# apt-get install openssh-server emacs sudo build-essential gdb
```


#### 4.2.7 [Optional] Configure PATH
```shell
I have no name!@****:/# cat /etc/login.defs
#ENV_PATH        PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
ENV_PATH        PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games:/usr/sbin:/sbin
```


## 5. Create bootable SD card
### 5.1 Build gpftdisk
- To create a bootable SD card for VC707, dedicated gpftdisk to set special GUID
- Clone and Build
```shell
git clone https://github.com/tmagik/gptfdisk.git
cd gptfdisk/
git checkout 3d6a1587
make
```
  - `gpftdisk/sgdisk` is the gpftdisk

### 5.2 Create partition
#### Write the sgdisk_sd.sh
```shell:sgdisk_sd.sh
#!/bin/sh
<path/to/sgdisk> -g \
      --clear \
      --new=1:2048:67583  --change-name=1:"SiFive bare-metal" --typecode=1:2e54b353-1271-4842-806f-e436d6af6985 \
      --new=2:264192:     --change-name=2:"Linux filesystem"  --typecode=2:8300 \
      --new=4:67584:67839 --change-name=4:"SiFive FSBL"       --typecode=4:5b193300-fc78-40cd-8002-e86c45580b47 \
      </dev/sdX>
```
  - Replace `<path/to/sgdisk>` with the path to sgdisk built in `5.1 Build gpftdisk`
  - Replace `</dev/sdX>` with the path to the SD card

#### Run the sgdisk.sh
- Run
```shell
sudo ./sgdisk_sd.sh
```

- When the script completed successfully, `sudo gdisk -l` will show like this:

```shell
Disk /dev/sdX: 14.87 GiB, 15962472448 bytes, 31176704 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 04A9D0C7-013E-4076-9C1F-0A1DD1EE4267

Device     Start      End  Sectors  Size Type
/dev/sdX1   2048    67583    65536   32M HiFive Unleashed BBL
/dev/sdX2  67840 31176670 31108830 14.8G Linux filesystem
/dev/sdX3  67584    67839      256  128K HiFive Unleashed FSBL
```

### 5.3 Put the artifacts
#### fw_payload.bin
```shell
sudo dd if=<keystone-devel/build/sm.build/platform/generic/firmware/fw_payload.bin> of=/dev/sdX1 bs=4096 conv=fsync
```

#### FSBL.bin
```shell
sudo dd if=<freedom-devel/bootrom/freedom-u540-c000-bootloader/FPGAfsbl.bin> of=/dev/sdX4 bs=4096 conv=fsync
```

### rootfs
```shell
sudo mount /dev/sdX2 /mnt
sudo cp -ar <rootfs-dir>/* /mnt
```

## 6. Boot the vc707 board
### program_fpga.tcl
```tcl
if { $argc != 1 } {
    puts ".bit must be specified by tclargs"
    exit 1
}
set bit [lindex $argv 0]
puts stderr ".bit is "
puts stderr $bit

open_hw
connect_hw_server
open_hw_target

current_hw_device [get_hw_devices xc7vx485t_0]
refresh_hw_device -update_hw_probes false [lindex [get_hw_devices xc7vx485t_0] 0]

set_property PROBES.FILE {} [get_hw_devices xc7vx485t_0]
set_property FULL_PROBES.FILE {} [get_hw_devices xc7vx485t_0]
set_property PROGRAM.FILE $bit [get_hw_devices xc7vx485t_0]
program_hw_devices [get_hw_devices xc7vx485t_0]
refresh_hw_device [lindex [get_hw_devices xc7vx485t_0] 0]

close_hw_target
disconnect_hw_server
close_hw
```

### Power on the board
- Insert configured SD card
- Set the boot switches on the boatd to "JTAG boot"
- Power on

### Load the bitstream into the board
```shell
vivado -mode batch -source <path/to/program_fpga.tcl> -tclargs <path/to/VC707PCIeShell.bit>
```

## 7. Connect to the board
- UART
  - the OS and Kernel show boot logs and error messages on UART
```shell
sudo minicom -b 115200 -D /dev/serial/by-id/usb-Silicon_Labs_CP2103_USB_to_UART_Bridge_Controller_0001-if00-port0
```

- ssh
  - the configured vc707-u500devkit has an ethernet over PCIe
  - You can connect it by using ssh


