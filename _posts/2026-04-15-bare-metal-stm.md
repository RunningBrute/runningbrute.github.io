---
title: Bare Metal STM32 Nucleo
date: 2026-04-15
categories: [embedded]
tags: [stm32, bare-metal]
---

There’s no better way to truly understand something than building it from scratch.
So I decided to dust off my STM32 Nucleo board and try to bring it up completely “bare metal”. Let’s start with the simplest possible program:


```cpp
/* main.cpp */

int main()
{
    while (1) {}
}
```

---

## First problem – libc

When trying to compile, we immediately hit the first issue:

```console
arm-none-eabi-g++ main.cpp -o app.elf

... undefined reference to `_exit'
... undefined reference to `_write'
... undefined reference to `_sbrk'
```

The linker is trying to pull in functions from libc, which in turn assume the presence of an operating system
(e.g. via syscalls like `_write`, `_sbrk`, etc.).

The problem is — in bare metal, there is no OS.

So the first attempt to fix this is to disable the standard library:

```console
arm-none-eabi-g++ main.cpp -nostdlib -ffreestanding -o app.elf
```

---

## Missing entry point

This time we get:

```console
ld: warning: cannot find entry symbol _start
```

By default, the linker expects a `_start` symbol (the “hosted” world),
but in embedded systems we define the entry point ourselves — usually `Reset_Handler`.

So let’s create a minimal startup:

```cpp
/* startup.cpp */

extern "C" void Reset_Handler()
{
    extern int main();
    main();

    while (1) {}
}
```

And tell the linker to start from there:

```console
-Wl,-e,Reset_Handler
```

---

## C++ and exceptions

Next attempt:

```console
undefined reference to `__aeabi_unwind_cpp_pr1'
```

This happens because the compiler generated exception handling code.

In bare metal environments:

* exceptions are usually disabled
* RTTI is typically not used

So we disable everything:

```console
-fno-exceptions
-fno-unwind-tables
-fno-asynchronous-unwind-tables
-fno-rtti
```

---

## First successful build

Final command:

```console
arm-none-eabi-g++ main.cpp startup.cpp \
  -nostdlib -ffreestanding \
  -fno-exceptions -fno-unwind-tables -fno-asynchronous-unwind-tables -fno-rtti \
  -Wl,-e,Reset_Handler \
  -o app.elf
```

An ELF file is generated — so at least we’re moving forward.

---

## Is the code in the right place?

Before flashing anything, it’s worth checking where the linker actually placed our code:

```console
arm-none-eabi-objdump -h app.elf
```

We get:

```text
.text  00008000
```

That looks suspicious.

For STM32, code should start in FLASH at:

```
0x08000000
```

So clearly our code ended up in the wrong place.

---

## Linker script

We need to tell the linker what the memory layout of the microcontroller looks like.

A minimal linker script:

```
/* linker_script.ld */

MEMORY
{
  FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 1024K
}

SECTIONS
{
  .text :
  {
    *(.text*)
    *(.rodata*)
  } > FLASH
}
```

Add it to the build:

```console
-T linker_script.ld
```

---

## Verification

```console
arm-none-eabi-objdump -h app.elf
```

Now we get:

```text
.text  08000000
```

That looks much better.

Next step is to actually flash this to the board and see what happens.

And a lot will go wrong — which is exactly what we want.

