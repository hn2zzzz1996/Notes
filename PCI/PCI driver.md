[TOC]

# PCI driver

PCI devices are `automatically configured at boot time`. The device driver must be able to access configuration information in the device in order to complete initialization.

## PCI Addressing

Each PCI peripheral is identified by a `bus` number, a `device` number, and a `function` number. The PCI specification permits a single system to host up to 256 buses, each bus hosts up to 32 devices, and each device can be a multifunction board with a maximum of 8 functions. Therefore, each function can be identified at hardware level by a `16-bit` address. Device drivers don't need to deal with binary address, they use **pci_dev** to act on the devices.

When the hardware address is displayed, it can be shown:

* two values: 8-bit bus number and 8-bit device and function number

  * ```C
    hsq@q:~$ cat /proc/bus/pci/devices | cut -f1
    0000
    0008
    0010
    0040
    0090
    00a0
    00a2
    00a8
    00b0
    00b8
    00d8
    00e0
    00f8
    00fb
    00fc
    00fd
    00fe
    0100
    0101
    0200
    0300
    ```

* three values: bus, device, and function

  * ```C
    hsq@q:~$ lspci
    00:00.0 Host bridge: Intel Corporation Device 9b43 (rev 05)
    00:01.0 PCI bridge: Intel Corporation 6th-10th Gen Core Processor PCIe Controller (x16) (rev 05)
    00:02.0 VGA compatible controller: Intel Corporation CometLake-S GT2 [UHD Graphics 630] (rev 05)
    00:08.0 System peripheral: Intel Corporation Xeon E3-1200 v5/v6 / E3-1500 v5 / 6th/7th/8th Gen Core Processor Gaussian Mixture Model
    00:12.0 Signal processing controller: Intel Corporation Comet Lake PCH Thermal Controller
    00:14.0 USB controller: Intel Corporation Comet Lake USB 3.1 xHCI Host Controller
    00:14.2 RAM memory: Intel Corporation Comet Lake PCH Shared SRAM
    00:15.0 Serial bus controller [0c80]: Intel Corporation Comet Lake PCH Serial IO I2C Controller #0
    00:16.0 Communication controller: Intel Corporation Comet Lake HECI Controller
    00:17.0 SATA controller: Intel Corporation Device 06d2
    00:1b.0 PCI bridge: Intel Corporation Comet Lake PCI Express Root Port #21 (rev f0)
    00:1c.0 PCI bridge: Intel Corporation Device 06bd (rev f0)
    00:1f.0 ISA bridge: Intel Corporation Device 0687
    00:1f.3 Audio device: Intel Corporation Comet Lake PCH cAVS
    00:1f.4 SMBus: Intel Corporation Comet Lake PCH SMBus Controller
    00:1f.5 Serial bus controller [0c80]: Intel Corporation Comet Lake PCH SPI Controller
    00:1f.6 Ethernet controller: Intel Corporation Ethernet Connection (11) I219-LM
    01:00.0 VGA compatible controller: NVIDIA Corporation GK208B [GeForce GT 730] (rev a1)
    01:00.1 Audio device: NVIDIA Corporation GK208 HDMI/DP Audio Controller (rev a1)
    02:00.0 Non-Volatile memory controller: SK hynix BC501 NVMe Solid State Drive 512GB
    03:00.0 PCI bridge: Texas Instruments XIO2001 PCI Express-to-PCI Bridge
    ```

* four values: domain, bus, device and function

  * ```Shell
    hsq@q:~$ tree /sys/bus/pci/devices/
    /sys/bus/pci/devices/
    ├── 0000:00:00.0 -> ../../../devices/pci0000:00/0000:00:00.0
    ├── 0000:00:01.0 -> ../../../devices/pci0000:00/0000:00:01.0
    ├── 0000:00:02.0 -> ../../../devices/pci0000:00/0000:00:02.0
    ├── 0000:00:08.0 -> ../../../devices/pci0000:00/0000:00:08.0
    ├── 0000:00:12.0 -> ../../../devices/pci0000:00/0000:00:12.0
    ├── 0000:00:14.0 -> ../../../devices/pci0000:00/0000:00:14.0
    ├── 0000:00:14.2 -> ../../../devices/pci0000:00/0000:00:14.2
    ├── 0000:00:15.0 -> ../../../devices/pci0000:00/0000:00:15.0
    ├── 0000:00:16.0 -> ../../../devices/pci0000:00/0000:00:16.0
    ├── 0000:00:17.0 -> ../../../devices/pci0000:00/0000:00:17.0
    ├── 0000:00:1b.0 -> ../../../devices/pci0000:00/0000:00:1b.0
    ├── 0000:00:1c.0 -> ../../../devices/pci0000:00/0000:00:1c.0
    ├── 0000:00:1f.0 -> ../../../devices/pci0000:00/0000:00:1f.0
    ├── 0000:00:1f.3 -> ../../../devices/pci0000:00/0000:00:1f.3
    ├── 0000:00:1f.4 -> ../../../devices/pci0000:00/0000:00:1f.4
    ├── 0000:00:1f.5 -> ../../../devices/pci0000:00/0000:00:1f.5
    ├── 0000:00:1f.6 -> ../../../devices/pci0000:00/0000:00:1f.6
    ├── 0000:01:00.0 -> ../../../devices/pci0000:00/0000:00:01.0/0000:01:00.0
    ├── 0000:01:00.1 -> ../../../devices/pci0000:00/0000:00:01.0/0000:01:00.1
    ├── 0000:02:00.0 -> ../../../devices/pci0000:00/0000:00:1b.0/0000:02:00.0
    └── 0000:03:00.0 -> ../../../devices/pci0000:00/0000:00:1c.0/0000:03:00.0
    ```

