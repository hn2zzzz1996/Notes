# Virtio Source Code Analysis

This code analysis based on kernel 4.5.0-rc6. And this version has the clear logic of virtio.

All virtio devices is a pci device. So it will first be first handled by the `virtio_pci_driver`, and then the `virtio_pci_driver`  will register the device on the `virtio_bus`, and then the device will be managed by the `virtio_bus`, and the `virtio_bus` will find the actually driver the device belongs to.

## Virtio data structure

The `struct virtio_device` is the common information for every device that using `virtio` protocol to communicate with frontend and backend.

```C
// include/linux/virtio.h
// virtio_device - representation of a device using virtio
struct virtio_device {
    int index;	// unique position on the virtio bus
    struct device dev; // underlying device.
    struct virtio_device_id; // the device type identification (used to match it with a driver)
    struct virtio_config_ops *config;	// the configuration ops for this device.
    struct vringh_config_ops *vringh_config; // configuration ops for host vrings.
}
```

The `virtio_pci_device` is based on `pci` and `virtio`. So it will first been detected by the PCI driver frame, and then it will be attached to the `virtio-bus` which implemented by virtio protocol. So these devices will be controlled by the virtio frame after the pci_probe has executed.

```C
struct virtio_pci_device {
    struct virtio_device vdev;
    struct pci_dev *pci_dev;
  	struct virtio_pci_legacy_device ldev;
    
    /* Whether we have vector per vq */
	bool per_vq_vectors;

	struct virtqueue *(*setup_vq)(struct virtio_pci_device *vp_dev,
				      struct virtio_pci_vq_info *info,
				      unsigned int idx,
				      void (*callback)(struct virtqueue *vq),
				      const char *name,
				      bool ctx,
				      u16 msix_vec);
	void (*del_vq)(struct virtio_pci_vq_info *info);

	u16 (*config_vector)(struct virtio_pci_device *vp_dev, u16 vector);
};
```

## Virtio PCI init

First, the virtio will register a pci driver which name is `virtio_pci_driver`:

```C
// driver/virtio/virtio_pci_common.c
561 static struct pci_driver virtio_pci_driver = {
  562         .name           = "virtio-pci",
  563         .id_table       = virtio_pci_id_table,
  564         .probe          = virtio_pci_probe,
  565         .remove         = virtio_pci_remove,
  566 #ifdef CONFIG_PM_SLEEP
  567         .driver.pm      = &virtio_pci_pm_ops,
  568 #endif
  569 };
```

It declaims the pci_device_id `0x1af4` of the virtio device is:

```C
  468 /* Qumranet donated their vendor ID for devices 0x1000 thru 0x10FF. */
  469 static const struct pci_device_id virtio_pci_id_table[] = {
  470         { PCI_DEVICE(0x1af4, PCI_ANY_ID) },
  471         { 0 }
  472 };
```

Due to it register a `pci_driver`, and the virtio device will also use the PCI Bus framework to detect it, so it will be detected by the `virtio_pci` driver, and calling the `virtio_pci_probe` function.

If there are `virtio device being detected` by linux kernel. The `virtio_pci_probe` will be called:

(**One more thing should be say: When the virtio_pci driver loaded, it will call driver_register, and this will find these virtio devices and call virtio_pci_probe for these devices.**)

```C
487 static int virtio_pci_probe(struct pci_dev *pci_dev,
  488                         ¦   const struct pci_device_id *id)
  489 {
			  struct virtio_pci_device *vp_dev;
    
  493         /* allocate our structure and fill it out */
  494         vp_dev = kzalloc(sizeof(struct virtio_pci_device), GFP_KERNEL);
  495         if (!vp_dev)
  496                 return -ENOMEM;
  497
  498         pci_set_drvdata(pci_dev, vp_dev);
    
    505         /* enable the device */
  506         rc = pci_enable_device(pci_dev);
    
    virtio_pci_legacy_probe(vp_dev);
    
    register_virtio_device(&vp_dev->vdev);
}
```

In `virtio_pci_probe` function, it first allocates the `virtio_pci_device` to denote a virtio device which use the PCI bus protocol. And then call `virtio_pci_legacy_probe` to do some initialization work. There has a `modern` pci, but the legacy is simple. So let's take a look at the legacy mode.

