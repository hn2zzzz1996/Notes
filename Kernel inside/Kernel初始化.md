# Kernel初始化

```C
#define __START_KERNEL    (__START_KERNEL_map + __PHYSICAL_START)

#define __PHYSICAL_START  ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN)
```

* __START_KERNEL - `0xffffffff81000000`，the virtual address of the Linux kernel;
* __PHYSICAL_START - `0x1000000`, the physical address of the Linux kernel;
* __START_KERNEL_map - `0xffffffff80000000`;

Kernel被运行的第一个函数是`startup_64`，其位于`arch/x86/kernel/head_64.S`中，并且跳转到这里的时候，其物理地址和虚拟地址都是`0x1000000`，因为这个时候内核的虚拟地址空间还没有建立，只有虚拟和物理地址相同的映射.

`fixup_pointer`这个函数是用来修正地址的，因为此时虚拟地址还没有建立，所以引用变量的都是虚拟地址，与实际的地址是不一样的（此时的地址还是一一映射的），所以用计算偏移的方式得到实际的地址。

```C
void fixup_pointer(void *ptr, unsigned long physaddr)
{
    // ptr - _text 得到偏移，加上physaddr得到实际的地址
    return ptr - (void *)_text + (void *)physaddr;
}
```



```
 physaddr + level3_kernel_pgt - _text
```