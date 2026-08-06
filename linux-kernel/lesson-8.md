# Lesson 8: Hardware Interrupts (IRQ) and Top/Bottom Halves

System calls are **predictable**. A program intentionally asks the kernel to perform a task (like reading a file), so the CPU switches from user space to kernel space.

Hardware is different. It is **unpredictable**.

A network packet may arrive, a disk may finish reading data, or a USB mouse may move at any moment. The CPU could be:

- Running a user program
- Executing kernel code
- Completely idle

It does not matter.

The hardware immediately signals the CPU because it needs attention **right now**.

This entire mechanism is handled mainly inside:

```text
kernel/irq/               -> Generic (architecture-independent) IRQ handling
arch/x86/kernel/irq.c     -> x86-specific interrupt routing
```

---

# Why Do We Need Interrupts?

Imagine if Linux had to constantly ask every hardware device:

```text
Mouse, did you move?
-> No.

Keyboard, was a key pressed?
-> No.

Network card, any packets?
-> No.
```

The CPU would waste most of its time checking devices that have nothing to report.

Instead, Linux uses **Interrupts**.

When hardware needs attention, it sends an electrical signal to the CPU's interrupt controller (APIC on x86). The CPU immediately pauses its current work and handles the event.

---

# Interrupt Context

When an interrupt occurs, the CPU automatically:

1. Stops the current execution.
2. Saves the current instruction pointer.
3. Disables local interrupts.
4. Jumps to the Interrupt Service Routine (ISR).

At this moment, the kernel is running in a special execution mode called:

> **Interrupt Context**

This is very different from normal process execution.

There is **no current process** that the interrupt belongs to.

Because of that, many normal kernel operations become illegal.

---

## Golden Rule

> **Never sleep or block inside Interrupt Context.**

This means you **cannot**:

- Take a Mutex lock (may sleep)
- Wait for disk I/O
- Wait for a network response
- Call `kmalloc()` with normal allocation flags

Violating this rule can crash the kernel.

---

## Why Must It Be Fast?

While an interrupt handler is running:

- Other interrupts may be delayed.
- Hardware keeps waiting.
- User input can become unresponsive.

Imagine a network interrupt taking **100 ms**.

During those 100 ms:

- Keyboard interrupts may be delayed.
- Mouse movement may lag.
- Incoming packets may be dropped.

Therefore, interrupt handlers must finish as quickly as possible.

---

# Linux Solution

Linux splits interrupt handling into **two parts**.

## 1. Top Half (Hard IRQ)

Runs immediately after the interrupt.

Characteristics:

- Runs in Interrupt Context
- Interrupts are disabled
- Must execute extremely quickly
- Cannot sleep

Its job is only to:

1. Acknowledge (silence) the hardware.
2. Copy important data from hardware registers.
3. Schedule the Bottom Half.
4. Return immediately.

Usually this takes only a few microseconds.

---

## 2. Bottom Half (Deferred Work)

The Bottom Half performs the expensive work later.

Examples:

- Parsing network packets
- Processing USB data
- Waking user-space processes

Linux mainly uses:

- **Softirq**
- **Workqueue**

---

## Execution Flow

```text
 [ Hardware ] signals APIC
      |
      v
 [ CPU Traps to IDT (Interrupt Descriptor Table) ]
      |
 [ Top Half (Hard IRQ - e.g., do_IRQ) ]
      |
      |--> Acknowledge hardware
      |--> Copy important data
      |--> Schedule Bottom Half
      |
      +---- CPU resumes interrupted task

...later...

 [ Bottom Half ]
      |
      +--> Softirq
      |
      +--> Workqueue
```

---

# Softirq vs Workqueue

## Softirq

Runs later, but still in **Interrupt Context**.

Characteristics:

- Very fast
- Cannot sleep
- Used for high-performance work

Example:

Network packet processing.

---

## Workqueue

Runs inside a normal kernel thread.

Characteristics:

- Runs in Process Context
- Can sleep
- Can take mutexes
- Can perform disk I/O

Used for heavy or slow operations.

---

# Important Data Structures

Location:

```text
include/linux/irqdesc.h
include/linux/interrupt.h
```

---

## `struct irq_desc`

Represents one interrupt line inside the kernel.

Think of it as:

```text
IRQ Number
     |
     +---- List of registered interrupt handlers
```

Each IRQ has one `irq_desc`.

---

## `struct irqaction`

One interrupt line may be shared by multiple devices.

Each registered driver gets its own:

```c
struct irqaction
```

The kernel stores them as a linked list.

Example:

```text
IRQ 12
   |
   +--> Mouse Driver
   |
   +--> Touchpad Driver
   |
   +--> Other Device
```

When IRQ 12 fires, Linux calls every registered handler until the correct device claims the interrupt.

---

# Complete Execution Flow

## 1. Driver Registration

A driver registers its interrupt handler using:

```c
request_irq()
```

The kernel creates a new:

