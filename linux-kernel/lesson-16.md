# Lesson 16 — Boot Process (Power On → start_kernel())

## 1. Power On

When you press the power button, the CPU receives an electrical signal to wake up. However, the CPU has no idea what an operating system is. It wakes up in a state of amnesia and immediately looks at a hardcoded physical memory address (the **Reset Vector**) to find its very first instruction.

On x86 processors, to maintain backward compatibility with CPUs from the 1970s, the CPU wakes up in **16-bit Real Mode**. It can only access 1 MB of memory and has no security protections.

The reset vector address is:

- **Legacy BIOS:** `0xFFFF0` (the last 16 bytes of the 1 MB address space) – this contains a jump to the BIOS ROM.
- **UEFI:** The CPU starts in 64-bit mode and jumps to the UEFI firmware entry point in flash memory.

## 2. BIOS vs UEFI

The instruction at the Reset Vector jumps to the motherboard's firmware.

### BIOS (Legacy)

The old standard. It initializes basic hardware (RAM, keyboard), looks for a Master Boot Record (MBR) on the first sector of the hard drive, and blindly executes the 512 bytes of code found there.

**Limitations:**

- 16-bit real mode (only 1 MB addressable).
- MBR limits partitions to 2 TB.
- Slow and outdated.

### UEFI (Modern)

The new standard. It is practically a mini-operating system. It understands partition tables (GPT) and filesystems (FAT32). It looks for an EFI System Partition (ESP) and runs a `.efi` boot application.

**Advantages:**

- 64-bit mode from the start.
- Supports large disks (GPT).
- Secure Boot and faster boot times.

The firmware's only job is to find the **Bootloader** and hand over control.

## 3. Bootloader

The bootloader (like **GRUB** or **systemd-boot**) is the bridge between the firmware and the Linux kernel.

The bootloader reads its configuration file, presents a menu to the user, and then does two critical things:

1. Loads the **Kernel Image** from the hard drive into RAM.
2. Loads the **initramfs** (Initial RAM Filesystem) into RAM. (This contains early drivers needed to mount the real root filesystem).

Finally, the bootloader jumps to the start of the kernel image in memory, passing along a structure called `boot_params` which contains information about the hardware and the kernel command line.

## 4. Kernel Image (`bzImage`)

The file you see in `/boot` (e.g., `vmlinuz-linux`) is not a standard executable. It is a **bzImage** (Big Zipped Image).
It is actually a self-extracting archive containing:

1. Early 16-bit/32-bit setup code.
2. A decompression stub.
3. The actual, compressed 64-bit kernel (`vmlinux`).

**Why compression?**

A modern kernel is about 30-50 MB uncompressed. Compressed, it fits in about 5-10 MB of memory, drastically reducing boot time (less data to read from slow storage).

## 5. Early Assembly and Decompression (`arch/x86/boot/`)

Once the bootloader jumps to the kernel, the architecture-specific boot code takes over.

1. **`header.S`**: Execution starts here. It defines the magic signatures and boot protocol the bootloader used.
2. **`main.c`**: Execution jumps to C code (still in 16-bit mode). It queries the BIOS/UEFI for the physical memory map, sets up a basic video mode, and prepares the CPU.
3. **Protected Mode:** The CPU is switched into 32-bit Protected Mode, gaining access to more memory.
4. **Decompression:** Execution jumps to `arch/x86/boot/compressed/head_64.S`. The decompression stub unpacks the real `vmlinux` kernel into its final physical memory location.

### Decompression Key Functions

- `startup_64()`: Entry point for 64-bit decompressor (assembly).
- `extract_kernel()`: The C function that performs the actual decompression.
- `decompress_kernel()`: Wrapper that calls the appropriate algorithm (gzip, xz, lzma, etc.).

**Source files:**

- `arch/x86/boot/compressed/head_64.S`
- `arch/x86/boot/compressed/misc.c`
- `lib/decompress_*.c` (e.g., `lib/decompress_inflate.c` for gzip)

## 6. Early Boot (`arch/x86/kernel/`)

Now we have an uncompressed kernel, but the CPU is still in 32-bit mode.

Execution jumps to `startup_64` in `arch/x86/kernel/head_64.S`. This assembly file performs the final hardware transition:

1. It switches the CPU into **64-bit Long Mode**.
2. It creates the very first, basic **Page Tables** (Virtual Memory).
3. It sets up a minimal C stack.
4. It clears the BSS segment (zeroing out uninitialized global variables).
5. Finally, it executes the most important jump instruction in the kernel: `jmp start_kernel`.

