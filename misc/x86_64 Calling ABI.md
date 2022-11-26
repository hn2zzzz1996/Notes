# x86_64 Calling ABI

| Register | Purpose                                | Saved across calls |
| -------- | -------------------------------------- | ------------------ |
| `%rax`   | temp register; return value            | No                 |
| %rbx     | callee-saved                           | Yes                |
| %rcx     | used to pass 4th argument to functions | No                 |
| %rdx     | used to pass 3rd argument to functions | No                 |
| `%rsp`   | stack pointer                          | Yes                |
| `%rbp`   | callee-saved; base pointer             | Yes                |
| %rsi     | used to pass 2nd argument to functions | No                 |
| %rdi     | used to pass 1st argument to functions | No                 |
| %r8      | used to pass 5th argument to functions | No                 |
| %r9      | used to pass 6th argument to functions | No                 |
| %r10-r11 | temporary                              | No                 |
| %r12-r15 | callee-saved registers                 | Yes                |

## calling Convention

The caller uses registers to pass the first 6 arguments to the callee. Given the arguments in left-to-right order, the order of registers used is :

%rdi, %rsi, %rdx, %rcx, %r8, %r9. Any remaining arguments are passed on the stack in reverse order so that they can be popped off the stack in order.

The callee is responsible for preserving the value of registers %rbp, %rbx and %r12-r15, as these registers are owned by caller. The remaining registers are owned by the callee.

