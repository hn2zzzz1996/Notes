首先在HOST上执行如下命令（注意执行以下命令会断网）：

```bash
// 这个设置为本机上能上网的那个网卡的名字
$network = enp0s31f6
ip link add outbr0 type bridge
ip addr flush dev $network
ip link set $network master outbr0
ip link set dev outbr0 up
dhclient outbr0
```

然后qemu的启动参数如下所示：

```bash
[3:19 PM] Dong, Chuanxiao
-netdev tap,vhost=on,id=mytap,script=/home/hsq/my-qemu-ifup -device virtio-net-pci,netdev=mytap \
```

其中`my-qemu-ifup`是一个脚本，加上可执行权限，内容如下所示：

```bash
#!/bin/sh
#switch=$(brctl show| sed -n 2p |awk '{print $1}')
switch=outbr0
echo "switch is $switch"
echo "ifconfig up $1"
/sbin/ifconfig $1 0.0.0.0 up
/sbin/brctl addif ${switch} $1
```

最后启动虚拟机之后，运行一下`dhclient`命令，获取ip地址。