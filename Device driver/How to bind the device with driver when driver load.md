# How to bind the device with driver when driver load

```
#0  virtio_dev_match (_dv=0xffff8881009e2010, _dr=0xffffffff830862a0 <virtio_balloon_driver>) at drivers/virtio/virtio.c:85
#1  0xffffffff818150b8 in driver_match_device (dev=0xffff8881009e2010, drv=0xffffffff830862a0 <virtio_balloon_driver>) at drivers/base/base.h:147
#2  __driver_attach (dev=0xffff8881009e2010, data=0xffffffff830862a0 <virtio_balloon_driver>) at drivers/base/dd.c:1111
#3  0xffffffff818120b9 in bus_for_each_dev (bus=<optimized out>, start=start@entry=0x0 <fixed_percpu_data>, data=data@entry=0xffffffff830862a0 <virtio_balloon_driver>,
    fn=fn@entry=0xffffffff81815090 <__driver_attach>) at drivers/base/bus.c:301
#4  0xffffffff81813d8e in driver_attach (drv=drv@entry=0xffffffff830862a0 <virtio_balloon_driver>) at drivers/base/dd.c:1161
#5  0xffffffff81813712 in bus_add_driver (drv=drv@entry=0xffffffff830862a0 <virtio_balloon_driver>) at drivers/base/bus.c:618
#6  0xffffffff81815b95 in driver_register (drv=drv@entry=0xffffffff830862a0 <virtio_balloon_driver>) at drivers/base/driver.c:171
#7  0xffffffff817828d0 in register_virtio_driver (driver=driver@entry=0xffffffff830862a0 <virtio_balloon_driver>) at drivers/virtio/virtio.c:347
#8  0xffffffff832417d1 in virtio_balloon_driver_init () at drivers/virtio/virtio_balloon.c:1168
#9  0xffffffff81000fd7 in do_one_initcall (fn=0xffffffff832417bc <virtio_balloon_driver_init>) at init/main.c:1300
#10 0xffffffff831efa3f in do_initcall_level (command_line=0xffff8881003a3780 "noinitrd", level=6) at init/main.c:1373
#11 do_initcalls () at init/main.c:1389
#12 do_basic_setup () at init/main.c:1408
#13 kernel_init_freeable () at init/main.c:1613
#14 0xffffffff81c972ca in kernel_init (unused=<optimized out>) at init/main.c:1502
#15 0xffffffff81001b9f in ret_from_fork () at arch/x86/entry/entry_64.S:295
#16 0x0000000000000000 in ?? ()
```

Here is an example that the virtio_balloon driver loaded, and it will bind with the device.

First we see the `virtio_balloon_driver_init` and it calls the `register_virtio_driver`.

```C
  342 int register_virtio_driver(struct virtio_driver *driver)
  343 {
  344         /* Catch this early. */
  345         BUG_ON(driver->feature_table_size && !driver->feature_table);
  346         driver->driver.bus = &virtio_bus;
  347         return driver_register(&driver->driver);
  348 }
```
