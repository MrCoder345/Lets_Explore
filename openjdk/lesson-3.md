# OpenJDK Internals: Day 3 – Java Launcher & JVM Startup

You have a compiled `.class` file. The OS (Linux, Windows, macOS) has **no idea** what a `.class` file is – it only knows native binaries. So how does your Java program actually start?

In this lesson, we trace the exact path from the moment the OS creates a new C process to the moment the JVM invokes your `main()` method. You'll learn that the `java` command is just a thin C wrapper that loads HotSpot as a shared library and embeds it.

---

## 1. The Big Picture (Mental Model)

Think of the JVM not as a standalone executable, but as a **heavy C++ library** (`libjvm.so` or `jvm.dll`) that can be loaded into any native process.

```
  OS Shell (e.g., bash)
         │
         ▼ fork() & execve()
  ┌─────────────────────────────────────────────────────────────┐
  │ 'java' executable (JLI – Java Launcher Infrastructure)     │  (C Code)
  │  • Parses: -Xmx, -cp, MainClass                            │
  │  • Finds libjvm.so on disk                                 │
  │                     │                                      │
  │                     ▼ dlopen() / LoadLibrary()             │
  │             [ libjvm.so ]                                  │
  │                     │                                      │
  │                     ▼                                      │
  │             JNI_CreateJavaVM()                             │  (C++ Code)
  │                     │                                      │
  │                     ▼                                      │
  │             Threads::create_vm()                           │
  │             (Initializes GC, Threads, Compiler)            │
  │                     │                                      │
  │                     ▼ JNI Call                             │
  │         MainClass.main(String[] args)                      │  (Java Code)
  └─────────────────────────────────────────────────────────────┘
```

---

## 2. JVM Specification vs. HotSpot (Startup Context)

| Aspect | What it says |
| :--- | :--- |
| **JVM Specification** | Requires that the JVM starts by loading an initial class and invoking `public static void main(String[])`. It also defines the shutdown behaviour (VM exits when only daemon threads remain). |
| **HotSpot Implementation** | The spec doesn't mention the `java` executable, flags like `-Xmx`, or `libjvm.so`. The entire launcher infrastructure is an OpenJDK engineering solution to fulfil the spec's startup requirements. |

---

## 3. Where the Code Lives (Directories & Files)

| Path | Purpose |
| :--- | :--- |
| `src/java.base/share/native/libjli/` | The **Java Launcher Infrastructure** – pure C code for the `java` command. |
| `src/hotspot/share/prims/` | JNI interfaces – the bridge from launcher to HotSpot (C++). |
| `src/hotspot/share/runtime/` | VM initialization, threading, and runtime core (C++). |
| **Key files** | `java.c` (launcher), `jni.cpp` (JNI entry points), `threads.cpp` (VM bootstrap). |

---

## 4. Essential Concept: The JNI Invocation API

You already know JNI lets Java call C. But there's also an **Invocation API** that lets a C program **create a JVM inside its own process**.  
The C program:

- Populates a `JavaVMInitArgs` struct with flags.
- Loads `libjvm.so`.
- Finds the `JNI_CreateJavaVM` function pointer.
- Calls it to spawn the VM.

This is exactly what the `java` launcher does.

---

## 5. High-Level Architecture (The 6 Steps)

1. **OS Process Creation** – You type `java MyProgram`; the OS starts the `java` C binary.
2. **Argument Parsing** – The launcher separates JVM flags (`-Xmx`, `-XX:`) from application arguments (`arg1`, `arg2`).
3. **Library Loading** – The launcher dynamically loads `libjvm.so` (using `dlopen` or `LoadLibrary`).
4. **VM Bootstrapping** – It calls `JNI_CreateJavaVM`; HotSpot allocates the heap, initialises the GC, creates the main thread, etc.
5. **Java Execution** – Using JNI, the launcher finds your `MainClass` and invokes its `main` method.
6. **Teardown** – The launcher waits for the main thread to finish, then calls `DestroyJavaVM`.

---

## 6. Key Data Structures (C/C++)

| Struct | Role |
| :--- | :--- |
| `JavaVMInitArgs` | C struct that holds all JVM options (passed from launcher to HotSpot). |
| `JavaVM` | Represents the VM instance; contains function pointers for destroying the VM or attaching threads. |
| `JNIEnv` | Thread‑local structure with pointers to JNI functions (`FindClass`, `CallStaticVoidMethod`, etc.). |

---

## 7. Critical Functions – The Call Chain

### Launcher (C code)

- **`JLI_Launch()`** – the main coordinator in `java.c`. Parses args, finds `libjvm.so`, sets up a new thread to run `JavaMain()`.
- **`LoadJavaVM()`** – wraps `dlopen` / `dlsym` to locate `JNI_CreateJavaVM`.
- **`JavaMain()`** – the thread routine that actually invokes `JNI_CreateJavaVM` and later calls `main()`.

### HotSpot (C++ code)

- **`JNI_CreateJavaVM()`** – the exported C‑linkage function in `jni.cpp`. Delegates immediately to `Threads::create_vm()`.
- **`Threads::create_vm()`** – the **Big Bang** of HotSpot. Initialises OS layers, memory managers, the interpreter/JIT, and creates the main `JavaThread`.

---

## 8. Under the Hood: Dynamic Linking

The launcher uses platform-specific APIs:

