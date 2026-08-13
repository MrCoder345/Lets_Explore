# OpenJDK Internals: Day 11 – Method Invocation, Dispatch & Method Handles

You understand how objects are laid out in memory and how the JIT compiles bytecode into machine code. But how does the JVM know *which* machine code to execute?

In C++, virtual dispatch is handled by the compiler – it embeds a hidden `vptr` in each object that points to a `vtable`. When you call `obj->foo()`, the CPU executes a predictable memory offset jump.

Java is dynamically linked and heavily polymorphic. Classes are loaded at runtime, and interfaces can be implemented in any order. The JVM must resolve method calls on the fly, build its own vtables and itables in Metaspace, and aggressively optimise these lookups so that virtual method dispatch doesn't destroy performance.

In this lesson, we'll dissect method invocation: the `invoke` bytecodes, the C++ `LinkResolver`, HotSpot's custom vtable/itable memory layouts, Inline Caches (ICs), and the foundation of modern dynamic dispatch – Method Handles.

---

## 1. The Big Picture (Mental Model)

Method invocation is split into **Resolution** (finding the target method conceptually) and **Dispatch** (executing the target method at runtime).

```
       1. The Call Site (Bytecode)
┌────────────────────────────────────────────────────────────────────────┐
│  invokevirtual #14  // Calls Shape.draw()                              │
└────────────────────────────────────────────────────────────────────────┘
            │
            ▼  (First Execution: Resolution via C++ LinkResolver)
┌────────────────────────────────────────────────────────────────────────┐
│  [ Constant Pool ]  ──▶  [ LinkResolver ]                              │
│  Symbol: "draw:()V"      Checks access rules, looks up Method*         │
│                          Returns: vtable index (e.g., index 5)         │
└────────────────────────────────────────────────────────────────────────┘
            │
            ▼  (Every Execution: Dispatch via vtable)
┌────────────────────────────────────────────────────────────────────────┐
│  [ oop (Java Object) ]                                                 │
│   └── _mark                                                            │
│   └── _klass  ─────────▶ [ InstanceKlass (Metaspace) ]                 │
│                          │ ...                                         │
│                          │ [ vtable ] (Array appended to InstanceKlass)│
│                          │  [0] Object.clone()                         │
│                          │  ...                                        │
│                          │  [5] Circle.draw()  ──▶ [ Method* ]         │
└────────────────────────────────────────────────────────────────────────┘
            │
            ▼  (JIT Optimization: Inline Cache)
┌────────────────────────────────────────────────────────────────────────┐
│  Machine Code (Compiled Code Cache)                                    │
│  [ CMP RAX, <Circle_Klass_Ptr> ]  // Is it still a Circle?             │
│  [ JNE slow_path               ]  // If not, fallback to vtable        │
│  [ JMP Circle_draw_nmethod     ]  // Fast direct jump!                 │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | Spec says... | HotSpot does... |
| :------ | :----------- | :-------------- |
| **Invoke bytecodes** | Defines 5: `invokestatic`, `invokespecial`, `invokevirtual`, `invokeinterface`, `invokedynamic`. | Implements all of them. |
| **Dispatch mechanism** | Not specified – only the semantics. | Uses `klassVtable` (classes) and `klassItable` (interfaces). |
| **Optimisation** | Not mentioned. | Uses **Inline Caches** in the Code Cache to bypass vtable lookups for ~95% of calls. |
| **Method Handles** | Not part of the spec (it's a Java API). | Provides a native backend (`methodHandles.cpp`) that the JIT can inline and optimise. |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/interpreter/linkResolver.cpp` | The JVM-spec-compliant method resolution logic. |
| `src/hotspot/share/oops/klassVtable.cpp` | C++ implementation of vtables and itables. |
| `src/hotspot/share/code/compiledIC.cpp` | Inline Caches – self-modifying dispatch optimisation. |
| `src/hotspot/share/prims/methodHandles.cpp` | Native backend for `java.lang.invoke`. |

---

## 4. Key Concepts You Need to Know

### vtable vs. itable

| Table | Used For | Characteristics |
| :---- | :------- | :-------------- |
| **vtable** (Virtual Table) | `invokevirtual` – class inheritance. | Single inheritance; layout is fixed. `Object.toString()` is at the same index in every class. Lookup is O(1) array access. |
| **itable** (Interface Table) | `invokeinterface` – interface dispatch. | Multiple inheritance; order varies per class. Must search for the interface ID first, then the method offset. O(N) – slower than vtable. |

### Inline Caches (ICs)
Memory reads (fetching the vtable, then the method pointer) are slow. HotSpot self‑modifies its machine code. At a call site, the JIT emits a check:

