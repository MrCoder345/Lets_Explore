# OpenJDK Internals: Day 10 – JIT Compilation & Code Cache

You’ve seen the Template Interpreter in action – fast to start, but it can only process one bytecode at a time. It cannot inline methods, keep local variables in CPU registers across instructions, or eliminate dead code. For methods that run only once, the interpreter is perfect. But for “hot” methods (called thousands of times), the JVM needs to go further.

In this lesson, we’ll study **Tiered Compilation**, the **C1** and **C2** Just‑In‑Time (JIT) compilers, the **Code Cache** where compiled machine code lives, and the mind‑bending concept of **Deoptimization**. We’ll see how HotSpot smoothly transitions a method from interpreted bytecode to heavily optimised native machine code.

---

## 1. The Big Picture (Mental Model)

HotSpot uses a phased approach called **Tiered Compilation**. As a method gets hotter (called more often), it moves up through compilation tiers.

```
       Execution of a Java Method: MyClass.hotLoop()
┌────────────────────────────────────────────────────────────────────────┐
│ Tier 0: Interpreter                                                    │
│  - Executes raw bytecodes.                                             │
│  - Gathers basic profiling data (invocation count, loop backedges).    │
├────────────────────────────────────────────────────────────────────────┤
│                     [ CompileBroker threshold met ]                    │
│                                   ▼                                    │
│ Tier 1-3: C1 Compiler (Client JIT)                                     │
│  - Fast C++ compilation, low‑level optimisations.                      │
│  - Generates machine code with heavy profiling hooks (`MethodData`).   │
│  - Records branch probabilities and concrete class types at runtime.   │
├────────────────────────────────────────────────────────────────────────┤
│                     [ C1 profiling threshold met ]                     │
│                                   ▼                                    │
│ Tier 4: C2 Compiler (Server JIT / Opto)                                │
│  - Slow C++ compilation, graph‑based Intermediate Representation (IR). │
│  - Extreme optimisations: inlining, escape analysis, loop unrolling.   │
│  - Speculative execution based on C1's profiling data.                 │
└────────────────────────────────────────────────────────────────────────┘
                                   │
              (Writes machine code to Code Cache)
                                   ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        Native Code Cache                               │
│ [ nmethod (Tier 3) ]  ──▶  [ nmethod (Tier 4: highly optimised) ]      │
└────────────────────────────────────────────────────────────────────────┘
                                   │
            (What if a speculative C2 optimisation is proven wrong?)
                                   ▼
                         [ DEOPTIMIZATION ]
               (Tear down the machine code stack frame,
                reconstruct the Interpreter stack frame,
                and jump back to Tier 0 Interpreter!)
```

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | Spec says... | HotSpot does... |
| :------ | :----------- | :-------------- |
| **JIT compilation** | Nothing – a JVM can be purely interpreted. | Implements a full multi‑tier JIT system. |
| **Profiling** | Not mentioned. | Uses `MethodData` to collect runtime stats. |
| **Compiled code representation** | Not defined. | Uses `nmethod` (native method) blobs in the Code Cache. |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/compiler/` | The `CompileBroker` coordinator and generic compiler interfaces. |
| `src/hotspot/share/c1/` | The **C1** compiler (fast, lightweight). |
| `src/hotspot/share/opto/` | The **C2** compiler (slow, aggressive optimisation). |
| `src/hotspot/share/code/` | Code Cache management and `nmethod` definitions. |
| `src/hotspot/share/oops/` | `methodData.hpp` – the profiling data attached to each `Method`. |

---

## 4. Key Concepts You Need to Know

### Profiling & MethodData
When the interpreter or C1 runs a method, they update a C++ struct called `MethodData`. For example, if you have `if (x > 10)`, the profiler counts how many times the branch is taken vs. not taken. If it’s true 99.9% of the time, C2 can optimise the machine code to *assume* it’s always true, saving CPU cycles.

### nmethod (Native Method)
When a JIT compiler finishes, it outputs raw assembly. HotSpot wraps this assembly in a C++ object called `nmethod` (subclass of `CodeBlob`). It contains:

- The actual machine code instructions.
- Metadata for the GC to find object pointers inside CPU registers.
- Scope information for deoptimization.

### On‑Stack Replacement (OSR)
Normally, compilation replaces a method for the *next* invocation. But what about a `while(true)` loop inside `main()`? The method is only called once, so it never returns. OSR compiles the loop body and swaps the interpreter frame for a compiled machine‑code frame *while the loop is still running*.

---

## 5. Architecture – How Tiered Compilation Works

1. **Execution & profiling** – The interpreter runs the method, incrementing the invocation counter in the `Method` object.
2. **Compilation request** – When the counter reaches a threshold (e.g., 2,000), the interpreter submits a compilation task to the `CompileBroker`.
3. **Background compilation** – The Java thread keeps running in the interpreter. A background native `CompilerThread` picks up the task.
4. **Parsing & IR** – The JIT (C1 or C2) parses the bytecodes into an Intermediate Representation (IR) graph.
5. **Optimisation & emission** – The JIT optimises the graph, performs register allocation, and emits machine code into the Code Cache, wrapping it in an `nmethod`.
6. **Code installation** – The `Method` object’s `_code` pointer is updated to point to the new `nmethod`. The next time the method is called, execution jumps straight into the Code Cache.

---

## 6. Important C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `CompileBroker` | `compiler/compileBroker.hpp` | Manages compilation queues and dispatches tasks to compiler threads. |
| `MethodData` | `oops/methodData.hpp` | Stores runtime profiling statistics for a method. |
| `nmethod` | `code/nmethod.hpp` | Represents a compiled method in the Code Cache. |
| `Deoptimization` | `runtime/deoptimization.hpp` | Handles fallback from compiled code to the interpreter. |

---

## 7. Critical Functions

- `CompileBroker::compile_method()` – enqueues a compilation task.
- `nmethod::make()` – factory that wraps raw assembly into an executable `nmethod`.
- `Deoptimization::deoptimize_frame()` – the magic that unwinds a compiled frame and recreates an interpreter frame.

---

## 8. Important Macros / Flags

- `TieredCompilation` – the global flag (enabled by default) that enables the multi‑tier system.
- `-XX:+PrintCompilation` – logs every compilation event, showing the tier levels and method names.

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: The CompileBroker
- **Open:** `src/hotspot/share/compiler/compileBroker.cpp`
- **Look for:** `CompileBroker::compile_method_base` – check how it decides the compilation level, creates a `CompileTask`, and pushes it to a `CompileQueue`.

### Tour 2: The nmethod
- **Open:** `src/hotspot/share/code/nmethod.hpp`
- **Look at:** the class layout comment. An `nmethod` contains:
  - Header (C++ metadata).
  - Relocation information.
  - Constants.
  - **The actual machine code instructions**.
  - `oop` pointers (for GC).

### Tour 3: Deoptimization
- **Open:** `src/hotspot/share/runtime/deoptimization.cpp`
- **Look for:** `Deoptimization::deoptimize_frame`. This is one of HotSpot’s most complex functions. When a speculative optimisation fails (e.g., a null check that was omitted), this function pauses the thread, unpacks CPU registers, reconstructs an interpreter frame on the stack, and resumes execution in the interpreter.

---

## 10. Execution Flow – From Interpreter to Compiled Code

1. Interpreter runs `math.calculate()` 2,000 times.
2. The backedge counter triggers `CompileBroker::compile_method()`.
3. Task enters the queue for **Tier 3** (C1 with full profiling).
4. A `CompilerThread` picks it up; C1 compiles it.
5. C1 writes an `nmethod` to the Code Cache; `Method->_code` is updated.
6. Next call to `math.calculate()` jumps directly to the Code Cache.
7. The C1 code runs fast and continuously updates `MethodData` with branch/type statistics.
8. After 10,000 more calls, `CompileBroker` promotes it to **Tier 4** (C2).
9. C2 aggressively optimises and writes a new `nmethod`.
10. The old C1 `nmethod` is marked as “zombie” and eventually swept away.

---

## 11. Real Java Example – Watching Compilation

```java
public class JITTest {
    public static void main(String[] args) {
        for (int i = 0; i < 100_000; i++) {
            compute(i);
        }
    }

