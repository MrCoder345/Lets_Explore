# Lesson 18: Functions & Call Mechanism

We are now bridging the gap between the Objects we just studied and the Interpreter Loop we looked at in Phase 2.

In Python, you can put parentheses `()` after many things: classes, methods, C‑builtins, and Python functions. To the interpreter, these are all just `PyObject *` pointers. How does CPython know what C code to execute when it encounters a `CALL` instruction?

The answer lies in the **Vectorcall Protocol** and the function objects.

---

## 1. Lesson Overview

In this lesson, we study how functions are called in CPython. We examine the difference between:

- A Python function (`PyFunctionObject`).
- A C built‑in function (`PyCFunctionObject`).

Crucially, we dissect **PEP 590 (Vectorcall)** – the modern, highly optimised C API that allows CPython to pass arguments without wasting memory on temporary tuples and dictionaries.

**Why it matters:**  
Function calls are the most frequent operation in Python. Historically, they were a major performance bottleneck. If you write C extensions, using the modern Vectorcall API instead of the legacy `tp_call` API will dramatically speed up your code.

**Prerequisites:** You understand `PyCodeObject` (Lesson 5) and the Evaluation Stack (Lesson 4).

---

## 2. Mental Model

There is a strict difference between the *blueprint* of code and a *callable function*.

- **`PyCodeObject`:** The immutable bytecode. Created by the compiler (Lesson 5).
- **`PyFunctionObject`:** The actual Python object you interact with at runtime. It holds a pointer to the `PyCodeObject`, plus the dynamic state needed to run it (default arguments, closures, and a pointer to the global variables dictionary).

When you call a function, CPython uses the **Vectorcall** mechanism. Instead of packing arguments into a `PyTuple` (which requires `pymalloc` and refcounting overhead), the interpreter passes a raw C array of `PyObject *` pointers directly from the Evaluation Stack to the C function.

```
[ Evaluation Stack ]
  ...
  obj_ptr_3 (kwarg val)
  obj_ptr_2 (arg 2)
  obj_ptr_1 (arg 1)
  function_ptr ◄────── Interpreter says: "Call this, pass the 3 pointers above it."

       ↓ (Vectorcall Dispatch)

Is it a Python Function? ──► Create Frame, point it at func_code, run ceval.c
Is it a C Function? ───────► Execute the raw C function pointer directly
```

---

## 3. Where We Are in the Repository

| **Path / File**                         | **Role**                                                                         |
|-----------------------------------------|----------------------------------------------------------------------------------|
| `Objects/call.c`                        | The central hub for all `()` operations – routing logic.                         |
| `Objects/funcobject.c`                  | Implementation of Python function objects.                                       |
| `Include/cpython/funcobject.h`          | Definition of `PyFunctionObject`.                                                |
| `Objects/methodobject.c`                | Implementation of C function (built‑in) objects.                                 |
| `Include/cpython/methodobject.h`        | Definition of `PyCFunctionObject` and `PyMethodDef`.                             |

**Why they matter:**  
`call.c` contains the routing logic that figures out *how* to call an object. The other files define the concrete structs for the callables.

---

## 4. Concepts We Need First

### 4.1 Legacy `tp_call`
Historically, every callable type implemented the `tp_call(PyObject *callable, PyObject *args, PyObject *kwargs)` slot. This forced the interpreter to:

- Take arguments off the stack.
- Allocate a tuple for `args`.
- Allocate a dict for `kwargs`.
- Pass them to the C function.
- Instantly destroy the tuple and dict.

This was **tremendously slow** due to all the intermediate allocations.

### 4.2 Vectorcall (Fastcall)
Modern CPython bypasses `tp_call`. It passes:

- A C array: `PyObject **args`.
- The length as an integer.
- Keyword argument names as a separate tuple.

Zero intermediate allocations are required.

---

## 5. Architecture

1. **The Bytecode** – The interpreter hits a `CALL` instruction.
2. **The Dispatcher** – `ceval.c` pops the callable and calls `_PyObject_Vectorcall()` (or similar routing functions in `call.c`).
3. **The Check** – The router checks if the callable's type supports vectorcall (via the `Py_TPFLAGS_HAVE_VECTORCALL` flag).
4. **The Execution**:
   - If it's a **C Function** (like `len()`), it directly invokes the underlying C function pointer, passing the array of arguments.
   - If it's a **Python Function**, it routes to `_PyFunction_Vectorcall`. This allocates a `_PyInterpreterFrame` (Lesson 4), copies the C array of arguments into the frame's `localsplus` array, and pushes the frame onto the thread stack to be executed by the bytecode loop.

---

## 6. Important Data Structures

