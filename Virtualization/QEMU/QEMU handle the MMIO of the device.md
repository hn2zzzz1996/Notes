# QEMU handle the MMIO of the device

Here is one code path that the `edu_mmio_read` eventual been called. The `edu_mmio_read` is the edu device, it provides this function.

```C
#0  edu_mmio_read (opaque=0x555557be0eb0, addr=4, size=4) at ../hw/misc/edu.c:190
#1  0x0000555555d82e7a in memory_region_read_accessor (mr=0x555557be1810, addr=4, value=0x7ffff4894640, size=4, shift=0, mask=4294967295, attrs=...) at ../softmmu/memory.c:440
#2  0x0000555555d833d4 in access_with_adjusted_size (addr=4, value=0x7ffff4894640, size=4, access_size_min=4, access_size_max=8, access_fn=
    0x555555d82e33 <memory_region_read_accessor>, mr=0x555557be1810, attrs=...) at ../softmmu/memory.c:554
#3  0x0000555555d860d1 in memory_region_dispatch_read1 (mr=0x555557be1810, addr=4, pval=0x7ffff4894640, size=4, attrs=...) at ../softmmu/memory.c:1424
#4  0x0000555555d861a7 in memory_region_dispatch_read (mr=0x555557be1810, addr=4, pval=0x7ffff4894640, op=MO_32, attrs=...) at ../softmmu/memory.c:1452
#5  0x0000555555df9831 in flatview_read_continue (fv=0x7fffec846ae0, addr=4271898628, attrs=..., ptr=0x7ffff7fbc028, len=4, addr1=4, l=4, mr=0x555557be1810) at ../softmmu/physmem.c:2841
#6  0x0000555555df9988 in flatview_read (fv=0x7fffec846ae0, addr=4271898628, attrs=..., buf=0x7ffff7fbc028, len=4) at ../softmmu/physmem.c:2880
#7  0x0000555555df9a15 in address_space_read_full (as=0x555556a40440 <address_space_memory>, addr=4271898628, attrs=..., buf=0x7ffff7fbc028, len=4) at ../softmmu/physmem.c:2893
#8  0x0000555555df9b41 in address_space_rw (as=0x555556a40440 <address_space_memory>, addr=4271898628, attrs=..., buf=0x7ffff7fbc028, len=4, is_write=false) at ../softmmu/physmem.c:2921
#9  0x0000555555e51859 in kvm_cpu_exec (cpu=0x555556d48bb0) at ../accel/kvm/kvm-all.c:2893
#10 0x0000555555d41ccd in kvm_vcpu_thread_fn (arg=0x555556d48bb0) at ../accel/kvm/kvm-accel-ops.c:49
#11 0x00005555560fc5e8 in qemu_thread_start (args=0x555556d55c10) at ../util/qemu-thread-posix.c:541
#12 0x00007ffff777f450 in start_thread (arg=0x7ffff4895640) at pthread_create.c:473
#13 0x00007ffff76a1d53 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
```

