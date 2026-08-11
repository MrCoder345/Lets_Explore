# Lesson 15: Lists, Tuples & Sequence Objects

We have explored the foundational scalar types (integers, floats, strings). Now, we are ready to tackle collections, starting with the two most ubiquitous sequences in Python: the mutable `list` and the immutable `tuple`.

As a C programmer, you are intimately familiar with arrays and pointers. In CPython, `list` and `tuple` are both arrays of `PyObject *` pointers under the hood, but they employ completely different memory allocation strategies to achieve their respective performance goals.

---

## 1. Lesson Overview

In this lesson, we study the C structures that power Python lists and tuples. We examine:

- How a `PyListObject` implements an **over‑allocated dynamic array** for **O(1)** amortized appends.
- How a `PyTupleObject` utilises C **flexible array members** to achieve perfect memory compactness and cache locality.

**Why it matters:**  
Python programmers constantly debate whether to use a list or a tuple. If you understand the C layouts, you don't have to guess—you will mathematically *know* the exact memory and performance implications of your choice.

**Prerequisites:** You understand `PyObject` ownership (Lesson 9), `pymalloc` (Lesson 11), and Free Lists (Lesson 12).

---

## 2. Mental Model – Two‑Piece vs. One‑Piece

A **List** is a two‑piece structure:

- A fixed‑size header struct containing a pointer to a **separately allocated** heap array.
- The heap array is intentionally created larger than necessary (“over‑allocated”) so that `append()` can usually just drop a pointer into a pre‑existing slot.

A **Tuple** is a one‑piece structure:

- The array is physically embedded at the very end of the header struct itself.
- It is **perfectly sized** – no extra capacity.

```
      PyListObject (Mutable)                  PyTupleObject (Immutable)
 ┌───────────────────────────┐           ┌───────────────────────────┐
 │ ob_refcnt: 1              │           │ ob_refcnt: 1              │
 │ ob_type: &PyList_Type     │           │ ob_type: &PyTuple_Type    │
 │ ob_size: 3 (Current items)│           │ ob_size: 3 (Current items)│
 │ allocated: 4 (Capacity)   │           ├───────────────────────────┤
 │ ob_item: ───────────────┐ │           │ ob_item[0]: &PyLong(1)    │
 └─────────────────────────│─┘           │ ob_item[1]: &PyLong(2)    │
                           │             │ ob_item[2]: &PyLong(3)    │
   (Separately allocated)  ▼             └───────────────────────────┘
 ┌───────────────────────────┐          
 │ [0]: &PyLong(1)           │           (Memory is contiguous and exact)
 │ [1]: &PyLong(2)           │
 │ [2]: &PyLong(3)           │
 │ [3]: NULL (Empty slot)    │
 └───────────────────────────┘
```

---

## 3. Where We Are in the Repository

| **Path / File**                      | **Role**                                                                        |
|--------------------------------------|---------------------------------------------------------------------------------|
| `Include/cpython/listobject.h`       | Definition of `PyListObject`.                                                   |
| `Objects/listobject.c`               | Implementation of list operations (append, insert, resize, etc.).               |
| `Include/cpython/tupleobject.h`      | Definition of `PyTupleObject` (with flexible array member).                     |
| `Objects/tupleobject.c`              | Implementation of tuple creation, deallocation, and sequence protocol.          |

**Why they matter:**  
- The header files expose the C structs.  
- The C files implement the sequence protocols (resizing, indexing, iteration).

---

## 4. Concepts We Need First

### Flexible Array Members (The “Struct Hack”)
In C, if you declare an array of size `1` at the very end of a struct (`PyObject *ob_item[1];`), you can over‑allocate memory when creating the struct, e.g.:

```c
PyTupleObject *op = PyObject_Malloc(sizeof(PyTupleObject) + (size - 1) * sizeof(PyObject *));
```

This allows you to safely access `ob_item[0]` through `ob_item[size-1]`. The array data lives in the **exact same memory block** as the struct header, guaranteeing perfect CPU cache locality.

---

## 5. Architecture

### 5.1 List Resizing
- When a list's `ob_size` equals its `allocated` capacity and `append()` is called, CPython calls `list_resize()`.
- It computes a new capacity using a growth pattern (roughly `new = (old >> 3) + old + 6`, meaning it grows by ~12.5%).
- It calls `realloc()` (or a custom wrapper) on the `ob_item` array pointer.

### 5.2 Tuple Exactness
- Tuples are immutable, so their size is known perfectly at compile/runtime.
- `PyTuple_New(size)` allocates exactly `sizeof(PyTupleObject) + (size - 1) * sizeof(PyObject *)`.
- There is no `allocated` field because `ob_size` is the absolute truth.

### 5.3 Ownership
Both structs hold **Strong References** to their items:

