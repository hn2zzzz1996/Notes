# Device, DMA and IOMMU

## DMA and bus address

The virtual memory system (TLB, page tables, etc.) translates virtual
addresses to CPU physical addresses, which are stored as "phys_addr_t" or
"resource_size_t".  The kernel manages device resources like registers as
physical addresses.  These are the addresses in /proc/iomem.  The physical
address is not directly useful to a driver; it must use **ioremap()** to map
the space and produce a virtual address.

I/O devices use a third kind of address: a "**bus address**".  If a device has
registers at an MMIO address, or if it performs DMA to read or write system
memory, the addresses used by the device are bus addresses.  In some
systems, bus addresses are identical to CPU physical addresses, but in
general they are not.  IOMMUs and host bridges can produce arbitrary
mappings between physical and bus addresses.

From a device's point of view, DMA uses the `bus address space`.

Here is an example:

```
			   CPU                  CPU                  Bus
             Virtual              Physical             Address
             Address              Address               Space
              Space                Space

            +-------+             +------+             +------+
            |       |             |MMIO  |   Offset    |      |
            |       |  Virtual    |Space |   applied   |      |
          C +-------+ --------> B +------+ ----------> +------+ A
            |       |  mapping    |      |   by host   |      |
  +-----+   |       |             |      |   bridge    |      |   +--------+
  |     |   |       |             +------+             |      |   |        |
  | CPU |   |       |             | RAM  |             |      |   | Device |
  |     |   |       |             |      |             |      |   |        |
  +-----+   +-------+             +------+             +------+   +--------+
            |       |  Virtual    |Buffer|   Mapping   |      |
          X +-------+ --------> Y +------+ <---------- +------+ Z
            |       |  mapping    | RAM  |   by IOMMU
            |       |             |      |
            |       |             |      |
            +-------+             +------+
```

**Discover the device and know it's MMIO address space**:

During the enumeration process, the kernel learns about I/O devices and their `MMIO space` and the host bridges that connect them to the system.  For example, if a PCI device has a BAR, the kernel reads the bus address (A) from the BAR and converts it to a CPU physical address (B).  The address B is stored in a struct resource and usually exposed via `/proc/iomem`.  When a driver claims a device, it typically uses `ioremap()` to map physical address B at a virtual address (C).  It can then use, e.g., ioread32(C), to access the device registers at bus address A.

The translation from `physical address` to `bus address` is produced by the host bridge.

**If a device has DMA**:

If a device supports DMA, and the driver sets up a buffer using kmalloc() which return a virtual address (X). The virtual memory system maps X to a physical address (Y) in system RAM. The driver can use virtual address X to access the buffer. `But how device access the physical address (Y)`? In some simple systems, the device can do DMA directly to physical address Y. But in many others, there is IOMMU hardware that translates DMA addresses to physical addresses, e.g., it translates Z to Y.  This is part of the reason for the DMA API: the driver can give a virtual address X to an interface like `dma_map_single()`, which sets up any required IOMMU mapping and returns the DMA address Z.  The driver then tells the device to do DMA to Z, and the IOMMU maps it to the buffer at address Y in system RAM.

The driver has to take into account that DMA addresses should be mapped only for the time they are actually used and unmapped after the DMA transfer.

## DMA API Attention

DMA API works with bus independent. So you should use the DMA API rather than the bus-specific DMA API. Use the `dma_map_*()` interfaces rather than the `pci_map_*()` interfaces.

First should `#include <linux/dma-mapping.h>` in your driver, it provides the definition of `dma_addr_t`. 

```
dma_addr_t:
This type can hold any valid DMA address for the platform and should be used everywhere you hold a DMA address returned from the DMA mapping functions.
```

## DMA addressing capabilities

By default, the kernel assumes that your device can address `32-bit` of DMA addressing. For a 64-bit capable device, this needs to be increased, and for a device with limitations, it needs to be decreased.

For correct operation, you must set the DMA mask to inform the kernel about your devices DMA addressing capabilities.

This is performed via a call to dma_set_mask_and_coherent():

```C
int dma_set_mask_and_coherent(struct device *dev, u64 mask);
```

which will set the mask for both streaming and coherent APIs together. If you have some special requirements, then the following two separate calls can be used instead:



## IOMMU

IOMMU is the input-output memory management unit (IOMMU) is a MMU that connects a `direct memory access capable` I/O bus to the memory. And it translates the bus address (which the device see the address space) to the physical address.

IOMMU provides two main functions: `I/O Translation` and `Device Isolation`.

* I/O Translation: bus address -> physical address.
* Device Isolation: limit the ability of devices to access specific regions of memory.
  * A sample solution is: for a device in a domain, using a bit-vectored table to describe each page's access rights in that domain. It forces a permission check of all device DMAs indicating whether devices in that domain are allowed to access the corresponding physical page.

### Linux IOMMU support and the DMA mapping API

There are three DMA API implementations: 

* NOMMU: The system has neither a hardware IOMMU nor software emulation. Using physical address as DMA address.
* SWIOTLB: A software implementation of an IOMMU's translation function. It provides translation using `bounce buffering`. 

## Reference

https://www.kernel.org/doc/Documentation/DMA-API-HOWTO.txt