## How PCI works

### Boot Time

When system boot, The devices are configured.

At system boot, the firmware communicate with every PCI peripheral in order to allocate a safe place for each address region it offers. The device's memory and I/O regions have already been mapped into the processor's address space.

The individual PCI device directories in the sysfs tree can be found in `/sys/bus/pci/devices`. It contains a number of different files:

```Shell
$ tree /sys/bus/pci/devices/0000:00:10.0
/sys/bus/pci/devices/0000:00:10.0
|-- class
|-- config			# binary file that allows the raw PCI config information to be read from the device
|-- detach_state
|-- device
|-- irq
|-- power
| `-- state
|-- resource
|-- subsystem_device
|-- subsystem_vendor
`-- vendor
```

### Configuration Registers and Initialization

All PCI devices feature at least a 256-byte address space. The first 64 bytes are standardized, which the rest are device dependent.

![The standardized PCI configuration registers](.\picture\The standardized PCI configuration registers.png)

Three or five PCI registers identify a device:

* vendorID

  16-bit register identifies a hardware manufacturer. Every Intel device is marked with the same vendor number 0x8086.

* deviceID

  16-bit register. Usually paired with the vendor ID to make a unique 32-bit identifier for a hardware device. And the paired vendorID and deviceID always as a `signature`, a device driver relies on it to identify its device.

* class

  Every peripheral device belongs to a class. The class register is a 16-bit value whose top 8 bits identify the `base class (or group)` . For example, "ethernet" and "token ring" are two classes belonging to the "network" group.

* subsystem vendorID

* subsystem deviceID

  These fields can be used for further identification of a device. If the chip is a generic interface chip to a local (onboard) bus, it is often used in several completely different roles, and the driver must identify the actual device it is talking with. The subsystem identifiers are used to this end.

The `struct pci_device_id` structure is used to define a list of the different types of PCI devices that a driver supports.

```C
/**
 * struct pci_device_id - PCI device ID structure
 * @vendor:		Vendor ID to match (or PCI_ANY_ID)
 * @device:		Device ID to match (or PCI_ANY_ID)
 * @subvendor:		Subsystem vendor ID to match (or PCI_ANY_ID)
 * @subdevice:		Subsystem device ID to match (or PCI_ANY_ID)
 * @class:		Device class, subclass, and "interface" to match.
 *			See Appendix D of the PCI Local Bus Spec or
 *			include/linux/pci_ids.h for a full list of classes.
 *			Most drivers do not need to specify class/class_mask
 *			as vendor/device is normally sufficient.
 * @class_mask:		Limit which sub-fields of the class field are compared.
 *			See drivers/scsi/sym53c8xx_2/ for example of usage.
 * @driver_data:	Data private to the driver.
 *			Most drivers don't need to use driver_data field.
 *			Best practice is to use driver_data as an index
 *			into a static list of equivalent device types,
 *			instead of using it as a pointer.
 * @override_only:	Match only when dev->driver_override is this driver.
 */
struct pci_device_id {
	__u32 vendor, device;		/* Vendor and device ID or PCI_ANY_ID*/
	__u32 subvendor, subdevice;	/* Subsystem ID's or PCI_ANY_ID */
	__u32 class, class_mask;	/* (class,subclass,prog-if) triplet */
	kernel_ulong_t driver_data;	/* Data private to the driver */
	__u32 override_only;
};
```

There are two helper macros that should be used to initialize a struct pci_device_id structure:

* PCI_DEVICE(vendor, device)

  This creates a struct pci_device_id that matches only the specific vendor and device ID. The macro sets the subvendor and subdevice fields of the structure to PCI_ANY_ID.

* PCI_DEVICE_CLASS(device_class, device_class_mask)

  This creates a struct pci_device_id that matches a specific PCI class.

### Registering a PCI Driver

The main structure that all PCI drivers must create is `struct pci_driver`, and then driver can register the pci device to kernel.

Here are the fields in this structure that a PCI driver needs to be aware of:

* **const char *name;**

  The name of the driver. It must be unique among all PCI drivers in the kernel and is normally set to the same name as the module name of the driver. It shows up in sysfs under `/sys/bus/pci/drivers/` when the driver is in the kernel.

* **const struct pci_device_id *id_table;**

  Pointer to table of device IDs the driver is interested in.

