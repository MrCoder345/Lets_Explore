# Lesson 9: Reference Counting & Memory Management

We are now tackling the most important operational rule in the entire CPython codebase.

As a C programmer, you are used to managing memory with explicit `malloc()` and `free()`. In CPython, you rarely call `free()` on a Python object directly. Instead, you manage **ownership**.

If you do not master reference counting, any C code you write for CPython will either leak memory or cause a Use‑After‑Free segfault.

---

## 1. Lesson Overview

In this lesson, we study **Reference Counting**, CPython’s primary memory management strategy. We learn:

- How `ob_refcnt` is manipulated.
- How objects are destroyed.
- The strict rules governing reference ownership: **new**, **strong**, **borrowed**, and **stolen** references.

**Why it matters:**  
Every C API function in CPython either returns a *new* reference, a *borrowed* reference, or *steals* a reference. Misunderstanding these semantics is the **#1 cause of crashes** in C extensions and CPython pull requests.

**Prerequisites:** You know that `PyObject` contains the `ob_refcnt` field (Lesson 8).

---

## 2. Mental Model – Hot Air Balloon & Ropes

Think of an object in memory as a **hot air balloon**, and pointers to it as physical ropes holding it down.  
The `ob_refcnt` is exactly the number of ropes attached.

- If a C variable, a list, or a dictionary wants to hold onto the object, it **attaches a rope** (`Py_INCREF`).
- When it is done, it **cuts its rope** (`Py_DECREF`).
- If the rope count hits **0**, the balloon instantly flies away (the memory is deallocated).

```
       [ C Variable 'my_list' ]        [ C Variable 'x' ]
                │ (Rope 1)                     │ (Rope 2)
                ▼                              ▼
            ┌──────────────────────────────────────┐
            │ PyObject (ob_refcnt: 2)              │
            │ ob_type: &PyList_Type                │
            └──────────────────────────────────────┘
```

---

## 3. Where We Are in the Repository

| **Path / File**              | **Role**                                                                  |
|------------------------------|---------------------------------------------------------------------------|
| `Include/object.h`           | Public API – macros `Py_INCREF`, `Py_DECREF`, etc.                         |
| `Include/cpython/object.h`   | Internal representation – `ob_refcnt` field (with modern immortality bits). |
| `Objects/object.c`           | Implementation of deallocation and generic object lifecycle.              |

**Why they matter:**  
- The macros that increment/decrement the count live in the headers so they can be aggressively inlined by the C compiler.
- The actual deallocation logic lives in `object.c`.

---

## 4. Concepts We Need First – The Four Types of References

Understanding these four categories is critical for writing correct C code in CPython:

| Reference Type    | How You Get It                                   | Responsibility                                                                     |
|-------------------|--------------------------------------------------|------------------------------------------------------------------------------------|
| **New Reference** | Calling constructors like `PyLong_FromLong(42)`   | Object created with `ob_refcnt = 1`. **You own the rope** – must call `Py_DECREF` when done. |
| **Strong Reference** | You explicitly called `Py_INCREF(obj)`          | You now own a rope – must call `Py_DECREF` when done.                              |
| **Borrowed Reference** | Calling functions like `PyList_GetItem(list, 0)` | Returns a pointer **without** incrementing refcount. You are “borrowing” the list’s rope. If the list is destroyed, your pointer becomes dangling. To keep it safely, call `Py_INCREF` to upgrade to a Strong Reference. |
| **Stolen Reference** | Passing an object to `PyList_SetItem(list, idx, obj)` | The function **takes ownership** of *your* rope. You are no longer allowed to `Py_DECREF` it – the function has “stolen” it. |

---

## 5. Architecture – Decentralised & Immediate

Reference counting is **decentralised**. There is no background thread checking refcounts.

- Whenever a variable goes out of scope in the interpreter (e.g., `STORE_FAST` overwrites an old local), the VM calls `Py_DECREF(old_obj)`.
- `Py_DECREF` is a macro/inline function that decrements the count. If the count reaches `0`, it directly calls `_Py_Dealloc(obj)`.
- `_Py_Dealloc` looks at `obj->ob_type` and calls its `tp_dealloc` function pointer (e.g., `list_dealloc`). This function:
  - Frees the payload.
  - `DECREF`s any child objects (e.g., items in a list).
  - Frees the struct memory itself.

