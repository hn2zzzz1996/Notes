# Retpoline

"Retpoline" sequences are a software construct which allow indirect branches to be isolated from speculative execution. 

这会被用来保护敏感的二进制程序（比如操作系统和hypervisor的实现），免受针对其间接分支的分支目标注入攻击。

新的GCC编译选项：`-mindirect-branch=thunk-extern`，并且提供相应的thunks.

```
Currently our retpolines consist of 2 symbols,
__x86_indirect_thunk_\reg, which is the compiler target, and
__x86_retpoline_\reg, which is the actual retpoline.
```



开启RETPOLINE后pkvm编译错误：

```C
ld: arch/x86/kvm/vmx/pkvm/pkvm.o: in function `pte_for_address':
/home/hsq/working/src/pkvm/pkvm-ia/arch/x86/kvm/vmx/pkvm/rt/ept.c:73: undefined reference to `__pkvm___x86_indirect_thunk_r13'
ld: arch/x86/kvm/vmx/pkvm/pkvm.o:(.altinstructions+0x4): undefined reference to `__pkvm___x86_indirect_alt_call_r13'
```

EPT violation



## 源码分析

首先要知道，在开启了GCC的编译选项`-mindirect-branch=thunk-extern`之后，如果遇到类似于`call *rax`或者`jmp *rax`之类的间接调用，会将该指令替换成`CALL __x86_indirect_thunk_\reg`指令，之后会通过内核中的`objtool`工具重写该指令，会将该指令重写为：

```C
commit 9bc0bb5072
ALTERNATIVE "call __x86_indirect_thunk_\reg",
            "call *%reg", ALT_NOT(X86_FEATURE_RETPOLINE)
```

该指令所表达的意思就是该地址当前的指令为`call __x86_indirect_thunk_\reg`，如果满足条件`ALT_NOT(X86_FEATURE_RETPOLINE)`，也就是没有`X86_FEATURE_RETPOLINE`这个FEATURE，那么就将当前指令替换成为`call *%reg`。

要实现这种替换，是通过`objtool`直接写入`.altinstruction`这个`section`中完成的，这个`section`中保存的每一个项都是`struct alt_instr`结构体，这个结构体中的成员如下所示：

```C
struct alt_instr {
	s32 instr_offset;	/* original instruction */
	s32 repl_offset;	/* offset to replacement instruction */
	u16 cpuid;		/* cpuid bit set for replacement */
	u8  instrlen;		/* length of original instruction */
	u8  replacementlen;	/* length of new instruction */
} __packed;
```

很明显可以看到`struct alt_instr`中记录了原来的指令的位置和长度，用于替换的指令的位置和长度，以及通过检测cpu的feature来决定是否替换该指令的`cpuid`变量。

替换指令的操作在`apply_alternatives`函数中实现，该函数的位置在`arch/x86/kernel/alternative.c`中，稍后再做解释。

### retpoline.S

现在我们把目光转到`retpoline.S`文件中，该文件的位置在`arch/x86/lib/retpoline.S`，该函数中实现了有关于`RETPOLINE`的主要信息。

首先要注意的是在retpoline.S的开头有一个关于section的声明：

```C
.section .text.__x86.indirect_thunk
```

这也就保证了该函数中定义的函数都会在`.text.__x86.indirect_thunk`这个section中出现。

注意到在该文件中定义了`__x86_indirect_thunk_\reg`函数，该函数名称刚好于gcc所生成的函数名一致：

```C
SYM_FUNC_START(__x86_indirect_thunk_\reg)

	ALTERNATIVE_2 __stringify(ANNOTATE_RETPOLINE_SAFE; jmp *%\reg), \
		      __stringify(RETPOLINE \reg), X86_FEATURE_RETPOLINE, \
		      __stringify(lfence; ANNOTATE_RETPOLINE_SAFE; jmp *%\reg), X86_FEATURE_RETPOLINE_AMD

SYM_FUNC_END(__x86_indirect_thunk_\reg)
```

这里不看AMD的情况，首先看`ANNOTATE_RETPOLINE_SAFE`，这是为了防止`objtool`报错，然后紧接着的`jmp`指令就是在`__x86_indirect_thunk_\reg`这个函数当中实现的原始指令，如果有CPU有`X86_FEATURE_RETPOLINE`，那么则将指令替换为`RETPOLINE \reg`，这里的`RETPOLINE`也是个宏，实现如下：

```
.macro RETPOLINE reg
	ANNOTATE_INTRA_FUNCTION_CALL
	call    .Ldo_rop_\@
