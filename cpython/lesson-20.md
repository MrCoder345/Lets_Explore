# Lesson 20: Namespaces, Scopes & Symbol Tables

We have studied the Compiler Pipeline (Lesson 6) and the Execution Frame (Lesson 19). Now, we must connect them.

How does the compiler know whether a variable should be compiled as a fast C array lookup (`LOAD_FAST`) or a slow dictionary lookup (`LOAD_GLOBAL`)? And more importantly, if a variable is used in a nested function (a closure), how does it survive when the outer function's frame is destroyed?

The answer is the **Symbol Table** and **Cell Objects**.

---

## 1. Lesson Overview

In this lesson, we study namespaces, lexical scoping, and closures in CPython. We explore:

- How the compiler builds the **Symbol Table** to categorise variables.
- How CPython uses a special heap‑allocated struct called `PyCellObject` to safely share variables between different execution frames.

**Why it matters:**  
Scope bugs (like `UnboundLocalError`) confuse many Python programmers. As a systems engineer, understanding the C‑level difference between Locals, Globals, and Free variables allows you to write perfectly optimised closures and understand the VM's memory behaviour.

**Prerequisites:** You understand `localsplus` (Lesson 19) and the difference between the Compiler and the VM (Lesson 3).

---

## 2. Mental Model

Python uses the **LEGB Rule** (Local, Enclosing, Global, Built‑in) for variable resolution. But at the C level, these are **not** equal operations.

The Symbol Table maps out the entire script *before* bytecode is generated, translating LEGB into specific bytecode instructions that target different C memory structures:

```
[ Variable Access in Python ]
             │
 ┌───────────┼────────────┐
 ▼           ▼            ▼
Local      Global      Enclosing (Closure)
 │           │            │
LOAD_FAST  LOAD_GLOBAL  LOAD_DEREF
 │           │            │
 ▼           ▼            ▼
C Array    C Dict       PyCellObject
(locals)   (f_globals)  (Heap struct holding PyObject *)
```

---

## 3. Where We Are in the Repository

| **Path / File**                           | **Role**                                                                      |
|-------------------------------------------|-------------------------------------------------------------------------------|
| `Python/symtable.c`                       | The compiler's scope analyser – builds the Symbol Table from the AST.         |
| `Include/internal/pycore_symtable.h`      | Internal definitions for `struct symtable` and symbol flags.                  |
| `Objects/cellobject.c`                    | Implementation of `PyCellObject` – the runtime closure wrapper.               |
| `Include/cpython/cellobject.h`            | Definition of `PyCellObject` and its macros.                                  |

**Why they matter:**  
`symtable.c` makes the decisions at compile time. `cellobject.c` provides the C struct that makes closures physically possible at runtime.

---

## 4. Concepts We Need First

### 4.1 The Closure Problem
In C, if an inner function tries to access a local variable of an outer function after the outer function has returned, you get a **Use‑After‑Free** segfault – the stack frame is gone.

### 4.2 The Cell Solution
Python solves this by detecting the shared variable at compile time. Instead of putting the variable directly into the fast `localsplus` array:

- The outer function allocates a `PyCellObject` on the heap.
- Both the outer function and the inner function hold pointers to this Cell.
- The Cell holds the actual `PyObject *` data.

---

## 5. Architecture

1. **Symbol Table Pass (`symtable.c`)** – The compiler walks the AST. Every time it enters a new block (function/class/module), it creates a `_symtable_entry`.

2. **Tagging Names** – When it sees `x = 10`, it tags `x` as `DEF_LOCAL`. If it sees `global y`, it tags `y` as `DEF_GLOBAL`.

3. **Resolving Free Variables** – After walking the AST, it performs a bottom‑up pass. If an inner function uses a variable `z` that it didn't define (`DEF_FREE`), the compiler traces up to the enclosing function and changes that function's `z` tag from `DEF_LOCAL` to `DEF_CELL`.

4. **Bytecode Generation**:
   - `DEF_LOCAL` → emits `LOAD_FAST` / `STORE_FAST`.
   - `DEF_GLOBAL` → emits `LOAD_GLOBAL` / `STORE_GLOBAL`.
   - `DEF_CELL` / `DEF_FREE` → emits `LOAD_DEREF` / `STORE_DEREF`.

5. **Runtime Execution** – When `make_closure()` runs, it allocates `PyCellObject`s and passes them to the inner function's `PyFunctionObject`.

---

## 6. Important Data Structures