```C
/* the PCI probing function */
int virtio_pci_legacy_probe(struct virtio_pci_device *vp_dev)
{
	struct virtio_pci_legacy_device *ldev = &vp_dev->ldev;
	struct pci_dev *pci_dev = vp_dev->pci_dev;
	int rc;

	ldev->pci_dev = pci_dev;

	rc = vp_legacy_probe(ldev);
	if (rc)
		return rc;

	vp_dev->isr = ldev->isr;
	vp_dev->vdev.id = ldev->id;

    // Set the virtio_config_ops to virtio_pci_config_opos.
    // Due to different protocol use different config, so need this ops.
	vp_dev->vdev.config = &virtio_pci_config_ops;

	vp_dev->config_vector = vp_config_vector;
	vp_dev->setup_vq = setup_vq;
	vp_dev->del_vq = del_vq;

	return 0;
}
```

```C
int vp_legacy_probe(struct virtio_pci_legacy_device *ldev)
{
	struct pci_dev *pci_dev = ldev->pci_dev;
	int rc;

	/* We only own devices >= 0x1000 and <= 0x103f: leave the rest. */
	if (pci_dev->device < 0x1000 || pci_dev->device > 0x103f)
		return -ENODEV;

	if (pci_dev->revision != VIRTIO_PCI_ABI_VERSION)
		return -ENODEV;

	rc = pci_request_region(pci_dev, 0, "virtio-pci-legacy");

    // Store the virtual address to access the bar[0] of the virtio device.
	ldev->ioaddr = pci_iomap(pci_dev, 0, 0);

	ldev->isr = ldev->ioaddr + VIRTIO_PCI_ISR;

	ldev->id.vendor = pci_dev->subsystem_vendor;
	ldev->id.device = pci_dev->subsystem_device;

	return 0;
err_iomap:
	pci_release_region(pci_dev, 0);
	return rc;
}
```

The `virtio_pci_config_ops` is very important, it will be called many times in the later code. So we can have a glace at here:

```C
// driver/virtio/virtio_pci_modern.c
478 static const struct virtio_config_ops virtio_pci_config_ops = {
  479         .get            = vp_get,
  480         .set            = vp_set,
  481         .generation     = vp_generation,
  482         .get_status     = vp_get_status,
  483         .set_status     = vp_set_status,
  484         .reset          = vp_reset,
  485         .find_vqs       = vp_modern_find_vqs,
  486         .del_vqs        = vp_del_vqs,
  487         .get_features   = vp_get_features,
  488         .finalize_features = vp_finalize_features,
  489         .bus_name       = vp_bus_name,
  490         .set_vq_affinity = vp_set_vq_affinity,
  491 };
```

And the last, the function `register_virtio_device(&vp_dev->vdev)` has been called.

### register_virtio_device

```C
int register_virtio_device(struct virtio_device *dev)
{
    dev->dev.bus = &virtio_bus;
    device_initialize(&dev->dev);
    
    /* Assign a unique device index and hence name. */
	err = ida_alloc(&virtio_index_ida, GFP_KERNEL);
	if (err < 0)
		goto out;

	dev->index = err;
    // Set the device name, use the unique id allocated from xarray.
	err = dev_set_name(&dev->dev, "virtio%u", dev->index);
    
    /* We always start by resetting the device, in case a previous
     * driver messed it up.  This also tests that code path a little. */
    dev->config->reset(dev);

    /* Acknowledge that we've seen the device. */
    add_status(dev, VIRTIO_CONFIG_S_ACKNOWLEDGE);

    /* device_register() causes the bus infrastructure to look for a
     * matching driver. */
    err = device_register(&dev->dev);
}
```

This bind the virtio_device to the virtio_bus, so after the `device_register` is called, the `virtio_dev_probe` in the `virtio_bus` will be called. And we can see it in the below.

Why when `device_register()` being called, the `device` will be registered on the `virtio_bus`, and the `virtio_bus` is integrated to the entire Linux Bus System. So the `device_register()` will trigger the match process which is iterate all of the driver on the `virtio_bus`, and if a driver can drive this device, the `bus->probe` function will be called. At here, it's the `virtio_dev_probe` function.

The function `call_driver_probe` below showed the process.

