0xffffffff82005ea0 __pkvm_indirect_thunk的地址

0xffffffffa00061e3

感觉还是模块加载的时候，符号表重定位的时候出了问题。

objdump -r 查看重定位表

这里以ip_tables.ko的重定位表为源头进行查询：

```C
[24] .rela.altinstructions
       RELA             0000000000000000  0000000000004f68  59
       0000000000000180 0000000000000018  23                8
       [0000000000000040]: INFO LINK
```

这里是加载ip_tables模块的时候重定位altinstructions的过程：

```C
[   40.765448] module: ip_tables: relocate symbols
[   40.765747] Applying relocate section 24 to 23
[   40.766033] type 2 st_value ffffffffa001b000 r_addend 4a loc ffffffffa001e1e9
[   40.766498] type 2 st_value ffffffffa001de3a r_addend 0 loc ffffffffa001e1ed
[   40.766948] type 2 st_value ffffffffa001b000 r_addend 70d loc ffffffffa001e1f5
[   40.767419] type 2 st_value ffffffffa001de3a r_addend 4 loc ffffffffa001e1f9
[   40.767872] type 2 st_value ffffffffa001b000 r_addend 1ae4 loc ffffffffa001e201
[   40.768340] type 2 st_value ffffffffa001de3a r_addend 8 loc ffffffffa001e205
[   40.768793] type 2 st_value ffffffffa001b000 r_addend 506 loc ffffffffa001e20d
    // 可以看到这里引用的地址为0xffffffff82006eef，这是__pkvm开头的地址空间
[   40.769253] type 2 st_value ffffffff82006eef r_addend 0 loc ffffffffa001e211
[   40.769702] type 2 st_value ffffffffa001b000 r_addend bbd loc ffffffffa001e219
[   40.770161] type 2 st_value ffffffff82006eef r_addend 0 loc ffffffffa001e21d
[   40.770615] type 2 st_value ffffffffa001b000 r_addend c08 loc ffffffffa001e225
[   40.771074] type 2 st_value ffffffff82003c31 r_addend 0 loc ffffffffa001e229
[   40.771530] type 2 st_value ffffffffa001b000 r_addend 1143 loc ffffffffa001e231
[   40.771999] type 2 st_value ffffffff82003c31 r_addend 0 loc ffffffffa001e235
[   40.772448] type 2 st_value ffffffffa001b000 r_addend 11d5 loc ffffffffa001e23d
[   40.772915] type 2 st_value ffffffff82003c31 r_addend 0 loc ffffffffa001e241

```

重定位表中出现了`ffffffff82006eef`，这是pkvm的retpoline的函数的地址。`__pkvm___x86_indirect_alt_call_rdx`.

