# Lesson 3: The CPython Execution Pipeline

Now that we know how the C programs are compiled into the `python` binary, we look at how that binary actually processes a Python script.

This lesson is the most important architectural map you will build. Every future lesson fits into one of the stages we define here.

---

## 1. Lesson Overview

In this lesson, we trace the complete execution pipeline of CPython. We follow a string of Python source code as it is **tokenized**, **parsed**, **analyzed**, **compiled into bytecode**, and finally **executed** by the virtual machine.

**Why it matters:** If you don't understand this pipeline, you won't know where to look when debugging or contributing:
- Syntax errors → the **Parser**
- Scope bugs (e.g., `UnboundLocalError`) → the **Symbol Table**
- Execution crashes / slow performance → the **Interpreter**

You must know which component owns which phase.

**Prerequisites:** You understand CPython's repository structure (`Parser/`, `Python/`, `Objects/`).

---

## 2. Mental Model – The Compiler Pipeline

CPython does **not** execute Python source code directly. It acts as a traditional compiler that stops halfway, handing a custom binary format (bytecode) to an internal C loop (the VM).

```
       [Source Code]         "x = 10 + 20"
             │
 1. Lexing   ▼   (Parser/tokenizer.c)
          [Tokens]           NAME("x"), EQUAL, NUMBER(10), PLUS, NUMBER(20)
             │
 2. Parsing  ▼   (Parser/parser.c)
           [AST]             Assign(targets=[Name(id='x')], value=BinOp(...))
             │
 3. Analysis ▼   (Python/symtable.c)
      [Symbol Table]         "x" is a LOCAL variable
             │
 4. Compile  ▼   (Python/compile.c)
       [Code Object]         PyCodeObject (contains bytecode array)
             │
 5. Execute  ▼   (Python/ceval.c)
      [Interpreter]          while(1) { switch(opcode) { ... } }
```

---

## 3. Where We Are in the Repository

| **Directory / File**                 | **Role**                                                                 |
|--------------------------------------|--------------------------------------------------------------------------|
| `Parser/tokenizer.c`                 | Lexical analysis – turns characters into tokens.                         |
| `Parser/parser.c`                    | Syntactic analysis – builds the AST using a PEG grammar.                |
| `Python/ast.c`                       | AST construction utilities (helper functions used by the parser).       |
| `Python/symtable.c`                  | Symbol table builder – determines scope of every variable.              |
| `Python/compile.c`                   | Bytecode compiler – turns AST + symbol table into `PyCodeObject`.       |
| `Python/ceval.c`                     | The VM loop – executes bytecode instructions.                           |
| `Include/cpython/code.h`             | Definition of `PyCodeObject` (the compiled code unit).                  |

**Why they matter:** The script enters the pipeline as an array of `char` in `Parser/` and exits `Python/compile.c` as a `PyCodeObject`. That object is then fed to `Python/ceval.c`.

---

## 4. Concepts We Need First

### 4.1 Abstract Syntax Tree (AST)
A C structure consisting of nested nodes representing the grammatical structure of the code, independent of text formatting (like spaces, indentation, or comments). For example, an `if` statement is represented as an `If` node with a `test` child (condition), a `body` child, and an `orelse` child.

### 4.2 Bytecode vs Machine Code
- **C programs** compile directly to **machine code** (x86/ARM) executed by the hardware CPU.
- **Python** compiles to **bytecode** – virtual instructions (like `LOAD_FAST`, `STORE_NAME`) executed by a **software CPU** – the C `switch` loop in `ceval.c`.

---

## 5. Architecture – Strictly Phased Execution

The execution model is strictly phased; each stage has a clearly defined responsibility:

1. **Tokenizer** – Reads the C string character by character, chunking it into "tokens" (keywords, identifiers, operators, indents, newlines).

2. **Parser (PEG)** – Reads the tokens and applies Python's grammatical rules to build the **AST** in memory (using dynamically allocated C structs). Python 3.9+ uses a PEG (Parsing Expression Grammar) parser, which replaced the older LL(1) parser.

