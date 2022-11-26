gdt表结构定义如下所示：

```C
// arch/x86/include/asm/desc.h
struct gdt_page {
	struct desc_struct gdt[GDT_ENTRIES];
} __attribute__((aligned(PAGE_SIZE)));
```

实际定义在，在这里利用宏进行了表项的初始化：

```C
// arch/x86/kernel/cpu/common.c
DEFINE_PER_CPU_PAGE_ALIGNED(struct gdt_page, gdt_page) = { .gdt = {
	/*
	 * We need valid kernel segments for data and code in long mode too
	 * IRET will check the segment types  kkeil 2000/10/28
	 * Also sysret mandates a special GDT layout
	 *
	 * TLS descriptors are currently at a different place compared to i386.
	 * Hopefully nobody expects them at a fixed place (Wine?)
	 */
	[GDT_ENTRY_KERNEL32_CS]		= GDT_ENTRY_INIT(0xc09b, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_CS]		= GDT_ENTRY_INIT(0xa09b, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_DS]		= GDT_ENTRY_INIT(0xc093, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER32_CS]	= GDT_ENTRY_INIT(0xc0fb, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_DS]	= GDT_ENTRY_INIT(0xc0f3, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_CS]	= GDT_ENTRY_INIT(0xa0fb, 0, 0xfffff),
}
```

GDT的初始化宏定义如下，也就是拆解了相应的部分：

```C
arch/x86/include/asm/desc_defs.h
struct desc_struct {
        u16     limit0;
        u16     base0;
        u16     base1: 8, type: 4, s: 1, dpl: 2, p: 1;
        u16     limit1: 4, avl: 1, l: 1, d: 1, g: 1, base2: 8;
} __attribute__((packed));

#define GDT_ENTRY_INIT(flags, base, limit)                      \
          {                                                       \
                  .limit0         = (u16) (limit),                \
                  .limit1         = ((limit) >> 16) & 0x0F,       \
                  .base0          = (u16) (base),                 \
                  .base1          = ((base) >> 16) & 0xFF,        \
                  .base2          = ((base) >> 24) & 0xFF,        \
                  .type           = (flags & 0x0f),               \
                  .s              = (flags >> 4) & 0x01,          \
                  .dpl            = (flags >> 5) & 0x03,          \
                  .p              = (flags >> 7) & 0x01,          \
                  .avl            = (flags >> 12) & 0x01,         \
                  .l              = (flags >> 13) & 0x01,         \
                  .d              = (flags >> 14) & 0x01,         \
                  .g              = (flags >> 15) & 0x01,         \
          }
```

