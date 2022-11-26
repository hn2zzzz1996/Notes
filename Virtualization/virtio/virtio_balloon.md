# Virtio Memory Ballooning

The Virtio Memory Ballooning is a mechanism that the host system can reclaim memory from virtual machines (VM) by telling them to give back part of their memory to the host system.

This is achieved by `inflating the memory balloon` inside the VM, which reduced the memory available to other tasks inside the VM. Which memory pages are given back is the decision of the guest operating system (OS): It just tells the host OS which pages it does no longer need and will no longer access. The host OS then un-maps those pages from the guests and marks them as unavailable for the guest VM. The host system can then use them for other tasks like starting even more VMs or other processes.

If later on the VM need more free memory itself, the host can later on return pages to the guest and shrink the holes. This allows to dynamically adjust the memory available to each VM even while the VMs keep running.

## The Design of Balloon

### The Device part

The balloon is a virtio_pci device, so it will be detected by the virtio pci driver.

It's definition in kvmtool is `struct bln_dev`:

```C
struct bln_dev {
	struct list_head	list;
	struct virtio_device	vdev;

	/* virtio queue */
	struct virt_queue	vqs[NUM_VIRT_QUEUES];
	struct thread_pool__job	jobs[NUM_VIRT_QUEUES];

	struct virtio_balloon_stat stats[VIRTIO_BALLOON_S_NR];
	struct virtio_balloon_stat *cur_stat;
	u32			cur_stat_head;
	u16			stat_count;
	int			stat_waitfd;

	struct virtio_balloon_config config;
};
```

It will calls the `virtio_init` to init the balloon device. It need to pass the `virtio_ops` which is `bln_dev_virtio_ops` to it. And due to it's a `virtio_pci` device, it will calls the `virtio_pci__init` function.

In `virtio_pci__init`, it will allocate the address of the `BAR[0~2]`, and register these three bars to system.



### The Driver Part

The first function being called is the `virtio_pci_probe`, 



In kernel virtio driver, the `virtio_device->config` is needed due to there are two types of `virtio_device`, maybe more in the future. But for initializing the `virtio_device`, it's a common code for both device. But for different device, the `config` way is different. So there need a `virtio_config_ops` to point to different type.

For pci devices: it's `config_ops` will be `virtio_pci_config_ops`.

For mmio devices: it's `config_ops` will be `virtio_mmio_config_ops`.

The `struct virtqueue` is a abstract struct that is used to describe a communication channel. 

```C
// Virtqueue - a queue to register buffers for sending or receiving
struct virtqueue {
    // the chain of virtqueues for this device.
    struct list_head list;
    // The function to call when buffers are consumed (can be NULL)
    void (*callback)(struct virtqueue *vq);
    // The name of this virtqueue
    const char *name;
    // The virtio device this queue was created for.
    struct virtio_device *vdev;
    // The zero-based ordinal number for this queue.
    unsigned int index;
    // Number of elements we expect to be able to fit.
    unsigned int num_free;
    // A pointer for the virtqueue implementation to use.
    void *priv;
};
```