### `struct symtable` & `_symtable_entry`
Massive C structs used during compilation to track blocks, name dictionaries, and optimisation flags. They contain bitfields like `DEF_LOCAL`, `DEF_GLOBAL`, `DEF_FREE`, and `DEF_CELL` that encode the scope of every variable.

### `PyCellObject` – The Runtime Closure Wrapper
```c
typedef struct {
    PyObject_HEAD
    PyObject *ob_ref;       // Pointer to the actual object being shared
} PyCellObject;
```
That's it – just a `PyObject` header and a single pointer. This tiny struct is the key to closures.

---

## 7. Important Functions

| Function                                | Purpose                                                                                       |
|-----------------------------------------|-----------------------------------------------------------------------------------------------|
| `_PySymtable_Build()`                   | Creates the `symtable` context and starts the AST walk.                                      |
| `symtable_enter_block()`                | Pushes a new scope onto the compiler's stack.                                                 |
| `PyCell_New(PyObject *obj)`             | Allocates a new Cell object holding a strong reference to `obj`.                              |
| `PyCell_Get(PyObject *cell)`            | Returns a **new strong reference** to the object inside the cell.                             |
| `PyCell_Set(PyObject *cell, PyObject *value)` | Sets the cell to hold `value` (manages refcounts safely).                                |

---

## 8. Important Macros

| Macro                    | Purpose                                                                                                 |
|--------------------------|---------------------------------------------------------------------------------------------------------|
| `PyCell_GET(cell)`       | Fast macro that returns a **borrowed reference** to `cell->ob_ref`. Use with caution.                   |
| `PyCell_SET(cell, value)`| Fast macro that overwrites the pointer inside the cell. **Warning:** Does *not* decref the old value – use `PyCell_Set` for safe refcounting. |

---

## 9. Source Code Exploration

1. **`Include/internal/pycore_symtable.h`** – Search for `#define DEF_GLOBAL`. Look at the bit flags (`DEF_LOCAL`, `DEF_PARAM`, `DEF_FREE`, `DEF_CELL`). This is the vocabulary of scoping.

2. **`Include/cpython/cellobject.h`** – Search for `PyCellObject`. Observe how simple it is – just a header and a single pointer (`ob_ref`).

3. **`Python/symtable.c`** – Search for `analyze_cells`. This is the brilliant recursive function that links inner functions' `DEF_FREE` variables to outer functions' `DEF_CELL` variables.

---

## 10. Execution Flow – Compiling and Running a Closure

Let's trace:

```python
def outer():
    x = 10
    def inner():
        return x
    return inner
```

### Compile Time
1. **Symtable Pass:**
   - `outer()` is a block. `x` is assigned, so it starts as `DEF_LOCAL`.
   - `inner()` is a block. `x` is read but not assigned. It is marked `DEF_FREE` (a free variable).
   - `analyze_cells` sees `inner` wants `x`. It looks at `outer` and upgrades `outer`'s `x` from `DEF_LOCAL` to `DEF_CELL`.

2. **Compilation Pass:**
   - `outer` emits `MAKE_CELL` for `x`.
   - `inner` emits `LOAD_DEREF` for `x`.

### Runtime
3. `outer()` runs. The VM hits `MAKE_CELL`, creates a `PyCellObject`, points it to `10`, and stores the Cell in `outer`'s `localsplus` array.

4. `outer()` creates the `inner` function object, putting a pointer to the Cell into `inner->func_closure`.

5. `outer()` returns. **Its frame is destroyed!** But the `PyCellObject` lives on the heap.

6. The caller executes `inner()`.
   - `inner` runs. It hits `LOAD_DEREF`.
   - It reaches into its `func_closure`, grabs the `PyCellObject`, extracts the `10`, and returns it.

**Safe and sound** – no Use‑After‑Free!

---

## 11. Real Python Example – Observing Cells

You can observe the cell objects directly from Python using the `__closure__` attribute:

```python
def outer():
    x = 10
    def inner():
        return x
    return inner

func = outer()
print(func.__closure__)          # (<cell at 0x...: int object at 0x...>,)
print(func.__closure__[0].cell_contents)   # 10
```

This proves that the `PyFunctionObject` (Lesson 18) is actively holding strong references to `PyCellObject`s, keeping `x` alive long after `outer()`'s frame has been destroyed.

---

## 12. Why This Design?

### Why does `x = 10` shadow globals by default?
In Python, if you assign to a variable, it is automatically `DEF_LOCAL`. If Python didn't do this, every assignment would require searching the global dict, the built‑in dict, and all closures – which would be incredibly slow and lead to accidental mutations of global state. The compiler enforces `DEF_LOCAL` for assignments unless you explicitly use the `global` or `nonlocal` keywords.

