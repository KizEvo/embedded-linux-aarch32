## Intro

Practical guide and troubleshoot while learning embedded linux on STM32MP157.

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
