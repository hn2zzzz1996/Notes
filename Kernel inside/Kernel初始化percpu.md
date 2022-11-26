# Kernel初始化percpu

首先我们来看Kernel链接文件中对percpu的链接方式：

```C
// /arch/x86/kernel/vmlinux.lds.S	
/*
	 * percpu offsets are zero-based on SMP.  PERCPU_VADDR() changes the
	 * output PHDR, so the next output section - .init.text - should
	 * start another segment - init.
	 */
	PERCPU_VADDR(INTERNODE_CACHE_BYTES, 0, :percpu)
	ASSERT(SIZEOF(.data..percpu) < CONFIG_PHYSICAL_START,
	       "per-CPU data too large - increase CONFIG_PHYSICAL_START")
```

这里用宏定义的方式链接了percpu段，下面我们看一下这个宏具体的定义：

```C
// /include/asm-generic/vmlinux.lds.h
#define PERCPU_VADDR(cacheline, vaddr, phdr)				\
	__per_cpu_load = .;						\
	.data..percpu vaddr : AT(__per_cpu_load - LOAD_OFFSET) {	\
		PERCPU_INPUT(cacheline)					\
	} phdr								\
	. = __per_cpu_load + SIZEOF(.data..percpu);
```

首先看一下这里提供的三个参数：

* cacheline: 指定的是cacheline的大小，用来对其的，防止不同的`subsection`之间有cache的共享。提高运行时的效率。

* vaddr: 指定percpu这个段运行时的虚拟地址是多少，这里将其设置成了0。也就是说percpu变量的地址都是从0开始的。

* phdr：这是指将percpu包含在那个PHDR里，也就是程序的节头，这里设置成了`:percpu`，也就是percpu变量单独占用了一个PHDR，就是`percpu`，这个可以在`vmlinux.lds`中看到：

  ```C
  PHDRS {
   text PT_LOAD FLAGS(5);
   data PT_LOAD FLAGS(6);
   percpu PT_LOAD FLAGS(6);
   init PT_LOAD FLAGS(7);
   note PT_NOTE FLAGS(0);
  }
  ```

然后来看一下首先定义了**__per_cpu_load**这个变量，并且设置成为了当前编译的虚拟地址，也就是说这个地址是percpu这个段实际被加载到的虚拟地址。并且还有一点要注意的是，因为percpu的变量都是全局变量，其初始值都是0，但是因为没有放到bss段而是单独放到了percpu段，所以其占据了文件的大小，这一点我们可以通过查看vmlinux的section来验证：

```shell
readelf -t vmlinux
```

```shell
[20] .data..percpu
       PROGBITS         0000000000000000  0000000002000000  0
       000000000002f000 0000000000000000  0                 4096
       [0000000000000003]: WRITE, ALLOC
```

```Shell
readelf -x 20 vmlinux
```

```C
Hex dump of section '.data..percpu':
  0x00000000 00000000 00000000 00000000 00000000 ................
  0x00000010 00000000 00000000 00000000 00000000 ................
  0x00000020 00000000 00000000 00000000 00000000 ................
  0x00000030 00000000 00000000 00000000 00000000 ................
  0x00000040 00000000 00000000 00000000 00000000 ................
```

然后我们继续看一下如下的部分：

```C
	.data..percpu vaddr : AT(__per_cpu_load - LOAD_OFFSET) {	\
		PERCPU_INPUT(cacheline)					\
	} phdr	
```

首先是定义了`.data..percpu`的section，然后虚拟地址为`vaddr`，这是传进来的参数，为0，然后冒号后边的地址的意思是加载地址。然后又是一个宏`PERCPU_INPUT(cacheline)`，最后是`phdr`表明这个section放到`phdr`这个程序节头里边。

下边来看一下`PERCPU_INPUT`这个宏定义：

```C
// include/asm-generic/vmlinux.lds.h
#define PERCPU_INPUT(cacheline)						\
	__per_cpu_start = .;						\
	*(.data..percpu..first)						\
	. = ALIGN(PAGE_SIZE);						\
	*(.data..percpu..page_aligned)					\
	. = ALIGN(cacheline);						\
	*(.data..percpu..read_mostly)					\
	. = ALIGN(cacheline);						\
	*(.data..percpu)						\
	*(.data..percpu..shared_aligned)				\
	PERCPU_DECRYPTED_SECTION					\
	__per_cpu_end = .;
```

可以看到这里将很多个不同的`percpu`的section都包含了进来，首先要关注的是两个变量：

* **__per_cpu_start**: 这个变量就等于之前传进来的`vaddr`，也就是0.
* **__per_cpu_end**：这个变量标识了percpu这个section的结束地址，相当于percpu section的大小。

然后我们来看一下这个percpu的subsection的排列顺序：

`.data..percpu..first`必须排在第一个，为什么？

首先我们要知道有哪些变量的定义是在这个section里面的，可以看到，**DECLARE_PER_CPU_FIRST**这个宏定义的变量就在`.data..percpu..first`这个section里面。

而使用这个宏定义的只有一个变量，那就是`fixed_percpu_data`：

```C
// arch/x86/include/asm/processor.h
struct fixed_percpu_data {
	/*
	 * GCC hardcodes the stack canary as %gs:40.  Since the
	 * irq_stack is the object at %gs:0, we reserve the bottom
	 * 48 bytes of the irq stack for the canary.
	 *
	 * Once we are willing to require -mstack-protector-guard-symbol=
	 * support for x86_64 stackprotector, we can get rid of this.
	 */
	char		gs_base[40];
	unsigned long	stack_canary;
};

DECLARE_PER_CPU_FIRST(struct fixed_percpu_data, fixed_percpu_data) __visible;
```

这个变量一定是存在于整个percpu section的最前端的，也就是占据了虚拟地址0的位置。

另外，还定义了一个额外的变量：

```C
// arch/x86/kernel/vmlinux.lds.S
#define INIT_PER_CPU(x) init_per_cpu__##x = ABSOLUTE(x) + __per_cpu_load
INIT_PER_CPU(fixed_percpu_data);
```

展开来就是`init_per_cpu__fixed_percpu_data`，通过`fixed_percpu_data`的地址0加上percpu section实际加载的虚拟地址`__per_cpu_load`，也就是计算出来了percpu变量`fixed_percpu_data`刚开始加载到内存中的地址。

在刚开始运行内核的时候，需要去设置gs寄存器的基地址，gs寄存器被用来作为percpu变量的基地址：

```C
// arch/x86/kernel/head_64.S
	/* Set up %gs.
	 *
	 * The base of %gs always points to fixed_percpu_data. If the
	 * stack protector canary is enabled, it is located at %gs:40.
	 * Note that, on SMP, the boot cpu uses init data section until
	 * the per cpu areas are set up.
	 */
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr
```

可以看到，这里将`initial_gs`写到了GS的MSR寄存器中，因为GS的基地址需要保存在MSR中，查看`initial_gs`的定义：

```C
SYM_DATA(initial_gs,	.quad INIT_PER_CPU_VAR(fixed_percpu_data))
```

发现其就指向`INIT_PER_CPU_VAR(fixed_percpu_data)`，也就是percpu load上来的地址。

这是内核在初始化的时候使用的percpu变量的地址，这个时候percpu区域还没有初始化，所以读取出来的任何的percpu变量都是0.
