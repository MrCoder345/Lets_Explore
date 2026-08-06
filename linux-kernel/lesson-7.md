# Lesson 7: Kernel Memory Allocation and System Calls

In **Lesson 6**, we learned how Linux manages memory for **user-space programs** using:

- Virtual Memory Areas (VMAs)
- Demand Paging
- Page Faults

But the kernel itself is also just a C program.

That raises two important questions:

- When `kernel_clone()` creates a new `task_struct`, **where does that memory come from?**
- How does a user-space program actually ask the kernel to perform operations like `clone()`, `read()`, or `write()`?

In this lesson, we'll answer both.

---

# Two Topics We'll Cover

## 1. Kernel Memory Allocation

How the kernel allocates memory for its own internal objects using:

```text
kmalloc()
```

---

## 2. System Calls

How user-space safely enters kernel-space through the **System Call Interface**.

This is one of the most important security boundaries in an operating system.

---

# Part 1: Kernel Memory Allocation

---

# Why Can't the Kernel Use the Buddy Allocator Directly?

In **Lesson 6**, we learned that the Buddy Allocator (`__alloc_pages()`) manages memory in **4 KB pages**.

Suppose the kernel only needs:

```text
64 bytes
```

for a small data structure.

If it requested one page directly:

```text
Allocated : 4096 bytes
Used      :   64 bytes
Wasted    : 4032 bytes
```

That would waste a huge amount of RAM.

---

# The Solution: SLAB / SLUB Allocator

Linux solves this using a higher-level allocator called the **Slab Allocator**.

The modern implementation is:

```text
SLUB
```

located in:

```text
mm/slub.c
```

Kernel code normally allocates memory using:

```c
kmalloc(size, flags)
```

instead of requesting pages directly.

---

# How SLUB Works

The allocator works in three main steps.

### Step 1

Request one or more pages from the Buddy Allocator.

```text
Buddy Allocator
        │
        ▼
4 KB Page
```

---

### Step 2

Split those pages into many equal-sized objects.

Example:

```text
4 KB Page

+----+----+----+----+----+
|64B |64B |64B |64B |64B |
+----+----+----+----+----+
```

Different slab caches exist for different object sizes.

Examples:

- 32-byte objects
- 64-byte objects
- 128-byte objects
- `task_struct` objects
- inode objects
- dentry objects

---

### Step 3

When the kernel calls:

```c
kmalloc(64, GFP_KERNEL)
```

SLUB immediately returns one free 64-byte object.

No page splitting is needed at allocation time because the slabs were already prepared.

This makes allocation extremely fast.

---

# `kmalloc()` vs `malloc()`

Although their names look similar, they behave very differently.

| `kmalloc()` | `malloc()` |
|-------------|------------|
| Used inside the kernel | Used in user-space |
| Returns physically allocated memory immediately | Usually returns only virtual memory |
| Memory is physically contiguous | Physical pages are allocated later (Demand Paging) |
| No page faults required | May trigger a page fault on first access |
| Uses SLUB | Uses the C library allocator (glibc, musl, etc.) |

---

# Important Difference

Unlike `malloc()`,

```c
kmalloc()
```

does **not** use lazy allocation.

When it returns,

- Physical memory already exists.
- The memory is contiguous.
- It is ready to use immediately.

---

# The Kernel Trusts Itself

User-space has many safety limits.

The kernel generally does not.

If kernel code repeatedly executes:

```c
while (1)
    kmalloc(...);
```

eventually the system can run out of memory and panic.

The kernel assumes its own code is trusted and written correctly.

---

# Part 2: Crossing the User ↔ Kernel Boundary

User-space and kernel-space run at different privilege levels.

| Mode | CPU Ring |
|------|----------|
| User-space | Ring 3 |
| Kernel-space | Ring 0 |

User-space **cannot** directly call kernel functions.

For example, this is impossible:

```text
User Program
      ↓
copy_process()
```

The CPU blocks such attempts to protect the operating system.

Instead, user-space must go through a controlled gateway called a:

```text
System Call
```

---

# System Call Flow

```text
[ User Space (Ring 3) ]

printf()
    │
    ▼
libc write()
    │
    ▼
syscall instruction
    │
────────────────────────────────────
      CPU switches to Ring 0
────────────────────────────────────
    │
    ▼
entry_SYSCALL_64
    │
    ▼
do_syscall_64()
    │
    ▼
sys_write()
```

This is the standard path from user-space into the kernel.

---

# Step 1: The Hardware Trigger

Suppose a program executes:

```c
write(fd, buf, count);
```

The C library (such as glibc) prepares the system call.

It places:

- System call number into `%rax`
- Arguments into predefined CPU registers

Then it executes the special CPU instruction:

```text
syscall
```

This instruction is implemented by the processor itself.

---

# What Does `syscall` Do?

The CPU performs several operations automatically.

1. Saves the current user instruction pointer (`%rip`)
2. Switches from Ring 3 to Ring 0
3. Loads the kernel stack
4. Jumps to a kernel entry address registered during boot

All of this happens in hardware.

---

# Step 2: Assembly Entry Point

Execution begins at:

```text
arch/x86/entry/entry_64.S
```

Specifically:

```text
entry_SYSCALL_64
```

At this point, the kernel has just entered Ring 0.

---

## Why Switch Stacks?

The kernel **must not** continue using the user-space stack.

The user stack could be:

- Corrupted
- Invalid
- Unmapped
- Intentionally malicious

Instead, every process owns its own **Kernel Stack**.

Typical size:

