---
title: 'x86 ASM - Part 3'
description: 'Low level programming in assembly'
pubDate: 'May 11 2025'
---

## System call 

System calls are done using the `syscall` instruction.

`syscall` uses a (linux) specific calling convention to specify which system call you want the kernel to execute
and supply additional arguments:
- `%rax` contains the system call number (see table below)
- Additional arguments are passed in `%rdi`, `%rsi`, `%rdx`, `%r10`, `%r8`, `%r9`

If the syscall needs to return a value, the return value is passed into `%rax` as well.
Beside, some registers are *clobbered* by `syscall`:
- `%rcx` contains the next instruction location after the system call returns,
- `%r11` contains a copy of the `%eflags` register,
- `%rax` contains the returned value of the system call if any.

The calling code should save these registers if needed, before executing a system call.

### Examples

#### Unix system time

System call number is `201` or `0xC9`.
You supply the address of the return value in `%rdi` which is returned in `%rax` on success.

```asm
.global _start

.section .data
curtime:
  .quad 0 # reserve memory for return value

.section .text
_start:
  movq $0xC9, %rax # select system call
  movq $curtime, %rdi # return unix time at curtime
  syscall

  # Exit 0
  movq $0x3C, %rax
  movq $0, %rdi
```

#### Writing output

Call interface:
- `%rax` contains `1` (the system call number)
- `%rdi` contains the output file descriptor
- `%rsi` contains the address of the data to write
- `%rdx` contains the length in bytes of the data to write.

```asm
.global _start

.section .data
helloworld:
  .ascii "Hello World!\n"
helloworld_end:
.equ STR_LEN, (helloworld_end - helloworld)

.section .text
_start:
  # Write "Hello World!\n" to stdout
  movq $0x01, %rax
  movq $1, %rdi
  movq $helloworld, %rsi
  movq $STR_LEN, %rdx
  syscall

  # Exit 0
  movq $0x3C, %rax
  movq $0, %rdi
  syscall
```


