我们想要自己制作一个虚拟硬盘，这个虚拟硬盘能够被QEMU所识别并且能够成功的使用grub加载一个操作系统，下面是步骤。

首先要创建一个虚拟磁盘，直接使用dd创建：

```C
dd if=/dev/zero of=helloOS.img bs=512 count=204800
```

这会创建一个100M大小的虚拟磁盘。

然后首先给这个磁盘分区，不分区的话没有MBR，之后安装grub的时候就会报错。

我们给这个虚拟硬盘创建一个dos分区表，这会给`helloOS.img`创建一个MBR，然后创建一个主分区：

```C
# 相当于依次输入命令:o, n, p, 1, enter, enter, w
# 详细命令的作用可以查看fdisk的操作说明
printf "o\nn\np\n1\n\n\nw\n" | fdisk helloOS.img
```

然后将这个helloOS.img设置为一个`loop device`，`loop device`可以使得一个普通文件被当作块设备去访问。

```C
# --partscan 创建一个分区的loop设备
# $(losetup -f)获取第一个未被使用的loop设备(/dev/loop*)
LOOP_DEIVCE=$(losetup -f)
sudo losetup --partscan $(LOOP_DEIVCE) helloOS.img
```

然后我们就可以给之前创建的主分区建立文件系统：

```C
sudo mkfs.ext4 "${LOOP_DEIVCE}p1"
```

然后我们就可以在`${LOOP_DEIVCE}`这个设备上（也就是helloOS.img虚拟磁盘）安装grub了。

在安装grub的时候，因为需要指定grub安装的目录（否则就安装在自己pc上的根目录下了），所以还需先将`loop`设备挂载到一个文件夹下：

```shell
# 注意挂载的仅仅是之前创建的分区
mkdir image
sudo mount "${LOOP_DEIVCE}p1" ./image
```

最后将`grub`安装到虚拟磁盘中去：

```C
sudo grub-install --root-directory=$(pwd)/boot --no-floppy --target=i386-pc ${LOOP_DEVICE}
```

安装成功。

然后就可以在qemu中测试一下啦：

```shell
qemu-system-x86_64 -drive format=raw,file=helloOS.img
```

如果出现GRUB的命令就证明安装成功了。

```shell
grub>
```

