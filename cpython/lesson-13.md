# Lesson 13: Built‑in Types – int, float, and bool

We have laid the theoretical foundation for memory, compilation, and execution. Now, we are going to look at the concrete C structs that actually hold your data, starting with the most fundamental primitives: integers, floats, and booleans.

As a C programmer, you are used to integers overflowing when they hit 32 or 64 bits. Python's integers *never* overflow. Today we look at how CPython implements that infinite precision in C.

---

## 1. Lesson Overview

In this lesson, we study the C implementations of Python's `int`, `float`, and `bool`. We explore:

- **Arbitrary‑precision arithmetic (bignums)** – how Python ints grow to hold any number.
- The **small‑integer cache** – why `256 is 256` is `True` but `257 is 257` is not.
- How object‑oriented inheritance is used at the C level to make `bool` a subclass of `int`.

**Why it matters:**  
Numeric types are the most heavily optimised objects in the interpreter. Understanding how they manage memory and arithmetic is crucial for writing performant C extensions and understanding CPython's overall object design philosophy.

**Prerequisites:** You understand `PyObject` headers (Lesson 8) and `pymalloc` (Lesson 11).

---

## 2. Mental Model – Three Numeric Primitives

- **Float** – A Python `float` is exactly what you expect: a thin wrapper around a standard C `double` (64‑bit IEEE 754).
- **Int** – A Python `int` is **not** a C `int` or `long long`. It is an *array* of 30‑bit digits (a “bignum”). CPython handles integer math much like you do on paper: digit by digit, carrying the remainder.
- **Bool** – A Python `bool` is simply an `int` with specialised behaviour. `True` and `False` are just immortalised integer objects representing `1` and `0`.

```
       PyFloatObject                    PyLongObject (int)
 ┌───────────────────────┐        ┌────────────────────────────┐
 │ ob_refcnt: 1          │        │ ob_refcnt: 1               │
 │ ob_type: &PyFloat_Type│        │ ob_type: &PyLong_Type      │
 ├───────────────────────┤        │ (Size & Sign metadata)     │
 │ ob_fval: 3.14159      │        ├────────────────────────────┤
 └───────────────────────┘        │ ob_digit: [ 234, 18, 0...] │ ◄─ Base‑2³⁰ array
                                  └────────────────────────────┘
```

---

## 3. Where We Are in the Repository

| **Path / File**                         | **Role**                                                                        |
|-----------------------------------------|---------------------------------------------------------------------------------|
| `Objects/longobject.c`                  | Implementation of integer arithmetic (`long_add`, `long_mul`, etc.).            |
| `Include/cpython/longintrepr.h`         | Internal representation of `PyLongObject` (the digit array).                    |
| `Objects/floatobject.c`                 | Implementation of float arithmetic and formatting.                              |
| `Include/cpython/floatobject.h`         | Definition of `PyFloatObject`.                                                  |
| `Objects/boolobject.c`                  | Definition of `PyBool_Type` and the `True`/`False` singletons.                  |
| `Include/cpython/boolobject.h`          | Public macros for booleans (`Py_True`, `Py_False`, `Py_RETURN_TRUE`).            |

**Why they matter:**  
- The `.h` files define the C structs (`PyLongObject`, `PyFloatObject`).
- The `.c` files define the type singletons (`PyLong_Type`) and the math operations.

---

## 4. Concepts We Need First

### 4.1 Sign‑Magnitude vs Two's Complement
Hardware uses Two's Complement for negative numbers. CPython's `int` uses **Sign‑Magnitude**:

- The array of digits (`ob_digit`) is *always* positive (the absolute value).
- The sign (`+` or `-`) is stored separately in the object's header metadata.

### 4.2 Base‑2³⁰
To optimise math, CPython does not store integers in base‑10 or base‑256. On a 64‑bit system, each “digit” in the array is a **30‑bit chunk** of the number, stored in a 32‑bit C `uint32_t`.  
> It leaves 2 bits empty to prevent overflow during C‑level multiplication of two digits (the product of two 30‑bit digits fits safely in a 64‑bit CPU register).

---

## 5. Architecture – Numeric Slots

### 5.1 The `tp_as_number` Slot
`PyTypeObject` has a pointer called `tp_as_number`. This points to a `PyNumberMethods` struct filled with function pointers like `nb_add`, `nb_subtract`, and `nb_multiply`. When you do `a + b`:

1. The VM calls `Py_TYPE(a)->tp_as_number->nb_add(a, b)`.
2. This delegates to the type‑specific C function (e.g., `long_add` for integers, `float_add` for floats).