> *"The last time we were here, the object was a `String`. Is it a `String` again?"*

If yes, it jumps directly to `String`'s compiled method. This is a **Monomorphic Inline Cache**. If it sees two types, it becomes **Bimorphic**. If it sees many types, it becomes **Megamorphic** and falls back to the vtable.

---

## 5. Architecture – How Method Invocation Works

1. **Preparation (Class Loading)** – When a class is loaded, `ClassFileParser` calculates the size of its vtable and itable. The `InstanceKlass` in Metaspace has these tables appended directly to its memory block for data locality.

2. **Resolution (First Call)** – The interpreter hits `invokevirtual`. It calls `LinkResolver::resolve_invoke()`, which checks visibility (public/private), finds the target `Method*`, and looks up its vtable index.

3. **Interpreter Dispatch** – The interpreter caches the vtable index in the **Constant Pool Cache**. Subsequent interpreted calls just read the index and perform `klass->vtable()[index]`.

4. **JIT Dispatch** – The JIT compiler emits an **Inline Cache**. At runtime, the machine code checks the object's `Klass*`. If it matches the cached type, it branches directly to the `nmethod`.

---

## 6. Important C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `LinkResolver` | `interpreter/linkResolver.hpp` | Implements the strict JVM spec for resolving symbolic references. |
| `klassVtable` / `vtableEntry` | `oops/klassVtable.hpp` | The virtual table – an array of `Method*` pointers. |
| `klassItable` / `itableOffsetEntry` | `oops/klassItable.hpp` | Interface dispatch – pairs of `(interface, method_offset)`. |
| `CompiledIC` | `code/compiledIC.hpp` | C++ representation of an Inline Cache in the Code Cache. |
| `MethodHandle` (Java) | `java/lang/invoke/MethodHandle.java` | A strongly typed, directly executable reference to a method. |

---

## 7. Critical Functions

- `LinkResolver::resolve_virtual_call()` – resolves a standard instance method call.
- `klassVtable::initialize_vtable()` – populates the vtable during class loading, overriding parent entries with child methods.
- `MethodHandles::resolve_MemberName()` – the native function that resolves a Java `MethodHandle` to an internal `Method*`.

---

## 8. Important Macros / Utilities

- **`CallInfo`** – a C++ struct populated by `LinkResolver` that holds the resolved `Method*`, the vtable index, and the call kind.

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: The LinkResolver (Spec Logic)
- **Open:** `src/hotspot/share/interpreter/linkResolver.cpp`
- **Look for:** `LinkResolver::resolve_virtual_call`. Notice how it checks access permissions (throws `IllegalAccessError` if invalid) and then determines the vtable index.

### Tour 2: The vtable Memory Layout
- **Open:** `src/hotspot/share/oops/klassVtable.hpp`
- **Look for:** the `method_at(int i)` function. Notice that it performs simple pointer arithmetic – the vtable memory is physically appended to the `InstanceKlass` in Metaspace.

### Tour 3: Method Handles Native Bridge
- **Open:** `src/hotspot/share/prims/methodHandles.cpp`
- **Look for:** `MethodHandles::resolve_MemberName`. When Java code looks up a `MethodHandle`, it calls into this C++ function to securely resolve the underlying `Method*`.

---

## 10. Execution Flow – `invokeinterface List.add()`

1. Interpreter hits `invokeinterface`.
2. `LinkResolver::resolve_interface_call()` is called. It verifies the receiver implements `List`.
3. The interpreter looks at the target object's `InstanceKlass`.
4. It iterates through the `klassItable`, searching for the `List` interface pointer.
5. It finds `List`, reads the offset for `add()`, and fetches the `Method*` (e.g., `ArrayList.add()`).
6. It invokes the method.
7. **Later, in JIT code:** An Inline Cache is emitted. If the call is always made on an `ArrayList`, the JIT skips the itable search entirely. It compares the object's `Klass*` to `ArrayList`'s `Klass*` and jumps directly to `ArrayList.add()`'s compiled code.

---

## 11. Real Java Example – Method Handles vs. Reflection

```java
import java.lang.invoke.*;

public class Dispatch {
    public static void main(String[] args) throws Throwable {
        // 1. invokevirtual (vtable lookup, optimised by ICs)
        Object obj = "Hello";
        System.out.println(obj.toString());

        // 2. MethodHandle – direct, JIT-optimised pointer
        MethodHandles.Lookup lookup = MethodHandles.lookup();
        MethodType type = MethodType.methodType(String.class);
        MethodHandle mh = lookup.findVirtual(Object.class, "toString", type);

        // 3. invokeExact – intrinsic, same performance as invokevirtual
        String result = (String) mh.invokeExact(obj);
    }
}
```

