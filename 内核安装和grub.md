# 内核安装

## 编译image，安装modules

内核配置：

```bash
make menuconfig
```

还需修改.config文件中与签名相关的一项，否则编译失败：

```bash
CONFIG_SYSTEM_TRUSTED_KEYS="debian/canonical-certs.pem" 
                |
                v
CONFIG_SYSTEM_TRUSTED_KEYS=""
```

编译：

```bash
make

# 编译arm架构
make ARCH=arm CORSS_COMPILE=aarch64-none-linux-gnu-
```

安装kernel modules，将内核需要的模块安装到/lib/modules/{kernel version}下：

```bash
sudo make modules_install INSTALL_MOD_STRIP=1 # 减小/lib/modules的大小
INSTALL_MOD_PATH=~	# 指定安装路径
```

如果是安装到远程的机器，则需要将modules拷贝过去。

在/boot目录下生成initrd.img文件：

```bash
dracut initrd.img-{kernel version} {kernel version}
dracut initrd.img-5.11.0-25-generic 5.11.0-25.generic
```

拷贝bzImage文件：

src/arc/x86/boot/bzImage文件是内核的镜像文件，拷贝到/boot目录下，并改名成相应的内核名字

```bash
scp src/arc/x86/boot/bzImage vmlinuz-5.11.0-25.generic
```

## 修改grub

查看当前系统中的启动项：

```bash
grep -Ei 'submenu|menuentry ' /boot/grub/grub.cfg | sed -re "s/(.? )'([^']+)'.*/\1 \2/" | awk '{print i++ " " $0}'

# 一般得到如下结果
0 menuentry  Ubuntu
1 submenu  Advanced options for Ubuntu
2       menuentry  Ubuntu, with Linux 5.11.0-25-generic
3       menuentry  Ubuntu, with Linux 5.11.0-25-generic (recovery mode)
4       menuentry  Ubuntu, with Linux 5.11.0-16-generic
5       menuentry  Ubuntu, with Linux 5.11.0-16-generic (recovery mode)
6 menuentry  UEFI Firmware Settings
```

在ubuntu下，与grub相关的文件有如下几个：

* /boot/grub/grub.cfg: 由**update-grub**命令自动生成的文件，里面列出了启动目录项
* /etc/default/grub: grub启动的默认设置文件，其中的**GRUB_DEFAULT**项说明了哪一个项是默认启动项

**GRUB_DEFAULT**选项的两种写法：

* GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 5.11.0-16-generic" 
* GRUB_DEFAULT=0

数字的选项对应上面的输出，menuentry Ubuntu启动的是/boot目录下的vmlinuz文件，这是一个符号链接，链接到相应的内核文件上。

## grub-reboot 临时修改启动项

```
grub-reboot - set the default boot entry for GRUB, for the next boot only
```

```bash
grub-reboot [OPTION] MENU_ENTRY
```

