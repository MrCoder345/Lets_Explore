# OpenJDK Internals: Day 1 – Navigating the Repository

Welcome to your first day on the OpenJDK team.

Before we dive into pointers or garbage collection, we need to learn how to **find things**. OpenJDK is a massive repository written in C, C++, Java, and Assembly. Without a roadmap, you'll waste hours looking for `javac` inside the VM or hunting for `java.lang.String` in the wrong folder.

This guide will give you a mental map of the codebase, break down the folder structure, and teach you how to search effectively using `rg` (ripgrep) and `git`.

---

## 1. The Big Picture (Mental Model)

Forget the phrase *"Java is the JVM"*. They are two separate components that just happen to live in the same repo.

```
                          ┌─────────────────────────────────────────────┐
                          │          OpenJDK Repository                │
                          └─────────────────────────────────────────────┘
                                          │
         ┌────────────────────────────────┼────────────────────────────────┐
         │                                │                                │
   ┌─────┴─────┐                  ┌───────┴───────┐                ┌───────┴───────┐
   │  HotSpot  │                  │  JDK Libraries│                │     Tools     │
   │ (C/C++)   │                  │  (Java + C)   │                │  (Java/C++)   │
   └───────────┘                  └───────────────┘                └───────────────┘
   │ Interpreter │                │  java.base    │                │  javac        │
   │ JIT Compiler│                │  java.xml     │                │  javadoc      │
   │ GC (G1/ZGC) │                │  java.desktop │                │  jcmd / jstat │
   │ Thread Mgmt │                └───────────────┘                │  jlink        │
   └─────────────┘                                                       │
         │                         JNI Bridge                           │
         └───────────────────────────────────────────────────────────────┘
                                         │
                                 Operating System
                               (Linux / Windows / macOS)
```

- **HotSpot** → The Virtual Machine. Written in C/C++.  
  It knows about **bytecodes**, **object headers**, **heap memory**, and **threads**.  
  *It does NOT know what an `ArrayList` is.*

- **JDK Libraries** → The Java Standard Library (`java.*`, `javax.*`).  
  Mostly Java, but uses native C code (via JNI) to talk to the OS.

- **Tools** → Command-line utilities like `javac` (written in Java!), `javadoc`, and `jcmd`.

---

## 2. JVM Specification vs. HotSpot Implementation

| Concept | What it is |
| :--- | :--- |
| **JVM Specification** | A **document** that defines *what* a valid `.class` file looks like, the exact behavior of every bytecode (e.g., `iadd` adds two ints), and the conceptual memory areas (Heap, Method Area). |
| **HotSpot Implementation** | OpenJDK's **actual C++ code** that implements that spec. The spec doesn't say "you must have an Eden space" – that's a HotSpot design choice to make things fast. |

---

## 3. Repository Structure (The Important Directories)

| Path | What lives here? |
| :--- | :--- |
| `src/hotspot/` | The **JVM** itself (C/C++). |
| `src/java.base/` | The **core** Java libraries (`java.lang`, `java.io`, `java.nio`). (Java + native C code). |
| `src/jdk.compiler/` | The `javac` compiler (written in **Java**). |
| `make/` | Build system files (GNU Make). |
| `test/` | Massive test suites (jtreg, gtest). |

---

## 4. The Module System (Project Jigsaw)

Since Java 9, OpenJDK is modular. The `src/` folder is split into **modules**.  
The most important module is **`java.base`** – every other module depends on it.

**Key pattern:** `src/<module_name>/<OS>/<directory>`

OpenJDK explicitly separates:

- **Shared** code (platform-independent)
- **OS-specific** code (Linux, Windows, macOS)
- **CPU-specific** code (x86, ARM, RISC-V)

---

## 5. How the Architecture Fits Together

```
1. Java Code       →  calls library methods (e.g., `FileInputStream.read()`)
2. JDK Libraries   →  execute Java logic until they hit a `native` method
3. JNI             →  bridges the Java world to the C world
4. HotSpot / Native→  executes C/C++ code to make the actual OS system call
```

---

## 6. Key Directories to Know (Your "Structs")

| Directory | Contents |
| :--- | :--- |
| `src/hotspot/share/` | ~80% of the JVM. Platform-independent C++ (Garbage Collectors, JIT, Interpreter). |
| `src/hotspot/os/linux/` | Linux-specific threading, memory, and signals. |
| `src/hotspot/cpu/x86/` | x86-64 assembly generation and CPU-specific stubs. |

---

## 7. The Single Most Important Entry Point

- **`JNI_CreateJavaVM()`** – The C function that bootstraps HotSpot.  
  This is where the `java` launcher becomes a living, breathing Virtual Machine.