At this point, we have a 64-bit environment with paging enabled, and we are about to enter the architecture-agnostic part of the kernel.

## 7. `start_kernel()`

Welcome to `init/main.c`.
`start_kernel()` is the mother of all initialization functions. It is architecture-agnostic standard C code. When this function begins, the kernel has no scheduler, no memory allocator, and no interrupts. By the time it finishes, you have a fully functioning operating system.

It acts as a giant checklist, calling subsystem initialization functions one by one.

### Code snippet: start_kernel() simplified

```c
asmlinkage __visible void __init start_kernel(void)
{
    char *command_line;
    
    /* Early debugging */
    boot_init_stack_canary();
    
    /* Architecture-specific setup (memory map, CPU features) */
    setup_arch(&command_line);
    
    /* Print Linux banner */
    pr_notice("%s", linux_banner);
    
    /* Interrupt handling setup */
    trap_init();
    init_IRQ();
    
    /* Memory management */
    mm_init();
    
    /* Scheduling */
    sched_init();
    
    /* Time and timers */
    time_init();
    init_timers();
    
    /* Create kernel threads and launch init */
    rest_init();
}
```

## 8. Memory Initialization

Inside `start_kernel()`, memory is bootstrapped in phases:

### Early Memory (memblock)

- `setup_arch()` parses the physical memory map (from BIOS/UEFI) and populates the `memblock` allocator. This is a simple early allocator used before the buddy system is ready.

### Buddy Allocator

- `mm_init()` calls `mem_init()`, which initializes the buddy allocator (page allocator). From this point, `alloc_pages()` works.

### Slab Allocator

- `kmem_cache_init()` initializes the slab (or slub) allocator for small kernel objects. After this, `kmalloc()` becomes available.

**Memory initialization order:**

```c
start_kernel()
  ├── setup_arch()         /* memblock setup */
  ├── mm_init()
  │   ├── mem_init()       /* buddy allocator */
  │   ├── kmem_cache_init()/* slab allocator */
  │   └── pgtable_init()   /* finalize page tables */
  └── ...
```

**Source files:**

- `mm/memblock.c`
- `mm/page_alloc.c`
- `mm/slub.c`

## 9. Scheduler Initialization

Next, `start_kernel()` calls `sched_init()`.
This initializes the Completely Fair Scheduler (CFS), the per-CPU Runqueues, and the wait queue infrastructure.

However, multitasking hasn't started yet. There is currently only one thread of execution (the boot CPU). The kernel manually crafts the very first `task_struct` in memory, famously known as `init_task` (PID 0).

## 10. The `init` Process

At the very end of `start_kernel()`, it calls `rest_init()`.

1. `rest_init()` calls `kernel_thread()` to spawn a brand new thread. This thread is given **PID 1**.
2. PID 1 runs the function `kernel_init()`. Its job is to find the user-space `init` program (like `/sbin/init` or `systemd`) on the hard drive and execute it. **This is the birth of user-space.**
3. Meanwhile, the original boot thread (PID 0) has finished its job. It calls `cpu_idle()`. It becomes the **Idle Task**. Whenever the CPU has absolutely no other processes to run, it falls back to PID 0, which simply halts the CPU to save power.

### Code snippet: rest_init()

```c
static noinline void __init rest_init(void)
{
    /* Create PID 1 (init process) */
    pid_t pid = kernel_thread(kernel_init, NULL, CLONE_FS);
    
    /* Create PID 2 (kthreadd) */
    kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
    
    /* Become the idle process (PID 0) */
    init_idle(current, smp_processor_id());
    cpu_startup_entry(CPUHP_ONLINE);
}
```

## 11. Source Files to Explore

| File Path | Purpose |
|-----------|---------|
| `arch/x86/boot/header.S` | Legacy 16-bit entry point and boot protocol definitions |
| `arch/x86/boot/main.c` | Early hardware probing before decompression |
| `arch/x86/boot/compressed/head_64.S` | Decompressor entry point (64-bit) |
| `arch/x86/boot/compressed/misc.c` | Decompression logic |
| `arch/x86/kernel/head_64.S` | 64-bit entry point, page table setup, jump to `start_kernel` |
| `init/main.c` | The home of `start_kernel()` and `rest_init()` |
| `mm/memblock.c` | Early boot memory allocator |
| `mm/page_alloc.c` | Buddy allocator initialization |
| `kernel/sched/core.c` | Scheduler core initialization |

