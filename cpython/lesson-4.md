# Lesson 4: The CPython Virtual Machine

We are now entering the beating heart of CPython.

If you look at older tutorials (pre‑Python 3.12), they will tell you that `Python/ceval.c` contains a massive 3,000‑line C `switch` statement. **That is no longer true.** The modern interpreter loop has been radically re‑architected for performance and maintainability.

Let's dive into how the modern Virtual Machine actually executes your code.

---

## 1. Lesson Overview

In this lesson, we study the CPython Virtual Machine (VM). We look at how CPython takes the compiled `PyCodeObject` from Lesson 3, sets up an execution environment (a **frame**), and runs a highly optimised C loop to process the bytecode instructions.

**Why it matters:**  
> 90% of a Python program's execution time is spent inside this specific C loop.

Understanding how this loop manages the stack, dispatches instructions, and interacts with the OS thread is critical for profiling, optimising, or debugging CPython.

**Prerequisites:** You understand that Python code is compiled into a flat array of bytecode instructions (a `PyCodeObject`).

---

## 2. Mental Model – Stack Machine

CPython is a **Stack Machine**.

Unlike your physical CPU (which is a *Register Machine* that moves data between hardware registers like `RAX` and `RBX`), CPython's VM evaluates expressions by **pushing** and **popping** `PyObject *` pointers onto an array called the **Evaluation Stack**.

Think of the interpreter loop as an infinite loop that constantly tracks three things:

1. **Instruction Pointer** – Where are we in the bytecode array?
2. **Evaluation Stack** – A temporary scratchpad array for holding pointers to objects.
3. **Locals Array** – An array holding the actual local variables (like `x` or `y`).

```
[Instruction: BINARY_OP]
      │
      ▼
+-----------------+      1. POP() -> gets pointer to '20'
| PyObject * (20) |      2. POP() -> gets pointer to '10'
+-----------------+      3. Call C function: add(10, 20)
| PyObject * (10) |      4. PUSH() -> puts pointer to '30' on stack
+-----------------+
[ Evaluation Stack ]
```

---

## 3. Where We Are in the Repository

| **Directory / File**                 | **Role**                                                                 |
|--------------------------------------|--------------------------------------------------------------------------|
| `Python/ceval.c`                     | Contains the VM setup, the main loop, and the dispatch logic.            |
| `Python/bytecodes.c`                 | **DSL definition** – describes each bytecode instruction in a custom language. |
| `Python/generated_cases.c.h`         | **Generated C code** – the actual implementation of every instruction.   |
| `Include/internal/pycore_frame.h`    | Defines `_PyInterpreterFrame` – the lightweight execution frame.         |

**Why they matter:**  
`ceval.c` sets up the loop, but the actual C code that executes each instruction is now written in a custom Domain Specific Language (DSL) inside `bytecodes.c`. This DSL is processed by a Python script to produce `generated_cases.c.h` – the plain C code that the compiler actually sees.

---

## 4. Concepts We Need First

### 4.1 Computed Gotos (the "trick" behind speed)

In standard C, you dispatch opcodes with a `switch(opcode) { case LOAD_FAST: ... }`. This is slow because:
- The C compiler generates a single jump table.
- The CPU's **branch predictor** struggles with one central jump point – it cannot predict which case will be taken next.

Modern CPython (since 3.11) uses a GCC/Clang extension called **computed gotos**:

- It creates an array of memory addresses for every instruction label (`&&TARGET_LOAD_FAST`).
- At the end of every instruction, it looks up the *next* opcode and jumps directly to its memory address:  
  `goto *labels[next_opcode]`.

This completely **decentralises** the jumps, which massively improves hardware branch prediction and makes the VM 10‑20% faster.

---

## 5. Architecture – The Three Layers

1. **Thread State** – The OS thread running Python has a `_PyThreadState` struct. This struct holds a contiguous chunk of C memory used as the call stack (the "frame stack").

2. **Frame Initialisation** – When a Python function is called, a lightweight `_PyInterpreterFrame` struct is pushed onto this C memory stack. It points to the `PyCodeObject` that is about to execute.

3. **The Loop** – `_PyEval_EvalFrameDefault()` starts. It sets up the computed‑goto jump table, fetches the first opcode, and jumps to it.

4. **Instruction Execution** – The code in `generated_cases.c.h` pops arguments, calls C API functions (e.g., for adding numbers), pushes the result, and calls the `DISPATCH()` macro to jump to the next instruction.

---

## 6. Important Data Structures

### `_PyInterpreterFrame` (defined in `Include/internal/pycore_frame.h`)

This is the **execution state** for a single call (function, module, or class body).

