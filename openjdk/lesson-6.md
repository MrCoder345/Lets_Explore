# OpenJDK Internals: Day 6 – Bytecode, .class File Format & Verification

You have seen how the `ClassLoader` finds a `.class` file and hands it to HotSpot. Now we ask: **what exactly is being handed over?**

In C/C++, source code is compiled into OS‑specific binaries (ELF, PE, Mach‑O) containing machine code and symbol tables. In Java, source code is compiled into a **universally portable binary**: the `.class` file. It contains **bytecode** – a hardware‑agnostic instruction set for a stack‑based virtual machine.

Because `.class` files can be downloaded from untrusted sources, HotSpot **cannot** just execute them blindly. If a bytecode instruction tries to treat an integer as an object pointer, it would break C++ memory safety and crash the VM. Therefore, HotSpot performs **rigorous bytecode verification** before any code is executed.

In this lesson, we’ll dissect the `.class` file format, see how it maps to internal C++ structures (`ConstantPool`, `Method`), and explore the Bytecode Verifier.

---

## 1. The Big Picture (Mental Model)

Think of a `.class` file as a highly compact, self‑contained binary structure.

```
┌────────────────────────────────────────────────────────────┐
│                     The .class File                        │
├────────────────────────────────────────────────────────────┤
│ Magic Number (0xCAFEBABE)                                  │
│ Version (e.g., 65.0 for Java 21)                           │
├────────────────────────────────────────────────────────────┤
│ Constant Pool  (like ".rodata" + symbol table)             │
│  [1] Methodref: java/lang/Object."<init>":()V              │
│  [2] String: "Hello World"                                 │
│  [3] Class: java/lang/String                               │
├────────────────────────────────────────────────────────────┤
│ Class Hierarchy Info (access flags, this_class, super)     │
├────────────────────────────────────────────────────────────┤
│ Fields Table (name, type, access flags)                    │
├────────────────────────────────────────────────────────────┤
│ Methods Table                                              │
│  └─ Method: "main"                                         │
│      └─ Code Attribute                                     │
│          ├─ Max Stack: 2, Max Locals: 1                    │
│          ├─ Bytecodes: [0x12, 0x02, 0xB6, 0x01, 0xB1]     │
│          └─ StackMapTable (for fast verification)          │
└────────────────────────────────────────────────────────────┘
                            │
                            ▼ (Parsed into C++ by HotSpot)
┌────────────────────────────────────────────────────────────┐
│  InstanceKlass (Metaspace)                                 │
│   ├── ConstantPool* _constants                             │
│   ├── Array<Method*>* _methods ──▶ Method (C++)            │
│   └── Array<u2>* _fields            └─ u1* _code           │
└────────────────────────────────────────────────────────────┘
```

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | What the Spec Says | How HotSpot Does It |
| :------ | :----------------- | :------------------ |
| **File format** | Chapter 4 of the JVM Spec strictly defines the binary layout. | `ClassFileParser` implements the spec exactly. |
| **Bytecode set** | Defines the opcodes (e.g., `aload_0` is `0x2A`). | `bytecodes.hpp` maps opcodes to C++ enums. |
| **Verification rules** | Defines type‑inference rules to prove code safety. | `ClassVerifier` implements these rules, using the `StackMapTable` for fast validation. |
| **Internal representation** | No specification. | HotSpot uses `ConstantPool`, `Method`, and `InstanceKlass` to hold the parsed data. |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/classfile/` | Parsing (`classFileParser.cpp`) and verification (`verifier.cpp`). |
| `src/hotspot/share/oops/` | Metadata representation: `constantPool.hpp`, `method.hpp`, `instanceKlass.hpp`. |
| `src/hotspot/share/interpreter/` | Bytecode definitions (`bytecodes.hpp`). |

---

## 4. Key Concepts You Need to Know

### The Constant Pool
In C, if you call `printf`, the compiler leaves a relocation entry, and the dynamic linker fixes the address at runtime. In Java, when you call `System.out.println()`, the bytecode instruction (`invokevirtual`) **does not contain** the string `"println"`. Instead it contains a 2‑byte index (e.g., `#14`) pointing to an entry in the **Constant Pool**. The Constant Pool is an array of symbols, type names, method signatures, and other constants.

### Stack Machine vs. Register Machine
x86 is a **register machine** – instructions operate on CPU registers like `RAX`, `RBX`. The JVM is a **stack machine** – bytecodes push values onto an operand stack, operate on the top elements, and push the result back.

For example, `x = 1 + 2` in bytecode becomes:
```
iconst_1     ; push 1
iconst_2     ; push 2
iadd         ; pop two ints, add, push result
istore_1     ; pop result, store into local variable 1
```

