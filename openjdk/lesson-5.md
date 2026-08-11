# OpenJDK Internals: Day 5 – Class Loading & Class Loader Subsystem

In C/C++, the linker resolves all symbols before the program runs. Java does this **dynamically at runtime** – the JVM doesn't know what `MyClass` looks like until it actually needs it. This is handled by the **Class Loader Subsystem**.

In this lesson, we’ll trace how a raw `.class` file on disk is found, loaded, parsed, verified, and converted into an internal C++ `InstanceKlass` in Metaspace.

---

## 1. The Big Picture (Mental Model)

Class loading is split into two halves:

- **Java half** – finds the bytes (e.g., reads from disk, network, or JARs).
- **C++ half** – parses, validates, and registers the class with the VM.

```
                  1. Java Level (Delegation & Finding Bytes)
┌────────────────────────────────────────────────────────────────────────┐
│  [ Application ClassLoader ] ─(delegates)─▶ [ Platform ClassLoader ]   │
│               │                                          │             │
│          (reads bytes)                              (delegates)        │
│               ▼                                          ▼             │
│        MyClass.class                          [ Bootstrap ClassLoader ]│
│                                                   (C++ Native)         │
└───────────────────────┬────────────────────────────────────────────────┘
                        │
                        │ 2. JNI Boundary (ClassLoader.defineClass)
                        ▼
                  3. HotSpot C++ Level (Parsing & Registration)
┌────────────────────────────────────────────────────────────────────────┐
│  [ ClassFileParser ]                                                   │
│  - Reads magic number, bytecodes, constant pool                        │
│             │                                                          │
│             ▼                                                          │
│  [ Metaspace Allocation ]                                              │
│  - Allocates C++ 'InstanceKlass'                                       │
│             │                                                          │
│             ▼                                                          │
│  [ SystemDictionary ]                                                  │
│  - Global Hash Table: (ClassName + ClassLoader) ──▶ InstanceKlass      │
└────────────────────────────────────────────────────────────────────────┘
```

### Lifecycle of a Class – Three Phases

1. **Loading** – finding the bytes and creating the `InstanceKlass`.
2. **Linking** – three sub‑steps:
   - *Verification* – ensures bytecodes are safe and legal.
   - *Preparation* – allocates memory for `static` fields.
   - *Resolution* – (optional) resolves symbolic references to direct memory addresses.
3. **Initialization** – executes the `<clinit>` method (static blocks and field initializers).

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | Spec says... | HotSpot does... |
| :------ | :----------- | :-------------- |
| **Loading phases** | Loading, Linking, Initialization. | Follows the spec exactly. |
| **Class identity** | Identified by the pair (class name, defining loader). | Uses the same tuple as the key in its internal `SystemDictionary`. |
| **Class representation** | Must provide a `java.lang.Class` mirror. | Creates a C++ `InstanceKlass` in Metaspace *and* a Java `Class` object on the heap. |
| **Delegation model** | Requires a hierarchical delegation (Bootstrap → Platform → Application). | This is implemented in Java code; HotSpot just calls back into Java to initiate loading. |

---

## 3. Important Directories & Files

| Path | Purpose |
| :--- | :--- |
| `src/java.base/share/classes/java/lang/ClassLoader.java` | Java `ClassLoader` API – `loadClass`, `defineClass`. |
| `src/hotspot/share/classfile/` | Heart of HotSpot's class loading (C++). |
| `src/hotspot/share/classfile/systemDictionary.cpp` | Global registry of loaded classes. |
| `src/hotspot/share/classfile/classFileParser.cpp` | The parser that verifies and converts byte arrays. |
| `src/hotspot/share/oops/instanceKlass.hpp` | Definition of `InstanceKlass` – the C++ class metadata. |
| `src/hotspot/share/classfile/classLoaderData.hpp` | Binds a Java `ClassLoader` to its native Metaspace memory. |

---

## 4. Key Concepts Before Reading

### Defining vs. Initiating Loader
- **Initiating loader** – the loader that was asked to load the class (e.g., `AppClassLoader`).
- **Defining loader** – the loader that actually called `defineClass` (e.g., `BootstrapClassLoader`).