| Field               | Purpose                                                                                  |
|---------------------|------------------------------------------------------------------------------------------|
| `f_executable`      | Pointer to the `PyCodeObject` (or a newer executable wrapper).                           |
| `localsplus`        | A **single dynamically sized C array** of `PyObject *`. This array holds **both** the local variables and the evaluation stack. |
| `stackpointer`      | A C pointer (`PyObject **`) that marks the current top of the evaluation stack inside `localsplus`. |
| `instr_ptr`         | A pointer to the current bytecode instruction (the instruction pointer).                 |
| `previous`          | Points to the caller's frame (for chaining).                                             |

> **Historical note:** Prior to Python 3.11, CPython heap‑allocated a heavy `PyFrameObject` for every function call. Now, it only allocates the lightweight `_PyInterpreterFrame` on a custom C stack. The heavy `PyFrameObject` is created **only if** Python code explicitly requests it (e.g., via `sys._getframe()`). This saves a lot of memory and allocation time.

---

## 7. Important Functions

### `_PyEval_EvalFrameDefault()`
- **Location:** `Python/ceval.c`
- **Signature:**  
  `PyObject* _PyEval_EvalFrameDefault(PyThreadState *tstate, _PyInterpreterFrame *frame, int throwflag)`
- **Purpose:** The central interpreter loop.
- **What it does:** Executes the bytecode in the given `frame` until it hits a `RETURN_VALUE` instruction or an unhandled exception. It returns the result of the function (or `NULL` if an exception occurred).

---

## 8. Important Macros

| Macro         | Purpose                                                                                             |
|---------------|-----------------------------------------------------------------------------------------------------|
| `DISPATCH()`  | Fetches the next instruction, increments the instruction pointer, and jumps (via computed goto) to the C code that implements that instruction. |
| `PUSH(v)`     | Pushes a `PyObject *` onto the evaluation stack (and increments `stackpointer`).                    |
| `POP()`       | Decrements `stackpointer` and returns the `PyObject *` that was on top.                             |

> **Ownership note:** The evaluation stack **owns strong references**. When you `PUSH`, the stack now holds a reference to the object. When you `POP`, you take ownership of that reference and must eventually call `Py_DECREF` to avoid leaks.

---

## 9. Source Code Exploration

### `Python/bytecodes.c` – The DSL
Open this file first. Search for `inst(BINARY_OP)`. You will see a C‑like DSL that defines what the instruction does:

```c
inst(BINARY_OP, (left, right -- res)) {
    res = _PyEval_BinaryOps[oparg](left, right);
}
```

This declares that `BINARY_OP`:
- Pops **two** items (`left`, `right`) from the stack.
- Pushes **one** item (`res`) back.
- The actual addition is delegated to a table of binary operation functions (`_PyEval_BinaryOps`).

### `Python/generated_cases.c.h` – The Generated C Code
This file is **generated** by a Python script that reads `bytecodes.c`. You will see the actual raw C code with the `goto` labels, and the `PUSH`/`POP` macros expanded. **You should never edit this file by hand.**

### `Python/ceval.c` – The Loop Setup
Search for `_PyEval_EvalFrameDefault`. Scroll past the massive variable declarations until you find the `DISPATCH()` macro definition and the start of the execution block, which often includes `generated_cases.c.h` directly.

---

## 10. Execution Flow – Step by Step

1. Python code calls a function (e.g., `add(10, 20)`).
2. C code allocates a new `_PyInterpreterFrame` on the thread's frame stack.
3. `_PyEval_EvalFrameDefault` is called with this new frame.
4. The `instr_ptr` is initialised to the first bytecode instruction.
5. `goto *dispatch_table[opcode]` executes (this is the computed‑goto dispatch).
6. Inside the instruction implementation (e.g., `LOAD_FAST`):
   - It reads the local variable at index `oparg`.
   - Calls `Py_INCREF` on that object (because the stack needs its own reference).
   - Calls `PUSH(obj)` to put it on the evaluation stack.
7. `DISPATCH()` is called, which fetches the **next** opcode and jumps to its implementation.
8. Repeat steps 5–7 until a `RETURN_VALUE` or `RAISE_VARARGS` is encountered.
9. On `RETURN_VALUE`, the result is popped from the stack, the frame is deallocated, and the result is returned to the caller.

---

## 11. Real Python Example

Consider this function:

```python
def add(a, b):
    return a + b
```

The compiler generates this bytecode (simplified):

```
1. LOAD_FAST   0      # Push local variable 'a' (argument 0)
2. LOAD_FAST   1      # Push local variable 'b' (argument 1)
3. BINARY_OP   0      # Pop two, add, push result
4. RETURN_VALUE       # Pop result and return it
```

When `add(10, 20)` runs:

- **`LOAD_FAST 0`** – Pushes a pointer to the `10` object.
- **`LOAD_FAST 1`** – Pushes a pointer to the `20` object.
- **`BINARY_OP 0`** – Pops `20` and `10`, calls the C function for integer addition, pushes a pointer to the new `30` object.
- **`RETURN_VALUE`** – Pops `30`, and returns it from `_PyEval_EvalFrameDefault`.

