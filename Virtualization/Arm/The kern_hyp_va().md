# The kern_hyp_va()

In arm, there has EL1 that running kernel and EL2 that running hypervisor. The EL2 only has the TTBR0_EL2 so it can only access a half of VA space. But the kernel mapping is in the top half of the VA space, that means the hypervisor can't access the kernel liner address directly, but there is indeed that hypervisor need to visit some kernel data like the structure of kvm. So there has a function kern_hyp_va() that translate the kernel virtual address to hyp virtual address.

Let we see how the translation is implemented.



commit 8f2afea51320

kern_hyp_va(), in include/asm/kvm_mmu.h



commit ed57cac83e05f

Introduce EL2 VA randomisation



commit : fd81e6bf3928



37c437532b0126 : first define the kern_hyp_va.



因为arm体系架构的原因，EL2只能映射EL1一半的虚拟内存，EL1的kernel的虚拟地址就超过了EL2的虚拟地址空间，所以arm用了一些特殊的计算让kernel的VA映射到了EL2的VA space，这两个地址之间就产生了一个offset，这个offset加上__pa()宏中定义的偏移，就得到了`hyp_physvirt_offset`，然后用这个地址就可以转换hyp的va和pa了。

简单来说，只要有一个偏移使得EL1中内核的虚拟地址能映射到EL2中的虚拟地址就可以。



In EL2, as the it need to `idmap the trampoline page` which is need to initialize EL2, this page has occupied a range of page. So we should not overlap with it.

If the idmap page is in the bottom half (VA_BITS - 2), we have to use the top half. If the page is in the top half, we have to use the bottom half:

```C
T = __pa_symbol(__hyp_idmap_text_start)
if (T & BIT(VA_BITS - 1))
    HYP_VA_MIN = 0 // idmap in upper half
else
    HYP_VA_MIN = 1 << (VA_BITS - 1)
// Calculate the range that the hyp_va will occupy.
HYP_VA_MAX = HYP_VA_MIN + (1 << (VA_BITS - 1)) - 1
```



使用`__hyp_idmap_text_start`的地址X，`hyp_va_msb` = X^BIT(VA_BITS - 1), 也就是EL2能够映射的地址空间，然后`tab_lsb`的结果为`kernel linear VA`占用的地址空间宽度，如果没有地址的随机化，那么一个EL1的地址A在EL2中的地址就为`(A & va_mask) | hyp_va_msb`. `hyp_va_msb`的地址会被kernel的KSLR随机化一部分，所以也算是随机化。

但是为了更好的保护EL2的代码，在`[V-2, tag_lsb]`这个区间里的bit可以被使用起来作为随机化的bit，`hyp_va_msb |= get_random_long() & GENMASK_ULL(VA_BITS - 2)`，然后新的`hyp_va_msb`作为EL1->EL2线性转换的一个地址。