This stack‑based design makes bytecode compact and portable.

---

## 5. How the Subsystems Interact

1. **Stream Creation** – `ClassLoader` passes a raw `byte[]` to HotSpot; HotSpot wraps it in a `ClassFileStream`.
2. **Parsing** – `ClassFileParser` reads the stream in order:
   - Magic, version, constant pool, class hierarchy, fields, methods.
   - For methods, it extracts the `Code` attribute containing the bytecode array.
3. **Verification** – Before linking, the `ClassVerifier` performs a linear sweep of the bytecodes in every `Method`. It simulates execution at the **type level** (not the value level) to ensure:
   - Stack depth is never violated.
   - Type transitions are valid (e.g., you don’t push an `int` and then pop it as an `oop`).
4. **Registration** – If verified, the `InstanceKlass` is finalized and stored in the `SystemDictionary`.

---

## 6. Important C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `ClassFileStream` | `classfile/classFileParser.hpp` | A safe wrapper around a byte buffer with read methods for `u1`, `u2`, `u4` (big‑endian). |
| `ConstantPool` | `oops/constantPool.hpp` | An array of resolved and unresolved constants for a class, stored in Metaspace. |
| `Method` | `oops/method.hpp` | Represents a Java method; contains a pointer to `ConstMethod` (which holds the bytecode) and runtime profiling data. |
| `ClassVerifier` | `classfile/verifier.cpp` | The engine that validates bytecodes, using the `StackMapTable` for fast type checking. |

---

## 7. Critical Functions

- `ClassFileParser::parse_constant_pool()` – reads the constant pool from the stream.
- `ClassFileParser::parse_methods()` – parses the methods and their `Code` attributes (including bytecodes).
- `Verifier::verify()` – static entry point for verification; delegates to `ClassVerifier::verify_method()`.
- `ClassVerifier::verify_method()` – simulates execution type‑by‑type to prove safety.

---

## 8. Important Macros / Utilities

- **`CHECK` / `CHECK_NULL`** – heavily used in `classFileParser.cpp`. If a malformed class file throws a `ClassFormatError`, these macros propagate the exception up the stack.
- **`Bytes::get_Java_u2()`** – utility to read big‑endian data (`.class` files are always big‑endian, regardless of the host CPU).

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: Bytecode Definitions
- **Open:** `src/hotspot/share/interpreter/bytecodes.hpp`
- **Look for:** the `enum Code`. You will see the entire JVM instruction set mapped to C++ enums (e.g., `_aload_0 = 42`, `_iadd = 96`). This is the “alphabet” of the JVM.

### Tour 2: Parsing the File
- **Open:** `src/hotspot/share/classfile/classFileParser.cpp`
- **Look for:** `parse_stream()`. Notice the order:
  ```cpp
  u4 magic = stream->get_u4();      // 0xCAFEBABE
  u2 minor = stream->get_u2();
  u2 major = stream->get_u2();
  ```
- **Look for:** `parse_constant_pool()`. It loops over the constant pool count, reads a tag (e.g., `JVM_CONSTANT_String`), and instantiates the appropriate entry.

### Tour 3: Verification
- **Open:** `src/hotspot/share/classfile/verifier.cpp`
- **Look for:** `ClassVerifier::verify_method()`.
- **Overview:** Modern Java uses a **split verifier**. `javac` pre‑computes the type state at every jump target and stores it in a `StackMapTable` attribute. HotSpot then sweeps through the bytecodes and simply **checks** that `javac`’s math is correct, rather than inferring types from scratch (which was the old approach).

---

## 10. Execution Flow – Full Trace

```
1. SystemDictionary requests parsing for a class.
2. ClassFileParser reads the magic number (0xCAFEBABE).
3. parse_constant_pool() extracts all symbolic constants.
4. parse_methods() creates Method C++ objects, copying the Code attribute (bytecodes) into a u1* array.
5. Verifier::verify() is invoked.
6. ClassVerifier reads the StackMapTable.
7. It steps through every bytecode:
   - For iadd, it checks: "Are there two ints on the simulated stack?" – if yes, it pops two ints and pushes one int.
   - For areturn (return object), it verifies the simulated stack has an object compatible with the return type.
8. If any check fails, it throws a VerifyError.
9. If successful, the InstanceKlass is linked and ready for use.
```

---

## 11. Real Java Example – Using `javap`

Compile this simple class:

```java
public class MyClass {
    public int add(int a, int b) {
        return a + b;
    }
}
```