- When you put an object in a list, the list steals or increments the reference.
- When the list dies, it loops through its array and calls `Py_DECREF()` on every item.

---

## 6. Important Data Structures

### `PyListObject`
```c
typedef struct {
    PyObject_VAR_HEAD          // Contains ob_refcnt, ob_type, and ob_size
    PyObject **ob_item;        // Pointer to heap array of object pointers
    Py_ssize_t allocated;      // Total capacity of the array
} PyListObject;
```

### `PyTupleObject`
```c
typedef struct {
    PyObject_VAR_HEAD          // Contains ob_refcnt, ob_type, and ob_size
    PyObject *ob_item[1];      // Flexible array member
} PyTupleObject;
```

---

## 7. Important Functions

| Function                          | Purpose                                                                                          |
|-----------------------------------|--------------------------------------------------------------------------------------------------|
| `PyList_New(size)`                | Allocates the struct and the separate `ob_item` buffer. If `size > 0`, initialises array slots to `NULL`. |
| `list_resize(PyListObject *self, Py_ssize_t newsize)` | The internal workhorse – handles growing (and shrinking!) the `ob_item` array.                   |
| `PyList_Append(PyListObject *self, PyObject *item)` | Public API to append an item (calls `list_resize` if needed).                                   |
| `PyTuple_New(size)`               | Allocates the single contiguous block of memory for the tuple and its items.                     |

---

## 8. Important Macros

| Macro                                      | Purpose                                                                                                                                  |
|--------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| `PyList_GET_ITEM(op, i)` / `PyTuple_GET_ITEM(op, i)` | Highly fast, **unchecked** macros that return `op->ob_item[i]`. They return a **Borrowed Reference** – no refcount increment. Use with extreme caution. |
| `PyList_SET_ITEM(op, i, item)` / `PyTuple_SET_ITEM(op, i, item)` | Directly overwrite the pointer at index `i`. **Warning:** These macros do *not* `Py_DECREF` the old item. They are meant only for populating newly created containers. |

---

## 9. Source Code Exploration

1. **`Include/cpython/listobject.h`** – Look at `PyListObject`. Identify the `allocated` field, which differentiates it from a Tuple.

2. **`Objects/listobject.c`** – Search for `list_resize`. Look at the bitwise arithmetic used to calculate `new_allocated`.  
   Example snippet (simplified):
   ```c
   static int
   list_resize(PyListObject *self, Py_ssize_t newsize)
   {
       Py_ssize_t allocated = self->allocated;
       if (allocated >= newsize && newsize >= allocated >> 1) {
           // Enough space, just update size
           self->ob_size = newsize;
           return 0;
       }
       // Calculate new allocation
       Py_ssize_t new_allocated = newsize + (newsize >> 3) + (newsize < 9 ? 3 : 6);
       // ... allocate/reallocate
   }
   ```

3. **`Objects/tupleobject.c`** – Search for `tupledealloc`. Notice how it loops over `ob_item[i]`, calls `Py_XDECREF`, and then returns the tuple struct to the Tuple Free List (from Lesson 12).

---

## 10. Execution Flow – Appending to a List

Trace: `l = [1, 2]; l.append(3)`

1. **`l = [1, 2]`** – `PyList_New(2)` is called.  
   `ob_size = 2`, `allocated = 2`. The array is full.

2. **`l.append(3)`** – The VM calls `list_append()` (the C implementation of the method).

3. The C code calls `app1()`, which realises `ob_size == allocated`.

4. It calls `list_resize(l, 3)`.

5. `list_resize` calculates the new capacity:  
   `new_allocated = 3 + (3 >> 3) + 3 = 3 + 0 + 3 = 6` (roughly).

6. It calls `realloc()` (or `PyMem_Realloc`) to expand the `ob_item` array to hold 6 pointers.

7. It updates `allocated = 6`.

8. It sets `l->ob_item[2] = 3` (stealing the reference to `3`).

9. It updates `ob_size = 3`.

The next few appends will take absolute **O(1)** time and require zero system memory allocation until the capacity is exhausted again.

---

## 11. Real Python Example – Observing Over‑Allocation

You can see the over‑allocation strategy from Python:

```python
import sys

l = []
print(sys.getsizeof(l))   # e.g., 56 bytes (Header only)

l.append(1)
print(sys.getsizeof(l))   # e.g., 88 bytes (Header + buffer for ~4 items)

l.append(2)
print(sys.getsizeof(l))   # 88 bytes (Size did not change!)

l.append(3)
print(sys.getsizeof(l))   # 88 bytes (Still the same buffer)

l.append(4)
print(sys.getsizeof(l))   # 88 bytes (Still)

l.append(5)
print(sys.getsizeof(l))   # 120 bytes (Resized to ~8 slots)
```

