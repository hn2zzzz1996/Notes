# APIC driver

When kernel boot, it will choose the apic driver in a for loop.

```C
// arch/x86/kernel/apic/probe_64.c
void __init default_setup_apic_routing(void)
{
    struct apic **drv;

    enable_IR_x2apic();

    for (drv = __apicdrivers; drv < __apicdrivers_end; drv++) {
        if ((*drv)->probe && (*drv)->probe()) {
            if (apic != *drv) {
                apic = *drv;
                pr_info("Switched APIC routing to %s.\n",
                        apic->name);
            }
            break;
        }
    }
}
```

The apic driver are list in the `.apicdrivers` section, and the probed sequence are based on how they are listed in the `.apicdrivers` section. The order is depended on the ordering of the apic driver files list in Makefile.

```C
// arch/x86/kernel/apic/Makefile
ifeq ($(CONFIG_X86_64),y)
  # APIC probe will depend on the listing order here
  obj-$(CONFIG_X86_NUMACHIP)      += apic_numachip.o
  obj-$(CONFIG_X86_UV)            += x2apic_uv_x.o
  obj-$(CONFIG_X86_X2APIC)        += x2apic_phys.o
  obj-$(CONFIG_X86_X2APIC)        += x2apic_cluster.o
  obj-y                           += apic_flat_64.o
  endif
```

And in the vmlinux.lds.S, the apic drivers will be listed together according to the sequence listed in the Makefile:

```C
.apicdrivers : AT(ADDR(.apicdrivers) - LOAD_OFFSET) {
                  __apicdrivers = .;
                 *(.apicdrivers);
                  __apicdrivers_end = .;
         }
```

Due to X2APIC extends APICID from 8 bits to 32 bits, but the device interrupt routed from IOAPIC or delivered in MSI mode will keep 8 bits destination APICID. The interrupt remapping is needed so it  can translate the destination APICID to keep the device compatible.

In KVM guest, if there are no interrupt remapping, the Linux as a kvm guest can only enable x2apic if there are no CPUs present at boot time with an APIC ID above 255. And without Interrupt Remapping, all CPUs can be addressed by IOAPIC/MSI only in physical mode.

### Function: try_to_enable_x2apic()

This function try to enable x2apic.

```C
static __init void try_to_enable_x2apic(int remap_mode)
{
	if (x2apic_state == X2APIC_DISABLED)
		return;

    // If the platform doesn't have irq remapping, the X2APIC can't be used in native. But except for guest, in guest we can use x2apic without irq remap.
	if (remap_mode != IRQ_REMAP_X2APIC_MODE) {
		u32 apic_limit = 255;

		/*
		 * Using X2APIC without IR is not architecturally supported
		 * on bare metal but may be supported in guests.
		 */
		if (!x86_init.hyper.x2apic_available()) {
			pr_info("x2apic: IRQ remapping doesn't support X2APIC mode\n");
			x2apic_disable();
			return;
		}

		/*
		 * If the hypervisor supports extended destination ID in
		 * MSI, that increases the maximum APIC ID that can be
		 * used for non-remapped IRQ domains.
		 */
		if (x86_init.hyper.msi_ext_dest_id()) {
			virt_ext_dest_id = 1;
			apic_limit = 32767;
		}

		/*
		 * Without IR, all CPUs can be addressed by IOAPIC/MSI only
		 * in physical mode, and CPUs with an APIC ID that cannot
		 * be addressed must not be brought online.
		 */
		x2apic_set_max_apicid(apic_limit);
		x2apic_phys = 1;
	}
	x2apic_enable();
}
```

There are some callback in `x86_init.hyper` being used, these callback are set at `init_hypervisor_platform`:

```C
void __init init_hypervisor_platform(void)
{
	const struct hypervisor_x86 *h;

	h = detect_hypervisor_vendor();

	if (!h)
		return;

	copy_array(&h->init, &x86_init.hyper, sizeof(h->init));
	copy_array(&h->runtime, &x86_platform.hyper, sizeof(h->runtime));

	x86_hyper_type = h->type;
	x86_init.hyper.init_platform();
}
```

And for kvm hypervisor, the callback are defined at:

```C
// arch/x86/kernel/kvm.c
const __initconst struct hypervisor_x86 x86_hyper_kvm = {
	.name				= "KVM",
	.detect				= kvm_detect,
	.type				= X86_HYPER_KVM,
	.init.guest_late_init		= kvm_guest_init,
	.init.x2apic_available		= kvm_para_available,
	.init.msi_ext_dest_id		= kvm_msi_ext_dest_id,
	.init.init_platform		= kvm_init_platform,
#if defined(CONFIG_AMD_MEM_ENCRYPT)
	.runtime.sev_es_hcall_prepare	= kvm_sev_es_hcall_prepare,
	.runtime.sev_es_hcall_finish	= kvm_sev_es_hcall_finish,
#endif
};
```