```C
// drivers/base/dd.c
553 static int call_driver_probe(struct device *dev, struct device_driver *drv)
   554 {
   555         int ret = 0;
   556
   557         if (dev->bus->probe)
   558                 ret = dev->bus->probe(dev);
   559         else if (drv->probe)
   560                 ret = drv->probe(dev);
   561
   562         switch (ret) {
   563         case 0:
   564                 break;
   565         case -EPROBE_DEFER:
   566                 /* Driver requested deferred probing */
   567                 dev_dbg(dev, "Driver %s requests probe deferral\n", drv->name);
   568                 break;
   569         case -ENODEV:
   570         case -ENXIO:
   571                 pr_debug("%s: probe of %s rejects match %d\n",
   572                         ¦drv->name, dev_name(dev), ret);
   573                 break;
   574         default:
   575                 /* driver matched but the probe failed */
   576                 pr_warn("%s: probe of %s failed with error %d\n",
   577                         drv->name, dev_name(dev), ret);
   578                 break;
   579         }
   580
   581         return ret;
   582 }
```

### Virtio Bus probe

As we see on the above, The `device_register()` function will trigger the `bus-probe` being called, let's see them.

When virtio_init, it register the `virtio_bus` to the system.

```C
static int virtio_init(void)
{
    bus_register(&virtio_bus);
}
```

The definition of `virtio_bus` is:

```C
static struct bus_type virtio_bus = {
    .name  = "virtio",
    .match = virtio_dev_match,
    .dev_groups = virtio_dev_groups,
    .uevent = virtio_uevent,
    .probe = virtio_dev_probe,
    .remove = virtio_dev_remove,
};
```

This will has bus type called virtio bus, and it all of the virtio devices will be managed by the virtio bus.

First we look at the `virtio_dev_match()` function, this function will find the matched driver for a virtio device. When the `device_register()`, it will iterate all the driver on the `virtio bus` and call match function which is `virtio_dev_match()` to look if a driver can drive a device.

```C
/* This looks through all the IDs a driver claims to support.  If any of them
 * match, we return 1 and the kernel will call virtio_dev_probe(). */
static int virtio_dev_match(struct device *_dv, struct device_driver *_dr)
{
    unsigned int i;
    struct virtio_device *dev = dev_to_virtio(_dv);
    const struct virtio_device_id *ids;

    ids = drv_to_virtio(_dr)->id_table;
    for (i = 0; ids[i].device; i++)
        if (virtio_id_match(dev, &ids[i]))
            return 1;
    return 0;
}
```

If the driver can take over the device, the kernel will call `virtio_dev_probe()`.

```C
int virtio_dev_probe(struct device *_d)
{
    struct virtio_device *dev = dev_to_virtio(_d);
    struct virtio_driver *drv = drv_to_virtio(dev->dev.driver);
    
    /* We have found a driver for the device. */
    add_status(dev, VIRTIO_CONFIG_S_DRIVER);
    
    // negotiate the feature between driver and device
    /* Figure out what features the device supports. */
	device_features = dev->config->get_features(dev);
    
    /* Figure out what features the driver supports. */
	driver_features = 0;
	for (i = 0; i < drv->feature_table_size; i++) {
		unsigned int f = drv->feature_table[i];
		BUG_ON(f >= 64);
		driver_features |= (1ULL << f);
	}
    
	if (device_features & (1ULL << VIRTIO_F_VERSION_1))
		dev->features = driver_features & device_features;
	else
        // This is the legacy device
		dev->features = driver_features_legacy & device_features;
    
    // Set the feature supported both in device and driver into the deivce.
  	err = dev->config->finalize_features(dev);
    
    // Last call the driver->probe.
    drv->probe(dev);
}
```

The `drv->probe(dev)` will call the function defined by the `virtio_driver`. For example, the `virtio_ballon`:

```C
static struct virtio_driver virtio_balloon_driver = {
	.feature_table = features,
	.feature_table_size = ARRAY_SIZE(features),
	.driver.name =	KBUILD_MODNAME,
	.driver.owner =	THIS_MODULE,
	.id_table =	id_table,
	.validate =	virtballoon_validate,
	.probe =	virtballoon_probe,
	.remove =	virtballoon_remove,
	.config_changed = virtballoon_changed,
};
```

The `virtballoon_probe` will be called. In this function will create the instance which is `virtio_balloon` to manage a balloon device.

```C
static int virtballoon_probe(struct virtio_device *vdev)
{
    struct virtio_balloon *vb;
    
    vdev->priv = vb = kzalloc(sizeof(*vb), GFP_KERNEL);
	if (!vb) {
		err = -ENOMEM;
		goto out;
	}
    
    // Create the virtqueue.
   	err = init_vqs(vb);
    
}
```

