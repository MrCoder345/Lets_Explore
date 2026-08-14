# OpenJDK Internals: Day 14 – Garbage Collection Fundamentals

In Lesson 13, we saw how threads rapidly allocate objects into their private TLABs. But memory is finite. Eventually, a thread will ask for a new TLAB and the `CollectedHeap` will say: *"I am out of contiguous space."*

When this happens, the JVM must reclaim memory. In C/C++, you do this manually via `free()` or `delete`. In Java, this is the domain of the Garbage Collector (GC).

Before we study modern algorithms like G1 or ZGC, we must understand the **foundational theory** of Garbage Collection in HotSpot: how the JVM defines "dead" memory, how it finds it, how the C++ code traverses the Java object graph, and why moving objects is dangerous but necessary.

---

## 1. The Big Picture (Mental Model)

Garbage collection in Java is **Tracing‑Based**, not Reference‑Counted. It works by identifying a set of **live** starting points (GC Roots) and walking the graph of references. Anything not reached is considered dead.

```
       1. The Object Graph
┌────────────────────────────────────────────────────────────────────────┐
│  [ GC Roots ]                                                          │
│   ├─ Thread Stack (Local Vars) ────▶ [ Object A ] ──▶ [ Object B ]     │
│   ├─ Static Variables (Classes) ───▶ [ Object C ]                      │
│   └─ JNI Native Handles                                                │
│                                                                        │
│                                      [ Object D ] ──▶ [ Object E ]     │
│                                       ^       (Dead Cycle)       │     │
│                                       └──────────────────────────┘     │
└────────────────────────────────────────────────────────────────────────┘
            │
            ▼
       2. The GC Phases (Generic)
┌────────────────────────────────────────────────────────────────────────┐
│  Phase 1: Mark                                                         │
│  - Traverse from Roots. Mark A, B, and C as "Live".                    │
│  - D and E are never reached.                                          │
│                                                                        │
│  Phase 2: Sweep / Compact / Evacuate                                   │
│  - Reclaim memory occupied by D and E.                                 │
│  - Optional: Move A, B, and C to a new contiguous region.              │
│  - CRITICAL: Update pointers! (Root must now point to A's new address) │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | Spec says... | HotSpot does... |
| :------ | :----------- | :-------------- |
| **Storage management** | Must be automatic. | Uses tracing‑based collectors exclusively (no reference counting). |
| **Collection algorithm** | Not defined. | Implements stop‑the‑world, generational, and concurrent collectors. |
| **Memory traversal** | Not mentioned. | Uses the **`OopClosure`** visitor pattern to traverse the heap. |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/gc/shared/` | Common infrastructure for all GC implementations. |
| `src/hotspot/share/memory/` | Memory iteration abstractions (`iterator.hpp`). |
| `src/hotspot/share/oops/` | Object traversal (`oop.inline.hpp`). |
| **Key files** | `gcCause.hpp`, `collectedHeap.hpp`, `iterator.hpp`, `oop.inline.hpp`. |

---

## 4. Key Concepts You Need to Know

### Reachability
An object is **alive** if and only if it can be reached by following references starting from a **GC Root**. If it's not reachable, it's garbage.

### GC Roots – Where the Trace Starts

1. **Thread Stacks** – local variables holding object references currently in use by executing Java methods.
2. **Metaspace (Statics)** – static fields of loaded `InstanceKlass` metadata.
3. **JNI Handles** – references held by native C code via the Java Native Interface.

### Stop‑The‑World (STW)
If a Java thread (a "mutator") changes pointers while the GC is tracing, the GC might miss a live object and delete it, causing a crash. To prevent this, many GC phases require a **Stop‑The‑World pause** – freezing all Java threads at a **Safepoint**.

---

## 5. Architecture – The GC Workflow

