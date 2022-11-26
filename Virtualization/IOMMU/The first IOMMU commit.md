commit `ba39592764ed20c` is the first IOMMU commit.

```C
Author: Keshavamurthy, Anil S <anil.s.keshavamurthy@intel.com>
Date:   Sun Oct 21 16:41:49 2007 -0700

    Intel IOMMU: Intel IOMMU driver

    Actual intel IOMMU driver.  Hardware spec can be found at:
    http://www.intel.com/technology/virtualization

    This driver sets X86_64 'dma_ops', so hook into standard DMA APIs.  In this
    way, PCI driver will get virtual DMA address.  This change is transparent to
    PCI drivers.
```

## The first commit analysis

The first thing to do it `detect_intel_iommu`, this job is very simple, just to get the DMA Remapping Reporting Structure from the ACPI table, if get it, then there has IOMMU, else there donen't has IOMMU.

Then it will do the `intel_iommu_init`. This step will parse the dmar_table to init every dmar_unit in the platform.
