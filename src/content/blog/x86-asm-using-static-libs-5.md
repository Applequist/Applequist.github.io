---
title: 'x86 ASM - Using static libs (Part 5)'
description: 'Low level programming in assembly'
pubDate: 'May 19 2025'
---

Note: These are notes from reading the book [Learn to program with assembly]() by Jonathan Bartlett.

In the previous chapter we saw how to write functions following the **System V ABI** convention. This allows us to call functions and be called by other functions that follow the same conventions, including those from libraries installed on your system.

These libraries come in 2 flavors :
- static libraries: these are just archive of object files (same as our compilation output so far) that can be copied verbatim into an executable during the link phase.
- dynamic libraries: these are loaded at runtime and the code is shared between process that use them.


## Using a static library

To call function `foo` from `libfoo`, you simply issue a `call foo`:
In `callfoo.s`:
```asm
.global _start
.section .text
_start:
  # set up arguments
  ...

  call foo

  # Rest of program
```

Note that we don't declare a `.global foo`. We just assume it is defined somewhere else.

You compile it as usual: `as callfoo.s -o callfoo.o`.

But if we attempt to link the object file as we did before:
```sh
$ ld callfoo.o -o callfoo
```

We will get an `undefined reference to 'foo'` error.

We need to tell the linker where it can find `foo` and how to link to it:
```sh
$ ld callfoo.o -static -lfoo -o callfoo
```

There is a catch: not all functions are that simple to use:
- some call into the kernel using syscall,
- some require some setup with the help of the OS.

To support that the C compiler will link special pieces of code into an executable that
will execute on startup. The `_start` symbol is then defined by compiler to 
do the libary setup and call into the **user entry point** called `main`. 

So if you use some libraries, it is safer to use the `main` entry point: 
```asm
.global main
.section .text
main:
  # set up arguments
  ...

  call foo

  # Rest of program
  ...

  ret
```

And to compile and link using the C compiler:
```sh
gcc callfoo.s -static -o callfoo
```