1. **Trigger** – an allocation fails; HotSpot calls `Universe::heap()->collect(GCCause)`.
2. **Safepoint** – the VM coordinates to suspend all Java mutator threads.
3. **Root Scanning** – the GC walks JVM internal structures (stacks, Metaspace, JNI) to find all GC Roots.
4. **Marking** – the GC traverses the object graph using C++ closures, setting bits in a `MarkBitMap` to indicate live objects.
5. **Reclamation** – the GC reclaims dead space. Modern GCs usually **compact** or **evacuate** live objects to prevent fragmentation.
6. **Pointer Fixing** – because objects moved, all pointers (in roots and other objects) must be updated to the new addresses.
7. **Resume** – Java threads are unpaused.

---

## 6. Important C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `GCCause` | `gc/shared/gcCause.hpp` | Enum of all reasons a GC can start (e.g., allocation failure, `System.gc()`). |
| `OopClosure` | `memory/iterator.hpp` | A C++ Visitor that is applied to every `oop` pointer during traversal. |
| `MarkBitMap` | `gc/shared/markBitMap.hpp` | A bitmap used to mark live objects during tracing (separate from the objects). |
| `OopStorage` | `gc/shared/oopStorage.hpp` | A concurrent data structure for storing global native roots (JNI handles). |
| `CollectedHeap` | `gc/shared/collectedHeap.hpp` | Abstract base for the heap; provides the `collect()` method. |

---

## 7. Critical Functions

- `CollectedHeap::collect(GCCause::Cause cause)` – the entry point to trigger a GC cycle.
- `oopDesc::oop_iterate(OopClosure* cl)` – called on each object; the object uses its `Klass` to find reference fields and invokes `cl->do_oop()` for each.
- `OopClosure::do_oop(oop* p)` – the virtual callback executed for every pointer found during tracing.

**Important:** `do_oop` takes a **pointer to a pointer** (`oop*`). Why? Because if the GC moves the object, it needs the memory address of the reference itself to overwrite it with the new address!

---

## 8. Source Code Exploration (Guided Tour)

### Tour 1: Why did the GC start?
- **Open:** `src/hotspot/share/gc/shared/gcCause.hpp`
- Look at the `enum Cause`. You'll see `_allocation_failure` (the most common) and `_java_lang_system_gc` (when someone calls `System.gc()`).

### Tour 2: The Visitor Pattern – OopClosure
- **Open:** `src/hotspot/share/memory/iterator.hpp`
- Find `class OopClosure`. It defines `virtual void do_oop(oop* p)`. Notice the double pointer.

### Tour 3: Traversing an Object
- **Open:** `src/hotspot/share/oops/oop.inline.hpp`
- Look for `oopDesc::oop_iterate_backwards(OopClosure* cl)`. It delegates to the `Klass`, which provides an `OopMapBlock` detailing the exact offsets of reference fields.

---

## 9. Execution Flow – Marking Phase Using Closures

Let's trace the marking phase step‑by‑step:

1. GC creates a `MarkingClosure` (inherits from `OopClosure`).
2. GC iterates Thread 1's stack, finds a local variable pointing to `Object A`.
3. GC marks `A` as live in the `MarkBitMap` and pushes `A` onto a work queue.
4. GC pops `A` and calls `A->oop_iterate(&markingClosure)`.
5. `A`'s `Klass` tells the loop that there's a reference field at offset 16.
6. The loop calls `markingClosure.do_oop(&field_pointer)`.
7. The closure reads the pointed‑to object, marks it live, and pushes it onto the queue.
8. This repeats until the queue is empty – the entire live graph is traced.

---

## 10. Real Java Example – Cycles & Roots

```java
public class GCTest {
    static Object globalRoot; // Root: Static field in Metaspace

    public static void main(String[] args) {
        Object localRoot = new Object(); // Root: Local variable on stack

        Object deadCycle1 = new Object();
        Object deadCycle2 = new Object();
        deadCycle1 = deadCycle2; // deadCycle1 now points to deadCycle2
        deadCycle2 = deadCycle1; // deadCycle2 points back → cycle

        // Even though they reference each other, they are NOT reachable
        // from any GC Root. HotSpot will reclaim both.
    }
}
```

