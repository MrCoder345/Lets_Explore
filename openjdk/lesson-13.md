# OpenJDK Internals: Day 13 – Heap Organization & TLAB Allocation

We've thoroughly explored how the JVM executes code. Now we must look at where that code stores its data.

In C/C++, you manage memory manually with `malloc()` – which searches free-lists, often requiring mutex locks. In a multi‑threaded Java application serving 10,000 requests per second, with every thread allocating thousands of objects per request, a global lock on the heap would paralyse the JVM.

In this lesson, we'll bridge the gap between the Execution Engine and the Garbage Collector. We'll learn how the Java Heap is logically organised, and study the **TLAB (Thread Local Allocation Buffer)** – the engineering marvel that makes object allocation in Java practically lock‑free and as fast as allocating on the C stack.

---

## 1. The Big Picture (Mental Model)

The Java Heap is a massive, contiguous virtual memory range. Threads don't write to it randomly – the heap is partitioned so threads can allocate without blocking each other.

```
                           The Java Heap (Managed by GC)
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  [ Shared Space (e.g., Eden / Young Gen / ZGC Region) ]                      │
│                                                                              │
│  ┌────────────────────┐  ┌────────────────────┐  ┌─────────────────────────┐ │
│  │ TLAB for Thread 1  │  │ TLAB for Thread 2  │  │ Unallocated Heap Memory │ │
│  │                    │  │                    │  │ (Available for new TLABs│ │
│  │ [Obj A][Obj B]     │  │ [Obj C]            │  │  or large objects)      │ │
│  │               ^    │  │        ^           │  │                         │ │
│  │           (Bump ptr│  │    (Bump ptr)      │  │                         │ │
│  └────────────────────┘  └────────────────────┘  └─────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

- **Heap** – the global memory pool. Requires synchronisation (CAS) to partition.
- **TLAB** – a small chunk of the Heap given exclusively to one thread.
- **Bump Pointer** – a CPU register or local variable tracking the current end of the TLAB.

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | Spec says... | HotSpot does... |
| :------ | :----------- | :-------------- |
| **Heap** | A shared memory area where class instances and arrays are allocated. | Implements `CollectedHeap` with GC‑specific subclasses. |
| **Thread safety** | Not specified – allocations must be thread‑safe. | Uses TLABs to eliminate locking on the fast‑path; only uses CAS for slow‑path refills. |
| **Allocation model** | Not specified. | Bump‑pointer allocation inside TLABs; large objects allocated directly in the global heap. |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/gc/shared/` | Core memory management abstractions shared by all GCs. |
| `src/hotspot/share/gc/shared/collectedHeap.hpp` | The base interface for the heap. |
| `src/hotspot/share/gc/shared/threadLocalAllocBuffer.hpp` | The TLAB implementation. |
| `src/hotspot/share/gc/shared/memAllocator.hpp` | The unified C++ allocation pipeline. |

---

## 4. Key Concepts You Need to Know

### Bump‑Pointer Allocation vs. Free‑List
- **Free‑list** (`malloc`) – memory is fragmented. Finding a 16‑byte hole requires traversing a linked list. Slow.
- **Bump‑pointer** – memory is contiguous. You have a `start`, `end`, and `top` pointer. To allocate 16 bytes:
  ```cpp
  if (top + 16 <= end) { obj = top; top += 16; return obj; }
  ```
  That's just **two CPU instructions** – an addition and a branch.

### CAS (Compare‑And‑Swap)
An atomic CPU instruction. When a TLAB is full, the thread must grab a new chunk from the global heap. Multiple threads might compete for the same chunk. HotSpot uses atomic CAS to ensure only one thread wins, without putting threads to sleep using a heavy OS mutex.

---

## 5. Architecture – The Object Allocation Pipeline

### Fast Path (Assembly / Code Cache)
- The JIT or Interpreter checks if the current thread's TLAB has enough space.
- If yes → bump the pointer, initialise the object header (`markWord` and `Klass*`), and continue.
- The C++ runtime is **never called**.

### Slow Path (C++)
- If the TLAB is full, the assembly stub calls into `InterpreterRuntime::_new` (or `OptoRuntime::new_instance_C`).

### Refill TLAB
- The C++ `MemAllocator` asks the global `CollectedHeap` for a new TLAB.
- This requires an atomic CAS on the shared heap space.

