# How to Build A Custom Linux Kernel For Qemu

## Initial Ramdisk

Linux needs `initial ramdisk` to complete its boot process. The initial ramdisk is a filesystem image that is mounted temporarily at `/` while the Linux boots. The purpose of the initial ramdisk is to provide kernel modules and necessary utilities that might be needed in order to bring up the *real* root filesystem.

TODO: Complete the comparison with initrd and initramfs.

## A minimal initrd (initial ramdisk)

vim init.c:

```C
#include <stdio.h>
void main()
{
	printf("welcome to qemu test init\n");
	while(1);
}
```

静态链接编译：

```bash
gcc -static -o init init.c
```

将init制作成cpio：

```bash
echo init | cpio -o --format=newc > initramfs
```

使用qemu引导initramfs:

```bash
# -nographic不显示图形界面
# init 表示内核启动后引导的第一个应用程序
# console 一定得选对，否则没有信息
qemu-system-x86_64 -m 512M -kernel ./bzImage -initrd ./initramfs -nographic -append "init=/init console=ttyS0"
```

内核有关console的[说明](https://www.kernel.org/doc/html/latest/admin-guide/serial-console.html)：

```
console=device,options

device:         tty0 for the foreground virtual console
                ttyX for any other virtual console
                ttySx for a serial port
                lp0 for the first parallel port
                ttyUSB0 for the first USB serial device

options:        depend on the driver. For the serial port this
                defines the baudrate/parity/bits/flow control of
                the port, in the format BBBBPNF, where BBBB is the
                speed, P is parity (n/o/e), N is number of bits,
                and F is flow control ('r' for RTS). Default is
                9600n8. The maximum baudrate is 115200.
```

## Use busybox build initrd

busybox需要采用静态编译，这样才不会导致链接库的错误.

```bash
Settings  --->  
     [*] Build static binary (no shared libs)
```

```bash
# busybox源码目录下
make install
# 安装完的东西在 busybox/_install 目录下
```

编译了busybox之后，就可以制作initrd了。

使用脚本制作initrd：

```bash
#!/bin/sh
# Modify busybox folder to find _install folder
busybox_folder="../software/busybox-1.33.1"
rootfs="rootfs"

if [ ! -d $rootfs ]; then
        echo "create folder $rootfs"
        mkdir $rootfs
fi

cp -rf $busybox_folder/_install/* $rootfs/
cd $rootfs
mkdir -p proc sys dev etc etc/init.d

if [ -f etc/init.d/rcS ]; then
        rm etc/init.d/rcS
fi

cat > etc/init.d/rcS << EOF
#!/bin/sh
mount -t proc none /proc
mount -t sysfs nonde /sys
/sbin/mdev -s

# mount net9p fs
mkdir -pv /mnt/net9p
mount -t 9p -o trans=virtio net9p /mnt/net9p
EOF

chmod +x ./etc/init.d/rcS

if [ -f ../rootfs.img  ]; then
        rm ../rootfs.img
fi
find . | cpio -o --format=newc > ../rootfs.img
```

## qemu和主机共享文件夹

https://wiki.qemu.org/Documentation/9psetup

通过一个虚拟的文件系统，virtio技术实现。需要对内核也进行相应的配置才行：

```bash
# 新建一个my.config文件
CONFIG_NET_9P=y
CONFIG_NET_9P_VIRTIO=y
CONFIG_NET_9P_DEBUG=y (Optional)
CONFIG_9P_FS=y
CONFIG_9P_FS_POSIX_ACL=y
CONFIG_PCI=y
CONFIG_VIRTIO_PCI=y
# PCI and virtio options
CONFIG_PCI=y
CONFIG_VIRTIO_PCI=y
CONFIG_VIRTIO_BLK=y
CONFIG_PCI_HOST_GENERIC=y (only needed for the QEMU Arm 'virt' board)
```

然后通过内核源文件中自带的`scripts/kconfig/merge_config.sh`脚本修改内核`.config`：

```bash
# -m only merge the fragments, do not execute the make command
# -O dir to put generated output files.
# Using .config as base, mergin my.config, written to .config
scripts/kconfig/merge_config.sh -m -O . .config my.config
```

同时initrd中的rcS脚本中也要加入相应的脚本：

```bash
# mount net9p fs
mkdir -pv /mnt/net9p
mount -t 9p -o trans=virtio net9p /mnt/net9p
```

## 使用qemu运行内核

创建一个虚拟磁盘：

```bash
qemu-img create disk.raw 4G
mkfs.ext4 disk.raw
```

启动qemu：

```bash
# rdinit: Run the init process from the ramdisk.
qemu-system-x86_64 --enable-kvm -m 512M -smp 4 -kernel ./bzImage \
        -initrd ./rootfs.img -drive format=raw,file=./disk.raw \
        --fsdev local,id=share9p,path=./sharefs,security_model=none \
        -device virtio-9p-pci,fsdev=share9p,mount_tag=net9p \
        -nographic -append "console=ttyS0 root=/dev/sda rdinit=sbin/init"
```

--fsdev中的含义请查看：

https://wiki.qemu.org/Documentation/9psetup

local: Lets QEMU call the individual VFS functions directly on host.

id: Specifies identifier for this fsdev device.

path: The shared folder.

security_model: passthrough with nonroot.

mount_tag: Specifies the tag name to be used by the guest to mount this export point. (mount net9p /mnt/net9p)

## Reference

https://luomuxiaoxiao.com/?p=743

https://luomuxiaoxiao.com/?p=754

https://blog.csdn.net/weixin_38227420/article/details/88402738
