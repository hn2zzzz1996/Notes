# How to debug QEMU

We sometimes want to debug the QEMU process, but we always put the qemu command in a shell script. Like:

```C
		qemu-system-x86_64 \
                          -m 4G \
                          -machine q35 \
                          --enable-kvm \
                          -cpu host \
                          -nographic $SMP -kernel arch/x86/boot/bzImage \
                          -append "nokaslr noinitrd console=ttyS0 crashkernel=256M
                                          root=/dev/vda rootfstype=ext4 rw loglevel=8" \
                          -drive if=none,file=rootfs_debian_x86_64.ext4,id=hd0 \
                          -device virtio-blk-pci,drive=hd0 \
                          --fsdev local,id=kmod_dev,path=./sharefs,security_model=none \
                          -device virtio-9p-pci,fsdev=kmod_dev,mount_tag=kmod_mount\
                          -netdev user,id=mynet\
                          -device virtio-net-pci,netdev=mynet\
                          -drive if=pflash,format=raw,readonly,file=./bios.bin \
                          -device edu \
                          $DBG

```

We just need to add a command before the command above:

```C
gdb --directory ~/working/software/qemu-6.1.0/ --args \
    qemu-system-x86_64 ...
```

And now we can debug QEMU.