### `PyFunctionObject` (A pure Python function)
```c
typedef struct {
    PyObject_HEAD
    PyObject *func_code;      // Pointer to PyCodeObject
    PyObject *func_globals;   // Pointer to the module's dictionary
    PyObject *func_defaults;  // Tuple of default arguments
    PyObject *func_closure;   // Tuple of cell objects (for nested scope)
    PyObject *func_doc;       // Docstring
    PyObject *func_name;      // Name of the function
    PyObject *func_dict;      // __dict__ for arbitrary attributes
    // ... more fields
} PyFunctionObject;
```

### `PyCFunctionObject` (A C built‑in like `len` or `id`)
```c
typedef struct {
    PyObject_HEAD
    PyMethodDef *m_ml;        // Struct holding the C function pointer and signature info
    PyObject    *m_self;      // The object this method is bound to (e.g., the list for list.append)
    PyObject    *m_module;    // The module this function belongs to
} PyCFunctionObject;
```

### `PyMethodDef` (Defines a C function for export)
```c
typedef struct PyMethodDef {
    const char  *ml_name;     // Python name of the function
    PyCFunction  ml_meth;     // The actual C function pointer
    int          ml_flags;    // Flags (METH_VARARGS, METH_KEYWORDS, METH_O, etc.)
    const char  *ml_doc;      // Docstring
} PyMethodDef;
```

---

## 7. Important Functions

| Function                                      | Purpose                                                                                                      |
|-----------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| `_PyObject_Vectorcall(callable, args, nargsf, kwnames)` | The primary internal C API for calling anything. `args` is a `PyObject **` (C array). `nargsf` is the number of positional args (with flag bits). `kwnames` is a tuple of keyword names. |
| `PyObject_Call(callable, args, kwargs)`       | The legacy C API. Avoid this in new C extensions; use `PyObject_Vectorcall` instead.                         |
| `_PyFunction_Vectorcall()`                    | Handles calling a `PyFunctionObject` – allocates a frame and starts the bytecode loop.                       |

---

## 8. Important Macros

| Macro                                   | Purpose                                                                                                        |
|-----------------------------------------|----------------------------------------------------------------------------------------------------------------|
| `PY_VECTORCALL_ARGUMENTS_OFFSET`        | A flag bit. When keyword arguments are passed, the C array contains positional args *followed* by keyword values. Setting this flag tells the callee: "You may temporarily modify `args[-1]`" (used for extreme optimisations in method binding). |

---

## 9. Source Code Exploration

1. **`Include/cpython/funcobject.h`** – Locate `PyFunctionObject`. Compare it to `PyCodeObject` (Lesson 5). Notice that `PyCodeObject` has no concept of "globals" – it is just static bytecode. `PyFunctionObject` binds that static code to a specific global namespace.

2. **`Objects/call.c`** – Search for `_PyObject_Vectorcall`. It checks the type's flags for `Py_TPFLAGS_HAVE_VECTORCALL`. If present, it dereferences a function pointer and calls it, passing the C array directly.

3. **`Objects/call.c`** – Search for `_PyFunction_Vectorcall`. This extracts the `func_code`, handles default arguments, and prepares the frame for the VM.

---

## 10. Execution Flow – A Simple Call

Trace: `result = my_func(10, 20)`

1. The VM pushes `my_func`, `10`, and `20` to the evaluation stack.
2. The VM executes `CALL 2` (Call with 2 positional args).
3. The VM calculates a C pointer (`PyObject **args`) pointing to the slot on the stack holding `10`.
4. The VM calls `_PyObject_Vectorcall(my_func, args, 2, NULL)`.
5. The vectorcall router sees `my_func` is a `PyFunctionObject`.
6. `_PyFunction_Vectorcall` allocates a new `_PyInterpreterFrame`.
7. It copies `args[0]` (10) and `args[1]` (20) into the local variables slots of the new frame.
8. It points the frame at `my_func->func_code`.
9. The VM begins executing the bytecode of `my_func`.

---

## 11. Real Python Example – Closures

```python
def make_adder(x):
    def add(y):
        return x + y  # 'x' is in the closure
    return add

func1 = make_adder(10)
func2 = make_adder(20)
```

In C memory, both `func1` and `func2` (which are `PyFunctionObject`s) point to the **exact same** `PyCodeObject` (the static bytecode for `add`). However, their `func_closure` fields point to different memory holding `10` and `20`. The function object provides the runtime context for the static code.

---

## 12. Why This Design?

### Why split `PyCodeObject` and `PyFunctionObject`?
Memory efficiency and strict immutability. Bytecode is identical every time a function is created. By splitting them:

- A recursive function or a function generated in a loop only ever allocates **one** `PyCodeObject` (heavy).
- Multiple `PyFunctionObject` wrappers (lightweight) are allocated to hold different default arguments or closures.