    private static int compute(int val) {
        // 0–2000: interpreted
        // 2000–10,000: C1 compiled (profiling)
        // 10,000+: C2 compiled (inlined, register‑mapped)
        return val * 2;
    }
}
```

Run with `-XX:+PrintCompilation` to see HotSpot log the compilation tiers in real‑time.

---

## 12. Why This Design? (The "Why")

**Why not compile everything with C2 immediately?**  
C2 is *slow* – it uses advanced graph‑coloring algorithms and exhaustive escape analysis. If every method were compiled with C2 at startup, the JVM would take 30 seconds just to reach `main`.

**Why can JIT be faster than static C++ compilers?**  
C++ virtual calls (`virtual void foo()`) must always go through the vtable because the compiler doesn’t know the exact class at runtime. In Java, if profiling shows that 100% of `Shape.draw()` calls are actually on a `Circle`, C2 will **devirtualize** – it skips the vtable and inlines the `Circle.draw()` assembly directly. If a `Square` ever appears, the JVM deoptimises, falls back to the interpreter, and recompiles. Static compilers cannot do this speculatively.

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| “Interpreted languages are always slow.” | Java is interpreted for a short startup phase; long‑running apps spend almost all time in highly optimised machine code. |
| `-Xcomp` (compile everything) is always faster. | `-Xcomp` forces C2 compilation without collecting runtime profiles, often producing **slower** code than tiered compilation. |

---

## 14. Summary

HotSpot’s execution engine evolves code from the Template Interpreter through C1 (fast, profiled) to C2 (slow, aggressive). The `CompileBroker` orchestrates this based on invocation thresholds. Compiled code lives in the Code Cache as `nmethod` blobs. When speculative optimisations fail, **Deoptimization** safely reverts execution to the interpreter.

---

## 15. Mental Model to Remember

```
Interpreter (Tier 0)  →  C1 (Tiers 1-3)  →  C2 (Tier 4)
     ↓                        ↓                    ↓
  profiles               fast compile      optimised code
                               ↓                    ↓
                         Code Cache (nmethod)  →  Deoptimization (safety net)
