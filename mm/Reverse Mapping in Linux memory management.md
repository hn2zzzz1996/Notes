# Reverse Mapping in Linux memory management

As the name means, Reverse Mapping record the mapping information from physical page to virtual page.

Back to Linux version 2.4. The Linux had no mechanism for translating physical address to user virtual address. In that version, it can only walk through each process's mappings and select pages to unmap. Only after all a page's mappings were removed, then it could be selected for pageout.

## PTE Chains

## Object-based reverse mapping