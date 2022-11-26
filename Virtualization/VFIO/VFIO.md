# VFIO (Virtual Function I/O)

The VFIO driver is an IOMMU/device agnostic framework for exposing direct device access to userspace, in a secure, IOMMU protected environment. This allows safe, non-privileged, userspace drivers.



Why VFIO? Virtual machines often make use of direct device access ("device assignment") when configured for the highest possible I/O performance. From a device and host perspective, this simply `turns the VM into a userspace driver`, with the benefits of significantly reduces latency, higher bandwidth, and direct use of bare-metal device drivers.



The first commit is [VFIO core framework](https://lwn.net/Articles/473975/). This is proposed by `Alex Williamson`, and it's aimed to replace the UIO interface, which is also a driver to make userspace directly access the device.

The initial code can be found at: https://github.com/awilliam/linux-vfio/tree/vfio-next-20111221

## Groups, Devices and IOMMUs

The userspace drivers want to manipulate individual devices and setting up mappings in IOMMU for those devices. But, the IOMMU doesn't have the granularity to track mappings for individual device. The devices are not always independent with respect to the IOMMU. Translation setup for one device can be used by another device in these scenarios.



The IOMMU API exposes these relationships by identifying an "IOMMU group" for these dependent devices.  Devices on the same bus with the same IOMMU group are not isolated from each other with respect to DMA mappings.  For userspace usage, we must grant ownership of a group, which may contain one or more devices.



These `groups` therefore become a `fundamental component` of VFIO and the `working unit` we use for exposing devices and granting permissions to userspace.



## VFIO bus driver

Bus driver, such as PCI, have three jobs:

1. Add/remove devices from vfio
2. Provide vfio_device_ops for device access
3. Device binding and unbinding



## The design of the VFIO

First it define the `struct vfio_device` that is used to define the device that has been found by the vfio driver. And these vfio device will belong to some group, and VFIO define it as `struct vfio_group`.

```C
struct vfio_device {
    struct device		*dev;
    // The ops to operate the vfio_device.
    const struct vfio_device_ops *ops;
    struct vfio_group	*group;
    // link to vfio_group->device_list.
    struct list_head	device_next;
}
```

A device can only belong to one group.

```C
struct vfio_group {
    // One group owns a minor number which is pre-allocated by the vfio driver initialization. So every vfio_group will create a device use the devt.
    dev_t		devt;
    // Get from the iommu_device_group(), so it's the device belong to which group in the iommu level.
    unsigned int groupid;
    struct bus_type		*bus;
    // Links all the devices belongs to this group.
    struct list_head	device_list;
}
```

At the vfio_pci initalization, it will scan all the pci devices and record them as `vfio_device` and build the relationship that which device belongs to which group, and which group include how many devices.

And in the `__vfio_create_group`, it will create a device for every group, thus it will appear into the `/dev` directory. And this device is used by the userspace, The user need to first open the `group`, and then it can do the operation to the device that is belong to the group.

Before use the VFIO, we should unbind the devices which we want to use at userspace from it's native driver, and then we should bind the device to the vfio_pci driver.

Once the device bind to the vfio driver. The `vfio_pci_probe` will be called, in there will do the bind process. And at here, a new `struct vfio_pci_device` has been defined, it is aimed to record the specific pci information of the `vfio_device`.

```C
struct vfio_pci_device {
    // Point to the pci_dev pointer that passed from the probe() function.
    struct pci_dev		*pdev;
    void __iomem		*barmap[PCI_STD_RESOURCE_END + 1];
    u8					*pci_config_map;
    // Others specific to the pci bus.
}
```

The relationship between `pci_device`, `device`, `vfio_device` and `vfio_pci_device` is at below:

```C
pci_device->dev  ---->   device
device->p->driver_data  ---->  vfio_device
vfio_device->device_data  ---->  vfio_pci_device
```

### Open the group

#### Create iommu

Once the user open a group, the vfio will create a `struct vfio_iommu` for it. A `vfio_iommu` can include many group, but one group can only belong to one `vfio_iommu`.

```C
struct vfio_iommu {
    // The acture physical iommu_domain it related to
    struct iommu_domain		*domain;
    struct bus_type			*bus;
    // If it has been used by a user level device driver. Bind the mm.
    struct mm_struct		*mm;
    // Links all the group that uses the iommu.
    struct list_head		group_list;
    // Links all the mappings been build in the iommu_domain.
    struct list_head		dma_list;
	int 					refcnt;
};
```

The `group->iommu` will be set to the iommu it belongs to. And the `group->iommu_next` will be linked to the `iommu->group_list`.

### group_ioctl: VFIO_GROUP_GET_INFO

This will detect whether the group is viable. A group is viable means all of the device in the group has been bind to the `vfio_pci` driver. And all is ok.

So this ioctl will detect whether the group is viable, and return the result to the userspace.

### group_ioctl: VFIO_GROUP_GET_IOMMU_FD

This will get a iommu_fd that is used to operate the vfio_iommu. 

And will build the relationship between the `vfio_iommu` and the `iommu_domain`. In the `__vfio_iommu_open` function, it calls the `iommu_domain_alloc()` that gets a new `iommu_domain`, and assign it to the `vfio_iommu->domain`.  And it will attach all the devices in the group to the `iommu_domain` using the `iommu_attach_device` function. After attached success, the `vfio_device->attached` will be set true.

And the `vfio_iommu->mm` will be set to the `current->mm`. Means the the iommu has been used by the current process. And other process can't use it.

And to do the operation to the iommu fd, the `vfio_iommu_fops` will be used.

### iommu_ioctl: VFIO_IOMMU_GET_INFO

This will get the `struct vfio_iommu_info`:

```C
struct vfio_iommu_info {
    __u32	argsz;
    __u32   flags;
    __u64   iova_max;       /* Maximum IOVA address */
    __u64   iova_min;       /* Minimum IOVA address */
    __u64   pgsize_bitmap;  /* Bitmap of supported page sizes */
};
```

### iommu_ioctl: VFIO_IOMMU_MAP_DMA

This ioctl will pass a `struct vfio_dma_map`, it includes the information that the user want to build the mapping in the iommu. Thus the device can be directly control by a user level driver.

```C
// Map process virtual addresses to IO virtual addresses.
struct vfio_dma_map {
    __u32	argsz;
    __u32	flags;
    __u64	vaddr;	// Process virtual address, which has mapped to a physical memory, the physical memory will be retrived and used to build mapping in IOMMU, the mapping will be iova -> hpa (The hpa is get from get_user_page(vaddr))
    __u64	iova;	// IO virtual address, which will be used by the device when DMA
    __u64	size;	// Size of mapping (bytes)
}
```

The user fills the struct and passes to the vfio driver. The vfio will build mappings in IOMMU.

The `struct vfio_dma_map_entry` is been used for record every IOMMU mapping which been build for the iommu_domain.

```C
struct vfio_dma_map_entry {
    struct list_head	list;	// linked to the vfio_iommu->dma_list
    dma_addr_t			iova;	// Device address
    unsigned long		vaddr;	// Process virtual addr
    long				npage;	// Number of pages
    int					prot;	// IOMMU_READ/WRITE
}
```

For each `dma_map_entry`, the vfio will build the IOMMU mappings for it by using the `iommu_map` function.

The implementation about create mapping can be found in the `__vfio_dma_map` function. It will first use the `get_user_page(vaddr)` to get the physical page and pin it, and then use the `iommu_map` to create mapping in IOMMU.

For the passthrough device, it must create the IOMMU mapping for every memory_slot (which is the mapping from gpa->hpa). So the device can access the gpa when do DMA operation.

### group_ioctl: VFIO_GROUP_GET_DEVICE_FD

The userspace need to pass a string which is the device name like "0000:06:0d.0", and the vfio driver will use the `vfio_device->ops->match` operation to compare the user passed device name with the device name. Notice, the `vfio_device->ops` is the `vfio_pci_ops`.

After matched a device, the `vfio_pci_enable` will be called to enable the `pci_device`. This process includes read the pci configuration space and others.

Then we will create a new fd which is for the device, and it's ops is `vfio_device_fops`.

Here we talk a little about the connection between the `vfio_device` and the `vfio_pci_device`. Why split it into two? Because the `vfio_device` is represent a general device, it only know it's a device, and the device in which group, the `vfio_device` it self doesn't know any about it's implementation. So it doesn't know how to operate the device, so it has an ops which point to a specific device operation (like pci device operation), and a pointer point to the `vfio_pci_device` which is the actually implementation about the device, only the `vfio_pci_device` know how to operate the device and know the device inner state. So the `vfio_device` can also be another device, like USB device without modify the struct, just set the suitable ops and the specific pointer is ok.

### device_ioctl: VFIO_DEVICE_GET_INFO

The ioctl to vfio device will call the function that registered in the `vfio_device_fops`, in the implementation of the `vfio_device_fops->unlocked_ioctl`, it is called the `device->ops->ioctl`, that's the `vfio_pci_ops` we registered earlier in the device.

```C
static long vfio_device_unl_ioctl(struct file *filep,
                                  unsigned int cmd, unsigned long arg)
{
    struct vfio_device *device = filep->private_data;
    
    // The device_data is the vfio_pci_device.
    return device->ops->ioctl(device->device_data, cmd, arg);
}
```

And the `ops->ioctl` is pointed to the `vfio_pci_ioctl`, and we deals with the `VFIO_DEVICE_GET_INFO` ioctl at there.

The information will be passed by `struct vfio_device_info` between the userspace and vfio driver. It's defined as follow:

```C
struct vfio_device_info {
    __u32	argsz;
    __u32	flags;
    __u32	num_regions;	/* Max region index + 1 */
    __u32	num_irqs;		/* Max IRQ index + 1 */
}
```

### device_ioctl: VFIO_DEVICE_GET_REGION_INFO

This ioctl use the `struct vfio_region_info` to pass information. It's definition is at below:

```C
struct vfio_region_info {
    __u32	argsz;
    __u32	flags;
#define VFIO_REGION_INFO_FLAG_MMAP	(1 << 0) /* Region supports mmap */
#define VFIO_REGION_INFO_FLAG_RO	(1 << 1) /* Region is read-only */
    __u32	index;	/* Region index */
    __u32	resv;	/* Reserved for alignment */
    __u64	size;	/* Region size (bytes) */
    __u64	offset;	/* Region offset from the start of device fd */
}
```

The user need to provide the index to get the region information.

If we want to get a region which is a bar, and it's a MMIO region. The returned region_info will have the size and offset being set, and the size is the bar region's size, the offset is a fake file offset from the start of device fd.

So after user program get the region_info, it can allocate a MMIO region in the guest address space (gpa) use the `info.size` (By directly find a unused MMIO memory region in guest address space). 

After get the bar region information, the user need the map the bar to the userspace, thus it can provide the userspace driver (at here is guest) to access the bar region from userspace.

So user program use the mmap to map the bar region into userspace:

```C
user_base_addr = mmap(NULL, info.size, PROT_READ | PROT_WRITE, MAP_SHARED, device_fd, 						info.offset);
```

 This call will get into the `vfio_pci_mmap` function, which is the `mmap` callback on the device_fd. And in this function, the vfio_pci driver will get the actual physical address of the bar in host address space (hpa), and map it to a userspace address space using the `remap_pfn_range`, which will create mapping from userspace addr to the bar physical address.

So now the user program get a virtual address `user_base_addr` that is mapped to the device bar region. Until now, there still two things need to do:

1. Setting the IOMMU mappings for the vfio_device, due to the guest will use gpa to set to device to do dma operation. So the user program should use the `iommu_ioctl: VFIO_IOMMU_MAP_DMA` to register all of the memory_slot (which must maps to a real physical memory, not include the memory_slot used for MMIO bar region) to the IOMMU mappings.
2. Setting the bar region as a memory_slot (The useraddr is `user_base_addr` above, and gpa is allocated from gpa space.) (it has the mapping from virtual address -> bar MMIO physical address, be attention, the physical address is in the MMIO space, it's not a real physical memory) to the kvm, so when kvm has a EPT violation, kvm can use the memory_slot to set the EPT, so the vfio_device MMIO region can be directly accessed by the guest. That's the passthrough.

### device_ioctl: VFIO_DEVICE_GET_IRQ_INFO

vfio driver using `struct vfio_irq_info` to record a irq information. It's show below:

```C
struct vfio_irq_info {
    __u32	argsz;
    __u32	flags;
    __u32	index;	// IRQ index, user passed, can be set to VFIO_PCI_INTX_IRQ_INDEX or VFIO_PCI_MSI_IRQ_INDEX
    __u32	count;	// Number of IRQs within this index
}
```

The `irq_info.count` is calculated from the `vfio_pci_get_irq_count` function, in this function, it based on read the pci configuration space to get the number of IRQs within the index.