### Large Objects
- If an object is huge (e.g., a 10 MB `byte[]`), it bypasses the TLAB entirely.
- It is allocated directly in the global heap (or a special "Humongous" region) because it would instantly fill a normal TLAB.

### GC Trigger
- If the global heap cannot satisfy the TLAB refill or the large object, a Garbage Collection pause is triggered.

---

## 6. Important C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `CollectedHeap` | `gc/shared/collectedHeap.hpp` | Abstract base class for the JVM heap; implemented by G1, ZGC, etc. |
| `ThreadLocalAllocBuffer` | `gc/shared/threadLocalAllocBuffer.hpp` | Lives inside every `JavaThread`; tracks `_start`, `_top`, `_end`. |
| `MemAllocator` | `gc/shared/memAllocator.hpp` | Encapsulates the allocation logic: try TLAB → refill → allocate outside → OOM. |
| `ObjAllocator` / `ObjArrayAllocator` | `gc/shared/memAllocator.hpp` | Subclasses of `MemAllocator` for standard objects vs. arrays. |

---

## 7. Critical Functions

- `ThreadLocalAllocBuffer::allocate(size_t size)` – the C++ equivalent of the assembly fast‑path; bumps `_top`.
- `MemAllocator::allocate()` – the main C++ entry point when the fast‑path fails.
- `CollectedHeap::allocate_new_tlab()` – grabs a new chunk of memory from the shared heap for a thread.
- `Universe::heap()` – returns the global `CollectedHeap*` singleton.

---

## 8. Important Macros / Flags

| Flag | Effect |
| :--- | :--- |
| `UseTLAB` | Enabled by default. If disabled (`-XX:-UseTLAB`), every `new` forces an atomic CAS on the global heap – performance drops dramatically. |
| `ZeroTLAB` | Determines whether HotSpot zeroes out new TLAB memory before handing it to the thread (for security/determinism). |
| `TLABSize` | The default size of a TLAB (e.g., 256 KB). |

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: The TLAB Structure
- **Open:** `src/hotspot/share/gc/shared/threadLocalAllocBuffer.hpp`
- **Look at the fields:** `HeapWord* _start`, `HeapWord* _top`, `HeapWord* _end`.
- **Find the `allocate` method:** it's pure pointer math – `_top + size <= _end`.

### Tour 2: The Allocation Pipeline
- **Open:** `src/hotspot/share/gc/shared/memAllocator.cpp`
- **Look for:** `MemAllocator::allocate()`.
- **Follow the path:**
  1. Try `allocate_inside_tlab()`.
  2. If that fails, try `allocate_outside_tlab()` – this may refill the TLAB or allocate directly in the shared heap.

### Tour 3: The Heap Interface
- **Open:** `src/hotspot/share/gc/shared/collectedHeap.hpp`
- **Look for:** `virtual HeapWord* mem_allocate(size_t size, bool* gc_overhead_limit_was_exceeded) = 0;`
- This pure virtual function is the boundary where the unified allocation code hands off to the specific GC (G1, ZGC, etc.).

---

## 10. Execution Flow – `new Object()` when TLAB is Full

```
1. Assembly Fast Path:
   Thread tries to bump TLAB._top.  _top + 16 > _end.
   Branch jumps to slow path.

2. C++ Entry:
   Calls InterpreterRuntime::_new (or OptoRuntime::new_instance_C).

3. Allocator Creation:
   Creates an ObjAllocator configured for 16 bytes.

4. MemAllocator::allocate():
   Tries TLAB again (just in case), fails.
   Calls allocate_outside_tlab().

5. Refill Decision:
   HotSpot decides to give the thread a whole new TLAB (e.g., 256 KB) rather than just 16 bytes.

6. Global Heap Interaction:
   Calls Universe::heap()->allocate_new_tlab(256 KB).

7. Atomic CAS:
   The specific GC reserves 256 KB from the shared space using CAS.

8. TLAB Reset:
   The thread's _start, _top, and _end are set to this new 256 KB chunk.

9. Fulfillment:
   The original 16‑byte object is allocated from the new TLAB.
   The oop is returned to Java.
```

---

## 11. Real Java Example – Allocation in a Loop

```java
public class AllocationTest {
    public static void main(String[] args) {
        // Single thread allocating millions of short‑lived objects.
        // Because of TLABs, there is ZERO locking here.
        // This is purely CPU register math and memory writes.
        for (int i = 0; i < 10_000_000; i++) {
            Point p = new Point(i, i);
        }
    }
}

class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }
}
```

