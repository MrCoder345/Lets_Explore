# Lesson 14: Unicode & String Internals

We are moving from the simplicity of numbers into the most complex scalar type in CPython: **text**.

Historically, representing text in C meant dealing with `char *` and null terminators. But Python strings (`str`) are fully Unicode‑compliant. Implementing Unicode in C requires balancing execution speed, memory footprint, and **O(1)** array indexing.

To solve this, CPython uses one of the most brilliant struct designs in the entire repository: **PEP 393 (Flexible String Representation)**.

---

## 1. Lesson Overview

In this lesson, we study the internal C representation of Python strings. We learn how CPython:

- Inspects text at creation time to dynamically choose the smallest possible C array size (1, 2, or 4 bytes per character) while maintaining **O(1)** index access.
- Caches string hashes for fast dictionary lookups.
- Uses **string interning** to deduplicate commonly used strings and save memory.

**Why it matters:**  
Strings are everywhere. They are the keys to almost every dictionary, the names of every variable, and the core of most I/O. Understanding how strings manage memory and caching is critical for writing efficient Python code and C extensions.

**Prerequisites:** You understand `PyObject` headers (Lesson 8) and `pymalloc` (Lesson 11).

---

## 2. Mental Model – The Flexible Representation

If CPython stored every Unicode character as UTF‑8, indexing (`my_string[500]`) would take **O(N)** time because UTF‑8 characters are variable‑length (1 to 4 bytes). You would have to scan from the beginning to find the 500th character.

If CPython stored every character as a 4‑byte integer (UTF‑32) to guarantee O(1) indexing, pure English text would waste 75% of its memory with leading zeros.

**The Solution:** CPython dynamically changes the internal array type based on the *largest code point* in the string.

```
       "hello" (Pure ASCII)
 ┌───────────────────────────┐
 │ PyASCIIObject Header      │
 │  - kind: 1-byte (Latin-1) │
 │  - hash: 89345...         │
 ├───────────────────────────┤
 │ Payload: [h][e][l][l][o]  │ ◄─ C array of 8-bit chars
 └───────────────────────────┘

       "hello 🐍" (Contains Emoji)
 ┌───────────────────────────┐
 │ PyCompactUnicodeObject    │
 │  - kind: 4-byte (UCS-4)   │
 │  - hash: 12431...         │
 ├───────────────────────────┤
 │ Payload: [h   ][e   ]...  │ ◄─ C array of 32-bit uints
 └───────────────────────────┘
```

---

## 3. Where We Are in the Repository

| **Path / File**                      | **Role**                                                                        |
|--------------------------------------|---------------------------------------------------------------------------------|
| `Include/cpython/unicodeobject.h`    | Defines `PyASCIIObject`, `PyCompactUnicodeObject`, and all string macros.        |
| `Objects/unicodeobject.c`            | Implementation of string creation, concatenation, hashing, interning, and more. |

**Why they matter:**  
- The header contains the deeply nested C structs that make the flexible representation possible.  
- The C file contains the logic that scans strings upon creation to determine their “kind” (1, 2, or 4 bytes).

---

## 4. Concepts We Need First

### 4.1 Code Points
A single Unicode character (e.g., `A`, `Ф`, or `🐍`). CPython stores these as integer values (up to 0x10FFFF).

### 4.2 String Interning
CPython maintains a hidden global dictionary of short strings that look like variable names. If you create the string `"my_var"` twice, CPython ensures both point to the exact same `PyObject` in memory. This allows the VM to compare dictionary keys using lightning‑fast C pointer equality (`a == b`) instead of slow string comparison (`strcmp`).

---

## 5. Architecture

1. **Creation** – When you create a string, `PyUnicode_New(size, maxchar)` is called. The C code scans the input to find the highest Unicode code point.

2. **Allocation** – Based on `maxchar`:
   - If `maxchar < 256` → allocates a **1‑byte‑per‑char** array (Latin‑1/ASCII).
   - If `maxchar < 65536` → allocates a **2‑byte‑per‑char** array (UCS‑2).
   - Otherwise → allocates a **4‑byte‑per‑char** array (UCS‑4).

3. **Immutability** – Python strings are strictly immutable. Once created, the payload never changes.

4. **Hash Caching** – The first time `hash(s)` is called, the C code computes it and stores it in the struct. Future calls return the cached integer instantly.

---

## 6. Important Data Structures

*(Note: As of Python 3.12, legacy string structs were purged. We only look at modern compact strings.)*