.Lspec_trap_\@:
	UNWIND_HINT_EMPTY
	pause
	lfence
	jmp .Lspec_trap_\@
.Ldo_rop_\@:
	mov     %\reg, (%_ASM_SP)
	UNWIND_HINT_FUNC
	ret
.endm
```

这也就是标准的`RETPOLINE`的一个实现，`ANNOTATE_INTRA_FUNCTION_CALL`是为了防止报错的声明，因为`call .Ldo_rop_\@`所call到的地方不是一个函数的开头，需要加上该声明表示没问题。

再回顾一下之前的指令替换，函数名刚好对应上了：

```C
ALTERNATIVE "call __x86_indirect_thunk_\reg",
            "call *%reg", ALT_NOT(X86_FEATURE_RETPOLINE)
```

对于这个实现，每一个有`indirect call`的`.o`文件都会生成两个section，分别是`.altinstruction`和`.altinst_replacement`。其中：

* .altinstruction：其中保存的是`struct alt_instr`信息；
* .altinst_replacement：其中保存的就是要替换的指令。

如果是上面这种实现，则会产生很多重复的指令内容保存在`.altinst_replacement `section中，为了避免这种情况，使用一个统一的函数符号`__x86_indirect_alt_`*代替这些离散的指令。

这些函数也定义在该文件中：

```C
/*
 * This generates .altinstr_replacement symbols for use by objtool. They,
 * however, must not actually live in .altinstr_replacement since that will be
 * discarded after init, but module alternatives will also reference these
 * symbols.
 *
 * Their names matches the "__x86_indirect_" prefix to mark them as retpolines.
 */
.macro ALT_THUNK reg

	.align 1

SYM_FUNC_START_NOALIGN(__x86_indirect_alt_call_\reg)
	ANNOTATE_RETPOLINE_SAFE
1:	call	*%\reg
2:	.skip	5-(2b-1b), 0x90 // 填充到5个byte
SYM_FUNC_END(__x86_indirect_alt_call_\reg)

STACK_FRAME_NON_STANDARD(__x86_indirect_alt_call_\reg)

SYM_FUNC_START_NOALIGN(__x86_indirect_alt_jmp_\reg)
	ANNOTATE_RETPOLINE_SAFE
1:	jmp	*%\reg
2:	.skip	5-(2b-1b), 0x90
SYM_FUNC_END(__x86_indirect_alt_jmp_\reg)

STACK_FRAME_NON_STANDARD(__x86_indirect_alt_jmp_\reg)

.endm
```

这里的函数中也就是简单的`call *%\reg`指令，之所以要填充到5个byte，是因为`call function`就是5个byte。

可以从注释中看到，objtool会主动的产生这些符号，具体的后面看下objtool的源码。

```C
/tools/objtool/arch/x86/decode.c
int arch_rewrite_retpolines(struct objtool_file *file)
{
	struct instruction *insn;
	struct reloc *reloc;
	struct symbol *sym;
	char name[32] = "";

    // 对于每一个indirect call指令
	list_for_each_entry(insn, &file->retpoline_call_list, call_node) {

		if (insn->type != INSN_JUMP_DYNAMIC &&
		    insn->type != INSN_CALL_DYNAMIC)
			continue;
		
        // 当前指令不在.text.__x86.indirect_thunk sectoon中
		if (!strcmp(insn->sec->name, ".text.__x86.indirect_thunk"))
			continue;

		reloc = insn->reloc;
		
        // 构造出__x86_indirect_alt_*的symbol
		sprintf(name, "__x86_indirect_alt_%s_%s",
			insn->type == INSN_JUMP_DYNAMIC ? "jmp" : "call",
			reloc->sym->name + 21);
		
        // 当前.o文件要引用该symbol
		sym = find_symbol_by_name(file->elf, name);
		if (!sym) {
			sym = elf_create_undef_symbol(file->elf, name);
			if (!sym) {
				WARN("elf_create_undef_symbol");
				return -1;
			}
		}
		
        // 将信息写入到.altinstructions中
        // 因为insn和sym的地址都还未确定，所以要在reloc节中写入要重定位的地方
		if (elf_add_alternative(file->elf, insn, sym,
					ALT_NOT(X86_FEATURE_RETPOLINE), 5, 5)) {
			WARN("elf_add_alternative");
			return -1;
		}
	}

	return 0;
}
```

