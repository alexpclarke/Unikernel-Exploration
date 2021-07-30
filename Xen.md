## Building Xen With Yocto

Step 1: Install Yocto Poky and required layers

```bash
$ cd ~
$ git clone -b dunfell git://git.yoctoproject.org/poky
$ cd poky
$ git clone -b dunfell git://git.openembedded.org/meta-openembedded
$ git clone -b dunfell git://git.yoctoproject.org/meta-virtualization
$ git clone -b dunfell git://git.yoctoproject.org/meta-raspberrypi
```

Step 2: Initialize the Yocto build environment

```bash
$ source ./oe-init-build-env
```

Step 3: Edit conf/bblayers.conf

```
BBLAYERS ?= " \
	~/poky/meta \
	~/poky/meta-poky \
	~/poky/meta-yocto-bsp \
	~/poky/meta-openembedded/meta-oe \
	~/poky/meta-openembedded/meta-filesystems \
	~/poky/meta-openembedded/meta-python \
	~/poky/meta-openembedded/meta-networking \
	~/poky/meta-virtualization \
	~/poky/meta-raspberrypi \
 "
```

Step 4: Edit conf/local.conf

```
MACHINE = "raspberrypi4-64"
DISTRO = "poky"
SDKMACHINE = "aarch64"
DISTRO_FEATURES += " virtualization xen"
BUILD_REPRODUCIBLE_BINARIES = "1"
IMAGE_FSTYPES = "cpio.gz"
# PATCHRESOLVE = "noop"

INHERIT_pn-xen += "externalsrc"
INHERIT_pn-xen-tools += "externalsrc"
EXTERNALSRC_pn-xen = "/scratch/repos/xen"
EXTERNALSRC_pn-xen-tools = "/scratch/repos/xen"
EXTERNALSRC_BUILD_pn-xen = "/scratch/repos/xen"
EXTERNALSRC_BUILD_pn-xen-tools = "/scratch/repos/xen"
```

Step 5: run bitbake

```bash
$ bitbake xen-image-minimal
```

Sources:

- Yocto Project Reference Manual - https://www.yoctoproject.org/docs/2.4/ref-manual/ref-manual.html#var-IMAGE_FSTYPES
- Xen on ARM and Yocto - https://wiki.xenproject.org/wiki/Xen_on_ARM_and_Yocto

Step 5: it gives you something like this:

```
# Load files over TFTP
tftpb 0xE00000 xen
tftpb 0x1000000 Image
tftpb 0x2200000 xen-image-minimal-raspberrypi4-64.cpio.gz
tftpb 0x8400000 bcm2711-rpi-4-b.dtb

# Set FDT address
fdt addr 0x8400000

# Add some extra size to the FDT so we can add extra values.
fdt resize 1024

# Add properties to /chosen
fdt set /chosen \#address-cells <2>
fdt set /chosen \#size-cells <2>
fdt set /chosen xen,xen-bootargs "console=dtuart dtuart=serial0 dom0_mem=1G dom0_max_vcpus=1 bootscrub=0 vwfi=native sched=null"

# Create a node named dom0 in /chosen
fdt mknod /chosen dom0
fdt set /chosen/dom0 compatible "xen,linux-zimage" "xen,multiboot-module"
fdt set /chosen/dom0 reg <0x0 0x1000000 0x0 0x108AA00>
fdt set /chosen xen,dom0-bootargs "console=hvc0 earlycon=xen earlyprintk=xen clk_ignore_unused root=/dev/ram0"

# Create a node named dom0-ramdisk in /chosen
fdt mknod /chosen dom0-ramdisk
fdt set /chosen/dom0-ramdisk compatible "xen,linux-initrd" "xen,multiboot-module"
fdt set /chosen/dom0-ramdisk reg <0x0 0x2200000 0x0 0x6013E86>

# Do not copy fdt on boot
setenv fdt_high 0xffffffffffffffff

booti 0xE00000 - 0x8400000
```

Sources:

- https://gitlab.com/ViryaOS/imagebuilder
- https://wiki.xenproject.org/wiki/ImageBuilder





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

## 
