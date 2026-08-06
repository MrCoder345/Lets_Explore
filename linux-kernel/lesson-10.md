
# Lesson 10 — RCU (Read-Copy-Update)

> **Topics:** RCU, Grace Period, Lockless Reads, Memory Reclamation, Tree RCU

---

# 1. Why RCU Exists

In **Lesson 9**, we learned about **Spinlocks** and **Mutexes**.

These synchronization primitives provide **mutual exclusion**, meaning only one execution context can modify shared data at a time.

But consider a data structure like a:

- Network routing table
- File descriptor table
- VFS directory cache

These structures are:

- Read **thousands (or millions)** of times.
- Updated **very rarely**.

If every read operation acquires a spinlock or mutex, then on a machine with many CPU cores, all CPUs constantly compete for the same lock.

This becomes a major performance bottleneck.

---

## Why RCU Was Created

RCU (**Read-Copy-Update**) was invented by **Paul E. McKenney** to solve this exact problem.

Its goal is:

> Allow **thousands of readers** to access shared data simultaneously with **almost zero overhead**, while still allowing the data to be safely updated.

---

# 2. Problems with Traditional Locks

A common question is:

> "Why not use a Reader/Writer Lock (`rwlock_t`)?"

Although `rwlock_t` allows multiple readers, every reader still needs to update a shared counter.

Example:

```
CPU 1 -> reader count++

CPU 2 -> reader count++

CPU 3 -> reader count++
```

Even though readers are only reading data, they still **write** to the shared counter.

---

## Cache-Line Bouncing

When many CPUs repeatedly write to the same memory location, the CPU cache-coherency protocol continuously transfers ownership of that cache line between cores.

This is called **Cache-Line Bouncing**.

Effects:

- High memory traffic
- Reduced scalability
- Poor performance on multi-core systems

---

## RCU's Main Idea

RCU avoids this completely.

**Readers do not modify shared memory.**

They:

- Do not take a traditional lock.
- Do not update a reader counter.
- Simply read the data.

This makes RCU extremely fast.

---

# 3. Read-side Critical Sections

A reader begins an RCU critical section using:

```c
rcu_read_lock();
```

and finishes with:

```c
rcu_read_unlock();
```

During this time:

- The reader safely accesses RCU-protected data.
- No traditional lock is acquired.
- No shared counter is updated.

On many kernel configurations, the overhead is almost zero.

---

## Basic Example

```c
rcu_read_lock();

struct config *cfg = rcu_dereference(global_cfg);

printk("%d\n", cfg->timeout);

rcu_read_unlock();
```

Readers never modify the object.

They only access it safely.

---

# 4. Update-side Operations

If readers never lock the data, how can a writer safely update it?

The answer is the **Read-Copy-Update** algorithm.

---

## Rule

A writer must **never modify shared data in place** if readers may still be using it.

Instead, it performs three steps.

---

## Step 1 — Read

Read the current pointer.

```text
global_ptr ---> Old Data
```

---

## Step 2 — Copy

Allocate a completely new object.

Copy the old data.

```text
Old Data

↓

New Data (copy)
```

---

## Step 3 — Update

Modify the new copy.

Example:

```text
Old:

timeout = 50
```

New copy:

```text
timeout = 100
```

---

## Step 4 — Publish

Atomically replace the global pointer.

```text
Before

global_ptr
     |
     ▼
Old Data
```

After:

```text
global_ptr
     |
     ▼
New Data

Old Data
```

Old readers continue using **Old Data**.

New readers automatically see **New Data**.

---

# 5. Grace Period

After the pointer swap:

New readers:

```
New Data
```

Old readers:

```
Old Data
```

Therefore, the writer **cannot immediately free** the old object.

Otherwise:

```c
kfree(old_data);
```

would leave readers with a dangling pointer.

Result:

- Use-after-free
- Kernel crash
- Memory corruption

---

## Grace Period

The kernel waits until:

> Every reader that started before the pointer update has completely finished.

Only then is it safe to free the old object.

This waiting time is called the **Grace Period**.

---

# 6. RCU Lifecycle

Every RCU update follows the same sequence.

---

## Phase 1 — Publish

Publish the new pointer.

```c
rcu_assign_pointer(global_ptr, new_data);
```

---

## Phase 2 — Wait

Wait for all old readers.

```c
synchronize_rcu();
```

or

```c
call_rcu();
```

---

## Phase 3 — Reclaim

Old memory is finally released.

```c
kfree(old_data);
```

---

# 7. Synchronization vs Memory Reclamation

RCU provides two common ways to wait for the Grace Period.

---

## `synchronize_rcu()`

Synchronous.

```c
rcu_assign_pointer(...);

synchronize_rcu();

kfree(old_data);
```

The writer blocks until the Grace Period finishes.

Simple but slower.

---

## `call_rcu()`

Asynchronous.

```c
call_rcu(&old->rcu, callback);
```

The writer immediately continues executing.

Later, after the Grace Period ends, the kernel automatically invokes the callback.

Example:

```c
void free_callback(struct rcu_head *head)
{
    struct my_data *obj;

    obj = container_of(head, struct my_data, rcu);

    kfree(obj);
}
```

This is preferred when the writer should not block.

---

# 8. Types of RCU

Linux provides multiple RCU implementations.

---

## Classic RCU

Original implementation.

Used on non-preemptible kernels.

---

## Preemptible RCU

Allows readers to be preempted by the scheduler.

Important for **Real-Time Linux (PREEMPT_RT)**.

---

## SRCU (Sleepable RCU)

Normally:

Readers inside:

```c
rcu_read_lock();
```

must **not sleep**.

SRCU removes this restriction.

Readers are allowed to sleep.

It is slower than standard RCU.

---

## Tree RCU

