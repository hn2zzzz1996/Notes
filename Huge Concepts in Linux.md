# Huge Concepts in Linux

Struct Page Key Fields:

* Page Flags
  * PG_locked, PG_dirty, PG_active, PG_uptodate, PG_head, PG_hwpoison...
  * Reference Count
  * Map Count
* Commonly 64 bytes in size



## Huge Page APIs in Linux

* Transparent Huge Pages - THP
  * Transparent to the application
  * Automatic with some control
* Hugetlbfs
  * Requires application modification
  * Sysadmin intervention/setup

### Transparent Huge Pages - THP

* Primarily used for anonymous memory
  * Can be used in tmpfs
  * Limited support for file mappings (XFS, experimental)
* Currently PMD_SIZE support only
* System control via `/sys/kernel/mm/transparent_hugepage/enabled`

### Hugetlbfs

* Requires Application modification
* Huge Pages are generally preallocated via sysadmin control
* Has been used by database for many years
* More recent use as backing for Virtual Machines
  * THP commonly used to back VMs
* Multiple Huge Page Sizes as supported by Architecture



Pools of hugetlb pages are created/preallocated.

Application use the pages in these pools.

Can see `/sys/kernel/mm/hugepages/` for different huge page size.



**Default Hugetlb Page size**

```
# grep Huge /proc/meminfo

AnonHugePages:         0 kB
ShmemHugePages:    20480 kB
FileHugePages:         0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
Hugetlb:               0 kB
```

