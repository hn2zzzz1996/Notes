# Memory barrier

Compiler optimization and hardware reordering sometime may break out our code sequence, especially in I/O happens. 

A driver must ensure that no caching is performed and no read or write reordering takes place when accessing registers. So a memory barrier is needed.

The memory barrier make sure the operations execute in particular order and the hardware can see it.

There are some macros provided by Linux to cover all possible ordering needs:

* void barrier(void)

​		This function tells the compiler to insert a `software memory barrier` but has no effect on the hardware. Compiled code stores to memory all values that are currently modified and resident in CPU registers, and rereads them later when they are needed. A call to `barrier prevents compiler optimizations across the barrier` but leaves the hardware free to do its own reordering.

* void rmb(void);
* void read_barrier_depends(void);
* void wmb(void);
* void mb(void);

​		These functions insert `hardware memory barriers` in the compiled instruction flow; it's instantiation is platform dependent. An `rmb (read memory barrier)` guarantees that any reads appearing before the barrier are completed prior to the execution of any subsequent read. `wmb (write memory barrier)` guarantees ordering in write operations, and `mb` instruction guarantees both.

Also, there are versions of the barrier macros insert `hardware barriers` for SMP systems.

* void smp_rmb(void);
* void smp_read_barrier_depends(void);
* void smp_wmb(void);
* void smp_mb(void);

**Here is an typical usage of memory barriers in a device driver**:

```C
// The data should be read to by writeing the device controlling register
writel(dev->registers.addr, io_destination_address);
writel(dev->registers.size, io_size);
writel(dev->registers.operation, DEV_READ);
// The hardware should not prefetch the next instruction
wmb();
// Previous all write done, tells the device to do it
writel(dev->registers.control, DEV_GO);
```

Memory barriers will affect performance, so use it where they are really needed. Different types of barriers has different performance effect, so `use the most specific type`. Like, on x86 architecture, `wmb()` does nothing, since writes outside the processor are not reordered. But readers are reordered, so `mb()` is slower than `wmb()`.