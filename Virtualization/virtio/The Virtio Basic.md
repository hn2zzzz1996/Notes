# The Virtio Basic

The Virtio use `Scatter-Gather lists`(SG).

* Lists of buffers described as (base, limit) couples.
* Base: physical address.
* SG implementation be smarter than a simple list, it allows lock-free access.

## VirtQueue (VQ)

The VirtQueue transport abstraction used by virtio. It's a queue of SGs.

The guest (virtio driver) posts (inserts) SGs (buffers) in the VQ.

The host (VMM/hypervisor) consumes the SGs in the VQ. Pushes back SG lists as responses.

There are output SGs (used by the driver to send data) and input SG lists (used by the driver to receive data) .

A virtio device contains one or more VQs.

### VirtQueue Interface

* **add_buf**: used by the guest (virtio driver) to add SGs in the VQ.
  * These are commands sent to the virtio device (to the VMM/hypervisor)
  * A token is associated to each SGs to support out-of-order replies
* **get_buf**: used by the guest (virtio driver) to cleanup SGs.
  * Previously added to the VQ by the guest
  * Already processed by the host
  * Used to receive responses from the virtio device
* **kick**: used by the guest to notify the host that SGs have been added.

## Virtio Source code

Virtio alloc queue有两种方式，一种用dma_api，一种直接分配。

```C
// drivers/virtio/virtio_ring.c
static void *vring_alloc_queue(struct virtio_device *vdev, size_t size,
			      dma_addr_t *dma_handle, gfp_t flag)
{
	if (vring_use_dma_api(vdev)) {
		return dma_alloc_coherent(vdev->dev.parent, size,
					  dma_handle, flag);
	} else {
		void *queue = alloc_pages_exact(PAGE_ALIGN(size), flag);
    }
}
```

往virtqueue中添加desc的调用路径如下：

```C
virtqueue_add_sgs
    -> virtqueue_add
    	-> virtqueue_add_packed/split
    		-> (Follow split) vring_map_one_sg
    			-> dma_map_page
    				-> dma_map_page_attrs
    					-> dma_direct_map_page
    						-> swiotlb_map
```

DMA的分配是如何调用到`dma_set_decrypted`的？

```C
dma_alloc_coherent
    -> dma_alloc_attrs
    	-> dma_direct_alloc
	    	-> dma_set_decrypted
    			-> set_memory_decrypted
```

下面简单看到一些代码，能够看到调用的位置：

```C
void *dma_direct_alloc(struct device *dev, size_t size, dma_addr_t *dma_handle, gfp_t gfp, unsigned long attrs)
{
    /* we always manually zero the memory once we are done */
	page = __dma_direct_alloc_pages(dev, size, gfp & ~__GFP_ZERO, true);
    
    ret = page_address(page);
    // At here.
    if (dma_set_decrypted(dev, ret, size))
        goto out_free_pages;
}
```

## virtio_ring

```C
include/uapi/linux/virtio_ring.h
/* Virtio ring descriptors: 16 bytes.  These can chain together via "next". */
struct vring_desc {
	/* Address (guest-physical). */
	__virtio64 addr;
	/* Length. */
	__virtio32 len;
	/* The flags as indicated above. */
	__virtio16 flags;
	/* We chain unused descriptors via this, too */
	__virtio16 next;
};

struct vring_avail {
	__virtio16 flags;
	__virtio16 idx;
	__virtio16 ring[];
};

/* u32 is used here for ids for padding reasons. */
struct vring_used_elem {
	/* Index of start of used descriptor chain. */
	__virtio32 id;
	/* Total length of the descriptor chain which was used (written to) */
	__virtio32 len;
};

typedef struct vring_used_elem __attribute__((aligned(VRING_USED_ALIGN_SIZE)))
	vring_used_elem_t;

struct vring_used {
	__virtio16 flags;
	__virtio16 idx;
	vring_used_elem_t ring[];
};
```

The `vring` definition is at below:

```C
struct vring {
	unsigned int num;

	vring_desc_t *desc;

	vring_avail_t *avail;

	vring_used_t *used;
};
```

