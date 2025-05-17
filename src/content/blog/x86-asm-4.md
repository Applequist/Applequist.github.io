---
title: 'x86 ASM - Part 4'
description: 'Low level programming in assembly'
pubDate: 'May 17 2025'
---

## Functions and calling conventions

When a program is too large, it becomes hard to understand and maintain.
It is then necessary to split large programs into smaller **functions** that are easier to understand
but that also can be **reused**. 

Functions are piece of code with:
- a name (a global symbol),
- some input parameters,
- a return value (multiple return values can be *simulated* by passing output input parameters),
- some side effects (avoided but required in some situation).
- a clear and limited purpose

A program then becomes a function itself that **calls** other functions.

But implementing functions that can call each other poses certain questions/challenges:
- how to pass input parameters?
- where to store temporary values used by the function?
- how do we return value?
- where to resume execution after the function is done?

All these problems are solved by adhering to an **Application Binary Interface**.
The ABI defines rules for the orderly use of CPU registers and prescribes the use of a stack for local value storage
and chaining functions.

### what is the stack ? 

The stack is a reserved area of memory where functions can push/pop values.
The `%rsp`, the stack pointer, points to the tip of the stack where you push/pop values.

In the diagram below, each cell represents a quad (or 8 bytes). The tip of the stack contains 
the value `0x01`.

```
#128     |           ...           |
         +-------------------------+   
#120     | ab                      |    
         +-------------------------+  
#118     | 01                      | <-- %rsp
         +-------------------------+
         |                         |
```

To push a value on the stack, you proceed as follow:
- subtract from `%rsp` the size in byte of the value to push (stack grows toward low memory addresses),
- copy your value to the stack starting at `%rsp` (multibyte copy goes proceed toward high memory address).
   
To pop a value from the stack:
- copy the value from the stack starting at `%rsp`,
- add to `%rsp` the size in byte of the value you just read.

The `push` family of instructions helps to push data on the stack, eg `pushq %rax`
- it decrements `%rsp`,
- then copy the quad value in `%rax` to the location now pointed to by `%rsp`

The `pop` family of instructions to pop data off the stack, eg `popq %rcx`
- copy the quad value pointed to by `%rsp` into `%rcx`
- increment `%rsp`

You can also simply reserve space on the stack by subtracting the amount of memory to reserve
from `%rsp`.

### Calling a function

The **System V ABI** used in Linux states that `%rbp`, `%rbx` and `%r12` to `%r15` registers **should** be preserved
accross function calls. 

So if the caller is using any **other registers** (which **may not** be preserved accross a call),
it should save them on the stack.

The next order of business of the **caller** is to pass the required arguments to the **callee**.

In the **System V ABI** input parameters come into a function *primarily* in registers:
- the first 6 arguments are passed in the following registers in that order: first parameter in `%rdi`, 
  second in `%rsi`, then `%rdx`, `%rcx`, `%r8` and `%r9`.
- any additional parameters are pushed onto the stack in reverse order, i.e., last first then second to last... 
  Till the 7th parameter.

Before jumping to the called function code, the **caller** must push the address where execution should 
resume after the call, typically this is address of the next instruction after the call.
The `call` instruction does just that.

Upon return, the **caller** should restore the saved registers and resume execution.

The following code sample shows a function call sequence from the caller's perspective:
```asm
  # Call foo(5)
  # 1. Save registers
  pushq %r10

  # 2. Pass input parameter
  movq $5, %rdi 

  # 3. Push return address and jump to foo
  call foo

  # 4. Restore saved registers
  popq %r10

  # continue caller code 
  ...
```  

### Being a callee

Let's go back to the use of the stack as a local storage facility... 

How is a function going to keep track of its local data as it is pushing and poping data to/off the stack? 
`%rsp` will be changing, forcing the function to constantly track offsets from a moving reference? 

The solution is pretty simple: instead of tracking position in the stack from a moving `%rsp`, it will track 
position in the stack from a reference that is fixed for the duration of the function execution.

To do that, a function uses the **base pointer** register and upon entry, before anything, setup its stack frame
as follow.

Say the stack looks like this, just before jumping to the called function but after the caller pushed the return
address:
```
#130     |           ...           | <-- %rbp
         +-------------------------+   
#128     |           ...           |
         +-------------------------+   
#120     | ab                      |    
         +-------------------------+  
#118     | return address          | <-- %rsp
         +-------------------------+
         |                         |
```

The callee first save `%rbp` for the caller by pushing it on the stack: `pushq %rbp`. The stack now looks like below:
```
#130     |           ...           | <-- %rbp
         +-------------------------+   
#128     |           ...           |
         +-------------------------+   
#120     | ab                      |    
         +-------------------------+  
#118     | return address          | 
         +-------------------------+
         | 30 01                   | <-- %rsp
         +-------------------------+
```

Then, it updates `%rbp` with the current value of `%rsp`: `movq %rbp, %rsp`. The stack now looks like below:
```
#130     |           ...           | <-- previous %rbp
         +-------------------------+   
#128     |           ...           |
         +-------------------------+   
#120     | ab                      |    
         +-------------------------+  
#118     | return address          | 
         +-------------------------+
#110     | 30 01                   | <-- %rsp <-- %rbp
         +-------------------------+
```

