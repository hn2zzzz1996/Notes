First using `git format-patch` to generate patches:

```Shell
$ git format-patch A..B
```

This will generate patches in the range of commits (A, B]. The most things should be noticed is that the commit A will not be generated.

So I always prefer to do it in the follow way:

```Shell
$ git format-patch A^..B
```

This syntax will include the first commit object. And I always like to use this format.

## Versioning one patch revision

Using `--subject-prefix` to add a versioning.

```Shell
git format-patch --subject-prefix="PATCH v2"
```

## A description for a patchset

We can using `--cover-letter` to generate a cover letter. And use it to describe what you do in the patch set and indicate what you changes differs from the previous version. You can use cover letter to describe them.

Here is an simple:

```
This is the v6 of this series which tries to implement the fd-based KVM
guest private memory. The patches are based on latest kvm/queue branch
commit:

  2764011106d0 (kvm/queue) KVM: VMX: Include MKTME KeyID bits in
shadow_zero_check
 
and Sean's below patch:

  KVM: x86/mmu: Add RET_PF_CONTINUE to eliminate bool+int* "returns"
  https://lkml.org/lkml/2022/4/22/1598

Introduction
------------
In general this patch series introduce fd-based memslot which provides
guest memory through memory file descriptor fd[offset,size] instead of
hva/size. The fd can be created from a supported memory filesystem
like tmpfs/hugetlbfs etc. which we refer as memory backing store. KVM
and the the memory backing store exchange callbacks when such memslot
gets created. At runtime KVM will call into callbacks provided by the
backing store to get the pfn with the fd+offset. Memory backing store
will also call into KVM callbacks when userspace fallocate/punch hole
on the fd to notify KVM to map/unmap secondary MMU page tables.

Comparing to existing hva-based memslot, this new type of memslot allows
guest memory unmapped from host userspace like QEMU and even the kernel
itself, therefore reduce attack surface and prevent bugs.

Based on this fd-based memslot, we can build guest private memory that
is going to be used in confidential computing environments such as Intel
TDX and AMD SEV. When supported, the memory backing store can provide
more enforcement on the fd and KVM can use a single memslot to hold both
the private and shared part of the guest memory. 

mm extension
---------------------
Introduces new MFD_INACCESSIBLE flag for memfd_create(), the file created
with these flags cannot read(), write() or mmap() etc via normal
MMU operations. The file content can only be used with the newly
introduced memfile_notifier extension.

The memfile_notifier extension provides two sets of callbacks for KVM to
interact with the memory backing store:
  - memfile_notifier_ops: callbacks for memory backing store to notify
    KVM when memory gets allocated/invalidated.
  - backing store callbacks: callbacks for KVM to call into memory backing
    store to request memory pages for guest private memory.

The memfile_notifier extension also provides APIs for memory backing
store to register/unregister itself and to trigger the notifier when the
bookmarked memory gets fallocated/invalidated.

memslot extension
-----------------
Add the private fd and the fd offset to existing 'shared' memslot so that
both private/shared guest memory can live in one single memslot. A page in
the memslot is either private or shared. A page is private only when it's
already allocated in the backing store fd, all the other cases it's treated
as shared, this includes those already mapped as shared as well as those
having not been mapped. This means the memory backing store is the place
which tells the truth of which page is private.

Private memory map/unmap and conversion
---------------------------------------
Userspace's map/unmap operations are done by fallocate() ioctl on the
backing store fd.
  - map: default fallocate() with mode=0.
  - unmap: fallocate() with FALLOC_FL_PUNCH_HOLE.
The map/unmap will trigger above memfile_notifier_ops to let KVM map/unmap
secondary MMU page tables.

Test
----
To test the new functionalities of this patch TDX patchset is needed.
Since TDX patchset has not been merged so I did two kinds of test:

-  Selftest on normal VM from Vishal
   https://lkml.org/lkml/2022/5/10/2045
   The selftest has been ported to this patchset and you can find it in
   repo: https://github.com/chao-p/linux/tree/privmem-v6

-  Private memory funational test on latest TDX code
   The patch is rebased to latest TDX code and tested the new
   funcationalities. See below repos:
   Linux: https://github.com/chao-p/linux/commits/privmem-v6-tdx
   QEMU: https://github.com/chao-p/qemu/tree/privmem-v6

An example QEMU command line for TDX test:
-object tdx-guest,id=tdx \
-object memory-backend-memfd-private,id=ram1,size=2G \
-machine q35,kvm-type=tdx,pic=no,kernel_irqchip=split,memory-encryption=tdx,memory-backend=ram1

What's missing
--------------
  - The accounting for longterm pinned memory in the backing store is
    not included since I havn't come out a good solution yet.
  - Batch invalidation notify for shmem is not ready, as I still see
    it's a bit tricky to do that clearly.

Changelog
----------
v6:
  - Re-organzied patch for both mm/KVM parts.
  - Added flags for memfile_notifier so its consumers can state their
    features and memory backing store can check against these flags.
  - Put a backing store reference in the memfile_notifier and move pfn_ops
    into backing store.
  - Only support boot time backing store register.
  - Overall KVM part improvement suggested by Sean and some others.
v5:
  - Removed userspace visible F_SEAL_INACCESSIBLE, instead using an
    in-kernel flag (SHM_F_INACCESSIBLE for shmem). Private fd can only
    be created by MFD_INACCESSIBLE.
  - Introduced new APIs for backing store to register itself to
    memfile_notifier instead of direct function call.
  - Added the accounting and restriction for MFD_INACCESSIBLE memory.
  - Added KVM API doc for new memslot extensions and man page for the new
    MFD_INACCESSIBLE flag.
  - Removed the overlap check for mapping the same file+offset into
    multiple gfns due to perf consideration, warned in document.
  - Addressed other comments in v4.
v4:
  - Decoupled the callbacks between KVM/mm from memfd and use new
    name 'memfile_notifier'.
  - Supported register multiple memslots to the same backing store.
  - Added per-memslot pfn_ops instead of per-system.
  - Reworked the invalidation part.
  - Improved new KVM uAPIs (private memslot extension and memory
    error) per Sean's suggestions.
  - Addressed many other minor fixes for comments from v3.
v3:
  - Added locking protection when calling
    invalidate_page_range/fallocate callbacks.
  - Changed memslot structure to keep use useraddr for shared memory.
  - Re-organized F_SEAL_INACCESSIBLE and MEMFD_OPS.
  - Added MFD_INACCESSIBLE flag to force F_SEAL_INACCESSIBLE.
  - Commit message improvement.
  - Many small fixes for comments from the last version.

Links to previous discussions
-----------------------------
[1] Original design proposal:
https://lkml.kernel.org/kvm/20210824005248.200037-1-seanjc@google.com/
[2] Updated proposal and RFC patch v1:
https://lkml.kernel.org/linux-fsdevel/20211111141352.26311-1-chao.p.peng@linux.intel.com/
[3] Patch v5: https://lkml.org/lkml/2022/3/10/457

Chao Peng (6):
  mm: Introduce memfile_notifier
  mm/memfd: Introduce MFD_INACCESSIBLE flag
  KVM: Extend the memslot to support fd-based private memory
  KVM: Add KVM_EXIT_MEMORY_FAULT exit
  KVM: Handle page fault for private memory
  KVM: Enable and expose KVM_MEM_PRIVATE
```
