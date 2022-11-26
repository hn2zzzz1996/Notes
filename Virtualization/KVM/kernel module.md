# Kernel Module

Makefile模板：

```C
obj-m += hello.o

KDIR := /lib/modules/$(shell uname -r)/build
PWD := $(CURDIR)

all:
        make -C $(KDIR) M=$(PWD) modules
clean:
        make -C $(KDIR) M=$(PWD) clean

```

## Building Module

### The \_\_init and \_\_exit Macros

The `__init `macro causes the init function to be discarded and its memory freed once the init function finishes for built-in drivers, but not loadable modules.

There is also an `__initdata `which works similarly for init variables.

The `__exit `macro causes the omission of the function when the module is built into the kernel, and like `__init `, has no effect for loadable modules.

### 内核选项

打开`MODULE_FORCE_UNLOAD`内核编译选项，当该选项被打开的时候，可以使用`sudo rmmod -f module`命令强制卸载一个模块。该选项可以节约很多重启的时间当你在开发一个模块的时候。

## 预备知识

### 模块能够使用的函数

比如我们在模块中调用了`pr_info()`函数，但是在模块中并没有定义该函数，也没有相应的标准库进行链接。这是因为模块是`object`文件，它的符号是在`insmod`阶段被解析的，这些符号的定义是从内核自身来的，所以唯一能使用的外部符号就是被内核提供的。可以从`/proc/kallsyms`中看一下当前内核所导出的符号。
