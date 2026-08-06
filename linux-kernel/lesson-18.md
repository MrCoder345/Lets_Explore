# Lesson 18 — eBPF & Tracing (Modern Linux Internals)

## 1. Why eBPF Exists

Historically, if you wanted to observe exactly what the Linux kernel was doing (tracing) or dynamically alter network packet processing, you had to write a **Kernel Module**.

Writing kernel modules is dangerous. A single null-pointer dereference or an infinite loop in a module will instantly cause a Kernel Panic and crash the entire server. Furthermore, maintaining custom modules across different kernel updates is a nightmare.

**eBPF (Extended Berkeley Packet Filter)** was created to solve this. It provides a way to run user-defined code *inside* the kernel, at native speed, with absolute safety. It guarantees that your code will never crash the kernel, never loop infinitely, and never access unauthorized memory.

## 2. History (BPF → eBPF)

- **Classic BPF (cBPF):** Introduced in 1992, this was a tiny virtual machine in the kernel used exclusively for filtering network packets (this is the technology powering `tcpdump`). It was highly restricted.
- **Extended BPF (eBPF):** In 2014, kernel engineers drastically expanded this virtual machine. They upgraded it to 64-bit, increased the registers, added data structures (Maps), and allowed it to be attached to almost *anything* in the kernel, not just network packets. Today, when we say "BPF", we almost always mean "eBPF".

## 3. eBPF Architecture

Think of eBPF as a highly secure, sandboxed Virtual Machine running inside the Linux kernel.

```text
[ User Space ]
      |
      |   1. Write C code -> Compile via LLVM -> eBPF Bytecode
      |   2. System Call: bpf() loads the bytecode into the kernel
      v
---------------------------------------------------------------------
[ Kernel Space ]

      +-------------------+
      |   eBPF Verifier   | <--- Rejects dangerous/buggy code!
      +-------------------+
               | (If safe)
               v
      +-------------------+       +-------------+
      |   JIT Compiler    |       |  eBPF Maps  | <--- Shared memory
      +-------------------+       +-------------+      with User Space
               | (Native Code)           ^
               v                         |
      +-----------------------------------------+
      |               eBPF Hook                 | 
      | (e.g., Network, Syscall, Tracepoint)    |
      +-----------------------------------------+
```

## 4. eBPF Bytecode

eBPF is not a high-level language. It is a custom, RISC-like instruction set architecture.

- It has 11 64-bit registers (`R0` to `R10`).
- `R0` stores the return value.
- `R1` to `R5` hold arguments passed to functions.
- `R10` is a strictly read-only frame pointer (pointing to the stack).
- It has a tiny, strict **512-byte stack limit**.

You don't write eBPF assembly by hand. You write standard C code, and the **LLVM/Clang** compiler translates your C into eBPF bytecode.

## 5. The Verifier

The Verifier is the magic that makes eBPF safe. Before the kernel allows your eBPF bytecode to run, the Verifier analyzes it using a Directed Acyclic Graph (DAG) to simulate every possible execution path.

It guarantees:

1. **No infinite loops:** (Historically, no loops were allowed at all; modern kernels allow strictly bounded loops).
2. **No invalid memory access:** It verifies that all pointer accesses are strictly within safe bounds. You cannot read arbitrary kernel memory.
3. **No uninitialized variables:** You cannot read memory before writing to it (preventing data leaks).
4. **Termination:** The program must reach an exit instruction.

If the Verifier finds even a *hint* of danger, it rejects the program with an error.

## 6. JIT Compiler (Just-In-Time)

Once the Verifier approves the bytecode, the kernel does not interpret it. The kernel's **JIT Compiler** instantly translates the eBPF bytecode into native machine code (e.g., x86_64 or ARM64 assembly).

This means your eBPF program runs at bare-metal execution speeds, identical to compiled kernel C code.

## 7. Maps

How does an eBPF program output data? It cannot call `printf` to user-space.

It uses **eBPF Maps**. These are generic key/value data structures residing in kernel memory.

- Types include: Hash tables, Arrays, LRU caches, Ring buffers.
- **Data Sharing:** The eBPF program updates the map (e.g., incrementing a counter for a specific IP address), and a user-space Go or Python program periodically reads that map using the `bpf()` system call.

## 8. Programs and 9. Hooks

An eBPF program does nothing until it is attached to a **Hook**. A hook is an event in the kernel. When the event fires, the eBPF program runs.

Common program types and their hooks:

