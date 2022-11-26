[TOC]

## 如何获取cpu的个数

在`/include/linux/cpumask.h`中有很多的宏，都是与cpu个数相关的，详情可以去里面查看。

一般使用`num_online_cpus()`函数获取当前能使用的cpu个数：

```C
/**
 * num_online_cpus() - Read the number of online CPUs
 *
 * Despite the fact that __num_online_cpus is of type atomic_t, this
 * interface gives only a momentary snapshot and is not protected against
 * concurrent CPU hotplug operations unless invoked from a cpuhp_lock held
 * region.
 */
static inline unsigned int num_online_cpus(void)
{
	return atomic_read(&__num_online_cpus);
}
```

## mmu相关

### is_vmalloc_or_module_addr

定义在`/mm/vmalloc.c`中，用于判断一个地址是否在`vmalloc`或者`module`的地址空间中。

```C
/*
 * ARM, x86-64 and sparc64 put modules in a special place,
 * and fall back on vmalloc() if that fails. Others
 * just put it in the vmalloc space.
 */
int is_vmalloc_or_module_addr(const void *x);
```

### \_\_phys_to_pfn, __pfn_to_phys

定义在`/include/asm-generic/memory_model.h`中：

```C
/*
 * Convert a physical address to a Page Frame Number and back
 */
#define	__phys_to_pfn(paddr)	PHYS_PFN(paddr)
#define	__pfn_to_phys(pfn)	PFN_PHYS(pfn)
```

PHYS_PFN()这个宏则定义在`/include/linux/pfn.h`中：

```C
/*
 * pfn_t: encapsulates a page-frame number that is optionally backed
 * by memmap (struct page).  Whether a pfn_t has a 'struct page'
 * backing is indicated by flags in the high bits of the value.
 */
typedef struct {
	u64 val;
} pfn_t;
#endif

#define PFN_ALIGN(x)	(((unsigned long)(x) + (PAGE_SIZE - 1)) & PAGE_MASK)
#define PFN_UP(x)	(((x) + PAGE_SIZE-1) >> PAGE_SHIFT)
#define PFN_DOWN(x)	((x) >> PAGE_SHIFT)
#define PFN_PHYS(x)	((phys_addr_t)(x) << PAGE_SHIFT)
#define PHYS_PFN(x)	((unsigned long)((x) >> PAGE_SHIFT))
```

## 位操作相关

### Bitfield access macros

一些相关的宏如下：

```C
// include/linux/bits.h
GENMASK(6, 0)	// 用于生成一个连续的位mask，比如这个就得到了0x3f，也就是第0-6位都是1

// 以下宏位于 include/linux/bitfield.h
FIELD_{GET,PREP} // 用于取得一个位掩码当中的第几位，下面看几个例子

#define REG_FIELD_A GENMASK(32, 1)
// FIELD_PREP 是将一个mask作为第一个参数，是一些连续的位，第二个参数说明要mask中的哪些位
a = FIELD_PREP(REG_FIELD_A, 0x1)		// 得到a为0x2
a = FIELD_PREP(REG_FIELD_A, 0x2)		// 得到a为0x4
a = FIELD_PREP(REG_FIELD_A, 0x3)		// 得到a为0x6
    
// FIELD_GET 是从一个结果里面得到是mask中的哪些位
a = FIELD_GET(REG_FILED_A, (1 << 32))	// 得到a为(1 << 31)
```

## 一些编译的宏检查

* IS_ENABLED()

  IS_ENABLED(CONFIG_KVM)，用于检查内核config中某个选项是否打开。

* 
