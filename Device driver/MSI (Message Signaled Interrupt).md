# MSI (Message Signaled Interrupt)

## What are MSIs?

A Message Signaled Interrupt (MSI) is `a write from the device to a special address` which causes an interrupt to be received by the CPU.

The MSI first proposed in PCI 2.2. Later enhanced in PCI3.0 to allow each interrupt to be masked individually.

The MSI-X also introduced in PCI 3.0. It supports more interrupts per device than MSI and allows interrupts to be independently configured.

Devices may support both MSI and MSI-X, but only `one can be enabled at a time`.

## Why use MSIs?

There advantages than traditional pin-based interrupts.

1. Pin-based PCI interrupts are often shared by several devices (The INTx connect to which IOAPIC input is based on the MPtable or ACPI table). To support this, the kernel must call each interrupt handler associated with an interrupt, which reduced performance for the system. `MSIs are never shared`, so this problem cannot arise.

2. In the traditional pin-based interrupt, when a device writes data to memory, they raises a interrupt to tell the kernel that I write it, you can read. But it's possible that the interrupt may arrive before all the data has arrived in memory (Like a device behind PCI-PCI bridges). So the kernel interrupt handler need to read a register on the device which raised the interrupt to make sure the data has arrived in memory.

   Using MSIs can avoid this problem, due to the `interrupt-generating write can't pass the data writes`, so interrupt handler can easy to handler it (All data has arrived in memory when handler being triggered).

3. PCI device can only support a single pin-based interrupt per function. Often drivers have to query the device to find out what event has occurred, slowing down interrupt handling for the common case.

   With MSIs, a device can `support more interrupts`, allowing each interrupt to be specific to a different purpose.

## How to use MSIs

PCI devices are initialised to use pin-based interrupts. The device driver has to set up the device to use MSI or MSI-X.

### Include kernel support for MSIs

The kernel must built with `CONFIG_PCI_MSI`.

### Using MSI

Most of the hard work is done for the driver in the PCI layer. The driver simply has to request that the PCI layer set up the MSI capability for this device.

To automatically use MSI or MSI-X interrupt vectors, use the following function:

```C
int pci_alloc_irq_vectors(struct pci_dev *dev, unsigned int min_vecs,
              unsigned int max_vecs, unsigned int flags);
```

Allocates [min_vecs, max_vecs] number of interrupt vectors for a PCI device. It returns the number of vectors allocated or a negative error. If it can't allocate min_vecs number of vectors, return `-ENOSPC`.

The flags argument is used to specify which type of interrupt can be used by the device and the driver (PCI_IRQ_LEGACY, PCI_IRQ_MSI, PCI_IRQ_MSIX). A convenient short-hand (PCI_IRQ_ALL_TYPES) is also available to ask for any possible kind of interrupt.  If the PCI_IRQ_AFFINITY flag is set, pci_alloc_irq_vectors() will spread the interrupts around the available CPUs.





## Reference

https://docs.kernel.org/PCI/msi-howto.html

https://docs.oracle.com/cd/E19253-01/816-4854/interrupt-11/index.html