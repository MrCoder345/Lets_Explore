# Lesson 19: Frames & Execution Stack

We have seen how functions are called and how the Virtual Machine loops over bytecode. Now, we must study the actual execution context: the **Frame**.

If you read any CPython internal documentation written before 2022, you will read that CPython allocates a heavy `PyFrameObject` on the heap for every single function call. **Erase that from your memory.** Python 3.11 completely revolutionised the frame stack.

Understanding the modern, lightweight CPython stack is essential for understanding how Python achieves performance and how generators and coroutines (async/await) work under the hood.

---

## 1. Lesson Overview

In this lesson, we study the Execution Stack and the modern `_PyInterpreterFrame`. We learn how CPython:

- Maintains a custom C‑level memory stack per thread.
- Executes functions without calling `malloc()`.
- "Lazily" creates Python‑visible frame objects only when requested.

**Why it matters:**  
The frame is the living, breathing state of a function. It holds the local variables, the instruction pointer, and the evaluation stack. If you don't understand frame memory management, you cannot understand closures, tracebacks, or profiling.

**Prerequisites:** You understand the Interpreter Loop (Lesson 4), Code Objects (Lesson 5), and Vectorcall (Lesson 18).

---

## 2. Mental Model

**Do not confuse** the **C Call Stack** (managed by the OS/hardware via the `RSP` register) with the **Python Execution Stack**.

When a Python thread starts, CPython allocates a large, contiguous chunk of heap memory. This acts as a **custom stack**.

When a Python function is called, CPython simply bumps a pointer in this chunk to carve out a lightweight `_PyInterpreterFrame`.

```
[ _PyThreadState -> datastack_chunk ] (A massive pre-allocated C array)
  │
  ├─ [ _PyInterpreterFrame for main() ]
  │    ├─ instr_ptr
  │    └─ localsplus array (locals + eval stack)
  │
  ├─ [ _PyInterpreterFrame for my_func() ] ◄── CPython just bumps a pointer here!
  │    ├─ instr_ptr: &bytecode[4]
  │    └─ localsplus array
  │
  └─ (Empty unused stack memory...)
```

Only if you explicitly call `sys._getframe()` does CPython allocate a heavy `PyFrameObject` on the actual heap and link it to this lightweight C struct.

---

## 3. Where We Are in the Repository

| **Path / File**                         | **Role**                                                                  |
|-----------------------------------------|---------------------------------------------------------------------------|
| `Include/internal/pycore_frame.h`       | Definition of the lightweight `_PyInterpreterFrame` (the internal state). |
| `Include/cpython/frameobject.h`         | Definition of the heavy `PyFrameObject` (Python‑visible).                 |
| `Objects/frameobject.c`                 | Frame lifecycle, materialisation, and `locals()` implementation.          |
| `Python/ceval.c`                        | Frame creation and destruction during bytecode execution.                 |

**Why they matter:**  
The architectural split between the *internal execution state* (`pycore_frame.h`) and the *Python‑accessible object* (`frameobject.h`) is the defining feature of modern CPython execution.

---

## 4. Concepts We Need First

### 4.1 The `localsplus` Array
A frame does **not** have separate arrays for local variables, cell variables (closures), and the evaluation stack. It has **one** contiguous C array called `localsplus`. The compiler knows exactly how many locals exist, so it puts:

- Locals at the beginning.
- Cell variables (closures) next.
- The remaining space at the end **is** the evaluation stack we discussed in Lesson 4.

### 4.2 Lazy Materialisation
Allocating a `PyObject` requires setting types, reference counts, and tracking it in the GC (Lesson 10). This takes time. By keeping the execution state as a raw C struct (`_PyInterpreterFrame`) and only allocating the `PyFrameObject` if an exception is raised or a debugger asks for it, CPython saves millions of CPU cycles.

---

## 5. Architecture

1. **Thread State** – Every OS thread has a `_PyThreadState` struct. This struct holds a pointer to a `_PyStackChunk`, which provides the raw memory for frames.

2. **Function Entry** – When Vectorcall triggers a Python function, CPython asks the Thread State for the next available slot on the data stack.

3. **Initialisation** – It casts that memory to `_PyInterpreterFrame`, writes the `f_executable` (Code Object), copies the passed arguments into the beginning of `localsplus`, and sets the `instr_ptr`.

4. **Execution** – The frame is passed to `_PyEval_EvalFrameDefault` (Lesson 4).

5. **Function Exit** – When `RETURN_VALUE` is executed, the interpreter simply decrements the Thread State's stack pointer. The memory is instantly reclaimed – **zero `free()` calls**.

---

## 6. Important Data Structures

