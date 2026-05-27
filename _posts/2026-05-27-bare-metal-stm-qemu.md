---
title: Bare Metal STM32 - QEMU simulator
date: 2026-05-27
categories: [embedded]
tags: [stm32, bare-metal, qemu]
---

For better and, most importantly, much faster testing, it is worth checking the whole software stack before running it on the real board. Because of that i wanted to see how the QEMU simulator handles STM32 bare metal projects in practice. Luckily installation and basic usage are pretty simple.

Install QEMU package:

```bash
sudo apt install qemu-system-arm
```

Then we need to check supported STM32 boards. For now i use `olimex-stm32-h405` as it is one of the supported targets (they has Cortex-M4 as my nucleo board) and works well enough for basic testing:

```bash
qemu-system-arm -machine help | grep stm32
olimex-stm32-h405    Olimex STM32-H405 (Cortex-M4)
stm32vldiscovery     ST STM32VLDISCOVERY (Cortex-M3)
```

Minimal run looks like this:

```bash
qemu-system-arm \
    -M olimex-stm32-h405 \
    -kernel firmware.elf \
    -nographic
```

We can also run QEMU together with GDB support by adding `-S` (CPU starts halted) and `-s` (starts gdb server on `localhost:1234`) options.

```bash
qemu-system-arm \
    -M olimex-stm32-h405 \
    -kernel firmware.elf \
    -S -s \
    -nographic
```

Then, finally we can connect from GDB as usual using target remote `localhost:1234`:

```bash
~/nanortos$ arm-none-eabi-gdb build/nano_rtos.elf
GNU gdb (Arm GNU Toolchain 15.2.Rel1 (Build arm-15.86)) 16.3.90.20250906-git
Copyright (C) 2024 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "--host=x86_64-pc-linux-gnu --target=arm-none-eabi".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<https://gitlab.arm.com/tooling/gnu-devtools-for-arm/-/issues/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from build/nano_rtos.elf...

(gdb) target remote :1234
Remote debugging using :1234
0x08000040 in Reset_Handler ()
...

```
