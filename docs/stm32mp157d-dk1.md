## Intro

Practical guide and troubleshoot while learning embedded linux on STM32MP157.

### Zephyr RTOS

**Setup Zephyr workspace**

> This section describe how to setup Zephyr workspace for Cortex-M4 development on STM32MP157D-DK1.

- Follow the guides on [Zephyr page](https://docs.zephyrproject.org/latest/develop/getting_started/index.html)
    - In the install SDK section, after SDK installation, run the `setup.sh` script inside the SDK directory - refer [link](https://docs.zephyrproject.org/latest/develop/toolchains/zephyr_sdk.html#zephyr-sdk-installation).
- Create an application inside the Zephyr workspace (same directory as `zephyr` project) - refer [link](https://docs.zephyrproject.org/latest/develop/application/index.html#creating-an-application).
    - Clone the example-application - `git clone https://github.com/zephyrproject-rtos/example-application my-app`
 - Inside the application directory, update west manifest path to point to the application, `west config manifest.path my-app`
 - Inside my-app, checkout a desired release tag, `git checkout v4.4.0` then run `west update`
 - Create `zephyrrc` file to store Zephyr important enviroment variables - that will be used for firmware build - refer [zephyrrc link](https://docs.zephyrproject.org/latest/develop/env_vars.html#option-3-using-zephyrrc-files) and [env var link](https://docs.zephyrproject.org/latest/develop/env_vars.html#important-environment-variables)
    - Create `zephyrrc` in `$HOME/.config/zephyr/zephyrrc` then add desired environment variables.
    - `export BOARD=custom_plank` `export ZEPHYR_SDK_INSTALL_DIR=<path_to_sdk>`
- Build firmware in application directory `west build app`

**STM32MP157D-DK1 M4 firmware build, flash and run**

> This section describe step by step the process of building the workspace, build firmware, then flash and finally run firmware on Cortex-M4.

> [!NOTE]
> You don't need to follow the steps here if you just want to use the zephyr-stm32mp157 branch, it has everything needed to run a simple OpenAMP application on Cortex-M4.
> But some of the steps are nice to know (e.g u-boot loads M4 firmware images)

- For technical details, refer the commits of [zephyr-stm32mp157 repository](https://github.com/KizEvo/zephyr-stm32mp157d/tree/phunguyen/stm32mp157d-dev)
- We'll use Zephyr's example-application code as boilerplate for our Zephyr STM32MP157D workspace.
- Create `stm32mp157d_dk1` board directory, refer [Zephyr Board Porting Guide](https://docs.zephyrproject.org/latest/hardware/porting/board_porting.html#create-your-board-directory)
    - The [`stm32mp157c_dk2` board directory](https://github.com/zephyrproject-rtos/zephyr/tree/main/boards/st/stm32mp157c_dk2) is a good reference, all of the configs (Kconfigs and DTS) can be apply to STM32MP157D.
- Update the example-application `app/src/main.c`, remove unrelated code to test simple build - refer [commit](https://github.com/KizEvo/zephyr-stm32mp157d/commit/5cb5d3dc0c1e2601eaa2d03211e5189599095b8f).
    - `west build app` or `west build -b stm32mp157d_dk1 app` to build firmware image.
- Build and get the M4 firmware image under build directory `build/zephyr/zephyr.elf`.
- Use u-boot to load M4 firmware image ([u-boot STM32MP1 spec](https://docs.u-boot.org/en/v2021.04/board/st/stm32mp1.html#coprocessor-firmware)):
    - Two ways to store image:
        - Store `zephyr.elf` in the ext4 partition of the SDCard.
        - Store `zephyr.elf` in TFTP folder of host machine (e.g `/srv/tftp`) that the STM32MP157D can access via Ethernet.
    - During u-boot, load the `zephyr.elf` to RAM:
        - If image in SDCard, `load mmc 0:4 0xc1000000 zephyr.elf`
        - If image in TFTP folder, `tftp 0xc1000000 zephyr.elf`
    - Use `rproc` command to start Cortex-M4.
        - `rproc init`
        - `rproc list` - get the M4 device id.
        - `rproc load ${dev_id} ${loadaddr_copro} ${filesize_in_byte}`
        - `rproc start ${dev_id}`
    - At this stage, you'll see u-boot warn user about M4 .elf doesn't have .resource_table section defined. This is necessary for inter-process communication between A7 and M4 cores - [see document](https://wiki.st.com/stm32mpu/wiki/Cortex-M_remote_processor_management_overview#System_overview).
        - But if there is no requirement for IPC then you can ignore it.
- Use Linux to load M4 firmware image ([STM32Wiki Linux remoteproc spec](https://wiki.st.com/stm32mpu/wiki/Linux_remoteproc_framework_overview#Framework_purpose)).
    - Config Linux:
        - Activate the remoteproc driver and framework in the kernel configuration using the Linux Menuconfig tool.
        - Device drivers ---> Remoteproc drivers ---> [x] Support for Remote Processor subsystem [x] STM32 remoteproc support.
            - The module should be built statically into the kernel. If not, we'll need to load it during boot with `modprobe stm32_ipcc`.
        - Build the Linux kernel using Buildroot.
    - Two ways to store image, but first we'll need to rename the image to `rproc-m4-fw`:
        - Store the image in `/lib/firmware` of **rootfs**. The standard way in Buildroot is to use a Root Filesystem Overlay. It is copied directly onto the target filesystem after the build but before the image is created. Then we'll create a filesystem image (e.g SquashFS image) of the rootfs and store it in a partition of the SDCard.
        - Store the image in `/lib/firmware` of **nfsroot**. This method allow target system to load root filesystem via NFS (host machine store the root filesystem, target machine simply use it via NFS). This method allow fast development process as we don't need to keep moving SDCard between host and target machine.

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

**Connect to guest OS with SSH**

> This section describe how to setup the guest OS so that the host machine can use SSH to access it.

- Configure the guest OS Network. Network -> Adapter # -> Attached to: NAT -> Advanced -> Port Forwarding -> Update Host Port to 2222 and Guest Port to 22 (leave everything else as is).
- Start the guest OS and install OpenSSH -> `sudo apt install openssh-server` -> `sudo systemctl enable ssh` -> `sudo systemctl status ssh` (optional check).
- Open a terminal in host machine and connect to guest OS -> `ssh <username>@127.0.0.1 -p 2222`
- VSCode SSH -> Install Remote - SSH extension -> Connect to remote SSH -> Add new host `ssh <username>@127.0.0.1 -p 2222`

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
