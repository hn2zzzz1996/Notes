# Grub

[grub manual](https://www.gnu.org/software/grub/manual/grub/grub.html#Introduction)

## Overview

a `boot loader`是计算机启动时运行的第一个软件程序。它的职责是加载kernel并且将控制权传递给操作系统的kernel，例如Linux。

## Installation

GRUB 自带boot image，被放置在`/usr/lib/grub/<cpu>-<platform>`目录下（对于基于BIOS的机器来说是/usr/lib/grub/i386-pc），该目录通常被称为`image directory`。

`boot direcotry`通常是`/boot`目录。???

Loop Device 是一个伪设备，它可以把一个普通文件当作block device去访问（使用 losetup来绑定一个普通文件与块设备）。



## Writing configuration file

`grub-mkconfig`会生成`grub.cfg`文件，它会寻找合适的kernel并且给他们生成相应的目录。

文件`/etc/default/grub`控制着`grub-mkconfig`的行为。