The next step is create the virtqueue for the virtio device, which is used to do the communication between device and driver.

```C
static int init_vqs(struct virtio_balloon *vb)
{
	struct virtqueue *vqs[VIRTIO_BALLOON_VQ_MAX];
	vq_callback_t *callbacks[VIRTIO_BALLOON_VQ_MAX];
	const char *names[VIRTIO_BALLOON_VQ_MAX];
	int err;
    
    // At least the balloon will have two virtqueue
   	callbacks[VIRTIO_BALLOON_VQ_INFLATE] = balloon_ack;
	names[VIRTIO_BALLOON_VQ_INFLATE] = "inflate";
	callbacks[VIRTIO_BALLOON_VQ_DEFLATE] = balloon_ack;
	names[VIRTIO_BALLOON_VQ_DEFLATE] = "deflate";
    
    // Will call the vdev->config->find_vqs()
    err = virtio_find_vqs(vb->vdev, VIRTIO_BALLOON_VQ_MAX, vqs,
			      callbacks, names, NULL);
    
   	vb->inflate_vq = vqs[VIRTIO_BALLOON_VQ_INFLATE];
	vb->deflate_vq = vqs[VIRTIO_BALLOON_VQ_DEFLATE];
}
```

We will see the intx way.

```C
/* the config->find_vqs() implementation */
int vp_find_vqs(struct virtio_device *vdev, unsigned int nvqs,
		struct virtqueue *vqs[], vq_callback_t *callbacks[],
		const char * const names[], const bool *ctx,
		struct irq_affinity *desc)
{
	int err;

	/* Try MSI-X with one vector per queue. */
	err = vp_find_vqs_msix(vdev, nvqs, vqs, callbacks, names, true, ctx, desc);
	if (!err)
		return 0;
	/* Fallback: MSI-X with one vector for config, one shared for queues. */
	err = vp_find_vqs_msix(vdev, nvqs, vqs, callbacks, names, false, ctx, desc);
	if (!err)
		return 0;
	/* Finally fall back to regular interrupts. */
	return vp_find_vqs_intx(vdev, nvqs, vqs, callbacks, names, ctx);
}
```

```C
static int vp_find_vqs_intx(struct virtio_device *vdev, unsigned int nvqs,
		struct virtqueue *vqs[], vq_callback_t *callbacks[],
		const char * const names[], const bool *ctx)
{
	struct virtio_pci_device *vp_dev = to_vp_device(vdev);
	int i, err, queue_idx = 0;

	vp_dev->vqs = kcalloc(nvqs, sizeof(*vp_dev->vqs), GFP_KERNEL);
	if (!vp_dev->vqs)
		return -ENOMEM;

	err = request_irq(vp_dev->pci_dev->irq, vp_interrupt, IRQF_SHARED,
			dev_name(&vdev->dev), vp_dev);
	if (err)
		goto out_del_vqs;

	vp_dev->intx_enabled = 1;
	vp_dev->per_vq_vectors = false;
    // For each valid virtqueue, initialize them.
	for (i = 0; i < nvqs; ++i) {
		if (!names[i]) {
			vqs[i] = NULL;
			continue;
		}
		vqs[i] = vp_setup_vq(vdev, queue_idx++, callbacks[i], names[i],
				     ctx ? ctx[i] : false,
				     VIRTIO_MSI_NO_VECTOR);
		if (IS_ERR(vqs[i])) {
			err = PTR_ERR(vqs[i]);
			goto out_del_vqs;
		}
	}

	return 0;
out_del_vqs:
	vp_del_vqs(vdev);
	return err;
}
```

```C
static struct virtqueue *vp_setup_vq(struct virtio_device *vdev, unsigned int index,
				     void (*callback)(struct virtqueue *vq),
				     const char *name,
				     bool ctx,
				     u16 msix_vec)
{
	struct virtio_pci_device *vp_dev = to_vp_device(vdev);
	struct virtio_pci_vq_info *info = kmalloc(sizeof *info, GFP_KERNEL);
	struct virtqueue *vq;
	unsigned long flags;

	/* fill out our structure that represents an active queue */
	if (!info)
		return ERR_PTR(-ENOMEM);

    // Do the actual process
	vq = vp_dev->setup_vq(vp_dev, info, index, callback, name, ctx,
			      msix_vec);
	if (IS_ERR(vq))
		goto out_info;

	info->vq = vq;
	if (callback) {
		spin_lock_irqsave(&vp_dev->lock, flags);
		list_add(&info->node, &vp_dev->virtqueues);
		spin_unlock_irqrestore(&vp_dev->lock, flags);
	} else {
		INIT_LIST_HEAD(&info->node);
	}

	vp_dev->vqs[index] = info;
	return vq;

out_info:
	kfree(info);
	return vq;
}
```

