# Lesson 5: Bytecode & Code Objects

We need to look at the **static data** that feeds into that loop.

When you write a Python function, the compiler's entire goal is to produce exactly one C struct: a **`PyCodeObject`**. Understanding this struct is the key to understanding how Python manages scope, constants, and variables.

---

## 1. Lesson Overview

In this lesson, we study `PyCodeObject` and the CPython bytecode format. We learn:

- How a stream of 16‑bit integers encodes instructions.
- How those instructions reference Python objects (constants, variable names).
- How this static data is structured in memory.

**Why it matters:**  
The virtual machine cannot function without Code Objects. If you want to understand how CPython maps a local variable `x` to a fast array index, or how it caches data for performance, you must understand the layout of `PyCodeObject`.

**Prerequisites:** You understand that the Interpreter Loop (Lesson 4) iterates over a stream of instructions.

---

## 2. Mental Model – The "Executable Binary"

Think of a `PyCodeObject` as a statically compiled **"executable binary"** for a single Python block (like a function or a module).

Bytecode instructions are small (typically 16‑bit). They cannot physically contain a large string or a 64‑bit integer. Instead, the `PyCodeObject` contains **C arrays (tuples) of pointers**. The bytecode simply contains 8‑bit *indices* into those arrays.

```
       PyCodeObject
 ┌─────────────────────────────┐
 │ co_consts:   (None, 42)     │ ◄── C Array of PyObject* (Constants)
 │ co_varnames: ('x', 'y')     │ ◄── C Array of PyObject* (Local var names)
 │ co_names:    ('print',)     │ ◄── C Array of PyObject* (Global/Attr names)
 │                             │
 │ co_code_adaptive:           │
 │  [ LOAD_CONST, 1 ] ─────────┼───► Pushes co_consts[1] (42) to stack
 │  [ STORE_FAST, 0 ] ─────────┼───► Stores top of stack into local slot 0 ('x')
 │  [ LOAD_GLOBAL,0 ] ─────────┼───► Looks up co_names[0] ('print')
 └─────────────────────────────┘
```

---

## 3. Where We Are in the Repository

| **Path / File**                    | **Role**                                                                    |
|------------------------------------|-----------------------------------------------------------------------------|
| `Include/cpython/code.h`           | Defines the `PyCodeObject` C struct.                                        |
| `Objects/codeobject.c`             | Contains functions to create, destroy, and inspect `PyCodeObject`s.         |
| `Include/opcode_ids.h` (generated) | Defines the integer values for every bytecode instruction (e.g., `LOAD_FAST`). |

**Why they matter:**  
- `code.h` gives us the blueprint.  
- `codeobject.c` implements the lifetime management.  
- `opcode_ids.h` is the dictionary of the VM – every instruction has a unique number.

---

## 4. Concepts We Need First

### 4.1 Immutability
A `PyCodeObject` is **strictly immutable**. Once the compiler generates it, its C struct fields never change. This makes them:
- **Thread‑safe** – they can be shared across multiple threads without locks.
- **Cacheable** – this is exactly what `.pyc` files in `__pycache__` are: serialised `PyCodeObject`s written to disk.

### 4.2 The 16‑bit Code Unit
CPython bytecode consists of 16‑bit words (`uint16_t`). Each word is split into two 8‑bit parts:

- **Byte 1 (Opcode)** – the instruction, e.g., `LOAD_CONST` (which might be 100).
- **Byte 2 (Oparg)** – the argument/index, e.g., `1`.

This compact encoding keeps the bytecode dense and cache‑friendly.

---

## 5. Architecture – Nested Code Objects

Every distinct "block" of Python code gets its own `PyCodeObject`.

If you write a Python script with one class that contains two methods, the compiler generates:

- 1 `PyCodeObject` for the **module level**.
- 1 `PyCodeObject` for the **class body**.
- 2 `PyCodeObject`s for the **methods**.

