# Lesson 11 — Wait Queues, Completions & Kernel Threads

## 1. Why Waiting Mechanisms Exist

In an operating system, tasks frequently need to wait for things to happen. A text editor reading a file must wait for the hard drive to spin up and fetch the data. A network server must wait for a packet to arrive over the Ethernet cable.

If a task simply continues executing before the hardware is ready, it will read garbage data. We need a way to safely pause a thread of execution, let the hardware do its job, and resume the thread the exact microsecond the data is available.

## 2. Busy Waiting vs Sleeping

When a thread needs to wait, it has two choices:

1. **Busy Waiting (Spinning):** The thread loops endlessly in a `while(data_not_ready);` loop. We saw this with Spinlocks in Lesson 9. This is only acceptable if the wait time is a few microseconds. If you busy-wait for a hard drive (which takes milliseconds), you waste millions of CPU cycles that could have been used by other programs.
2. **Sleeping (Yielding):** The thread tells the kernel, "I cannot proceed. Take me off the CPU, run another program, and wake me up when my data is here." This is highly efficient and is the cornerstone of operating system multitasking.

## 3. Wait Queues

How does the kernel remember *which* sleeping task is waiting for *which* hardware event? It uses a data structure called a **Wait Queue**.

A wait queue is essentially a linked list of sleeping tasks attached to a specific condition or resource.

For example, every network socket has its own wait queue. If five different programs call `recv()` on that socket when there is no data, all five tasks are put to sleep and added to that socket's wait queue.

## 4. Wait Queue Lifecycle

Waiting safely is notoriously difficult because of race conditions. What if the hardware finishes and sends the wake-up signal *exactly* as your task is going to sleep, but before it is fully asleep? You might sleep forever!

The wait queue lifecycle solves this:

1. **Declare:** The driver initializes a `wait_queue_head_t`.
2. **Evaluate:** The task checks if the condition is already met (e.g., `if (data_ready)`).
3. **Prepare to Sleep:** The task adds itself to the wait queue and changes its state to `TASK_INTERRUPTIBLE` (sleeping).
4. **Yield:** The task calls `schedule()` to hand the CPU to someone else.
5. **Wake Up:** A hardware interrupt (Top/Bottom Half) signals the wait queue.
6. **Re-evaluate (The Spurious Wakeup):** When the task wakes up, it *must* check the condition again. Why? Because multiple tasks might have woken up, and another one stole the data first! Or, a UNIX signal (like `Ctrl+C`) might have interrupted the sleep.

The kernel handles all this boilerplate for you via the `wait_event()` macros.

## 5. Completions

Sometimes, wait queues are overkill. You don't have multiple tasks waiting for varying conditions. You just have Thread A doing some setup, and Thread B waiting for that exact setup to finish.

A **Completion** (`struct completion`) is a specialized, extremely lightweight synchronization primitive built for this exact "one-off" scenario.

It solves a specific race condition: If Thread A finishes the work *before* Thread B even starts waiting, a raw wait queue might cause Thread B to miss the wake-up and sleep forever. A completion remembers that the event *already happened* and lets Thread B pass through instantly.

## 6. Kernel Threads

User-space programs have threads (created via `pthread_create`). But the kernel itself often needs to run background tasks that never interact with user-space.

These are called **Kernel Threads**.

- They run exclusively in Ring 0 (Kernel Space).
- They have a `task_struct`, just like a normal process.
- They **do not** have a user-space memory map (their `task_struct->mm` pointer is `NULL`).
- Example: `kswapd` runs in the background, constantly scanning for unused physical memory to swap out to disk.

Kernel threads frequently use Wait Queues and Completions to sleep until there is background work to do.

## 7. Process Context

As a reminder from Lesson 8: **You can only sleep in Process Context.**

- A user-space system call (like `sys_read`) runs in Process Context. It has a `task_struct`. It can sleep.
- A Kernel Thread has a `task_struct`. It can sleep.
- A Top Half hardware interrupt (ISR) has *no* persistent context. **It cannot sleep.** It can only *wake up* other sleeping tasks.

## 8. Sleeping and Wake-up Mechanism

When you sleep, your `task_struct->state` changes.

- `TASK_RUNNING`: The task is actively on the CPU, or waiting in the Runqueue (from Lesson 5) ready to run.
- `TASK_INTERRUPTIBLE`: The task is sleeping, but can be woken up early by a user signal (like `SIGINT` from `kill`).
- `TASK_UNINTERRUPTIBLE`: The task is in a deep sleep (usually waiting for hardware I/O). It ignores all signals. (If a task gets stuck here, you see the dreaded `D` state in `htop`, and you cannot kill it without rebooting).

When `wake_up()` is called, the kernel finds the task on the wait queue, sets its state back to `TASK_RUNNING`, and places it back into the Scheduler's Red-Black tree.

## 9. Scheduler Interaction

The `wait_event()` macro is a fascinating piece of C code. Under the hood, it expands into a loop that interacts directly with the scheduler:

```c
// Conceptual expansion of wait_event(wq, condition)
for (;;) {
    prepare_to_wait(&wq, &wait_entry, TASK_UNINTERRUPTIBLE);
    if (condition)
        break; // Data is ready, don't sleep!
    schedule(); // Hand over the CPU
}
finish_wait(&wq, &wait_entry);
```

Notice the safety: It prepares to sleep, *then* checks the condition, *then* actually yields the CPU. This prevents the lost wake-up race condition.

## 10. Important APIs

### Wait Queues