In HotSpot, a class is uniquely identified by the tuple:
```
(ClassName, DefiningClassLoader)
```

### Handles in C++
HotSpot C++ code cannot hold raw `oop` pointers across operations that might trigger GC (GC moves objects). Instead, it uses **Handles** – smart pointers that the GC can update. For example, `Handle class_loader` wraps a `oop` and keeps it safe.

### Exception Handling – `TRAPS` and `CHECK`
C++ exceptions are disabled in HotSpot. Instead, most functions take a `Thread* THREAD` parameter (often hidden behind the `TRAPS` macro). If a Java exception occurs, it’s stored on the thread. The `CHECK` macro immediately returns from the C++ function if an exception is pending.

---

## 5. How the Subsystems Interact – Step by Step

1. A Java thread executes a bytecode like `new MyClass` or `getstatic`.
2. The interpreter asks the **`SystemDictionary`** if `MyClass` is already loaded for the current class loader.
3. If **not** found, the JVM calls into Java: `ClassLoader.loadClass("MyClass")`.
4. The Java `ClassLoader`:
   - Checks if the class is already loaded (`findLoadedClass`).
   - Delegates to its parent (if any).
   - If the parent fails, calls `findClass()` (which reads bytes from disk/JARs).
5. Once bytes are read, the loader calls `defineClass()` – a native method that passes the byte array to HotSpot.
6. HotSpot takes over:
   - The **`ClassFileParser`** validates the bytes (magic number, version, constant pool, etc.).
   - Allocates an **`InstanceKlass`** in **Metaspace**.
   - Registers it in the **`SystemDictionary`** with the key `(ClassName, Loader)`.
   - Creates the Java `java.lang.Class` mirror object on the Java heap.
   - If needed, runs the static initializer (`<clinit>`) lazily.

---

## 6. Key C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `SystemDictionary` | `classfile/systemDictionary.hpp` | Global hash table mapping `(name, loader)` → `InstanceKlass`. |
| `ClassFileParser` | `classfile/classFileParser.cpp` | Reads and verifies the raw `.class` bytes. |
| `InstanceKlass` | `oops/instanceKlass.hpp` | C++ representation of a loaded class (methods, fields, vtable, etc.). |
| `ClassLoaderData` (CLD) | `classfile/classLoaderData.hpp` | Ties a Java `ClassLoader` to the native Metaspace memory it owns. When the `ClassLoader` is GC’d, the CLD unloads the corresponding `InstanceKlass` metadata. |

---

## 7. Critical Functions

### Java side
- `ClassLoader.loadClass(String name)` – entry point for delegation.
- `ClassLoader.findClass(String name)` – overridden by custom loaders to locate bytes.
- `ClassLoader.defineClass(byte[] b, ...)` – native method that hands bytes to the VM.

### HotSpot C++ side
- `SystemDictionary::resolve_or_fail()` – looks up a class; if not found, triggers loading (throws `NoClassDefFoundError` if impossible).
- `ClassFileParser::parseClassFile()` – validates the byte array and builds the `InstanceKlass`.
- `InstanceKlass::initialize()` – runs the static initializer (`<clinit>`).

---

## 8. Execution Trace – `new MyClass()`

```
1. Interpreter executes 'new' bytecode.
2. Calls InterpreterRuntime::_new().
3. Calls SystemDictionary::resolve_or_fail("MyClass", current_loader).
4. Class not found → HotSpot upcalls to Java's AppClassLoader.loadClass().
5. Java reads MyClass.class from disk into a byte[].
6. Java calls defineClass() -> native method.
7. ClassFileParser verifies magic number (0xCAFEBABE), version, constant pool...
8. Allocates InstanceKlass in Metaspace.
9. Registers it in SystemDictionary.
10. Later, InstanceKlass::initialize() runs the static block.
11. Finally, allocates the instance on the Java heap.
```

---

## 9. Real Java Example – Lazy Initialization