```

---

## 16. Important Classes / Structs

- `CompileBroker`
- `nmethod`
- `MethodData`
- `Deoptimization`

---

## 17. Important Functions / Methods

- `CompileBroker::compile_method()`
- `nmethod::make()`
- `Deoptimization::deoptimize_frame()`

---

## 18. Important Files

- `src/hotspot/share/compiler/compileBroker.cpp`
- `src/hotspot/share/code/nmethod.cpp`
- `src/hotspot/share/oops/methodData.hpp`
- `src/hotspot/share/runtime/deoptimization.cpp`

---

## 19. Code‑Reading Exercises

1. **CompileBroker** – open `src/hotspot/share/compiler/compileBroker.cpp` and find `CompileBroker::compile_method`. Trace where it creates a `CompileTask` and adds it to the compile queue.

2. **nmethod layout** – open `src/hotspot/share/code/nmethod.hpp` and read the class layout comment. Identify the offset fields that point to the start of the machine code (`_insts_offset`).

3. **Profiling data** – open `src/hotspot/share/oops/methodData.hpp` and scan the huge amount of tracking data. Look for `BranchData` – the structure that holds branch probabilities.

---

## 20. Self‑Check Questions

1. If a Java method is huge (e.g., 10,000 lines) and runs frequently, what limits might it hit in the JIT and Code Cache?

2. Why must an `nmethod` keep track of Java object pointers (`oop`s) embedded in its machine code (e.g., a constant String loaded into a register)? Consider what happens during GC.

3. Explain why “Deoptimization” is actually C2’s secret weapon, not a failure. What does the ability to deoptimise allow that a static C compiler cannot do?

---

## 21. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/oops/method.hpp` | See the `_code` pointer that links the method to its compiled version. |
| `src/hotspot/share/interpreter/linkResolver.cpp` | Understand how the JVM resolves which method to invoke. |

---

## 22. Coming Up Next

**Lesson 11 – Method Invocation, Dispatch & Method Handles**  
We now know how code is executed and compiled, but before that, the JVM must find *which* code to run. How does HotSpot navigate class hierarchies, interfaces, and vtables to locate the correct `Method*` in Metaspace when you call `obj.toString()`?
