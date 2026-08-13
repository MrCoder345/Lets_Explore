# OpenJDK Internals: Day 8 – Object Model, Object Layout & Memory Representation

You know that Java objects live on the Java Heap. But what *is* a Java object when we strip away the abstraction?

In C/C++, a `struct` is just a block of raw memory with no hidden metadata – no type info, no size info, no lock state. Java is different: the JVM must know an object's exact class (for `instanceof` and virtual dispatch), its size (for the GC), and lock ownership (for `synchronized`). All this information is embedded directly in the object itself.

In this lesson, we'll study the **oop** (Ordinary Object Pointer) and the HotSpot object model. We'll look at memory layouts, object headers, field packing, alignment, and Compressed Oops.

---

## 1. The Big Picture (Mental Model)

Every Java object in HotSpot consists of an **Object Header** followed by **Instance Fields**, with optional **Padding** to align the memory.

```
                     Java Object Layout (64-bit JVM, Compressed Oops Enabled)
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                    Object Header (12 bytes)                            │
├────────────────────────────────────────┬───────────────────────────────────────────────┤
│ markWord (8 bytes)                     │ Klass* / Compressed Class Pointer (4 bytes)   │
│ - Identity HashCode                    │ - Pointer to the InstanceKlass in Metaspace   │
│ - GC Age (Generational count)          │ - Tells the JVM: "I am an instance of X"      │
│ - Lock bits (Thread ID if locked)      │                                               │
├────────────────────────────────────────┴───────────────────────────────────────────────┤
│                                   Instance Fields                                      │
├────────────────────────────────────────┬───────────────────────────────────────────────┤
│ int myInt (4 bytes)                    │ Object reference myObj (4 bytes, compressed)  │
├────────────────────────────────────────┴───────────────────────────────────────────────┤
│                                  Padding / Alignment                                   │
├────────────────────────────────────────────────────────────────────────────────────────┤
│ (4 bytes padding to ensure the total size is a multiple of 8 bytes)                    │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

- For arrays, there is a **third header field**: a 4‑byte `length` immediately after the `Klass*`.

---

## 2. JVM Specification vs. HotSpot

| Aspect | Spec Says... | HotSpot Does... |
| :------ | :----------- | :-------------- |
| **Object structure** | Objects have a class, can be locked, and are garbage‑collected. | Defines a specific C++ layout: `markWord` + `Klass*` + fields. |
| **Alignment** | No requirement. | Forces **8‑byte alignment** to optimise CPU cache and GC. |
| **Field ordering** | Not specified. | Performs **field reordering** to minimise padding and pack efficiently. |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/oops/` | The core directory for **Object‑Oriented Pointers** – all object/class representation. |
| `src/hotspot/share/oops/oop.hpp` | Base C++ class for all Java objects (`oopDesc`). |
| `src/hotspot/share/oops/markWord.hpp` | The 64‑bit header structure (locks, GC age, hashcode). |
| `src/hotspot/share/oops/arrayOop.hpp` | Layout for arrays (adds a `length` field). |

---

## 4. Key Concept: Compressed Oops

On 64‑bit systems, pointers are 8 bytes, which bloats the heap compared to 32‑bit. To reduce memory, HotSpot uses `-XX:+UseCompressedOops` (enabled by default). Instead of storing a full 64‑bit address, it stores a **32‑bit offset** from the heap base.

Since all objects are 8‑byte aligned, the last 3 bits of any address are `000`. HotSpot shifts the 32‑bit offset left by 3 bits to reconstruct the full address. This allows a 32‑bit pointer to address up to **32 GB** of heap (\(2^{32} \times 8 = 32\) GB), cutting reference size in half.

---

## 5. How Subsystems Interact with the Header