- Reflection (`Method.invoke`) boxes arguments into an `Object[]` and checks security on every call.
- `MethodHandle.invokeExact` is an **intrinsic** – the JIT compiles it down to the exact same machine code as a normal `invokevirtual`.

---

## 12. Why This Design? (The "Why")

### Why not just use C++ virtual functions for Java objects?
HotSpot is written in C++, but the C++ `virtual` keyword belongs to the C++ class (`InstanceKlass`). Java classes are loaded dynamically at runtime. HotSpot must build a completely custom, dynamic vtable system in Metaspace to simulate object‑oriented behaviour for the Java code running on top.

### Why Method Handles?
Before Java 7, dynamically typed languages (like JRuby or Groovy) on the JVM had a hard time. They couldn't use `invokevirtual` because the type wasn't known at compile time, and Reflection was too slow. Method Handles provided a safe, JIT‑optimisable way to say: *"Here is a pointer to a method. I'll provide the arguments at runtime – please compile it as fast as a normal method call."*

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| "Interfaces are just as fast as abstract classes." | In the interpreter or at megamorphic call sites, `invokeinterface` (itable scan) is slower than `invokevirtual` (direct vtable index). However, with a successful Inline Cache, both are equally fast. |
| "Reflection is fine for performance-critical loops." | Reflection is extremely slow – boxing, argument arrays, security checks. Use `MethodHandle` instead. |

---

## 14. Summary

Method invocation in HotSpot has two phases: **Resolution** (ensuring the method exists and is accessible, via `LinkResolver`) and **Dispatch** (actually executing the method). Dispatch uses custom `klassVtable` and `klassItable` structures in Metaspace. To overcome memory latency, the JIT uses **Inline Caches** – self‑modifying machine code that memorises the target method based on the object's class. **Method Handles** provide a modern, JIT‑optimisable way to refer to methods dynamically, used heavily by Lambdas and invokedynamic.

---

## 15. Mental Model to Remember

```
Resolution:  LinkResolver (slow, done once)
Dispatch:    vtable (O(1)) or itable (O(N) search)
Optimisation: Inline Cache (direct JMP if Klass* matches)
Modern API:  MethodHandle (JIT‑optimisable function pointer)
```

---

## 16. Important Classes / Structs

- `LinkResolver`
- `klassVtable` / `klassItable`
- `CompiledIC`
- `MethodHandle`

---

## 17. Important Functions / Methods

- `LinkResolver::resolve_invoke()`
- `klassVtable::method_at()`
- `MethodHandles::resolve_MemberName()`

---

## 18. Important Files

- `src/hotspot/share/interpreter/linkResolver.cpp`
- `src/hotspot/share/oops/klassVtable.cpp`
- `src/hotspot/share/code/compiledIC.cpp`
- `src/hotspot/share/prims/methodHandles.cpp`

---

## 19. Code‑Reading Exercises

1. **LinkResolver logic** – open `src/hotspot/share/interpreter/linkResolver.cpp` and find `LinkResolver::resolve_virtual_call`. Follow the steps: interface check, access check, and finally populating the `CallInfo` struct.

2. **vtable access** – open `src/hotspot/share/oops/klassVtable.hpp` and look at `method_at(int i)`. See how it uses pointer arithmetic to read the `Method*` from the vtable array.

3. **Inline Cache states** – open `src/hotspot/share/code/compiledIC.hpp` and read the comments explaining Monomorphic, Bimorphic, and Megamorphic states.

---

## 20. Self‑Check Questions

1. Why is `invokestatic` inherently faster for the JVM than `invokevirtual`? (Think about vtables and the `oop` receiver.)

2. An Inline Cache checks if the `oop`'s `Klass*` matches a known type. If you have `animal.speak()` and you pass a `Dog`, a `Cat`, and a `Bird` repeatedly in a loop, what happens to the Inline Cache, and what must the JIT do?

3. Why must HotSpot implement its own `klassVtable` in Metaspace instead of using C++ `virtual` functions?

---

## 21. What to Read Next

| File | Why |
| :--- | :--- |
| `src/java.base/share/classes/java/lang/invoke/LambdaMetafactory.java` | The Java side of Lambda creation. |
| `src/java.base/share/classes/java/lang/invoke/CallSite.java` | The anchor for `invokedynamic` call sites. |

---

## 22. Coming Up Next

**Lesson 12 – invokedynamic, Lambdas & Dynamic Language Support**  
Method Handles gave us a way to point to methods dynamically. In the next lesson, we'll see how Java 7 introduced a new bytecode, `invokedynamic`, to support dynamic languages – and how Java 8 cleverly hijacked it to implement Lambdas without generating thousands of anonymous inner classes.
