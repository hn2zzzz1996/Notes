# Why not skip instruction when vmlaunch and vmresume cause vmexit in nested virtualization

First we should know some basic terminology.

The L0 is running on the native, the L1 is the guest running on the L0, the L2 is the guest running on the L1. That means the L2 is the VM hosted by L1.

But only one VMCS in the cpu. So the multiple VMCS will share the one cpu. Thus there has VMCS01, VMCS12 and VMCS02 on the cpu, and what is it?

Very simple, the number after the VMCS is the Level between two entity. Like the VMCS01 is the vmcs stores the information about the L0 and L1, L0 is the host and L1 is the guest. VMCS12 is the vmcs stores the information about the L1 and L2, L1 is the host and L2 is the guest.

So if we want to run L2, the L1 will first setup all the regions in the VMCS12, and in the last it will calls VMLAUNCH instruction to run the L2. And this instruction will trap into L0 to deal with it. In L0, it shouldn't skip the VMLAUNCH instruction, why? That's because the L1 has setup the `HOST_RIP`in the VMCS12, that's the real RIP when L2 returns to L1, the L1 resumes to run's position. So no need to skip the VMLAUNCH and update the `HOST_RIP` of the L1, this will cause something wrong. Ok, let's back to the L0 handle the VMLAUNCH, the L0 now has the VMCS01 on the cpu, it should switch to the VMCS02 that is ready for the L2 running, and it will sync all of the guest information in the VMCS12 to the VMCS02, that's all of the information that set up by L1 and L0 shouldn't modify it, just sync it. And after everything is ok, L0 will call VMLAUNCH itself, due to the guest state is the L2's state, it will run into the L2. That's the nested virtualization.

The same, when L2 meet something need to vmexit to L1. It will first return to the L0, and L0 switch the VMCS from VMCS02 to VMCS01, and sync all of guest fields in VMCS02 to VMCS12. And then back to the L1, and L1 and continue running to handle the vmexit of L2.

