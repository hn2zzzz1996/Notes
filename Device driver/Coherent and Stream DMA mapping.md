# Coherent and Stream DMA mapping

There are two different type of DMA mapping in Linux. And here is the problem, why it has two types and what's the difference between them?

Let's first talk about the Coherent DMA mapping. It's proposed due to the cache coherency issue, and here is an good illustrated from LDD3:

> Let's imagine a CPU equipped with a cache and an external memory that can be accessed directly by devices using DMA. When the CPU accesses location X in the memory, the current value will be stored in the cache. Subsequent operations on X will update the cached copy of X, but not the external memory version of X, assuming a write-back cache. If the cache is not flushed to the memory before the next time a device tries to access X, the device will receive a stale value of X. Similarly, if the cached copy of X is not invalidated when a device writes a new value to the memory, then the CPU will operate on a stale value of X.

And if we want to address this issue, we should use Coherent DMA mapping. And it has two implementations:

1. Hardware-based solution. Some systems provide Cache Coherent Interconnect in the hardware, so can use it to implement the Cache coheret.
2. Software-based solution. Disable cache is a way to implement Coherent.

And for the use scene. The `coherent DMA mappings` can use the former over several transfers, which automatically addresses cache coherency issues. Therefore, allocate `coherent memory` is expensive. And Coherent mapping usually exists for the life of the driver. It provide:

* **Consistent**: since it allocates uncached and unbuffered memory for a device to perform DMA.
* **Synchronous**: since a write by either the device or the CPU can be immediately read by either without worrying about cache coherency.

In contrast, the `streaming DMA mappings` doesn't provide cache coherency. So it has a lot of constrains. It usually unmapped once the DMA transfer completes. And here is it's constraint:

* Mappings need to work with a buffer that has already been allocated.
* Mappings may accept several non-contiguous and scattered buffers.
* `A mapped buffer belongs to the device and not to the CPU anymore`. Before the CPU can use the buffer, it should be unmapped first (after `dma_unmap_single()` or `dma_unmap_sg`). This is for caching purposes.
* For write transactions (CPU to device), the driver should place data in the buffer before the mapping.
* The direction the data should move has be specified, and the data should only be used based on this direction.