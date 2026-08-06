# Lesson 9 — Synchronization Fundamentals
> **Topics:** Race Conditions, Atomic Operations, Spinlocks, Mutexes

---

# 1. Introduction

In previous lessons, we learned that the Linux kernel is a **highly concurrent environment**.

Multiple things can happen at the same time:

- Multiple processes run on different CPU cores (**SMP - Symmetric Multiprocessing**).
- The scheduler can preempt (pause) a running task at any time.
- Hardware interrupts can interrupt execution unexpectedly.

When multiple CPUs or tasks try to **read or modify the same memory simultaneously**, incorrect behavior can occur.

To solve this problem, the Linux kernel uses **Synchronization Primitives**.

---

# 2. Why Synchronization Exists

Synchronization exists to protect **shared data** from being modified by multiple execution contexts at the same time.

### Real-Life Example

Imagine a bank account containing **$100**.

Two people try to withdraw **$100 simultaneously**.

Without synchronization:

1. Person A checks the balance → `$100`
2. Person B checks the balance → `$100`
3. Both withdraw `$100`
4. Final balance becomes incorrect.

Both operations looked valid individually, but together they corrupted the data.

---

## Kernel Example

In the kernel, the shared resource isn't money.

It can be:

- Reference counters
- Linked lists
- Scheduler runqueue
- Device registers
- Shared structures

If two CPUs modify something like `struct list_head` simultaneously, the linked list can become corrupted, often causing a **kernel panic**.

---

## Goal of Synchronization

Synchronization provides **Mutual Exclusion (Mutex)**.

**Meaning:**

> Only one execution context can modify a shared resource at a time.

---

# 3. Race Conditions

## Definition

A **Race Condition** happens when the program's output depends on the unpredictable timing of two or more threads or CPUs.

---

## Example

This C statement looks simple:

```c
count++;
```

But internally it is **NOT one instruction**.

It performs three operations:

1. Read
2. Modify
3. Write

---

### Read → Modify → Write

```text
count = 5

CPU 1                    CPU 2
-----                    -----

Read 5
                          Read 5

Increment -> 6
                          Increment -> 6

Write 6
                          Write 6
```

Final value:

```text
count = 6
```

Expected value:

```text
count = 7
```

One increment was lost.

This is called a **Race Condition**.

---

# 4. Critical Section

A **Critical Section** is a part of the code that accesses shared resources.

Only **one execution context** should execute that section at a time.

Example:

```c
spin_lock(&lock);

/* Critical Section */

shared_list->next = node;

spin_unlock(&lock);
```

Think of it like a room with only one key.

Before entering:

- Lock the door

After finishing:

- Unlock the door

---

# 5. Atomic Operations

## Why Atomic Operations Exist

Sometimes the shared resource is only a single integer.

Example:

- Reference count
- Usage counter
- Packet counter

Using a heavy lock just to increment one integer is inefficient.

---

## `atomic_t`

Linux provides a special integer type:

```c
atomic_t
```

Example:

```c
atomic_t refcount = ATOMIC_INIT(1);

atomic_inc(&refcount);
```

No matter how many CPUs execute this simultaneously, the result is always correct.

---

## How It Works

On x86 CPUs, the compiler generates instructions using the **`LOCK` prefix**.

Example:

```asm
lock addl
```

This makes the Read → Modify → Write operation **indivisible (atomic)**.

Other CPUs must wait until it completes.

---

## Common Atomic APIs

```c
atomic_inc()

atomic_dec()

atomic_read()

atomic_set()
```

---

# 6. Memory Ordering (Basic Introduction)

Modern CPUs and compilers aggressively optimize execution.

One optimization is **instruction reordering**.

Example:

The CPU may internally execute instructions in a different order if it believes the result is unchanged for the current thread.

This optimization can break synchronization between multiple CPUs.

---

## Memory Barrier

Linux provides **Memory Barriers**.

Example:

```c
smp_mb();
```

Meaning:

> "Do not move memory operations across this point."

All major kernel locks already contain the required memory barriers.

So normally you don't need to call them manually while using spinlocks or mutexes.

---

# 7. Spinlocks

## Why Spinlocks Exist

Interrupt handlers **cannot sleep**.

So they cannot use sleeping locks like mutexes.

Instead, they use **Spinlocks**.

---

## How Spinlocks Work

If the lock is already taken:

The CPU simply keeps checking until it becomes free.

```c
while (lock_is_taken(&my_lock))
{
    cpu_relax();
}
```

