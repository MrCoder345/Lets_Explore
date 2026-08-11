# OpenJDK Internals: Day 4 – JVM Architecture & HotSpot Overview

You know how the launcher boots the JVM. Now, we're standing *inside* HotSpot. Before diving into any single component, you need a high-level map of the entire JVM. Without this mental map, the C++ source code will look like a chaotic web of interdependent singletons.

In this lesson, we'll build a conceptual map of HotSpot's core subsystems: **Class Loading**, **Runtime Data Areas**, **Execution Engine**, and **Runtime Services**. We'll also introduce the HotSpot-specific terminology you'll see everywhere in the code, like `oop`, `Klass`, and `CodeCache`.

---

## 1. The 10,000-Foot View (Mental Model)

Burn this architecture map into your memory:

```
┌────────────────────────────────────────────────────────────────────────────┐
│                          HotSpot JVM Architecture                          │
│                                                                            │
│  ┌────────────────────┐      ┌──────────────────────────────────────────┐  │
│  │ 1. Class Loader    │      │ 2. Runtime Data Areas                    │  │
│  │    Subsystem       │      │                                          │  │
│  │ ────────────────── │      │  [ Java Heap ]     [ Code Cache ]        │  │
│  │ - Loading          │─────▶│  (Java objects)    (JIT machine code)    │  │
│  │ - Linking          │      │                                          │  │
│  │ - Initialization   │      │  [ Metaspace ]     [ Thread Stacks ]     │  │
│  └────────────────────┘      │  (Class metadata)  (Frames / Locals)     │  │
│                               └──────────────────────────────────────────┘  │
│                                                  │                         │
│  ┌───────────────────────────────────────────────▼──────────────────────┐  │
│  │ 3. Execution Engine                                                  │  │
│  │ ──────────────────────────────────────────────────────────────────── │  │
│  │  [ Template Interpreter ] ──────▶ [ Profiling / Counters ]           │  │
│  │  (Executes raw bytecode)                    │                        │  │
│  │                                             ▼                        │  │
│  │  [ C2 JIT Compiler ] ◀─────────── [ C1 JIT Compiler ]               │  │
│  │  (Heavy optimizations)            (Fast compilation)                 │  │
│  └───────────────────────────────────────────────┬──────────────────────┘  │
│                                                  │                         │
│  ┌───────────────────────────────────────────────▼──────────────────────┐  │
│  │ 4. Runtime Services & GC                                             │  │
│  │ ──────────────────────────────────────────────────────────────────── │  │
│  │  [ Garbage Collector ]    [ Safepoints & Thread Coordination ]       │  │
│  │  [ JNI / JVMTI ]          [ Synchronization / Monitors ]             │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | Spec says... | HotSpot does... |
| :--- | :--- | :--- |
| **Method Area** | Holds class metadata. | Implements it as **Metaspace** (native C++ memory, not Java heap). |
| **Execution Engine** | Must execute bytecode. | Uses **Mixed-Mode**: an optimized Assembly Interpreter + JIT compilers (C1/C2). |
| **Compiled Code** | No mention. | Stores JIT-compiled machine code in a specialized native region called the **Code Cache**. |

---

## 3. Where the Code Lives (Directories)

HotSpot's modularity maps directly to its C++ directories in `src/hotspot/share/`:

| Directory | What lives here |
| :--- | :--- |
| `classfile/` | Class parsing, verification, and loading. |
| `oops/` | Object representation (Object-Oriented Pointers). The C++ structs for Java objects and classes. |
| `interpreter/` | The Template Interpreter (bytecode execution). |
| `c1/` & `opto/` | The C1 compiler (fast) and C2 / Opto compiler (heavy optimization). |
| `gc/` | Garbage collection implementations (G1, ZGC, Shenandoah, Serial, Parallel). |
| `runtime/` | Thread management, safepoints, synchronization, and JNI. |
| `memory/` | Heap allocation and Metaspace management. |

---

## 4. Essential Concepts You Must Know

### Mixed-Mode Execution
Java compiles to bytecode (`.class` files), not native machine code.  
When a method is called:

1. HotSpot starts by **interpreting** bytecode (slow start, but instant).
2. While interpreting, it **counts** how many times the method runs or loops.
3. When a threshold is crossed (e.g., 10,000 calls), the method is handed to a **JIT compiler** to be turned into optimized machine code *on the fly*.
4. Future calls jump directly to the **Code Cache** (fast execution).

### oop (Ordinary Object Pointer)
In HotSpot C++ code, you will constantly see the type `oop`.  
This is HotSpot's internal C++ representation of a **pointer to a Java object** sitting on the garbage-collected heap. It is *not* a raw `void*` – it has a specific header structure.

---

## 5. How the Subsystems Interact

Here is the step-by-step interaction flow:

1. **Class Loading** → Reads a `.class` file, verifies bytecode, creates C++ `Klass` structures in **Metaspace**.
2. **Allocation** → When Java calls `new`, HotSpot asks the **Garbage Collector** for memory in the **Java Heap** and initialises an `oop`.
3. **Execution** → The **Thread** pushes a stack frame. The **Interpreter** reads the method's bytecode and executes it.
4. **Compilation** → If a method is "hot" (e.g., 10,000 iterations), the **Runtime** submits a compile task. The **JIT Compiler** translates bytecode to assembly and stores it in the **Code Cache**.
5. **Safepoints** → Periodically, the JVM must stop all Java threads (e.g., to run a GC). The **Runtime Services** orchestrate this global pause.

---

## 6. Key Classes / Structs (C++)

| Class / Struct | File | Role |
| :--- | :--- | :--- |
| `Universe` | `memory/universe.hpp` | The **global root** of the VM. Holds pointers to the heap, the system dictionary, and globally shared objects. |
| `CollectedHeap` | `gc/shared/collectedHeap.hpp` | The abstract base class for **all garbage collectors**. |
| `JavaThread` | `runtime/javaThread.hpp` | Represents a running Java thread. Contains OS thread handle, Java stack, and JNI environment. |
| `oopDesc` | `oops/oop.hpp` | The C++ struct defining the **header** of every single Java object. |
| `Klass` | `oops/klass.hpp` | C++ struct representing **class metadata** (vtable, field layouts) in Metaspace. |

---

## 7. Critical Functions (C++)

- **`Universe::initialize_heap()`** – Called during VM startup. Asks the selected GC to allocate the Java heap.
- **`CompileBroker::compile_method()`** – The bridge between the interpreter and the JIT compilers. Submits a method for background compilation.
- **`Threads::create_vm()`** – The big bang initialisation of all the above (covered in the previous lesson).

---

## 8. Important Macros / Flags

HotSpot uses a globally accessible flags system defined in `globals.hpp`.

| Flag | Effect |
| :--- | :--- |
| `UseG1GC` | Enable G1 garbage collector. |
| `UseZGC` | Enable ZGC (low-latency). |
| `TieredCompilation` | Use both C1 (fast) and C2 (optimising) JIT compilers. |
| `MaxHeapSize` | Maximum Java heap size (maps to `-Xmx`). |

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: The Global State Holder – `Universe`
- **Open:** `src/hotspot/share/memory/universe.hpp` and `.cpp`
- **What to find:**
  - `static CollectedHeap* _collectedHeap;` → The global pointer to the Java heap. At runtime, this points to a `G1CollectedHeap` or `ZCollectedHeap`.
  - `Universe::initialize()` → Notice the order: initializes GC → Code Cache → Metaspace → core Java classes (like `java.lang.Object`).

### Tour 2: The Flags System
- **Open:** `src/hotspot/share/runtime/globals.hpp`
- **What to find:**
  - Search for `UseG1GC` or `MaxHeapSize`.
  - Notice the heavy macro magic (e.g., `product(bool, UseG1GC, true, ...)`) that defines the type, name, default value, and documentation all in one line.

---

## 10. Full Execution Flow (Lifecycle of a VM)

```text
1. Launcher calls Threads::create_vm()
2. Command-line flags are parsed (populating globals.hpp variables)
3. Universe::initialize() is called
4. The configured GC allocates the CollectedHeap
5. ClassLoader loads java.lang.Object and java.lang.Class into Metaspace
6. The CompileBroker starts JIT compiler threads
7. The main thread transitions to execute Java bytecode via the Interpreter
```

---

## 11. Real Java Example – What Happens Under the Hood

```java
public class ArchitectureTest {
    public static void main(String[] args) {
        // 1. ClassLoader loads 'ArchitectureTest' into Metaspace.
        // 2. Interpreter begins executing main().

        while (true) {
            // 3. 'new' requests memory from CollectedHeap.
            //    Returns an 'oop' pointing to the Java Heap.
            Object obj = new Object();

            // 4. HotSpot profiler notices this loop is "hot".
            // 5. C1/C2 compiles main() into the Code Cache.
            // 6. Execution jumps from Interpreter to Code Cache.
        }
    }
}
```

---

## 12. Why This Design? (The "Why")

**Why not just compile everything ahead-of-time (AOT) like C++?**

- **Dynamic Class Loading** – Java can load classes at runtime (e.g., from the network). AOT compilation requires knowing all classes upfront.
- **Platform Independence** – You ship bytecode. The JIT generates machine code *specific to your exact CPU* (e.g., AVX-512 instructions) and runtime profile (e.g., which branch is taken 99% of the time).
- **Adaptive Optimisation** – HotSpot recompiles methods if they get hotter, or de-optimises if assumptions change. Static compilation cannot do that.

> The JIT compiler often generates **faster** native code than static C++ compilers because it knows the *actual* runtime behaviour, not just the source code.

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :--- | :--- |
| Thinking "The Heap" contains everything. | The **Java Heap** only contains objects and arrays. **Metaspace** holds class definitions (native memory). **Code Cache** holds machine code (native memory). Thread stacks are OS-native memory. |
| Confusing `Klass` with `java.lang.Class`. | `Klass` is a C++ struct in **Metaspace** representing class metadata. `java.lang.Class` is a **Java object** on the heap that acts as a mirror/wrapper exposing that metadata to Java code. |
| Assuming the interpreter is slow and useless. | The interpreter provides **instant startup** and gathers **profiling data** that guides the JIT to make better optimisations. It's not useless—it's a strategic data gatherer. |

---

## 14. Summary

HotSpot is a highly modular C++ application divided into:

1. **Class Loading** (Metaspace for metadata).
2. **Execution** (Interpreter + JIT compilers + Code Cache).
3. **Memory Management** (Java Heap + Garbage Collectors).
4. **Runtime Services** (Threads, Safepoints, Synchronisation).

`Universe` holds the global state. `globals.hpp` configures everything. The interpreter runs bytecode; the JIT compiles hot parts into native code; the GC keeps the heap clean.

---

## 15. Code-Reading Exercises

*(Try these with your cloned OpenJDK repo.)*

1. **Explore the flags** – Open `src/hotspot/share/runtime/globals.hpp`. Find `MaxHeapSize`. What is its default value (look at the macro arguments)?
2. **Find the heap pointer** – Open `src/hotspot/share/memory/universe.hpp`. Find the declaration for the global `CollectedHeap`. What is its exact type?
3. **Understand `oop`** – Open `src/hotspot/share/oops/oopsHierarchy.hpp`. Look at how `oop` is typedef'd. It's fundamentally just a pointer to a C++ class (`oopDesc`).

---

## 16. Self-Check Questions

1. An application dynamically creates 10,000 new classes at runtime (e.g., using proxies or bytecode generation). Which native memory area will grow the most – the Java Heap, Metaspace, or Code Cache?

2. Why does HotSpot invest engineering effort into building an Interpreter when it already has highly optimising JIT compilers? What is the primary trade-off?

3. A Java program throws `OutOfMemoryError: Java heap space`. Looking at our architecture map, which C++ subsystem failed to fulfil a memory request, and where was that request likely initiated (which subsystem called `new`)?

---

## 17. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/classfile/classLoader.cpp` | The entry point for bringing `.class` files into the VM. |
| `src/hotspot/share/classfile/systemDictionary.hpp` | Where HotSpot keeps track of all loaded classes (a global registry). |

---

\