This is **deterministic** – objects are freed instantly when no longer needed.

---

## 6. Important Data Structures

### `PyObject` – The Refcount Field
```c
typedef struct _object {
    _PyObject_HEAD_EXTRA
    Py_ssize_t ob_refcnt;   // The reference count
    PyTypeObject *ob_type;
} PyObject;
```

> **Note:** As of Python 3.12 (PEP 683), CPython supports **“Immortal Objects”** (like `None`, `True`, `False`). Their `ob_refcnt` is initialised to a special value. The macros are designed to ignore DECREF operations on these specific values, meaning they are never deallocated.

### `PyTypeObject` – The `tp_dealloc` Slot
```c
typedef struct _typeobject {
    // ... many fields ...
    destructor tp_dealloc;   // Function pointer to the deallocation routine
    // ...
} PyTypeObject;
```

---

## 7. Important Functions

| Function            | Location           | Purpose                                                                                         |
|---------------------|--------------------|-------------------------------------------------------------------------------------------------|
| `_Py_Dealloc(op)`   | `Objects/object.c` | Internal function called when `ob_refcnt` hits 0. Routes to the type‑specific `tp_dealloc`.     |
| `Py_CLEAR(op)`      | `Include/object.h` | **Critical safety function** – sets a C pointer variable to `NULL` **before** calling `Py_DECREF` on the original pointer. Prevents re‑entrancy bugs where a destructor might trigger Python code that tries to access the dying object. |

---

## 8. Important Macros

| Macro                         | Purpose                                                                                              |
|-------------------------------|------------------------------------------------------------------------------------------------------|
| `Py_INCREF(op)`               | Increments refcount. (Assumes `op` is not `NULL`.)                                                   |
| `Py_DECREF(op)`               | Decrements refcount. If it hits 0, deallocates. (Assumes `op` is not `NULL`.)                        |
| `Py_XINCREF(op)` / `Py_XDECREF(op)` | The “X” variants check `if (op == NULL)` first – if so, they do nothing. Used heavily in error‑cleanup paths. |

---

## 9. Source Code Exploration

1. **`Include/object.h`** – Search for `Py_DECREF`. You will see it implemented as a `static inline` function or macro. Look at the core logic:
   ```c
   if (--op->ob_refcnt == 0) {
       _Py_Dealloc(op);
   }
   ```
   (In current versions, you will see extra bitwise logic checking `_Py_IsImmortal(op)`, but the concept is identical.)

2. **`Objects/object.c`** – Search for `_Py_Dealloc`. Notice how it ultimately calls:
   ```c
   destructor dealloc = Py_TYPE(op)->tp_dealloc;
   dealloc(op);
   ```

---

## 10. Execution Flow – Reassigning a Variable

Let's trace: `x = 10; x = 20`

1. **First assignment `x = 10`**
   - VM gets a pointer to the integer `10` object.
   - VM calls `Py_INCREF(10)`.
   - VM stores the pointer in local variable slot `x`.

2. **Second assignment `x = 20`**
   - VM gets a pointer to the integer `20` object.
   - VM calls `Py_INCREF(20)`.
   - VM **swaps** the pointer in local variable slot `x` (so it now points to `20`).
   - VM calls `Py_DECREF(10)` on the old value.
   - If `10`'s refcount hits 0, it is deallocated. (In reality, small integers are immortal/cached, but conceptually this is the flow.)

---

## 11. Real Python Example – Object Lifetime

```python
def f():
    a = [1, 2, 3]   # List created, refcnt = 1
    b = a           # b points to same list, refcnt = 2
    del a           # Name 'a' removed, refcnt = 1
    # Function ends. 'b' goes out of scope, refcnt = 0.
    # List is destroyed.
```

When the list is destroyed, its `tp_dealloc` (which is `list_dealloc`) will iterate through its internal array and call `Py_DECREF` on each integer object (1, 2, 3). Destruction cascades.

---

