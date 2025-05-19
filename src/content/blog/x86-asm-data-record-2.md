---
title: 'x86 ASM - Data Records (Part 2)'
description: 'Low level programming in assembly'
pubDate: 'May 09 2025'
---


## Data Records

A data record is a collection of attributes or fields. For instance a `Person` record may collect
the age, height and weight of a person.

Each attribute value is associated with a *type* which defines the range of possible, valid
values for that attribute and a method to encode all possible values into a bit patterns.

For instance, the person's age value may recorded as a number of complete years since birth
and be associated with a type whose range of possible values
is `0..255` and encoded in memory as a single quad (that is more than enough to encode 256 values!).

A person's height value may be recorded as a number of centimeters and be associated to a type
whose range of possible values is also `0..255` and also encoded in memory as a single quad as well.

Note that the age and height type might be the same or the might be 2 different types that happen
to have the same range and encoding.

A person's weight value may be recorded as a number of kilograms and associated to a type 
whose range of possible values is `0..450` and encoded in memory as a single quad.

Now we know about the attribute data types. We need to figure out a *layout* for our record.
No need to get fancy. We can decide to layout the data sequentially as follow:
- first 1 quad for the age,
- immediately followed by a quad for the height
- immediately followed by a quad for the weight

So if we start a record at a given address `#abcd4320`, given that the x86_64 ISA is
**little endian** (least significant byte comes first):
```
          +----------------------------------------+
#abcd4320 | 30 |    |    |    |    |    |     |    |  person.age = 48 yo
          +----------------------------------------+
#abcd4328 | ba |    |    |    |    |    |     |    |  person.height = 186 cm
          +----------------------------------------+
#abcd4330 | 60 |    |    |    |    |    |     |    |  person.weight = 96 kg
          +----------------------------------------+
```

So it is definitely not the most memory efficient layout but it works.

How to implement that in assembly?

In a `person_data.s` file, define constant offsets from the start of the record for each field and 
a constant for the size of the record:
```asm
.section .data

.global AGE_OFFSET, HEIGHT_OFFSET, WEIGHT_OFFSET
.equ AGE_OFFSET, 0
.equ HEIGHT_OFFSET, AGE_OFFSET + 8
.equ WEIGHT_OFFSET, HEIGHT_OFFSET + 8 

.global PERSON_RECORD_SIZE
.equ PERSON_RECORD_SIZE, 24
```

In the same file, define an array of person records:
```asm
.section .data

.global persons, numpersons
numpersons: 
  # Calcuate the number of records
  .quad (endpersons - persons) / PERSON_RECORD_SIZE 
persons:
  .quad 48, 186, 96 # me
  .quad  7, 135, 25 # my son
  .quad  5, 122, 20 # my daughter
endpersons: # mark the end of the array
```

Now compile:
```sh
$ as person_data.s -o person_data.o
```

We can use these data to extract the height of the tallest family member in a file `tallest.s`:
```asm
.global _start
.section .text
_start:
  leaq persons, %rbx     # %rbx points to the first person record.
  movq numpersons, %rcx
  movq $0, %rdi          # height of tallest person so far

  cmpq $0, %rcx
  je finish

start_loop:
  movq HEIGHT_OFFSET(%rbx), %rax

  cmpq %rdi, %rax
  jbe continue           # if %rax <= %rdi continue

  movq %rax, %rdi        # else update %rdi
  
continue:
  addq $PERSON_RECORD_SIZE, %rbx # %rbx points to the next person record
  loopq start_loop

finish:
  movq $60, %rax
  syscall
```

Compile and link the 2 object files:
```sh
$ as tallest.s -o tallest.o
$ ld person_data.o tallest.o -o tallest
```

Run:
```sh
$ ./tallest
$ echo $?
186
```

Success!