---

## 11. Why This Design? (The "Why")

**Why Tracing instead of Reference Counting (like Python, Swift, or C++ `shared_ptr`)?**

| Reference Counting | Tracing (HotSpot) |
| :----------------- | :---------------- |
| **Cycles** leak memory (counts never reach 0). | Tracing inherently handles cycles. |
| **Performance** – every pointer assignment requires atomic CAS operations (slow). | Moves the cost to GC pauses, not to application execution. |
| **Compaction** – difficult to move objects because there's no central pointer update mechanism. | Tracing can easily update all pointers to moved objects. |
| **Throughput** – high overhead for frequent writes. | High throughput for mutator threads. |

**Tracing** gives higher application throughput and automatic cycle detection at the cost of occasional pauses.

---

## 12. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| Setting an object to `null` forces immediate GC. | `null` only removes one pointer. The object remains until the **next** GC cycle discovers it's unreachable. |
| `System.gc()` is a good way to fix memory issues. | `System.gc()` triggers a full Stop‑The‑World GC – it should almost never be used in production. |
| Java has no memory leaks. | If you store objects in a static `HashMap` and never remove them, they remain strongly reachable – that's a Java memory leak. |

---

## 13. Summary

- HotSpot uses **tracing‑based** garbage collection.
- It starts from **GC Roots** (thread stacks, statics, JNI handles) and traverses the object graph.
- Traversal is done via the **`OopClosure`** visitor pattern.
- Objects that are **not reachable** are considered garbage and are reclaimed (and optionally compacted).
- Tracing handles cycles automatically and allows compaction, but requires Stop‑The‑World pauses for some phases.

---

## 14. Mental Model to Remember

```
GC Roots (Stacks / Statics / JNI)
       │
       ▼
Trace via OopClosure  →  Mark live objects in BitMap
       │
       ▼
Reclaim / Evacuate dead space
       │
       ▼
Fix pointers (because objects may have moved)
```

---

## 15. Important Classes / Structs

- `OopClosure`
- `GCCause`
- `MarkBitMap`
- `CollectedHeap`

---

## 16. Important Functions / Methods

- `oopDesc::oop_iterate()`
- `OopClosure::do_oop()`
- `CollectedHeap::collect()`

---

## 17. Important Files

- `src/hotspot/share/gc/shared/gcCause.hpp`
- `src/hotspot/share/memory/iterator.hpp`
- `src/hotspot/share/oops/oop.inline.hpp`

---

## 18. Code‑Reading Exercises

1. **GC Causes** – open `src/hotspot/share/gc/shared/gcCause.hpp` and read the `enum Cause`. Find the `_g1_inc_collection_pause` cause – it shows how specific GCs extend the shared framework.

2. **OopClosure** – open `src/hotspot/share/memory/iterator.hpp` and look at the definition of `class OopClosure`. Notice the overloaded `do_oop` for `oop*` and `narrowOop*` (Compressed Oops).

3. **Object traversal** – open `src/hotspot/share/oops/oop.inline.hpp` and search for `oop_iterate`. See how templates are used to inline the closure logic for performance.

---

## 19. Self‑Check Questions

1. Why must an `OopClosure`'s `do_oop` method take a **pointer to a pointer** (`oop*`) instead of just the object reference (`oop`)? (Hint: think about object movement during compaction.)

2. If an application thread is inside a C++ JNI method executing native code, does the JVM need to stop that thread at a Safepoint for GC? Why or why not?

3. Explain why a C++ memory leak (forgetting to `delete`) is fundamentally different in mechanism from a Java memory leak.

---

## 20. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/gc/g1/g1CollectedHeap.cpp` | The entry point for the G1 Garbage Collector. |
| `src/hotspot/share/gc/z/zCollectedHeap.cpp` | The entry point for the ZGC Garbage Collector. |

---
