---
title: Bare Metal STM32 - runtime errors
date: 2026-04-16
categories: [embedded]
tags: [stm32, bare-metal]
---

This time we’ll focus on what happens in runtime.

We’ll start from the binary we built previously and see how it behaves on real hardware.

To flash the program, I’m using `openocd` together with `arm-none-eabi-gdb`.
Since I’m working with an STM32 Nucleo L476RG, I’m using the appropriate target config:

```console
openocd -f interface/stlink.cfg -f target/stm32l4x.cfg

...
Info : STLINK V2J45M31 (API v2) VID:PID 0483:374B
Info : Target voltage: 3.273161
Info : [stm32l4x.cpu] Cortex-M4 r0p1 processor detected
Info : [stm32l4x.cpu] target has 6 breakpoints, 4 watchpoints
Info : starting gdb server for stm32l4x.cpu on 3333
Info : Listening on port 3333 for gdb connections
[stm32l4x.cpu] halted due to breakpoint, current mode: Thread 
xPSR: 00000000 pc: 0x08000040 msp: 0x20020000
```

After starting OpenOCD, we get information about where to connect using GDB.

---

## Connecting and flashing

In another terminal:

```console
arm-none-eabi-gdb app.elf

...
(gdb) target remote localhost:3333
Remote debugging using localhost:3333
0x08000040 in ?? ()
(gdb) monitor reset halt
Unable to match requested speed 500 kHz, using 480 kHz
[stm32l4x.cpu] halted due to debug-request, current mode: Thread 
xPSR: 00000000 pc: 0x08000040 msp: 0x20020000
(gdb) load
Loading section .text, size 0x1c lma 0x8000000
Start address 0x0800000c, load size 28
Transfer rate: 121 bytes/sec, 28 bytes/write.
```

So far, so good.

The code is flashed, the CPU is doing *something*, so let’s try to run it:

```console
(gdb) continue
Continuing.
```

---

## First runtime failure

GDB appears to hang on `Continuing.`
Meanwhile OpenOCD reports:

```console
[stm32l4x.cpu] halted due to debug-request, current mode: Thread 
xPSR: 00000000 pc: 0xe28db000 msp: 0xe52db004
```

Both the program counter and stack pointer point to completely invalid addresses.

This is neither FLASH nor RAM.

---

## What actually happens on reset

After reset, the CPU expects:

* initial stack pointer at `0x08000000`
* reset handler address at `0x08000004`

But instead of a vector table, we have raw machine code at the beginning of FLASH.

So the CPU interprets the first instructions as addresses
and jumps into completely random memory.

---

## Adding a minimal vector table

Let’s fix that by defining a minimal interrupt vector table:

```cpp
/* isr_vector.cpp */

extern "C" void Reset_Handler();

__attribute__((section(".isr_vector")))
const void* vector_table[] =
{
    (void*)0x20020000,   // initial stack pointer (top of RAM)
    (void*)Reset_Handler // reset handler
};
```

And update the linker script accordingly:

```
/* linker_script.ld */

MEMORY
{
  FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 1024K
}

SECTIONS
{
  .isr_vector :
  {
    KEEP(*(.isr_vector))
  } > FLASH

  .text :
  {
    *(.text*)
    *(.rodata*)
  } > FLASH
}
```

---

## Still crashing?

After rebuilding and flashing again:

```console
(gdb) monitor reset halt
[stm32l4x.cpu] halted due to debug-request
pc: 0xe28db000 msp: 0xe52db004

(gdb) load
Loading section .isr_vector, size 0x8 lma 0x8000000
Loading section .text, size 0x1c lma 0x8000008

(gdb) continue
Continuing.

(gdb) x/4x 0x08000000
0x8000000 <vector_table>: 0x20020000 0x08000014 0xe52db004 0xe28db000
```

At first glance, everything looks correct.

The vector table is there.
The stack pointer is valid.
The reset handler address looks reasonable.

And yet — the CPU still crashes.

---

## The subtle bug

The issue turns out to be more subtle.

Even though the code is compiled in Thumb mode,
function pointers and instruction set are two separate things.

On Cortex-M, the least significant bit of a function pointer must be set
to indicate Thumb mode.

Right now we have:

```text
0x08000014
```

But the correct value should be:

```text
0x08000015
```

Without this bit, the CPU attempts to execute in ARM mode —
which is not supported — and immediately triggers a HardFault.

---

## Fixing the vector table

The correct approach is to use proper function pointer types:

```cpp
typedef void (*handler_t)();

extern "C" void Reset_Handler();

__attribute__((section(".isr_vector")))
const handler_t vector_table[] =
{
    (handler_t)0x20020000,
    Reset_Handler
};
```

Now the compiler takes care of setting the Thumb bit.

---

## A quick note on Thumb mode

At this point something might look a bit confusing.

In the vector table we expect the reset handler to point to something like:

```text
0x08000015
```

But when debugging, GDB shows:

```text
pc: 0x08000014
```

So which one is correct?

Both.

---

### Function address vs function pointer

On Cortex-M, there is an important distinction between:

* where the function actually lives in memory
* how the CPU is supposed to execute it

The function itself is located at:

```text
0x08000014
```

This is the real, aligned address in FLASH.

But a function pointer is not just an address.

---

### The Thumb bit

On ARM Cortex-M, the least significant bit of a function pointer is used to indicate execution mode:

* `bit 0 = 1` → Thumb mode
* `bit 0 = 0` → ARM mode (not supported on Cortex-M → crash)

So the correct pointer stored in the vector table is:

```text
0x08000015
```

---

### What the CPU actually does

When the CPU reads the reset handler address from the vector table:

1. it sees `0x08000015`
2. it sets the Thumb mode internally
3. it clears bit 0
4. it starts executing at `0x08000014`

That’s why GDB shows:

```text
pc: 0x08000014
```

---

### Why it broke before

Previously, the vector table contained:

```text
0x08000014
```

Which means:

* Thumb bit = 0
* CPU tries to execute in ARM mode
* immediate HardFault

---

### Why using function pointers fixes it

When we define the vector table like this:

```cpp
typedef void (*handler_t)();

__attribute__((section(".isr_vector")))
const handler_t vector_table[] =
{
    (handler_t)0x20020000,
    Reset_Handler
};
```

the compiler automatically sets the Thumb bit for function pointers.

---

This is one of those details that is easy to miss —
and incredibly confusing when everything *looks* correct but still crashes immediately.

---

## Finally — the system boots

After rebuilding and flashing:

```console
(gdb) monitor reset halt
[stm32l4x.cpu] halted due to debug-request
pc: 0x08000014 msp: 0x20020000

(gdb) load
Loading section .isr_vector...
Loading section .text...

(gdb) continue
Continuing.

(gdb) bt
#0  0x08000014 in Reset_Handler ()
```

At this point, the system finally boots.

The CPU:

* loads the stack pointer
* jumps to `Reset_Handler`
* starts executing our code

---

## But we’re not done yet

There is still no proper runtime environment:

* no `.data` initialization
* no `.bss` zeroing
* no guarantees about global state

We are essentially jumping straight into `main()` and hoping for the best.

---

## And what does the program do?

Right now:

```cpp
int main()
{
    while (1) {}
}
```

So the CPU ends up exactly where we told it to:

an infinite loop.

Which might not look impressive —
but it’s the first time everything is actually under control.