| Program Type | Hook |
|--------------|------|
| `BPF_PROG_TYPE_KPROBE` | Attaches to almost any kernel function entry |
| `BPF_PROG_TYPE_TRACEPOINT` | Attaches to static kernel tracepoints |
| `BPF_PROG_TYPE_XDP` | Attaches to the network driver |
| `BPF_PROG_TYPE_CGROUP_SKB` | Attaches to network traffic for a specific cgroup |

## 10. XDP (eXpress Data Path)

XDP is a revolutionary networking feature powered by eBPF.
Normally, when a packet arrives (Lesson 14), the kernel allocates a massive `sk_buff` structure and parses the headers. This is slow.

XDP places an eBPF hook **inside the NIC driver itself**, *before* the `sk_buff` is even allocated!
An XDP program can read the raw packet bytes in RAM and instantly return:

- `XDP_DROP`: Silently kill the packet (e.g., for 100Gbps DDoS mitigation).
- `XDP_TX`: Bounce the packet right back out the network card.
- `XDP_PASS`: Send it up the normal Linux network stack.

## 11. kprobes (Kernel Probes)

**kprobes** allow you to dynamically insert a breakpoint into almost any running kernel function.

- **kprobe:** Fires when the function is called. You can inspect the arguments (`R1`-`R5`).
- **kretprobe:** Fires when the function returns. You can inspect the return value (`R0`).

*(Warning: Because kernel internal functions change between Linux versions, kprobe-based scripts often break when you update your kernel).*

## 12. uprobes (User Probes)

Exactly like kprobes, but for user-space programs!
You can attach an eBPF program to a function inside `libc.so` (like `malloc`), or even a function inside your own compiled C/C++/Go binary. When the user-space app hits that instruction, it traps into the kernel, runs your eBPF code, and resumes the app.

## 13. Tracepoints

Because kprobes break when kernel internals change, kernel developers added **Tracepoints**.
These are static, heavily documented, hardcoded hooks placed in critical areas of the kernel (like `sched_switch` for context switches, or `sys_enter_openat` for opening files). They provide a stable API (ABI) that will not break across kernel updates.

## 14. perf Events

Historically, `perf` was a tool to read hardware performance counters (e.g., CPU cache misses). The eBPF subsystem deeply integrated with the `perf` subsystem. eBPF programs can be triggered by `perf` events (like "run this eBPF program every 10,000 CPU cycles"), and eBPF uses `perf` ring buffers to stream large amounts of trace data to user-space in real-time.

## 15. BTF (BPF Type Format)

In the past, to compile an eBPF program that reads a `task_struct`, you needed the exact kernel headers (gigabytes of C files) installed on your production server.

**BTF** is a highly compressed metadata format. The modern Linux kernel compiles a description of every single one of its data structures and embeds it directly into the `vmlinux` binary (usually accessible at `/sys/kernel/btf/vmlinux`).

## 16. CO-RE (Compile Once – Run Everywhere)

BTF enabled the Holy Grail of tracing: **CO-RE**.
Because of BTF, you can compile your eBPF program *once* on your laptop. When you copy the binary to a production server with a totally different kernel version, the `libbpf` loader reads the target server's BTF data.

If the `pid` field in `task_struct` moved from byte offset 120 to byte offset 128 in the new kernel version, `libbpf` automatically patches your eBPF bytecode in memory to use the correct offset before loading it into the kernel!

## 17. libbpf Overview

`libbpf` is the official user-space C library for interacting with eBPF. It handles reading the ELF object file generated by Clang, performing CO-RE relocations, creating the Maps, and issuing the `bpf()` system calls to load everything into the kernel.

## 18. Important APIs

**User-Space API (System Call):**

- `int bpf(int cmd, union bpf_attr *attr, unsigned int size);`: The master syscall for all eBPF operations.

**Kernel-Space API (eBPF Helper Functions):**
eBPF code cannot call standard kernel functions. It can only call specific "helpers" provided by the kernel:

- `bpf_map_lookup_elem()`: Fetch a value from a Map.
- `bpf_map_update_elem()`: Write a value to a Map.
- `bpf_trace_printk()`: The "printf" of eBPF. Prints debugging text to `/sys/kernel/tracing/trace_pipe`.
- `bpf_get_current_pid_tgid()`: Gets the PID of the user-space process that triggered the hook.

## 19. Important Structs

- `union bpf_attr`: The massive structure passed to the `bpf()` syscall. It contains the payload for commands like `BPF_PROG_LOAD` or `BPF_MAP_CREATE`.
- `struct bpf_prog`: The kernel's internal representation of a loaded, verified eBPF program.
- `struct bpf_map`: The kernel's internal representation of a data map.

## 20. Execution Flow

