# Lesson 8: The Python Object System

We have crossed the boundary from the compiler into the runtime.

Up until now, we have talked about “objects” in the abstract. When the interpreter executes `LOAD_CONST` or `BINARY_OP`, it pushes and pops `PyObject *` pointers to the evaluation stack. Now, we are going to look at exactly what that pointer points to in C memory.

---

## 1. Lesson Overview

In this lesson, we dissect **`PyObject`**, the fundamental C structure that represents *everything* in Python. We explore how CPython implements object‑oriented inheritance and polymorphism in pure C using **struct composition** and **type pointers**.

**Why it matters:**  
Every single Python variable, function, class, and module is a `PyObject` at the C level. If you do not understand the exact memory layout of a `PyObject`, you cannot:

- Write a C extension.
- Debug a memory leak.
- Understand how the interpreter executes operations.

**Prerequisites:** You understand C structs, pointers, and memory casting.

---

## 2. Mental Model – C‑Style Inheritance

CPython achieves object‑orientation in C via **structural composition** (often called “C inheritance”).

Every object in memory consists of a standard **header** followed by type‑specific **payload** data. Because the header is strictly identical across all objects, CPython can pass everything around as a `PyObject *`. When it needs to know what the object *actually* is, it looks at the `ob_type` pointer inside the header.

```
[ PyObject * ] ───► ┌──────────────────────┐
                    │ ob_refcnt: 1         │ ◄── How many places hold this pointer?
     [ Header ]     │ ob_type: &PyLong_Type│ ◄── What is this object? (Integer)
                    ├──────────────────────┤
     [ Payload ]    │ ob_digit: [42]       │ ◄── The actual C data (hidden from PyObject)
                    └──────────────────────┘
```

---

## 3. Where We Are in the Repository

| **Directory / File**           | **Role**                                                             |
|--------------------------------|----------------------------------------------------------------------|
| `Include/object.h`             | Public API for object manipulation.                                  |
| `Include/cpython/object.h`     | Internal, exact C layouts of `PyObject` and `PyVarObject`.           |
| `Objects/object.c`             | Generic lifecycle functions – creation, destruction, refcount ops.   |

**Why they matter:**  
- `object.h` contains the public macros and structs.  
- `cpython/object.h` contains the internal, exact C layouts.  
- `Objects/object.c` contains the generic lifecycle functions for creating and destroying these structs.

---

## 4. Concepts We Need First

### “C‑Style Inheritance”
C does not have `class Child : Base`. Instead, C compilers guarantee that if you place a struct as the **very first member** of another struct, their memory addresses are identical.

```c
struct Base { int x; };
struct Child { struct Base base; int y; };
```

You can safely cast a `struct Child *` to a `struct Base *` and back. CPython uses this extensively. Every concrete type (like a list or int) has `PyObject` as its first field.

---

## 5. Architecture – The Three Pillars

The CPython Object Model is built on three pillars:

1. **Identity (Memory Address)** – The actual C pointer address is the object's identity (this is what Python's `is` keyword checks).

2. **State / Lifetime (Reference Count)** – The `ob_refcnt` field tracks how many variables/stacks currently point to this memory. When it hits zero, the memory is freed.

3. **Behavior (Type Object)** – The `ob_type` field is a pointer to a singleton `PyTypeObject` (a massive struct of function pointers like `tp_add`, `tp_hash`). This dictates how the object behaves.

---

## 6. Important Data Structures

### `PyObject` – The Base Header

```c
typedef struct _object {
    _PyObject_HEAD_EXTRA  // Used only in debug builds for a doubly-linked list
    Py_ssize_t ob_refcnt; // Reference count (note: a union in modern CPython)
    PyTypeObject *ob_type;// Pointer to the type definition
} PyObject;
```

### `PyVarObject` – The Base Header for Variable‑Sized Objects

Objects like strings, lists, and tuples have a **variable size**. They use this header:

```c
typedef struct {
    PyObject ob_base;     // Standard header (C inheritance!)
    Py_ssize_t ob_size;   // Number of items in this object (e.g., list length)
} PyVarObject;
```

> **Advanced C Note:** In Python 3.12+, `ob_refcnt` was changed internally from a simple integer to a union with bitfields to support “Immortal Objects” (PEP 683), where specific bit flags mark an object as never to be deallocated. In 3.13 free‑threaded builds, it uses atomics. However, conceptually and via the API, it remains an integer count.

---

## 7. Important Functions

| Function                     | Purpose                                                                                      |
|------------------------------|----------------------------------------------------------------------------------------------|
| `_PyObject_New(type)`        | Allocates heap memory for a new object based on the size defined in its `type`, and initialises the `PyObject` header (`ob_refcnt = 1`, `ob_type = type`). |
| `_PyObject_GC_New(...)`      | Same as above, but allocates the object with extra memory tracking so the Cyclic Garbage Collector can see it (Lesson 10). |
| `PyObject_Repr(op)`          | The C implementation of Python's `repr(x)`. It delegates the work to `op->ob_type->tp_repr`. |

---

## 8. Important Macros

| Macro           | Purpose                                                                                          |
|-----------------|--------------------------------------------------------------------------------------------------|
| `Py_TYPE(op)`   | Returns the `ob_type` pointer of `op`. **Always use this macro** – never read `op->ob_type` directly. |
| `Py_REFCNT(op)` | Returns the reference count.                                                                     |
| `Py_SIZE(op)`   | Returns the `ob_size` if the object is a `PyVarObject`.                                          |
| `PyObject_HEAD` | A macro that expands to the fields of `PyObject`. You will see this used when defining concrete types. |

---

## 9. Source Code Exploration

