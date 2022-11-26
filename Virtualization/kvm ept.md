## kvm ept

```C
[    1.712611] smp: Bringing up secondary CPUs ...
[    1.717058] x86: Booting SMP configuration:
[   27.650428] kvm: setting pte gfn: 0, pfn: 148c00, level 1
[   27.651829] kvm: vcpu ffff8881460e0000 reload ept prev ffffffffffffffff, now 144166000
[   27.652678] kvm: setting pte gfn: 9a, pfn: 148c9a, level 1
[   27.653211] kvm: setting pte gfn: 9e, pfn: 148c9e, level 1
[   27.654064] kvm: setting pte gfn: 99, pfn: 148c99, level 1
[   27.654581] kvm: setting pte gfn: 9d, pfn: 148c9d, level 1
[   27.655098] kvm: vcpu ffff8881460e0000 reload ept prev ffffffffffffffff, now 144158000
[   27.655859] kvm: setting pte gfn: 9a, pfn: 148c9a, level 1
[   27.656354] kvm: setting pte gfn: 9d, pfn: 148c9d, level 1
[   27.656880] kvm: vcpu ffff8881460e0000 reload ept prev ffffffffffffffff, now 144188000
[   27.661564] kvm: setting pte gfn: 9a, pfn: 148c9a, level 1
[   27.662112] kvm: vcpu ffff8881460e0000 reload ept prev ffffffffffffffff, now 1441a3000
[   27.663136] kvm: vcpu ffff8881460e0000 reload ept prev ffffffffffffffff, now 1441a3000
```

## pkvm ept

```
[    1.356868] smp: Bringing up secondary CPUs ...
[    1.361320] x86: Booting SMP configuration:
[   16.046195] kvm: setting pte gfn: 0, pfn: 148c00, level 1
[   16.047561] kvm: vcpu ffff8881460e0000 reload ept prev ffffffffffffffff, now 14413b000
[   16.048613] kvm: setting pte gfn: 9a, pfn: 148c9a, level 1
[   16.049134] kvm: setting pte gfn: 9e, pfn: 148c9e, level 1
[   16.049840] kvm: setting pte gfn: 99, pfn: 148c99, level 1
[   16.050354] kvm: setting pte gfn: 9d, pfn: 148c9d, level 1
[   16.050889] kvm: vcpu ffff8881460e0000 reload ept prev ffffffffffffffff, now 144130000
[   16.051958] kvm: gfn 39e10 has a mapping!
[   16.052314] kvm: gfn 39e10 has a mapping!
[   16.052665] kvm: gfn 39e10 has a mapping!
[   16.053019] kvm: gfn 39e10 has a mapping!
[   16.053373] kvm: gfn 39e10 has a mapping!
[   16.053726] kvm: gfn 39e10 has a mapping!
[   16.054079] kvm: gfn 39e10 has a mapping!
[   16.054432] kvm: gfn 39e10 has a mapping!
[   16.054821] kvm: gfn 39e10 has a mapping!
[   16.055179] kvm: gfn 39e10 has a mapping!
[   16.055858] kvm: setting pte gfn: 9a, pfn: 148c9a, level 1
[   16.056330] kvm: setting pte gfn: 9d, pfn: 148c9d, level 1
[   16.056829] kvm: vcpu ffff8881460e0000 reload ept prev ffffffffffffffff, now 14422c000
[   16.057585] kvm: setting pte gfn: 9a, pfn: 148c9a, level 1
[   16.058099] kvm: vcpu ffff8881460e0000 reload ept prev ffffffffffffffff, now 144200000
[   16.058890] kvm: gfn 9c has a mapping!
[   16.059506] kvm: gfn 39e10 has a mapping!
```

感觉像是少了mmio的映射，就是少了mmio的映射