Here is how you trace all `open()` system calls:

1. **Code:** You write C code defining an eBPF program of type `TRACEPOINT` targeting `syscalls/sys_enter_openat`.
2. **Compile:** Clang compiles it to an ELF `.o` file containing eBPF bytecode.
3. **Load:** Your user-space loader (using `libbpf`) parses the `.o` file, performs CO-RE, creates the required Maps, and calls `bpf(BPF_PROG_LOAD)`.
4. **Verify:** The in-kernel Verifier checks the bytecode for safety.
5. **JIT:** The JIT compiler translates it to x86 assembly.
6. **Attach:** The loader attaches the program to the tracepoint.
7. **Execution:** A user runs `cat /etc/passwd`. The `openat` syscall fires.
8. **Trace:** The tracepoint triggers. The kernel pauses, executes your native JIT'd eBPF code, updates a Map, and then resumes the syscall instantly.
9. **Consume:** Your user-space loader reads the Map and prints "Process X opened /etc/passwd".

## 21. Source Files to Explore

| File Path | Purpose |
|-----------|---------|
| `kernel/bpf/syscall.c` | The entry point for the `bpf()` system call and map creation |
| `kernel/bpf/verifier.c` | The legendary, massively complex eBPF Verifier |
| `include/linux/bpf.h` | The main kernel definitions |
| `include/uapi/linux/bpf.h` | The user-space facing header (defines all `bpf_cmd`s and helpers) |
| `tools/lib/bpf/` | The source code for the `libbpf` user-space library |

## 22. Mental Model

Think of eBPF as **JavaScript for the Linux Kernel**.

- Just as a web browser is a complex C++ engine that safely runs user-provided JavaScript scripts in a sandbox to dynamically alter a webpage (DOM)...
- The Linux kernel is a complex C engine that safely runs user-provided eBPF bytecode in a sandbox to dynamically observe or alter system behaviors.

## 23. Summary

eBPF has revolutionized Linux observability, networking, and security. By providing an in-kernel, JIT-compiled virtual machine backed by a rigorous static Verifier, engineers can execute custom logic in response to almost any kernel event safely and at native speeds. Technologies like BTF and CO-RE have made eBPF portable across kernel versions, while XDP allows for unprecedented networking performance bypassing standard kernel overhead.

## 24. Code Reading Exercises

1. Open `include/uapi/linux/bpf.h`. Search for `enum bpf_cmd`. Look at all the commands you can issue via the `bpf()` syscall (e.g., `BPF_MAP_CREATE`, `BPF_PROG_LOAD`).
2. In the same file, search for `enum bpf_prog_type`. Look at the massive list of places eBPF can be attached (Kprobes, Tracepoints, XDP, Cgroups).
3. Open `kernel/bpf/verifier.c` and search for `do_check()`. This is the core loop where the Verifier walks the execution DAG and checks register states.

## 25. Questions for Understanding

1. If an eBPF program attached to an XDP hook wants to block an IP address, why doesn't it just use a standard `while()` loop to wait for user-space to confirm the IP? *(Hint: What rule does the Verifier enforce about execution?)*

2. Why are Tracepoints generally preferred over kprobes for long-term production observability tools?

3. If a developer compiles an eBPF program on a Linux 5.15 kernel, how does CO-RE allow that exact same compiled binary to run safely on a Linux 6.1 kernel where the internal structs are a completely different size?

---

## 💡 Answer Key (For Your Reference)

**Q1:** The Verifier enforces **no infinite loops** (or strictly bounded loops). A `while()` loop waiting for user-space would never terminate, causing the kernel to hang or the Verifier to reject the program. XDP programs must execute and return a decision (DROP, PASS, TX) in microseconds—they cannot block or wait. All decisions must be made immediately using in-kernel data structures (like maps).

**Q2:** Tracepoints provide a **stable ABI** (Application Binary Interface). Kernel developers guarantee that tracepoint names and their argument structures will not change between kernel versions. kprobes, on the other hand, attach to internal kernel functions that are implementation details—they can be renamed, removed, or change their function signatures between kernel releases, causing production tools to break unexpectedly.

**Q3:** CO-RE (Compile Once – Run Everywhere) uses **BTF** (BPF Type Format). When the eBPF program is compiled on the 5.15 kernel, it records the field offsets it expects for structures like `task_struct`. At load time on the 6.1 kernel, `libbpf` reads the target kernel's BTF data (which describes the *actual* layout of structures in that kernel). It then automatically relocates and patches the eBPF bytecode to use the correct offsets for the new kernel version, making the binary portable across kernel versions.