And finally, it reserves some space by subracting from `%rsp`: `subq $16, %rsp` (2 quads). The stack now looks like 
below:
```
#130     |           ...           | <-- previous %rbp
         +-------------------------+   
#128     |           ...           |
         +-------------------------+   
#120     | ab                      |    
         +-------------------------+  
#118     | return address          | 
         +-------------------------+
#110     | 30 01                   | <-- %rbp
         +-------------------------+
#108     |      -8(%rbp)           |
         +-------------------------+   
#100     |     -16(%rbp)           | <-- %rsp
         +-------------------------+   
         |                         |
```

Just before returning, the callee must clean up its stack.

First, update `%rsp` with `%rbp`: `movq %rbp, %rsp`. 

The stack will then look like below:
```
#130     |           ...           | <-- previous %rbp
         +-------------------------+   
#128     |           ...           |
         +-------------------------+   
#120     | ab                      |    
         +-------------------------+  
#118     | return address          | 
         +-------------------------+
#110     | 30 01                   | <-- %rbp <-- %rsp
         +-------------------------+
         |                         |
```

Then, pop the value now at `%rsp` (the `%rbp` of the caller) into `%rbp`: `popq %rbp`.

The stack now looks like this:
```
#130     |           ...           | <-- %rbp
         +-------------------------+   
#128     |           ...           |
         +-------------------------+   
#120     | ab                      |    
         +-------------------------+  
#118     | return address          | <-- %rsp
         +-------------------------+
         |                         |
```

And finally, the **callee** must pop the return address (pushed for him by the caller) and jump to it using `ret`.
```
#130     |           ...           | <-- %rbp
         +-------------------------+   
#128     |           ...           |
         +-------------------------+   
#120     | ab                      | <-- %rsp   
         +-------------------------+  
         |                         |
```

Now all of this can be achieved using `pushq`, `popq`, `movq` and `jmp` instructions and the `%rsp` and `%rbp` 
registers but it is so common and important that there are dedicated instructions:
- In the caller: `call foo` pushes the address of the next instruction (return address) and jump to foo.
- In the callee: `enter $16, $0` setup the callee stack by saving `%rbp` on the stack, updating `%rpb` and 
  reserving 16 bytes on the stack.
- In the callee `leave` resets the stack to the state it was in just before entering foo, that is with the return 
  address on top.
- In the callee: `ret` pop the return address from the stack and jump to it resuming executiong at the instruction 
  following the caller's `call foo` instruction.

You can verify that this call sequence leaves the stack unchanged.

### Other convention rules.

Now leaving the stack *untouched* is only one part of the convention. And there are other rules that must be 
followed to guarantee the proper operations of functions.

Here is a summary:

|  Rule              |          Registers               |       Remarks                |
|--------------------|----------------------------------|------------------------------|
| Input parameters   | 1: `%rdi`, 2: `%rsi`, 3: `%rdx`, |                              |
|                    | 4: `%rcx`, 5: `%r8`, 6: `%r9`    |                              |
| Return value       | `%rax` | Optional 2nd return val in `%rdx`. |
|                    |        | This is however rareley used. |
| Preserved reg      | `%rbp`, `%rbx`, `%r12` to `%r15` | Called function must save on entry and |
|                    |                                  | restore content before return. |
|                    |                                  | Caller must expect **other** registered to be clobbered. |
| Stack alignment    | `%rsp` must be a multiple of 16! | Since we always push a return address and the previous |
|                    |                                  | base pointer, functions should always reserve a multiple |
|                    |                                  | of 16 bytes. |

## Summary

### Stack structure

```
         |                         |    |
         +-------------------------+    |
         |                         |    | caller's frame
         +-------------------------+    |
         |                         | <--+
         +-------------------------+
         |  return address         | <--+
         +-------------------------+    |
%rbp --> |  previous %rbp          |    |
         +-------------------------+    |
         |  local1 -8(%rbp)        |    | stack frame of currently executing function
         +-------------------------+    |
         |  local2 -16(%rbp)       |    |
         +-------------------------+    |
%rsp --> |  local3 -24(%rbp)       | <--+
         +-------------------------+
         |                         |
```

### Call sequence

In caller:
```asm
  # Initialize parameters, eg:
  movq $1, %rdi

  # Push address of next instruction and jump to foo
  call foo
  ...
```

In foo:
```asm
  # Setup foo's stack frame
  enter $NUMBYTES, $0
  # Same as but slower than
  # pushq %rbp           # Save caller's base pointer
  # movq %rsp, %rbp      # Initialize foo's base pointer
  # subq $NUMBYTES, %rsp # reserve space for local vars

  # Rest of foo code 
  ...

  # before returning
  leave
  # Same as
  # movq %rbp, %rsp      # Return reserved memory
  # popq %rbp            # Pop previous base pointer from stack (pushed on entry)

  # Return 
  ret # pop return value of the stack and jump to it
```