| Subsystem | What it Does with the `markWord` |
| :-------- | :------------------------------- |
| **IdentityHashCode** | On first call to `System.identityHashCode(obj)`, generates a number and stores it in the `markWord`. |
| **Garbage Collector** | Reads the `markWord` to get the object's **age** (how many GC cycles it survived) to decide promotion to Old Generation. Uses `Klass*` to ask the class for the object's size. |
| **Synchronization** | When you write `synchronized(obj)`, the `markWord` is modified to store a pointer to a lock structure or the thread ID of the locker. |
| **Virtual Dispatch** | When you call `obj.toString()`, the interpreter follows the `Klass*` into Metaspace, looks up the vtable, and finds the correct method. |

---

## 6. Important C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `oopDesc` | `oops/oop.hpp` | **Base class** for every Java object. Contains the header. |
| `markWord` | `oops/markWord.hpp` | Wraps a `uintptr_t`; uses bit‑manipulation to cram lock state, GC age, and hashcode into 64 bits. |
| `arrayOopDesc` | `oops/arrayOop.hpp` | Inherits from `oopDesc`; adds a `length` field. |
| `typeArrayOopDesc` | `oops/typeArrayOop.hpp` | Represents arrays of primitives (`int[]`, `byte[]`, etc.). |
| `objArrayOopDesc` | `oops/objArrayOop.hpp` | Represents arrays of object references (`String[]`). |

**Inheritance Hierarchy:**
```
oopDesc
  ↑
arrayOopDesc
  ↑
typeArrayOopDesc   objArrayOopDesc
```

---

## 7. Critical Functions

- `oopDesc::mark()` / `oopDesc::set_mark()` – accessors for the `markWord`.
- `oopDesc::klass()` – reads the (compressed or not) class pointer and returns the `Klass*` in Metaspace.
- `oopDesc::size()` – calculates the total size of the object in heap words. Delegates to the `Klass` because only the class knows the field count.

---

## 8. Important Macros / Utilities

- `align_up(size, MinObjAlignmentInBytes)` – a ubiquitous HotSpot utility. If an object requires 20 bytes, `align_up(20, 8)` returns 24.
- `cast_to_oop(ptr)` – safely casts a raw heap address to an `oop`.

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: The Root of All Objects
- **Open:** `src/hotspot/share/oops/oop.hpp`
- **Look for:**
```cpp
volatile markWord _mark;
union _metadata {
  Klass*      _klass;
  narrowKlass _compressed_klass;
} _metadata;
```
This is the **exact proof** of the object header. Every Java object has these two C++ fields.

### Tour 2: The Magic Mark Word
- **Open:** `src/hotspot/share/oops/markWord.hpp`
- **Read the block comment** at the top – it explains the bit layout in detail.
- **Look for:** the bit‑slicing diagram showing how bits are used for lock states (Unlocked, Lightweight Locked, Heavyweight Monitor).

### Tour 3: Arrays
- **Open:** `src/hotspot/share/oops/arrayOop.hpp`
- **Look for:** the `length()` method, which calculates the offset right after the `_metadata` to read the 32‑bit array length.

---

## 10. Execution Flow – Allocating an Object

Trace `Object obj = new Object();`:

1. Interpreter hits the `new` bytecode.
2. `InstanceKlass::allocate_instance()` is called.
3. The class (`java.lang.Object`) has 0 fields. Size = Header (12 bytes) + 0 = 12 bytes. Aligned to 8 → **16 bytes**.
4. HotSpot requests 16 bytes from the GC.
5. GC returns a raw memory address.
6. HotSpot casts it to `oopDesc*` and initialises:
   - `_mark` = default unlocked state.
   - `_metadata` = pointer to `java.lang.Object`'s `InstanceKlass` in Metaspace.
7. The `oop` is returned and pushed onto the thread's operand stack.

---

## 11. Real Java Example – Field Layout

Consider this class on a 64‑bit JVM with Compressed Oops:

```java
class User {
    byte age;         // 1 byte
    long id;          // 8 bytes
    String name;      // 4 bytes (compressed reference)
    boolean isActive; // 1 byte
}
```

**Naïve calculation:** Header(12) + byte(1) + long(8) + ref(4) + boolean(1) = 26 bytes → padded to 32.