## 12. Why This Design?

### Why Reference Counting over Tracing Garbage Collection (like Java/Go)?
1. **Deterministic destruction** – The moment a file object or socket goes out of scope, its refcount hits 0, and it is closed instantly. You don't have to wait for a GC pause.
2. **C Extension Simplicity** – It is much easier for C programmers writing extensions to manually `Py_INCREF`/`Py_DECREF` than to interface with a complex background root‑tracking system.

### Performance cost
The tradeoff is that *every* variable assignment in Python executes C‑level integer math (increment/decrement). This adds overhead, but the deterministic behaviour is worth it for most use cases.

---

## 13. Common Beginner Mistakes

- **Mistake:** Storing a borrowed reference in a C struct.
  ```c
  PyObject *item = PyList_GetItem(list, 0);  // Borrowed!
  my_struct->ptr = item;                     // DANGER!
  ```
  **Correction:** The list owns the object. If the list is cleared, `item` is freed, and `my_struct->ptr` becomes dangling. You **must** call `Py_INCREF(item)` to take strong ownership.

- **Mistake:** Using `Py_DECREF(op)` when `op` might be `NULL`.
  **Correction:** Always use `Py_XDECREF(op)` if you aren't 100% sure the pointer is valid.

- **Mistake:** Forgetting that `PyList_SetItem` **steals** a reference. If you create an object, put it in a list, and then `Py_DECREF` it, you will double‑free.

---

## 14. Summary

Reference counting is CPython's deterministic memory management model:

- `Py_INCREF` and `Py_DECREF` manipulate the `ob_refcnt` field.
- When the count hits 0, the object is immediately destroyed via its type's `tp_dealloc` function.
- In the C API, you must strictly track whether you are handling **New**, **Strong**, **Borrowed**, or **Stolen** references.

---

## 15. Mental Model to Remember

```
                  C Code / Interpreter Frame
                         │
             (Py_INCREF) │ (Py_DECREF)
                         ▼
┌─────────────── PyObject ───────────────┐
│ ob_refcnt: N                           │
│ if (--ob_refcnt == 0) -> tp_dealloc()  │
└────────────────────────────────────────┘
```

---

## 16. Important Functions (Quick Reference)

| Function         | Purpose                                       |
|------------------|-----------------------------------------------|
| `_Py_Dealloc()`  | Internal deallocator – calls `tp_dealloc`.    |
| `Py_CLEAR()`     | Sets pointer to `NULL` *before* decref – prevents re‑entrancy. |

---

## 17. Important Structs

| Struct         | Relevant Field          |
|----------------|-------------------------|
| `PyObject`     | `ob_refcnt`             |
| `PyTypeObject` | `tp_dealloc`            |

---

## 18. Important Files

| File                   | Role                                                       |
|------------------------|------------------------------------------------------------|
| `Include/object.h`     | Macros `Py_INCREF`, `Py_DECREF`, `Py_CLEAR`.               |
| `Objects/object.c`     | Implementation of `_Py_Dealloc` and generic object lifecycle. |

---

## 19. Code‑Reading Exercises

1. Open `Include/object.h`. Find the macro/inline definition for `Py_DECREF`. Notice how it handles the conditional jump to `_Py_Dealloc`.

2. Search `Include/object.h` for `Py_CLEAR`. Read the macro definition. Observe how it sets the pointer variable to `NULL` **before** calling `Py_DECREF` on the temporary pointer.

3. Open `Objects/listobject.c`. Find `list_dealloc()`. Look at how it loops over its items, calls `Py_XDECREF` on every item, and then frees its own memory. This is cascading destruction.

---

## 20. Understanding Questions

1. If you write a C function that returns a `PyObject *` to the Python interpreter, **should that pointer be a Borrowed reference or a New reference**?

2. If an object's `tp_dealloc` function executes Python code (e.g., via a `__del__` method), **what might happen if you didn't use `Py_CLEAR`** and the Python code tried to access the object being destroyed?

3. Reference counting perfectly tracks linear ownership. But **what happens** if Object A holds a reference to Object B, and Object B holds a reference to Object A, and both are removed from the local namespace?

---
