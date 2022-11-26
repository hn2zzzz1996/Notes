# memory

The memory API models the memory and I/O buses and controllers of a QEMU machine. It attempts to allow modeling of:

* ordinary RAM
* memory-mapped I/O (MMIO)
* memory controllers that can dynamically reroute physical memory regions to different destinations

Memory is modelled as an acyclic graph of `MemoryRegion` objects. Leaves are RAM and MMIO regions, while other nodes represent buses, memory controllers, and memory regions that have been rerouted.

## Types of regions

A memory region is represented by `struct MemoryRegion` in QEMU. There are multiple types of memory regions.

* RAM: a RAM region is simply a range of host memory that can be made available to the guest. You typically initialize these with `memory_region_init_ram()`.

* MMIO: a range of guest memory that is implemented by host callbacks; each read or write causes a callback to be called on the host. You initialize these with `memory_region_init_io()`, passing it a `MemoryRegionOps` structure describing the callbacks.

* container: A container simply includes each memory regions, each at a different offset. Containers are useful for grouping several regions into one unit. For example, a PCI BAR may be composed of a RAM region and an MMIO region.

  A container's subregions are usually non-overlapping.

  Initialize a pure container with `memory_region_init`.

* alias: a subsection of another region. Aliases allow a region to be split apart into discontiguous regions. Examples of uses are a memory controller that splits main memory to expose a "PCI hole".

## Region names

For most regions these are only used for debugging purposes.

## Region lifecycle

## Overlapping regions and priority

`memory_region_add_subregion_overlap()` allows the region to overlap any other region in the same container, and specifies a priority that allows the core to decide which of two regions at the same address are visible (highest wins). Priority values are signed, and the default value is zero.



## QEMU source code

In `cpu_exec_init_all()`, will call `io_mem_init()` and `memory_map_init()`. `memory_map_init()` is a important function.

```C
// softmmu/physmem.c
  3053 void cpu_exec_init_all(void)
  3054 {
  3055     qemu_mutex_init(&ram_list.mutex);
  3056     /* The data structures we set up here depend on knowing the page size,
  3057      * so no more changes can be made after this point.
  3058      * In an ideal world, nothing we did before we had finished the
  3059      * machine setup would care about the target page size, and we could
  3060      * do this much later, rather than requiring board models to state
  3061      * up front what their requirements are.
  3062      */
  3063     finalize_target_page_bits();
  3064     io_mem_init();
  3065     memory_map_init();
  3066     qemu_mutex_init(&map_client_list_lock);
  3067 }
```

In `memory_map_init`, qemu will create two memory region and initialize two address space.

```C
// softmmu/physmem.c
    87 static MemoryRegion *system_memory;
    88 static MemoryRegion *system_io;

    90 AddressSpace address_space_io;
    91 AddressSpace address_space_memory;

  2657 static void memory_map_init(void)
  2658 {
  2659     system_memory = g_malloc(sizeof(*system_memory));
  2660
  2661     memory_region_init(system_memory, NULL, "system", UINT64_MAX);
      // This will set system_memory as root of the address_space_memory.
  2662     address_space_init(&address_space_memory, system_memory, "memory");
  2663
  2664     system_io = g_malloc(sizeof(*system_io));
  2665     memory_region_init_io(system_io, NULL, &unassigned_io_ops, NULL, "io",
  2666         ¦       ¦       ¦ 65536);
      // This will set system_io as root of the address_space_io.
  2667     address_space_init(&address_space_io, system_io, "I/O");
  2668 }
```

Then in the `pc_init1` function, there will create the memory for guest.

```C
// hw/i386/pc_piix.c
   74 /* PC hardware initialisation */
   75 static void pc_init1(MachineState *machine,
   76         ¦       ¦    const char *host_type, const char *pci_type)
   77 {
       
   91     MemoryRegion *ram_memory;
   92     MemoryRegion *pci_memory;
   93     MemoryRegion *rom_memory;

  163     if (pcmc->pci_enabled) {
  164         pci_memory = g_new(MemoryRegion, 1);
  165         memory_region_init(pci_memory, NULL, "pci", UINT64_MAX);
  166         rom_memory = pci_memory;
	  	  }

  183     /* allocate ram and load rom/bios */
  184     if (!xen_enabled()) {
  185         pc_memory_init(pcms, system_memory,
  186         ¦       ¦      rom_memory, &ram_memory);
  		  }
```