### `_PyInterpreterFrame` – The Lightweight Internal Frame
```c
typedef struct _PyInterpreterFrame {
    PyObject *f_executable;              // The PyCodeObject (or PyFunctionObject)
    struct _PyInterpreterFrame *previous; // Link to the caller's frame
    PyObject *f_funcobj;                 // The function that created this frame
    PyObject *f_globals;                 // Dict of global variables
    PyObject *f_builtins;                // Dict of built-in variables
    PyObject *f_locals;                  // Dict of locals (usually NULL unless requested)
    PyFrameObject *frame_obj;            // Pointer to the heavy object (NULL by default)
    _Py_CODEUNIT *instr_ptr;             // Instruction pointer
    PyObject **stackpointer;             // Top of the evaluation stack
    PyObject *localsplus[1];             // Flexible array for locals + stack
} _PyInterpreterFrame;
```

### `PyFrameObject` – The Heavy Python‑Visible Frame
```c
typedef struct _frame {
    PyObject_VAR_HEAD
    PyFrameObject *f_back;               // Previous frame (as PyObject)
    PyCodeObject *f_code;                // Bytecode being executed
    PyObject *f_builtins;                // Builtins dict
    PyObject *f_globals;                 // Global dict
    PyObject *f_locals;                  // Dictionary of locals (materialised)
    // ... more fields for tracing, line numbers, etc.
} PyFrameObject;
```

---

## 7. Important Functions

| Function                                    | Purpose                                                                                                 |
|---------------------------------------------|---------------------------------------------------------------------------------------------------------|
| `_PyThreadState_PushFrame(tstate, size)`    | Reserves space on the custom C stack for a new frame.                                                   |
| `_PyThreadState_PopFrame(tstate, frame)`    | Cleans up the references in `localsplus` and releases the stack space.                                  |
| `_PyFrame_GetFrameObject(_PyInterpreterFrame *frame)` | The **lazy materialiser**. If a debugger or exception asks for the frame, this allocates the heavy `PyFrameObject` on the heap, links it, and returns it. |

---

## 8. Important Macros

| Macro                     | Purpose                                                                                          |
|---------------------------|--------------------------------------------------------------------------------------------------|
| `GETLOCAL(i)`             | Reads `frame->localsplus[i]`. This is how `LOAD_FAST` bytecode fetches a local variable – pure C array lookup, no dictionary hashing! |
| `SETLOCAL(i, value)`      | Writes to `frame->localsplus[i]`.                                                                |
| `FRAME_IS_EXECUTING(frame)` | Checks if a frame is currently being executed (vs. suspended).                                  |

---

## 9. Source Code Exploration

1. **`Include/internal/pycore_frame.h`** – Search for `_PyInterpreterFrame`. Look closely at `localsplus`. This is the exact memory the VM loop manipulates constantly.

2. **`Objects/frameobject.c`** – Search for `_PyFrame_GetFrameObject`. Notice the check `if (frame->frame_obj != NULL)`. If it already exists, it just returns it. If not, it calls `_PyObject_GC_New()` to allocate the heap object.

3. **`Python/ceval.c`** – Search for `TARGET(LOAD_FAST)`. You will see it translates directly into `GETLOCAL(oparg)`. Notice how incredibly short the C code is.

---

## 10. Execution Flow – Nested Call

Trace: `main()` calls `process()`

1. **`main()` is running** – Its `_PyInterpreterFrame` occupies bytes 0–128 of the thread's stack chunk.

2. **`main()` executes a `CALL` instruction** to `process()`.

3. **Vectorcall requests a new frame** – The VM looks at the stack pointer (byte 128) and creates a new `_PyInterpreterFrame` starting there.

4. **Linking** – The new frame's `previous` pointer is set to `main()`'s frame.

5. **Execution** – `_PyEval_EvalFrameDefault` begins executing `process()`.

6. **Exception raised!** – The traceback machinery needs to expose this to Python. It calls `_PyFrame_GetFrameObject()` on `process()`'s frame, and then recursively on `main()`'s frame.

7. **Materialisation** – Heap memory is finally allocated so the Python `traceback` object can hold strong references to these frames.

---

## 11. Real Python Example – Lazy Frames

```python
import sys

def f():
    x = 10
    # At this exact moment, NO PyFrameObject exists in C memory.
    
    # We explicitly request it:
    frame = sys._getframe()
    # CPython pauses, calls _PyFrame_GetFrameObject(),
    # allocates it on the heap, and returns the pointer.
    
    print(frame.f_locals['x'])   # 10
```

This is why profiling tools that constantly call `sys._getframe()` drastically slow down Python – they force the VM to allocate heavy heap objects that it had intentionally avoided creating.

---

## 12. Why This Design?

### Why not use the C Call Stack directly?
In C, when a function returns, its stack frame is irrevocably destroyed by the hardware `RET` instruction. Python has **Generators** (`yield`). A generator must pause its execution, return to the caller, and *resume* later with all its local variables intact. If CPython used the C call stack for Python locals, generators would be impossible. By keeping `_PyInterpreterFrame` on a custom heap‑allocated chunk, CPython can detach a frame from the thread stack, save it in a Generator object, and plug it back in later.