When you check `sys.getsizeof()`, tuples scale linearly with every item. Lists jump in **steps** because `sys.getsizeof()` reports the size of the *allocated capacity*, not the current item count.

---

## 12. Why This Design?

### Why use a dynamic array instead of a linked list?
Python `list` is definitively **not** a linked list. Linked lists have terrible CPU cache locality (pointers jump all over the heap) and take **O(N)** time to access `my_list[500]`. CPython lists use a contiguous C array of pointers, giving **O(1)** index access and perfect pre‑fetching for the CPU cache.

### Why have Tuples at all if Lists exist?
Tuples are highly optimised for **memory density** and **lifetime**:

- Because the array is embedded in the header, creating a tuple requires **1** `malloc`; creating a list requires **2** `malloc`s (one for the header, one for the buffer).
- Because tuples are immutable, they can be safely used as dictionary keys (their hashes never change), whereas lists cannot.

---

## 13. Common Beginner Mistakes

- **Mistake:** Assuming Python lists store the actual data sequentially in memory.  
  **Correction:** The `ob_item` array stores **pointers**. The array is perfectly sequential, but the actual integers, strings, or objects it points to are scattered randomly across the `pymalloc` heap.

- **Mistake:** Using `PyList_SET_ITEM` to replace an existing item in a C extension.  
  **Correction:** If you do `PyList_SET_ITEM(list, 0, new_obj)`, the macro blindly overwrites the pointer. The old object's refcount is never decremented, causing a permanent memory leak. You must use `PyList_SetItem` (note the lowercase), which safely `Py_DECREF`s the old item first.

- **Mistake:** Assuming tuples are always faster than lists.  
  **Correction:** For indexing, both are equally fast (just pointer dereferencing). However, tuples are smaller and allocate fewer times, so they are faster to create. But if you need to modify the sequence, a list is the only option.

---

## 14. Summary

- **Lists** are dynamic arrays consisting of a header and an over‑allocated heap buffer, granting **O(1)** amortized appends at the cost of slight memory overhead.
- **Tuples** are tightly packed, immutable sequences utilising C flexible array members for perfect cache locality and minimal memory footprint.
- Both hold strong references to their constituent items.

---

## 15. Mental Model to Remember

```
List  = [ Header Struct ] ---> [ Pointer Array + Empty Capacity ]
Tuple = [ Header Struct | Pointer Array ]
```

---

## 16. Important Functions (Quick Reference)

| Function               | Purpose                                     |
|------------------------|---------------------------------------------|
| `PyList_New()`         | Create a new list.                          |
| `list_resize()`        | Internal resize routine.                    |
| `PyList_Append()`      | Append an item (handles resizing).          |
| `PyTuple_New()`        | Create a new tuple with given size.         |

---

## 17. Important Structs

| Struct           | Purpose                                                      |
|------------------|--------------------------------------------------------------|
| `PyListObject`   | Header + pointer to separate array + capacity.               |
| `PyTupleObject`  | Header + embedded flexible array (size = `ob_size`).         |

---

## 18. Important Files

| File                         | Role                                                   |
|------------------------------|--------------------------------------------------------|
| `Include/cpython/listobject.h`  | Definition of `PyListObject`.                       |
| `Objects/listobject.c`       | List implementation (resizing, appending, etc.).        |
| `Include/cpython/tupleobject.h` | Definition of `PyTupleObject`.                      |
| `Objects/tupleobject.c`      | Tuple implementation (creation, deallocation).          |

---

## 19. Code‑Reading Exercises

1. Open `Include/cpython/tupleobject.h`. Find `PyTupleObject`. Confirm it uses `PyObject *ob_item[1];` – the classic struct hack.

2. Open `Objects/listobject.c`. Search for `PyList_SetItem`. Look at how it explicitly captures the old pointer, overwrites the array slot, and then calls `Py_XDECREF(old_item)`.

3. Open `Objects/listobject.c`. Find `list_resize`. Follow the integer math used to calculate `new_allocated`.

---

## 20. Understanding Questions

1. If you create a tuple `t = ([],)`, the tuple is immutable, but you can do `t[0].append(1)`. From a C `PyTupleObject` perspective, **why does this not violate immutability?**

2. If a list is constantly appended to and popped from, its `ob_size` might drop to 0. **Does `list_resize` automatically shrink the `ob_item` buffer** to return memory to the OS, or does it hold onto it?

3. Why does `PyTuple_New(0)` return a **global singleton** instead of allocating a new empty tuple struct every time?

---

## 21. Suggested Next Files to Read

- **`Objects/dictobject.c`** – To prepare for the most heavily engineered data structure in all of CPython.

---

## 22. Next Lesson

**Lesson 16 – Dictionaries & Hash Tables**  
We will dive into the hash table that powers Python's `dict`, exploring its compact representation, resizing strategies, and the clever tricks used to keep it fast and memory‑efficient.
