# 2 - Network Booting the Pi with U-boot

To facilitate the development process, we are going to network boot our Pi over TFTP. This means that once we have the pi set up, we can remotely load any other files we need from our host machine. No more swapping the SD card between machines after every change! But what if we can to change the U-Boot script? To solve this issue, we will use two boot scripts, the first of which will just load and run the second one. This allows us to keep our second script on the TFTP server and modify it at will.

## 2.1: Creating initial boot script

Step 1: Find the host's IP address. The Pi's address will not be set so you can make one up, just make sure that the subnet is the same and the address is not already being used by something else.

Step 2: Create the source file.

```bash
$ cd /media/user/boot/
$ echo 'echo -- Running boot.scr --
setenv ipaddr <Pi's IP>
setenv serverip <Host's IP>
tftpb 0xC00000 boot2.scr
source 0xC00000' > boot.source
```

Step 3: Use the boot-tools to compile our boot code.

```bash
$ mkimage -T script -A arm64 -C none -a 0x2400000 -e 0x2400000 -d boot.source boot.scr
```

Step 4: Unmount your drive and place the SD card into the Raspberry Pi 4.

```bash
$ umount /media/user/boot
$ umount /media/user/root
```

## 2.2 Set up TFTP server on your host

Step 1: Install TFTP onto your machine.

```bash
$ sudo apt-get install tftpd-hpa
```

Step 2: Make sure that it is running.

```bash
$ sudo service tftpd-hpa status
```

Step 3: You should now have a directory under `/srv/tftp` that stores any of the files that we want to remotely load.

## 2.3 Creating secondary boot script

Step 1: Create the source file.

```bash
$ cd /srv/tftp/
$ echo 'echo -- Running boot.scr --' > boot2.source
```

Step 2: Use the boot-tools to compile our boot code.

```bash
$ mkimage -T script -A arm64 -C none -a 0xC00000 -e 0xC00000 -d boot2.source boot2.scr
```

## Sources

- [Xen on Raspberry Pi 4 adventures](https://www.linux.com/featured/xen-on-raspberry-pi-4-adventures/)
- [TFTP](https://help.ubuntu.com/community/TFTP)