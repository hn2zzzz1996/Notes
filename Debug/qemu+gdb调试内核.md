首先需要在编译内核的时候**打开调试信息**和**关闭ASLR**.

* 打开调试信息

```
# gdb config
CONFIG_GDB_SCRIPTS=y
CONFIG_DEBUG_INFO=y
```

## 避免时钟中断

使用qemu+gdb刚开始进行调试的时候，总是会遇到跳到时钟中断的情况，为了避免这种情况，我们打开如下如下内核编译选项：

```C
make menuconfig
> Processor type and features >
[*] Support x2apic

> Processor type and features > Linux guest support
[*]   Enable paravirtualization code
...
[*]   KVM Guest support (including kvmclock)
```

可以完美的解决在qemu中调试经常跳到时钟中断的情况。

原因还是因为遇到断点后，虽然虚拟CPU停止了，但是虚拟的时钟计时并没有停止，只是这个时候CPU在停止状态，中断无法触发。如果虚拟CPU继续开始执行，这时候执行的下一条命令就会在时钟中断处了，而不是希望的函数中的下一条指令。

## 关闭地址随机化

如果遇到gdb断点停不下来的情况，记得关闭地址随机化。

有两种方法可以关闭地址随机化：

1. 在内核启动命令行里面加入参数`nokaslr`

2. 在内核编译选项里面关闭地址随机化：

   ```C
    Symbol: RANDOMIZE_BASE [=n]                                                           
     │ Type  : bool                                                                       
     │ Defined at arch/x86/Kconfig:2096                                                   
     │   Prompt: Randomize the address of the kernel image (KASLR)                       
     │   Depends on: RELOCATABLE [=y]                                                     
     │   Location:
     │     -> Processor type and features
     │ (3)   -> Build a relocatable kernel (RELOCATABLE [=y]) 
   ```


## 避免函数优化

内核编译会优化很多东西，导致调试函数的时候看不到变量并且单步调试困难，在函数名前面加上:

```C
__attribute__((optimize("O0")))
```

来阻止gcc对函数进行优化，方便调试。
