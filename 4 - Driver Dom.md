# 4 - Driver Dom

Before we can implement our driver domain, we are going to need to understand what device drivers are and how Xen handles them. A device driver is a small chunk of code that provides an interface between your code and a specific hardware component, be that a hard drive, a mouse, a display or a network card. The drivers themselves do not usually support concurrernt use so, we need to find a way for multiple domains to access them at the same time. Xen handles this by using the "split driver model". 

The split driver model describes the use of a shared memory ring buffer and event transmissions to allow secure communication between our domains and our driver. This ring is split into two components, the driver domain backend and the DomU front end. The DomU will write a request to the ring, then notify the driver dom. The driver dom will then handle the request, write a response and notify the DomU. This allows for there to be a queue of requests, not all of which are from the same domain.

## 4.1 - Creating a VM

For this step, we are going to need a Virtual Machine that contains both the device driver we want as well as the Xenstore and Xenbus drivers. Any Linux distro with dom0 Xen support should work so we will just repurpose our Dom0 install. We are also going to need to make sure we have `xen-utils` installed since that is what provides the ability to receive the events from the guest Dom's.

Step 1: Copy Dom0.

```bash
$ mkdir ~/driver-dom1
$ cd ~/driver-dom1
$ cp /srv/tftp/Image-dom0 .
$ cp /srv/tftp/bcm2711-rpi-4-b.dtb .
$ cp ~/ramdisk.img .
```

Step 2: Chroot in and make sure everything we need is there.

```bash
$ sudo chroot /mnt/ramdisk qemu-aarch64-static /bin/bash
$ apt-get install xen-utils
$ exit
```

Step 3: Compress and copy over the ramdisk.

```bash
$ sudo find /mnt/ramdisk -print |sudo cpio -H newc -o |gzip -9 > /srv/tftp/driver-dom1/domU-ramdisk.cpio.gz
```

## 4.2 - Set up PCI passthrough

Step 1: Determine the BDF of our driver we want to pass through. You can read more about this [here](https://wiki.xenproject.org/wiki/Bus:Device.Function_(BDF)_Notation).

```bash
$ lspci
```

Step 2: Make sure Dom0 has xen-pciback module loaded.

```bash
$ modprobe xen-pciback
```

Step 3: Dynamically assign the driver using xl. (Replace with the appropriate BDF.)

```bash
$ xl pci-assignable-add 08:00.0
```

## 4.3 - Re-run imagebuilder

Step 1: modify `~/imagebuilder/config.txt`.

```
MEMORY_START="0x0"
MEMORY_END="0x20000000"

DEVICE_TREE="/srv/tftp/bcm2711-rpi-4-b.dtb"
XEN="/srv/tftp/xen"
DOM0_KERNEL="/srv/tftp/Image-dom0"
DOM0_RAMDISK="/srv/tftp/dom0-ramdisk.cpio.gz"

NUM_DOMUS=1
DOMU_KERNEL[0]="driver-dom1/Image-domU"
DOMU_RAMDISK[0]="driver-dom1/domU-ramdisk.cpio"
DOMU_PASSTHROUGH_DTB[0]="driver-dom1/passthrough-example-part.dtb"

UBOOT_SOURCE="/srv/tftp/boot2.source"
UBOOT_SCRIPT="/srv/tftp/boot2.scr"
```

Step 2: Run scripts

```bash
$ bash ~/imagebuilder/scripts/uboot-script-gen -c ./config -d /srv/tftp -t tftp -o boot2
```

## Sources

- https://wiki.xenproject.org/wiki/Driver_Domain#Setup
- https://wiki.xenproject.org/wiki/Xen_PCI_Passthrough
- https://wiki.xenproject.org/wiki/Bus:Device.Function_(BDF)_Notation