These objects are **nested**. The module's `co_consts` tuple will contain a pointer to the class's `PyCodeObject`, which in turn contains pointers to the methods' `PyCodeObject`s.

This nesting allows the VM to recursively compile and execute nested scopes.

---

## 6. Important Data Structures

### `PyCodeObject` (defined in `Include/cpython/code.h`)

| Field                 | Type                        | Purpose                                                                                          |
|-----------------------|-----------------------------|--------------------------------------------------------------------------------------------------|
| `co_argcount`         | `int`                       | Number of positional arguments the function takes.                                               |
| `co_kwonlyargcount`   | `int`                       | Number of keyword‑only arguments.                                                                |
| `co_nlocals`          | `int`                       | Total number of local variables (used to size the `localsplus` array in the frame – Lesson 4).   |
| `co_stacksize`        | `int`                       | Maximum size needed for the evaluation stack (pre‑allocated by the VM).                          |
| `co_flags`            | `int`                       | Bit flags (e.g., `CO_OPTIMIZED`, `CO_NESTED`, `CO_GENERATOR`).                                   |
| `co_code_adaptive`    | `_Py_CODEUNIT[]` (flexible) | The raw 16‑bit instruction stream.                                                               |
| `co_consts`           | `PyTupleObject *`           | Tuple of **constant values** (numbers, strings, `None`, nested `PyCodeObject`s).                 |
| `co_names`            | `PyTupleObject *`           | Tuple of **global/attribute names** (strings).                                                   |
| `co_varnames`         | `PyTupleObject *`           | Tuple of **local variable names** (strings).                                                     |
| `co_filename`         | `PyObject *` (string)       | Source file name (for tracebacks).                                                               |
| `co_name`             | `PyObject *` (string)       | The name of the function/block.                                                                  |
| `co_firstlineno`      | `int`                       | Starting line number in the source file.                                                         |
| `co_lnotab` (or `co_lines`) | `PyObject *`         | Mapping from bytecode offset to source line number (for debugging).                              |

> **Note on `co_code_adaptive`:** In modern CPython (3.11+), this field is named to reflect the adaptive interpreter. It is a flexible array member of `_Py_CODEUNIT` – the actual bytecode stream. The older `co_code` pointer has been replaced.

---

## 7. Important Functions

| Function             | Location            | Purpose                                                                 |
|----------------------|---------------------|-------------------------------------------------------------------------|
| `PyCode_New()`       | `Objects/codeobject.c` | Allocates and initialises a new `PyCodeObject` with the given parameters. |
| `_PyCode_CODE(co)`   | Macro in `code.h`   | Takes a `PyCodeObject*` and returns a `_Py_CODEUNIT*` pointer to the raw bytecode array for the interpreter to execute. |

---

## 8. Important Macros

| Macro                | Purpose                                                                         |
|----------------------|---------------------------------------------------------------------------------|
| `_Py_CODEUNIT`       | A typedef for `uint16_t` – one instruction (opcode + oparg).                    |
| `_Py_OPCODE(word)`   | Extracts the 8‑bit opcode from a 16‑bit code unit.                              |
| `_Py_OPARG(word)`    | Extracts the 8‑bit argument from a 16‑bit code unit.                            |
| `_Py_SET_OPCODE(word, op)` | Sets the opcode part of a `_Py_CODEUNIT`.                                   |

---

## 9. Source Code Exploration

1. **`Include/cpython/code.h`** – Open this file and find `struct PyCodeObject`. Read the comments next to `co_consts`, `co_names`, and `co_varnames`. Observe how much metadata is stored here just to make execution and tracebacks possible.

2. **`Include/opcode_ids.h`** – Look at the `#define` list. You will see things like:
   ```c
   #define LOAD_CONST      100
   #define LOAD_FAST       124
   #define STORE_FAST      125
   #define BINARY_OP       122
   ```
   This is the "dictionary" of the virtual machine.

---

## 10. Execution Flow – From Code Object to VM

