kernel panic发生在加载模块的时候，这个时候加载模块的时候copy了pkvm的代码，导致了EPT错误，从而导致内核panic。

目前发现的地址是`0xffffffff82005eef`，dump出来的结果是：

```C
(gdb) x/20 0xffffffff82005eef
0xffffffff82005eef <__pkvm___x86_indirect_alt_call_rdx>:        0x9090d2ff      0x90e2ff90      0xd6ff9090      0xff909090
0xffffffff82005eff <__pkvm___x86_indirect_alt_jmp_rsi+1>:       0x909090e6      0x9090d7ff      0x90e7ff90      0xd5ff9090
0xffffffff82005f0f <__pkvm___x86_indirect_alt_call_rbp+2>:      0xff909090      0x909090e5      0x90d0ff41      0xe0ff4190
0xffffffff82005f1f <__pkvm___x86_indirect_alt_jmp_r8+3>:        0xff419090      0x419090d1      0x9090e1ff      0x90d2ff41
```

可以看到里面的内容就是pkvm中编译出来的retpoline的代码。

在加载ip_tables.ko模块的时候，出现了内核panic。不仅仅是因为它导致的。

```C
#0  apply_alternatives (start=0xffffffff834046d0, end=0xffffffff83430bc0) at arch/x86/kernel/alternative.c:263
#1  0xffffffff831dc0d3 in alternative_instructions () at arch/x86/kernel/alternative.c:652
#2  0xffffffff831def9f in check_bugs () at arch/x86/kernel/cpu/bugs.c:149
#3  0xffffffff831d07b2 in start_kernel () at init/main.c:1118
#4  0xffffffff831cf54a in x86_64_start_reservations (real_mode_data=real_mode_data@entry=0x13cf0 <bts_ctx+3312> <error: Cannot access memory at address 0x13cf0>) at arch/x86/kernel/head64.c:525
#5  0xffffffff831cf5d7 in x86_64_start_kernel (real_mode_data=0x13cf0 <bts_ctx+3312> <error: Cannot access memory at address 0x13cf0>) at arch/x86/kernel/head64.c:506
#6  0xffffffff81000107 in secondary_startup_64 () at arch/x86/kernel/head_64.S:283
#7  0x0000000000000000 in ?? ()
```

