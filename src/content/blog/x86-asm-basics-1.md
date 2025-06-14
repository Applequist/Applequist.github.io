---
title: 'x86 ASM - Basics (Part 1)'
description: 'Low level programming in assembly'
pubDate: 'May 08 2025'
---

Lately I felt the need to go back to the basics to strengthen my understanding of computing and programming languages.
So learning a bit of assembly is one thing I decided to do to get a better idea of how higher-level programming language (I am talking C and Rust here) are implemented and maybe better understand them.

So this is the first part of my notes on following the book [Learn to program with assembly](https://www.amazon.com/Learn-Program-Assembly-Foundational-Programmers/dp/1484274369) by Jonathan Bartlett.

# x86 Assembly basics

## Registers

|  Name    |       Kind      |              Purpose                |
|----------|-----------------|-------------------------------------|
|  %rax    | General purpose | Accumulator register |
|  %rbx    | General purpose | base register, used for indexing |
|  %rcx    | General purpose | counter register, used in loops |
|  %rdx    | General purpose | data register with special functions in some instructions |
|  %rbp    | General purpose | base pointer register. Leave it! |
|  %rsp    | General purpose | stack pointer register. Leave it too! |
|  %rsi    | General purpose | the source register |
|  %rdi    | General purpose | the destination register |
|  %r8     | General purpose | |
|  %r9     | General purpose | |
|  %r10    | General purpose | |
|  %r11    | General purpose | |
|  %r12    | General purpose | |
|  %r13    | General purpose | |
|  %r14    | General purpose | |
|  %r15    | General purpose | |
|  %rip    | Special purpose | Instruction pointer |
|  %eflags | Special purpose | CPU flags: ZF, CF, OF, SF... |


All registers above except `%eflags` are 64-bit registers but can be accessed as 32-bit, 16-bit and 8-bit registers.

```
<--------------------------- %rax(64b) ------------------------->
+-------------------------------+---------------+-------+-------+
|                               |               |       |       |
+-------------------------------+---------------+-------+-------+
                                <----------- %eax(32b) --------->
                                                <--- %ax(16b) -->
                                                <- %ah-><- %al ->
```

For instance, for `%rax` (64-bit): 
- `%eax` accesses the least significant 32-bit half, 
- `%ax` accesses the least significant 16-bit half of `%eax` 
- and the 2 8-bit bytes of `%eax` are `%ah` and `%al`:

## Addressing modes

All assembly instructions load (and store) data. 

These data can be supplied at compilation time, or read from registers or memory 
and can be stored into registers or memory.
Addressing modes are the mechanisms that let the programmer select where instructions 
should read data from and where to store data.

### Immediate mode 

In this mode, we *encode* a value directly into the instruction by preceding it with the `$` (dollar) sign.

Eg, in
```asm
movq $5, %rax
```

`$5` is the immediate mode operand and the instruction moves the value '5' into the destination operand `%rax`.

Not really an addressing mode since there is no address involved but is necessary to supply compile time data
and constant values as source operands.

### Register mode 

In this mode, we use a value stored in a register as a source, or a register as a storage destination.

Eg, in
```asm
movq $5, %rax
```

`%rax` is the register mode operand and the instruction moves the value '5' directly into the register `%rax`.

Not really an addressing mode since there is no address involved but is necessary to load/store values from/to
registers.

### Direct addressing mode 

This is the first *real* addressing mode!

In this mode, we refer to a value by its address.

Eg, in
```asm
movq 0, %rbx
```

`0` is the direct mode operand and the instruction moves the value at address `0` into the register `%rbx`.
This is often used with **labels**, special tag in assembly program, that are replaced by their address in the 
compiled program.

### Register indirect mode 

In the mode, we refer to a value by an address stored in a register. 

Eg, in
```asm
movq (%rbx), %rax
```

`(%rbx)` is the register indirect mode operand and the instruction moves the value whose address is in `%rbx` into `%rax`.

Here `%rbx` acts as a *pointer*: it does not contain the value but an address that points to it.

### Generalized addressing mode

Both the direct addressing mode and the register indirect address mode are special cases of the *generalized addressing mode*.

In this addressing mode, values are refered by an address encoded as: 
```
value(base_reg, index_reg, multiplier)
```

where:
- `value` is fixed value (number or label), 
- `base_reg` and `index_reg` are register names, 
- and `multiplier` is 1 (default if left out), 2, 4, or 8.

The actual address is then: 
```
value + val(base_reg) + val(index_reg) * multiplier
``` 

where:
- `val(base_reg)` and `val(index_reg)` are the values stored in the corresponding registers.
- any part missing is assumed to be 0 (1 for the multiplier)

With this *scheme*, it is easy to see that:
- direct mode = `value`
- register indirect mode = `(base_reg)`

Two other commonly modes derived from this generalized addressing mode are:
- the **indexed mode** used to access array elements:
  ```
  array(, index_reg, multiplier)
  ```
  where 
  - `array` is the address of the 1st element, 
  - `index_reg` contains the index of the element to access 
  - and `multiplier` is the size **in bytes** of elements: 8 for 64bit values, 4 for 32bit values...

- the **base pointer mode** aka **displacement mode** used to access *struct* fields:
  ```
  field_offset(base_reg,,)
  ```
  where
  - `base_reg` is the base register containing the address of the *start* of the struct in memory,
  - `field_offset` is a offset in bytes from the start of the struct of the field to access.

One instruction allows you to compute an *effective address* from a generalized addressing mode operand
and store it in a register.

Do this: 
```asm
leaq number, %rbx
```

Don't do this:
```asm
moveq $number, %rbx
```

## Basic Instructions

### Comparing, branching and looping

The CPU pulls the instructions to execute one by one using the `%rip` register to know where 
to fetch the next instruction.
By manipulating this register, we can implement different branching and looping constructs.

#### Unconditional jump

By using the `jmp` instruction, we can set the `%rip` value and alter the flow of a program:
```asm
jmp label # next instruction found at label
```

#### Conditional jumps

These instructions only jump if some flags are set in the `%eflags` register.

| Instruction  | Jump if...                          |
|--------------|-------------------------------------|
| jz           | the zero flag ZF is set (= 1) |
| jnz          | the zero flag ZF is NOT set (= 0) |
| jc           | the carry flag CF is set (= 1) |
| jnc          | the carry flag CF is NOT set (= 0) |


The CF and ZF flags are set by various instructions during their execution.

Another way to do conditional jumps is by comparing 2 *unsigned* numbers:
```asm
cmpq op1, op2
```

And then jump based on the relative order of the operand values:

| Instruction  | Jump if...                          |
|--------------|-------------------------------------|
| je           | op2 = op1 |
| jne          | op2 != op1 |
| ja           | op2 > op1 |
| jae          | op2 >= op1 |
| jb           | op2 < op1 |
| jbe          | op2 <= op1 |


Watch out for the order of comparison in the table above!

Instead of jumping after a `cmpq`, we can do conditional moves:

| Instruction  | Move src to dst if...                          |
|--------------|------------------------------------------------|
| cmovgq src, dst | op2 >= op1 |
| cmovleq src, dst | op2 <= op1 |
| ...           | |

Other variants of the instruction exist using the same prefix `cmov` and a conditional code suffix like `e`, `ne`,...

#### loop

Finally another kind of jump is used to implement loops.

The `loop` family of instruction decrements the `%rcx` register and then conditionally jumps to the specified label
based on the value of `%rcx` and the conditional code of the instruction.

The way to use it to execute N loops is as follow:
- set `%rcx` to the number of iteration N,
- set a label `start_loop` to mark the start of the loop in the program
- at the end of the body of the loop, execute a `loopneq start_loop` which decrements `%rcx` and jump to `start_loop`
  if `%rcx` is not 0 (-neq conditional code)


## Program layout

An assembly program can be divided into multiple sections.

### data sections

Assembly programs need to store data: constants, global varialbes... 

These data are stored in data sections. There are multiple type of data sections:
- `.data` for initialized data and constants
- `.rodata` for readonly data
- `.bss` for unitialized data. The size of the data is specified but not the values. 

The following assembly sample defines a data section and 2 constants:
```asm
.section .data
foo:
  .quad 42
bar:
  .quad 24
```

We use `.quad` to initialize quads (64bit) values in memory. We could initialize more than 1 value:
```asm
numbers:
  .quad 1, 2, 3, 4, 5, 6, 7
```

But that is just lays numbers in memory one after the other starting at address labelled `numbers`. 
If we want to sum these numbers we can start at `numbers`, load the value at that address into a register, 
move to the next one, load it and add it to the first one and so on... But wait! Where/when do we stop? 

To know when to stop we can either add a *sentinel* value like a `\0` in C-strings or simply declare the number of
value: 
```asm
number_count: 
  .quad 7
numbers:
  .quad 1, 2, 3, 4, 5, 6, 7
```

There are other types of data section:
- **.rodata*** sections contain **read-only** data. Attempt to modify `.rodata` memory will abort the program.
- **.bss** sections contain uninitialized data (initialized to all zeroes by the OS). Instead of values you specify
  size. This allows to keep executable size low as the memory is not reserved until runtime by the OS.

You can also reserve data in the `.bss` section using 2 directives:
- `.lcomm sym, 8` reserves `8` bytes in the `.bss` section and defines a **local** symbol `sym` for the address,
  unless `sym` is also marked as `.global`.
- `.comm sym, 8` is similar to `.lcomm` except that the linker will merge all declarations for similarly named symbols.

### Text section

The text section contains the actual code of the program.

The following assembly sample defines a text section (in addition to the data section above),
add the 2 contants and return the result as the exit code:
```asm
.global _start

.section .data
foo:
  .quad 42
bar:
  .quad 24

.section .text
_start:
  # load values in registers using direct addressing mode
  movq foo, %rbx
  movq bar, %rcx

  # Add the values and store the result in %rcx
  addq %rbx, %rcx 

  # Return the sum
  movq $60, %rax  # exit system call
  movq %rcx, %rdi # exit code
  syscall
```

To compile and run:
```sh
$ as sum.s -o sum.o && ld sum.o -o sum
$ ./sum
$ echo $?
$ 66
```o

### Alignment 

Alignment is quite important, not only for data but also for code: some instructions cannot be executed if not 
properly aligned in memory.

Alignment can be adjusted (in `.data` or `.text` section) using the following directives:
- `.balign 8` aligns the next address to the given multiple. Padding is filled with 0s in `.data` sections,
- `.p2align 3` specifies the number of bits used for alignments. Here the last 3 bits must be 0, so this is equivalent
  to `.balign 8` (2^3 = 8).
- `.align` NOT RECOMMENDED as it behaves either as `.balign` or as `.p2align` depending on the context.
  or `nop` instructions in `.text` sections.

Note that the stack must be 16-byte aligned.

### Including other code

You can `.include "a_file.s"` a file in your program. Usually included files do not contain any `.data` or `.text` section at all
only `.equ` constants.

You can `.incbin "my_img.png"` a binary file verbatim into your object file. 

### Annotating code

You can annotate `.global` symbols with `.type` annotations, eg: 
```asm
.global my_var, my_func
.type my_var, @object
.type my_func, @function
```

