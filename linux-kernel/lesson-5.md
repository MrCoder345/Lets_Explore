# Lesson 5: The Scheduler Framework and CFS

Now that a `task_struct` has been created, it exists in memory with a state like `TASK_NEW` or `TASK_RUNNING`.

However, **a CPU core can execute only one instruction stream at a time**.

So what happens if you have:

- 4 CPU cores
- 1,000 runnable tasks

The scheduler decides:

- Which task runs next
- How long it runs
- How CPU time is shared fairly
- How interactive tasks (mouse, keyboard, UI) stay responsive while background tasks (compiling, encoding) continue making progress.

Welcome to `kernel/sched/`.

---

# Scheduler Overview

The Linux scheduler is **not a single scheduling algorithm**.

Instead, it is a **scheduler framework**, mainly controlled by:

```text
kernel/sched/core.c
```

It organizes tasks into **Scheduling Classes**, which are checked in **priority order**.

| Priority | Scheduling Class | Purpose |
|----------|------------------|---------|
| 1 | Stop / Deadline | Kernel internals and hard real-time tasks |
| 2 | Real-Time (RT) | POSIX real-time scheduling (`kernel/sched/rt.c`) |
| 3 | Fair (CFS / EEVDF) | Normal user processes (`kernel/sched/fair.c`) |
| 4 | Idle | CPU idle task (PID 0) |

For almost every normal Linux program, the **Fair Scheduler** is used.

---

# CFS → EEVDF

Older Linux kernels used:

> **CFS (Completely Fair Scheduler)**

Since Linux **6.6+**, CFS has been upgraded to:

> **EEVDF (Earliest Eligible Virtual Deadline First)**

Although the scheduling algorithm changed internally, the **main data structures and overall concepts remain almost the same**.

So learning CFS is still the best way to understand the scheduler.

---

# The Main Idea: `vruntime`

Instead of giving every task a fixed time slice (for example, 10 ms each), Linux imagines an **ideal CPU** where every runnable task progresses fairly.

To achieve this, every task keeps a value called:

```text
vruntime (Virtual Runtime)
```

Rules:

- When a task runs → `vruntime` increases.
- When a task sleeps (waiting for disk, network, etc.) → `vruntime` stops increasing.
- The scheduler always chooses the task with the **smallest `vruntime`**.

This is the core idea behind fair scheduling.

---

## Nice Value and Priority

Not every task should receive exactly the same amount of CPU time.

Linux adjusts `vruntime` using the task's **nice value**.

- Higher priority (lower nice value)
  - `vruntime` increases **more slowly**
  - Task stays on the CPU longer

- Lower priority (higher nice value)
  - `vruntime` increases **faster**
  - Task gives up CPU sooner

So priority affects **how quickly virtual runtime grows**, not simply the length of a time slice.

---

# Important Data Structures

## 1. `struct sched_entity`

Just like `task_struct` embeds `list_head`, scheduling information is embedded inside:

```c
struct sched_entity {
    struct load_weight      load;           /* Priority weight */
    struct rb_node          run_node;       /* Tree node */
    u64                     vruntime;       /* The core metric */
    /* ... */
};
```

Important fields:

| Field | Purpose |
|-------|----------|
| `load` | Priority weight |
| `run_node` | Node inside the Red-Black Tree |
| `vruntime` | Used to decide which task runs next |

---

## 2. `struct rq` (Runqueue)

Imagine every CPU core has its own waiting room.

That waiting room is called a **Runqueue**.

```text
struct rq
```

Each CPU has **its own** runqueue.

Example:

```text
CPU0
 └── rq

CPU1
 └── rq

CPU2
 └── rq

CPU3
 └── rq
```

When a task wakes up, Linux decides which CPU should execute it and places the task into that CPU's runqueue.

### Why separate runqueues?

Instead of thousands of tasks competing for one giant lock, every CPU manages its own queue.

Benefits:

- Less lock contention
- Better scalability
- Better cache performance
- Faster scheduling

---

## 3. Red-Black Tree (`rb_root`)

Inside each `cfs_rq`, runnable tasks are stored in a:

```text
Red-Black Tree
```

The tree is sorted using:

```text
vruntime
```

The task with the **lowest `vruntime`** is always the **leftmost node**.

Example:

```text
          30
         /
       20
      /
    10   <- leftmost
```

The scheduler simply picks the leftmost node.

### Why use a Red-Black Tree?

Operation | Complexity
---------|------------
Insert | `O(log N)`
Remove | `O(log N)`
Find smallest `vruntime` | `O(1)` (leftmost node)

This makes scheduling efficient even with thousands of tasks.

---

# How Does Scheduling Start?

The scheduler is entered in two main ways.

---

## 1. Voluntary Scheduling

A task willingly gives up the CPU.

Example:

```c
sleep();
```

or waiting for disk/network input.

The kernel changes its state to:

```text
TASK_UNINTERRUPTIBLE
```

Then it calls:

```c
schedule();
```

---

## 2. Involuntary Scheduling (Preemption)

Sometimes a task runs for too long.

A hardware timer interrupts the CPU many times every second.

