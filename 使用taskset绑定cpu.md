使用taskset 把vcpu pin 在物理cpu上。

```
taskset - set or retrieve a process's CPU affinity
```

```
# 使用十六进制表示cpu
0x00000001 is processor #0,

0x00000003 is processors #0 and #1,
```

```
 -c, --cpu-list
 Interpret mask as numerical list of processors instead of a
 bitmask. Numbers are separated by commas and may include
 ranges. For example: 0,5,8-11.

-p, --pid
Operate on an existing PID and do not launch a new task.
```

```bash
查看pid的cpu affinity
taskset -p pid

设置pid的cpu affinity
taskset -p mask pid
```

```bash
在启动qemu的时候直接设置qemu的cpu affinity
taskset 0x00000003 qemu-system-x86_64 –hda windows.img –m 512

qemu启动之后想要设置，先查找pid
[root@localhost ~]# ps -eo pid,comm | grep qemu
 7532 qemu-system-x86

查看当前cpu亲和性
[root@localhost ~]# taskset -p 7532
pid 7532's current affinity mask: 3

使用-c选项输出具体的cpu
[root@localhost ~]# taskset -c -p 7532
pid 7532's current affinity list: 0,1

更新一个进程的cpu亲和性
[root@localhost ~]# taskset -p 0x00000001 7532
pid 7532's current affinity mask: 3
pid 7532's new affinity mask: 1
```

QEMU的patch，未合入，添加了pin vcpu的功能

https://lists.gnu.org/archive/html/qemu-devel/2017-07/msg00018.html

https://groups.google.com/g/linuxkernelnewbies/c/qs5IiIA4xnw