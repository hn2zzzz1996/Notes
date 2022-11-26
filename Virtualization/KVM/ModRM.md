0x88, mov r/m8, r8. 

```C
{
.flags = DstMem | SrcReg | ModRM | Mov | PageTable | ByteOp
.u.execute = em_mov
}
```

## ModR/M and SIB Bytes

The form of ModR/M:

```C
Mod | REG | R/M
```

* mod field 和 r/m field 绑定在一起表示32种可能得值：8个寄存器和24个寻址模式。
* reg/opcode field 指定了一个寄存器号或者三个比特长度的opcode的补充信息。reg/opcode field的用途在primary opcode中指定。
* r/m field 指定一个寄存器作为操作数或者与mod field一起去encode一种寻址模式。

## Addressing-Mode Encoding of ModR/M

The mod and r/m field 可以表示32个effective addresses，作为第一个操作数。前24个选项提供了指定内存区域的方式；后8个(Mod = 11B)提供了指定通用寄存器，MMX技术和XMM寄存器的方式。

The Reg/Opcode field 被用来指定第二个操作数的位置。第二个操作数必须是一个通用寄存器，MMX技术或者XMM寄存器。