### 5.2 The Small Integer Cache
CPython pre‑allocates an array of integers from **`-5` to `256`** at startup. If an operation results in one of these numbers, CPython returns a **pointer to the cached object** instead of allocating a new one. (In Python 3.12+, these are marked as Immortal Objects).

---

## 6. Important Data Structures

### `PyFloatObject` – A Thin Wrapper
```c
typedef struct {
    PyObject_HEAD
    double ob_fval;
} PyFloatObject;
```
That's it – just a `PyObject` header followed by a C `double`.

### `PyLongObject` – The Bignum
*(Note: Python 3.12 introduced a “compact” representation to save memory, but conceptually it remains:)*
```c
struct _longobject {
    PyObject_HEAD
    _PyLongValue long_value;    // Contains sign/size tag and the ob_digit array
};
```
The `ob_digit` array is a flexible array member (or managed via the `long_value` wrapper) that holds the 30‑bit digits of the absolute value.

---

## 7. Important Functions

| Function                       | Purpose                                                                                   |
|--------------------------------|-------------------------------------------------------------------------------------------|
| `PyLong_FromLong(long v)`      | Takes a C `long`, allocates a `PyLongObject`, and populates the `ob_digit` array. Returns a *New Reference*. If `v` is between -5 and 256, it returns a cached immortal object. |
| `PyLong_AsLong(PyObject *obj)` | Extracts the C `long` value from a Python int. **Must check for overflow** – use `PyLong_AsLongAndOverflow`. |
| `PyFloat_FromDouble(double v)` | Allocates a new `PyFloatObject`. (Floats are *not* cached like small ints, though they have free lists as seen in Lesson 12). |
| `PyNumber_Add(PyObject *a, PyObject *b)` | Generic API entry point – looks up `nb_add` and calls it.                               |

---

## 8. Important Macros

| Macro                            | Purpose                                                                                    |
|----------------------------------|--------------------------------------------------------------------------------------------|
| `PyFloat_AS_DOUBLE(op)`          | Fast, unchecked macro to cast `op` and read `ob_fval`. (Only use after verifying type).    |
| `Py_True` / `Py_False`           | Global singletons for booleans. Must `Py_INCREF(Py_True)` if returning as a new reference. |
| `Py_RETURN_TRUE` / `Py_RETURN_FALSE` | Helper macros that `Py_INCREF` the singleton and `return` it.                             |

---

## 9. Source Code Exploration

1. **`Include/cpython/floatobject.h`** – Search for `PyFloatObject`. Observe how perfectly simple it is – just a header and a `double`.

2. **`Include/cpython/longintrepr.h`** – Search for `struct _longobject`. Look at how the `ob_digit` array is defined (often as a flexible array member or via a wrapper).

3. **`Objects/longobject.c`** – Search for the internal function `x_add`. This is the raw C loop that iterates through the `ob_digit` arrays of two numbers, adds them together, and handles the carry bit. It looks exactly like a C implementation of grade‑school addition.

---

## 10. Execution Flow – Adding Two Ints

Trace: `x = 1000 + 2000`

1. The VM executes `BINARY_OP`.
2. It pops `1000` and `2000` (`PyLongObject` pointers) from the stack.
3. It delegates to the generic API: `PyNumber_Add(a, b)`.
4. `PyNumber_Add` looks at `a`'s type (`PyLong_Type`).
5. It dereferences `PyLong_Type.tp_as_number->nb_add`, which points to `long_add` in `longobject.c`.
6. `long_add` sees both are positive. It allocates a new `PyLongObject` big enough to hold the result.
7. It adds the base‑2³⁰ digits (with carry).
8. It returns the new `PyLongObject` pointer (`3000`).
9. The VM pushes `3000` to the stack and calls `Py_DECREF` on `1000` and `2000`.

---

## 11. Real Python Example – The Small Int Cache

Consider this classic Python quirk:

```python
a = 256
b = 256
print(a is b)   # True! Both point to the exact same cached PyLongObject

c = 257
d = 257
print(c is d)   # False! Both allocate a distinct PyLongObject
```

Because integers `-5` to `256` are requested so frequently (for list indices, loops, boolean truthiness), CPython initialises them once at startup. When the compiler or runtime needs the integer `256`, `PyLong_FromLong` intercepts the request and hands out the cached pointer. `257` falls outside the cache, triggering a fresh `pymalloc` allocation.

---

## 12. Why This Design?

### Why arbitrary precision for all integers?
- In Python 2, there was `int` (C‑level 32/64‑bit integer) and `long` (arbitrary precision). Programmers constantly hit `OverflowError` or had to manually cast `x = 100L`.
- Python 3 unified them. The mental overhead of integer overflow was removed from the programmer and pushed into the C runtime.

