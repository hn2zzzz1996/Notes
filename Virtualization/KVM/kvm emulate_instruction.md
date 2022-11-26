# kvm emulate_instruction

首先来看一下x86指令的格式：

```
instruction prefixes | opcode | ModR/M | SIB | displacement | immediate
```

x86指令由多个部分组成：

* 指令前缀（instruction prefixes），比如lock前缀，rep前缀等。
* 操作码(opcode)，是指令的索引，占1~3字节，所有的指令都必须由opcode，其他的5个域都是可选的。
* 操作码指定了寄存器以及嵌入在指令中的立即数，至于是在哪个寄存器、在内存的哪个位置、使用哪个寄存器索引内存位置，则由ModR/M和SIB通过编码查表得方式确定。
* SIB是Scale-Index-Base.
* displacement，偏移。
* immediate，立即数。

> REX prefixes are used to generate 64-bit operand sizes or to reference registers R8-R15.

## MMIO emulation

MMIO的区域一般来说会被EPT设置成`misconfig`，当发现访问到`misconfig`的页时，会发生vmexit，处理函数为`handle_ept_misconfig`。调用流程如下所示：

```C
handle_ept_misconfig
    -> kvm_mmu_page_fault
    	-> x86_emulate_instruction
    		-> x86_decode_emulated_instruction
    			-> init_emulate_ctxt
    			-> x86_decode_insn
    		-> x86_emulate_insn
```

在`x86_decode_insn`这个函数中会进行`instruction decode`的操作。在这个函数里，首先会做的就是预取指令。

```C
int x86_decode_insn(struct x86_emulate_ctxt *ctxt, void *insn, int insn_len, int 					emulation_type)
{
    int mode = ctxt->mode;
    
    if (insn_len > 0)
        memcpy(ctxt->fetch.data, insn, insn_len);
    else {
        // 从handle_ept_misconfig调用过来的insn为NULL，insn_len也为0，所以调用这个函数
        // 预取一部分的指令
        rc = __do_insn_fetch_bytes(ctxt, 1);
        if (rc != X86EMUL_CONTINUE)
            goto done;
    }
    
    switch(mode) {
        // 首先检查当前在什么模式下，决定操作数的长度
        case X86EMUL_MODE_PROT64:
            def_op_bytes = 4;
            def_ad_bytes = 8;
            break;
	}
    
    // 首先检查前缀prefixes.
    switch(ctxt->b = insn_fetch(u8, ctxt)) {
        case 0xf0:	// LOCK
            ctxt->lock_prefix = 1;
        default:
            goto done_prefixes;
	}
    
done_prefixes:
    
    // Opcode byte(s).
    // 查一个大表，看当前的opcode的属性是什么
    opcode = opcode_table[ctxt->b];
    
	/* Two-byte opcode? */
    if (ctxt->b == 0x0f) {
        ctxt->opcode_len = 2;
        ctxt->b = insn_fetch(u8, ctxt);
        opcode = twobyte_table[ctxt->b];

        /* 0F_38 opcode map */
        if (ctxt->b == 0x38) {
            ctxt->opcode_len = 3;
            ctxt->b = insn_fetch(u8, ctxt);
            opcode = opcode_map_0f_38[ctxt->b];
        }
    }
    
    // 把标志属性位拿出来，flags里面标志了当前的opcode有没有ModRM，有没有操作数等等
    ctxt->d = opcode.flags;
    
    // 如果有ModRM，取出来
    if (ctxt->d & ModRM)
    	ctxt->modrm = insn_fetch(u8, ctxt);
    
    ...
    
    // Decode and fetch the source operand: register, memory
    // or immediate.
    rc = decode_operand(ctxt, &ctxt->src, (ctxt->d >> SrcShift) & OpMask);
}

static int __do_insn_fetch_bytes(struct x86_emulate_ctxt *ctxt, int op_size)
{
    // 当前已经取了多少字节的指令
    int cur_size = ctxt->fetch.end - ctxt->fetch.data;
    
    // 该函数计算对于当前段还能读取的最大字节数是多少，结果返回给max_size
    rc = __linearize(ctxt, addr, &max_size, 0, false, true, ctxt->mode,
                          ¦&linear);
    
    // 因为不知道到底要取多少字节，所以取的越多越好，但是不能超过15字节的限制、段的结尾和当前页		的结尾
    size = min_t(unsigned, 15UL ^ cur_size, max_size);
    size = min_t(unsigned, size, PAGE_SIZE - offset_in_page(linear));
    
    // 去真正的取指令
    // 这个函数会调用到kvm_fetch_guest_virt，最终会调用到__kvm_read_guest_page（该函数在kvm_main.c里），这里会调用copy_from_user将指令拷贝到ctxt->fetch.end中.
	rc = ctxt->ops->fetch(ctxt, linear, ctxt->fetch.end,
                          size, &ctxt->exception);
    ctxt->fetch.end += size;
    return X86EMUL_CONTINUE;

} 
```

在`decode_operand`函数中会去解析操作数，下面来看看这个函数里面具体做了什么。

```C
static int decode_operand(struct x86_emulate_ctxt *ctxt, struct operand *op, unsigned d)
{
    int rc = X86EMUL_CONTINUE;

    // 首先判断操作数的类型
    switch (d) {
        case OpReg:
            decode_register_operand(ctxt, op);
            break;
        case OpImmUByte:
            rc = decode_imm(ctxt, op, 1, false);
            break;
        case OpMem:
            ctxt->memop.bytes = (ctxt->d & ByteOp) ? 1 : ctxt->op_bytes;
            mem_common:
            // 在decode_modrm中解析出来得ctxt->memop在这里直接赋值给了operand
            *op = ctxt->memop;
            ctxt->memopp = op;
            if (ctxt->d & BitOp)
                fetch_bit_operand(ctxt);
            op->orig_val = op->val;
            break;
    }
}
```

