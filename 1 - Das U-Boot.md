# 1 - Das U-Boot

The first step to getting our Pi set up is setting up our our bootloader. New to the Pi 4b, this board comes with a 512KB SPI-attached EEPROM which is used as its bootloader. While this bootloader works well for simple applications, it is closed source and lacks certain functionality that we will need to run our system. We will use this built-in bootloader as our primary boot loader and load U-Boot from our SD card, using that as our secondary bootloader.

U-Boot is a powerful open-source boot loader designed for embeded solutions with support for various architectures, including AARCH64.

## 1.1 - Connecting over UART

Before we get started putting anything onto our Pi, we will need to be able to communicate with it. We will be doing this using a USB to TTY serial cable and the application screen. 

Step 1: Hook up your cable.

![TTY Connector_bb](file:///Users/alexanderclarke/dev/Unikernel-Exploration/images/TTY%20Connector_bb.png?lastModify=1627662875)

Step 2: Download a serial terminal program. For this example, we will use `screen`. For more information on how to use screen, click [here](https://linuxize.com/post/how-to-use-linux-screen/).

```bash
$ sudp apt-get install screen
```

## 1.2 - Updating EEPROM bootloader

First, we want to make sure that our EEPROM bootloader is up to date.

Step 1: Go to [this website](https://www.raspberrypi.org/software/) and download the appropriate version of the Raspberry Pi Imager.

Step 2: Plug your SD card into your host machine.

Step 3: Open the Raspberry Pi Imager.

Step 4: For the operating system, select: Misc utility images > Bootloader > SD Card Boot. 

Step 5: For the storage, select your SD card.

Step 6: Click Write.

Step 7: Unmount your SD card, place it into the Pi, and power it on for about 5 seconds.

## 1.3 - Downloading the cross-compiler toolchain

This documentation assumes that you are building on x86 Linux host, meaning that we will need to set up a cross-compiler for our binaries to work on our Pi. Go to the [arm website]( https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads) and decide which toolchain is appropriate for you. We will be using version 10.2-2020.11, just make sure that you select the appropriate host with the AArch64 GNU/Linux target.

Step 1: Install the appropriate toolchain.

```bash
$ cd ~
$ wget https://developer.arm.com/-/media/Files/downloads/gnu-a/10.2-2020.11/binrel/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu.tar.xz
```

Step 2: Unzip it.

```bash
$ tar -xf gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu.tar.xz
```

Step 3: Export it to the local environment so that it can be used later. (Make sure to run this step every time you reboot your system.)

```bash
$ export CROSS_COMPILE="$(pwd)/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-"
```

## 1.4 - Formatting your SD card

Next, we need to format our SD card so that we can store our boot files onto it.

Step 1: Find the path to your SD card and unmount any partitions in it. It should be labled something like `/dev/mmcblk0` or `/dev/sdb`. Replace the SD's path with your when appropriate.

```bash
$ sudo fdisk -l
```

```
...

Disk /dev/mmcblk0: 29.78 GiB, 31954305024 bytes, 62410752 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device         Boot Start      End  Sectors  Size Id Type
/dev/mmcblk0p1       2048 62410751 62408704 29.8G  b W95 FAT32

...
```

```bash
$ sudo umount /dev/mmcblk0p1
$ sudo umount /dev/mmcblk0p2
```

Step 2: Partition the drive. We are only concerned with the FAT32 partition but we can use any additional space for other files.

```bash
$ sudo parted -s /dev/mmcblk0 \
mklabel msdos \
mkpart primary fat32 1M 100M \
mkpart primary ext4 100M 100%
```

Step 4: Format the partitions.

```bash
$ sudo mkfs.vfat /dev/mmcblk0p1
$ sudo mkfs.ext4 /dev/mmcblk0p2
```

Step 4: Add labels to the partitions.

```bash
$ sudo fatlabel /dev/mmcblk0p1 boot
$ sudo e2label /dev/mmcblk0p2 root
```

Step 5: Mount the drives back.

```bash
$ sudo mkdir -p /media/user/boot && sudo mount /dev/mmcblk0p1 /media/user/boot
$ sudo mkdir -p /media/user/root && sudo mount /dev/mmcblk0p2 /media/user/root
```

## 1.4 - Getting firmware boot files

There are certain files that the primary boot loader requires to load our binaries. `start4.elf` is where the firmware is contained, `fixup4.dat` is a linker file that is paired with the firmware and `bcm2711-rpi-4-b.dts` is the device tree blob that describes the hardware of the Pi. We will also create our own `config.txt` file which will contain options for the firmware bootloader.

Step 1: Clone firmware.

```bash
$ cd ~
$ git clone --depth 1 https://github.com/raspberrypi/firmware
$ cd firmware
```

Step 2: Copy over the required files.

```bash
$ cp ./boot/bcm2711-rpi-4-b.dtb /media/user/boot/
$ cp ./boot/fixup4.dat /media/user/boot/
$ cp ./boot/start4.elf /media/user/boot/
```

Step 3: Create the file `/media/user/boot/config.txt` containing:

```
kernel=u-boot.bin
kernel_address=0x80000
device_tree=bcm2711-rpi-4-b.dtb
#device_tree_address=0x2600000
#total_mem=8192
total_mem=2048
start_file=start4.elf
fixup_file=fixup4.dat
arm_64bit=1
enable_uart=1
init_uart_baud=115200
```

## 1.5 - Building U-Boot

Our next step is to get our U-Boot binary. You could download a pre-built one, but it is easy to build it yourself so we will go through the steps of doing so.

Step 1: Make sure you have the cross-compiler installed from section 1.1.

```bash
$ printenv CROSS_COMPILE
```

Step 2: Clone U-Boot source.

```bash
$ cd ~
$ git clone --depth 1 https://github.com/u-boot/u-boot.git
$ cd u-boot
```

Step 3: Set config to Raspberry Pi 4.

```bash
$ make rpi_4_defconfig
```

Step 4: Build u-boot binary.

```bash
$ make
```

Step 5: Copy over u-boot binary.

```bash
$ cp ./u-boot.bin /media/user/boot/
```

Step 6: Unmount your SD card and place in your Pi.

```bash
$ sudo umount /dev/mmcblk0p1
$ sudo umount /dev/mmcblk0p2
```

Step 7: Run screen and turn on your Pi.

```bash
$ sudo screen /dev/ttyUSB0 115200
```

```

```

## Sources

- [Raspberry Pi 4 boot EEPROM](https://www.raspberrypi.org/documentation/hardware/raspberrypi/booteeprom.md)
- [The boot folder](https://www.raspberrypi.org/documentation/configuration/boot_folder.md)
- [Xen on Raspberry Pi 4 adventures](https://www.linux.com/featured/xen-on-raspberry-pi-4-adventures/)
- [Hacking Raspberry Pi 4 with Yocto: Formatting the SD Card](https://lancesimms.com/RaspberryPi/HackingRaspberryPi4WithYocto_Part2.htmll)
- [Hacking Raspberry Pi 4 with Yocto: Using the UART](https://lancesimms.com/RaspberryPi/HackingRaspberryPi4WithYocto_Part1.html)
- [U-Boot on Raspberry Pi](https://andrei.gherzan.ro/linux/uboot-on-rpi/)
- [Rpi U-Boot](https://elinux.org/RPi_U-Boot)

