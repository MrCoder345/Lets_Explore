# OpenJDK Internals: Day 12 – invokedynamic, Lambdas & Dynamic Language Support

In the previous lesson, we learned that standard method invocation (`invokevirtual`, `invokestatic`) requires the JVM to know the exact class and method signature at compile time.

But what if you are running Ruby (JRuby) or JavaScript (Nashorn) on the JVM? In Ruby, `a.add(b)` could mean adding integers, concatenating strings, or invoking a custom method on a dynamically generated object. The type of `a` isn't known until runtime. Before Java 7, dynamically typed languages had to use heavy Reflection or generate thousands of tiny `.class` files on the fly to simulate dynamic dispatch, causing massive performance bottlenecks.

To solve this, the JVM introduced its first new bytecode in decades: **`invokedynamic`**. It effectively says to the JVM: *"I don't know what method to call yet. When you hit this instruction for the first time, call a piece of user‑defined Java code to figure it out, and then wire this instruction directly to that method for all future calls."*

In Java 8, language designers realised `invokedynamic` was the perfect mechanism to implement **Lambdas** without polluting the JVM with thousands of anonymous inner `.class` files.

---

## 1. The Big Picture (Mental Model)

The lifecycle of an `invokedynamic` instruction has two phases: **Unlinked** (first execution) and **Linked** (subsequent executions).

```
       Phase 1: First Execution (Unlinked)
┌────────────────────────────────────────────────────────────────────────┐
│ Bytecode: invokedynamic #123 (Needs a method to execute!)              │
└───────────────────────┬────────────────────────────────────────────────┘
                        │ (JVM pauses execution, looks up Bootstrap Method)
                        ▼
┌────────────────────────────────────────────────────────────────────────┐
│ Bootstrap Method (BSM) - User-written Java code (e.g., LambdaMetafactory)
│  - Receives the call site name, signature, and context.                │
│  - Executes arbitrary logic (e.g., dynamically generating a class).    │
│  - Returns a `CallSite` object containing a `MethodHandle`.            │
└───────────────────────┬────────────────────────────────────────────────┘
                        │ (JVM wires the bytecode to the CallSite)
                        ▼
       Phase 2: Subsequent Executions (Linked)
┌────────────────────────────────────────────────────────────────────────┐
│ Bytecode: invokedynamic #123                                           │
│                       │                                                │
│                       ▼ (Direct, JIT-optimizable jump)                 │
│ [ CallSite ] ──▶ [ MethodHandle ] ──▶ Target Method (Machine Code)     │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | What the Spec says | How HotSpot does it |
| :------ | :----------------- | :------------------ |
| **Bytecode** | Defines `invokedynamic`. | Implements it in the interpreter and JIT. |
| **Bootstrap Method (BSM)** | Must be present in the class file; returns a `CallSite`. | Calls the BSM via JNI on first execution. |
| **Linking** | The instruction is permanently linked to the `CallSite`. | Stores the `CallSite` in the `ConstantPoolCacheEntry` as an "appendix". |
| **Optimisation** | Not specified. | JIT aggressively inlines through `CallSite` and `MethodHandle` chains, removing overhead. |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/interpreter/` | The interpreter runtime for `invokedynamic`. |
| `src/hotspot/share/prims/` | Native engine for dynamic invocation (`methodHandles.cpp`). |
| `src/java.base/share/classes/java/lang/invoke/` | Java API for `CallSite`, `MethodHandle`, `LambdaMetafactory`. |

---

## 4. Key Concepts You Need to Know

### Bootstrap Method (BSM)
A standard static Java method. It is pointed to by the `.class` file's Constant Pool. The JVM calls it **exactly once** per `invokedynamic` instruction. Its job is to return a `CallSite` that provides the actual target method.

### CallSite
A Java object that holds a `MethodHandle`. There are several types:

| CallSite Type | Behaviour |
| :------------ | :-------- |
| `ConstantCallSite` | The target never changes – used by Lambdas. |
| `MutableCallSite` | The target can be changed at runtime – useful for dynamic languages when a variable’s type changes. |
| `VolatileCallSite` | Like `MutableCallSite`, but with volatile memory semantics. |

---

## 5. Architecture – How the JVM Links the Instruction

