
# Lesson 4: Linux Process Management and Task Representation

We are now leaving hardware-specific initialization and entering the core, architecture-independent part of the kernel.

The main challenge of an operating system is managing thousands of running programs.

The kernel must track:

- Process state.
- Memory.
- Open files.
- CPU execution context.
- Parent-child relationships.

Linux solves this using a unified concept called a **task**.

In Linux, there is no fundamental difference between a process and a thread at the kernel level.

Both are represented as a:

```c
task_struct
````

A thread is simply a task that shares resources, such as memory, with another task.

---

# 1. The Core Structure: `struct task_struct`

File:

```text
include/linux/sched.h
```

contains:

```c
struct task_struct
```

This is one of the most important structures in the kernel.

Every running thread has its own `task_struct`.

It stores everything the kernel needs to manage that task.

---

## Important Fields

Example:

```c
struct task_struct {

    volatile long state;
    void *stack;

    pid_t pid;
    pid_t tgid;

    struct task_struct *real_parent;

    struct list_head children;
    struct list_head sibling;

    struct mm_struct *mm;
    struct files_struct *files;

};
```

---

## Task State

```c
state
```

Stores the current execution state.

Examples:

* Running.
* Waiting.
* Stopped.
* Sleeping.

The scheduler uses this information to decide which task can run.

---

## Kernel Stack

```c
void *stack;
```

Each task has its own kernel stack.

The kernel uses this stack when:

* Handling system calls.
* Handling interrupts.
* Executing kernel code on behalf of the task.

---

## PID and TGID

Two important identifiers:

```c
pid
tgid
```

### `pid`

The unique kernel task ID.

Every thread has its own PID.

Example:

```text
Thread 1 -> pid 1001
Thread 2 -> pid 1002
Thread 3 -> pid 1003
```

---

### `tgid`

Thread Group ID.

All threads belonging to the same process share the same TGID.

Example:

```text
Process A

Thread 1:
pid 1001
tgid 1000

Thread 2:
pid 1002
tgid 1000

Thread 3:
pid 1003
tgid 1000
```

When user-space tools like `ps` show a process ID, they usually display the TGID.

---

# 2. Resource Sharing: Process vs Thread

The difference between a process and a thread is based on resource sharing.

Important field:

```c
struct mm_struct *mm;
```

This represents memory management information.

It contains:

* Page tables.
* Virtual memory information.

---

## Normal Process

Two separate processes have different memory spaces:

```text
Process A

task_struct
     |
     v
 mm_struct A


Process B

task_struct
     |
     v
 mm_struct B
```

---

## Thread

Threads share the same memory:

```text
Thread 1

task_struct
     |
     v
 mm_struct


Thread 2

task_struct
     |
     v
 mm_struct
```

Both tasks point to the same memory structure.

---

# 3. Linux Intrusive Linked Lists

Inside `task_struct`, we saw:

```c
struct list_head
```

Linux uses a special linked list design called an **intrusive linked list**.

---

## Traditional Linked List

Normally:

```text
Node
 |
 +--> Data
 |
 +--> Next Node
```

The list node is separate from the data.

---

## Linux Design

Linux embeds the list node directly inside the structure.

Example:

```c
struct task_struct {

    pid_t pid;

    struct list_head sibling;

};
```

Memory layout:

```text
+-------------------------+
| task_struct             |
|                         |
| pid                     |
|                         |
| list_head sibling       |-----> next task
|                         |
+-------------------------+
```

---

# 4. Why Use Intrusive Lists?

## 1. No Extra Allocation

The structure already contains the list node.

No need for:

```c
malloc()
```

to create list elements.

---

## 2. Better Performance

The kernel avoids:

* Extra memory allocations.
* Extra pointer lookups.

This matters because the kernel manages thousands of objects.

---

# 5. The `container_of()` Macro

A common question:

If we only have a pointer to:

```c
list_head
```

how do we find the original:

```c
task_struct
```

?

Linux uses:

```c
container_of()
```

Example:

```c
struct task_struct *task;

task = container_of(
        list_ptr,
        struct task_struct,
        sibling);
```

---

## How It Works

The macro calculates the location of the parent structure.

Concept:

```text
task_struct
+----------------+
| pid            |
+----------------+
| sibling        | <---- pointer we have
+----------------+
```

The kernel knows the offset of `sibling`.

It subtracts that offset and reaches the beginning of `task_struct`.

---

# 6. Process Creation

File:

```text
kernel/fork.c
```

contains the process creation logic.

Linux follows the Unix model:

1. Copy the current process.
2. Modify the child.
3. Optionally replace its program using `execve()`.

---

# 7. Process Creation Flow

The flow:

```text
User Program
      |
      v
clone() system call
      |
      v
kernel_clone()
      |
      v