3. **Symbol Table Builder** – Walks the AST twice to figure out the scope of every variable.  
   - Is `x` local? Global? In a closure (cell variable)?  
   - The compiler must know this *before* generating bytecode so it can emit the correct `STORE_FAST` (local) or `STORE_DEREF` (closure) instructions.

4. **Compiler** – Walks the AST and the Symbol Table together, emitting an array of bytes (bytecode). It packages this array, along with constants (e.g., numbers, strings) and variable names, into a **`PyCodeObject`**.

5. **Interpreter** – Takes the `PyCodeObject`, creates an execution frame (stack), and runs a massive `for/switch` loop over the bytecode array.

---

## 6. Important Data Structures

| **Structure**              | **Defined In**                         | **Purpose**                                                                 |
|----------------------------|----------------------------------------|-----------------------------------------------------------------------------|
| `struct tok_state`         | `Parser/tokenizer.c`                   | Tracks tokenizer state: current line, character pointer, indentation stack. |
| `expr_ty` / `stmt_ty`      | `Include/internal/pycore_ast.h`        | AST node structs (generated from `Parser/Python.asdl`).                     |
| `struct symtable`          | `Python/symtable.c` (internal)         | Tracks all scopes (namespaces) in the program.                              |
| `PyCodeObject`             | `Include/cpython/code.h`               | Holds compiled bytecode, constants tuple, and variable names.               |

> **Note on AST structs:** `expr_ty` and `stmt_ty` are typedefs for pointers to AST nodes. They are generated by Python scripts from `Parser/Python.asdl`, so you should never edit them by hand.

---

## 7. Important Functions (Entry Points)

| **Function**                                      | **Location**          | **What it does**                                                                 |
|---------------------------------------------------|-----------------------|----------------------------------------------------------------------------------|
| `PyParser_ASTFromStringObject()`                  | `Parser/`             | High‑level entry point that triggers the tokenizer and parser, returning an AST. |
| `_PySymtable_Build()`                             | `Python/symtable.c`   | Takes the AST, returns a populated Symbol Table.                                 |
| `_PyAST_Compile()`                                | `Python/compile.c`    | Takes the AST and Symbol Table, returns a `PyCodeObject`.                        |
| `_PyEval_EvalFrameDefault()`                      | `Python/ceval.c`      | The heart of the VM. Takes a frame (which holds a `PyCodeObject`), executes it. |

---

## 8. Important Macros

| **Macro**           | **Used In**          | **Purpose**                                                                                       |
|---------------------|----------------------|---------------------------------------------------------------------------------------------------|
| `AST_GEN(...)`      | `Python/ast.c`       | Helper macro to dynamically allocate AST structs (used heavily in parser actions).                |
| `DISPATCH()`        | `Python/ceval.c`     | Jumps to the next bytecode instruction. Modern CPython uses "computed gotos" (GCC/Clang extension) for speed instead of a plain `switch`. |

---

## 9. Source Code Exploration – The Key Files

To trace the compiler, read these files **in order**:

1. **`Parser/Python.asdl`** – A text file that defines the AST structure in a declarative syntax.  
   Example: `stmt = Assign(expr* targets, expr value)` means an assignment statement has a list of targets and a value expression.

2. **`Python/compile.c`** – The main compiler logic.  
   - Find the function `compiler_visit_stmt()`. It contains a massive `switch (stmt->kind)` statement.  
   - If it sees an `Assign_kind`, it generates bytecode to evaluate the right‑hand side, then bytecode to store it in the left‑hand side.

3. **`Include/cpython/code.h`** – Definition of `struct PyCodeObject`.  
   - Look at `co_code` (the bytecode array, `const char *`).  
   - Look at `co_consts` (tuple of constants).  
   - Look at `co_names` (tuple of global variable names).

---

