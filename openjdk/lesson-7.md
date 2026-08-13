# OpenJDK Internals: Day 7 – JVM Runtime Data Areas

You know how classes are parsed and verified. But before the JVM can execute a single instruction, it must lay out its memory.

In C, you have the `.data` segment for globals, the `heap` for `malloc`, and the `stack` for local variables. The JVM is a virtual machine running inside your C program, so it must carve out its *own* virtualised versions of these memory areas from the host OS.

In this lesson, we’ll map the JVM’s memory topology: the **Java Heap**, **Metaspace**, **Code Cache**, and **Thread Stacks**. Understanding where each piece of data lives is essential for debugging memory leaks, GC pauses, and performance issues.

---

## 1. The Big Picture (Mental Model)

JVM memory is broadly divided into:

- **Thread-Local** areas – fast, no locks, created/destroyed with the thread.
- **Shared** areas – require synchronisation, managed by GC or subsystems, outlive threads.

```
┌─────────────────────────────── OS Process Memory ───────────────────────────────┐
│                                                                                 │
│  [ Shared: Java Heap ]                 [ Shared: Metaspace ] (Native Mem)       │
│  ┌───────────────────────────────┐     ┌──────────────────────────────────┐     │
│  │ Java Objects                  │     │ Class Metadata (InstanceKlass)   │     │
│  │ Arrays                        │     │ Constant Pools                   │     │
│  │ (Managed by GC)               │     │ Method Bytecodes                 │     │
│  └───────────────────────────────┘     └──────────────────────────────────┘     │
│                                                                                 │
│  [ Shared: Code Cache ] (Native)       [ Shared: Other Native Memory ]          │
│  ┌───────────────────────────────┐     ┌──────────────────────────────────┐     │
│  │ JIT Compiled Machine Code     │     │ DirectByteBuffers (NIO)          │     │
│  │ Interpreter Assembly Stubs    │     │ GC Data Structures (Card Tables) │     │
│  └───────────────────────────────┘     └──────────────────────────────────┘     │
│                                                                                 │
│ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│                                                                                 │
│  [ Thread 1 (Main) ]                   [ Thread 2 ]                             │
│  ┌───────────────────────────────┐     ┌───────────────────────────────┐        │
│  │ PC Register (Instruction ptr) │     │ PC Register                   │        │
│  │ Java / Native Stack           │     │ Java / Native Stack           │        │
│  │ ├─ Frame 2: MyClass.foo()     │     │ ├─ Frame 1: Worker.run()      │        │
│  │ └─ Frame 1: MyClass.main()    │     │ └─ [ Thread-local state ]     │        │
│  └───────────────────────────────┘     └───────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | What the Spec Says | HotSpot Does |
| :------ | :----------------- | :----------- |
| **Method Area** | Holds class structures. | Uses **Metaspace** – native memory (`malloc`/`mmap`), not part of the Java Heap. |
| **Java Stack** | Separate from Native Stack. | **Combines** them – both Java frames and C/C++ frames share the same OS thread stack. |
| **Runtime Constant Pool** | Belongs to the Method Area. | Allocated in Metaspace as a C++ `ConstantPool` object. |
| **Heap** | Stores all objects. | Managed by the Garbage Collector; reserved as a contiguous virtual address space. |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/memory/` | Heap (`universe.cpp`), Metaspace, and memory management (`arena.cpp`). |
| `src/hotspot/share/code/` | Code Cache (`codeCache.cpp`) – JIT-compiled machine code. |
| `src/hotspot/share/gc/shared/` | The abstract `CollectedHeap` and GC‑specific implementations. |
| `src/hotspot/share/runtime/` | Thread handling (`javaThread.cpp`) – stacks and thread-local state. |

---

## 4. Key Concepts You Need to Know

### Virtual vs. Physical Memory
When you set `-Xmx8G`, HotSpot does **not** immediately consume 8GB of RAM. It calls `mmap` with `PROT_NONE` to *reserve* 8GB of contiguous **virtual address space** from the OS. It only *commits* (backs with physical RAM pages) as objects are allocated.

### Native Memory Tracking (NMT)
Because HotSpot allocates memory outside the Java Heap (Metaspace, Code Cache, thread stacks), you can get an `OutOfMemoryError` from the OS even if your heap is empty. NMT is an internal HotSpot subsystem that tracks every byte of C++ native memory it allocates. Enable it with `-XX:NativeMemoryTracking=summary`.