下面来看一下`decode_register_operand`：

```C
static void decode_register_operand(struct x86_emulate_ctxt *ctxt, struct operand *op)
{
    // 哪个寄存器
    unsigned reg = ctxt->modrm_reg;
    
    ...
    op->type = OP_REG;
    op->bytes = (ctxt->d & ByteOp) ? 1 : ctxt->op_bytes;
    // 获取该寄存器的地址
    op->addr.reg = decode_register(ctxt, reg, ctxt->d & ByteOp);
    
    // 真正的读取该寄存器的值
    fetch_register_operand(op);
    op->orig_val = op->val;
}
```



在`x86_emulate_insn`这个函数里面会做真正的模拟操作。根据每一个指令的不同，去做相应的操作。

对于Mem的操作，在decode阶段解析出地址，在emulate阶段才去真正的读地址里的内容。



## 一些问题

* 如何不经过decode知道指令长度是多少？

  在IA-32e Mode下，指令最长15字节，kvm在decode阶段预取指令的时候也不知道有多长，所以要求取不超过15字节，不超过当前段的结尾，不超过当前页的结尾的最小值。不decode不能知道到底有多长。

* 找一个具体的例子说明要用的register.

  参见下一章。

* 在decode阶段，会不会发生memory的访问，会不会产生exception？

  在decode阶段只有预取指令会去读取guest的内存，在decode指令的时候，对于要访问内存的指令，只会计算出内存的地址，然后将该地址传递给emulate阶段进行访问。

* Decoder和Emulator之间的interface是怎么样的？

  interface主要是`struct x86_emulate_ctxt`结构体，该结构体中的operand src, src2 and dst, memop等等都是在decode阶段解析出来的操作数，提供给emulate进行真正的操作。

* 分析decoder和emulator的代码量。

| function                | line of code |
| ----------------------- | ------------ |
| x86_decode_insn         | 300          |
| __do_insn_fetch_bytes   | 40           |
| decode_modrm            | 133          |
| decode_abs              | 15           |
| decode_operand          | 162          |
| decode_register_operand | 27           |
| decode_imm              | 36           |

| function         | line of code |
| ---------------- | ------------ |
| x86_emulate_insn | 349          |

## 一个具体的decode和emulate的分析

这里以0x89，mov指令为例，`MOV r/m32, r32`, Move r32 to r/m32.

在opcode_table中的定义为：

```C
{
.flags = DstMem | SrcReg | ModRM | Mov | PageTable
.u.execute = em_mov
}
```

首先是会在`x86_decode_insn`中decode指令，（在emulate.c中），解析出opcode为0x89，保存到ctxt->b中，然后读取该指令的相关信息`ctxt->d = opcode.flags`，然后因为该指令的目的操作数是地址，并且有ModRM标志，所以调用`decode_modrm(ctxt, &ctxt->memop)`函数，在该函数里面解析对应的内存地址，大部分情况会调用到一个通用的ModR/M处理的部分，如果有SIB，则会与SIB一起进行计算，这里考虑没有SIB的情况：

```C
{
    base_reg = ctxt->modrm_rm;
    modrm_ea += reg_read(ctxt, base_reg);
    adjust_modrm_seg(ctxt, base_reg);
}
```

并且在这里读取相应的寄存器，并把地址计算保存到`modrm_ea`中，然后将结构保存到op中，这里的op就是之前传进来的`ctxt->memop`.

```C
op->addr.mem.ea = modrm_ea;
```

然后是对操作数进行解析，调用`decode_operand`进行解析，这里面对于寄存器，会直接读取出值，保存起来，对于内存，不直接读取值。

然后到`x86_emulate_insn`中，这里会对具体的指令进行模拟，主要通过调用`ctxt->execute`实现，这里的`ctxt->execute`的值在decode的时候将opcode_table中的u.exec这一项赋值给了它。

还是以0x89这个mov操作为例，因为几个特殊的判断它都不满足，所以直接调用了`em_mov`函数，在这个函数里只是简单的将一个值拷贝过去。

```C
static int em_mov(struct x86_emulate_ctxt *ctxt)
{
    memcpy(ctxt->dst.valptr, ctxt->src.valptr, sizeof(ctxt-			>src.valptr));
    return X86EMUL_CONTINUE;
}
```

在往后看，在`x86_emulate_insn`中，最后会调用到writeback:

```C
// 这里0x89的flags没有NoWrite标志位，所以调用writeback函数
if (!(ctxt->d & NoWrite)) {
    rc = writeback(ctxt, &ctxt->dst);
    if (rc != X86EMUL_CONTINUE)
        goto done;
}
```

再来看看`writeback`函数，因为传进来的参数是`ctxt->dst`，所以应该是OP_MEM的操作：

```C
static int writeback(struct x86_emulate_ctxt *ctxt, struct operand *op)
{
    switch (op->type) {
        case OP_REG:
            write_register_operand(op);
            break;
        case OP_MEM:
            if (ctxt->lock_prefix)
                return segmented_cmpxchg(ctxt,
                                         ¦op->addr.mem,
                                         ¦&op->orig_val,
                                         ¦&op->val,
                                         ¦op->bytes);
            else
                // 没有lock前缀调用这个
                return segmented_write(ctxt,
                                       op->addr.mem,
                                       &op->val,
                                       op->bytes);
            break;
}
```

这里可以看到的是，这里将`op->addr.mem，目的地址`和`op->val，源操作数的值`都传递给了`segmented_write`函数，这里面才去执行真正的写guest内存的操作。