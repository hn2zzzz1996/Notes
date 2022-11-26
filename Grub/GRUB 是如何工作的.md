# GRUB 是如何工作的

GRUB知道文件系统和内核的可执行格式，所以只需要指出要加载的内核的名字以及在磁盘上的分区，GRUB就可以直接加载内核。

# 命名规范

GRUB有自己的对设备的表示方式，用来指定一个drive/partition（磁盘分区）。

首先来看个例子：

```C
(fd0)
```

GRUB要求用小括号包围设备名称，`fd`意味着这是一个软盘。数字0是`drive number`，从0开始。`(fd0)`意味着GRUB会使用整个软盘。

```C
(hd0,msdos2)
```

再看一个，`hd`意味着硬盘，数字`0`表示`drive number`，也就是第一个硬盘，`msdos`表示分区格式，数字`2`表示分区号，分区号从1开始。`(hd0,msdos2)`表示使用第一个硬盘的第二个分区。

# 使用grub-install安装GRUB

# grub2如何引导操作系统

grub2遵循multiboot2规范，使用grub2。操作系统的开发者不必操心模式切换的开发，因为在将控制权从bootloader传递到操作系统之前，bootloader就已经进入到32位的保护模式中了，因为bootloader需要将操作系统加载到高于1MB地址的位置，而这在实模式中是做不到的（实模式无法访问高于1MB的地址），这大大减轻了操作系统开发者的工作。

因为操作系统可能有各种各样的格式，bootloader不可能去解析每一个操作系统的格式，所以Multiboot2定义了OS镜像必须包含一个`Multiboot2 header`，bootloader去解析这个头就好。