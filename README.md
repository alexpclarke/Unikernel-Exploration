# Unikernel-Exploration
## Table of Contents

- 0 - Intro
  - 0.1 - Isolation
  - 0.2 - Attack Surface
- 1 - Das U-Boot
  - 1.1 - Downloading the cross compiler toolchain
  - 1.2 - Formatting your SD card
  - 1.3 - Getting firmware boot files
  - 1.4 - Building U-Boot
  - 1.5 - Connecting over UART
- 2 - Network Booting the Pi with U-boot
  - 2.1: Creating initial boot script
  - 2.2 Set up TFTP server on your host
  - 2.3 Creating secondary boot script
- 3 - Xen and Dom0
  - 3.1: Building the Xen binary
  - 3.2: Building the Linux Kernel
  - 3.3: Update boot2
  - 3.4: Build rootfs
  - 3.5: Imagebuilder
