## /include/linux/percpu-defs.h

包含一些定义的宏，例如`DEFINE_PER_CPU`.

还有一些可以用的宏:

```C
// 偏移数组
#define per_cpu_ptr(ptr, cpu)						\
({									\
	__verify_pcpu_ptr(ptr);						\
	SHIFT_PERCPU_PTR((ptr), per_cpu_offset((cpu)));			\
})

// arch_raw_cpu_ptr 读取一个this_cpu_off的percpu变量加到当前ptr的地址上
#define raw_cpu_ptr(ptr)						\
({									\
	__verify_pcpu_ptr(ptr);						\
	arch_raw_cpu_ptr(ptr);						\
})

// ptr需要是一个指针
#define this_cpu_ptr(ptr) raw_cpu_ptr(ptr)

#define per_cpu(var, cpu)	(*per_cpu_ptr(&(var), cpu))

// 禁止抢占，获取当前per_cpu变量的宏
#define get_cpu_var(var)						\
(*({									\
	preempt_disable();						\
	this_cpu_ptr(&var);						\
}))

#define get_cpu_ptr(var)						\
({									\
	preempt_disable();						\
	this_cpu_ptr(var);						\
})
```

## /include/asm-generic/percpu.h

```C
/*
 * per_cpu_offset() is the offset that has to be added to a
 * percpu variable to get to the instance for a certain processor.
 *
 * Most arches use the __per_cpu_offset array for those offsets but
 * some arches have their own ways of determining the offset (x86_64, s390).
 */
#ifndef __per_cpu_offset
extern unsigned long __per_cpu_offset[NR_CPUS];

#define per_cpu_offset(x) (__per_cpu_offset[x])
#endif
```



## 各个宏的翻译

```C
this_cpu_read(pkvm_test_percpu);
{ u8 pfo_val__; asm volatile ("mov" "b " "%%""gs"":" "%" "[var]" ", " "%[val]" : [val] "=" "q" (pfo_val__) : [var] "m" (pkvm_test_percpu)); (typeof(pkvm_test_percpu))(unsigned long) pfo_val__; }

raw_cpu_read(pkvm_test_percpu);
{ u8 pfo_val__; asm ("mov" "b " "%%""gs"":" "%" "[var]" ", " "%[val]" : [val] "=" "q" (pfo_val__) : [var] "m" (pkvm_test_percpu)); (typeof(pkvm_test_percpu))(unsigned long) pfo_val__; }

per_cpu(pkvm_test_percpu, 1);
{ unsigned long __ptr; __asm__ ("" : "=r"(__ptr) : "0"((typeof(*((&(pkvm_test_percpu)))) *)((&(pkvm_test_percpu))))); (typeof((typeof(*((&(pkvm_test_percpu)))) *)((&(pkvm_test_percpu))))) (__ptr + (((__per_cpu_offset[(1)])))); }); }

// 获取当前cpu的指针的话，用当前变量的地址加上this_cpu_off这个per_cpu变量中存的值.
this_cpu_ptr(&pkvm_test_percpu);
(({ unsigned long tcp_ptr__; asm volatile("add " "%%""gs"":" "%" "1" ", %0" : "=r" (tcp_ptr__) : "m" (this_cpu_off), "0" (&pkvm_test_percpu)); (typeof(*(&pkvm_test_percpu)) *)tcp_ptr__; }); });

this_cpu_read_stable(pkvm_test_percpu);
({ u8 pfo_val__; asm("mov" "b " "%%""gs"":" "%" "P[var]" ", " "%[val]" : [val] "=" "q" (pfo_val__) : [var] "p" (&(pkvm_test_percpu))); (typeof(pkvm_test_percpu))(unsigned long) pfo_val__; });
```