Trace the C‑level memory access when the VM executes `LOAD_CONST 5`:

1. The VM reads the current `_Py_CODEUNIT` at `frame->instr_ptr`.
2. It extracts `opcode = 100` (`LOAD_CONST`) and `oparg = 5`.
3. It jumps (via computed goto) to the `LOAD_CONST` C code block.
4. The C code reads the `PyCodeObject` attached to the frame (`frame->f_executable`).
5. It accesses the `co_consts` tuple at C array index 5:  
   `obj = co->co_consts[5]`.
6. It increments the reference count of `obj` (because the stack must hold a reference).
7. It pushes `obj` onto the frame's evaluation stack using the `PUSH()` macro.

> **Key insight:** The VM never looks at the string `"x"` during local variable access – it uses integer indices into the `localsplus` C array. This is why local variable access is so fast.

---

## 11. Real Python Example

Consider this Python function:

```python
def f():
    x = 42
    return x
```

### What the Compiler Does:

1. **Constants:** Sees `42` → puts it in `co_consts` at index 1 (index 0 is often `None`).
2. **Local variables:** Sees `x` → puts it in `co_varnames` at index 0.
3. **Bytecode emitted:** (simplified)

| Offset | Opcode      | Argument | Meaning                            |
|--------|-------------|----------|------------------------------------|
| 0      | `LOAD_CONST`| 1        | Push `co_consts[1]` (the integer 42) |
| 2      | `STORE_FAST`| 0        | Pop stack, store in local slot 0 (`x`) |
| 4      | `LOAD_FAST` | 0        | Push local slot 0 (`x`)            |
| 6      | `RETURN_VALUE`| 0      | Pop result, return it              |

### Inspecting from Python:

```python
>>> def f():
...     x = 42
...     return x
...
>>> code = f.__code__
>>> code.co_consts
(None, 42)
>>> code.co_varnames
('x',)
>>> code.co_names
()
```

You are directly inspecting the C struct fields from Python!

### Disassembling the Bytecode:

```python
>>> import dis
>>> dis.dis(f)
  2           0 LOAD_CONST               1 (42)
              2 STORE_FAST               0 (x)
  3           4 LOAD_FAST                0 (x)
              6 RETURN_VALUE
```

`dis` reads the raw `co_code_adaptive` array and translates the opcodes and arguments back to human‑readable mnemonics.

---

## 12. Why This Design?

### Why use 8‑bit arguments (`oparg`)?
- Keeps the bytecode **incredibly dense** – only 16 bits per instruction.
- Maximises CPU instruction cache efficiency (more instructions fit in L1 cache).
- The VM spends most of its time fetching the next instruction – smaller instructions mean fewer cache misses.

### What if an array index is larger than 255?
CPython uses a special instruction called `EXTENDED_ARG`. For example, to access constant 300:

```
EXTENDED_ARG 1
LOAD_CONST   44
```

The VM bit‑shifts and accumulates: `(1 << 8) | 44 = 300`. This allows arbitrarily large indices while keeping the common case (small indices) as a single 16‑bit word.

### Why separate `co_names` (globals) from `co_varnames` (locals)?
- **Local variables** are statically known at compile time – they map to fast C array indices (`STORE_FAST` / `LOAD_FAST`). No string lookup at runtime.
- **Global variables** require slow dictionary lookups at runtime (`LOAD_GLOBAL`), so their string names **must** be preserved in `co_names`.

---

## 13. Common Beginner Mistakes

- **Mistake:** Thinking Python functions parse strings like `"x"` to find local variables at runtime.  
  **Correction:** Local variable names are compiled into integer offsets (`STORE_FAST 0`). This is why local variable access in Python is **drastically faster** than global variable access (which requires a hash table lookup).

- **Mistake:** Assuming `co_code` is just an array of bytes (`char*`).  
  **Correction:** It is an array of **16‑bit words** (`_Py_CODEUNIT`). Each instruction occupies exactly 2 bytes. This makes instruction decoding faster and simpler.

