# Translate Virtual Address To Physical Address

The followings are some ways to translate a virtual address to the corresponding physical address, if exist.

Step 1. Walk the page tables to get `pte_t *`.

```
// struct mm_struct *mm : The memory descriptor of the current process.
// unsigned long address: The virtual address to be translated.
pgd_t *pgd = pgd_offset(mm, address);
if (!pgd_none(*pgd) && !pgd_bad(*pgd)) {
    pud_t *pud = pud_offset(pgd, address);
    if (!pud_none(*pud) && !pud_bad(*pud)) {
        pmd_t *pmd = pmd_offset(pud, address);
        if (!pmd_none(*pmd) && !pmd_bad(*pmd)) {
            pte_t *pte = pte_offset_map(pmd, address);
            if (!pte_none(*pte)) {
                // now you get a pointer pointing to a pte entry!
            }
            pte_unmap(pte);
        }
    }
}       
```

Step 2.

- A) Calculate the bits directly.

```
// Assume it is on 32-bit machine with 4 KB pages.
unsigned long phys = pte_val(*pte) & 0xfffff000 + address & 0x00000fff;
```

- B) With the help of a page descriptor.

```
struct page *pg = pte_page(*pte);
unsigned long phys = page_to_phys(pg);
```

## Userspace:

`cat /proc/<pid>/maps` to get virtual address intervals.

`xxd /proc/<pid>/pagemap` to get virtual - physical address mappings. [documention](https://www.mjmwired.net/kernel/Documentation/vm/pagemap.txt)

## Others:

- `virt_to_page(unsigned long address)` seems to be a faster way to get `struct page*`, but you have to make sure the return value is valid. ([code](https://elixir.free-electrons.com/linux/v3.9/source/arch/x86/include/asm/page.h#L63))
- `virt_to_phys(volatile void *address)` is only suitable for addresses directly mapped or mapped by kmalloc. ([code](https://elixir.free-electrons.com/linux/v3.9/source/arch/x86/include/asm/io.h#L111))

## Reference:

- Understanding the Linux Kernel 3rd (Daniel P. Bovet, Marco Cesati), Sections 2.5, 8.1, 9.2, etc.
- [page_to_phys](https://elixir.free-electrons.com/linux/v3.9/source/arch/x86/include/asm/io.h#L137) from <asm/io.h>
- How to get the physical address from the logical one in a Linux kernel module? [Stack Overflow](https://stackoverflow.com/questions/6252063/how-to-get-the-physical-address-from-the-logical-one-in-a-linux-kernel-module)
- How to find the physical address of a variable from user-space in Linux? [Stack Overflow](https://stackoverflow.com/questions/2440385/how-to-find-the-physical-address-of-a-variable-from-user-space-in-linux/13949855)