1. **Interpreter Execution** – The C++ interpreter hits the `invokedynamic` bytecode.
2. **Cache Check** – It looks at the `ConstantPoolCacheEntry` for this specific index. If the `_f1` field (the function pointer) is `null`, the instruction is **unlinked**.
3. **C++ Fallback** – The interpreter jumps to `InterpreterRuntime::resolve_invokedynamic()`.
4. **Upcall to Java** – HotSpot finds the BSM in the constant pool and calls it via JNI.
5. **Java Logic** – The BSM executes. For a lambda, `LambdaMetafactory` generates a tiny class in memory that implements the functional interface, and returns a `ConstantCallSite` pointing to its constructor.
6. **Linking** – HotSpot takes the `CallSite` from Java, extracts the underlying native method pointer, and stores it in the `ConstantPoolCacheEntry`.
7. **Resumption** – The interpreter reads the newly linked cache entry and executes the `MethodHandle`.

---

## 6. Important Classes & Structs

### Java (public API)
- `java.lang.invoke.LambdaMetafactory` – the standard BSM for all Java Lambdas.
- `java.lang.invoke.CallSite` / `ConstantCallSite` – the linkage target.
- `java.lang.invoke.MethodHandle` – a direct reference to a method.

### C++ (HotSpot internals)
- `ConstantPoolCacheEntry` – a C++ struct representing a resolved bytecode instruction. It holds the "appendix" (a reference to the `CallSite` or `MethodHandle` object).
- `CallInfo` – a C++ struct used by `LinkResolver` to hold the results of the BSM invocation.

---

## 7. Critical Functions

### Java side
- `LambdaMetafactory.metafactory(...)` – the method that dynamically generates the lambda implementation.

### HotSpot C++ side
- `LinkResolver::resolve_invokedynamic()` – orchestrates calling the BSM and linking the instruction.
- `SystemDictionary::find_bootstrap_method()` – locates the BSM in the class file.

---

## 8. Source Code Exploration (Guided Tour)

### Tour 1: The Interpreter Runtime Fallback
- **Open:** `src/hotspot/share/interpreter/interpreterRuntime.cpp`
- **Find:** `InterpreterRuntime::resolve_invokedynamic(JavaThread*)`. This is where the assembly interpreter lands when it hits an unlinked instruction. It delegates to `LinkResolver`.

### Tour 2: The Link Resolver's Dynamic Path
- **Open:** `src/hotspot/share/interpreter/linkResolver.cpp`
- **Look for:** `LinkResolver::resolve_invokedynamic()`.
- **Follow the logic:** It calls `SystemDictionary::invoke_bootstrap_method()` – a massive JNI upcall from C++ into the Java BSM. After the Java code returns the `CallSite`, it extracts the `MethodHandle` and stores it in the `ConstantPoolCacheEntry`.

### Tour 3: The Lambda Metafactory (Java)
- **Open:** `src/java.base/share/classes/java/lang/invoke/LambdaMetafactory.java`
- **Read the Javadoc** – it explains what `javac` emits and how this method generates the implementation class at runtime. It delegates to `InnerClassLambdaMetafactory`, which uses ASM (a bytecode generation library embedded in the JDK) to write a new `.class` directly into memory.

---

## 9. Execution Flow – Tracing a Lambda

Consider this Java code:

```java
Runnable r = () -> System.out.println("Hello");
r.run();
```

### Compile time (`javac`)
1. `javac` moves the lambda body (`System.out.println("Hello")`) into a hidden static method inside your class, e.g., `private static void lambda$main$0()`.
2. `javac` replaces the lambda expression with an `invokedynamic` instruction pointing to `LambdaMetafactory.metafactory`.

### Runtime – First execution
3. The interpreter hits `invokedynamic` – it is **unlinked**.
4. HotSpot calls `LambdaMetafactory.metafactory()` via JNI.
5. `LambdaMetafactory` dynamically generates a byte array for a new class, e.g., `MyClass$$Lambda$1`, that implements `Runnable`. Its `run()` method simply calls `MyClass.lambda$main$0()`.
6. It returns a `ConstantCallSite` pointing to the constructor of `MyClass$$Lambda$1`.
7. HotSpot links the `invokedynamic` instruction to this constructor.
8. The instruction executes, returning a new instance of the generated lambda class.

### Subsequent executions
9. The next time the loop hits the lambda creation, the `invokedynamic` instruction is already linked. It skips the BSM, reads the cached constructor, and instantly returns a new lambda instance (or the same one if no variables are captured).

---

## 10. Real Java Example – Dump Generated Classes

Run this with `-Djdk.internal.lambda.dumpProxyClasses=.` to see the generated class files on disk:

```java
import java.util.function.Supplier;

public class LambdaTest {
    public static void main(String[] args) {
        String captured = "World";
        Supplier<String> sup = () -> "Hello " + captured;  // invokedynamic
        System.out.println(sup.get());
    }
}
```

If you inspect the dumped `.class` file, you'll see a generated class implementing `Supplier`, with a field for the `captured` string, and a `get()` method that routes to the hidden static method in `LambdaTest`.