### `PyASCIIObject` – The Base Struct
```c
typedef struct {
    PyObject_HEAD
    Py_ssize_t length;   // Number of characters (not bytes)
    Py_hash_t hash;      // -1 if not yet computed
    struct {
        unsigned int interned:2;  // Is it in the intern dict?
        unsigned int kind:3;      // 1=ASCII, 2=UCS2, 4=UCS4
        unsigned int ascii:1;     // Is it pure ASCII?
        // ... padding bits
    } state;
    // Payload follows immediately in memory
} PyASCIIObject;
```

### `PyCompactUnicodeObject` – For Non‑ASCII Strings
Used if the string is *not* pure ASCII. It inherits `PyASCIIObject` (C struct composition) and adds a pointer to cache a UTF‑8 encoded version of the string, which is heavily used by the C API.

---

## 7. Important Functions

| Function                               | Purpose                                                                                      |
|----------------------------------------|----------------------------------------------------------------------------------------------|
| `PyUnicode_New(size, maxchar)`         | Core allocator. Calculates memory footprint based on `maxchar` and allocates struct + payload in one contiguous block. |
| `PyUnicode_InternInPlace(PyObject **p)`| Looks the string up in the global intern dictionary. If found, it decrefs the new string, swaps the pointer to the existing singleton, and increments the singleton. |
| `PyUnicode_DecodeUTF8(const char *s, ...)` | Converts a standard C UTF‑8 string into a CPython `PyUnicodeObject`.                         |

---

## 8. Important Macros

C extensions cannot just do `str[i]` because the underlying C array could be `uint8_t`, `uint16_t`, or `uint32_t`. CPython provides macros to abstract this:

| Macro                         | Purpose                                                                                       |
|-------------------------------|-----------------------------------------------------------------------------------------------|
| `PyUnicode_KIND(op)`          | Returns the integer `kind` (1, 2, or 4).                                                      |
| `PyUnicode_DATA(op)`          | Returns a generic `void *` pointer to the start of the character array.                       |
| `PyUnicode_READ(kind, data, index)` | The magic macro. It safely casts the `void *` to the correct C integer array type and returns the character at `index`. |

---

## 9. Source Code Exploration

1. **`Include/cpython/unicodeobject.h`** – Search for `struct { unsigned int interned:2;`. Look at how heavily CPython relies on C bitfields to cram all the string metadata into a single 32‑bit integer to save memory.

2. **`Objects/unicodeobject.c`** – Search for `PyUnicode_New`. Notice that it allocates the struct and the array payload **simultaneously** via `PyObject_Malloc`. The payload immediately follows the struct in memory for perfect CPU cache locality.

3. **`Objects/unicodeobject.c`** – Search for `unicode_hash`. See how it checks `if (self->hash != -1) return self->hash;`. If not, it runs the SipHash algorithm on the raw bytes and caches it.

---

## 10. Execution Flow – Concatenating Strings

Trace: `c = "a" + "🐍"`

1. The VM executes `BINARY_OP` (addition).
2. It delegates to `unicode_concatenate(a, b)`.
3. The C code inspects both strings:
   - `"a"` is kind 1 (ASCII).
   - `"🐍"` is kind 4 (requires 4 bytes).
4. The C code determines the result must be kind 4 (to fit the emoji).
5. It calls `PyUnicode_New(2, 0x1F40D)` (length 2, max char is the snake).
6. It copies `"a"` into the new 4‑byte array (padding it with 3 zero‑bytes).
7. It copies `"🐍"` into the next 4‑byte slot.
8. It returns the new `PyCompactUnicodeObject`.

---

## 11. Real Python Example – Memory Footprint

You can observe the memory impact of the flexible representation using `sys.getsizeof()`:

```python
import sys

# Pure ASCII: 1 byte per character
s1 = "a" * 1000
print(sys.getsizeof(s1))   # ~1049 bytes (Header + 1000 bytes)

# Contains a Cyrillic character (requires UCS‑2): 2 bytes per char
s2 = "a" * 999 + "Ф"
print(sys.getsizeof(s2))   # ~2082 bytes (Header + 2000 bytes)

# Contains an Emoji (requires UCS‑4): 4 bytes per char
s3 = "a" * 999 + "🐍"
print(sys.getsizeof(s3))   # ~4082 bytes (Header + 4000 bytes)
```

Notice how adding a single emoji to a 1000‑character string **quadruples** its total memory footprint! The entire array must be upgraded to 4‑byte integers to maintain O(1) indexing.

---

## 12. Why This Design?

### Why not share memory when slicing?
In some languages, `substring = s[10:20]` creates a view (a pointer + length) into the original string. CPython rarely does this. Why? **Memory leaks.** If you read a 10 GB log file into a string, extract a 5‑character slice, and delete the main string, a view would force the entire 10 GB allocation to stay alive in memory. CPython copies slices into new allocations to allow the massive parent string to be garbage collected.

