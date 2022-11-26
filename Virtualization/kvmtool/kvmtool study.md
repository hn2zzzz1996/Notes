# kvmtool study

The kvmtool initialize it's all submodule using a privileged list, like `core_init` is initialized before the `base_init`.

And here we list it's initialization sequenece:

1. core_init(): `kvm__init`.
2. base_init(): `kvm_cpu__init`, `kvm_ipc__init` , `ioeventfd__init`.
3. dev_base_init(): `pci__init`, `vfio__init`, `ioport__setup_arch`, `irq__init`.
4. dev_init(): `serial8250__init`, `term__init`.
5. virtio_dev_init(): `virtio_console__init`, `virtio_blk__init`, etc..
6. firmware_init(): `mptable__init`.

We can follow every function from start to end. And the entire flow is very clear.

## The relationship between term and serial8250

The `term` is used to get input from the current terminal, which is a host userspace input. After `term` get the input, it should pass the input to the `serial` which is a emulated input/output device for the guest. And `serial` will get input from `term` when every time `term` triggers `serial` and say hey, I have some input, please read it. And then `serial` read these data and store in it's own buffer and send an interrupt to the kvm guest, and the guest can handle this just like it has a real device.



