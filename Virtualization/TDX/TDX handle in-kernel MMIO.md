# TDX handle in-kernel MMIO

We can refer the commit 31d58c4e557d4 in upstream kernel to find a commit about how TDX handle in-kernel MMIO. And here is the link;

https://lore.kernel.org/all/20220405232939.73860-12-kirill.shutemov@linux.intel.com/T/#u

This commit 9a22bf6debbf5 imports #VE. (x86/traps: Add #VE support for TDX guest)

```
In non-TDX VMs, MMIO is implemented by providing the guest a mapping
which will cause a VMEXIT on access and then the VMM emulating the
instruction that caused the VMEXIT. That's not possible for TDX VM.

To emulate an instruction an emulator needs two things:

  - R/W access to the register file to read/modify instruction arguments
    and see RIP of the faulted instruction.

  - Read access to memory where instruction is placed to see what to
    emulate. In this case it is guest kernel text.

Both of them are not available to VMM in TDX environment:

  - Register file is never exposed to VMM. When a TD exits to the module,
    it saves registers into the state-save area allocated for that TD.
    The module then scrubs these registers before returning execution
    control to the VMM, to help prevent leakage of TD state.

  - TDX does not allow guests to execute from shared memory. All executed
    instructions are in TD-private memory. Being private to the TD, VMMs
    have no way to access TD-private memory and no way to read the
    instruction to decode and emulate it.

In TDX the MMIO regions are instead configured by VMM to trigger a #VE
exception in the guest.

Add #VE handling that emulates the MMIO instruction inside the guest and
converts it into a controlled hypercall to the host.

This approach is bad for performance. But, it has (virtually) no impact
on the size of the kernel image and will work for a wide variety of
drivers. This allows TDX deployments to use arbitrary devices and device
drivers, including virtio. TDX customers have asked for the capability
to use random devices in their deployments.

In other words, even if all of the work was done to paravirtualize all
x86 MMIO users and virtio, this approach would still be needed. There
is essentially no way to get rid of this code.

This approach is functional for all in-kernel MMIO users current and
future and does so with a minimal amount of code and kernel image bloat.

MMIO addresses can be used with any CPU instruction that accesses
memory. Address only MMIO accesses done via io.h helpers, such as
'readl()' or 'writeq()'.

Any CPU instruction that accesses memory can also be used to access
MMIO.  However, by convention, MMIO access are typically performed via
io.h helpers such as 'readl()' or 'writeq()'.

The io.h helpers intentionally use a limited set of instructions when
accessing MMIO.  This known, limited set of instructions makes MMIO
instruction decoding and emulation feasible in KVM hosts and SEV guests
today.

MMIO accesses performed without the io.h helpers are at the mercy of the
compiler.  Compilers can and will generate a much more broad set of
instructions which can not practically be decoded and emulated.  TDX
guests will oops if they encounter one of these decoding failures.

This means that TDX guests *must* use the io.h helpers to access MMIO.

This requirement is not new.  Both KVM hosts and AMD SEV guests have the
same limitations on MMIO access.

=== Potential alternative approaches ===

== Paravirtualizing all MMIO ==

An alternative to letting MMIO induce a #VE exception is to avoid
the #VE in the first place. Similar to the port I/O case, it is
theoretically possible to paravirtualize MMIO accesses.

Like the exception-based approach offered here, a fully paravirtualized
approach would be limited to MMIO users that leverage common
infrastructure like the io.h macros.

However, any paravirtual approach would be patching approximately 120k
call sites. Any paravirtual approach would need to replace a bare memory
access instruction with (at least) a function call. With a conservative
overhead estimation of 5 bytes per call site (CALL instruction),
it leads to bloating code by 600k.

Many drivers will never be used in the TDX environment and the bloat
cannot be justified.

== Patching TDX drivers ==

Rather than touching the entire kernel, it might also be possible to
just go after drivers that use MMIO in TDX guests *and* are performance
critical to justify the effrort. Right now, that's limited only to virtio.

All virtio MMIO appears to be done through a single function, which
makes virtio eminently easy to patch.

This approach will be adopted in the future, removing the bulk of
MMIO #VEs. The #VE-based MMIO will remain serving non-virtio use cases.
```