## 12. Execution Timeline

```text
[ Power Button ]
      |
[ BIOS / UEFI ] (Hardware init, loads Bootloader)
      |
[ Bootloader ] (Loads bzImage and initramfs to RAM)
      |
[ arch/x86/boot/header.S ] (16-bit entry)
      |
[ arch/x86/boot/main.c ] (Memory map, video setup)
      |
[ arch/x86/boot/compressed/head_64.S ] (Decompresses vmlinux)
      |
[ arch/x86/kernel/head_64.S ] (64-bit Long Mode, early Page Tables)
      |
[ init/main.c ] -> start_kernel()
      |
      +---> setup_arch() (Architecture init)
      +---> mm_init()    (Memory allocators)
      +---> sched_init() (Runqueues)
      |
[ rest_init() ]
      |
      +---> Spawns PID 1 (kernel_init -> /sbin/init) (User-space begins!)
      +---> PID 0 enters cpu_idle() loop.
```

## 13. Mental Model

Think of booting a computer like **colonizing a new planet**.

1. **Firmware/Bootloader:** The unmanned probe. It lands on the planet, checks the atmosphere (hardware), and unloads the compressed habitat module (`bzImage`).
2. **Early Assembly (`arch/`):** The automated unpacking sequence. It inflates the habitat, turns on the life support (Page Tables), and powers up the 64-bit mainframe.
3. **`start_kernel()`:** The base commander waking up. They sit at the console and turn on the subsystems one by one: water (Memory), work schedules (Scheduler), and communications (Interrupts).
4. **`rest_init()`:** The commander opens the front door and lets the colonists (User-Space / PID 1) out to start living, while the commander retires to the basement (PID 0) to monitor the power grid.

## 14. Summary

The Linux boot process is a complex chain of trust and hardware state transitions. Because PC hardware starts in a primitive 16-bit state, architecture-specific assembly code (`arch/x86`) must carefully transition the CPU into 64-bit mode, set up virtual memory, and uncompress the kernel. Once the environment is stable, execution jumps to the generic C function `start_kernel()`, which systematically initializes the OS subsystems before spawning the first user-space process (PID 1) and putting the boot thread to sleep as the Idle Task (PID 0).

## 15. Exercises

1. Open `init/main.c` and find `start_kernel()`. Scroll through the function and look at how many subsystems are initialized. Find `trap_init()`, `mm_init()`, and `sched_init()`.
2. In `init/main.c`, find `rest_init()`. Observe the calls to `kernel_thread()` and notice that it creates PID 1 and PID 2. See how it finishes by calling `cpu_startup_entry()`, which loops forever as the idle task.
3. Open `arch/x86/kernel/head_64.S`. Search for the string `start_kernel`. This is the exact assembly instruction that bridges the architecture-specific world to the generic C world.
4. Run `dmesg | head -20` on your system and locate the first message from the kernel. Notice the timestamp starts at 0.000000.

## 16. Questions

1. Why is the kernel stored on disk in a compressed format (`bzImage`) instead of just being an uncompressed executable file?

2. What is the fundamental difference between PID 0 (the Idle Task) and PID 1 (`init`) at the end of the boot process?

3. If `start_kernel()` is written in standard C, why can't the bootloader just jump directly to it, skipping the `arch/x86/boot` assembly entirely?

---

## 💡 Answer Key (For Your Reference)

**Q1:** Compression reduces the kernel image size significantly (from ~30 MB to ~5 MB), which speeds up loading from disk and reduces memory footprint during boot. The decompressor adds minimal overhead compared to the I/O savings.

**Q2:** PID 0 (Idle Task) is the **scheduler's fallback**—it runs when no other process is ready. It has no user-space memory and simply halts the CPU to save power. PID 1 (`init`) is the **first user-space process**; it sets up the user environment, spawns all other processes, and acts as the ultimate parent of all orphaned processes.

**Q3:** The bootloader cannot jump directly to `start_kernel()` because the CPU starts in 16-bit real mode, with paging disabled and no memory management. The assembly code in `arch/x86/boot` is essential to:
- Transition the CPU through protected mode to 64-bit long mode.
- Set up initial page tables.
- Decompress the kernel to its final location.
- Clear BSS and set up a C stack.
- Only after these steps can a C function (like `start_kernel`) be safely called.