* **int (*probe) (struct pci_dev *dev, const struct pci_device_id *id);**

  New device inserted.

* **void (*remove) (struct pci_dev *dev);**

  Device removed.

In summary, to create a proper struct pci_driver structure, only four fields need to be initialized:

```C
static struct pci_driver pci_driver = {
 .name = "pci_skel",
 .id_table = ids,
 .probe = probe,
 .remove = remove,
};`
```

**pci_register_driver** is used to register the **pci_driver** to the PCI core.

```C
static int __init pci_skel_init(void)
{
    // Return 0 means success.
	return pci_register_driver(&pci_driver);
}
```

**pci_unregister_driver** is used to unregister the **pci_driver**.

```C
static void __exit pci_skel_exit(void)
{
	pci_unregister_driver(&pci_driver);
}
```

**The new_id file**

The 2.6 kernel allows new `PCI IDs` to be dynamically allocated to a driver after is has been loaded. This is done through the file `new_id` that is created in all PCI driver directories in sysfs. `This is very useful` if a new device is being used that the kernel doesn’t know about just yet. A user can write the PCI ID values to the new_id file, and then the driver binds to the new device.

### Enabling the PCI Device

In the `probe` function, before the driver can access any device resource (I/O region or interrupt) of the PCI device, the driver must call the `pci_enable_device` function:

```C
int pci_enable_device(struct pci_dev *dev);

This function actually enables the device. It wakes up the device and in some
cases also assigns its interrupt line and I/O regions.
```

### Accessing the Configuration Space

Accessing the configuration space is vital to the driver, because it is the only way it can find out where `the device is mapped in memory and in the I/O space`.

The CPU can't access the configuration space directly, CPU should write and read the registers in the PCI controller. Linux offers a standard interface to access the configuration space.

```C
int pci_read_config_byte(struct pci_dev *dev, int where, u8 *val);
int pci_read_config_word(struct pci_dev *dev, int where, u16 *val);
int pci_read_config_dword(struct pci_dev *dev, int where, u32 *val);
```

Read one, two, or four bytes from the configuration space of the device identified by `dev`. The `where` argument is the **byte offset from the beginning** of the configuration space. The value fetched from the configuration space is returned through the `val` pointer, and the `return value` of the functions is an error code.

The word and dword functions convert the value just read from little-endian to the native byte order of the processor, so you need not deal with byte ordering.

```C
int pci_write_config_byte(struct pci_dev *dev, int where, u8 val);
int pci_write_config_word(struct pci_dev *dev, int where, u16 val);
int pci_write_config_dword(struct pci_dev *dev, int where, u32 val);
```

That's the write function. The parameter is same with read function.

### Accessing the I/O and Memory Spaces

A PCI device implements up to `six I/O address regions (See Picture Above, the Base Address0-5)`. Each region consists of either **memory (most device use it, commonly MMIO)** or I/O location.

The I/O registers **should not be cached** by the CPU because each access can **have side effects**.  The PCI device that implements I/O registers as a memory region marks the difference by setting a “**memory-is-prefetchable**” bit in its configuration register. If the memory region is marked as prefetchable, the CPU can cache its contents and do all sorts of optimization with it.

In kernel, the `I/O regions of PCI devices` **have been integrated into the generic resource management.** So you don't need to access the configuration register. You should use the following functions to get region information:

```C
unsigned long pci_resource_start(struct pci_dev *dev, int bar);
```

The function returns the first address (memory address or I/O port number) associated with one of the six PCI I/O regions. The region is selected by the integer bar (the base address register), ranging from 0–5 (inclusive).

```C
unsigned long pci_resource_end(struct pci_dev *dev, int bar);
```

The function returns the last address that is part of the I/O region number bar. Note that this is the last usable address, not the first address after the region.

```C
unsigned long pci_resource_flags(struct pci_dev *dev, int bar);
```

This function returns the flags associated with this resource.



Resource flags are used to define some features of the individual resource.

All resource flags are defined in ; the most important are:

**IORESOURCE_IO** 

**IORESOURCE_MEM** 

​	If the associated I/O region exists, one and only one of these flags is set.

**IORESOURCE_PREFETCH**

**IORESOURCE_READONLY**

​	These flags tell whether a memory region is prefetchable and/or write protected. 

​	The latter flag is never set for PCI resources.

### PCI Interrupts

By the time linux boots, the computer's firmware `has already assigned a unique interrupt number to the device`, and the driver just need to use it.

The interrupt number is stored in configuration register 60 (**PCI_INTERRUPT_LINE**), one byte wide. The value found in `PCI_INTERRUPT_LINE` is guaranteed to be the right one.

```C
result = pci_read_config_byte(dev, PCI_INTERRUPT_LINE, &myirq);
if (result) {
    /* deal with error */
}
```

## PCI Driver Example

The QEMU provide a edu device which is a pci device, and can use it to learn the pci driver.

https://github.com/qemu/qemu/blob/master/docs/specs/edu.txt

Here is an example driver code: https://github.com/malvag/qemu_edu_device_driver