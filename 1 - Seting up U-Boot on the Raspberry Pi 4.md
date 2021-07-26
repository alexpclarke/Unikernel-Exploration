# 1 - Setting up U-Boot on the Raspberry Pi 4

The first step to getting our Pi set up is installing U-Boot. Since the Pi relies on a closed-source boot loader, we will be using that to load U-Boot and then use U-Boot to do everything else.

## 1.1 - Downloading the cross compiler toolchain

This documentation assumes that you are building on x86 Linux host, meaning that we will need to set up a cross-compiler for our binaries to work on our Pi. Go to the [arm website]( https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads) and decide which toolchain is appropriate for you. We will be using version 10.2-2020.11, just make sure that you select the appropriate host with the AArch64 GNU/Linux target.

Step 1: Install the appropriate toolchain.

```bash
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

## 1.2 - Formatting your SD card

Next, we need to format our SD card so that we can store out boot files onto it.

Step 1: Plug in your SD card and use the [Raspberry Pi Imager](https://www.raspberrypi.org/software/) to update the firmware for SD boot.

Step 2: Find the path to your SD card. and unmount any partitions in it. It should be labled something like `/dev/mmcblk0` or `/dev/sdb`.

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
Disk identifier: 0x505de259

Device         Boot  Start      End  Sectors  Size Id Type
/dev/mmcblk0p1        2048   194559   192512   94M  c W95 FAT32 (LBA)
/dev/mmcblk0p2      194560 62410751 62216192 29.7G 83 Linux

...
```

```bash
$ sudo umount /dev/mmcblk0p1
$ sudo umount /dev/mmcblk0p2
```

Step 3: Partition the drive.

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

## 1.3 - Getting firmware boot files

There are certain files that the firmware requires to load our binaries. `start4.elf` is where the firmware is contained, `fixup4.dat` is a linker file that is paired with the firmware and `bcm2711-rpi-4-b.dts` is the device tree blob that describes the hardware of the Pi. We will also create our own `config.txt` file which will contain options for the firmware bootloader.

Step 1: Clone firmware.

```bash
$ git clone https://github.com/raspberrypi/firmware
$ cd firmware
```

Step 2: Copy over the required files.

```bash
$ cp ./boot/bcm2711-rpi-4-b.dtb /media/user/boot/
$ cp ./boot/fixup4.dat /media/user/boot/
$ cp ./boot/start4.dat /media/user/boot/
```

Step 3: Create config file.

```bash
$ cd /media/user/boot/
$ echo 'kernel=u-boot.bin
kernel_address=0x80000
device_tree=bcm2711-rpi-4-b.dtb
device_tree_address=0x2600000
start_file=start4.elf
fixup_file=fixup4.dat
arm_64bit=1
enable_uart=1
init_uart_baud=115200' > config.txt
```

## 1.4 - Building U-Boot

Our next step is to get our U-Boot binary. You could download a pre-built one, but it is easy to build it yourself so we will go through the steps of doing so.

Step 1: Make sure you have the cross-compiler installed from section 1.1.

```bash
$ printenv CROSS_COMPILE
```

Step 2: Clone U-Boot source.

```bash
$ git clone https://github.com/u-boot/u-boot.git
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

## 1.5 - Connecting over UART

Our SD card now has everything that it needs on it but we need to figure out how to interact with it. To communicate with our board, we will be using a TTY to USB cable that can be easily found from online vendors.

Step 1: Unmount your drive and place the SD card into the Raspberry Pi 4.

```bash
$ umount /media/user/boot
$ umount /media/user/root
```

Step 2: Hook up your cable.

![TTY Connector_bb](/Users/alexanderclarke/Documents/QVLx/TTY Connector_bb.png)

Step 3: Download a serial terminal program. For this example, we will use `screen`.

```bash
$ sudp apt-get install screen
```

Step 4: Run screen and turn on your Pi.

```bash
$ sudo screen /dev/ttyUSB0 115200
```

## Sources

- [Xen on Raspberry Pi 4 adventures](https://www.linux.com/featured/xen-on-raspberry-pi-4-adventures/)
- [Hacking Raspberry Pi 4 with Yocto: Formatting the SD Card](https://lancesimms.com/RaspberryPi/HackingRaspberryPi4WithYocto_Part2.htmll)
- [Hacking Raspberry Pi 4 with Yocto: Using the UART](https://lancesimms.com/RaspberryPi/HackingRaspberryPi4WithYocto_Part1.html)
- [U-Boot on Raspberry Pi](https://andrei.gherzan.ro/linux/uboot-on-rpi/)
- [Rpi U-Boot](https://elinux.org/RPi_U-Boot)
- [The boot folder](https://www.raspberrypi.org/documentation/configuration/boot_folder.md)