## 10. Execution Flow – Memory Ownership

Trace the C‑level memory lifecycle during execution of a Python script:

1. **Input:** A C string (source code) is passed to `PyParser_ASTFromStringObject()`.

2. **AST Generation:** The parser allocates AST nodes (`expr_ty`, `stmt_ty`) via an **arena allocator** – this allows fast bulk deallocation later.

3. **Symbol Table:** `_PySymtable_Build()` walks the AST and allocates `PyDictObject`s to hold variable names and their scopes.

4. **Compilation:** `_PyAST_Compile()` generates a `PyCodeObject`.  
   - It allocates space for the bytecode array, the constants tuple, and the names tuple.

5. **Crucial step – The AST and Symbol Table are now DESTROYED.**  
   The VM does not need them after compilation. All memory from the AST arena and symbol table is freed.

6. **Execution:** The `PyCodeObject` is passed to the interpreter.  
   - The interpreter creates a `PyFrameObject` (representing the call stack frame).  
   - The frame references the `PyCodeObject`.  
   - `_PyEval_EvalFrameDefault()` runs the bytecode.

---

## 11. Real Python Example – `x = 10 + 20`

Let's trace this simple statement through the pipeline:

### Step 1 – Tokens
```
NAME('x'), EQUAL, NUMBER(10), PLUS, NUMBER(20)
```

### Step 2 – AST (simplified C representation)
```c
// Pseudo‑C code for the AST node
Assign_kind {
    .targets = [Name(id="x")],
    .value = BinOp_kind {
        .left = Constant(10),
        .op = Add,
        .right = Constant(20)
    }
}
```

> **Note:** In practice, CPython's AST optimizer (peephole) will fold `10 + 20` into `30` at compile time, but we ignore that for this example.

### Step 3 – Bytecode Generated by `compile.c`
```
1. LOAD_CONST    10       # Push 10 onto stack
2. LOAD_CONST    20       # Push 20 onto stack
3. BINARY_OP     +        # Pop two, add, push result (30)
4. STORE_NAME    x        # Pop result, store in variable 'x'
```

### Step 4 – Interpreter (`ceval.c`) executes each instruction:
- `LOAD_CONST`: pushes a value from `co_consts` onto the stack.
- `BINARY_OP`: calls the C function for addition.
- `STORE_NAME`: looks up `'x'` in the local namespace dictionary and assigns the value.

---

## 12. Why This Design?

### Why not interpret the AST directly?
Walking a tree of C structs is slow due to pointer chasing and poor CPU cache locality. A flat array of bytes (bytecode) is extremely fast to iterate over – the CPU can predict the loop and cache the bytecode efficiently.

### Why have a separate Symbol Table phase?
In C, you declare `int x;`. In Python, you just write `x = 10`. The compiler doesn't know if `x` is local or global until it reads the **entire** function. It must walk the AST once to map all variables (Symbol Table), then walk it again to emit the correct bytecode (`STORE_FAST` for locals, `STORE_GLOBAL` for globals).

### Why a PEG parser (instead of LL(1))?
CPython used an LL(1) parser until version 3.8. LL(1) can only look ahead **one token**, making complex syntax features like pattern matching (`match`/`case`) nearly impossible to implement cleanly. PEG (Parsing Expression Grammar) allows **infinite lookahead** and is far more expressive.

---

## 13. Common Beginner Mistakes

- **Mistake:** "Python is an interpreted language, so it reads my script line‑by‑line while running."  
  **Correction:** CPython reads the **entire** script, compiles the **entire** script into a `PyCodeObject`, and **then** executes it. If there is a syntax error on line 1000, line 1 will never execute.

- **Mistake:** "Python bytecode is like Java bytecode or assembly."  
  **Correction:** Python bytecode is very **high‑level**. One instruction like `BINARY_OP` might trigger thousands of lines of C code if you are adding two custom objects that implement `__add__`.

