# The difference between driver_register and device_register

## driver_register

The `driver_register` will register the driver on to a given bus. And in the same time iterate all the device on the bus and try to match a device on it.

```C
/**
 * driver_register - register driver with bus
 * @drv: driver to register
 *
 * We pass off most of the work to the bus_add_driver() call,
 * since most of the things we have to do deal with the bus
 * structures.
 */
int driver_register(struct device_driver *drv)
{
    ret = bus_add_driver(drv);
}
```

```C
/**
 * bus_add_driver - Add a driver to the bus.
 * @drv: driver.
 */
int bus_add_driver(struct device_driver *drv)
{
    struct bus_type *bus;
    
    bus = bus_get(drv->bus);
    
    error = driver_attach(drv);
}
```

```C
/**
 * driver_attach - try to bind driver to devices.
 * @drv: driver.
 *
 * Walk the list of devices that the bus has on it and try to
 * match the driver with each one.  If driver_probe_device()
 * returns 0 and the @dev->driver is set, we've found a
 * compatible pair.
 */
int driver_attach(struct device_driver *drv)
{
	return bus_for_each_dev(drv->bus, NULL, drv, __driver_attach);
}
```

The `bus_for_each_dev` will iterate all the device on the bus that the driver belongs to, and call `__driver_attach` for each device.

```C
/drivers/base/dd.c
int __driver_attach(struct device *dev, void *data)
{
    struct device_driver *drv = data;
    
    ret = driver_match_device(drv, dev);
    
    // If find driver can control this device, call probe
    driver_probe_device(drv, dev);
}
```

```C
driver_probe_device
    -> __driver_probe_device
    	-> really_probe

int really_probe(struct device *dev, struct device_driver *drv)
{
    dev->driver = drv;
    
    ret = call_driver_probe(dev, drv);
}
```

```C
static int call_driver_probe(struct device *dev, struct device_driver *drv)
{
    if (dev->bus->probe)
        // The bus_probe will also call dri->probe(), but do a little more thing before call it.
		ret = dev->bus->probe(dev);
	else if (drv->probe)
		ret = drv->probe(dev);
}
```

## device_register

The `device_register` will register the device onto the given bus. And will find a matched driver and attach on it.

```C
/**
 * device_register - register a device with the system.
 * @dev: pointer to the device structure
 */
int device_register(struct device *dev)
{
	device_initialize(dev);
	return device_add(dev);
}
```

```C
/**
 * device_add - add device to device hierarchy.
 * @dev: device.
 */
int device_add(struct device *dev)
{
    error = bus_add_device(dev);
}
```

```C
int bus_add_device(struct device *dev)
{
    bus_probe_device(dev);
}
```

```C
void bus_probe_device(struct device *dev)
{
	struct bus_type *bus = dev->bus;
    
    device_initial_probe(dev);
}
```

```C
void device_initial_probe(struct device *dev)
{
	__device_attach(dev, true);
}

static int __device_attach(struct device *dev, bool allow_async)
{
    ret = bus_for_each_drv(dev->bus, NULL, &data,
					__device_attach_driver);
}
```

```C
static int __device_attach_driver(struct device_driver *drv, void *_data)
{
    ret = driver_match_device(drv, dev);
    
    ret = driver_probe_device(drv, dev);
}
```

## Conclusion

To simplify, the `driver_register` must be called when a driver loaded into system, and it must register it with a specific `bus_type` then it can take over the device on this bus. The `device_register` is called when the device is found by system and register it on a specific bus, and this will also find a driver to take over it.