This is called **Busy Waiting**.

---

## Characteristics

✅ Very fast

✅ No sleeping

❌ Wastes CPU cycles

❌ Should only protect **very short** critical sections

---

## Basic Example

```c
spinlock_t lock;

spin_lock(&lock);

/* Critical Section */

spin_unlock(&lock);
```

---

## Important Rule

Never do these while holding a spinlock:

- Sleep
- `schedule()`
- Long loops
- Disk I/O
- Network waits
- `mutex_lock()`

---

## Local Interrupt Trap

Imagine:

Process Context:

```c
spin_lock(&lock);
```

Before releasing it...

An interrupt occurs on the **same CPU**.

The interrupt handler also executes:

```c
spin_lock(&lock);
```

Now:

- Process waits for interrupt.
- Interrupt waits for Process.

Result:

**Self Deadlock**

---

## Solution

Use:

```c
unsigned long flags;

spin_lock_irqsave(&lock, flags);

/* Critical Section */

spin_unlock_irqrestore(&lock, flags);
```

This:

- Saves interrupt state
- Disables local interrupts
- Acquires the lock
- Restores interrupt state after unlocking

> **Note:** `spin_lock_irqsave()` disables interrupts only on the **current CPU**, not on all CPUs.

---

# 8. Raw Spinlocks

Normally:

```c
spinlock_t
```

disables preemption.

However, in **PREEMPT_RT (Real-Time Linux)**:

Normal spinlocks may internally become **sleeping locks**.

If the kernel developer **must guarantee** busy waiting even on an RT kernel:

Use:

```c
raw_spinlock_t
```

Commonly used inside:

- Scheduler
- Memory management
- Architecture-specific code

---

# 9. Reader/Writer Spinlocks

Sometimes data is:

- Read frequently
- Written rarely

Using a normal spinlock blocks every reader.

Instead use:

```c
rwlock_t
```

### Rules

Many readers:

```
CPU1 -> Read ✅

CPU2 -> Read ✅

CPU3 -> Read ✅
```

One writer:

```
CPU4 -> Write

Must wait until all readers finish.
```

Only one writer is allowed.

---

# 10. Mutexes

## Why Mutexes Exist

Spinlocks waste CPU time if the critical section is long.

Example:

Waiting for:

- Disk I/O
- USB device
- Network
- User input

Instead of spinning, the task should sleep.

---

## How Mutex Works

Suppose:

Task A owns the mutex.

Task B tries:

```c
mutex_lock(&lock);
```

Kernel behavior:

1. Put Task B to sleep.
2. Add Task B to a wait queue.
3. Call:

```c
schedule();
```

4. CPU starts running another task.

When Task A unlocks:

```c
mutex_unlock(&lock);
```

Task B wakes up and continues.

---

## Example

```c
struct mutex lock;

mutex_lock(&lock);

/* Critical Section */

mutex_unlock(&lock);
```

---

## Rule

Mutexes **may sleep**.

Therefore they can **ONLY** be used in:

- ✅ Process Context

Never in:

- ❌ Interrupt Context

---

# 11. Semaphores (Brief Comparison)

A semaphore is older than a mutex.

---

## Mutex

- Binary (0 or 1)
- Has ownership
- Same thread must unlock it

---

## Semaphore

Acts like a counter.

Example:

```text
Semaphore = 5
```

Up to five threads can enter simultaneously.

Also:

Thread A can lock.

Thread B can unlock.

---

## Modern Linux Recommendation

Prefer:

```c
struct mutex
```

Use semaphores only when you specifically need:

- Counting
- Cross-thread unlock behavior

---

# 12. Spinlock vs Mutex

| Feature | `spinlock_t` | `struct mutex` |
|----------|--------------|----------------|
| Waiting | Busy Wait | Sleep |
| CPU Usage | High while waiting | Low |
| Can Sleep? | ❌ No | ✅ Yes |
| Interrupt Context | ✅ Yes | ❌ No |
| Process Context | ✅ Yes | ✅ Yes |
| Best For | Very short critical sections | Long operations |

---

# 13. Common Kernel Bugs

## 1. Deadlock (ABBA)

Example:

Thread 1:

```text
Lock A
Lock B
```

Thread 2:

```text
Lock B
Lock A
```

Both wait forever.

---

## 2. Self Deadlock

Same thread locks the same lock twice.

Example:

```c
spin_lock(&lock);

function();

function()
{
    spin_lock(&lock);
}
```

Deadlock.

---

## 3. Livelock