---

## 8. Platform Abstractions (Macros & Templates)

HotSpot uses heavy macro-based code generation.  
Example: You'll see `os::malloc` in the shared code.

- Declaration: `src/hotspot/share/runtime/os.hpp`
- Definition: Conditionally compiled from:
  - `src/hotspot/os/linux/os_linux.cpp`
  - `src/hotspot/os/windows/os_windows.cpp`

---

## 9. How to Read the Source Code (Navigation Pro-Tips)

**Don't read randomly.** Use this map:

| What you want to see | Where to go |
| :--- | :--- |
| How `String` works | `src/java.base/share/classes/java/lang/String.java` |
| How the JVM allocates an object | `src/hotspot/share/gc/shared/` or `src/hotspot/share/oops/` |
| `javac` parsing syntax | `src/jdk.compiler/share/classes/com/sun/tools/javac/` |
| C code backing `System.currentTimeMillis()` | `src/java.base/share/native/libjava/System.c` *(outside HotSpot!)* |

**Search like a pro with `rg` (ripgrep):**

```bash
# Find a Java class
rg "class String " src/java.base

# Find a HotSpot C++ class
rg "class Universe" src/hotspot
```

---

## 10. Execution Flow: Running `java HelloWorld`

```text
1. Launcher starts    →  The C program `java` parses command-line args.
2. VM loads           →  Dynamically loads `libjvm.so` (or `jvm.dll`).
3. VM initializes     →  Calls `JNI_CreateJavaVM()` to boot HotSpot.
4. Class loading      →  Loads `HelloWorld.class` from disk.
5. Execution          →  Finds `main()` and hands it to the Interpreter/JIT.
```

---

## 11. Real Example: `System.out.println("Hello");`

```text
1. System.out is a PrintStream (JDK Library).
2. It calls FileOutputStream.writeBytes() (a native Java method).
3. JVM maps this to a C function in io_util.c.
4. The C code makes an OS write() syscall.
```

---

## 12. Why This Design? (Separation of Concerns)

If IBM, Azul, or Oracle want to build a different JVM (like OpenJ9 or GraalVM), they **only** need to implement the JVM Spec. They can reuse:

- The entire `java.base` Java library
- The `javac` compiler

Decoupling the runtime (HotSpot) from the standard libraries ensures **vendor portability**.

---

## 13. Common Beginner Mistakes (Don't Do This)

| Mistake | Reality |
| :--- | :--- |
| Looking for `javac` inside HotSpot. | `javac` is written in Java. It lives in `src/jdk.compiler`. |
| Thinking HotSpot runs `.java` files. | HotSpot only understands `.class` (bytecode) files. |
| Assuming all C/C++ belongs to HotSpot. | A huge amount of C code lives in `src/java.base/.../native` for OS-level I/O, networking, and compression. |

---

## 14. Summary (The Rule of Thumb)

> **Ask yourself constantly:** *"Does this logic belong to the Java language, the OS bridge, or the VM engine?"*

- **HotSpot** (C++) runs **Bytecode**.
- **Bytecode** is generated by **javac** (Java).
- **javac** relies on **java.base** (Java + C) to handle OS interactions.

---

## 15. Code-Reading Exercises

*(Assuming you have OpenJDK cloned locally on your Linux machine.)*

1. **Find the Java Thread vs. the VM Thread**  
   ```bash
   rg "class Thread " src/java.base
   rg "class JavaThread" src/hotspot
   ```
   Notice the difference between the Java-level representation and the internal C++ one.

2. **Explore OS Support**  
   ```bash
   ls src/hotspot/os/
   ```
   See how many operating systems HotSpot supports natively.

3. **Trace `System.nanoTime()`**  
   ```bash
   rg "nanoTime" src/hotspot/share/prims/
   ```
   Find the JNI bridge that backs this native method.

---

## 16. Self-Check Questions

1. You find a bug in `HashMap`'s hash calculation.  
   **Do you go to `src/hotspot/` or `src/java.base/`? Why?**

2. Why does OpenJDK separate `src/hotspot/share/` from `src/hotspot/os/` and `src/hotspot/cpu/`?  
   How does this help a developer writing a new Garbage Collector?

3. When you run `javac MyProgram.java`, does HotSpot run?  
   Explain the relationship between the compiler and the VM.

---

## 17. What to Read Next

| File | Why |
| :--- | :--- |
| `make/Main.gmk` | See how the monolithic build system ties C++ and Java compilation together. |
| `doc/building.md` | The official guide on how to build OpenJDK from source. |
