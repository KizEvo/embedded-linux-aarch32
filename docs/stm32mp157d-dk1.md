## Intro

Practical guide and troubleshoot while learning embedded linux on STM32MP157.

### Linux Kernel

#### Guides

**Kernel/Modules compilation and installation**

> This section describe how Kernel/Modules compilation and installation steps.

- Specify the architecture for the kernel to build (can be define as environment var or pass with `make`).
    - Set `ARCH` to the name of a directory under `linux/arch/`: ARCH=arm or ARCH=arm64 or ARCH=riscv, etc
- Choose a compiler `CROSS_COMPILE` (can be define as environment var or pass with `make`).
    - The compiler invoked by the kernel Makefile is `$(CROSS_COMPILE)gcc`: CROSS_COMPILE=arm-linux-
- Pick initial config.
    - Default configurations stored in-tree as minimal configuration files `arch/<arch>/configs/`
    - `make foo_defconfig`
- Customize config - `make menuconfig`.
    - `make oldconfig`. Useful to upgrade a `.config` file from an earlier kernel release. If you edit a `.config` file by hand, it’s useful to run make oldconfig afterwards, to set values to new parameters that could have appeared because of dependency changes.
- Kernel compilation.
    - `make`. Use more CPU core `make -j n`
    - Only works from the top kernel source directory
- Kernel compilation result.
    - `arch/<arch>/boot/Image`, uncompressed kernel image that can be booted
    - `arch/<arch>/boot/*Image*`, compressed kernel images that can also be booted: `bzImage` for x86, `zImage` for ARM (32-bit), `Image.gz` for RISC-V, `vmlinux.bin.gz` for ARC, etc.
    - `arch/<arch>/boot/dts/<vendor>/*.dtb`, compiled Device Tree Blobs.
    - All kernel modules, spread over the kernel source tree, as .ko (Kernel Object) files: use `grep "*.ko" ./* -r`.
    - `vmlinux`, a raw uncompressed kernel image in the ELF format, useful for debugging purposes but generally not used for booting purposes
- Modules installation.
    - Compile modules `make modules`.
    - `make INSTALL_MOD_PATH=<dir>/ modules_install` (`INSTALL_MOD_PATH` is useful for cross-compiling target as without this they would install under host `/lib/modules`).

**Note: Compile Device Tree Source DTS `make dtbs`**.

### VirtualBox

#### Guides

**USB connections (SDCard, Serial communication)**

> This section describe how to setup the guest OS to access the USB devices on host OS.

- Refer to [USB Support](https://docs.oracle.com/en/virtualization/virtualbox/6.0/user/usb-support.html).
- To achieve this, Oracle VM VirtualBox presents the guest OS with a virtual USB controller. As soon as the guest system starts using a USB device, it will appear as unavailable on the host.
- **USB Device Filters**: When USB support is enabled for a VM, you can determine in detail which devices will be automatically attached to the guest. For this, you can create filters by specifying certain properties of the USB device.
- In VirtualBox, turn off the guest OS completely, then right click on the desired guest OS and click `Settings -> USB -> Add new USB filters`

<div align="center">
  <img width="623" height="470" alt="image" src="https://github.com/user-attachments/assets/b455f2ef-0336-4d34-8a3e-cdf2ead9c13a" />
</div>

- Power on the guest OS, check for connected USB devices with `lsusb` command or `dmesg` and go through the logs. In our case, the devices are `/dev/sdb` (USB SD Card Reader) and `/dev/ttyACM0` (ST-Link serial communication).

**Network configuration for TFTP protocol**

> This section describe how to setup the guest OS to communicate with STM32MP157 using Ethernet with TFTP protocol.

Oracle VM VirtualBox support enabling networking adapters and they can be configured in multiple modes, the mode we're going to use is [Bridged networking](https://www.virtualbox.org/manual/ch06.html#network_bridged).
- When enabled, VirtualBox connects to one of your installed network cards and exchanges network packets directly, circumventing your host operating system's network stack.
- In VirtualBox, turn off the guest OS completely, then right click on the desired guest OS and click `Settings -> Network`.
    - In our case, the host OS workstation support WiFi and Ethernet, so keep the default Adapter (in `NAT` mode) and enable a new Adapter running `Bridged Adapter`, then select the Controller of the Ethernet port and finally expand the `Advanced` tab and select `Allow All` for Promiscuous mode.

<div align="center">
  <img width="622" height="268" alt="image" src="https://github.com/user-attachments/assets/08e10302-3f54-4107-8148-53f07ccf8353" />
</div>

- Turn on the guest OS and run `ip a` to check the new network interface.
- Config the guest OS ip address via NetworkManager CLI, `nmcli con add type ethernet ifname <network-interface> ip4 <ip-addr>/24`
    - Example `network-interface=enp0s3` and `ip-addr=192.168.0.1` (make sure STM32MP157 IP address belong to this network segment).
- Install TFTP server on development workstation (guest OS).
    - `sudo apt install tftpd-hpa`
- TFTP client (STM32MP157) can access to files placed under `/srv/tftp`. If not, try to update directory permission `chmod 777 /srv/tftp`.
    - Add a test file `textfile.txt` to this folder and run `tftp 0xc2000000 textfile.txt` on TFTP client to download the file.
    - If firewall is enabled on workstation, make sure it does not filter TFTP client request `sudo ufw allow from 192.168.0.100`