copy_process()
      |
      v
New task_struct
      |
      v
wake_up_new_task()
      |
      v
Scheduler
```

---

# 8. `copy_process()`

The main work happens inside:

```c
copy_process()
```

It creates a new task.

Simplified:

```c
static struct task_struct *copy_process(...)
{

    struct task_struct *p;

    p = dup_task_struct(current);

    p->__state = TASK_NEW;

    copy_files();
    copy_fs();
    copy_mm();
    copy_thread();

    p->pid = new_pid;

    return p;
}
```

---

# 9. Steps Inside `copy_process()`

## Step 1: Duplicate Task Structure

```c
dup_task_struct()
```

Creates a copy of the parent's:

* `task_struct`
* Kernel stack

---

## Step 2: Set New State

```c
TASK_NEW
```

The task exists but is not ready to run yet.

---

## Step 3: Copy or Share Resources

Depending on clone flags:

### Files

```c
copy_files()
```

Handles:

* File descriptors.
* Open files.

---

### Filesystem Information

```c
copy_fs()
```

Handles:

* Current directory.
* Root directory.

---

### Memory

```c
copy_mm()
```

Handles memory sharing.

---

### CPU Context

```c
copy_thread()
```

Copies:

* Registers.
* Stack information.

---

# 10. Creating Threads with `clone()`

The important flag:

```c
CLONE_VM
```

controls memory sharing.

Without:

```text
Parent
 |
 +--> Memory A

Child
 |
 +--> Memory B
```

With:

```text
Parent
 |
 +--> Memory A
          ^
          |
Child ----+
```

The two tasks share the same memory.

This creates a thread.

---

# 11. Adding the New Task

After creation:

```c
kernel_clone()
```

calls:

```c
wake_up_new_task()
```

The new task is handed to the scheduler.

The scheduler decides when it gets CPU time.

---

# Summary

Linux represents both processes and threads using:

```c
struct task_struct
```

Each task contains:

* Identity (`pid`, `tgid`)
* Memory information (`mm_struct`)
* Open files (`files_struct`)
* Parent-child relationships
* Scheduling information

Linux uses intrusive linked lists (`list_head`) to efficiently manage tasks.

New tasks are created through:

```text
clone()
   |
   v
kernel_clone()
   |
   v
copy_process()
   |
   v
new task_struct
```

The child either gets copied resources or shares them depending on the clone flags.

---

# Mental Model

Think of `task_struct` as a passport for every running task.

It contains:

* Identity.
* Family relationships.
* Resources.
* Current state.

`fork.c` is the passport office.

It creates a copy of an existing passport, changes the identity information, and registers the new task with the scheduler.

---

# Important Functions

| Function             | Purpose                                              |
| -------------------- | ---------------------------------------------------- |
| `kernel_clone()`     | Entry point for creating tasks                       |
| `copy_process()`     | Creates and initializes a new task                   |
| `dup_task_struct()`  | Copies the parent's task structure                   |
| `wake_up_new_task()` | Sends the new task to the scheduler                  |
| `container_of()`     | Retrieves a parent structure from an embedded member |

---

# Important Structures

| Structure             | Purpose                       |
| --------------------- | ----------------------------- |
| `struct task_struct`  | Represents a process/thread   |
| `struct list_head`    | Intrusive linked list node    |
| `struct mm_struct`    | Memory management information |
| `struct files_struct` | Open file information         |

---

# Code Reading Exercises

## Exercise 1

Open:

```text
include/linux/sched.h
```

Find:

```c
struct task_struct
```

Look for:

```c
mm
```

and:

```c
fs
```

---

## Exercise 2

Open:

```text
include/linux/list.h
```

Find:

```c
list_add()
```

and:

```c
list_add_tail()
```

Observe the pointer manipulation.

---

## Exercise 3

Open:

```text
kernel/fork.c
```

Find:

```c
copy_process()
```

Look for:

```c
copy_mm()
copy_files()
copy_thread()
```

---

# Questions for Understanding

## 1. Same TGID, Different PID

If two tasks have:

```text
same TGID
different PID
```

they are:

* Threads belonging to the same process.

---

## 2. Why Use Intrusive Lists?

Because they:

* Avoid extra memory allocation.
* Reduce pointer overhead.
* Improve kernel performance.

---

## 3. Why Doesn't TASK_NEW Run Immediately?

Because the task must first be:

* Fully initialized.
* Added to scheduler data structures.

Only after:

```c
wake_up_new_task()
```

can it compete for CPU time.

---

# Suggested Next Files to Read

Now that tasks exist, they need CPU scheduling.

Read:

* `kernel/sched/core.c`

  * Scheduler framework.

* `kernel/sched/fair.c`

  * Completely Fair Scheduler implementation.
