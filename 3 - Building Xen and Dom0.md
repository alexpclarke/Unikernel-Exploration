# 3 - Building Xen and Dom0

## 3.1: Building the Xen binary

Step 1: Download the necessary packages.

```bash
$ sudo apt-get install gettext acpica-tools libpixman-1-dev libyajl-dev libfdt-dev
```

Step 2: Make sure you have the cross-compiler installed from section 1.1.

```bash
$ printenv CROSS_COMPILE
```

Step 3: Clone Xen source.

```bash
$ git clone git://xenbits.xen.org/xen.git
$ cd xen
$ git checkout stable-4.15
```

Step 4: Make Xen.

```bash
$ ./configure --host=aarch64-linux-gnu
$ make dist-xen XEN_TARGET_ARCH=arm64
```

Step 5: Copying over the binary.

```bash
$ cp ./xen/xen /srv/tftp
```

## 3.2: Building the Linux Kernel

Step 1: Make sure you have the cross-compiler installed from section 1.1.

```bash
$ printenv CROSS_COMPILE
```

Step 2: Clone Linux source

```bash
$ git clone --depth=1 -b rpi-5.10.y https://github.com/raspberrypi/linux.git
$ cd linux
```

Step 3: Add some extra options to the default RPi4 defconfig.

```bash
$ echo '
CONFIG_XEN=y
CONFIG_PARAVIRT=y
CONFIG_HVC_DRIVER=y
CONFIG_HVC_XEN=y
CONFIG_XEN_FBDEV_FRONTEND=y
CONFIG_XEN_BLKDEV_FRONTEND=y
CONFIG_XEN_NETDEV_FRONTEND=y
CONFIG_XEN_PCIDEV_FRONTEND=y
CONFIG_INPUT_XEN_KBDDEV_FRONTEND=y
CONFIG_XEN_XENBUS_FRONTEND=y
CONFIG_XEN_SAVE_RESTORE=y
CONFIG_XEN_GRANT_DEV_ALLOC=m
CONFIG_ACPI=y
CONFIG_XEN_DOM0=y
CONFIG_XEN_DEV_EVTCHN=y
CONFIG_XENFS=y
CONFIG_XEN_COMPAT_XENFS=y
CONFIG_XEN_SYS_HYPERVISOR=y
CONFIG_XEN_GNTDEV=y
CONFIG_XEN_BACKEND=y
CONFIG_XEN_NETDEV_BACKEND=m
CONFIG_XEN_BLKDEV_BACKEND=m
CONFIG_XEN_BALLOON=y
CONFIG_XEN_SCRUB_PAGES=y
' >> ./arch/arm64/configs/bcm2711_defconfig
```

Step 4: Load the RPi4 configuration.

```bash
$ make ARCH=arm64 bcm2711_defconfig
```

Step 5: Build the kernel with optimizations for the Arm Cortex A72.

```bash
$ make ARCH=arm64 CXXFLAGS="-march=armv8-a+crc -mtune=cortex-a72" CFLAGS="-march=armv8-a+crc -mtune=cortex-a72" bindeb-pkg
```

Step 6: Copy the files to our TFTP directory.

```bash
$ cp arch/arm64/boot/Image /srv/tftp/Image-dom0
$ mkdir /srv/tftp/overlays && cp arch/arm64/boot/dts/overlays/*.dtbo /srv/tftp/overlays/
$ cp arch/arm64/boot/dts/broadcom/*.dtb /srv/tftp/
```

## 3.3: Update boot2

Step 1: Get the sizes of the Dom0 kernel and the rootfs.

```bash
$ cd /srv/tftp
$ hexdump /Image-dom0 | tail -n 1
```

Step 2: Replace boot2.source.

```bash
$ echo '# Load files over TFTP
tftpb 0xE00000 xen
tftpb 0x1000000 Image-dom0
#tftpb 0x2200000 xen-image-minimal-raspberrypi4-64.cpio.gz
tftpb 0x8400000 bcm2711-rpi-4-b.dtb

# Set FDT address
fdt addr 0x8400000

# Add some extra size to the FDT so we can add extra values.
fdt resize 1024

# Add properties to /chosen.
fdt set /chosen \#address-cells <1>
fdt set /chosen \#size-cells <1>
fdt set /chosen xen,xen-bootargs "console=dtuart dtuart=serial0 dom0_mem=1G dom0_max_vcpus=1 bootscrub=0 vwfi=native sched=null"

# Create a node named dom0 in /chosen
fdt mknod /chosen dom0
fdt set /chosen/dom0 compatible "xen,linux-zimage" "xen,multiboot-module"
fdt set /chosen/dom0 reg <0x1000000 0x108AA00>
fdt set /chosen xen,dom0-bootargs "console=hvc0 earlycon=xen earlyprintk=xen clk_ignore_unused root=/dev/ram0"

# Create a node named dom0-ramdisk in /chosen
fdt mknod /chosen dom0-ramdisk
fdt set /chosen/dom0-ramdisk compatible "xen,linux-initrd" "xen,multiboot-module"
fdt set /chosen/dom0-ramdisk reg <0x2200000 0x6013E86>

# Do not copy fdt on boot
setenv fdt_high 0xffffffffffffffff

booti 0xE00000 - 0x8400000' > boot2.source
```

notes:

- xen-bootargs/bootscrub - Scrub free RAM during boot. This is a safety feature to prevent accidentally leaking sensitive VM data into other VMs if Xen crashes and reboots.
- xen-bootargs/vwfi - When setting vwfi to native, Xen doesn’t trap either instruction (guest WFI and WFE), running them in guest context. Setting vwfi to native reduces irq latency significantly. It can also lead to suboptimal scheduling decisions, but only when the system is oversubscribed (i.e., in total there are more vCPUs than pCPUs).
- xen-bootargs/sched - Choose the default scheduler. Note the default scheduler is selectable via Kconfig and depends on enabled schedulers. Check CONFIG_SCHED_DEFAULT to see which scheduler is the default.

Step 3: Use the boot-tools to compile our boot code:

```bash
$ mkimage -T script -A arm64 -C none -a 0xC00000 -e 0xC00000 -d boot2.source boot2.scr
```

## 3.4: Build rootfs

## 3.5: Imagebuilder

## Sources

- [Raspberry Pi 4 Arm64 Kernel Cross-Compile](https://gist.github.com/G-UK/ee7edc4844f14fec12450b2211fc886e) 
- [Mainline Linux Kernel Configs](https://wiki.xenproject.org/wiki/Mainline_Linux_Kernel_Configs)
- https://xenbits.xen.org/docs/unstable/misc/arm/device-tree/booting.txt
- https://xenbits.xenproject.org/docs/unstable/features/dom0less.html
- http://xenbits.xen.org/docs/unstable/misc/xen-command-line.html#dom0_max_vcpus