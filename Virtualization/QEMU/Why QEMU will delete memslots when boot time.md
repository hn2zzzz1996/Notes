# Why QEMU will delete memslots when boot time

Here is a commit about it:

```C
commit b8af1afb("memory: separate building the final memory map into two steps")

Instead of adding and deleting regions in one pass, do a delete
    pass followed by an add pass.  This fixes the following case:

    from:
      0x0000-0x0fff ram  (a1)
      0x1000-0x1fff mmio (a2)
      0x2000-0x2fff ram  (a3)

    to:
      0x0000-0x2fff ram  (b1)

    The single pass algorithm removed a1, added b2, then removed a2 and a3,
    which caused the wrong memory map to be built.  The two pass algorithm
    removes a1, a2, and a3, then adds b1.
```

Here is a stack info about how qemu delete memslot:

```
#0  kvm_set_user_memory_region (kml=0x555556ac4f60, slot=0x7ffff515c0e8, new=false) at ../accel/kvm/kvm-all.c:353
#1  0x0000555555e4dfc3 in kvm_set_phys_mem (kml=0x555556ac4f60, section=0x7ffff5097240, add=false) at ../accel/kvm/kvm-all.c:1404
#2  0x0000555555e4e30f in kvm_region_del (listener=0x555556ac4f60, section=0x7ffff5097240) at ../accel/kvm/kvm-all.c:1502
#3  0x0000555555d84d7f in address_space_update_topology_pass (as=0x555556a40440 <address_space_memory>, old_view=0x7fffec000bf0, new_view=0x7fffec031420, adding=false)
    at ../softmmu/memory.c:948
#4  0x0000555555d85352 in address_space_set_flatview (as=0x555556a40440 <address_space_memory>) at ../softmmu/memory.c:1050
#5  0x0000555555d8551e in memory_region_transaction_commit () at ../softmmu/memory.c:1103
#6  0x0000555555995430 in i440fx_update_memory_mappings (d=0x555556e9af80) at ../hw/pci-host/i440fx.c:81
#7  0x00005555559954b9 in i440fx_write_config (dev=0x555556e9af80, address=92, val=858993459, len=4) at ../hw/pci-host/i440fx.c:94
#8  0x00005555559940e3 in pci_host_config_write_common (pci_dev=0x555556e9af80, addr=92, limit=256, val=858993459, len=4) at ../hw/pci/pci_host.c:83
#9  0x0000555555994252 in pci_data_write (s=0x555556e63710, addr=2147483740, val=858993459, len=4) at ../hw/pci/pci_host.c:120
#10 0x0000555555994388 in pci_host_data_write (opaque=0x555556e62690, addr=0, val=858993459, len=4) at ../hw/pci/pci_host.c:167
#11 0x0000555555d83182 in memory_region_write_accessor (mr=0x555556e62ab0, addr=0, value=0x7ffff5097588, size=4, shift=0, mask=4294967295, attrs=...) at ../softmmu/memory.c:492
#12 0x0000555555d833d4 in access_with_adjusted_size (addr=0, value=0x7ffff5097588, size=4, access_size_min=1, access_size_max=4, access_fn=
    0x555555d83082 <memory_region_write_accessor>, mr=0x555556e62ab0, attrs=...) at ../softmmu/memory.c:554
#13 0x0000555555d8647c in memory_region_dispatch_write (mr=0x555556e62ab0, addr=0, data=858993459, op=MO_32, attrs=...) at ../softmmu/memory.c:1504
#14 0x0000555555df95ed in flatview_write_continue (fv=0x7fffec015d60, addr=3324, attrs=..., ptr=0x7ffff7fc0000, len=4, addr1=0, l=4, mr=0x555556e62ab0) at ../softmmu/physmem.c:2777
#15 0x0000555555df9736 in flatview_write (fv=0x7fffec015d60, addr=3324, attrs=..., buf=0x7ffff7fc0000, len=4) at ../softmmu/physmem.c:2817
#16 0x0000555555df9ab0 in address_space_write (as=0x555556a403e0 <address_space_io>, addr=3324, attrs=..., buf=0x7ffff7fc0000, len=4) at ../softmmu/physmem.c:2909
#17 0x0000555555df9b21 in address_space_rw (as=0x555556a403e0 <address_space_io>, addr=3324, attrs=..., buf=0x7ffff7fc0000, len=4, is_write=true) at ../softmmu/physmem.c:2919
#18 0x0000555555e51006 in kvm_handle_io (port=3324, attrs=..., data=0x7ffff7fc0000, direction=1, size=4, count=1) at ../accel/kvm/kvm-all.c:2632
#19 0x0000555555e51809 in kvm_cpu_exec (cpu=0x555556cff510) at ../accel/kvm/kvm-all.c:2883
#20 0x0000555555d41ccd in kvm_vcpu_thread_fn (arg=0x555556cff510) at ../accel/kvm/kvm-accel-ops.c:49
#21 0x00005555560fc5e8 in qemu_thread_start (args=0x555556d0d0d0) at ../util/qemu-thread-posix.c:541
#22 0x00007ffff7781450 in start_thread (arg=0x7ffff5098640) at pthread_create.c:473
#23 0x00007ffff76a3d53 in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:95
```

这里对QEMU运行的时候进行了trace，下面是一些output：