```java
public class LoadingTest {
    static {
        System.out.println("LoadingTest initialized!");
    }

    public static void main(String[] args) throws Exception {
        System.out.println("Started.");

        // Phase 1: Loading & Linking (but NOT initialized yet)
        ClassLoader loader = LoadingTest.class.getClassLoader();
        Class<?> clazz = loader.loadClass("SomeOtherClass");

        System.out.println("Class loaded, but static block not run yet.");

        // Phase 2: Initialization (triggered by reflection, new, or static access)
        Object obj = clazz.getDeclaredConstructor().newInstance();
    }
}
```

- `loadClass()` loads the class and links it (verification + preparation) but **does not run** static initializers.
- Initialization happens only on **first active use** (e.g., `new`, reflection, `getstatic`, `invokestatic`).

---

## 10. Why This Design? (The "Why")

### ClassLoader Isolation
If a web server (like Tomcat) hosts two applications that use different versions of the same library (`com.example.Utils`), each application has its own `ClassLoader`. Because HotSpot keys the `SystemDictionary` by `(ClassName, ClassLoader)`, both versions can coexist – they are two distinct `InstanceKlass` structures. This provides secure sandboxing and dependency isolation.

### Separation of `InstanceKlass` and `java.lang.Class`
- `InstanceKlass` – C++ metadata in Metaspace (native memory, GC‑unaware).
- `java.lang.Class` – Java mirror on the heap (GC‑managed, accessible to Java code).

This separation allows HotSpot to manage metadata efficiently without polluting the heap, and allows the GC to move/collect the mirror object independently.

---

## 11. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| Two objects of class `MyClass` are identical in type, regardless of who loaded them. | If loaded by different loaders, they are **different types**. `objA.getClass() == objB.getClass()` returns `false`. |
| `static` variables live forever. | They belong to the `InstanceKlass`. When the `ClassLoader` is GC’d, the `ClassLoaderData` triggers unloading of the `InstanceKlass`, destroying the `static` variables. |
| `loadClass()` runs the static initializer. | No – `loadClass()` only loads and links. Initialization happens on first active use. |

---

## 12. Summary

- Java class loading is **dynamic and runtime‑based**.
- The **Java half** finds the bytes; the **C++ half** parses and registers them.
- The **`SystemDictionary`** is the central registry – keyed by `(ClassName, DefiningLoader)`.
- **`InstanceKlass`** is the native metadata stored in **Metaspace**; the Java `java.lang.Class` is its heap mirror.
- The delegation model prevents malicious loading of core classes (e.g., `java.lang.String`).

---

## 13. Code‑Reading Exercises

*(Try these with your cloned OpenJDK repo.)*

1. **Java delegation** – Open `src/java.base/share/classes/java/lang/ClassLoader.java`. Find `loadClass()` – where is the synchronization that prevents two threads from loading the same class simultaneously?
2. **The registry** – Open `src/hotspot/share/classfile/systemDictionary.hpp`. Find `resolve_or_null`. What type does it return? (Hint: it’s a pointer to class metadata.)
3. **Magic number** – Open `src/hotspot/share/classfile/classFileParser.cpp`. Search for `0xCAFEBABE` – see how the parser rejects any file that doesn’t start with this magic number.

---

## 14. Self‑Check Questions

1. A malicious user creates a class named `java.lang.String` and tries to load it with a custom `ClassLoader`. How does the delegation model prevent this from hijacking the real `String`?

2. Why does HotSpot keep the C++ `InstanceKlass` in native Metaspace instead of storing it as a regular Java object on the heap?

3. What happens in the `SystemDictionary` when a custom `ClassLoader` becomes unreachable and is garbage collected?

---

## 15. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/oops/instanceKlass.hpp` | See exactly what metadata a loaded class holds. |
| `src/hotspot/share/interpreter/bytecode.hpp` | Prepare to read the bytecode instructions inside methods. |

---

## 16. Coming Up Next

**Lesson 6 – Bytecode, .class File Format & Verification**  
Now that we know *how* bytes enter the JVM, we’ll look at what’s *inside* the `.class` file – the bytecode instruction set, the constant pool, and the rigorous verification HotSpot performs to keep the VM safe.