If the scheduler decides another task should run, it sets:

```text
TIF_NEED_RESCHED
```

When the kernel returns to user space, it calls:

```c
schedule();
```

This is called **preemption**.

---

# Core Scheduling Flow

The heart of scheduling is:

```text
__schedule()
```

located in:

```text
kernel/sched/core.c
```

Simplified flow:

```c
static void __sched notrace __schedule(unsigned int sched_mode)
{
    struct task_struct *prev, *next;
    struct rq *rq;

    prev = current;
    rq = this_rq();

    if (prev_state != TASK_RUNNING)
        deactivate_task(rq, prev, ...);

    next = pick_next_task(rq, prev, ...);

    if (likely(prev != next)) {
        rq->curr = next;
        context_switch(rq, prev, next, ...);
    }
}
```

---

## Step 1

```c
prev = current;
```

Current running task.

---

## Step 2

```c
rq = this_rq();
```

Get the current CPU's runqueue.

---

## Step 3

If the current task is sleeping,

```c
deactivate_task(...)
```

removes it from the runqueue.

---

## Step 4

```c
pick_next_task()
```

asks every scheduling class to provide the next runnable task.

---

## Step 5

If the selected task is different,

```c
context_switch(...)
```

switches execution to the new task.

---

# What Happens Inside `context_switch()`?

This is where the actual task switch occurs.

It performs two major operations.

---

## 1. `switch_mm()`

Changes the active page tables inside the CPU's MMU.

After this call:

- Old process memory disappears.
- New process memory becomes visible.

The CPU is now using the new process's virtual address space.

---

## 2. `switch_to()`

This is architecture-specific assembly code.

It:

- Saves CPU registers of the old task.
- Restores CPU registers of the new task.

Registers include things like:

- Stack Pointer (SP)
- Instruction Pointer (IP/PC)
- General-purpose registers

When `switch_to()` finishes, the CPU is literally executing a different thread.

The previous task is frozen exactly where it stopped and resumes from that same point when scheduled again.

---

# Useful Kernel Tricks

## `likely()` and `unlikely()`

Example:

```c
if (likely(prev != next))
```

These macros wrap:

```c
__builtin_expect
```

Purpose:

Tell GCC/Clang which branch is expected most often.

Benefits:

- Better branch prediction
- Better instruction layout
- Fewer CPU pipeline flushes
- Faster execution

---

## Per-CPU Variables (`this_rq()`)

Each CPU owns its own runqueue.

Instead of searching a global array,

```c
this_rq()
```

uses a CPU-specific register (such as `GS` on x86) to instantly locate the current CPU's `rq`.

Benefits:

- No shared lock
- Less cache contention
- Better multicore performance

---

# Summary

- The scheduler shares CPU time among runnable tasks.
- Normal processes use the **Fair Scheduler (CFS/EEVDF)**.
- Every task tracks its CPU usage using `vruntime`.
- The task with the **lowest `vruntime`** runs next.
- Each CPU has its own `struct rq`.
- Runnable tasks are stored in a **Red-Black Tree**.
- `schedule()` eventually calls `context_switch()` to switch execution from one task to another.

---

# Mental Model

Imagine a hospital emergency room.

- Every patient = a task.
- The waiting room = `struct rq`.
- The Red-Black Tree = the triage system.
- `vruntime` = how much attention each patient has already received.

The doctor (CPU) always treats the patient who has received the **least attention**.

This keeps treatment fair while still respecting priority.

---

# Important Functions to Remember

- `schedule()`
- `__schedule()`
- `pick_next_task()`
- `context_switch()`
- `switch_mm()`
- `switch_to()`
- `deactivate_task()`
- `this_rq()`

---

# Important Structs to Remember

- `struct rq`
- `struct sched_entity`
- `struct rb_root`
- `struct rb_node`

---

# Code Reading Exercises

### Exercise 1

Open:

```text
kernel/sched/core.c
```

Find:

```c
__schedule()
```

Notice how it disables preemption using:

```c
preempt_disable()
```

Think about:

> What could happen if another context switch starts while one is already in progress?

---

### Exercise 2

Open:

```text
kernel/sched/fair.c
```

Search for:

```text
pick_next_entity
```

Observe how it retrieves the:

```text
rb_leftmost
```

node from the Red-Black Tree.

---

### Exercise 3

Open:

```text
include/linux/compiler.h
```

Find:

```c
likely()
unlikely()
```

See how they wrap:

```c
__builtin_expect
```

---

# Questions for Understanding

1. If an I/O-bound task wakes up after sleeping, its `vruntime` is much lower than a CPU-heavy task. Which task will the scheduler likely choose first, and why?

2. Why does Linux maintain a separate `struct rq` for each CPU core instead of one global runqueue?

3. After `switch_mm()` changes the page tables, what happens to the virtual memory addresses that belonged to the previous process?

---

# Suggested Next Files to Read

Now that we've seen how `switch_mm()` changes a process's address space, the next step is understanding how Linux manages memory.

Read these files:

```text
mm/mmap.c
```

- Virtual Memory Areas (VMAs)

```text
mm/page_alloc.c
```

- Physical page allocator