- **Mistake:** Modifying the AST structure without regenerating the C headers.  
  **Correction:** If you change `Parser/Python.asdl`, you must run `make regen-ast` to regenerate `Include/internal/pycore_ast.h` before compiling.

---

## 14. Summary

CPython execution is a **multi‑pass compiler pipeline**:

1. Source text → **Tokens** (Tokenizer)
2. Tokens → **AST** (Parser)
3. AST → **Symbol Table** (Scope analysis)
4. AST + Symbol Table → **`PyCodeObject`** (Compiler)
5. `PyCodeObject` → **Execution** (Interpreter)

The AST, Symbol Table, and all intermediate structures are **discarded** before the bytecode runs – only the `PyCodeObject` and the frame remain.

---

## 15. Mental Model to Remember

```
Text  →  Tokens  →    AST    →  Code Object  →  VM Execution
 |         |          |              |                |
[tokenizer] [parser.c] [compile.c]    |           [ceval.c]
                                      |
                               (AST & SymTable
                                 are freed)
```

---

## 16. Important Functions (Quick Reference)

| Function                                      | Purpose                                                       |
|-----------------------------------------------|---------------------------------------------------------------|
| `PyParser_ASTFromStringObject()`              | Entry point for tokenizing + parsing → returns AST.           |
| `_PyAST_Compile()`                            | Compiles AST + symbol table → returns `PyCodeObject`.         |
| `_PyEval_EvalFrameDefault()`                  | Executes a frame; the core VM dispatch loop.                  |

---

## 17. Important Structs

| Struct              | Purpose                                                              |
|---------------------|----------------------------------------------------------------------|
| `struct symtable`   | In‑memory representation of all scopes (global, local, closure).     |
| `PyCodeObject`      | The compiled code unit – contains bytecode, constants, names.        |
| `PyFrameObject`     | Execution frame (holds stack, local variables, and pointer to code). |

---

## 18. Important Files (By Pipeline Stage)

| Stage        | File(s)                                  |
|--------------|------------------------------------------|
| Tokenizer    | `Parser/tokenizer.c`                     |
| Parser       | `Parser/parser.c`, `Parser/Python.asdl`  |
| AST helpers  | `Python/ast.c`                           |
| Symbol Table | `Python/symtable.c`                      |
| Compiler     | `Python/compile.c`                       |
| Interpreter  | `Python/ceval.c`, `Python/bytecodes.c` (or `generated_cases.c.h`) |

---

## 19. Code‑Reading Exercises

1. **Open `Python/compile.c`** – Find `compiler_visit_stmt()`. Look at the `switch (stmt‑>kind)` block. This is where the compiler decides what bytecode to emit for each AST node type (`If_kind`, `While_kind`, `Assign_kind`, etc.).

2. **Open `Include/cpython/code.h`** – Inspect `struct PyCodeObject`. Identify:
   - `co_code` – the bytecode array.
   - `co_consts` – the tuple of constants.
   - `co_names` – the tuple of global names.

3. **Open `Parser/Python.asdl`** – Find the definition of `stmt` and look at the `If` node. Notice it has a `test` (condition), a `body` (list of statements), and an `orelse` (list of statements for the `else` clause).

---

## 20. Understanding Questions

1. If you misspell a variable name (`prnt("hello")`), **which component catches the error?**  
   (The parser, the symbol table, the compiler, or the interpreter at runtime?)

2. **Why does the VM execute a `PyCodeObject` instead of just executing the AST directly?**

3. If the AST and Symbol Table are destroyed **before** the code runs, **where does the interpreter look to find the names of local variables** for debugging tracebacks?

---

## 21. Suggested Next Files to Read

- **`Python/ceval.c`** – Contains the main VM loop. Python spends >90% of its runtime inside this file.
- **`Python/bytecodes.c`** (or `Python/generated_cases.c.h` in modern versions) – Defines the actual logic for each bytecode instruction (e.g., `LOAD_FAST`, `BINARY_OP`).
