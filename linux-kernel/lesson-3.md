# Lesson 3: Linux Kernel Boot Process and Entry

When you press the power button, the CPU starts in a very limited state.

For backward compatibility, modern x86_64 processors begin execution in **16-bit Real Mode**, similar to the old Intel 8086 processor.

In this mode:

- The CPU can only access 1 MB of memory.
- There is no memory protection.
- Virtual memory is not available.
- Running normal C code is not possible.

The main goal of the boot process is to **prepare the CPU environment for the kernel**.

The kernel must:

- Move from 16-bit mode to 32-bit mode.
- Move from 32-bit mode to 64-bit mode.
- Enable virtual memory.
- Create an initial stack.
- Decompress itself.
- Start executing C code.

Only after this setup can the kernel begin normal initialization.

---

# 1. Boot Process Overview

Early boot code is hardware-specific, so it lives inside the `arch/` directory.

After the CPU environment is prepared, execution moves to generic kernel C code inside `init/`.

The x86 boot flow:

```text
[ Bootloader (GRUB/UEFI) ]
            |
            v
Loads bzImage into RAM
            |
            v
[ arch/x86/boot/header.S ]
16-bit entry point
Boot header setup
            |
            v
[ arch/x86/boot/main.c ]
Hardware setup
Switch to 32-bit mode
            |
            v
[ arch/x86/boot/compressed/ ]
Decompress kernel
(vmlinux)
            |
            v
[ arch/x86/kernel/head_64.S ]
Enter 64-bit mode
Setup paging and stack
            |
            v
[ init/main.c ]
start_kernel()
            |
            v
Kernel initialization begins
````

---

# 2. Bootloader Handoff

The bootloader (such as GRUB) performs the first step.

It:

1. Reads the compressed kernel image (`bzImage`) from disk.
2. Loads it into memory.
3. Passes hardware information to the kernel.
4. Transfers execution to the kernel entry point.

The bootloader cannot directly jump to any kernel function.

The kernel defines a **Boot Protocol** that describes:

* Where execution should begin.
* Where boot information is stored.
* How hardware details are passed.

This information is stored in a structure called:

```c
struct boot_params
```

It contains information such as:

* Memory map.
* Hardware configuration.
* Kernel command line.

---

# 3. Early Assembly: `header.S`

File:

```text
arch/x86/boot/header.S
```

This contains the initial kernel entry point.

The first symbol is:

```text
_start
```

## Why Assembly?

At this point:

* The CPU is still in 16-bit mode.
* No C stack exists.
* Memory management is not enabled.

C code requires a working execution environment, so assembly must prepare it first.

---

## Responsibilities of `header.S`

It:

* Defines the kernel boot header.
* Checks whether a valid bootloader loaded the kernel.
* Verifies the boot magic value.
* Prepares information needed for the next stage.

After this, execution moves to:

```text
arch/x86/boot/main.c
```

---

# 4. Early C Setup

File:

```text
arch/x86/boot/main.c
```

At this stage, the kernel can execute limited C code.

It performs basic setup:

* Detects available memory.
* Collects hardware information.
* Prepares CPU mode switching.
* Sets up the environment for kernel decompression.

The CPU is then moved toward 32-bit Protected Mode.

---

# 5. Kernel Decompression

The kernel image loaded by the bootloader is compressed.

The decompression code lives in:

```text
arch/x86/boot/compressed/
```

This code works like a self-extracting archive.

The flow:

```text
bzImage
   |
   v
Decompression code
   |
   v
Extract vmlinux
   |
   v
Place kernel in memory
```

The result is the real kernel image:

```text
vmlinux
```

---

# 6. Entering 64-bit Mode

File:

```text
arch/x86/kernel/head_64.S
```

The kernel now starts executing in 64-bit mode.

The entry function is:

```text
startup_64
```

Before C code can run, two important things are required.

---

## 1. Stack Setup

C functions need a stack for:

* Local variables.
* Function calls.
* Return addresses.

The kernel creates an initial stack and sets:

```text
%rsp
```

(Stack Pointer register)

to a valid memory location.

---

## 2. Page Table Setup

The kernel uses virtual memory.

This assembly code creates the first page tables that map:

```text
Physical Memory
        |
        v