- **Mistake:** Modifying a `PyCodeObject` at runtime to optimise something.  
  **Correction:** `PyCodeObject` is immutable. However, modern CPython (3.11+) uses **adaptive bytecode** – the interpreter *caches* extra data alongside the code object (in the same `co_code_adaptive` array) without modifying the logical bytecode. This is why the field is called `co_code_adaptive`.

---

## 14. Summary

`PyCodeObject` is the immutable, static output of the Python compiler. It contains:

- A 16‑bit instruction array (`co_code_adaptive`).
- C arrays (`co_consts`, `co_names`, `co_varnames`) that hold the actual Python objects (strings, integers, nested code objects) that the instructions refer to.

The VM reads 8‑bit arguments from the instructions and uses them as **indices** into these arrays. This design allows CPython to be both fast (local variable access via C array index) and flexible (global lookups via string name).

---

## 15. Mental Model to Remember

```
(Compiler)  ─────────►  PyCodeObject (The Blueprint)
                             │
                             ▼
(Interpreter) ───────► _PyInterpreterFrame (The Execution State)
                             │
                             ▼
                    Reads co_code_adaptive array
              Uses opargs as indices into co_consts,
              co_names, or co_varnames
```

---

## 16. Important Functions (Quick Reference)

| Function           | Purpose                                                       |
|--------------------|---------------------------------------------------------------|
| `PyCode_New()`     | Allocates a new `PyCodeObject` from the compiler's data.      |
| `_PyCode_CODE(co)` | Macro to get a pointer to the raw bytecode array.             |

---

## 17. Important Structs

| Struct               | Defined In                    | Purpose                                 |
|----------------------|-------------------------------|-----------------------------------------|
| `PyCodeObject`       | `Include/cpython/code.h`      | The compiled code unit (bytecode + metadata). |
| `_Py_CODEUNIT`       | `Include/cpython/code.h`      | A 16‑bit instruction (opcode + oparg).  |

---

## 18. Important Files

| File                            | Role                                                       |
|---------------------------------|------------------------------------------------------------|
| `Include/cpython/code.h`        | Definition of `PyCodeObject` and related macros.           |
| `Objects/codeobject.c`          | Implementation of `PyCode_New`, deallocation, and helpers. |
| `Include/opcode_ids.h` (generated) | Numerical IDs for every bytecode instruction.             |
| `Lib/opcode.py`                 | Python‑side definitions used by the `dis` module.          |

---

## 19. Code‑Reading Exercises

1. **Open `Include/cpython/code.h`** and locate `struct PyCodeObject`. Identify the fields holding the tuples for constants, names, and local variables. Notice the flexible array member `co_code_adaptive[]` at the end.

2. **Launch Python and inspect a function's code object:**
   ```python
   def f(): x = 42
   print(f.__code__.co_consts)      # (None, 42)
   print(f.__code__.co_varnames)    # ('x',)
   ```

3. **Use the `dis` module** to disassemble any function. Pay attention to the numeric arguments after each instruction – they correspond to indices into `co_consts`, `co_varnames`, etc.
   ```python
   import dis
   dis.dis(f)
   ```

---

## 20. Understanding Questions

1. If a `PyCodeObject` is strictly immutable, **how can CPython optimise instructions dynamically at runtime** (like PEP 659 "In‑line Caches")?  
   *(Hint: Look closely at the name `co_code_adaptive` in modern CPython – the bytecode array is modified with cache entries, but the logical bytecode sequence stays the same.)*

2. **Why does `EXTENDED_ARG` exist** instead of just making all bytecode instructions 32‑bit?  
   *(Think about cache efficiency and the common case – most indices fit in 8 bits.)*

3. If a function is called 1,000 times, **how many `PyCodeObject`s are allocated**, and **how many `_PyInterpreterFrame`s** are allocated?  
   *(The code object is shared; each call gets a new frame.)*
