# 内核是如何初始化vmemmap的

这是在`setup_arch`中所调用的`x86_init.paging.pagetable_init`中进行初始化的，它实际指向的是`native_pagetable_init`，在x86_64系统中，`native_pagetable_init`被定义成了`paging_init`函数。

```C
/arch/x86/mm/init_64.c
void __init paging_init(void)
{
	sparse_init();
}
```

在其中调用了`sparse_init`函数。

```C
/mm/sparse.c
void __init sparse_init(void)
{
    for_each_present_section_nr(pnum_begin + 1, pnum_end) {
		int nid = sparse_early_nid(__nr_to_section(pnum_end));

		if (nid == nid_begin) {
			map_count++;
			continue;
		}
		/* Init node with sections in range [pnum_begin, pnum_end) */
		sparse_init_nid(nid_begin, pnum_begin, pnum_end, map_count);
		nid_begin = nid;
		pnum_begin = pnum_end;
		map_count = 1;
	}
}
```

其中会调用`sparse_init_nid`函数去初始化每一个node上的sparse.

```C
/mm/sparse.c
/*
 * Initialize sparse on a specific node. The node spans [pnum_begin, pnum_end)
 * And number of present sections in this node is map_count.
 */
static void __init sparse_init_nid(int nid, unsigned long pnum_begin,
				   unsigned long pnum_end,
				   unsigned long map_count)
{
    for_each_present_section_nr(pnum_begin, pnum) {
		unsigned long pfn = section_nr_to_pfn(pnum);

		map = __populate_section_memmap(pfn, PAGES_PER_SECTION,
				nid, NULL);
	}
}
```

对于node中存在的每一个section，调用`__populate_section_memmap`去初始化对应的vmemmap。

```C
struct page * __meminit __populate_section_memmap(unsigned long pfn,
		unsigned long nr_pages, int nid, struct vmem_altmap *altmap)
{
	unsigned long start = (unsigned long) pfn_to_page(pfn);
	unsigned long end = start + nr_pages * sizeof(struct page);

	if (WARN_ON_ONCE(!IS_ALIGNED(pfn, PAGES_PER_SUBSECTION) ||
		!IS_ALIGNED(nr_pages, PAGES_PER_SUBSECTION)))
		return NULL;

	if (vmemmap_populate(start, end, nid, altmap))
		return NULL;

	return pfn_to_page(pfn);
}
```

其中又调用了`vmemmap_populate`：

```C
/arch/x86/mm/init_64.c
int __meminit vmemmap_populate(unsigned long start, unsigned long end, int node,
		struct vmem_altmap *altmap)
{
	int err;

	VM_BUG_ON(!IS_ALIGNED(start, PAGE_SIZE));
	VM_BUG_ON(!IS_ALIGNED(end, PAGE_SIZE));

	if (end - start < PAGES_PER_SECTION * sizeof(struct page))
		err = vmemmap_populate_basepages(start, end, node, NULL);
	else if (boot_cpu_has(X86_FEATURE_PSE))
		err = vmemmap_populate_hugepages(start, end, node, altmap);
	else if (altmap) {
		pr_err_once("%s: no cpu support for altmap allocations\n",
				__func__);
		err = -ENOMEM;
	} else
		err = vmemmap_populate_basepages(start, end, node, NULL);
	if (!err)
		sync_global_pgds(start, end - 1);
	return err;
}
```

在其中做了一些检查，随后便调用了`vmemmap_populate_basepages`去在页表中给`vmemmap`对应的区域建立相应的映射。

```C
/mm/sparse-vmemmap.c
int __meminit vmemmap_populate_basepages(unsigned long start, unsigned long end,
					 int node, struct vmem_altmap *altmap)
{
	unsigned long addr = start;
	pgd_t *pgd;
	p4d_t *p4d;
	pud_t *pud;
	pmd_t *pmd;
	pte_t *pte;

	for (; addr < end; addr += PAGE_SIZE) {
		pgd = vmemmap_pgd_populate(addr, node);
		if (!pgd)
			return -ENOMEM;
		p4d = vmemmap_p4d_populate(pgd, addr, node);
		if (!p4d)
			return -ENOMEM;
		pud = vmemmap_pud_populate(p4d, addr, node);
		if (!pud)
			return -ENOMEM;
		pmd = vmemmap_pmd_populate(pud, addr, node);
		if (!pmd)
			return -ENOMEM;
		pte = vmemmap_pte_populate(pmd, addr, node, altmap);
		if (!pte)
			return -ENOMEM;
		vmemmap_verify(pte, node, addr, addr + PAGE_SIZE);
	}

	return 0;
}
```

可以看到，在其中遍历了每一级的页表，最终将相应的vmemmap的page建立了映射。

