# Paging

* The LME and NXE flags in IA32_EFFR MSR (bit 8 and bit 11, respectively).

分页机制只有在`CR0.PG=1`并且`CR0.PE=1`（开启保护模式）的时候才会生效，有如下4种分页模式：

* If CR4.PAE = 0, `32-bit paging` is used.
* If CR4.PAE = 1 and IA32_EFER.LME = 0, `PAE paging` is used.
* If CR4.PAE = 1, IA32_EFER.LME = 1, and CR4.LA57 = 0, `4-level paging` is used.
* If CR4.PAE = 1, IA32_EFER.LME = 1, and CR4.LA57 = 1, `5-level paging` is used.

> 32-bit paging and PAE paging 只能被用在legacy protected mode (IA32_EFER.LME = 0).  相反，4级和5级的页表映射只能被用在IA-32e模式下(IA32_EFER.LME = 1).

* **canonicality address**: In IA-32e 64-bit mode, the process enforces the upper bits of such address are identical: for 4-level paging, bits 63:47 are identical, it doesn't use bits 63:48. 

## Hirearchical pagin structures

 每一个`paging-structure entry`包含一个物理地址，或者是另一个`paging structure`的物理地址(reference the other paging structure)或者是一个物理页帧的地址(map a page)。

用来转换地址的第一个paging structure的物理地址保存在CR3寄存器中。

对于一个`paging-structure entry`，因为其总是指向一个`4K`对其的地址，所以其低12位可以被作为其他的标志位，例如：

* 如果在翻译过程中线性地址的剩余的比特位大于12个，`paging structure`中的第7位(PS -page size)将会被用于判断当前entry映射的是页帧还是指向下一个`paging structure`。如果bit 7是0，则指向另一个`paging structure`，如果是1，则`maps a page`.
* 如果在线性地址中剩余12位，当前`paging-structure entry`一定maps a page，这时候bit 7 被用作其他的用途。

在不同的页表级别中，`Paging Structure`都有自己的名字：

| Paging structure             | Entry Name |
| ---------------------------- | ---------- |
| PML5 table                   | PML5E      |
| PML4 table                   | PML4E      |
| Page-directory-pointer table | PDPTE      |
| Page directory               | PDE        |
| Page table                   | PTE        |

