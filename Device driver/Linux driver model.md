# Linux driver model

## Driver Binding

Driver binding is the process of associating a device with a device driver that can control it. Bus drivers have typically handled this.

### Bus

The bus type structure contains:

1. A list of all `devices` on that bus type.
2. A list of all `drivers` on that bus type.

When device or driver been added to the bus, it will trigger the driver binding.

### device_register

When a new device is added, the bus's list of drivers is iterated over to find one that supports it. In order to determine that, the device ID of the device must match one of the device IDs that the driver supports. It's up to bus driver to provide a callback to compare a device against the IDs of a driver:

```C
// Returns 1 if a match was found; 0 otherwise.
int match(struct device * dev, struct device_driver * drv);
```

If a match is found, the `device's driver field is set to the driver` and the driver's probe callback is called. This gives the driver a chance to verify that it really does support the hardware.

### Device Class

`struct class - device class.` A class is a higher-level view of a device that abstracts out low-level implementation details. Drivers may see a SCSI disk or an ATA disk, but, at the class level, they are all simply disks. Classes allow user space to work with devices based on what they do, rather than how they are connected or how they work.

Upon the successful completion of probe, the device is registered with the class it belongs to.

Device drivers belong to one and only one class, and that is set in the driver's `devclass` field.

### Driver

When a driver is attached to a device, the device is inserted into the driver's list of devices.

### sysfs

### driver_register

 The bus’s list of devices is iterated over to find a match. Devices that already have a driver are skipped. All the devices are iterated over, to bind as many devices as possible to the driver.

## Bus Types

### Declaration

Each bus type in the kernel (PCI, USB, etc) should declare one static object of this type. They must initialize the name field, and may optionally initialize the match callback:

```C
struct bus_type pci_bus_type = {
       .name = "pci",
       .match        = pci_bus_match,
};
```

The structure should be exported to drivers in a header file:

```C
extern struct bus_type pci_bus_type;
```

### Registration

When a bus driver is initialized, it calls bus_register. This initializes the rest of the fields in the bus object and inserts it into a global list of bus types. Once the bus object is registered, the fields in it are usable by the bus driver.

### match(): Attaching Drivers to Devices

The format of device ID structures and the semantics for comparing them are inherently bus-specific.

Drivers typically declare an array of device IDs of devices they supported that reside in a bus-specific driver structure.

When a driver is registered with the bus, the bus’s list of devices is iterated over, and the match callback is called for each device that does not have a driver associated with it.

### Device and Driver Lists

There are two helper functions for iterating the two lists:

```C
int bus_for_each_dev(struct bus_type * bus, struct device * start,
                     void * data,
                     int (*fn)(struct device *, void *));

int bus_for_each_drv(struct bus_type * bus, struct device_driver * start,
                     void * data, int (*fn)(struct device_driver *, void *));
```

See it's source code for more detail.

### sysfs

There is a top-level directory named 'bus'.

Each bus gets a directory in the bus directory, along with two default directories, like pci:

```Shell
/sys/bus/pci/
|-- devices
`-- drivers
```

Drivers registered with the bus get a directory in the bus's drivers directory:

```Shell
/sys/bus/pci/
|-- devices
`-- drivers
    |-- Intel ICH
    |-- Intel ICH Joystick
    |-- agpgart
    `-- e100
```

Each device that is discovered on a bus of that type gets a symlink in the bus's devices directory to the device's directory in the physical hierarchy:

```Shell
/sys/bus/pci/
|-- devices
|   |-- 00:00.0 -> ../../../root/pci0/00:00.0
|   |-- 00:01.0 -> ../../../root/pci0/00:01.0
|   |-- 00:02.0 -> ../../../root/pci0/00:02.0
|   `-- 0000:00:00.0 -> ../../../devices/pci0000:00/0000:00:00.0/
`-- drivers
```

## The kobject



https://www.youtube.com/watch?v=AdPxeGHIZ74

