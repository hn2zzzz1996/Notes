# Enable virtio in pkvm

The virtio 1.1 spec introduce the `VIRTIO_F_ACCESS_PLATFORM(33)` Reserved Feature Bit.

​	This feature indicates that the device can be used on a platform where device access to data in memory is limited and/or translated. And the device can be limited to only access certain memory addresses.

When set, causes a Linux guest to use the DMA API for virtio allocations.

* We can force the use of bounce-buffers by passing `swiotlb=force`
* Allow the host to access the bounce buffer pages
  * Expose SHARE/UNSHARE hypercalls to the guest
  * Hook the `set_memory_{decrypted, encrypted}()` API to share/unshare bounce buffer pages.

`set_memory_decrypted()`这个函数位于`/arch/x86/mm/pat/set_memory.c`中.

## How TDX share memory?

The patch in the mailinglist: https://lore.kernel.org/all/20211009003711.1390019-1-sathyanarayanan.kuppuswamy@linux.intel.com/

In the commit `20f07a044a76a`, Move common memory encryption code to mem_encrypt.c:

SEV and TDX both protect guest memory from host accesses. SEV and TDX both assume that all I/O (real devices and virtio) must be performed to pages **without** protection.

To add this support, AMD SEV forces `force_dma_unencrypted()` to decrypted DMA pages when DMA pages were allocated for I/O.

Introduce a new config option `X86_MEM_ENCRYPT` that can be selected by platforms which use x86 memory encryption features.

```C
+config X86_MEM_ENCRYPT
+       select ARCH_HAS_FORCE_DMA_UNENCRYPTED
+       select DYNAMIC_PHYSICAL_MASK
+       select ARCH_HAS_RESTRICTED_VIRTIO_MEMORY_ACCESS
+       def_bool n
```

> In pkvm, we need select the config.

In `force_dma_unencrypted()`, return true.

## What I do to enable virtio in pkvm

* Disable `CONFIG_AMD_MEM_ENCRYPT`.

## Reference

The arm side implementation: https://android-kvm.googlesource.com/linux/+/refs/heads/topic/virtio