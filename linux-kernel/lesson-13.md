# Lesson 13 — Block Layer & I/O Stack

## 1. Storage Stack Overview

In Lesson 12, we learned how the Virtual File System (VFS) abstracts files and directories. But when the ext4 filesystem decides it needs to write 4 KB of data to disk, how does that data actually reach the physical hardware?

The kernel storage stack is a multi-layered cake:

```text
[ User Space ]       open(), read(), write()
      |
[   VFS   ]          The abstraction layer (Inodes, Dentries)
      |
[ Filesystem ]       ext4, btrfs, xfs (Translates files to block numbers)
      |
[ Block Layer ]      <-- WE ARE HERE. Queues, sorts, and merges I/O.
      |
[ Device Driver ]    NVMe, SATA (AHCI), SCSI (Talks to the hardware)
      |
[  Hardware  ]       The physical SSD or HDD
```

The **Block Layer** sits between the filesystem and the device driver. Its primary job is **I/O Optimization**. Disks (especially mechanical ones) are the slowest components in a computer. The block layer ensures we talk to them as efficiently as possible.

## 2. Block Devices

In Linux, hardware devices are generally split into two categories:

- **Character Devices (`char`):** Accessed as a stream of bytes. You read them sequentially (e.g., keyboards, mice, serial ports).
- **Block Devices (`block`):** Accessed in fixed-size chunks called **blocks** (typically 512 bytes or 4 KB). You can randomly seek to any block on the device and read it (e.g., SSDs, HDDs, USB drives).

The block layer is exclusively dedicated to managing block devices.

## 3. BIO (`struct bio`)

When a filesystem wants to read or write data, it creates a **BIO** (`struct bio`).
The BIO is the fundamental unit of I/O in the Linux kernel. It represents an active, in-flight I/O operation.

- **What it contains:**
  - The target block device (`struct block_device`).
  - The physical sector number on the disk to read/write.
  - The direction (Read or Write).
  - A list of memory pages in RAM where the data should be copied to/from (this list is called a `bio_vec`).

Instead of sending one byte at a time, the filesystem packages the request into a `bio` and submits it to the block layer using the function `submit_bio()`.

## 4. Request Queue (`struct request_queue`)

If every `bio` went straight to the hardware immediately, the disk would be overwhelmed, and performance would plummet.

Instead, every block device in the system has a **Request Queue** (`struct request_queue`). When `submit_bio()` is called, the block layer converts the `bio` into a `struct request` and places it into this queue.

Why convert it? Because a `struct request` can contain **multiple** `bio` structures. If two `bio`s are trying to write to adjacent sectors on the disk, the block layer merges them into a single `request`.

## 5. Elevator Scheduler

Mechanical Hard Disk Drives (HDDs) have a physical read/write head that must swing across spinning magnetic platters. If you ask an HDD to read sector 1, then sector 10000, then sector 2, the head thrashes back and forth, destroying performance.

To solve this, the block layer historically used an **I/O Scheduler** (often called an Elevator).
Just like a real elevator doesn't go Up, Down, Up, Down based on who pressed the button first, the I/O Scheduler sorts the `struct request` queue by physical disk sector number.

Algorithms included:

- **NOOP:** First-in, First-out (does nothing).
- **Deadline:** Ensures requests don't wait forever by setting an expiration time.
- **CFQ (Completely Fair Queuing):** Distributes I/O bandwidth evenly among processes.

## 6. blk-mq (Block Multi-Queue)

The Elevator was great for mechanical drives delivering 150 IOPS (Input/Output Operations Per Second).
Then, modern NVMe SSDs arrived, capable of 1,000,000+ IOPS. The single `request_queue` became a massive bottleneck. Having 64 CPU cores all trying to take a single Spinlock to insert a `request` into one queue caused disastrous lock contention.

The kernel developers completely rewrote the block layer to create **blk-mq** (Block Multi-Queue).

```text
[ CPU 0 ]   [ CPU 1 ]   [ CPU 2 ]   [ CPU 3 ]
    |           |           |           |
[ SW Queue] [ SW Queue] [ SW Queue] [ SW Queue]   <-- Software Queues (One per CPU core)
    \           /           \           /
     \         /             \         /
    [ HW Queue 0 ]          [ HW Queue 1 ]        <-- Hardware Queues (On the SSD controller)
          |                       |
     [ NVMe SSD (Supports multiple queues) ]
```

- **How it works:** `blk-mq` gives every CPU core its own lockless software queue. The block layer then maps these software queues to the multiple hardware submission queues physically built into modern NVMe drives.
- **The Result:** CPUs no longer fight over locks. I/O scales linearly with the number of CPU cores.

## 7. Device Driver Interaction

The block layer is agnostic; it doesn't know *how* to talk to an NVMe drive or a SATA drive.
Once the `struct request` makes it through the queues, the block layer hands it over to the specific device driver (e.g., `drivers/nvme/host/core.c`).

The driver reads the sector numbers and memory addresses from the `request` and translates them into electrical signals for the hardware controller.

## 8. DMA Overview (Direct Memory Access)

When reading 4 MB of data from a disk, the CPU does not read it byte-by-byte. That would waste millions of CPU cycles.

Instead, Linux uses **DMA (Direct Memory Access)**.

1. The driver tells the disk controller: "Read sector 500, and copy the data directly into this physical RAM address."
2. The CPU is now free to run other tasks.
3. The disk controller hardware writes the data directly into the system's RAM over the PCIe bus.
4. When finished, the disk fires a hardware interrupt (Lesson 8).
5. The CPU wakes up, checks the RAM, and the data is magically there.

To support this, the `bio_vec` inside the `bio` uses **Scatter-Gather Lists** to give the hardware a map of exact physical memory pages to write to.

