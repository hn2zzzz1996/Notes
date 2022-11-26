# APIC (Advanced programmable interrupt controller)

The local APIC delivers the interrupt to the processor core using an interrupt delivery protocol that has been set up through a group of APIC registers called the `LOCAL vector table` or `LVT`.

The local APIC handles interrupts from the other two interrupt sources (externally connected I/O devices and IPIs) through its IPI message handling facilities.

A processor can generate `IPIs` by programming the `interrupt command register (ICR)` in its local APIC. When the target processor receives an IPI message, its local APIC handles the message automatically (using information included in the message such as `vector number and trigger mode`).

Individual pins on the I/O APIC can be programmed to generate a specific interrupt vector when asserted.

## Detail of Local APIC

Software interacts with local APIC by reading and writing its registers. APIC registers are memory-mapped to a 4K region of processor's physical address space with an initial starting address of `FEE00000H`.

There is one MSR that can be programmed about local APIC is the `IA32_APIC_BASE` MSR.

### Local APIC Status and Location

The status and location of the local APIC are contained in the `IA32_APIC_BASE` MSR. MSR bit functions are described below:

* BSP flag, bit 8 - Indicates if the processor is the bootstrap processor (BSP).
* APIC Global Enable flag, bit 11 - Enable or disables the local APIC.
* APIC Base field, bits 12 through 35 - Specifies the base address of the APIC registers. Following a power-up or reset, the field is set to `FEE00000H` (3GB, 448M). And it can be set to relocate the local APIC registers.

## Local APIC ID

At power up, system hardware assigns a unique APIC ID to each local APIC on the system bus or APIC bus. In MP systems, the `local APIC ID` is also used as a `processor ID` by the BIOS and the operating system.

