The PCI device will always using MMIO to communicate with CPU. Every PCI device has the memory that on its board, and CPU will use MMIO to access these memory. So the PCI device MMIO will occupy a range of physical address space from CPU view. 

And there is no sharing of memory range between the RAM and the MMIO range. Because the platform firmware will initializes the PCI device's memory to use memory range above the memory range consumed by main memory (RAM) in the CPU memory space. For example, if the installed RAM size is 256MB, the PCI memory range starts right after the 256MB boundary up until the 4GB memory space boundary.

Also, There are several special ranges in the memory range consumed by PCI device memory above the installed RAM size -- installed RAM size is termed top of main memory (TOM). Like APIC will always mapped to `FEE00000H`. And we can see from the memblock:

```Shell
[    0.000000] BIOS-e820: [mem 0x00000000b0000000-0x00000000bfffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fed1c000-0x00000000fed1ffff] reserved
# The APCI MMIO should resist in here
[    0.000000] BIOS-e820: [mem 0x00000000feffc000-0x00000000feffffff] reserved
```

# PCI configuration register

How does the PCI bus protocol map PCI devices memory to the system address map?

The mapping is accomplished by using a set of PCI device registers called `BAR (base address register)`.

## Base address register (BAR)

BARs are part of the so-called `PCI configuration register`. Every PCI device must implement the PCI configuration register dictated by the PCI specification.

The PCI configuration register consists of 256 bytes of registers, from (byte) offset 00h to offset FFh. And it has two parts:

1. **PCI configuration register header**: first 64 bytes.
2. **device-specific PCI configuration register**: reset bytes.

And the BARs located in the PCI configuration register header, only this affect the memory mapping to system address map.

The `PCI configuration register header` has two type:

1. type 0: other PCI device.
2. type 1: The PCI-to-PCI bridge device.

The picture show below is the type 0 header and it includes the BAR.

![BAR](.\picture\BAR.png)

## A sample that how routes the access MMIO to the PCI device memory

In this example, the platform firmware initializes the video memory to be mapped in memory range 256 mb to 288 mb, because the video memory size is 32 mb—the first 256 mb is mapped to RAM. The platform firmware does so by configuring/initializing the video chip BARs to accept accesses in the 256 mb to 288 mb memory range.

![access_patch](.\picture\access_patch.png)

Here is the step that CPU want to access the AGP memory:

1. Suppose that an application (running inside the OS) requests texture data in the video memory that ultimately translates into physical address 11C0_0000h (284MB), the read transaction would go from the CPU to the northbridge.
2. The northbridge forwards the read request to a device whose memory range contains the requested address, 11C0_0000h. The northbridge logic checks all of its registers related to address mapping to find the device that match the address requested. In the Intel 815E chipset, address mapping of device on the AGP port is controlled by four registers, in the AGP/PCI bridge part of the chipset. Those four registers are MBASE, MLIMIT, PMBASE, and PMLIMIT. Any access to a memory address within the range of either MBASE to MLIMIT or PMBASE to PMLIMIT is forwarded to the AGP. It turns out the requested address (11C0_0000h) is within the PMBASE to PMLIMIT memory range. Note that initialization of the four address mapping registers is the job of the platform firmware.
3. Because the requested address (11C0_0000h) is within the PMBASE to PMLIMIT memory range, the northbridge forwards the read request to the AGP, i.e., to the video card chip.
4. The video card chip’s BARs setting allows it to respond to the request from the northbridge. The video card chip then reads the video memory at the requested address (at 11C0_0000h) and returns the requested contents to the northbridge.
5. The northbridge forwards the result returned by the video card to the CPU. After this step, the CPU receives the result from the northbridge and the read transaction completes.

## PCI bus base address registers (BAR) initialization

PCI BAR initialization is the job of the platform firmware.

### PCI BAR Formats

There are two types of BAR:

* I/O space BAR
* Maps to the CPU memory space.

### I/O space Format

![BAR IO Space Format](.\picture\BAR IO Space Format.png)

### Memory space Format

![BAR memory space format](.\picture\BAR memory space format.png)

## Reference

https://resources.infosecinstitute.com/topic/system-address-map-initialization-in-x86x64-architecture-part-1-pci-based-systems/