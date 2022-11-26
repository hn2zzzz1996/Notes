# How TDX enable #VE

In a general explanation, the TDX can trigger #VE when the guest access the MMIO region. Here is the question, In kvm, the MMIO access always trigger `EPT_MISCONFIG` , but the intel spec said the `EPT_MISCONFIG` always make the guest exit, how TDX trigger #VE when access MMIO region?

The answer is easy, just set the MMIO entry to non-present, so when the guest access the region, it will trigger `EPT_VIOLATION`, and if not set the #SVE bit, this will to convert to #VE into the guest.

Here is some code implementation:

The `kvm_mmu_set_mmio_spte_mask` will set the `shadow_mmio_value` field, which will be used by `make_mmio_spte`.

```C
  254 void kvm_mmu_set_mmio_spte_mask(struct kvm *kvm, u64 mmio_value,
  255                                 u64 access_mask)
  256 {
  257         BUG_ON((u64)(unsigned)access_mask != access_mask);
  258         WARN_ON(mmio_value & (shadow_nonpresent_or_rsvd_mask << SHADOW_NONPRESENT_OR_RSVD_MASK_LEN));
  259         WARN_ON(mmio_value & shadow_nonpresent_or_rsvd_lower_gfn_mask);
  260         kvm->arch.shadow_mmio_value = mmio_value | SPTE_MMIO_MASK;
  261         shadow_mmio_access_mask = access_mask;
  262 }
```

The kvm and the tdx set this filed to a different value:

```C
int vmx_vm_init()
{
	kvm_mmu_set_mmio_spte_mask(kvm, VMX_EPT_MISCONFIG_WX_VALUE, 0);
}
```

```C
int tdx_vm_init()
{
    kvm_mmu_set_mmio_spte_mask(kvm, 0, 0);
}
```

The second parameter is the `mmio_value` which will be set to `arch.shadow_mmio_value`. And the macro `VMX_EPT_MISCONFIG_WX_VALUE` is `0x6`, which will cause `EPT_MISCONFIG`.

But tdx set it to 0, which will cause `EPT_VIOLATION`.

```C
49 u64 make_mmio_spte(struct kvm_vcpu *vcpu, u64 gfn, unsigned int access)
   50 {
   51         u64 gen = kvm_vcpu_memslots(vcpu)->generation & MMIO_SPTE_GEN_MASK;
   52         u64 mask = generation_mmio_spte_mask(gen);
   53         u64 gpa = gfn << PAGE_SHIFT;
   54
   55         access &= shadow_mmio_access_mask;
    // At here, we use the value.
   56         mask |= vcpu->kvm->arch.shadow_mmio_value | SPTE_MMIO_MASK | access;
   57         mask |= gpa | shadow_nonpresent_or_rsvd_mask;
   58         mask |= (gpa & shadow_nonpresent_or_rsvd_mask)
   59                 << SHADOW_NONPRESENT_OR_RSVD_MASK_LEN;
   60
   61         return mask;
   62 }

```

Also, for these normal pages, they should suppress #VE. Which is being set by the bit(63).

```C 
    52 static __init void vt_set_ept_masks(void)
    53 {
    54         const u64 u_mask = VMX_EPT_READABLE_MASK;
    55         const u64 a_mask = enable_ept_ad_bits ? VMX_EPT_ACCESS_BIT : 0ull;
    56         const u64 d_mask = enable_ept_ad_bits ? VMX_EPT_DIRTY_BIT : 0ull;
    57         const u64 x_mask = VMX_EPT_EXECUTABLE_MASK;
    58         const u64 nx_mask = 0ull;
        // Here set the init_value. Which is the bit(63) when enable tdx.
    59         const u64 init_value = enable_tdx ? VMX_EPT_SUPPRESS_VE_BIT : 0ull;
    60         const u64 p_mask = (cpu_has_vmx_ept_execute_only() ?
                                   // Also used here.
    61                                 0ull : VMX_EPT_READABLE_MASK) | init_value;
    62
        // In this function, the init_value will be set to shadow_present_mask.
    63         kvm_mmu_set_mask_ptes(u_mask, a_mask, d_mask, nx_mask, x_mask, p_mask,
    64                         ¦     VMX_EPT_RWX_MASK, 0ull);
    65
        // Here we set the shadow_init_value. Which will Initialize all entry in the ept page table entry.
    66         kvm_mmu_set_spte_init_value(init_value);
    67 }

```

And the `shadow_present_mask` will be used in `make_spte` like below. Which will set the present bit into the new spte.

```C
int make_spte()
{
    spte |= shadow_present_mask;
}
```



Here is an important commit which initialize every entry in the empty page that just allocated which will be insert into the page table.

* commit: 4d67bd07da9312

```C
arch/x86/kvm/mmu/mmu.c
static inline void kvm_init_shadow_page(void *page)
  {
  #ifdef CONFIG_X86_64
          int ign;

          asm volatile (
                  "rep stosq\n\t"
                  : "=c"(ign), "=D"(page)
                  : "a"(shadow_init_value), "c"(4096/8), "D"(page)
                  : "memory"
          );
  #else
          BUG();
  #endif
  }

  static int mmu_topup_shadow_page_cache(struct kvm_vcpu *vcpu)
  {
          struct kvm_mmu_memory_cache *mc = &vcpu->arch.mmu_shadow_page_cache;
          int start, end, i, r;

          if (vcpu->kvm->arch.gfn_shared_mask) {
                  r = kvm_mmu_topup_memory_cache(&vcpu->arch.mmu_private_sp_cache,
                                          ¦      PT64_ROOT_MAX_LEVEL);
                  if (r)
                          return r;
          }

          if (shadow_init_value)
                  start = kvm_mmu_memory_cache_nr_free_objects(mc);

          r = kvm_mmu_topup_memory_cache(mc, PT64_ROOT_MAX_LEVEL);
          if (r)
                  return r;

          if (shadow_init_value) {
                  end = kvm_mmu_memory_cache_nr_free_objects(mc);
                  for (i = start; i < end; i++)
                          kvm_init_shadow_page(mc->objects[i]);
          }
          return 0;
  }
```

This solves my question. We should set the #SVE bit in a empty page which is ready for some EPT violation, and these entry are not ready to be convert to #VE.  Due to if we have not build the ept page table, if the #VE triggered, it will trigger a ept violation again due to we haven't build any entry.