- On Linux/macOS: `dlopen("libjvm.so", RTLD_LAZY)` + `dlsym(handle, "JNI_CreateJavaVM")`.
- On Windows: `LoadLibrary("jvm.dll")` + `GetProcAddress(handle, "JNI_CreateJavaVM")`.

This runtime linking means the `java` binary is tiny and can work with any HotSpot version.

---

## 9. Full Execution Trace (Line by Line)

```text
$ ./java -Xmx1G MainApp arg1

  1. OS loads the C binary: /usr/bin/java
  2. main() → JLI_Launch() (java.c)
  3. JLI_Launch() parses "-Xmx1G" and "MainApp"
  4. LoadJavaVM() finds libjvm.so and gets JNI_CreateJavaVM address
  5. Launcher creates a new native thread and runs JavaMain() on it
  6. JavaMain() calls JNI_CreateJavaVM(jni.cpp)
  7. JNI_CreateJavaVM() calls Threads::create_vm() (threads.cpp)
  8. Threads::create_vm():
       - os::init()          (OS-specific setup)
       - universe_init()     (heap allocation, GC selection)
       - interpreter_init()  (bytecode interpreter)
       - JIT initialization (C1/C2)
       - creates the main JavaThread
  9. Back to JavaMain():
       - env->FindClass("MainApp")
       - env->GetStaticMethodID("main", "([Ljava/lang/String;)V")
       - env->CallStaticVoidMethod(... "arg1")
  10. Bytecode execution starts.
  11. When main returns, JavaMain() waits and calls jvm->DestroyJavaVM().
```

---

## 10. Real C Example – Embedding a JVM

If you write a simple C program like this:

```c
#include <jni.h>

int main() {
    JavaVMInitArgs vm_args;
    JavaVM *jvm;
    JNIEnv *env;

    vm_args.version = JNI_VERSION_1_8;
    // set -Xmx etc. in vm_args.options
    JNI_CreateJavaVM(&jvm, (void**)&env, &vm_args);

    jclass cls = env->FindClass("MyJavaClass");
    jmethodID mid = env->GetStaticMethodID(cls, "main", "([Ljava/lang/String;)V");
    env->CallStaticVoidMethod(cls, mid, NULL);

    jvm->DestroyJavaVM();
    return 0;
}
```

This is **exactly** what the `java` launcher does – just with more error handling and argument parsing.

---

## 11. Why This Design? (The "Why")

**Why not compile HotSpot directly into the `java` executable?**

- **Embeddability** – Other applications (databases, game engines, web servers) can load `libjvm.so` and run Java code inside their own process.
- **Upgradability** – You can swap `libjvm.so` without changing the launcher (e.g., use a newer GC).
- **Separation of concerns** – The launcher handles OS‑specific process setup; the VM handles execution.

The `java` command is just the **reference embedder**; you can write your own if you need custom behaviour.

---

## 12. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :--- | :--- |
| Believing the `java` command is written in Java. | It’s pure C (in `libjli`). |
| Thinking `java` and HotSpot are statically linked. | They are dynamically linked at runtime via `dlopen`. |
| Assuming the JVM parses `-Xmx` or `-classpath`. | The C launcher parses them and passes them as a struct to `JNI_CreateJavaVM`. |
| Thinking the main OS thread runs Java code. | The launcher typically creates a **new** C thread (`JavaMain`) to host the VM. |

---

## 13. Summary

- The **Java launcher** (`java`) is a native C program.
- It dynamically loads the **HotSpot shared library** (`libjvm.so`).
- It uses the **JNI Invocation API** (`JNI_CreateJavaVM`) to bootstrap the VM inside the same process.
- The VM initialises its subsystems (heap, GC, threads, compilers) and creates the main Java thread.
- The launcher then uses standard JNI calls to find and invoke your `main()` method.

---

## 14. Code-Reading Exercises

*(Try these with your cloned OpenJDK.)*

1. **Find the launcher entry point**  
   Open `src/java.base/share/native/libjli/java.c` and locate `JLI_Launch`. Follow it to where it calls `JavaMain` – notice how it spawns a new thread.

2. **See the crossing from C to C++**  
   Open `src/hotspot/share/prims/jni.cpp` and find `JNI_CreateJavaVM`. Observe how it converts the C `JavaVMInitArgs` into internal HotSpot flags before calling `Threads::create_vm`.

3. **Tour the VM bootstrap**  
   Open `src/hotspot/share/runtime/threads.cpp` and skim `Threads::create_vm()`. List the major subsystems you see being initialised (e.g., GC, interpreter, JIT, OS layer).

---

## 15. Self-Check Questions

1. You type `java -Xmxq 1G Main`. Which component – the C launcher or the C++ JVM – detects the invalid `-Xmxq` flag, and how is the error reported back to the user?

2. Why does the launcher create a **new** C thread (`JavaMain`) for the VM instead of running everything on the main OS thread? *(Hint: think about stack sizes and platform‑specific thread primitives.)*

3. You want to write a C++ game engine that uses Java for scripting. How would you start the Java runtime inside your game process?

---

## 16. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/runtime/vmOperations.hpp` | See how the VM manages internal operations once it's running. |
| `src/hotspot/share/oops/oop.hpp` | Get a first look at how Java objects are represented in C++ (the `oop` type). |

---

## 17. Coming Up Next

**Lesson 4 – JVM Architecture & HotSpot Overview**  
Now that we know how the VM starts, we'll build a complete internal map of HotSpot – its memory layout, execution engine, threading model, and core subsystems – before diving deeper into specific areas like class loading and garbage collection.
