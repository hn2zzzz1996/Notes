# Boot time memory management

`memblock`是在boot阶段管理内存的分配器。体系结构相关的初始化必须在`setup_arch()`函数中完成，并且在`mem_init()`函数中结束了自己的使命。

## Memblock Overview

当通用的内核内存分配器还未建立起来的时候，Memblock是在early boot阶段管理内存区域的方法。

Memblock将系统内存视为连续区域的集合，下面是这些集合的种类：

* memory - 描述了内核可用的物理内存；这会与系统上实际的物理内存有所区别，例如内存被命令行参数`mem=`限制；
* reserved - 描述了被分配的regions；
* physmem - 描述了boot阶段实际的物理内存，不管可能的限制和内存的插拔；physmem只在部分体系结构上有效。

每一个区域用`struct memblock_region`表示。

```C
struct memblock_region {
  phys_addr_t base;	// base address of the region
  phys_addr_t size;	// size of the region
  enum memblock_flags flags;	// memory region attributes
#ifdef CONFIG_NUMA;
  int nid;	// NUMA node id
#endif;
};
```

Every memory type is described by the [`struct memblock_type`](https://www.kernel.org/doc/html/latest/core-api/boot-time-mm.html#c.memblock_type) which contains an array of memory regions along with the allocator metadata. The “memory” and “reserved” types are nicely wrapped with [`struct memblock`](https://www.kernel.org/doc/html/latest/core-api/boot-time-mm.html#c.memblock).

```C
struct memblock {
  bool bottom_up;	// is bottom up direction?
  phys_addr_t current_limit;	// physical address of the current allocation limit
  struct memblock_type memory;	// usable memory regions
  struct memblock_type reserved;	// reserved memory regions
};
```

```C
struct memblock_type {
  unsigned long cnt;	// number of regions
  unsigned long max;	// size of the allocated array
  phys_addr_t total_size;	// size of all regions
  struct memblock_region *regions;	// array of regions
  char *name;
};
```

memblock的初始化是直接写死的，可以看下它的定义，在`mm/memblock.c`中：

```C
static struct memblock_region memblock_memory_init_regions[INIT_MEMBLOCK_REGIONS] __initdata_memblock;
static struct memblock_region memblock_reserved_init_regions[INIT_MEMBLOCK_RESERVED_REGIONS] __initdata_memblock;
#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
static struct memblock_region memblock_physmem_init_regions[INIT_PHYSMEM_REGIONS];
#endif

struct memblock memblock __initdata_memblock = {
	.memory.regions		= memblock_memory_init_regions,
	.memory.cnt		= 1,	/* empty dummy entry */
	.memory.max		= INIT_MEMBLOCK_REGIONS,
	.memory.name		= "memory",

	.reserved.regions	= memblock_reserved_init_regions,
	.reserved.cnt		= 1,	/* empty dummy entry */
	.reserved.max		= INIT_MEMBLOCK_RESERVED_REGIONS,
	.reserved.name		= "reserved",

	.bottom_up		= false,
	.current_limit		= MEMBLOCK_ALLOC_ANYWHERE,
};
```