```C
// hw/i386/pc.c
   853 void pc_memory_init(PCMachineState *pcms,
   854         ¦       ¦   MemoryRegion *system_memory,
   855         ¦       ¦   MemoryRegion *rom_memory,
   856         ¦       ¦   MemoryRegion **ram_memory)
   857 {
       
// Original, the ram_memory is allocated by:
// memory_region_allocate_system_memory(ram, NULL, "pc.ram", machine->ram_size);
   876     *ram_memory = machine->ram;
   877     ram_below_4g = g_malloc(sizeof(*ram_below_4g));
// The ram memory will be split into two part, one is below 4g, one is above 4g, so create two alias to the machine->ram.
   878     memory_region_init_alias(ram_below_4g, NULL, "ram-below-4g", machine->ram,
   879         ¦       ¦       ¦    0, x86ms->below_4g_mem_size);
   880     memory_region_add_subregion(system_memory, 0, ram_below_4g);
   881     e820_add_entry(0, x86ms->below_4g_mem_size, E820_RAM);
   882     if (x86ms->above_4g_mem_size > 0) {
   883         ram_above_4g = g_malloc(sizeof(*ram_above_4g));
   884         memory_region_init_alias(ram_above_4g, NULL, "ram-above-4g",
   885         ¦       ¦       ¦       ¦machine->ram,
   886         ¦       ¦       ¦       ¦x86ms->below_4g_mem_size,
   887         ¦       ¦       ¦       ¦x86ms->above_4g_mem_size);
   888         memory_region_add_subregion(system_memory, 0x100000000ULL,
   889         ¦       ¦       ¦       ¦   ram_above_4g);
   890         e820_add_entry(0x100000000ULL, x86ms->above_4g_mem_size, E820_RAM);
   891     }

   944     /* Initialize PC system firmware */
   945     pc_system_firmware_init(pcms, rom_memory);
```

At line 876, we notice that the `ram_memory` is directly assigned from `machine->ram`.  Where is the `machine->ram` being initialized? we can see from below:

```C
// ./hw/core/machine.c  
1191 void machine_run_board_init(MachineState *machine)
  1192 {
  1193     MachineClass *machine_class = MACHINE_GET_CLASS(machine);
  1194     ObjectClass *oc = object_class_by_name(machine->cpu_type);
  1195     CPUClass *cc;
  1196
  1197     /* This checkpoint is required by replay to separate prior clock
  1198        reading from the other reads, because timer polling functions query
  1199        clock values from the log. */
  1200     replay_checkpoint(CHECKPOINT_INIT);
  1201
  1202     if (machine->ram_memdev_id) {
  1203         Object *o;
  1204         o = object_resolve_path_type(machine->ram_memdev_id,
  1205         ¦       ¦       ¦       ¦    TYPE_MEMORY_BACKEND, NULL);
  1206         machine->ram = machine_consume_memdev(machine, MEMORY_BACKEND(o));
  1207     }
```

At line 1206, the `machine->ram` is assigned from memdev. And the `machine->ram_memdev_id` is `"pc.ram";`. The `ram_memdev_id` is assigned at `pc_machine_class_init` in `hw/i386/pc.c`.

```C
mc->default_ram_id = "pc.ram";
```

Here is the commit:

```Shell
commit bd457782b3b0a313f3991038eb55bc44369c72c6
Author: Igor Mammedov <imammedo@redhat.com>
Date:   Wed Feb 19 11:09:17 2020 -0500

    x86/pc: use memdev for RAM

    memory_region_allocate_system_memory() API is going away, so
    replace it with memdev allocated MemoryRegion. The later is
    initialized by generic code, so board only needs to opt in
    to memdev scheme by providing
      MachineClass::default_ram_id
    and using MachineState::ram instead of manually initializing
    RAM memory region.

Message-Id: <20200219160953.13771-44-imammedo@redhat.com>
```

### QEMU KVM set memory

This code using the QEMU v4.2.0.

The QEMU will use the `KVM_SET_USER_MEMORY_REGION` API which provided by KVM to set memslot. 

