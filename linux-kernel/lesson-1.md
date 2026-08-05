
# Linux Kernel Exploration — Lesson 1  
## Repository Structure and Navigation

Welcome to the Linux kernel.


The challenge of kernel development is not learning C again. The challenge is understanding:

- the enormous codebase
- concurrency
- hardware abstraction
- kernel-specific design patterns
- subsystem interactions

This course will approach the Linux kernel as an **architectural exploration**, not as a memorization exercise.

---

# 1. Getting the Source Code

The official Linux kernel repository is maintained by Linus Torvalds:

Repository:

https://github.com/torvalds/linux

Clone it:

```bash
git clone https://github.com/torvalds/linux.git
cd linux
````

The repository contains:

* architecture-specific code
* process management
* memory management
* filesystems
* networking
* device drivers
* kernel build infrastructure

The first challenge is learning how to navigate it.

---

# 2. Why Does the Kernel Have This Structure?

Linux uses a **monolithic kernel architecture**.

This means:

* core operating system services run in kernel space
* components share the same address space
* communication happens through direct function calls instead of message passing

The advantage:

* extremely fast interaction between subsystems

The disadvantage:

* the codebase becomes enormous

The directory structure exists to maintain separation of responsibility.

Instead of putting everything into one huge folder, Linux separates:

* architecture code
* generic kernel logic
* hardware drivers
* filesystems
* memory management

---

# 3. Linux Kernel Architecture Overview

```text
                    User Space
        Applications / Shell / glibc
                         |
                         |
              System Call Interface
                  arch/ + kernel/
                         |
 ------------------------------------------------
 |              |              |                |
 fs/            net/          kernel/           mm/
Filesystems   Networking    Scheduler       Memory
 VFS          TCP/IP        Processes       Virtual Memory
 ext4         Sockets       IPC             Paging
 btrfs                       Timers          Allocators
 ------------------------------------------------
                         |
                    Drivers
                 drivers/
                         |
 ------------------------------------------------
                         |
                    Hardware
              CPU / RAM / Disk / Devices
```

---

# 4. Important Top-Level Directories

## `arch/`

Architecture-specific implementation.

Examples:

```
arch/x86/
arch/arm64/
arch/riscv/
arch/powerpc/
```

The generic kernel may say:

> Switch from the current task to another task.

The architecture code performs the actual CPU operations:

* saving registers
* restoring registers
* switching stacks
* handling interrupts

Example:

```
arch/x86/kernel/
```

contains x86-specific implementations.

---

## `init/`

Kernel startup code.

The kernel begins execution here after the bootloader transfers control.

Important file:

```
init/main.c
```

Important function:

```c
start_kernel()
```

This is the main C entry point of the Linux kernel.

---

## `kernel/`

The core operating system logic.

Contains:

* scheduler
* process management
* locking primitives
* timers
* signals
* kernel threads

Examples:

```
kernel/sched/
kernel/fork.c
kernel/time/
kernel/locking/
```

---

## `mm/

Memory management subsystem.

Responsible for:

* virtual memory
* physical page allocation
* paging
* swapping
* memory mapping

Important areas:

```
mm/page_alloc.c
mm/memory.c
mm/slab.c
```

If you want to understand:

> How does Linux allocate RAM?

Start here.

---

## `fs/`

Filesystem subsystem.

Linux uses the Virtual File System (VFS) abstraction.

The VFS allows applications to interact with different filesystems using the same API.

Examples:

```
fs/ext4/
fs/btrfs/
fs/fat/
```

The abstraction looks like:

```
Application
     |
     |
    VFS
     |
 -----------------
 |       |       |
ext4   btrfs    fat
```

Linux filesystem design is a good example of object-oriented programming implemented in C.

---

## `drivers/`

The largest directory in the kernel.

Contains hardware support:

Examples:

```
drivers/net/
drivers/usb/
drivers/gpu/
drivers/block/
drivers/input/
```

Why is it so large?

Because every hardware vendor needs support code:

* graphics cards
* WiFi chips
* storage controllers
* USB devices
* sound cards
* sensors

The kernel core is relatively generic.

Drivers are where hardware diversity explodes.

---

## `include/`

Kernel header files.

Contains:

* structures
* macros
* inline functions
* function declarations

Examples:

