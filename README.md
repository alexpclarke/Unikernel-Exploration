# Unikernel-Exploration
## Table of Contents

- [0 - Intro](0---Intro.md)
- [1 - Setting up U-Boot on the Raspberry Pi 4](1%20-%20Seting%20up%20U-Boot%20on%20the%20Raspberry%20Pi%204.md)
  - 1.1 - Downloading the cross compiler toolchain
  - 1.2 - Formatting your SD card
  - 1.3 - Getting firmware boot files
  - 1.4 - Building U-Boot
  - 1.5 - Connecting over UART
- [2 - Network Booting the Pi with U-boot](2%20-%20Network%20Booting%20the%20Pi%20with%20U-boot.md)
  - 2.1: Creating initial boot script
  - 2.2 Set up TFTP server on your host
  - 2.3 Creating secondary boot script
- [3 - Building Xen and Dom0](3%20-%20Building%20Xen%20and%20Dom0.md)
  - 3.1: Building the Xen binary
  - 3.2: Building the Linux Kernel
  - 3.3: Update boot2
  - 3.4: Build rootfs
  - 3.5: Imagebuilder
