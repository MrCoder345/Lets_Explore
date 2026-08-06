# Lesson 6: Linux Memory Management

In **Lesson 5**, we saw that `context_switch()` calls `switch_mm()` to switch a process's virtual memory space.

Now it's time to understand **how that memory system actually works**.

Welcome to the **`mm/` (Memory Management)** subsystem.

Memory management is one of the most complex parts of the Linux kernel because it must solve **two major problems at the same time**.

---

# Two Major Goals of Memory Management

## 1. Isolation and the Illusion of Memory

Every user-space process should believe that:

- It owns one large, continuous block of memory.
- No other process can access its memory.

The kernel guarantees this isolation, making it impossible for normal user processes to read or modify another process's memory.

---

## 2. Physical Memory Fragmentation

Physical RAM is constantly changing.

Processes:

- Start
- Allocate memory
- Free memory
- Exit

Over time, free RAM becomes fragmented, like Swiss cheese.

The kernel must still be able to find **contiguous physical memory** when hardware (such as DMA devices) requires it.

---

# Two Worlds of Memory

Linux separates memory into two completely different views.

| Physical Memory | Virtual Memory |
|-----------------|----------------|
| Actual RAM chips | Address space seen by a process |
| Used by hardware | Used by applications |
| Managed by the Buddy Allocator | Managed using VMAs and page tables |

Think of it like this:

```text
Application
      |
Virtual Address
      |
Page Tables
      |
Physical RAM
```

Applications never work directly with physical memory.

---

# Part 1: Physical Memory

## Memory Is Divided into Pages

Linux manages physical memory in fixed-size blocks called **Pages**.

On most architectures (including x86_64):

```text
1 Page = 4 KB = 4096 bytes
```

Every page represents one physical frame in RAM.

---

## `struct page`

For **every single physical page**, Linux creates one:

```text
struct page
```

This structure stores information about that page.

Examples of information it tracks:

- Whether the page is free
- Whether it is allocated
- Reference count
- Mapping information
- Various status flags

If your system has:

```text
16 GB RAM
```

Then it contains:

```text
4,194,304 pages
```

which means Linux also maintains:

```text
4,194,304 struct page
```

objects inside the kernel.

---

# The Problem: Memory Fragmentation

Imagine a device driver needs:

```text
16 KB
```

of **contiguous physical memory** for DMA.

Suppose RAM looks like this:

```text
Free  Used  Free  Used  Free  Used  Free
```

There are many free pages, but none are next to each other.

Even though enough total memory exists, the request cannot be satisfied.

This is called **physical fragmentation**.

---

# The Solution: Buddy Allocator

File:

```text
mm/page_alloc.c
```

Linux solves fragmentation using the **Buddy Allocator**.

Instead of managing random page sizes, memory is grouped into blocks whose sizes are powers of two.

---

## Orders

| Order | Pages | Size |
|-------|------:|-----:|
| 0 | 1 | 4 KB |
| 1 | 2 | 8 KB |
| 2 | 4 | 16 KB |
| 3 | 8 | 32 KB |
| ... | ... | ... |
| 10 | 1024 | 4 MB |

---

# How Splitting Works

Suppose you request:

```text
Order 1 (8 KB)
```

but only an:

```text
Order 2 (16 KB)
```

block is available.

The Buddy Allocator simply splits it.

```text
[                 Order 2 (16 KB)                   ]
       /                                     \
[  Order 1 (8 KB)  ]                 [  Order 1 (8 KB)  ]
 (Allocated to you)                   (Added to free list)
```

The two 8 KB blocks become **buddies**.

---

# How Merging Works

Later, when your 8 KB block is freed:

Linux checks whether its buddy is also free.

If yes:

```text
8 KB + 8 KB
      ↓
16 KB
```

They are merged back into one larger block.

This automatic merging helps reduce fragmentation over time.

One of the biggest advantages of the Buddy Allocator is that buddy blocks can be found mathematically in **O(1)** time.

---

## Important Function

The heart of physical page allocation is:

```c
__alloc_pages()
```

Eventually, almost every request for physical RAM reaches this function.

---

# Part 2: Virtual Memory

Files:

```text
mm/mmap.c
mm/memory.c
```

Applications do **not** use physical memory directly.

Instead, they use **virtual addresses**.

The kernel translates those virtual addresses into physical memory.

---

# `struct mm_struct`

From **Lesson 4**, remember that every process has:

```c
task_struct
```

Inside it is:

```c
*mm
```

which points to:

```text
struct mm_struct
```

This structure represents the **entire virtual address space** of a process.

---

# Virtual Memory Areas (VMAs)

Inside `struct mm_struct` is a collection of:

```text
struct vm_area_struct
```

Each VMA represents one continuous range of virtual memory.

Examples:

- Executable code
- Heap
- Stack
- Shared libraries
- Memory-mapped files

Example:

```text
[ struct mm_struct ]
          |
          v
[ VMA: 0x400000 - 0x401000 (Executable Code) ]
[ VMA: 0x600000 - 0x602000 (Heap, Read/Write) ]
[ VMA: 0x7FFF00 - 0x7FFFF0 (Stack, Read/Write) ]
```

Each process owns its own set of VMAs.

---

# Modern Kernel Note

Older Linux kernels stored VMAs inside a:

```text
Red-Black Tree
```

Starting from **Linux 6.1**, this was replaced with the:

```text
Maple Tree
```

The Maple Tree is a highly optimized B-tree variant designed for:

- Better cache efficiency
- Faster lookups
- Lockless reads using RCU
- Better scalability on modern CPUs

You'll learn about **RCU** in a later lesson.