### Why does `bool` subclass `int`?
- Python 1 didn't have a boolean type; people just used `1` and `0`.
- When `bool` was added in Python 2.2, it had to subclass `int` so that old code doing `True + True` (which equals `2`) wouldn't break.

### Why are Booleans singletons?
There are only two possible states. Allocating memory for `True` repeatedly would be a massive waste. CPython creates one `_Py_TrueStruct` at startup and uses its pointer everywhere.

---

## 13. Common Beginner Mistakes

- **Mistake:** Writing a C extension that casts a Python `int` directly to a C `long` using `(long)my_py_int`.  
  **Correction:** `my_py_int` is a pointer to a struct. You must use `PyLong_AsLong(my_py_int)` to extract the C integer. **And you must check for exceptions**, because the Python integer might be too big to fit in a C `long`!

- **Mistake:** Returning `Py_True` from a C extension without incrementing its reference count.  
  **Correction:** Even though singletons are immortal in modern Python, strict API compliance requires you to own the reference you return. Always use `Py_RETURN_TRUE` (which handles the `Py_INCREF` and `return` for you).

---

## 14. Summary

Python isolates the programmer from hardware limitations:

- A **`float`** is a fast, C‑native `double` wrapped in a struct.
- An **`int`** is a dynamic array of 30‑bit digits capable of growing to consume all available RAM to prevent overflow.
- A **`bool`** is a singleton subclass of `int` representing `1` and `0`.

Operations on these objects route through the `tp_as_number` slot, delegating to highly optimised C math loops.

---

## 15. Mental Model to Remember

```
C hardware boundary
─────────────────────────────────────────────────────────────
Python Runtime     |   Float (Wrapper around hardware double)
                   |   Int (Dynamic array of 30‑bit chunks)
                   |   Bool (Subclass of Int, Singleton pointers)
```

---

## 16. Important Functions (Quick Reference)

| Function                       | Purpose                                               |
|--------------------------------|-------------------------------------------------------|
| `PyLong_FromLong()`            | Convert C `long` → Python `int` (cached for -5..256). |
| `PyLong_AsLong()`              | Convert Python `int` → C `long` (check overflow!).    |
| `PyFloat_FromDouble()`         | Convert C `double` → Python `float`.                  |
| `PyFloat_AsDouble()`           | Convert Python `float` → C `double`.                  |
| `PyNumber_Add()`               | Generic addition entry point (uses `tp_as_number`).   |

---

## 17. Important Structs

| Struct               | Purpose                                                 |
|----------------------|---------------------------------------------------------|
| `PyFloatObject`      | Header + C `double`.                                    |
| `PyLongObject`       | Header + array of 30‑bit digits (absolute value + sign). |
| `PyNumberMethods`    | Function table for numeric ops (`nb_add`, `nb_mul`, etc.). |

---

## 18. Important Files (By Component)

| Component       | Files                                                                      |
|-----------------|----------------------------------------------------------------------------|
| `int`           | `Objects/longobject.c`, `Include/cpython/longintrepr.h`                    |
| `float`         | `Objects/floatobject.c`, `Include/cpython/floatobject.h`                   |
| `bool`          | `Objects/boolobject.c`, `Include/cpython/boolobject.h`                     |
| Generic numbers | `Objects/abstract.c` (implements `PyNumber_Add`, `PyNumber_Multiply`, etc.) |

---

## 19. Code‑Reading Exercises

1. Open `Include/cpython/boolobject.h`. Notice that there is **no** `PyBoolObject` struct definition! The type `PyBool_Type` simply uses `PyLongObject` under the hood.

2. Open `Objects/longobject.c`. Search for `PyLong_FromLong`. Find the exact `if` statement that checks the small integer cache (`IS_SMALL_INT`).

3. Open `Objects/floatobject.c`. Search for `float_add`. Look at how it unwraps the C doubles using `PyFloat_AS_DOUBLE`, adds them using standard C hardware `+`, and then wraps the result in a new float using `PyFloat_FromDouble`.

---

## 20. Understanding Questions

1. If a Python `int` holds the number `0`, **what is the size of its `ob_digit` array?**

2. Since `bool` subclasses `int`, its `tp_as_number` struct inherits the integer addition function. When you do `True + True` in Python, **why does it return an `int` (`2`) instead of a `bool`?**  
   *(Hint: Think about what type `long_add` allocates for its return value.)*

3. Why doesn't CPython implement an “infinite precision” `float` the same way it does for `int`?

---

## 21. Suggested Next Files to Read

- **`Objects/unicodeobject.c`** – To understand the most complex built‑in scalar type in CPython: strings.