The `vp_dev->setup_vq` is pointed to the `setup_vq` in the `virtio_pci_legacy.c`.

```C
static struct virtqueue *setup_vq(struct virtio_pci_device *vp_dev,
				  struct virtio_pci_vq_info *info,
				  unsigned int index,
				  void (*callback)(struct virtqueue *vq),
				  const char *name,
				  bool ctx,
				  u16 msix_vec)
{
	struct virtqueue *vq;
	u16 num;
	int err;
	u64 q_pfn;

	/* Check if queue is either not available or already active. */
	num = vp_legacy_get_queue_size(&vp_dev->ldev, index);
	if (!num || vp_legacy_get_queue_enable(&vp_dev->ldev, index))
		return ERR_PTR(-ENOENT);

	info->msix_vector = msix_vec;

	/* create the vring */
	vq = vring_create_virtqueue(index, num,
				    VIRTIO_PCI_VRING_ALIGN, &vp_dev->vdev,
				    true, false, ctx,
				    vp_notify, callback, name);
	if (!vq)
		return ERR_PTR(-ENOMEM);

	vq->num_max = num;

	q_pfn = virtqueue_get_desc_addr(vq) >> VIRTIO_PCI_QUEUE_ADDR_SHIFT;
	if (q_pfn >> 32) {
		dev_err(&vp_dev->pci_dev->dev,
			"platform bug: legacy virtio-pci must not be used with RAM above 0x%llxGB\n",
			0x1ULL << (32 + PAGE_SHIFT - 30));
		err = -E2BIG;
		goto out_del_vq;
	}

	/* activate the queue */
	vp_legacy_set_queue_address(&vp_dev->ldev, index, q_pfn);

	vq->priv = (void __force *)vp_dev->ldev.ioaddr + VIRTIO_PCI_QUEUE_NOTIFY;

	if (msix_vec != VIRTIO_MSI_NO_VECTOR) {
		msix_vec = vp_legacy_queue_vector(&vp_dev->ldev, index, msix_vec);
		if (msix_vec == VIRTIO_MSI_NO_VECTOR) {
			err = -EBUSY;
			goto out_deactivate;
		}
	}

	return vq;

out_deactivate:
	vp_legacy_set_queue_address(&vp_dev->ldev, index, 0);
out_del_vq:
	vring_del_virtqueue(vq);
	return ERR_PTR(err);
}
```

Then will do the work to create a virtqueue. The `struct virtqueue` is a descriptor about a virtqueue, it's not a detailed implementation. The actual queue which is used to pass information is defined as `struct vring`. We must understand this structure, due to it's the basic of communication between host and guest.

```C
struct vring {
    unsigned int num;
    
    vring_desc_t *desc;
    
    vring_avail_t *avail;
    
    vring_used_t *used;
};
```

```C
/**
 * struct vring_desc - Virtio ring descriptors,
 * 16 bytes long. These can chain together via @next.
 *
 * @addr: buffer address (guest-physical)
 * @len: buffer length
 * @flags: descriptor flags
 * @next: index of the next descriptor in the chain,
 *        if the VRING_DESC_F_NEXT flag is set. We chain unused
 *        descriptors via this, too.
 */
struct vring_desc {
	__virtio64 addr;
	__virtio32 len;
	__virtio16 flags;
	__virtio16 next;
};
```

```C
struct vring_avail {
	__virtio16 flags;
	__virtio16 idx;
	__virtio16 ring[];
};
```

```C
/* u32 is used here for ids for padding reasons. */
struct vring_used_elem {
	/* Index of start of used descriptor chain. */
	__virtio32 id;
	/* Total length of the descriptor chain which was used (written to) */
	__virtio32 len;
};

struct vring_used {
	__virtio16 flags;
	__virtio16 idx;
	vring_used_elem_t ring[];
};
```