Threads keep yielding to each other.

Nobody actually makes progress.

Example:

Two people trying to pass in a hallway and both keep stepping to the same side.

---

## 4. Priority Inversion

Low-priority task owns mutex.

High-priority task waits.

Medium-priority task keeps running.

Result:

High-priority task is indirectly blocked by the medium-priority task.

---

# 14. Important Kernel APIs

## Atomic

```c
atomic_inc()

atomic_dec()
```

---

## Spinlock

```c
spin_lock_irqsave()

spin_unlock_irqrestore()
```

---

## Mutex

```c
mutex_lock()

mutex_unlock()
```

---

# 15. Important Structures

```c
atomic_t
```

Atomic integer.

---

```c
spinlock_t
```

Standard busy-wait lock.

---

```c
raw_spinlock_t
```

Guaranteed busy-wait lock.

---

```c
struct mutex
```

Sleeping lock.

---

# 16. Mutex Execution Flow

```text
CPU1 (Task A)

mutex_lock()
      │
      ▼
Critical Section
      │
      ▼
mutex_unlock()
      │
      ▼
Wake sleeping task
```

```text
CPU2 (Task B)

mutex_lock()
      │
      ▼
Lock Busy
      │
      ▼
Sleep
      │
      ▼
schedule()
      │
      ▼
Wake Up
      │
      ▼
Acquire Lock
```

---

# 17. Kernel Source Files to Read

```text
include/linux/spinlock.h
```

Spinlock API.

---

```text
include/linux/mutex.h
```

Mutex API.

---

```text
kernel/locking/mutex.c
```

Mutex implementation.

---

```text
Documentation/locking/mutex-design.rst
```

Official mutex design documentation.

---

# 18. Mental Model

## Spinlock

Imagine a bathroom with no waiting area.

Someone is inside.

You keep trying the door repeatedly until it opens.

Fast if they leave quickly.

Terrible if they stay for a long time.

---

## Mutex

Imagine a restaurant.

The host gives you a pager.

Instead of standing near the door, you sit comfortably.

When your table is ready, the pager buzzes.

That's exactly how a mutex works.

---

# 19. Summary

Linux prevents memory corruption using different synchronization primitives:

| Situation | Use |
|-----------|-----|
| Single integer | `atomic_t` |
| Very short critical section | `spinlock_t` |
| Interrupt Context | `spinlock_t` |
| Long critical section | `struct mutex` |
| Sleeping allowed | `struct mutex` |

The biggest rule to remember is:

> **Never sleep while holding a spinlock.**

---

# 20. Code Reading Exercises

### Exercise 1

Open:

```text
include/linux/mutex.h
```

Find:

```c
struct mutex
```

Look for:

```c
struct list_head wait_list;
```

Notice how it uses the intrusive linked list from Lesson 4 to store sleeping tasks.

---

### Exercise 2

Open:

```text
kernel/locking/mutex.c
```

Search for:

```c
__mutex_lock
```

Observe how the function switches to the **slow path** when the mutex is already locked.

---

### Exercise 3

Open:

```text
include/linux/spinlock.h
```

Search for:

```c
spin_lock_irqsave()
```

Notice how it saves the interrupt state using the `flags` variable before disabling local interrupts.

---

# 21. Questions for Understanding

1. If you call:

```c
kmalloc(sizeof(node), GFP_KERNEL)
```

while holding a `spinlock_t`, what kernel rule have you violated?

---

2. Why does:

```c
spin_lock_irqsave()
```

disable interrupts only on the **local CPU** instead of all CPUs?

---

3. In an **ABBA deadlock**, what locking strategy (lock ordering) is commonly used in the Linux kernel to prevent the deadlock?

---

# Quick Revision

| Primitive | Sleeps? | Interrupt Context | Best Use |
|-----------|----------|------------------|----------|
| `atomic_t` | ❌ | ✅ | Single integer operations |
| `spinlock_t` | ❌ | ✅ | Very short critical sections |
| `raw_spinlock_t` | ❌ | ✅ | Strict busy-waiting (PREEMPT_RT safe) |
| `rwlock_t` | ❌ | ✅ | Read-heavy shared data |
| `struct mutex` | ✅ | ❌ | Long critical sections in process context |
| `struct semaphore` | ✅ | ❌ | Counting resources (rarely used) |

> **Golden Rule:** If a lock **can sleep**, it **cannot** be used in **Interrupt Context**. If a lock **cannot sleep**, keep the critical section **as short as possible**.
