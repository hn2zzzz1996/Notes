```
[  116.310328]  ? vmx_secondary_exec_control+0x249/0x480
[  116.310335]  init_vmcs+0x112f/0x2100
[  116.310341]  vmx_vcpu_reset+0xa3e/0xf50
[  116.310346]  kvm_vcpu_reset+0x27b/0x5f0
[  116.310351]  kvm_arch_vcpu_create+0x238/0x300
[  116.310356]  kvm_vm_ioctl+0x87e/0xd90

[  116.310406] kvm: vmwrite failed: field=2 val=f2 err=0
```



这条路径也会重新加载MMU，会重新产生一个专为SMM的EPT。

```
#0  kvm_mmu_free_roots (vcpu=vcpu@entry=0xffff88814611c000, mmu=mmu@entry=0xffff88814611c290, roots_to_free=roots_to_free@entry=18446744073709551615) at arch/x86/kvm/mmu/mmu.c:3254
#1  0xffffffff81074e41 in kvm_mmu_unload (vcpu=vcpu@entry=0xffff88814611c000) at arch/x86/kvm/mmu/mmu.c:5125
#2  kvm_mmu_reset_context (vcpu=vcpu@entry=0xffff88814611c000) at arch/x86/kvm/mmu/mmu.c:5091
#3  0xffffffff81041512 in kvm_smm_changed (vcpu=vcpu@entry=0xffff88814611c000, entering_smm=entering_smm@entry=true) at arch/x86/kvm/x86.c:8129
#4  0xffffffff8104244e in enter_smm (vcpu=vcpu@entry=0xffff88814611c000) at arch/x86/kvm/x86.c:9605
#5  0xffffffff8104c9e3 in inject_pending_event (vcpu=vcpu@entry=0xffff88814611c000, req_immediate_exit=req_immediate_exit@entry=0xffffc90000a83e1f) at arch/x86/kvm/x86.c:9361
#6  0xffffffff81050608 in vcpu_enter_guest (vcpu=0xffff88814611c000) at arch/x86/kvm/x86.c:9992
#7  vcpu_run (vcpu=0xffff88814611c000) at arch/x86/kvm/x86.c:10288
#8  kvm_arch_vcpu_ioctl_run (vcpu=vcpu@entry=0xffff88814611c000) at arch/x86/kvm/x86.c:10498
#9  0xffffffff810293a3 in kvm_vcpu_ioctl (filp=<optimized out>, ioctl=44672, arg=<optimized out>) at arch/x86/kvm/../../../virt/kvm/kvm_main.c:3909
#10 0xffffffff813c811e in vfs_ioctl (arg=0, cmd=<optimized out>, filp=0xffff888142776700) at fs/ioctl.c:51
#11 __do_sys_ioctl (arg=0, cmd=44672, fd=12) at fs/ioctl.c:874
#12 __se_sys_ioctl (arg=0, cmd=44672, fd=12) at fs/ioctl.c:860
#13 __x64_sys_ioctl (regs=<optimized out>) at fs/ioctl.c:860
#14 0xffffffff81c92408 in do_syscall_x64 (nr=<optimized out>, regs=0xffffc90000a83f58) at arch/x86/entry/common.c:50
#15 do_syscall_64 (regs=0xffffc90000a83f58, nr=<optimized out>) at arch/x86/entry/common.c:80
#16 0xffffffff81e0007c in entry_SYSCALL_64 () at arch/x86/entry/entry_64.S:113

```



hb kvm_tdp_mmu_get_vcpu_root_hpa

display vcpu->arch.mmu->mmu_role.base