The highly scalable implementation used by modern Linux systems.

Supports thousands of CPU cores efficiently.

---

# 9. Tree RCU Overview

How does Linux know every reader has finished?

Using one global counter would create another bottleneck.

Instead, Tree RCU organizes CPUs into a hierarchy.

Example:

```text
                Root
               /    \
          Node A   Node B
          /   \      /  \
       CPU0 CPU1 CPU2 CPU3
```

Each CPU reports completion to its local node.

Local nodes report upward.

Eventually the root determines that every CPU has passed the Grace Period.

This avoids excessive contention on a single shared variable.

---

# 10. Important RCU APIs

## Reader APIs

```c
rcu_read_lock();
```

Starts a read-side critical section.

---

```c
rcu_read_unlock();
```

Ends the read-side critical section.

---

```c
rcu_dereference(ptr);
```

Safely reads an RCU-protected pointer.

Includes the required compiler and memory ordering semantics.

---

## Writer APIs

```c
rcu_assign_pointer(ptr, new_ptr);
```

Publishes the new pointer safely.

Ensures the object is fully initialized before readers can see it.

---

```c
synchronize_rcu();
```

Waits for the Grace Period.

---

```c
call_rcu();
```

Schedules asynchronous memory reclamation.

---

# 11. Important Structures

## `struct rcu_head`

Used when memory is reclaimed asynchronously.

Example:

```c
struct my_data
{
    int value;

    struct rcu_head rcu;
};
```

When using:

```c
call_rcu();
```

your structure **must** embed a `struct rcu_head`.

---

# 12. Execution Flow

```text
CPU 1 (Reader)

rcu_read_lock()

        │
        ▼

rcu_dereference(global_ptr)

        │
        ▼

Read Old Data

        │
        ▼

rcu_read_unlock()
```

At the same time:

```text
CPU 2 (Writer)

Allocate New Data

        │
        ▼

Copy Old Data

        │
        ▼

Modify Copy

        │
        ▼

rcu_assign_pointer()

        │
        ▼

call_rcu()

        │
        ▼

Continue Running
```

Later:

```text
Grace Period Ends

↓

Kernel executes callback

↓

Old Data Freed
```

---

# 13. Real Kernel Example

Writers still need traditional synchronization.

Example:

```c
struct config *new_cfg;

new_cfg = kmalloc(sizeof(*new_cfg), GFP_KERNEL);

new_cfg->timeout = 100;

spin_lock(&config_lock);

struct config *old_cfg = global_cfg;

rcu_assign_pointer(global_cfg, new_cfg);

spin_unlock(&config_lock);

synchronize_rcu();

kfree(old_cfg);
```

### Why is the spinlock still needed?

RCU protects **readers**.

It does **not** prevent two writers from updating the pointer simultaneously.

The spinlock serializes writers.

---

# 14. Kernel Source Files to Read

```text
include/linux/rcupdate.h
```

Main RCU API.

---

```text
kernel/rcu/tree.c
```

Tree RCU implementation.

---

```text
Documentation/RCU/
```

Official RCU documentation.

A great starting file is:

```text
Documentation/RCU/whatisRCU.rst
```

---

# 15. Mental Model

Imagine a public notice board.

## Reader

Walks up.

Reads the notice.

Leaves.

Never asks permission.

---

## Writer

Needs to change the notice.

Instead of erasing the current one:

- Makes a new copy.
- Updates it.
- Replaces the notice.

Readers already looking at the old notice continue reading it.

New readers automatically see the updated notice.

---

## Grace Period

The writer waits until everyone reading the old notice walks away.

Only then is the old paper removed.

---

# 16. Summary

RCU is designed for **read-mostly** data structures.

Instead of locking readers:

- Readers simply read.
- Writers create a copy.
- Writers update the copy.
- Writers publish the new pointer.
- Old memory is reclaimed only after the Grace Period.

This provides:

- Extremely fast reads
- Excellent scalability
- Minimal cache contention

---

# 17. Code Reading Exercises

### Exercise 1

Open:

```text
include/linux/rcupdate.h
```

Find:

```c
rcu_read_lock()
```

Notice that on some kernel configurations it expands to almost nothing except a compiler barrier.

---

### Exercise 2

Find:

```c
rcu_assign_pointer()
```

Observe its use of:

```c
smp_store_release()
```

This guarantees that the new object's contents are fully written before the pointer becomes visible to readers.

---

### Exercise 3

Open:

```text
kernel/rcu/tree.c
```

You don't need to understand every line.

Instead, notice how much complexity is required to efficiently manage Grace Periods across thousands of CPUs.

---

# 18. Questions for Understanding

### 1.

If readers are lock-free, why must writers still protect updates with a **Spinlock** or **Mutex**?

> **Hint:** What happens if two writers try to replace the same pointer simultaneously?

---

### 2.

Suppose a **Top Half Interrupt Handler** needs to free an old RCU object.

Should it use:

```c
synchronize_rcu()
```

or

```c
call_rcu()
```

Why?

---

### 3.

Why is it forbidden to call:

```c
sleep();
```

or

```c
mutex_lock();
```

inside:

```c
rcu_read_lock();
```

(Assume **Classic RCU**, not **SRCU**.)

---

# Quick Revision

| Feature | RCU |
|----------|-----|
| Reader Locking | Lock-free |
| Reader Overhead | Almost zero |
| Writers Modify In Place? | ❌ No |
| Update Method | Copy → Modify → Publish |
| Old Memory Freed Immediately? | ❌ No |
| Wait Before Freeing | Grace Period |
| Async Reclamation | `call_rcu()` |
| Blocking Reclamation | `synchronize_rcu()` |

> **Golden Rule:** **Readers never modify shared data. Writers never modify shared data in place.**
