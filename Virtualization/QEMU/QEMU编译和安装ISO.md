## 预备工作

```bash
sudo apt-get install libpixman-1-dev libglib2.0-dev libcap-ng-dev libattr1-dev ninja-build
```

安装qemu图形界面库：

```bash
sudo apt-get install libsdl2-dev
```

然后编译安装：

```bash
wget https://download.qemu.org/qemu-5.2.0.tar.xz
tar xvJf qemu-5.2.0.tar.xz
cd qemu-5.2.0
mkdir build && cd build
../configure --enable-kvm --enable-debug --target-list=x86_64-softmmu,aarch64-softmmu
# --enable-debug 打开调试信息
make -j4
sudo make install
```

## 安装ISO文件

1. 创建磁盘文件

   ```bash
   qemu-img create -f qcow2 qemu_disk.img 20G
   ```

2. 安装虚拟机

   ```bash
   qemu-system-x86_64 --enable-kvm -m 2G -smp 2 -boot order=dc -hda qemu_disk.img -cdrom ubuntu-14.04.iso -sdl
   
   boot order 的选项如下，代表系统的启动顺序，每种方式都有一个对应的字母缩写
   -boot [order=drives][,once=drives][,menu=on|off]
   　　'drives': floppy (a), hard disk (c), CD-ROM (d), network (n)
   ```

3. 安装完毕后，从虚拟硬盘启动虚拟机

   ```bash
   # 修改启动顺序即可
   -boot order=cd
   ```

   