### Why are local variables so much faster than globals?
- **Globals** live in `frame->f_globals`, which is a `PyDictObject`. Looking up a global requires a hash table probe (Lesson 16).
- **Locals** live in `frame->localsplus`. Looking up a local requires a single C array index (`ptr + offset`).

---

## 13. Common Beginner Mistakes

- **Mistake:** Assuming `locals()` returns a direct view of the local variables.  
  **Correction:** `localsplus` is a C array. Python dictionaries are hash tables. When you call `locals()` in Python, the C code must dynamically allocate a new `PyDictObject` and copy every variable from the C array into the dictionary. Modifying the dictionary returned by `locals()` does **not** reliably modify the actual C array!

- **Mistake:** Trying to hold a reference to a `_PyInterpreterFrame` in a C extension.  
  **Correction:** When a function returns, the `_PyInterpreterFrame` memory is instantly overwritten by the next function call. If you need a frame to survive, you **must** materialise it into a `PyFrameObject` using `_PyFrame_GetFrameObject()`.

---

## 14. Summary

Modern CPython executes Python code using lightweight, C‑level `_PyInterpreterFrame` structs allocated on a custom, pre‑allocated thread stack. This eliminates `malloc` overhead for function calls.

- The frame contains a single `localsplus` array handling local variables, closures, and the evaluation stack.
- Heavy, GC‑tracked `PyFrameObject`s are only materialised **on‑demand** for debuggers, tracebacks, or explicit introspection.

---

## 15. Mental Model to Remember

```
(Fast Path - 99% of execution)
Custom C Data Stack:
[ Frame 1 ] -> [ Frame 2 ] -> [ Frame 3 (Running) ]
                             │
(Slow Path - Exception or Introspection)
                             ▼
                    _PyFrame_GetFrameObject()
                             │
                             ▼
                    Heap-allocated PyFrameObject
```

---

## 16. Important Functions (Quick Reference)

| Function                            | Purpose                                               |
|-------------------------------------|-------------------------------------------------------|
| `_PyThreadState_PushFrame()`        | Reserve space on the custom C stack.                  |
| `_PyThreadState_PopFrame()`         | Release space and clean up references.                |
| `_PyFrame_GetFrameObject()`         | Lazy materialiser – creates a heavy frame on demand.  |
| `_PyEval_EvalFrameDefault()`        | Execute bytecode in a given frame.                    |

---

## 17. Important Structs

| Struct                 | Purpose                                                      |
|------------------------|--------------------------------------------------------------|
| `_PyInterpreterFrame`  | Lightweight, internal frame – the real execution context.    |
| `PyFrameObject`        | Heavy, Python‑visible frame – created only when needed.      |
| `_PyThreadState`       | Thread state – holds the custom frame stack.                 |

---

## 18. Important Files

| File                               | Role                                                         |
|------------------------------------|--------------------------------------------------------------|
| `Include/internal/pycore_frame.h`  | Definition of `_PyInterpreterFrame` (internal).              |
| `Include/cpython/frameobject.h`    | Definition of `PyFrameObject` (public).                      |
| `Objects/frameobject.c`            | Frame lifecycle, materialisation, and `locals()` handling.   |
| `Python/ceval.c`                   | Frame push/pop during bytecode execution.                    |

---

## 19. Code‑Reading Exercises

1. Open `Include/internal/pycore_frame.h` and locate `_PyInterpreterFrame`. Confirm the presence of the `localsplus` flexible array member.

2. Open `Objects/frameobject.c` and find `_PyFrame_FastToLocalsWithError`. This is the function CPython runs when you call `locals()`. Look at how it extracts items from `localsplus` and inserts them into a dictionary.

3. Open `Python/ceval.c` and search for `TARGET(STORE_FAST)`. See how it uses `SETLOCAL(oparg, value)` to update the C array.

---

## 20. Understanding Questions

1. If generators (`yield`) need their frames to survive after the function yields, **what must CPython do** with the `_PyInterpreterFrame` to prevent it from being overwritten by the next function call on the thread stack?

2. Why does a Python function executing a tight loop with local variables (`while x < 1000: x += 1`) cause **exactly zero allocations** in `pymalloc`?

3. If a global variable and a local variable both resolve to `PyObject *` pointers, **why does the compiler go through the trouble** of identifying them differently in the Symbol Table (Lesson 6)?

---

## 21. Suggested Next Files

- **`Python/symtable.c`** – To revisit how the compiler decides whether a variable ends up in `localsplus` (fast) or `f_globals` (slow).

---

## 22. Suggested Next Lesson

**Lesson 20 – Namespaces, Scopes & Symbol Tables**  
We will now trace how the compiler decides what goes into `localsplus`, what goes into `f_globals`, and how it handles nested scopes and closures.