```C
// accel/kvm/kvm-all.c
286 static int kvm_set_user_memory_region(KVMMemoryListener *kml, KVMSlot *slot, bool new)
   287 {
   288     KVMState *s = kvm_state;
   289     struct kvm_userspace_memory_region mem;
   290     int ret;
   291
   292     mem.slot = slot->slot | (kml->as_id << 16);
   293     mem.guest_phys_addr = slot->start_addr;
   294     mem.userspace_addr = (unsigned long)slot->ram;
   295     mem.flags = slot->flags;
   296
   297     if (slot->memory_size && !new && (mem.flags ^ slot->old_flags) & KVM_MEM_READONLY) {
   298         /* Set the slot size to 0 before setting the slot to the desired
   299         ¦* value. This is needed based on KVM commit 75d61fbc. */
   300         mem.memory_size = 0;
   301         ret = kvm_vm_ioctl(s, KVM_SET_USER_MEMORY_REGION, &mem);
   302         if (ret < 0) {
   303         ¦   goto err;
   304         }
   305     }
   306     mem.memory_size = slot->memory_size;
   307     ret = kvm_vm_ioctl(s, KVM_SET_USER_MEMORY_REGION, &mem);
   308     slot->old_flags = mem.flags;
   309 err:
   310     trace_kvm_set_user_memory(mem.slot, mem.flags, mem.guest_phys_addr,
   311         ¦       ¦       ¦     mem.memory_size, mem.userspace_addr, ret);
   312     if (ret < 0) {
   313         error_report("%s: KVM_SET_USER_MEMORY_REGION failed, slot=%d,"
   314         ¦       ¦    " start=0x%" PRIx64 ", size=0x%" PRIx64 ": %s",
   315         ¦       ¦    __func__, mem.slot, slot->start_addr,
   316         ¦       ¦    (uint64_t)mem.memory_size, strerror(errno));
   317     }
   318     return ret;
   319 }

```

The `kvm_set_user_memory_region` will be called at two places. One is `kvm_slot_update_flags`, another is `kvm_set_phys_mem`. We will look at kvm_set_phys_mem, it's the function which being called in `kvm_region_add` and `kvm_region_del`. And the two functions are the callback function in the `MemoryListener`.

```C
  1601 void kvm_memory_listener_register(KVMState *s, KVMMemoryListener *kml,
  1602         ¦       ¦       ¦       ¦ AddressSpace *as, int as_id, const char *name)
  1603 {
  1604     int i;
  1605
  1606     kml->slots = g_new0(KVMSlot, s->nr_slots);
  1607     kml->as_id = as_id;
  1608
  1609     for (i = 0; i < s->nr_slots; i++) {
  1610         kml->slots[i].slot = i;
  1611     }
  1612
  1613     kml->listener.region_add = kvm_region_add;
  1614     kml->listener.region_del = kvm_region_del;
  1615     kml->listener.log_start = kvm_log_start;
  1616     kml->listener.log_stop = kvm_log_stop;
  1617     kml->listener.priority = 10;
  1618     kml->listener.name = name;
  1619
  1620     if (s->kvm_dirty_ring_size) {
  1621         kml->listener.log_sync_global = kvm_log_sync_global;
  1622     } else {
  1623         kml->listener.log_sync = kvm_log_sync;
  1624         kml->listener.log_clear = kvm_log_clear;
  1625     }
  1626
  1627     memory_listener_register(&kml->listener, as);
  		}
```

Here is a function graph which the kvm_region_add being called.

```C
#0  kvm_region_add (listener=0x5555568d9a10, section=0x7fffffffd9e0) at ../accel/kvm/kvm-all.c:1457
#1  0x0000555555ce0eaf in address_space_update_topology_pass (as=0x555556841440 <address_space_memory>, old_view=0x555556b15010, new_view=0x555556b3c2d0, adding=true) at ../softmmu/memory.c:976
#2  0x0000555555ce11bb in address_space_set_flatview (as=0x555556841440 <address_space_memory>) at ../softmmu/memory.c:1052
#3  0x0000555555ce1380 in memory_region_transaction_commit () at ../softmmu/memory.c:1104
#4  0x0000555555ce5414 in memory_region_update_container_subregions (subregion=0x555556a219e0) at ../softmmu/memory.c:2613
#5  0x0000555555ce54c3 in memory_region_add_subregion_common (mr=0x555556a5f2e0, offset=4276092928, subregion=0x555556a219e0) at ../softmmu/memory.c:2628
#6  0x0000555555ce5545 in memory_region_add_subregion_overlap (mr=0x555556a5f2e0, offset=4276092928, subregion=0x555556a219e0, priority=4096) at ../softmmu/memory.c:2645
#7  0x0000555555b4d17f in x86_cpu_apic_realize (cpu=0x555556b35320, errp=0x7fffffffdbf8) at ../target/i386/cpu-sysemu.c:300

x86_cpu_apic_realize
    -> memory_region_add_subregion_overlap
    	-> memory_region_update_container_subregions
    		-> memory_region_transaction_commit
    			-> address_space_set_flatview
    				-> address_space_update_topology_pass
    					-> kvm_region_add
```