---

## 5. How Memory Is Carved Out (Startup Order)

1. **Thread Stacks** – When a new `JavaThread` is created, HotSpot asks the OS (via `pthread_create`) for a stack (default ~1MB).
2. **Java Heap** – `Universe::initialize_heap()` reserves a large, contiguous virtual address range; the GC then manages it.
3. **Metaspace** – Initialised dynamically. As classes are loaded, `ClassFileParser` requests chunks of native memory for `InstanceKlass` structures. It grows as needed.
4. **Code Cache** – A pre‑reserved chunk of native memory. The JIT writes compiled machine code here. If it fills up, the JIT disables itself (severe performance hit).

---

## 6. Important C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `CollectedHeap` | `gc/shared/collectedHeap.hpp` | Abstract base for the Java Heap; implemented by G1, ZGC, etc. |
| `Metaspace` | `memory/metaspace.hpp` | Manager for native memory used for class metadata. |
| `CodeCache` | `code/codeCache.hpp` | Global manager for all JIT‑compiled code blobs. |
| `JavaThread` | `runtime/javaThread.hpp` | Represents an OS thread; holds stack base and size limits. |
| `Arena` / `ResourceArea` | `memory/arena.hpp` | Fast, thread‑local bump‑pointer allocators for temporary internal C++ objects (e.g., during JIT). |

---

## 7. Critical Functions

- `os::reserve_memory()` / `os::commit_memory()` – low‑level OS wrappers (`mmap` on Linux, `VirtualAlloc` on Windows).
- `Universe::initialize()` – orchestrates the setup of all memory regions.
- `Metaspace::allocate()` – called by `ClassFileParser` to get memory for a new `InstanceKlass`.
- `CodeCache::allocate()` – called by the JIT compiler to store new assembly code.

---

## 8. Important Macros / Utilities

- **`NEW_C_HEAP_ARRAY` / `FREE_C_HEAP_ARRAY`** – HotSpot doesn’t use bare `malloc`/`free`; these macros hook into NMT to track C++ memory allocations and detect leaks.

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: Reserving the Heap
- **Open:** `src/hotspot/share/gc/shared/collectedHeap.cpp`
- **Look for:** the use of `ReservedSpace` and `VirtualSpace`. HotSpot reserves the maximum (`-Xmx`) upfront to guarantee contiguous addresses, but only commits the initial size (`-Xms`).

### Tour 2: The Code Cache
- **Open:** `src/hotspot/share/code/codeCache.hpp` (and `.cpp`)
- **Look for:** `CodeCache::allocate()`. Notice the Code Cache is often split into segments: one for non‑method code (stubs), one for profiled code (C1), and one for highly optimised code (C2).

### Tour 3: Native Memory Tracking
- **Open:** `src/hotspot/share/services/memTracker.hpp`
- **Overview:** Every C++ allocation is tagged with a category (e.g., `mtClass`, `mtThreadStack`, `mtCode`) so developers can track native memory usage.

---

## 10. Execution Flow – Memory Initialisation

```
Threads::create_vm()
    │
    ├─ os::init()                      // get OS page sizes
    ├─ MemTracker::init()              // start tracking C++ allocations
    ├─ CodeCache::initialize()         // reserve memory for JIT code
    ├─ Universe::initialize_heap()     // reserve Java Heap (via GC)
    ├─ Metaspace::global_initialize()  // prepare class metadata allocator
    └─ create main JavaThread          // allocate OS stack (~1MB)
```

---

## 11. Real Java Example – Where Data Lives

```java
public class MemoryMap {
    // 'CONSTANT' lives in the Constant Pool inside Metaspace
    public static final String CONSTANT = "Hello";
    
    // 'globalList' reference lives in Metaspace (static field of InstanceKlass).
    // The actual ArrayList object lives on the Java Heap.
    public static List<String> globalList = new ArrayList<>();

    public static void main(String[] args) {
        // 'args' is a reference on the Thread Stack.
        // The String[] array itself lives on the Java Heap.
        
        // 'x' is a primitive. It lives ENTIRELY on the Thread Stack in the current Frame.
        int x = 42; 
        
        // 'obj' is a reference on the Thread Stack.
        // The Object instance lives on the Java Heap.
        Object obj = new Object();
    }
}
```