Virtual Memory
```

This allows the kernel to use normal virtual addresses.

---

After the environment is ready, execution jumps to:

```c
start_kernel()
```

---

# 7. The C Entry Point: `start_kernel()`

File:

```text
init/main.c
```

Function:

```c
start_kernel()
```

This is the main C entry point of the Linux kernel.

Example:

```c
asmlinkage __visible void __init __no_sanitize_address start_kernel(void)
{
    set_task_stack_end_magic(&init_task);

    smp_setup_processor_id();

    boot_cpu_init();

    page_alloc_init();

    trap_init();

    mm_init();

    sched_init();

    arch_call_rest_init();
}
```

---

# 8. Understanding `start_kernel()`

## Compiler Attributes

### `__init`

Functions marked with:

```c
__init
```

are placed into a special memory section:

```text
.init.text
```

After boot completes:

* This memory is freed.
* The code is removed from RAM.

Reason:

Boot-only code is no longer needed.

---

### `asmlinkage`

This tells the compiler that function arguments are passed using the stack instead of CPU registers.

It is mainly used for architecture-specific calling conventions.

---

# 9. Kernel Subsystem Initialization

`start_kernel()` initializes major kernel systems.

Examples:

## Interrupt System

```c
trap_init();
```

Sets up CPU exception handling.

---

## Memory Management

```c
mm_init();
```

Initializes memory allocation systems.

---

## Scheduler

```c
sched_init();
```

Prepares process scheduling.

---

## First Process

The kernel creates the first process structure:

```c
init_task
```

This represents:

```text
PID 0
```

---

# 10. Creating PID 1

At the end of boot:

```c
arch_call_rest_init();
```

creates a new kernel thread.

This becomes:

```text
PID 1
```

Eventually, PID 1 becomes the first user-space process:

Examples:

```text
systemd
init
```

---

# 11. What Happens to PID 0?

After creating PID 1, the original boot thread becomes the idle process.

PID 0 enters:

```text
cpu_startup_entry()
```

Its job is simple:

* Wait when no processes need CPU time.
* Execute the CPU `HLT` instruction to save power.

The boot process is complete.

---

# 12. Summary

The Linux boot process is a carefully controlled transition:

```text
16-bit Real Mode
        |
        v
32-bit Protected Mode
        |
        v
64-bit Long Mode
        |
        v
Enable Paging
        |
        v
Create Stack
        |
        v
Decompress Kernel
        |
        v
start_kernel()
        |
        v
Initialize OS
```

The kernel starts with assembly code because the CPU environment is not ready for C.

Once basic hardware support exists, execution moves to C code, where the kernel initializes memory management, interrupts, scheduling, and finally starts the first user process.

---

# Mental Model

Think of kernel booting like building a spaceship during launch.

* **Assembly code** → Builds the basic tools.
* **Paging and stack setup** → Creates the working environment.
* **start_kernel()** → Starts the main systems.
* **PID 1** → Starts normal operation.
* **PID 0** → Becomes the idle worker waiting for tasks.

---

# Important Functions

| Function                | Purpose                      |
| ----------------------- | ---------------------------- |
| `startup_64`            | 64-bit assembly entry point  |
| `start_kernel()`        | Main C kernel initialization |
| `arch_call_rest_init()` | Creates PID 1                |
| `cpu_startup_entry()`   | Starts the idle loop         |

---

# Important Structures

## `struct boot_params`

Contains boot information passed from the bootloader.

Includes:

* Memory map.
* Hardware details.
* Kernel command line.

---

# Code Reading Exercises

## Exercise 1

Open:

```text
init/main.c
```

Find:

```c
start_kernel()
```

Observe how it initializes major kernel subsystems.

---

## Exercise 2

Inside `start_kernel()`, find:

```c
set_task_stack_end_magic(&init_task);
```

This places a magic value at the end of the stack to detect stack corruption.

---

## Exercise 3

Open:

```text
include/linux/init.h
```

Search:

```c
__init
```

Observe how it places boot-only code into:

```text
.init.text
```

for later memory cleanup.

---

# Questions for Understanding

## 1. Why decompress the kernel in memory?

The boot image is compressed to reduce storage size and loading time.

The kernel must be decompressed into memory before execution.

---

## 2. Why can't the bootloader directly call `start_kernel()`?

Because the CPU is not ready.

Before C execution:

* Stack must exist.
* CPU must enter 64-bit mode.
* Paging must be enabled.

Assembly performs this preparation.

---

## 3. What happens to `__init` functions after boot?

The memory containing these functions is released after initialization.

This saves RAM.

---
---

# Suggested Next Files to Read

Now that the kernel is running, the next step is understanding process management.

Read:

* `include/linux/sched.h`

  * Definition of `task_struct`

* `kernel/fork.c`

  * Process creation logic

```