---

## 11. Why This Design? (The "Why")

**Why use `invokedynamic` for Lambdas instead of anonymous inner classes?**

If Java 8 had used the old way (anonymous inner classes), `javac` would have generated a `LambdaTest$1.class` file on disk for *every* lambda in your codebase. A modern application with streams could generate thousands of `.class` files. This would bloat JAR sizes, slow down class loading, and increase Metaspace memory usage drastically.

With `invokedynamic`, the JAR remains small. The JVM decides *how* to implement the lambda at runtime. If the JVM developers later find a faster way to implement lambdas without generating a class, they can simply update `LambdaMetafactory`, and *all existing compiled Java 8 code* will instantly become faster – without recompilation!

---

## 12. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| `invokedynamic` is just another form of Reflection. | **No.** Reflection (`Method.invoke`) happens at the Java layer, checks security on every call, and is hard to inline. `invokedynamic` is a low‑level bytecode; once linked to a `MethodHandle`, the JIT treats it like a normal static/virtual call and can inline it aggressively. It is **much faster** than Reflection. |
| Executing a lambda uses `invokedynamic`. | **Not exactly.** *Creating* the lambda object uses `invokedynamic`. But when you call `r.run()`, that is just a standard `invokeinterface` call on the generated object. |

---

## 13. Summary

`invokedynamic` allows the JVM to defer method resolution to user‑defined Java code (the Bootstrap Method). Once the BSM returns a `CallSite` containing a `MethodHandle`, HotSpot links the instruction permanently. This provides the ultimate bridge between dynamic language semantics and the highly optimised HotSpot JIT. Java uses this mechanism via `LambdaMetafactory` to implement Lambdas dynamically at runtime, avoiding class file explosion.

---

## 14. Mental Model to Remember

```
Bytecode (invokedynamic)
       │
       ▼ (first time)
C++ cache miss → Java BSM (e.g., LambdaMetafactory)
       │
       ▼
Returns CallSite → C++ links cache entry
       │
       ▼ (subsequent calls)
Fast JIT execution (direct MethodHandle call)
```

---

## 15. Important Classes / Structs

- `ConstantPoolCacheEntry` (C++)
- `CallSite` / `ConstantCallSite` (Java)
- `MethodHandle` (Java)
- `LambdaMetafactory` (Java)

---

## 16. Important Functions / Methods

- `LinkResolver::resolve_invokedynamic()` (C++)
- `LambdaMetafactory.metafactory()` (Java)

---

## 17. Important Files

- `src/hotspot/share/interpreter/linkResolver.cpp`
- `src/hotspot/share/interpreter/interpreterRuntime.cpp`
- `src/java.base/share/classes/java/lang/invoke/LambdaMetafactory.java`

---

## 18. Code‑Reading Exercises

1. **Read the LambdaMetafactory Javadoc** – open `src/java.base/share/classes/java/lang/invoke/LambdaMetafactory.java`. The class‑level documentation is one of the best in the JDK and explains the translation strategy in detail.

2. **Trace the C++ resolution path** – open `src/hotspot/share/interpreter/linkResolver.cpp` and find `resolve_invokedynamic`. Follow it down to where it calls `SystemDictionary::invoke_bootstrap_method()`.

3. **Examine the cache entry** – open `src/hotspot/share/oops/cpCache.hpp` and look at the `ConstantPoolCacheEntry` struct. Read the comments about the `_f1` and `_f2` fields to see how HotSpot stores the `CallSite` appendix natively.

---

## 19. Self‑Check Questions

1. A dynamically typed language changes the type of a variable inside a loop (e.g., `x` becomes an integer, then a string). It cannot use a `ConstantCallSite`. Which `CallSite` class should its Bootstrap Method return, and how does it update the target when the type changes?

2. What is the architectural advantage of deferring lambda class generation to runtime (`LambdaMetafactory`) versus generating `.class` files at compile time?

3. When the JIT compiler (C2) encounters an already‑linked `invokedynamic`, why can it inline the target method just as easily as an `invokestatic` call?

---

## 20. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/gc/shared/collectedHeap.hpp` | Transitioning from execution back to memory management. |
| `src/hotspot/share/gc/shared/threadLocalAllocBuffer.hpp` | How objects are quickly allocated in the TLAB. |

---

## 21. Coming Up Next

**Lesson 13 – Heap Organization & Memory Management**  
We have now conquered class loading, memory layouts, bytecode, the interpreter, the JIT, and dynamic dispatch. We understand how execution works. Now, we must understand the lifecycle of the data we create. If we just allocate memory forever, we will eventually crash. It is time to dive into the Garbage Collector.