```C
pci_cfg_write i440FX 00:0 @0x58 <- 0x33333000
kvm_set_user_memory Slot#2 flags=0x0 gpa=0xc0000 size=0x0 ua=0x7ff567a00000 ret=0
kvm_set_user_memory Slot#3 flags=0x0 gpa=0xe0000 size=0x0 ua=0x7ff567c20000 ret=0
kvm_set_user_memory Slot#4 flags=0x0 gpa=0x100000 size=0x0 ua=0x7ff567f00000 ret=0
kvm_set_user_memory Slot#2 flags=0x0 gpa=0xc0000 size=0x10000 ua=0x7ff567ec0000 ret=0
kvm_set_user_memory Slot#3 flags=0x2 gpa=0xd0000 size=0x10000 ua=0x7ff567a10000 ret=0
kvm_set_user_memory Slot#4 flags=0x2 gpa=0xe0000 size=0x10000 ua=0x7ff567c20000 ret=0
kvm_set_user_memory Slot#5 flags=0x0 gpa=0xf0000 size=0x3ff10000 ua=0x7ff567ef0000 ret=0
kvm_set_user_memory Slot#65536 flags=0x0 gpa=0x0 size=0x0 ua=0x7ff567e00000 ret=0
kvm_set_user_memory Slot#65537 flags=0x0 gpa=0xc0000 size=0x0 ua=0x7ff567a00000 ret=0
kvm_set_user_memory Slot#65538 flags=0x0 gpa=0xe0000 size=0x0 ua=0x7ff567c20000 ret=0
kvm_set_user_memory Slot#65539 flags=0x0 gpa=0x100000 size=0x0 ua=0x7ff567f00000 ret=0
kvm_set_user_memory Slot#65536 flags=0x0 gpa=0x0 size=0xa0000 ua=0x7ff567e00000 ret=0
kvm_set_user_memory Slot#65537 flags=0x0 gpa=0xc0000 size=0x10000 ua=0x7ff567ec0000 ret=0
kvm_set_user_memory Slot#65538 flags=0x2 gpa=0xd0000 size=0x10000 ua=0x7ff567a10000 ret=0
kvm_set_user_memory Slot#65539 flags=0x2 gpa=0xe0000 size=0x10000 ua=0x7ff567c20000 ret=0
kvm_set_user_memory Slot#65541 flags=0x0 gpa=0xf0000 size=0x3ff10000 ua=0x7ff567ef0000 ret=0
pci_cfg_write i440FX 00:0 @0x5c <- 0x33333333
kvm_set_user_memory Slot#2 flags=0x0 gpa=0xc0000 size=0x0 ua=0x7ff567ec0000 ret=0
kvm_set_user_memory Slot#3 flags=0x0 gpa=0xd0000 size=0x0 ua=0x7ff567a10000 ret=0
kvm_set_user_memory Slot#4 flags=0x0 gpa=0xe0000 size=0x0 ua=0x7ff567c20000 ret=0
kvm_set_user_memory Slot#5 flags=0x0 gpa=0xf0000 size=0x0 ua=0x7ff567ef0000 ret=0
kvm_set_user_memory Slot#2 flags=0x0 gpa=0xc0000 size=0x3ff40000 ua=0x7ff567ec0000 ret=0
kvm_set_user_memory Slot#65537 flags=0x0 gpa=0xc0000 size=0x0 ua=0x7ff567ec0000 ret=0
kvm_set_user_memory Slot#65538 flags=0x0 gpa=0xd0000 size=0x0 ua=0x7ff567a10000 ret=0
kvm_set_user_memory Slot#65539 flags=0x0 gpa=0xe0000 size=0x0 ua=0x7ff567c20000 ret=0
kvm_set_user_memory Slot#65541 flags=0x0 gpa=0xf0000 size=0x0 ua=0x7ff567ef0000 ret=0
kvm_set_user_memory Slot#65537 flags=0x0 gpa=0xc0000 size=0x3ff40000 ua=0x7ff567ec0000 ret=0
```

这里需要重点注意的是**pci_cfg_write**，这里的输出分别对应着：

```C
qemu_log("pci_cfg_write " "%s %02u:%u @0x%x <- 0x%x" "\n", dev, devid, fnid, offs, val)
```

比如`pci_cfg_write i440FX 00:0 @0x58 <- 0x33333000`分别对应着：

* dev: i440FX
* devid: 00
* fnid: 0
* offs: 0x58
* val: 0x33333000

这里需要参考一下`i440FX`的手册来看一下这些参数到底是什么含义，具体可以看：

[i440FX手册](https://wiki.qemu.org/images/b/bb/29054901.pdf)

比如这里写入的0x58是`DRAMT-DRAM TIMING REGISTER`，这个寄存器可以控制`main memory DRAM timings`。

写入的0x5c是`PAM-PROGRAMMABLE ATTRIBUTE MAP REGISTERS`，这个寄存器控制内存的属性。

* pci的VGA写入也会导致内存条重设置

```C
pci_cfg_write VGA 02:0 @0x4 <- 0x503
pci_cfg_write VGA 02:0 @0x4 <- 0x103
pci_cfg_write VGA 02:0 @0x4 <- 0x100
kvm_set_user_memory Slot#3 flags=0x0 gpa=0xfd000000 size=0x0 ua=0x7feba9800000 ret=0
kvm_set_user_memory Slot#65538 flags=0x0 gpa=0xfd000000 size=0x0 ua=0x7feba9800000 ret=0
pci_cfg_write VGA 02:0 @0x10 <- 0xffffffff
pci_cfg_write VGA 02:0 @0x10 <- 0xfd000008
pci_cfg_write VGA 02:0 @0x4 <- 0x103
kvm_set_user_memory Slot#3 flags=0x1 gpa=0xfd000000 size=0x1000000 ua=0x7feba9800000 ret=0
kvm_set_user_memory Slot#65538 flags=0x1 gpa=0xfd000000 size=0x1000000 ua=0x7feba9800000 ret=0
```