Then run `javap -v -c MyClass`. The output shows exactly what `ClassFileParser` reads:

```
Constant pool:
   #1 = Methodref          #2.#3          // java/lang/Object."<init>":()V
   #2 = Class              #4             // java/lang/Object
   ...

  public int add(int, int);
    Code:
       0: iload_1
       1: iload_2
       2: iadd
       3: ireturn
```

Notice that `iload_1` and `iload_2` are single‑byte instructions – they don’t need an operand because they refer to fixed local variable slots.

---

## 12. Why This Design? (The "Why")

### Why bytecode instead of AST or machine code?
- **Abstract Syntax Tree (AST)** – too large and slow to transmit.
- **Machine code** – platform‑specific and cannot be shipped universally.
- **Bytecode** – the perfect middle ground: highly compact (many instructions are just 1 byte), hardware‑agnostic, and easy to JIT‑compile later.

### Why verification?
In C, if you write:
```c
int* p = (int*) 0xDEADBEEF;
*p = 5;
```
the compiler lets you, and the OS kills you with a segmentation fault. The JVM **guarantees memory safety**. Without the Verifier, a malicious `.class` file could manually craft bytecodes to cast a standard object into a raw pointer and hijack the HotSpot C++ process. The Verifier mathematically proves that the bytecode cannot violate type safety before a single instruction runs.

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| Thinking `javac` heavily optimizes code. | `javac` does **almost zero optimization** – it translates Java syntax literally to bytecode. The HotSpot JIT compiler does 99% of the optimisation at runtime. |
| Confusing the `.class` Constant Pool with the Java String Pool. | The **Constant Pool** is static binary data describing class symbols. The **String Pool** is a dynamic hash table on the Java heap holding `java.lang.String` objects created at runtime. |

---

## 14. Summary

- A `.class` file is a rigid binary contract containing a **Constant Pool** (symbols), structural metadata, and **Bytecode** instructions.
- HotSpot translates this binary blob into C++ `ConstantPool` and `Method` structures via `ClassFileParser`.
- The `ClassVerifier` then ensures that the bytecodes are type‑safe, protecting the C++ JVM from illegal memory access.

---

## 15. Mental Model to Remember

```
byte[]  →  ClassFileParser  →  C++ ConstantPool + Method (bytecodes)
                                   ↓
                            ClassVerifier (Type Checking)
                                   ↓
                         Executable Metadata (InstanceKlass)
```

---

## 16. Important Classes / Structs

- `ClassFileStream`
- `ClassFileParser`
- `ConstantPool`
- `Method`
- `ClassVerifier`

---

## 17. Important Functions / Methods

- `ClassFileParser::parse_constant_pool()`
- `ClassFileParser::parse_methods()`
- `Verifier::verify()`

---

## 18. Important Files

- `src/hotspot/share/interpreter/bytecodes.hpp`
- `src/hotspot/share/classfile/classFileParser.cpp`
- `src/hotspot/share/classfile/verifier.cpp`
- `src/hotspot/share/oops/constantPool.hpp`

---

## 19. Code‑Reading Exercises

1. **Bytecode enum** – open `src/hotspot/share/interpreter/bytecodes.hpp` and find the enum value for `_invokevirtual` (the bytecode for standard method dispatch).

2. **Parsing the constant pool** – open `src/hotspot/share/classfile/classFileParser.cpp` and search for `parse_constant_pool`. Look at the `switch (tag)` block to see how HotSpot handles different pool entries like `JVM_CONSTANT_String` and `JVM_CONSTANT_Methodref`.

3. **The Method object** – open `src/hotspot/share/oops/method.hpp` and find the field that stores the pointer to the `ConstMethod` (the holder of the actual bytecode array).

---

## 20. Self‑Check Questions

1. If you write a Java program and compile it, `javac` already checks all the types. Why does HotSpot waste time verifying the bytecodes again at runtime?

2. Why does the JVM use a stack‑based instruction set (push, pop, add) instead of a register‑based one like x86 or Dalvik? Think about portability and compactness.

3. A bytecode instruction is `getfield #4`. What exactly is `#4`, and where does HotSpot look to figure out which field is being accessed?

---

## 21. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/runtime/javaThread.hpp` | To see the execution context of a Java thread. |
| `src/hotspot/share/memory/universe.hpp` | Revisit this now that you know what goes into Metaspace vs. the Java heap. |

---

## 22. Coming Up Next

**Lesson 7 – JVM Runtime Data Areas**  
We have successfully parsed and verified our classes. But where do objects go? Where do thread stacks live? Before we execute bytecodes, we must understand the memory topology of the JVM.