- `DECLARE_WAIT_QUEUE_HEAD(name)`: Statically creates a wait queue.
- `init_waitqueue_head(wait_queue_head_t *wq)`: Dynamically initializes one.
- `wait_event(wq, condition)`: Puts the current task to sleep (Uninterruptible) until `condition` evaluates to true.
- `wait_event_interruptible(wq, condition)`: Same, but can be interrupted by signals (returns `-ERESTARTSYS` if interrupted).
- `wake_up(wait_queue_head_t *wq)`: Wakes up sleeping tasks on the queue.

### Completions

- `init_completion(struct completion *c)`: Initializes the struct.
- `wait_for_completion(struct completion *c)`: Sleeps until someone signals the completion.
- `complete(struct completion *c)`: Signals the completion, waking up a waiter.

### Kernel Threads

- `kthread_create(threadfn, data, namefmt)`: Creates a stopped kernel thread.
- `kthread_run(threadfn, data, namefmt)`: Creates a kernel thread and immediately wakes it up to start running.

## 11. Important Structs

- `wait_queue_head_t`: The anchor (head) of the linked list of sleeping tasks.
- `wait_queue_entry_t`: The node embedded in the waiting task, representing its place in the line.
- `struct completion`: A tiny struct containing a wait queue and a boolean "done" counter.
- `struct task_struct`: The core process descriptor (from Lesson 4) whose `state` field is manipulated during sleeps.

## 12. Execution Flow

Here is how a device driver uses a wait queue to handle a read request:

```text
[ User Space App ]               [ Hardware Device ]
       |
  read() syscall
       |
[ Kernel / Driver ]
       |
  Driver checks if data is ready. (It is not).
  wait_event_interruptible(wq, data_ready)
       |
       +--> Task state set to TASK_INTERRUPTIBLE
       +--> Task added to 'wq' list
       +--> schedule()  (CPU switches to another program!)
                                          |
        ... milliseconds pass ...         |
                                          |
                                 Hardware finishes!
                                 Fires Hardware Interrupt (IRQ)
                                          |
[ Top Half ISR ] <------------------------+
       |
  ISR reads data from hardware into RAM.
  Sets data_ready = true;
  wake_up(&wq);
       |
       +--> Removes task from 'wq' list
       +--> Sets task state to TASK_RUNNING
       +--> Puts task back in Scheduler Runqueue

[ Kernel / Driver ] (Resumes execution)
       |
  wait_event loop wakes up, checks data_ready (It is true!)
  Copies data to User Space.
  Returns from syscall.
```

## 13. Source Files to Read

- `include/linux/wait.h`: The macros and APIs for wait queues.
- `include/linux/completion.h`: The API for completions.
- `kernel/sched/wait.c`: The internal implementation of adding/removing tasks from the queue.
- `kernel/kthread.c`: The kernel thread spawning machinery.

## 14. Mental Model

- **Wait Queue:** Think of a restaurant pager system. You arrive (call `wait_event`), but your table isn't ready. The host hands you a pager and you sit on a bench (go to sleep). When a table opens up (the condition), the host buzzes your pager (`wake_up`), and you get up. If multiple people have pagers, the host might buzz everyone, and whoever gets to the table first gets it (handling spurious wakeups).

- **Completion:** A package delivery with a signature requirement. You (Thread B) are waiting for Thread A to deliver the package. If Thread A arrives before you get home, they leave the package in a lockbox (`complete()` increments a counter). When you get home (`wait_for_completion()`), you instantly see the package and don't have to wait at all.

- **Kernel Thread:** The restaurant's janitor. They don't serve customers (user-space), they just work in the background keeping the system clean, occasionally sleeping in the breakroom when there is no mess.

## 15. Summary

Wait queues are the fundamental mechanism the Linux kernel uses to safely pause execution while waiting for asynchronous events (like hardware I/O). By manipulating the `task_struct` state and calling the scheduler, the kernel ensures zero CPU cycles are wasted by waiting tasks. Completions offer a simpler, race-free alternative for one-off synchronization, and Kernel Threads utilize these primitives to run background OS maintenance tasks.

## 16. Code Reading Exercises

1. Open `include/linux/wait.h` and search for the `#define wait_event(`. Trace the macro expansion down to `___wait_event`. Notice the infinite `for (;;) { ... }` loop that encompasses the `schedule()` call.
2. Open `include/linux/completion.h` and look at `struct completion`. Observe how tiny it is: just an `unsigned int done` counter and a `wait_queue_head_t`.
3. Open `kernel/kthread.c` and find `kthread_create_on_node()`. Notice how creating a kernel thread doesn't magically spawn execution out of nowhere—it actually sends a message to an existing, special kernel thread called `kthreadd` (PID 2) to fork the new thread!

## 17. Questions for Understanding

1. If a hardware interrupt handler (ISR) needs to wait for a specific piece of data to arrive from the network, can it call `wait_event()`? Why or why not?

2. Imagine a thread calls `wait_event(wq, buffer_has_space)`. It wakes up. Why is it absolutely critical that it checks `buffer_has_space` again immediately after waking up, instead of blindly writing to the buffer?

3. What happens to a user-space process if a buggy kernel driver puts it to sleep using `TASK_UNINTERRUPTIBLE`, but the hardware breaks and never triggers the `wake_up()` call?

---

## 💡 Answer Key (For Your Reference)

**Q1:** No. An ISR runs in Interrupt Context, which has no `task_struct` and cannot sleep. It can only call `wake_up()` to wake other processes.

**Q2:** This handles **spurious wakeups**. Multiple tasks might wake simultaneously, and another task could consume the buffer space before this task runs. Re-checking ensures we only proceed when the condition is truly met.

**Q3:** The process enters the dreaded **"D" (uninterruptible sleep) state** and becomes unkillable (even `kill -9` won't work). The system remains functional, but that process hangs forever until reboot. This is why kernel developers must be extremely careful with `TASK_UNINTERRUPTIBLE`.
