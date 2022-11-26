# What is anonymous inode

In linux kernel, there is a function `anon_inode_getfd` that can be used to create an anonymous inode and return a `fd` to user.

We can use it like:

```C
fd = anon_inode_getfd("kvm", &kvm_fops, my_data_pointer, O_RDONLY | O_CLOEXEC);
```

Now the user has an `fd` with associated arbitrary `file_operations`, when the `fd` is closed, everything is freed.

The method is useful, e.g. if you want to have multiple `read` syscalls, but don't want to create multiple device files, which further pollutes `/dev`. You can just use `anon_inode_getfd` to get a fd which associated with a `file_operations`, and what you do on the fd is related to the `file_operations`.

## An example

The `anon_inode_getfd` is widely used in KVM, when created vm and vcpu, the KVM will use this API to create fd and bind it with `vcpu_ops` and `vm_ops`, thus the user space can use ioctl to communicate with KVM.