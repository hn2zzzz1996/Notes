# The bind, unbind, new_id file in /sys

Before we want to use VFIO framework to passthrough a device into guest. We should first unbind the device from its original driver, and rebind it to the `vfio_pci` driver. Thus the `vfio_pci` can take control of the device.

If we want to passthrough a device which BDF is `0000:00:02.0`, so we need to use these command:

```C
/sys/bus/pci/devices/0000:00:02.0$ ls -ld driver
lrwxrwxrwx 1 root root 0 Oct 10 01:16 driver -> ../../../bus/pci/drivers/i915
/sys/bus/pci/devices/0000:00:02.0$ cd driver
/sys/bus/pci/devices/0000:00:02.0/driver$ ls
0000:00:02.0  bind  module  new_id  remove_id  uevent  unbind
```

We can see, now the device `0000:00:02.0` is bind to the driver `i915`, and we can see multiple file in the driver directory. And we need to use them to unbind and bind device to a driver.

Continue:

```C
/sys/bus/pci/devices/0000:00:02.0/driver$ echo 0000:00:02.0 > unbind
```

Now the device is unbind from the `i915` driver, then we need to bind it to the `vfio_pci` driver.

```C
// Switch directory to vfio-pci driver
/sys/bus/pci/drivers/vfio-pci$ ls
bind  module  new_id  remove_id  uevent  unbind

// Get the vender:device
/sys/bus/pci/drivers/vfio-pci$ lspci -n -s 0000:02:00.0
02:00.0 0200: 8086:15f3 (rev 03)

/sys/bus/pci/drivers/vfio-pci$ echo 8086 15f3 > new_id
/sys/bus/pci/drivers/vfio-pci$ echo 0000:02:00.0 > bind
```

Now device `0000:00:02.0` has been bind to the `vfio_pci` driver.



We now know how to operate these file, but what it's mean, and what input they get. Here is an detailed explanation:

```
What:		/sys/bus/pci/drivers/.../bind
What:		/sys/devices/pciX/.../bind
Date:		December 2003
Contact:	linux-pci@vger.kernel.org
Description:
		Writing a device location to this file will cause
		the driver to attempt to bind to the device found at
		this location.	This is useful for overriding default
		bindings.  The format for the location is: DDDD:BB:DD.F.
		That is Domain:Bus:Device.Function and is the same as
		found in /sys/bus/pci/devices/.  For example::

		  # echo 0000:00:19.0 > /sys/bus/pci/drivers/foo/bind
		  
		The same time, the probe callback will be called in the driver.
```

```
What:		/sys/bus/pci/drivers/.../unbind
What:		/sys/devices/pciX/.../unbind
Date:		December 2003
Contact:	linux-pci@vger.kernel.org
Description:
		Writing a device location to this file will cause the
		driver to attempt to unbind from the device found at
		this location.	This may be useful when overriding default
		bindings.  The format for the location is: DDDD:BB:DD.F.
		That is Domain:Bus:Device.Function and is the same as
		found in /sys/bus/pci/devices/. For example::

		  # echo 0000:00:19.0 > /sys/bus/pci/drivers/foo/unbind
```

```
What:		/sys/bus/pci/drivers/.../new_id
What:		/sys/devices/pciX/.../new_id
Date:		December 2003
Contact:	linux-pci@vger.kernel.org
Description:
		Writing a device ID to this file will attempt to
		dynamically add a new device ID to a PCI device driver.
		This may allow the driver to support more hardware than
		was included in the driver's static device ID support
		table at compile time.  The format for the device ID is:
		VVVV DDDD SVVV SDDD CCCC MMMM PPPP.  That is Vendor ID,
		Device ID, Subsystem Vendor ID, Subsystem Device ID,
		Class, Class Mask, and Private Driver Data.  The Vendor ID
		and Device ID fields are required, the rest are optional.
		Upon successfully adding an ID, the driver will probe
		for the device and attempt to bind to it.  For example::

		  # echo "8086 10f5" > /sys/bus/pci/drivers/foo/new_id
```

We can read it completely at the kernel document.

https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-pci