```C[   40.792890] SMP alternatives: alt table ffffffffa001e1e9, -> ffffffffa001e249
[   40.793340] SMP alternatives: feat: 4*32+23, old: (0xffffffffa001b04a (ffffffffa001b04a) len: 5), repl: (ffffffffa001de3a, len: 4)
[   40.794079] SMP alternatives: ffffffffa001b04a:   old_insn: e8 51 58 62 e1
[   40.794573] SMP alternatives: ffffffffa001de3a:   rpl_insn: f3 0f b8 c7
[   40.795009] SMP alternatives: ffffffffa001b04a: final_insn: f3 0f b8 c7 90
[   40.795452] SMP alternatives: feat: 4*32+23, old: (0xffffffffa001b70d (ffffffffa001b70d) len: 5), repl: (ffffffffa001de3e, len: 4)
[   40.796190] SMP alternatives: ffffffffa001b70d:   old_insn: e8 8e 51 62 e1
[   40.796625] SMP alternatives: ffffffffa001de3e:   rpl_insn: f3 0f b8 c7
[   40.797044] SMP alternatives: ffffffffa001b70d: final_insn: f3 0f b8 c7 90
[   40.797478] SMP alternatives: feat: 4*32+23, old: (0xffffffffa001cae4 (ffffffffa001cae4) len: 5), repl: (ffffffffa001de42, len: 4)
[   40.798212] SMP alternatives: ffffffffa001cae4:   old_insn: e8 b7 3d 62 e1
[   40.798682] SMP alternatives: ffffffffa001de42:   rpl_insn: f3 0f b8 c7
[   40.799097] SMP alternatives: ffffffffa001cae4: final_insn: f3 0f b8 c7 90
[   40.799535] SMP alternatives: feat: !7*32+12, old: (0xffffffffa001b506 (ffffffffa001b506) len: 5), repl: (ffffffff82006eef, len: 5)
[   40.800276] SMP alternatives: ffffffffa001b506:   old_insn: e8 b5 85 fe e1
// 可以看到这里想要读取的地址为0xffffffff82006eef，与上面看到的一致
[   40.800800] SMP alternatives: ffffffff82006eef:   rpl_insn: 59 59 59 59 59
[   40.801293] SMP alternatives: ffffffffa001b506: final_insn: 59 59 e7 00 00
[   40.801739] SMP alternatives: feat: !7*32+12, old: (0xffffffffa001bbbd (ffffffffa001bbbd) len: 5), repl: (ffffffff82006eef, len: 5)
[   40.802504] SMP alternatives: ffffffffa001bbbd:   old_insn: e8 fe 7e fe e1
[   40.802948] SMP alternatives: ffffffff82006eef:   rpl_insn: 5d 5d 5d 5d 5d
[   40.803449] SMP alternatives: ffffffffa001bbbd: final_insn: 5d 5d e7 00 00
[   40.803899] SMP alternatives: feat: !7*32+12, old: (0xffffffffa001bc08 (ffffffffa001bc08) len: 5), repl: (ffffffff82003c31, len: 5)
[   40.804659] SMP alternatives: ffffffffa001bc08:   old_insn: e8 53 7e fe e1
[   40.805103] SMP alternatives: ffffffff82003c31:   rpl_insn: ff d0 90 90 90
[   40.805547] SMP alternatives: ffffffffa001bc08: final_insn: ff d0 90 90 90
[   40.805992] SMP alternatives: ffffffffa001bc08: [2:5) optimized NOPs: ff d0 0f 1f 00
[   40.806495] SMP alternatives: feat: !7*32+12, old: (0xffffffffa001c143 (ffffffffa001c143) len: 5), repl: (ffffffff82003c31, len: 5)
[   40.807257] SMP alternatives: ffffffffa001c143:   old_insn: e8 18 79 fe e1
[   40.807704] SMP alternatives: ffffffff82003c31:   rpl_insn: ff d0 90 90 90
[   40.808148] SMP alternatives: ffffffffa001c143: final_insn: ff d0 90 90 90
[   40.808593] SMP alternatives: ffffffffa001c143: [2:5) optimized NOPs: ff d0 0f 1f 00
[   40.809093] SMP alternatives: feat: !7*32+12, old: (0xffffffffa001c1d5 (ffffffffa001c1d5) len: 5), repl: (ffffffff82003c31, len: 5)
[   40.809850] SMP alternatives: ffffffffa001c1d5:   old_insn: e8 86 78 fe e1
[   40.810296] SMP alternatives: ffffffff82003c31:   rpl_insn: ff d0 90 90 90
[   40.810741] SMP alternatives: ffffffffa001c1d5: final_insn: ff d0 90 90 90
[   40.811192] SMP alternatives: ffffffffa001c1d5: [2:5) optimized NOPs: ff d0 0f 1f 00
```

(char*)((unsigned long)(&((struct kernel_symbol *)sym)->name_offset) + ((struct kernel_symbol *)sym)->name_offset)

 82059 ffffffff8273acf4 r __kstrtab___x86_indirect_alt_call_rdx
 82060 ffffffff8273acf4 r __pkvm___kstrtab___x86_indirect_alt_call_rdx

这是两个符号名，两个符号名地址一样，导致在`load_module`的时候去`resolve_symbols`解析符号的时候找到的符号不对，因为解析符号是通过对比符号名字去做的，但是这里的两个符号的符号名都在同一个地方，这也就导致`__pkvm___kstrtab___x86_indirect_alt_call_rdx`所对应的符号指向的函数名的`x86_indirect_alt_call_rdx`，这就导致符号解析的错误，使得模块指向了`pkvm`的地址空间，导致了内核的panic。

问题的原因在于两个符号的地址一样了。原因在于`EXPORT_SYMBOL`。

```C
#define __KSYMTAB_ENTRY(sym, sec)					\
	__ADDRESSABLE(sym)						\
	asm("	.section \"___ksymtab" sec "+" #sym "\", \"a\"	\n"	\
	    "	.balign	4					\n"	\
	    "__ksymtab_" #sym ":				\n"	\
	    "	.long	" #sym "- .				\n"	\
	    "	.long	__kstrtab_" #sym "- .			\n"	\
	    "	.long	__kstrtabns_" #sym "- .			\n"	\
	    "	.previous
```

可以看到在`EXPORT_SYMBOL`的时候，往`__ksymtab`section中写入了符号的信息，这里的符号信息对应于内核中的`struct kernel_symbol`结构体：

```C
struct kernel_symbol {
	int value_offset;
	int name_offset;
	int namespace_offset;
};
```

因为在这里export的时候，`name_offset`已经被计算好了，所以就算在之后修改了符号的名字，这里的`name_offset`依旧是不会变得，这也就导致了内核在连接的时候将两个不同的symbol名字都放在了相同的地址，因为地址一样了。