### Why Vectorcall?
Passing arguments from Python to C used to require `PyTuple_New()`. If you call `len(my_list)` 10 million times in a loop, Python allocated and destroyed 10 million temporary 1‑item tuples. Vectorcall eliminates this by pointing directly into the interpreter's pre‑existing stack memory.

---

## 13. Common Beginner Mistakes

- **Mistake:** Using `PyObject_CallObject` or `PyObject_CallFunction` in C extensions.  
  **Correction:** These are legacy APIs. They force CPython to pack your arguments into a tuple, defeating the Vectorcall optimisation. Always use `PyObject_Vectorcall` or `PyObject_CallOneArg` for maximum performance.

- **Mistake:** Thinking methods (like `list.append`) are the same as functions.  
  **Correction:** A method is a function **bound** to an instance. `list.append` is a `PyCFunctionObject` where the `m_self` field points to the specific list instance, allowing the C code to know which list to mutate.

- **Mistake:** Assuming `PyFunctionObject` contains the bytecode directly.  
  **Correction:** It contains a pointer to a `PyCodeObject`. The bytecode lives in the `PyCodeObject`, which is shared across all function instances.

---

## 14. Summary

Callable objects in CPython route through the highly optimised Vectorcall protocol in `Objects/call.c`, which passes arguments as raw C arrays to avoid allocation overhead.

- `PyFunctionObject`s bind static `PyCodeObject`s to dynamic runtime context (globals, closures).
- `PyCFunctionObject`s bind C function pointers to Python instances.
- The `CALL` bytecode simply sets up the C array and lets the Vectorcall router figure out how to execute the target.

---

## 15. Mental Model to Remember

```
           [ Evaluation Stack ]
                  │
          (Vectorcall Pointer)
                  ▼
         _PyObject_Vectorcall()
                  │
       ┌──────────┴──────────┐
       ▼                     ▼
 PyFunctionObject      PyCFunctionObject
(Python Runtime)        (C Extension/Builtin)
       │                     │
 Allocates Frame       Executes C Function Pointer
       │                     │
 Runs ceval.c          Returns PyObject*
```

---

## 16. Important Functions (Quick Reference)

| Function                           | Purpose                                             |
|------------------------------------|-----------------------------------------------------|
| `_PyObject_Vectorcall()`           | Main internal API for calling anything.             |
| `_PyFunction_Vectorcall()`         | Handles calling Python functions (allocates frame). |
| `PyObject_Call()`                  | Legacy API – avoid if possible.                     |

---

## 17. Important Structs

| Struct             | Purpose                                          |
|--------------------|--------------------------------------------------|
| `PyFunctionObject` | Python function – wraps a `PyCodeObject`.        |
| `PyCFunctionObject`| C built‑in function – wraps a C function pointer.|
| `PyMethodDef`      | Describes a C function for export to Python.     |

---

## 18. Important Files

| File                               | Role                                               |
|------------------------------------|----------------------------------------------------|
| `Objects/call.c`                   | Vectorcall routing and dispatch.                   |
| `Objects/funcobject.c`             | Implementation of Python function objects.         |
| `Include/cpython/funcobject.h`     | Definition of `PyFunctionObject`.                  |
| `Objects/methodobject.c`           | Implementation of C function objects.              |
| `Include/cpython/methodobject.h`   | Definition of `PyCFunctionObject` and `PyMethodDef`. |

---

## 19. Code‑Reading Exercises

1. Open `Include/cpython/funcobject.h`. Find `PyFunctionObject`. Verify that it contains `func_code`, `func_globals`, and `func_closure`.

2. Open `Objects/call.c`. Search for `_PyObject_Vectorcall`. Follow the logic where it extracts the offset to the vectorcall function pointer from the object's type and executes it.

3. Open `Include/cpython/methodobject.h`. Find `PyMethodDef`. This is the struct you define when writing a C extension to expose your C functions to Python. Notice it contains `ml_name` (string name) and `ml_meth` (the actual C function pointer).

---

## 20. Understanding Questions

1. If you define `def f(a, b=5): pass`, **does the integer `5` live inside the `PyCodeObject` or the `PyFunctionObject`?** Why?

2. Why does Vectorcall pass keyword arguments as a tuple of strings (`kwnames`) alongside a C array of values, **rather than passing a `PyDictObject`?**

3. When a C function is called via Vectorcall, **who owns the references** to the arguments inside the C array: the caller or the callee?

---

## 21. Suggested Next Files

- **`Include/internal/pycore_frame.h`** – We know how a Python function is dispatched. Now we need to look deeply at the Frame it allocates to execute.
- **`Python/ceval.c`** – specifically the code handling function entry.

---

## 22. Suggested Next Lesson

**Lesson 19 – Frames & Execution Stack**  
We will study how the interpreter manages the call stack, how frames are allocated and chained, and the role of the evaluation stack inside each frame.