```c
struct irqaction
```

and links it to the IRQ.

---

## 2. Hardware Fires

The device raises an interrupt.

CPU jumps into:

```text
arch/x86/entry/entry_64.S
```

---

## 3. Generic IRQ Handler

Assembly transfers control to:

```text
common_interrupt()
```

Eventually reaching:

```text
__handle_irq_event_percpu()
```

Located in:

```text
kernel/irq/handle.c
```

---

## 4. Top Half Executes

Linux walks through every registered:

```text
irqaction
```

and calls each driver's interrupt handler.

---

## 5. Schedule Bottom Half

The driver may defer work using:

```c
schedule_work()
```

or raise a Softirq.

Then it returns:

```c
IRQ_HANDLED
```

---

## 6. Return

Before returning to the interrupted program:

- Linux checks for pending Softirqs.
- Executes them if needed.
- Restores the previous CPU state.

Execution then continues exactly where it stopped.

---

# Important Kernel Idioms

## `in_interrupt()` / `in_task()`

Some kernel functions can run in both:

- Process Context
- Interrupt Context

Before doing something that may sleep:

```c
if (!in_task()) {
    /* We are inside an interrupt.
       Do NOT sleep. */
}
```

These helper macros tell the kernel what execution context it is currently in.

---

## `GFP_ATOMIC`

Normally:

```c
kmalloc()
```

may sleep while searching for free memory.

Inside an interrupt this is forbidden.

Instead use:

```c
kmalloc(size, GFP_ATOMIC);
```

Meaning:

> Allocate memory immediately.
>
> If memory isn't available, fail.
>
> Do not sleep.

---

## `container_of()` Returns Again

Workqueues use:

```c
struct work_struct
```

Drivers usually embed it inside their own private structures.

Example:

```c
struct my_device {
    ...
    struct work_struct work;
};
```

When the workqueue executes, it only provides:

```c
struct work_struct *
```

The driver then uses:

```c
container_of()
```

to recover the pointer to:

```c
struct my_device
```

---

# Summary

Hardware interrupts happen asynchronously.

To keep the system responsive, Linux divides interrupt handling into two stages:

**Top Half**

- Runs immediately
- Extremely fast
- Cannot sleep
- Acknowledges hardware
- Schedules deferred work

**Bottom Half**

- Runs later
- Performs the heavy processing

Bottom Halves can be implemented using:

- Softirq
- Workqueue

The most important rule is to always know your execution context:

| Process Context | Interrupt Context |
|-----------------|-------------------|
| Can sleep | Cannot sleep |
| Can block | Cannot block |
| Can use mutexes | Cannot use mutexes |
| Heavy work allowed | Must finish quickly |

---

# Mental Model

Think of a **Top Half** as a paramedic responding to an emergency.

The paramedic:

- Arrives immediately.
- Stabilizes the patient.
- Loads them into the ambulance.
- Leaves as quickly as possible.

They **do not** perform surgery at the accident scene.

The **Bottom Half** is the hospital surgeon.

The surgeon works in a safe environment where they can:

- Take their time
- Perform complex operations
- Wait for resources if necessary

---

# Important Functions to Remember

```c
request_irq()
```

Registers a device driver's Top Half interrupt handler.

```c
do_IRQ()
common_interrupt()
```

High-level kernel entry points for interrupt handling.

```c
schedule_work()
```

Queues a Bottom Half to run later in Process Context.

---

# Important Structs to Remember

```c
struct irqaction
```

Represents one registered interrupt handler.

```c
struct work_struct
```

Represents deferred work executed by a Workqueue.

---

# Code Reading Exercises

### 1.

Open:

```text
include/linux/interrupt.h
```

Find:

```c
request_irq()
```

Notice that it is a wrapper around:

```c
request_threaded_irq()
```

Study its parameters:

- irq
- handler
- flags
- name
- dev_id

---

### 2.

Open:

```text
kernel/workqueue.c
```

Read the large comment block at the top.

It explains how worker threads manage deferred jobs.

---

### 3.

Open:

```text
kernel/irq/handle.c
```

Find:

```c
__handle_irq_event_percpu()
```

Look for the loop that walks:

```c
action->next
```

This is how Linux handles shared IRQ lines.

---

# Questions for Understanding

1. If two devices share **IRQ 10**, how does Linux determine which device generated the interrupt?

2. Why might a network driver choose a **Softirq** instead of a **Workqueue**? What are the advantages and trade-offs?

3. What would happen if a driver accidentally acquired a **mutex** inside its Top Half interrupt handler?

---

# Suggested Next Files to Read

Now that you understand:

- Memory management
- Process execution
- Hardware interrupts

The next step is synchronization between multiple CPUs and execution contexts.

```text
include/linux/spinlock.h
include/linux/mutex.h
Documentation/RCU/
```

These files introduce:

- Spinlocks
- Mutexes
- Read-Copy-Update (RCU), one of Linux's most powerful synchronization mechanisms.