- The thread allocates points at blinding speed by bumping `_top`.
- Every few thousand iterations, `_top` hits `_end`, the thread transparently enters C++, grabs a new TLAB, and resumes.

---

## 12. Why This Design? (The "Why")

**Why not just use `malloc` for Java objects?**

| `malloc` | TLAB |
| :------- | :--- |
| Has per‑block overhead (headers). | Zero allocation overhead. |
| Prone to fragmentation. | Heap is tightly packed. |
| Requires locks for thread safety. | TLABs eliminate 99.9% of lock contention. |

**Java's allocation is faster than standard C++ `malloc`/`new`.**  
The trade‑off? Java pays the cost later during Garbage Collection.

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| `new Object()` involves an OS lock or syscall. | **No.** It's a lock‑free pointer bump inside the thread's TLAB – just a few CPU instructions. |
| Objects are allocated immediately across the whole heap. | Threads allocate into their own TLABs. The GC only deals with the global heap. |
| All objects go through the TLAB. | Large objects (e.g., 50 MB arrays) bypass the TLAB and go directly to the global heap. |

---

## 14. Summary

The Java Heap is the global memory pool managed by the GC, but threads don't allocate directly into it for every object. Instead, HotSpot uses **Thread Local Allocation Buffers (TLABs)** to give each thread a private chunk of memory. This allows `new` to be a fast, lock‑free bump‑pointer operation in assembly. The heavy C++ logic (`MemAllocator` and `CollectedHeap`) is only invoked when a TLAB runs out of space or an object is too large.

---

## 15. Mental Model to Remember

| Concept | Description |
| :------ | :---------- |
| **Heap** | Global, shared, managed by GC. |
| **TLAB** | Thread‑local, private, lock‑free bump allocation. |
| **Fast path** | JIT/Interpreter bumps TLAB pointer (assembly). |
| **Slow path** | TLAB full → C++ entry → refill from Heap via CAS. |

---

## 16. Important Classes / Structs

- `CollectedHeap`
- `ThreadLocalAllocBuffer`
- `MemAllocator`
- `ObjAllocator`

---

## 17. Important Functions / Methods

- `MemAllocator::allocate()`
- `ThreadLocalAllocBuffer::allocate()`
- `Universe::heap()`

---

## 18. Important Files

- `src/hotspot/share/gc/shared/collectedHeap.cpp`
- `src/hotspot/share/gc/shared/threadLocalAllocBuffer.cpp`
- `src/hotspot/share/gc/shared/memAllocator.cpp`

---

## 19. Code‑Reading Exercises

1. **TLAB allocation** – open `src/hotspot/share/gc/shared/threadLocalAllocBuffer.inline.hpp`. Look at `allocate(size_t size)`. Notice how simple it is:
   ```cpp
   HeapWord* obj = top();
   if (pointer_delta(end(), obj) >= size) {
       set_top(obj + size);
       return obj;
   }
   return NULL;
   ```

2. **Outside‑TLAB allocation** – open `src/hotspot/share/gc/shared/memAllocator.cpp` and find `MemAllocator::allocate_outside_tlab()`. See how it decides whether to allocate a new TLAB or allocate a huge object directly into the heap.

3. **Heap interface** – open `src/hotspot/share/gc/shared/collectedHeap.hpp` and look at the virtual functions `allocate_new_tlab` and `mem_allocate`. This is where the unified allocation code hands off to the specific GC (G1, ZGC).

---

## 20. Self‑Check Questions

1. You have an application with 10,000 active threads that are mostly idle. What is the memory implication of TLABs even if few objects are being allocated?

2. Two threads try to refill their TLABs at the exact same time. How does the JVM prevent them from receiving the same chunk of global heap memory?

3. Why is Java's bump‑pointer allocation faster than a C++ `malloc` call, and what is the delayed consequence of this design?

---

## 21. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/gc/g1/g1CollectedHeap.hpp` | A sneak peek at a concrete `CollectedHeap` implementation. |
| `src/hotspot/share/gc/shared/gcCause.hpp` | The enum listing every reason a GC might pause your application. |

---

## 22. Coming Up Next

**Lesson 14 – Garbage Collection Fundamentals**  
We've filled the heap with objects. TLABs are exhausting the global memory. Eventually, `CollectedHeap` will fail to provide a new TLAB. When that happens, the JVM must pause and clean up. Next, we study the theory, roots, and reachability of Garbage Collection before diving into specific algorithms.