## 9. I/O Flow

Let's trace a disk write from start to finish:

1. **Filesystem:** ext4 determines it needs to write to block 1024. It creates a `struct bio` and calls `submit_bio()`.
2. **Block Layer:** The BIO enters the `blk-mq` subsystem.
3. **Merging:** The block layer checks the software queue for this CPU. Is there already a `request` for block 1023? If yes, it merges this `bio` into that `request`.
4. **Dispatch:** If no merge is possible, it creates a new `request`. The `request` is pushed to the hardware dispatch queue.
5. **Driver:** The NVMe driver takes the `request` and maps the memory for DMA.
6. **Hardware:** The NVMe controller fetches the data via DMA and writes it to flash memory.
7. **Completion:** The NVMe drive fires a hardware interrupt. The kernel ISR marks the `request` as complete, and the waiting user-space task is woken up (using Wait Queues from Lesson 11!).

## 10. Important Structs

- `struct bio`: The filesystem's description of an I/O operation (in-flight data).
- `struct bio_vec`: A tuple of `<page, offset, length>` pointing to physical RAM for DMA.
- `struct request`: The block layer's container. May hold multiple merged `bio`s.
- `struct request_queue`: The queue that holds `struct request`s.
- `struct gendisk`: Represents a physical disk partition (e.g., `/dev/sda`).

## 11. Important APIs

- `submit_bio(struct bio *bio)`: The main entry point used by filesystems to send I/O.
- `blk_mq_submit_bio()`: The modern multi-queue routing function.
- `blk_mq_alloc_request()`: Allocates a new request from the queue's memory pool.
- `bio_for_each_segment()`: A macro used to loop through every memory page attached to a BIO.

## 12. Source Files to Explore

- `block/`: The home of the block layer.
- `block/blk-core.c`: The core initialization and submission logic.
- `block/blk-mq.c`: The modern Block Multi-Queue implementation.
- `block/blk-merge.c`: The incredibly complex logic for safely merging adjacent BIOs.
- `include/linux/blkdev.h`: The master header containing `struct request_queue` and `struct request`.
- `include/linux/bio.h`: The header containing `struct bio`.
- `drivers/block/`: Basic block drivers (like loopback devices and floppy drives). High-performance drivers live in their own subsystem folders (like `drivers/nvme/`).

## 13. Mental Model

Think of the I/O stack as a massive shipping and logistics company.

- **The Filesystem (The Factory):** Packages products into boxes (`struct bio`) and slaps a shipping address (sector number) on them.
- **The Block Layer (The Warehouse):** Receives the boxes. It puts boxes going to the same neighborhood into a single, larger shipping container (`struct request`) to save trips.
- **blk-mq (Multiple Loading Docks):** Instead of one door where all factory workers fight to drop off boxes, there is a door for every worker, leading to multiple fleets of trucks.
- **The Driver & DMA (The Truck):** Takes the container and drives it straight to the destination (RAM), without the warehouse manager (the CPU) having to carry it by hand.

## 14. Summary

The Linux Block Layer sits between the filesystem and the device drivers to maximize storage throughput. It converts filesystem `bio` structures into block layer `request` structures, merging adjacent I/O operations to reduce hardware overhead. To keep up with modern SSDs, the legacy single-queue Elevator schedulers have been largely replaced by `blk-mq`, which maps per-CPU software queues to multiple hardware queues, eliminating lock contention and allowing direct DMA transfers.

## 15. Exercises

1. Open `include/linux/bio.h` and find `struct bio`. Look for `bi_opf` (operation flags like Read/Write) and `bi_sector` (the target disk sector). Look for the `struct bio_vec *bi_io_vec` pointer.
2. Open `include/linux/blkdev.h` and find `struct request`. Notice that it contains a pointer to the `request_queue` it belongs to, and a linked list of `bio` structs attached to it.
3. Open `block/blk-core.c` and search for `submit_bio`. Follow the logic down to see how it eventually delegates to `blk_mq_submit_bio`.

## 16. Questions

1. If two user-space programs simultaneously write 4 KB to sector 100 and 4 KB to sector 101, how does merging them into a single 8 KB `request` improve performance on a mechanical hard drive?

2. Why was a single `request_queue` protected by a Spinlock perfectly fine in the year 2005, but completely disastrous for performance in the year 2020?

3. If a device driver uses DMA to write disk data directly into RAM, how does the CPU know when the transfer is finished so it can wake up the user-space program?

---

## 💡 Answer Key (For Your Reference)

**Q1:** Merging reduces **seek time**. A mechanical HDD has a physical read/write head that must move to the correct position. Reading sectors 100 and 101 separately requires two seek operations (the head moves, waits for the platter to spin, reads, then moves again). Merging them into one 8 KB request means the head moves once, reads both adjacent sectors in one pass, and saves the mechanical movement time (which is measured in milliseconds—an eternity for a CPU).

**Q2:** In 2005, systems typically had **single-core or dual-core** CPUs with relatively low I/O rates (150-200 IOPS). The spinlock contention was minimal. By 2020, servers had **64+ cores** and NVMe drives capable of millions of IOPS. All 64 cores trying to acquire the same spinlock to insert requests into a single queue caused **severe contention**—most time was spent waiting for the lock rather than doing useful work. blk-mq solved this by giving each core its own queue.

**Q3:** The **hardware interrupt (IRQ)** mechanism. The disk controller fires an interrupt signal to the CPU once the DMA transfer is complete. The CPU's interrupt handler (ISR) then runs, which marks the `struct request` as completed, calls `complete()` on the wait queue (Lesson 11), and wakes up the sleeping user-space process. The process can then safely access the data that DMA placed directly into RAM.
