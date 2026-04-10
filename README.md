# embedded-linux-aarch32
Embedded Linux learning journey using STM32MP157D platform with dual Arm Cortex-A7 core and Cortex-M4.

> Embedded Linux is the usage of the Linux kernel and various open-source components in embedded systems.

## Prerequisite

### Hardwares
- Any STM32MP157 series from [STmicroelectronic](https://www.st.com/en/microcontrollers-microprocessors/stm32mp157.html).
    - In this project, we'll be using [discovery kit with STM32MP157D MPU](https://www.st.com/en/evaluation-tools/stm32mp157d-dk1.html).
- Ethernet cable, 5 V / 3 A USB-C power supply, USB-A to micro USB-B cable (for serial communication between host and target machine).

### Softwares and Tools
- Host machine running Linux OS, or virtualization software to emulate Linux OS such as [VirtualBox](https://www.virtualbox.org/).
    - We'll use VirtualBox for this project.
- Serial communication program in Linux: _picocom_.

## System Architecture

### Host and Target

<div align="center">
  <img width="531" height="471" alt="el-host-target drawio" src="https://github.com/user-attachments/assets/2e45396d-f5d5-4301-8816-8685eb341b01" />
</div>

- **Cross-compilation toolchain**: Compiler that runs on the development machine, but generates code for the target.
- **Bootloader**: Started by the hardware, responsible for basic initialization, loading and executing the kernel.
- **Linux Kernel**: Contains the process and memory management, network stack, device drivers and provides services to user space applications.
- **C library**: library of C functions. Also the interface between the kernel and the userspace applications.
- **Libraries and applications**: Third-party or in-house.

### Enbedded Linux Work

Several distinct tasks are needed when deploying embedded Linux in a product:
- Board Support Package development
    - A BSP contains a bootloader and kernel with the suitable device drivers for the targeted hardware.
- System integration
    - Integrate all the components, bootloader, kernel, third-party libraries and applications and in-house applications into a working system.
- Development of applications
    - Normal Linux applications, but using specifically chosen libraries.

## Toolchain

### Overview

<div align="center">
  <img width="691" height="331" alt="el-toolchain drawio" src="https://github.com/user-attachments/assets/f714fe87-9bde-494e-887e-3653f82aa839" />
</div>

Many UNIX/Linux build mechanisms rely on architecture tuple names to identify machines.
- Examples: arm-linux-gnueabihf, mips64el-linux-gnu, arm-vendor-none-eabihf.

These tuples are 3 or 4 parts:
1. The architecture name: `arm`, `riscv`, `mips64el`, etc.
2. Optionally, a vendor name, which is a free-form string.
3. An operating system name, or `none` when not targeting an operating system.
4. The ABI/C library.

### Components of gcc toolchain

- Binutils
    - Set of tools to generate and manipulate binaries (usually with the ELF format) for a given CPU architecture. https://www.gnu.org/software/binutils/
- C/C++ compiler
    - GCC: GNU Compiler Collection, the famous free software compiler.
- Kernel headers
    - The C standard library and compiled programs need to interact with the kernel.
    - The kernel to userspace interface is **backward compatible**. Binaries generated with toolchain using kernel headers older than the running kernel will work but won't be able to use new system call, data structure, etc.
- C/C++ libraries
    - The C standard library is an essential component of a Linux system. Interface between the applications and the kernel, provides well-known standard C API to ease application development.
    - Several C standard libraries: `glibc`, `uClibc`, `musl`, `newlib`, etc.
    - The choice of the C standard library must be made at cross-compiling toolchain generation time. As the GCC compiler is compiled against a specific C standard library.
- GDB debugger (optional)

### Toolchain options

#### ABI - Application Binary Interface
When building a toolchain, the ABI used to generate binaries need to be defined.
- It defines the calling conventions:
    - How function arguments are passed.
    - How the return value is passed.
    - How system calls are made.
    - Organization of structures (alignments, etc.)

All binaries in a system are typically compiled with the same ABI, kernel must understand this ABI.

On Arm 32-bit, two mains ABIs: `EABI` and `EABIhf`:
- EABIhf passes floating-point arguments in floating-point registers. It need an Arm processor with a FPU (Floating Point Unit).

#### Floating point support
Armv7-A (32-bit) and Armv8-A (64-bit) processors have a floating point unit.

For processor without a floating point unit, two solutions for floating point computation:
- Generate hard float code and rely on kernel to emulate floating point instructions -> Very slow
- Generate soft float code, so that instead of generating floating point instructions, calls to a userspace library are generated.
Decision is taken at toolchain configuration time.

FPU on Arm: VFPv3, VFPv3-D16, VFPv4, VFPv4-D16, etc.

#### CPU optimization flags
GNU tools (gcc, binutils) can only be compiled for a specific target architecture at a time (Arm, x86, RISC-V, etc).

`gcc` offers further options:
- `-march` allows to select a specific target instruction set.
- `-mtune` allows to optimize code for a specific CPU.
- Example: `-march=armv7 -mtune=cortex-a8`
- `-mcpu=cortex-a8` can be used instead to allow gcc to infer the target instruction set and CPU optimizations.
- https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html

## Boot chain

### Generic boot sequence

- Starting Linux on a processor is done in several steps that progressively initialize the platform peripherals and memories. Image source [ST - Boot chain overview](https://wiki.st.com/stm32mpu/wiki/Boot_chain_overview#Linux_start-up). The following image show a generic boot sequence:

<div align="center">
    <img width="681" height="541" alt="el-boot-chain drawio" src="https://github.com/user-attachments/assets/538a3f4d-4028-461a-84e3-b456b0cdea1f" />
</div>

### Boot sequence on STM32MP15 series.

- [Image source](https://wiki.st.com/stm32mpu/wiki/Boot_chain_overview#STM32MP15_boot_chain).

<div align="center">
    <img width="700" height="525" alt="image" src="https://github.com/user-attachments/assets/10428761-6f24-42ce-8da3-4d32bb5b585f" />
</div>

### ROM - Boot from SDCard on STM32MP15 series.

- SD cards contain two versions of the FSBL. The ROM code tries to load and launch the first copy. In the case of failure, it then tries to load the second copy.
- The ROM code first looks for a Global Positioning Table (GPT).
- If it finds it:
    - On STM32MP15x lines, it locates two FSBLs by looking for the two first GPT entries of which the name begins with `fsbl`.

## Linux kernel

### Overview

- **Manage the hardware resources**: CPU, memory, I/O.
- **Provide a set of portable, architecture and hardware independent APIs** to allow user space applications and libraries to use the hardware resources.
- **Handle concurrent accesses and usage of hardware** resources from different applications.
    - Example: a single network interface is used by multiple user space applications through various network connections. The kernel is responsible for “multiplexing” the hardware resource.

=> The main interface between the kernel and user space is the set of system calls.

=> This system call interface is wrapped by the standard C library, and user space applications usually never make a system call.

- Linux makes system and kernel information available in user space through **pseudo filesystems**, sometimes also called **virtual filesystems**:
    - Directories and files that do not exist on any real storage.
    - `proc`, mounted on `/proc`
      - Operating system related information (processes, memory management parameters...)
      - `mount -t proc nodev /proc`
    - `sysfs`, mounted on `/sysfs`
      - Representation of the system as a tree of devices connected by buses.
      - `mount -t sysfs nodev /sys`

### Root filesystem

Filesystems are used to organize data in directories and files on storage devices or on the network. The directories and files are organized as a hierarchy
- In UNIX systems, applications and users see a single global hierarchy of files and
directories, which can be composed of several filesystems.
- Filesystems are mounted in a specific location in this hierarchy of directories
    - When a filesystem is mounted in a directory (called mount point), the contents of this directory reflect the contents of this filesystem.
    - When the filesystem is unmounted, the mount point is empty again.
= This allows applications to access files and directories easily, regardless of their exact storage location

`mount` allows to mount filesystems
- `mount -t type device mountpoint`
- `type` is the type of filesystem (optional for non-virtual filesystems)
- `device` is the storage device, or network location to mount
• `mountpoint` is the directory where files of the storage device or network location will be accessible

`umount` allows to unmount filesystems
- This is needed before rebooting, or before unplugging a USB key, because the Linux kernel caches writes in memory to increase performance. umount makes sure that these writes are committed to the storage

A particular filesystem is mounted at the root of the hierarchy, identified by `/`. This filesystem is called the root filesystem

As `mount` and `umount` are programs, they are files inside a filesystem. They are not accessible before mounting at least one filesystem.

As the root filesystem is the first mounted filesystem, it cannot be mounted with the normal mount command. It is mounted directly by the kernel, according to the `root=` kernel option

It can be mounted from different locations
- From the partition of a hard disk
- From the partition of a USB key
- From the partition of an SD card
- From the partition of a NAND flash chip or similar type of storage device
- From the network, using the NFS protocol
    - Install an NFS server on workstation (example: Debian, Ubuntu)
    - `sudo apt install nfs-kernel-server`
- From memory, using a pre-loaded filesystem (by the bootloader)

### Kernel building

1. Environment setup and configuration.
    1. Specify target architecture: `export ARCH=arm`
    2. Specify cross-compiler: `export CROSS_COMPILE=arm-linux-`
    3. Kernel configuration, get reference: `make soc_defconfig`
    4. Customize configuration: `make menuconfig`
2. Kernel building and deployment.
    1. Compile the kernel and modules: `make`
    2. Install kernel/modules:
       - Kernel: `make install` or manual copy
       - Modules: `make modules_install`

### Kernel to first user space application boot process

1. Bootloader (e.g `U-Boot`)
    1. Loads kernel image + `.dtb` to RAM, start the kernel.
2. Kernel
    1. Initialize hardware devices and kernel subsystems.
    2. Mount the `root /` filesystem indicated by `root=`.
    3. Start the init application, `/sbin/init` by default (e.g BusyBox).
3. `/sbin/init`
    1. Starts other user space services and application.
      - Shell.
      - Other application.

The sequence above could apply to booting with Network File System (NFS).