---

## 12. Why This Design? (The "Why")

### Why move from PermGen to Metaspace?
PermGen was a fixed‑sized chunk of the Java Heap. With large frameworks (Spring) or dynamic proxy generation, you’d exhaust `-XX:MaxPermSize` and get `OutOfMemoryError: PermGen space`, even if the main heap was empty. Metaspace uses native memory, so it can grow dynamically, limited only by available RAM.

### Why merge the Java Stack and Native Stack?
When Java calls C (JNI) or C calls Java, passing arguments across separate stacks is expensive. By interleaving Java frames and C frames on the same OS stack, JNI boundary crossings become simple function calls with standard C calling conventions, improving performance.

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| `-Xmx` controls the JVM’s total memory usage. | `-Xmx` **only** sets the Java Heap limit. Total memory = Heap + Metaspace + Code Cache + (Thread Count × Stack Size) + other native structures. |
| Primitives are always on the stack. | A local `int x = 5` is on the stack, but an object field `class Foo { int x; }` lives **inside the object on the heap**. |
| The `java.lang.Class` object is stored in Metaspace. | The **Java** `Class` mirror is on the heap; the **C++** `InstanceKlass` (with bytecodes, etc.) is in Metaspace. |

---

## 14. Summary

The JVM manages several distinct memory regions:

- **Java Heap** – stores dynamically allocated objects; managed by the GC.
- **Metaspace** – stores class metadata, bytecodes, and constant pools; uses native memory.
- **Code Cache** – stores JIT‑compiled machine code.
- **Thread Stacks** – standard OS stacks for local variables and method call frames.

Understanding these areas is vital for diagnosing memory issues and optimising performance.

---

## 15. Mental Model to Remember

| Region | What Lives There | Type |
| :----- | :--------------- | :--- |
| **Heap** | Java objects, arrays | Managed (GC) |
| **Metaspace** | `InstanceKlass`, ConstantPool, bytecodes | Native memory |
| **Code Cache** | JIT‑compiled assembly | Native, executable |
| **Thread Stacks** | Local variables, frames | Native, thread‑local |

---

## 16. Important Classes / Structs

- `CollectedHeap`
- `Metaspace`
- `CodeCache`
- `ReservedSpace` / `VirtualSpace`

---

## 17. Important Functions / Methods

- `Universe::initialize_heap()`
- `CodeCache::initialize()`
- `os::reserve_memory()`

---

## 18. Important Files

- `src/hotspot/share/memory/universe.cpp`
- `src/hotspot/share/memory/metaspace.hpp`
- `src/hotspot/share/code/codeCache.cpp`
- `src/hotspot/share/runtime/os.hpp`

---

## 19. Code‑Reading Exercises

1. **Heap initialisation** – open `src/hotspot/share/memory/universe.cpp` and locate `Universe::initialize_heap()`. See how it delegates the actual memory reservation to the selected GC.

2. **Code Cache allocation** – open `src/hotspot/share/code/codeCache.cpp` and search for `CodeCache::allocate()`. Notice how it takes a `CodeBlobType` to choose which segment to use.

3. **OS memory abstraction** – open `src/hotspot/share/runtime/os.hpp` and search for `reserve_memory`. This is the cross‑platform wrapper for `mmap`/`VirtualAlloc`.

---

## 20. Self‑Check Questions

1. You set `-Xmx10G` on a machine with only 8GB of RAM. The JVM may start up fine. Why does the OS allow this, and when would the JVM actually crash?

2. A Java application uses `ByteBuffer.allocateDirect()`. In which memory area does the actual byte buffer reside, and why?

3. Why must the Code Cache be marked as “executable” by the OS (using `mprotect`), whereas the Java Heap only needs “read/write” permissions?

---

## 21. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/oops/oop.hpp` | See what a Java object on the heap actually looks like. |
| `src/hotspot/share/oops/markWord.hpp` | Understand the object header (mark word). |

---

## 22. Coming Up Next

**Lesson 8 – Object Model, Object Layout & Memory Representation**  
We’ll dive into how a Java object is laid out in memory – the header, the instance fields, and how HotSpot accesses them efficiently.
