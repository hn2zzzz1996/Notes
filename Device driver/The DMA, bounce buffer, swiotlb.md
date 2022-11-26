# The DMA, bounce buffer, swiotlb

What is bounce buffer? For some device, it can't access all of the memory space, so the os setup a bounce buffer in the low memory that device can access. When device write to the bounce buffer, the os then copy the buffer to the high address, thus the bounce means.

```C
// kernel/dma/swiotlb.c
void  __init
swiotlb_init(int verbose)
{
    // Default 64MB
    size_t bytes = PAGE_ALIGN(default_nslabs << IO_TLB_SHIFT);
    
    /* Get IO TLB memory from the low pages */
	tlb = memblock_alloc_low(bytes, PAGE_SIZE);
	if (!tlb)
		goto fail;
	if (swiotlb_init_with_tbl(tlb, default_nslabs, verbose))
		goto fail_free_mem;
	return;
}
```

The backtrace is showed below:

```C
x86_64_start_kernel
    -> start_kernel
    	-> mm_init
    		-> mem_init
    			-> pci_iommu_alloc
    				-> pci_swiotlb_init
    					-> swiotlb_init
```

The `swiotlb_init` first allocate memory from memblock, then call `swiotlb_init_with_tlb`:

```C
int __init swiotlb_init_with_tbl(char *tlb, unsigned long nslabs, int verbose)
{
	struct io_tlb_mem *mem = &io_tlb_default_mem;
	size_t alloc_size;
    
    alloc_size = PAGE_ALIGN(array_size(sizeof(*mem->slots), nslabs));
	mem->slots = memblock_alloc(alloc_size, PAGE_SIZE);
    
    swiotlb_init_io_tlb_mem(mem, __pa(tlb), nslabs, false);
}

static void swiotlb_init_io_tlb_mem(struct io_tlb_mem *mem, phys_addr_t start,
				    unsigned long nslabs, bool late_alloc)
{
	void *vaddr = phys_to_virt(start);
	unsigned long bytes = nslabs << IO_TLB_SHIFT, i;

	mem->nslabs = nslabs;
	mem->start = start;
	mem->end = mem->start + bytes;
	mem->index = 0;
	mem->late_alloc = late_alloc;

	if (swiotlb_force == SWIOTLB_FORCE)
		mem->force_bounce = true;

	spin_lock_init(&mem->lock);
	for (i = 0; i < mem->nslabs; i++) {
		mem->slots[i].list = IO_TLB_SEGSIZE - io_tlb_offset(i);
		mem->slots[i].orig_addr = INVALID_PHYS_ADDR;
		mem->slots[i].alloc_size = 0;
	}
    
    memset(vaddr, 0, bytes);
	mem->vaddr = vaddr;
	return;
}
```



