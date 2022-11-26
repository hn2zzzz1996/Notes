首先要打开几个Kernel Config：

```
// refer https://unix.stackexchange.com/questions/414655/not-syncing-vfs-unable-to-mount-root-fs-on-unknown-block0-0
CONFIG_VIRTIO_BLK=y
CONFIG_VIRTIO_PCI=y

CONFIG_VIRTIO_NET=y
```

编译好内核之后，在qemu启动参数里面加上相应的参数。

```C
-nic user,model=virtio-net-pci
```

然后启动虚拟机之后，在虚拟机里面用：

```
dhclient
```

如果连不上网，记得加个代理。