**HotSpot's actual layout (field reordering):** HotSpot sorts fields by size to minimise padding:
1. Header: 12 bytes.
2. It places 4‑byte fields immediately to fill the gap:
   - `String name` (4 bytes) → now 16 bytes (aligned).
3. Then 8‑byte fields:
   - `long id` (8 bytes) → now 24 bytes.
4. Then 1‑byte fields:
   - `byte age` (1) → 25.
   - `boolean isActive` (1) → 26.
5. Padding (6 bytes) → total **32 bytes**.

So field declaration order does **not** determine memory order.

---

## 12. Why This Design? (The "Why")

**Why embed lock and GC age in the object header?**  
If HotSpot kept a separate hash table mapping objects to lock state or age, every access would require a hash lookup and extra memory. Embedding 64 bits directly into the object means checking a lock or age is a single CPU cache‑line read – blazing fast.

*Note: Project Lilliput is working to shrink the header to 8 bytes, but the dual‑word header remains standard in current LTS releases.*

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| Field declaration order in Java matches memory order. | **No** – HotSpot reorders fields to reduce padding. |
| `sizeof(oopDesc)` gives the size of a Java object. | No – `sizeof(oopDesc)` is only the header (16 bytes). The JVM calculates total size dynamically via `oopDesc::size()`. |
| Primitives are objects. | An `int` field is just 4 raw bytes inside the object – no header. |

---

## 14. Summary

A Java object (`oop`) is a contiguous block of memory on the Java Heap. It begins with:

- A **64‑bit `markWord`** – holding locks, GC age, and hashcode.
- A **pointer to its class metadata** (`Klass*`) – for type info and virtual dispatch.

After that, the **instance fields** are packed tightly, followed by **alignment padding** (to 8 bytes). This design allows the JVM to manage memory, enforce polymorphism, and handle synchronisation efficiently.

---

## 15. Mental Model to Remember

```
Java Object (oop) = markWord (state) + Klass* (type) + reordered fields + padding (to 8 bytes)
```

---

## 16. Important Classes / Structs

- `oopDesc`
- `markWord`
- `arrayOopDesc`

---

## 17. Important Functions / Methods

- `oopDesc::mark()`
- `oopDesc::klass()`
- `oopDesc::size()`

---

## 18. Important Files

- `src/hotspot/share/oops/oop.hpp`
- `src/hotspot/share/oops/markWord.hpp`
- `src/hotspot/share/oops/arrayOop.hpp`

---

## 19. Code‑Reading Exercises

1. **Mark Word layout** – open `src/hotspot/share/oops/markWord.hpp` and read the comment block. Find which bits store the GC `age`. What is the maximum age before promotion?

2. **Array detection** – open `src/hotspot/share/oops/oop.hpp` and find `is_array()`. How does it determine if an object is an array? (Hint: it checks a flag inside the `Klass*`.)

3. **Array length** – open `src/hotspot/share/oops/arrayOop.hpp` and examine `length()`. Notice how it uses pointer arithmetic to jump past the header and read the `length` integer.

---

## 20. Self‑Check Questions

1. You have a Java class with a single `boolean` field. Exactly how many bytes of heap memory does one instance consume on a 64‑bit JVM with Compressed Oops? Show the math.

2. When a thread calls `obj.hashCode()` for the first time, the JVM stores the hash in the `markWord`. What happens if another thread concurrently tries to lock `obj` using `synchronized(obj)`, which also needs space in the `markWord`?

3. Why does HotSpot force all objects to be 8‑byte aligned? How does this alignment make Compressed Oops possible?

---

## 21. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/interpreter/interpreter.hpp` | To see how HotSpot actually executes bytecodes against these objects. |
| `src/hotspot/share/interpreter/templateTable.hpp` | The mapping of bytecodes to assembly generation. |

---

## 22. Coming Up Next

**Lesson 9 – Execution Engine & Bytecode Interpreter**  
Now we have bytecodes in Metaspace and objects on the Heap. It's finally time to run the code. We'll step into the HotSpot Interpreter and see how it executes Java bytecodes, handles stack frames, and transitions to compiled code.
