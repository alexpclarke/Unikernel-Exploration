# 3 - Xen and Dom0

To run Xen as our hypervisor, we will need to have four files: the Xen binary; the Dom0 kernel; the Dom0 ramdisk; and the device tree blob. Once we have all of these files, we will use the imagebuilder tool to create a U-Boot script.

I initially wanted to use Yocto Poky to simplify this process but as of July 2021, the meta-virtualization layers provided a significantly outdated version of Xen which did not include the Raspberry Pi 4 support that was added in version 4.14. If/when this gets updated, it will provide a much simpler means of building Xen and Dom0.

## 3.1: Building the Xen binary

Step 1: Make sure you have the cross-compiler installed from section 1.3.

```bash
$ printenv CROSS_COMPILE
```

Step 2: Clone Xen source.

```bash
$ cd ~
$ git clone --depth 1 --branch stable-4.15 git://xenbits.xen.org/xen.git
$ cd xen
```

Step 3: Make Xen.

```bash
$ ./configure --host=aarch64-none-linux-gnu --with-xenstored=xenstored --enable-systemd
$ make dist-xen XEN_TARGET_ARCH=arm64
```

Step 4: Copying over the binary.

```bash
$ cp ./xen/xen /srv/tftp
```

## 3.2: Building the Dom0 Kernel

Step 1: Make sure you have the cross-compiler installed from section 1.3.

```bash
$ printenv CROSS_COMPILE
```

Step 2: Clone Linux source

```bash
$ cd ~
$ git clone --depth=1 -b rpi-5.10.y https://github.com/raspberrypi/linux.git
$ cd linux
```

Step 3: Load the RPi4 and Xen configuration.

```bash
$ make O=build-arm64 ARCH=arm64 bcm2711_defconfig xen.config
```

Step 4: Build the kernel with optimizations for the Arm Cortex A72. You can read more about the GCC options [here](https://gcc.gnu.org/onlinedocs/gcc/AArch64-Options.html).

```bash
$ make O=build-arm64 ARCH=arm64 CXXFLAGS="-march=armv8-a -mtune=cortex-a72" CFLAGS="-march=armv8-a -mtune=cortex-a72" -j $(nproc)
```

Step 5: Copy the files to our TFTP directory.

```bash
$ cp ./build-arm64/arch/arm64/boot/Image /srv/tftp/Image-dom0
$ cp ./build-arm64/arch/arm64/boot/dts/broadcom/bcm2711-rpi-4-b.dtb /srv/tftp/
```

## 3.3: Build ramdisk

Step 1: Download necessary packages.

```bash
$ sudo apt-get install debootstrap qemu
```

Step 2: Create, format, and mount our image file. (`-o loop` will allow us to mount a file as though it were a block device)

```bash
$ cd ~
$ dd if=/dev/zero bs=1M count=2048 of=ramdisk.img
$ mkfs.ext4 ramdisk.img
$ sudo mount -o loop -t ext4 ramdisk.img /mnt/ramdisk
```

Step 3: Populate the root file system by running the first stage of debootstrap.

```bash
$ sudo debootstrap --foreign --arch arm64 focal /mnt/ramdisk http://ports.ubuntu.com/
```

Step 4: Run the second stage of debootstrap.

```bash
$ sudo cp /usr/bin/qemu-aarch64-static /mnt/ramdisk/usr/bin
$ sudo chroot /mnt/ramdisk qemu-aarch64-static /bin/bash
$ ./debootstrap/debootstrap --second-stage
$ exit
```

Step 5: Install the kernel modules into the filesystem.

```bash
$ cd ~/linux
$ sudo make O=build-arm64 ARCH=arm64 INSTALL_MOD_PATH=/aarch64-chroot modules_install
```

Step 6: Add the Xen tools.

```bash
$ echo 'deb http://ports.ubuntu.com focal universe' >> /mnt/ramdisk/etc/apt/sources.list
$ apt-get update
$ apt-get install xen-system-arm64
$ exit
```

Step 7: Gather and compress our ramdisk

```bash
$ sudo find /mnt/ramdisk -print |sudo cpio -H newc -o |gzip -9 > /srv/tftp/dom0-ramdisk.cpio.gz
```

## 3.4: Imagebuilder

Step 1: Clone the Xen ImageBuilder tool

```bash
$ cd ~
$ git clone --depth 1 https://gitlab.com/ViryaOS/imagebuilder.git
$ cd imagebuildere
```

Step 2: modify `config.txt`.

```
MEMORY_START="0x0"
MEMORY_END="0x20000000"

DEVICE_TREE="/srv/tftp/bcm2711-rpi-4-b.dtb"
XEN="/srv/tftp/xen"
DOM0_KERNEL="/srv/tftp/Image-dom0"
DOM0_RAMDISK="/srv/tftp/dom0-ramdisk.cpio.gz"

NUM_DOMUS=0

UBOOT_SOURCE="/srv/tftp/boot2.source"
UBOOT_SCRIPT="/srv/tftp/boot2.scr"
```

Step 4: Run scripts

```bash
$ bash ./scripts/uboot-script-gen -c ./config -d /srv/tftp -t tftp -o boot2
```

## Sources

- [Raspberry Pi 4 Arm64 Kernel Cross-Compile](https://gist.github.com/G-UK/ee7edc4844f14fec12450b2211fc886e) 
- [Mainline Linux Kernel Configs](https://wiki.xenproject.org/wiki/Mainline_Linux_Kernel_Configs)
- https://xenbits.xen.org/docs/unstable/misc/arm/device-tree/booting.txt
- https://xenbits.xenproject.org/docs/unstable/features/dom0less.html
- http://xenbits.xen.org/docs/unstable/misc/xen-command-line.html