### Why use `PyCellObject` instead of keeping the outer frame alive?
If the inner function kept the entire outer `_PyInterpreterFrame` alive, it would create massive memory leaks. A frame might have 50 local variables. If the closure only uses 1, using a `PyCellObject` allows the other 49 variables to be safely deallocated when the outer function returns.

---

## 13. Common Beginner Mistakes

- **Mistake:** `UnboundLocalError`
  ```python
  x = 10
  def f():
      print(x)   # Fails here!
      x = 20
  ```
  **Correction:** Programmers think `print(x)` will read the global `10`. It won't. The Symbol Table pass sees `x = 20` later in the function. It tags `x` as `DEF_LOCAL` for the *entire* function block. The compiler emits `LOAD_FAST` for the `print(x)`. At runtime, the local array slot is empty, triggering `UnboundLocalError`.

- **Mistake:** Late binding in loops
  ```python
  funcs = [lambda: i for i in range(3)]
  ```
  **Correction:** All three lambdas point to the **exact same** `PyCellObject` for `i`. When the loop finishes, the cell contains `2`. When the lambdas execute, they all look inside that shared cell and print `2`.

---

## 14. Summary

Namespaces in CPython are dynamically resolved at compile time by the Symbol Table:

- **Local variables** → fast C arrays (`localsplus`).
- **Global variables** → dictionary lookups (`f_globals`).
- **Closure variables** → heap‑allocated `PyCellObject`s.

Cells provide a layer of indirection, allowing multiple functions to share mutable state safely even after the creating function's execution frame has been destroyed.

---

## 15. Mental Model to Remember

```
(Compiler Symtable Tags)       (Runtime Memory Location)
      DEF_LOCAL     ────────►   localsplus[index] (Fastest)
      DEF_GLOBAL    ────────►   f_globals dict    (Slower, hashing)
      DEF_CELL/FREE ────────►   PyCellObject      (Heap pointer)
```

---

## 16. Important Functions (Quick Reference)

| Function                 | Purpose                                               |
|--------------------------|-------------------------------------------------------|
| `_PySymtable_Build()`    | Builds the Symbol Table from the AST.                 |
| `PyCell_New()`           | Allocates a new Cell object.                          |
| `PyCell_Get()`           | Returns a strong reference to the cell's contents.    |
| `PyCell_Set()`           | Safely sets a cell's contents (manages refcounts).    |

---

## 17. Important Structs

| Struct             | Purpose                                                         |
|--------------------|-----------------------------------------------------------------|
| `struct symtable`  | Compiler context for scope analysis.                            |
| `_symtable_entry`  | Represents a single scope (function/class/module).              |
| `PyCellObject`     | Runtime wrapper for a shared variable (used in closures).       |

---

## 18. Important Files

| File                               | Role                                                         |
|------------------------------------|--------------------------------------------------------------|
| `Python/symtable.c`                | Symbol Table implementation – scope analysis.                |
| `Include/internal/pycore_symtable.h` | Internal definitions for symbol table flags and structs.   |
| `Objects/cellobject.c`             | `PyCellObject` implementation – creation, get, set.          |
| `Include/cpython/cellobject.h`     | Definition of `PyCellObject` and its macros.                 |

---

## 19. Code‑Reading Exercises

1. Open `Python/symtable.c`. Search for the string `"global"`. Find the `symtable_add_def` call and see how it applies the `DEF_GLOBAL` flag to the symbol table entry when it parses a `global` statement.

2. Open `Include/cpython/cellobject.h`. Verify the fields of `PyCellObject`.

3. Open `Objects/cellobject.c`. Search for `PyCell_Set`. Observe how it safely manages reference counting: it uses a temporary pointer, sets the new value, and then calls `Py_XDECREF` on the old value to prevent Use‑After‑Free bugs during assignment.

---

## 20. Understanding Questions

1. If you write a C extension, you cannot access a Python function's `localsplus` array directly. If you want to modify a caller's local variable, **why is this structurally impossible** in modern CPython?

2. If an inner function reads a variable from a `PyCellObject`, **why does the bytecode `LOAD_DEREF` push a *Strong Reference*** to the evaluation stack instead of a *Borrowed Reference*?  
   *(Hint: What if another thread executes `x = 20` in the outer function while the inner function is running?)*

3. The compiler converts local assignments to `LOAD_FAST` and `STORE_FAST`. If a developer uses `globals()["my_var"] = 10`, **does this affect the Symbol Table?** Why or why not?
