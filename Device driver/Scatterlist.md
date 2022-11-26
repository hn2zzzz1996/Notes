# Scatterlist

The `Scatterlist` comes from the scene that DMA assumes the data to be transferred is stored contiguously in memoy. But it's difficult to arrange a contiguous physical memory like if the size of buffer is too large or the buffer is in user space.

So it would be nice if DMA operations could work with buffers which split info a number of distinct pieces.

In any capable peripheral device, buffers can be split this way, and this is called `scatter/gather I/O`. The driver starts by filling in an array of `scatterlist` structures:

```C
struct scatterlist {
    struct page *page;		// Where the segment to be found in memory
    unsigned int offset;	// where the data begins within the page
    unsigned int length;	// the length of the segment
    dma_addr_t dma_address;
};
```

One the list has been filled in, the driver calls:

```C
int dma_map_sg(struct device *dev, struct scatterlist *sg, int nents,
               enum dma_data_direction direction);
```

This operation fills the **dma_address** field of each scatterlist entry, and the address can be given to the peripheral. It might do more: physically contiguous pages may be coalesced into a single `scatterlist` entry, or the system's I/O memory management unit might be programmed to make parts (or all) of the list virtually contiguous from the device's point of view.

**The newest scatterlist comes in**

The older scatterlists must fit within `a single page` for various reasons. The restriction puts an upper limit on the number of segments which may be represented. On the i386 architecure, `struct scatterlist` require 20 bytes, which limits a scatterlist to 240 entries. Despite each scatterlist entry points to a full page, the maximum size for a DMA transfer will be about 800KB.

So The new approach [scatterlist chaining patch](http://lwn.net/Articles/234605/) is submitted.  And the struct scatterlist has been changed to:

```C
struct scatterlist {
	unsigned long	page_link;
	unsigned int	offset;
	unsigned int	length;
	dma_addr_t	dma_address;
#ifdef CONFIG_NEED_SG_DMA_LENGTH
	unsigned int	dma_length;
#endif
};
```

Why using `page_link` to replace the `page`?

Notice, the `page_link` is `unsigned long`, and it's the physical address of the `struct page *page`.  Due to page is 4K aligned, so the lowest 12-bit can be used to represent something.

```C
/*
 * Notes on SG table design.
 *
 * We use the unsigned long page_link field in the scatterlist struct to place
 * the page pointer AND encode information about the sg table as well. The two
 * lower bits are reserved for this information.
 *
 * If bit 0 is set (SG_CHAIN), then the page_link contains a pointer to the next sg table list. Otherwise the next entry is at sg + 1.
 *
 * If bit 1 is set (SG_END), then this sg entry is the last element in a list.
 *
 * See sg_next().
 *
 */

#define SG_CHAIN	0x01UL
#define SG_END		0x02UL

#define SG_PAGE_LINK_MASK (SG_CHAIN | SG_END)
```

And to check the two flags:

```C
static inline unsigned int __sg_flags(struct scatterlist *sg)
{
	return sg->page_link & SG_PAGE_LINK_MASK;
}

static inline struct scatterlist *sg_chain_ptr(struct scatterlist *sg)
{
	return (struct scatterlist *)(sg->page_link & ~SG_PAGE_LINK_MASK);
}

static inline bool sg_is_chain(struct scatterlist *sg)
{
	return __sg_flags(sg) & SG_CHAIN;
}

static inline bool sg_is_last(struct scatterlist *sg)
{
	return __sg_flags(sg) & SG_END;
}
```

And In the chaining sg, we should use `sg_next()` to follow the chain links.

```C
/**
 * sg_next - return the next scatterlist entry in a list
 * @sg:		The current sg entry
 *
 * Description:
 *   Usually the next entry will be @sg@ + 1, but if this sg element is part
 *   of a chained scatterlist, it could jump to the start of a new
 *   scatterlist array.
 *
 **/
struct scatterlist *sg_next(struct scatterlist *sg)
{
	if (sg_is_last(sg))
		return NULL;

	sg++;
	if (unlikely(sg_is_chain(sg)))
		sg = sg_chain_ptr(sg);

	return sg;
}
```

## How to use scatterlist

```C

```



## Reference

https://lwn.net/Articles/234617/