```text
8 KB or 16 KB
```

The assembly code immediately changes:

```text
%rsp
```

to point to that safe kernel stack.

Then it saves all user registers before calling C code.

---

# Step 3: `do_syscall_64()`

Now execution enters C code.

Function:

```c
do_syscall_64()
```

Purpose:

- Read the syscall number from `%rax`
- Look it up in the syscall table
- Call the correct kernel function

Example:

```text
%rax = 1
        │
        ▼
sys_call_table
        │
        ▼
sys_write()
```

The syscall table is simply an array that maps syscall numbers to kernel functions.

---

# Part 3: Important Kernel Idioms

---

# The `__user` Annotation

Consider the `write()` system call.

```c
SYSCALL_DEFINE3(write,
    unsigned int, fd,
    const char __user *, buf,
    size_t, count)
{
    ...
}
```

Notice:

```c
__user
```

This annotation tells static analysis tools:

> This pointer came from user-space.

Treat it as **untrusted**.

---

# Golden Rule

**Never directly dereference a `__user` pointer.**

For example:

```c
char c = buf[0];
```

This is **wrong** inside kernel code.

---

# Why Is It Dangerous?

A malicious program could pass:

- An invalid pointer
- An unmapped address
- A kernel address
- Memory that triggers a page fault

If the kernel blindly trusted it, the entire operating system could crash or be compromised.

---

# The Correct Way

Always use:

```c
copy_from_user()
```

Example:

```c
char kernel_buffer[256];

if (copy_from_user(kernel_buffer, buf, count)) {
    return -EFAULT;
}
```

This safely copies data from user-space while properly handling faults.

Likewise, to return data back to user-space, the kernel uses:

```c
copy_to_user()
```

---

# `SYSCALL_DEFINE` Macros

Instead of writing:

```c
long sys_write(...)
```

Linux uses:

```c
SYSCALL_DEFINE3(...)
```

where the number indicates the number of arguments.

Examples:

```c
SYSCALL_DEFINE0(...)
SYSCALL_DEFINE1(...)
SYSCALL_DEFINE2(...)
SYSCALL_DEFINE3(...)
```

---

## Why Use a Macro?

These macros do much more than generate a function declaration.

They also help:

- Validate arguments
- Handle architecture-specific details
- Properly extend 32-bit values to 64-bit values
- Prevent several classes of security vulnerabilities (including CVE-2009-0029)

So every system call in Linux follows a consistent and secure implementation pattern.

---

# Summary

- The Buddy Allocator works with pages (minimum 4 KB).
- Small kernel allocations are handled by the **SLUB allocator**.
- Kernel code requests memory using `kmalloc()`.
- `kmalloc()` returns immediately allocated, physically contiguous memory.
- User-space cannot directly execute kernel functions.
- User-space enters the kernel through the **`syscall` instruction**.
- The CPU switches from Ring 3 to Ring 0 automatically.
- The kernel switches to a dedicated kernel stack before doing any work.
- `do_syscall_64()` routes requests using the syscall table.
- User pointers are always treated as untrusted and accessed only through `copy_from_user()` or `copy_to_user()`.

---

# Mental Model

Imagine a bank.

- User-space = Customer
- Kernel = Vault
- `syscall` = Bank buzzer
- Registers = Request form
- `entry_SYSCALL_64` = Security guard
- `copy_from_user()` = Teller verifying your documents
- Kernel stack = Secure work desk inside the vault

Customers are never allowed inside the vault.

Instead, they submit a request through a secure process, and only validated information is allowed to enter.

---

# Important Functions to Remember

- `kmalloc()`
- `kfree()`
- `copy_from_user()`
- `copy_to_user()`
- `do_syscall_64()`

---

# Important Structs / Macros to Remember

- `__user`
- `SYSCALL_DEFINE0()`
- `SYSCALL_DEFINE1()`
- `SYSCALL_DEFINE2()`
- `SYSCALL_DEFINE3()`

---

# Code Reading Exercises

## Exercise 1

Open:

```text
include/linux/slab.h
```

Find:

```c
kmalloc()
```

Notice the argument:

```c
gfp_t flags
```

(**GFP = Get Free Page**)

These flags tell the allocator how aggressively it may search or wait for memory.

---

## Exercise 2

Search for:

```text
SYSCALL_DEFINE3(read
```

You should find it in:

```text
fs/read_write.c
```

Notice how the buffer argument is marked with:

```c
__user
```

---

## Exercise 3

Open:

```text
arch/x86/entry/syscalls/syscall_64.tbl
```

This file is the master list of all x86_64 system calls.

Look at entries:

```text
0  read
1  write
2  open
```

Observe how each syscall number maps to its implementation.

---

# Questions for Understanding

### 1.

If `kmalloc()` requests a page from the Buddy Allocator but only needs **32 bytes**, what happens to the remaining space inside that page?

---

### 2.

Suppose a user program passes a pointer from `malloc()` to `sys_write()`, but that memory has never been accessed before.

Since no physical page exists yet, what happens when `copy_from_user()` tries to read it?

---

### 3.

Why does the kernel switch from the user stack to the kernel stack immediately after entering through a system call?

What security or stability problems could occur if it continued using the user's stack?

---

# Suggested Next Files to Read

System calls are **voluntary** entries into the kernel.

Next, learn about **interrupts**, where hardware forces the CPU to enter the kernel unexpectedly.

Read:

```text
arch/x86/kernel/irq.c
```

- Low-level interrupt handling

```text
kernel/irq/handle.c
```

- High-level interrupt processing