### Why cache the hash?
Strings are the primary keys for `dict` lookups. Hashing a 5,000‑character string requires scanning all 5,000 bytes. If CPython re‑hashed the string on every dictionary lookup, performance would collapse. Caching it in the struct header makes subsequent dictionary lookups O(1) for the hash phase.

---

## 13. Common Beginner Mistakes

- **Mistake:** Using string concatenation `+` in a tight C loop or Python loop.  
  **Correction:** Because strings are immutable, `s += "a"` creates a brand new `PyUnicodeObject` and copies all previous bytes every iteration – this is **O(N²)**. Use `''.join(list_of_strings)` in Python, or `_PyUnicodeWriter` in the C API, which pre‑allocates an over‑sized buffer and finalises the string at the end.

- **Mistake:** Assuming Python strings are backed by UTF‑8 bytes in memory.  
  **Correction:** They are backed by arrays of 8‑, 16‑, or 32‑bit integers. UTF‑8 is a variable‑length encoding, which CPython actively avoids using as the primary data store because it breaks O(1) indexing.

---

## 14. Summary

Python strings are highly optimised, immutable structures using a **flexible representation**:

- Upon creation, CPython scans the text and allocates an array of 1‑byte, 2‑byte, or 4‑byte integers based on the largest code point.
- This guarantees **O(1)** index access at the cost of potential memory expansion.
- String structs aggressively cache their hash values and are heavily interned to make dictionary lookups lightning fast.

---

## 15. Mental Model to Remember

```
string = "text"
            │
  Is max char < 256? ───YES──► Use PyASCIIObject (uint8_t array)
            │
            NO
            │
  Is max char < 65536? ─YES──► Use PyCompactUnicodeObject (uint16_t array)
            │
            NO
            │
            └────────────────► Use PyCompactUnicodeObject (uint32_t array)
```

---

## 16. Important Functions (Quick Reference)

| Function                         | Purpose                                           |
|----------------------------------|---------------------------------------------------|
| `PyUnicode_New()`                | Allocates a new string with a given size and max char. |
| `PyUnicode_InternInPlace()`      | Interns a string (deduplicates it).               |
| `PyUnicode_DecodeUTF8()`         | Converts a UTF‑8 C string to a Python `str`.      |

---

## 17. Important Structs

| Struct                    | Purpose                                                   |
|---------------------------|-----------------------------------------------------------|
| `PyASCIIObject`           | Base struct for ASCII strings (1‑byte per char).          |
| `PyCompactUnicodeObject`  | Extended struct for non‑ASCII strings (2‑ or 4‑byte per char). |

---

## 18. Important Files

| File                             | Role                                                               |
|----------------------------------|--------------------------------------------------------------------|
| `Include/cpython/unicodeobject.h`| Definition of string structs, kind macros, and bitfields.          |
| `Objects/unicodeobject.c`        | Implementation of creation, concatenation, hashing, interning, etc. |

---

## 19. Code‑Reading Exercises

1. Open `Include/cpython/unicodeobject.h`. Look at the `state` bitfield in `PyASCIIObject`. Identify the `kind` field – it is 3 bits wide, allowing it to store the values 1, 2, or 4.

2. Open `Include/cpython/unicodeobject.h`. Search for `#define PyUnicode_READ`. Look at the ternary operators (or switch statements) that cast the `void *data` to `Py_UCS1*`, `Py_UCS2*`, or `Py_UCS4*` depending on the `kind`.

3. Open `Objects/unicodeobject.c`. Search for `unicode_concatenate`. Observe how it calls `PyUnicode_MAX_CHAR_VALUE` on both strings to figure out the `kind` of the new string before calling `PyUnicode_New`.

---

## 20. Understanding Questions

1. If you write a C extension that loops over a Python string character by character, **why must you call `PyUnicode_KIND()` before starting the loop?**

2. If string interning is so effective for performance, **why doesn't CPython intern *every* string** created by a program?

3. Given the flexible string representation, **what happens under the hood** if you use `s.replace()` to replace a single emoji in a 1,000‑character string with a standard ASCII space?

---

## 21. Suggested Next Files to Read

- **`Objects/listobject.c`** – We have covered scalars (ints, floats, strings). It is time to look at sequences, starting with the ubiquitous Python List.

---

## 22.  Next Lesson

**Lists, Tuples & Sequence Objects**  
We will dive into the dynamic array that powers Python's most versatile data structure, and see how it differs from the immutable tuple.