---

## 12. Why This Design?

### Why a Stack Machine?
Stack machines have **much simpler compilers** than register machines. Because all operations happen on the top of the stack, bytecode instructions don't need to encode destination registers. This makes the bytecode array **smaller and more cache‑friendly**.

### Why the DSL (`bytecodes.c`)?
As of Python 3.12, CPython introduced a "Tier 2 JIT optimizer". By defining instructions in a DSL rather than raw C macros, Python scripts can parse `bytecodes.c` to generate **both**:
- The standard `ceval.c` interpreter (as we have now).
- Specialised JIT templates (for the future).

This keeps the instruction definitions in **one place** and avoids duplication.

---

## 13. Common Beginner Mistakes

- **Mistake:** Looking for a massive `switch` statement in `ceval.c` and being confused when it's not there.  
  **Correction:** It was extracted to `generated_cases.c.h` to support the new DSL generator. You should look at `bytecodes.c` for the readable definitions.

- **Mistake:** Assuming pushing/popping from the evaluation stack uses `malloc`/`free`.  
  **Correction:** The stack is just a **pre‑allocated array of pointers**. Pushing/popping is just incrementing/decrementing a C pointer. It is incredibly fast – practically free.

- **Mistake:** Thinking Python variables hold data directly.  
  **Correction:** The locals array and the stack **only** hold pointers (`PyObject *`). The actual integers or strings live elsewhere on the C heap. The VM never copies objects – it only moves references.

---

## 14. Summary

The CPython interpreter loop is a fast C loop that iterates over bytecode instructions. It uses a custom execution frame (`_PyInterpreterFrame`) that contains a pre‑allocated C array serving as **both** local variables and the evaluation stack.

Instructions are dispatched using hardware‑friendly **computed gotos**, and the implementation of those instructions is generated from a domain‑specific language in `bytecodes.c`. This design is both performant and maintainable.

---

## 15. Mental Model to Remember

```
_PyInterpreterFrame
 ├─ instr_ptr ────────────────► [LOAD_FAST, LOAD_FAST, BINARY_OP, RETURN]
 ├─ localsplus (Array)
 │   ├─ [0] Local 'a' (ptr to 10)
 │   ├─ [1] Local 'b' (ptr to 20)
 │   ├─ [2] <Stack Top>   <── PUSH/POP manipulate this area
 │   └─ [3] <Empty>
 └─ stackpointer ─────────────► (points to index 2)
```

---

## 16. Important Functions (Quick Reference)

| Function                         | Purpose                                                       |
|----------------------------------|---------------------------------------------------------------|
| `_PyEval_EvalFrameDefault`       | The main interpreter loop – executes a single frame.          |

---

## 17. Important Structs

| Struct                | Defined In                            | Purpose                                          |
|-----------------------|---------------------------------------|--------------------------------------------------|
| `_PyInterpreterFrame` | `Include/internal/pycore_frame.h`     | Execution state for one function call.           |
| `_PyThreadState`      | `Include/internal/pycore_pystate.h`   | Holds the frame stack and current exception.     |

---

## 18. Important Files

| File                           | Role                                                           |
|--------------------------------|----------------------------------------------------------------|
| `Python/ceval.c`               | VM setup, dispatch logic, and loop entry point.                |
| `Python/bytecodes.c`           | DSL definitions for every bytecode instruction.                |
| `Python/generated_cases.c.h`   | Generated C code (do not edit).                                |
| `Include/internal/pycore_frame.h` | Definition of `_PyInterpreterFrame`.                         |

---

## 19. Code‑Reading Exercises

1. Open `Include/internal/pycore_frame.h` and find `struct _PyInterpreterFrame`. Look at the `localsplus` array at the end of the struct (it uses a C99 flexible array member).

2. Open `Python/bytecodes.c`. Find `inst(LOAD_FAST)`. Read the DSL definition. Observe how it takes a value from `GETLOCAL(oparg)` and yields it to the stack.

3. Open `Python/ceval.c`. Search for `#define DISPATCH()`. See how it uses `goto` to jump to the next instruction label.

---

## 20. Understanding Questions

1. If the evaluation stack just holds pointers, **what manages the actual memory** of the `10` and `20` objects when they are popped and discarded?  
   (Hint: think about what `POP()` does to reference counts.)

2. Why does using computed gotos (`goto *labels[opcode]`) provide **better CPU branch prediction** than a single `switch(opcode)` statement?

3. If a function calls another function, **what happens at the C level** to `_PyInterpreterFrame`? Are we allocating a new one, or re‑using the current one?

---

## 21. Suggested Next Files to Read

- **`Include/cpython/code.h`** – To look closely at the `PyCodeObject` that feeds instructions into this loop.
- **`Python/compile.c`** – To briefly see how we generated the bytecode array in the first place.

---
