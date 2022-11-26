# Evolution of the x86 context switch

The short list context switching tasks:

1. Repointing the work space: Restore the stack (SS:SP)
2. Finding the next instruction: Restore the IP (CS:IP)
3. Reconstructing task state: Restore the general purpose registers
4. Swapping memory address spaces: Updating page directory (CR3)
5. ...and more: FPUs, OS data structures, debug registers, hardware workarounds, etc.