---

# The Biggest Illusion: Demand Paging

A common misconception is:

> Calling `malloc()` immediately allocates physical RAM.

This is **not true**.

Example:

```c
malloc(1024 * 1024 * 100);
```

Requesting **100 MB** does **not** immediately allocate 100 MB of RAM.

Instead, Linux simply creates a new:

```text
struct vm_area_struct
```

covering that virtual address range.

No physical pages are allocated yet.

This is called:

```text
Demand Paging
```

or

```text
Lazy Allocation
```

---

# When Is Physical Memory Actually Allocated?

Physical memory is allocated only when the process actually **uses** that memory.

Example:

```c
*ptr = 42;
```

This triggers the complete page fault process.

---

# Page Fault Flow

## Step 1

Your program writes to memory.

```c
*ptr = 42;
```

---

## Step 2

The CPU's MMU looks inside the page tables to translate the virtual address.

---

## Step 3

No mapping exists.

The MMU raises a hardware exception:

```text
Page Fault
```

Execution immediately enters the kernel.

---

## Step 4

The kernel executes:

```c
handle_mm_fault()
```

located in:

```text
mm/memory.c
```

---

## Step 5

The kernel checks:

> Is this address inside a valid VMA?

If not,

```text
Segmentation Fault
```

If yes,

continue.

---

## Step 6

The kernel requests a physical page by calling:

```c
__alloc_pages()
```

---

## Step 7

The kernel updates the page tables.

Now:

```text
Virtual Address
        ↓
Physical Page
```

has a valid mapping.

---

## Step 8

The kernel resumes the process exactly where it stopped.

---

## Step 9

The instruction

```c
*ptr = 42;
```

runs again.

This time it succeeds.

From the program's perspective, nothing unusual happened.

---

# Useful Kernel Tricks

## Page Alignment

Since every page is:

```text
4 KB = 2¹² bytes
```

the lower 12 bits of every page-aligned address are always:

```text
0
```

Example:

```text
0x1000
0x2000
0x3000
```

The kernel takes advantage of these unused bits.

Instead of wasting memory, it stores small boolean flags inside those lower bits and masks them out whenever it needs the real address.

This is a common optimization throughout the kernel.

---

## `PAGE_ALIGN()`

The kernel frequently rounds addresses to the nearest page boundary.

Example:

```c
#define PAGE_ALIGN(addr)  ALIGN(addr, PAGE_SIZE)
```

Internally, it often becomes something like:

```c
(addr + PAGE_SIZE - 1) & ~(PAGE_SIZE - 1)
```

This rounds an address **up** to the next 4 KB boundary.

---

# Summary

- Physical memory is divided into 4 KB pages.
- Every physical page is represented by a `struct page`.
- The Buddy Allocator manages physical memory and reduces fragmentation by splitting and merging power-of-two blocks.
- Every process has a `struct mm_struct`, which represents its virtual address space.
- Virtual memory is divided into **VMAs (`struct vm_area_struct`)**.
- `malloc()` usually allocates **virtual memory only**.
- Physical RAM is allocated later through **Demand Paging** when a page fault occurs.
- `handle_mm_fault()` connects virtual addresses to physical pages by updating the page tables.

---

# Mental Model

Think of **Virtual Memory** as writing a check.

When you call:

```c
malloc()
```

the kernel gives you a check promising memory.

No real cash changes hands yet.

Only when you actually spend that check (access the memory) does the kernel withdraw real cash from the bank:

- Cash = Physical RAM
- Bank = Buddy Allocator
- Teller = `handle_mm_fault()`

This is why large `malloc()` calls are usually very fast.

---

# Important Functions to Remember

- `__alloc_pages()`
- `handle_mm_fault()`
- `mmap_region()`

---

# Important Structs to Remember

- `struct page`
- `struct mm_struct`
- `struct vm_area_struct`

---

# Code Reading Exercises

## Exercise 1

Open:

```text
include/linux/mm_types.h
```

Find:

```c
struct page
```

Notice the heavy use of:

```c
union
```

Since millions of `struct page` objects exist, Linux overlaps mutually exclusive fields to save memory.

---

## Exercise 2

In the same file, find:

```c
struct vm_area_struct
```

Look for:

```c
vm_start
vm_end
vm_flags
```

Understand:

- Virtual address range
- Memory permissions (Read / Write / Execute)

---

## Exercise 3

Open:

```text
mm/page_alloc.c
```

Find:

```c
__alloc_pages()
```

Observe how it first tries a **fast path**.

If that fails, it switches to a **slow path**, where Linux may reclaim memory before retrying the allocation.

---

# Questions for Understanding

### 1.

When a process calls `fork()` (using `clone()` without `CLONE_VM`), the child initially receives a copy of the parent's memory.

How can Linux avoid copying gigabytes of RAM immediately?

**Hint:** Look up **Copy-on-Write (CoW)**.

---

### 2.

Why does the Buddy Allocator allocate memory only in powers of two instead of allocating the exact number of requested bytes?

---

### 3.

Since `malloc()` mostly allocates virtual memory, how does Linux prevent processes from reserving more memory than the system can actually provide?

**Hint:** Research **Memory Overcommit**.

---

# Suggested Next Files to Read

We've seen how user-space obtains memory.

Next, learn how the **kernel allocates memory for itself**, such as `task_struct`, inode caches, and other internal objects.

Read:

```text
mm/slab.c
```

or

```text
mm/slub.c
```

- Slab Allocator (`kmalloc`)

Then continue with:

```text
arch/x86/entry/entry_64.S
```

- How hardware exceptions (including page faults) and system calls enter the kernel.