1. **`Include/cpython/object.h`** – Search for `struct _object`. You will see the exact definition of `PyObject`. Notice the `union` tricks used for the reference count in recent versions.

2. **`Include/cpython/object.h`** – Search for `struct _varobject`. See how it embeds `PyObject ob_base;` as its first member, followed by `ob_size`.

3. **`Objects/object.c`** – Search for `PyObject_Repr(PyObject *v)`. This is the C implementation of Python's `repr(x)`. Notice how it checks `if (v == NULL)`, then delegates the actual work to `Py_TYPE(v)->tp_repr`. This is **polymorphism** in C!

---

## 10. Execution Flow – Creating a Float

Let's trace how CPython creates `x = 4.2` (a Python float):

1. The VM wants to create a float. It calls a C allocator for a `PyFloatObject`.
2. `PyFloatObject` is a C struct defined in `floatobject.h`. Its first field is a `PyObject` header, followed by a C `double ob_fval`.
3. The allocator sets `ob_refcnt = 1`.
4. The allocator sets `ob_type = &PyFloat_Type` (the global singleton defining float behavior).
5. The allocator sets `ob_fval = 4.2`.
6. The allocator returns this memory address as a `PyObject *`.
7. The VM stores this `PyObject *` in the local variables array.

---

## 11. Real Python Example – Identity and Reference Counting

Consider this Python script:

```python
a = [1, 2]
b = [1, 2]
c = a
```

In C memory:

- `a` points to a `PyListObject` (which starts with a `PyVarObject` header). Its `ob_refcnt` is **2** (pointed to by `a` and `c`). Its `ob_size` is `2`.

- `b` points to a **different** `PyListObject`. Its `ob_refcnt` is **1**. Its `ob_size` is `2`.

- Because `a` and `b` are different pointers, `a is b` evaluates to `False`.

- Because `a` and `c` hold the exact same pointer address, `a is c` evaluates to `True`.

---

## 12. Why This Design?

### Why structural inheritance?
It provides extremely fast, zero‑overhead polymorphism. A C function can accept *any* Python object by taking a `PyObject *`. To find out how to hash it, it simply dereferences `op->ob_type->tp_hash`. No expensive C++ vtable lookups are required.

### Why `PyVarObject`?
By standardising `ob_size` immediately after the header, CPython can quickly determine the length of lists, strings, and tuples via the `Py_SIZE()` macro without needing to know the concrete type of the sequence.

---

## 13. Common Beginner Mistakes

- **Mistake:** Thinking a `PyObject *` holds the actual data (like the integer value).  
  **Correction:** `PyObject` *only* holds the reference count and type. The payload data requires safely downcasting the pointer (e.g., `(PyFloatObject *)op`) to access the extended struct fields.

- **Mistake:** Modifying `ob_refcnt` or `ob_type` directly via `op->ob_refcnt++`.  
  **Correction:** *Never* touch struct fields directly. Always use `Py_INCREF(op)` or `Py_TYPE(op)`. In modern CPython, `ob_refcnt` is heavily optimised and tied to thread‑safety and immortality; writing to it directly will cause memory corruption.

---

## 14. Summary

Everything in Python is a `PyObject`. This struct is merely a header containing:

- A reference count (for memory management).
- A type pointer (for behaviour).

Concrete types (like Floats or Lists) embed this header as their **first struct member**, allowing CPython to safely cast any pointer back and forth between a generic `PyObject *` and a specific concrete type.

---

## 15. Mental Model to Remember

```
C Pointer (PyObject *)
 │
 ▼
[ Refcount ] \
[ Type *   ]  - PyObject Header (Generic)
-------------
[ Size     ]  - PyVarObject Header (Only if it's a sequence/collection)
-------------
[ Payload  ]  - Concrete Data (e.g., the actual characters of a string)
```

---

## 16. Important Functions (Quick Reference)

| Function           | Purpose                                              |
|--------------------|------------------------------------------------------|
| `_PyObject_New()`  | Allocates and initialises a new object.              |
| `PyObject_Repr()`  | Returns the string representation (`repr`) of an object. |

---

## 17. Important Structs

| Struct         | Purpose                                     |
|----------------|---------------------------------------------|
| `PyObject`     | The base header for all Python objects.     |
| `PyVarObject`  | The base header for variable‑sized objects. |

---

## 18. Important Files

| File                     | Role                                                         |
|--------------------------|--------------------------------------------------------------|
| `Include/object.h`       | Public API for object manipulation.                          |
| `Include/cpython/object.h` | Exact C layouts of `PyObject` and `PyVarObject`.           |
| `Objects/object.c`       | Generic lifecycle functions – creation, destruction, refcount ops. |

---

## 19. Code‑Reading Exercises

1. Open `Include/cpython/object.h` and find `struct _object`. Observe how small it is. Look for the `union` used for the reference count.

2. In the same file, find `struct _varobject`. See how it includes `PyObject ob_base;` as its first field. This is C‑inheritance in action.

3. Open `Include/cpython/floatobject.h` (or similar). Find `PyFloatObject`. See how it uses `PyObject_HEAD` followed by a `double ob_fval;`.

---

## 20. Understanding Questions

1. If you have a `PyObject *op` and you want to call its `__repr__` method, **why does CPython look at `op->ob_type`** instead of storing function pointers directly on `PyObject`?

2. If the interpreter holds a `PyObject *` that is actually pointing to a `PyListObject`, **what C mechanism prevents the interpreter** from accidentally reading past the end of the `PyObject` header and accessing invalid memory?

3. Based on the `PyVarObject` design, **why does calling `len()` on a list or string in Python take $O(1)$ time** instead of $O(N)$?

---