```
include/linux/sched.h
include/linux/mm.h
include/linux/fs.h
```

Important structure:

```c
struct task_struct
```

Located in:

```
include/linux/sched.h
```

This represents a Linux process/thread.

---

# 5. How Experts Navigate the Kernel

Reading Linux source code requires more than a text editor.

---

# Tool 1: clangd + LSP

Modern approach.

Generate:

```
compile_commands.json
```

using:

```bash
make LLVM=1 compile_commands.json
```

or:

```bash
python3 scripts/clang-tools/gen_compile_commands.py
```

Benefits:

* jump to definitions
* find references
* understand structures
* navigate macros

Works with:

* VS Code
* Vim
* Neovim
* Emacs

---

# Tool 2: cscope

Traditional kernel navigation tool.

Generate database:

```bash
make cscope
```

Useful commands:

Find function:

```
Ctrl + \
```

Find callers:

```
cscope -d
```

---

# Tool 3: ctags

Generate tags:

```bash
make tags
```

Then editors can jump directly to:

* functions
* structs
* macros

---

# Tool 4: ripgrep (`rg`)

Fast source searching.

Install:

```bash
sudo apt install ripgrep
```

Example:

Find `task_struct`:

```bash
rg -t c "struct task_struct {"
```

Find all calls:

```bash
rg "schedule\("
```

---

# Tool 5: Linux Cross Reference

A very useful online source browser:

[https://elixir.bootlin.com/linux/latest/source](https://elixir.bootlin.com/linux/latest/source)

It allows:

* searching functions
* finding callers
* following macros
* exploring different kernel versions

---

# 6. First Files To Explore

## 1. Root Makefile

Path:

```
Makefile
```

Purpose:

Understand:

* kernel build process
* compiler options
* architecture selection

---

## 2. Kconfig

Path:

```
Kconfig
```

Purpose:

Understand:

* kernel configuration system
* enabling/disabling features

Example:

```bash
make menuconfig
```

creates configuration from these files.

---

## 3. Kernel Startup

File:

```
init/main.c
```

Function:

```c
start_kernel()
```

This is where Linux begins initializing:

Example flow:

```
start_kernel()
 |
 +-- setup_arch()
 |
 +-- trap_init()
 |
 +-- mm_init()
 |
 +-- sched_init()
 |
 +-- rest_init()
```

---

# 7. Important Structures

## `struct task_struct`

Location:

```
include/linux/sched.h
```

Represents:

* processes
* threads
* scheduling state
* memory information
* signals
* credentials

Example:

```c
struct task_struct {
    volatile long state;
    void *stack;
    pid_t pid;
};
```

The real structure is much larger.

It connects many kernel subsystems together.

---

# 8. Mental Model

Think of Linux source code as an hourglass:

```
             User Applications
                    |
                    |
             System Calls
                    |
              kernel/
              mm/
              fs/
              net/
                    |
                    |
          Hardware-specific Code
              arch/
              drivers/
                    |
                    |
              Physical Hardware
```

The middle contains highly reusable generic code.

The bottom expands into thousands of hardware implementations.

---

# Exercises

## Exercise 1

Open:

```
include/linux/sched.h
```

Find:

```c
struct task_struct
```

Do not try to understand every field.

Observe:

* how many subsystems depend on it
* how large it is
* how interconnected Linux is

---

## Exercise 2

Open:

```
init/main.c
```

Find:

```c
start_kernel()
```

Read the initialization sequence.

---

## Exercise 3

Explore:

```
arch/
```

Find your CPU architecture.

Examples:

```
arch/x86/
arch/arm64/
```

Look inside:

```
boot/
```

---

# Understanding Questions

## Question 1

Why are headers separated into:

```
include/linux/
```

instead of keeping them beside:

```
kernel/
mm/
fs/
```

?

---

## Question 2

If you want to understand physical page allocation, where do you start?

Answer:

```
mm/
```

---

## Question 3

Why is:

```
drivers/
```

larger than:

```
kernel/
```

?

Think about:

* number of hardware devices
* vendor-specific implementations
* hardware differences

---

# Next Lesson

Recommended files:

1. Root `Makefile`

```
Makefile
```

2. Kernel configuration system

```
Kconfig
```

3. Kernel entry point

```
init/main.c
```
