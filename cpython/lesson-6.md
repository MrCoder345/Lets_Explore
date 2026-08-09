# Lesson 6: The CPython Compiler Pipeline

We have looked at the output of the compiler (`PyCodeObject`) and how it is executed by the VM (`ceval.c`). Now we go **backwards** to see how the compiler actually builds that Code Object.

This is where your systems C knowledge meets compiler theory. We are going to look at how CPython walks a tree of C structs (the AST) and flattens them into an array of bytecodes.

---

## 1. Lesson Overview

In this lesson, we study the CPython Compiler Pipeline. We trace how an Abstract Syntax Tree (AST) is:

- Analyzed for variable scopes (Symbol Table).
- Translated into a **control‑flow graph** of **Basic Blocks**.
- Finally assembled into a **`PyCodeObject`**.

**Why it matters:**  
If you want to add a new syntax feature to Python (like the `match` statement added in 3.10), or if you want to understand *why* local variables are faster than globals, you must understand how the compiler analyzes scope and emits instructions.

**Prerequisites:** You understand what an AST is conceptually (Lesson 3) and what a `PyCodeObject` is (Lesson 5).

---

## 2. Mental Model – Three‑Pass Compiler

The compiler does **not** generate bytecode in a single pass. It requires three distinct phases:

1. **Scope Analysis (Symbol Table)** – Walks the AST to answer:  
   *"Is this variable local, global, or in a closure?"*

2. **Code Generation** – Walks the AST again. Instead of emitting raw bytes directly, it emits instructions into **Basic Blocks** (chunks of sequential instructions with no jumps inside).

3. **Assembly** – Calculates the exact byte offsets for jump instructions, flattens the Basic Blocks into a single array, and creates the `PyCodeObject`.

```
       [ AST ] (Tree of C structs)
          │
          ▼
   1. _PySymtable_Build()  ──► [ Symbol Table ] (Tracks variable scopes)
          │
          ▼
   2. compiler_visit_*()   ──► [ Basic Blocks ] (Instruction sequences)
          │                           │
          │     [Block A] ──(jump)──► [Block B]
          │
          ▼
   3. assemble()           ──► [ PyCodeObject ] (Flat bytecode array)
```

---

## 3. Where We Are in the Repository

| **Directory / File**             | **Role**                                                                 |
|----------------------------------|--------------------------------------------------------------------------|
| `Python/symtable.c`              | Scope analysis – builds the Symbol Table.                                |
| `Python/compile.c`               | AST walk + instruction emission into Basic Blocks.                       |
| `Python/assemble.c`              | Resolves jump offsets, flattens blocks, creates `PyCodeObject`.          |
| `Python/flowgraph.c`             | (Older versions) Basic Block management; may be merged in modern code.   |
| `Include/internal/pycore_symtable.h` | Definition of `struct symtable`.                                       |

**Why they matter:**  
- `symtable.c` determines variable scope.  
- `compile.c` traverses the AST and generates instructions.  
- `assemble.c` calculates jump offsets and builds the final `PyCodeObject`.

---

## 4. Concepts We Need First

### 4.1 Forward Jumps
When compiling an `if` statement, you must emit a `POP_JUMP_IF_FALSE` instruction to skip the `if` body. But at the moment you emit the jump, you haven't compiled the body yet, so **you don't know the exact byte offset to jump to**.

### 4.2 Basic Blocks
To solve this, compilers emit instructions into linked lists called **Basic Blocks**. A jump points to a *Block*, not a byte offset. Only in the final "Assembly" phase, when all blocks are sized, does the compiler calculate the final byte offsets.

---

## 5. Architecture

1. **Symbol Table Pass** – The compiler calls `_PySymtable_Build()`. This creates a nested C struct hierarchy representing every namespace (module, class, function). It categorises every variable as:
   - `DEF_LOCAL` – local to the current function.
   - `DEF_GLOBAL` – explicitly declared `global`.
   - `DEF_FREE` – used in a nested function (closure).

2. **Compiler State** – A `struct compiler` context is allocated. It holds a pointer to the Symbol Table and the current Basic Block.

3. **AST Walk** – The compiler recursively calls:
   - `compiler_visit_stmt()` on every statement node.
   - `compiler_visit_expr()` on every expression node.

4. **Emission** – As it visits nodes, it uses macros like `ADDOP()` to append instructions to the current Basic Block.

5. **Assembly** – The blocks are passed to the assembler, which:
   - Computes jump offsets.
   - Builds the `co_consts` and `co_varnames` tuples.
   - Allocates the `PyCodeObject`.

---

## 6. Important Data Structures

| **Struct**           | **Defined In**                           | **Purpose**                                                                                    |
|----------------------|------------------------------------------|------------------------------------------------------------------------------------------------|
| `struct symtable`    | `Include/internal/pycore_symtable.h`     | Holds the scope information for the entire compilation unit (modules, functions, classes).     |
| `struct compiler`    | `Python/compile.c` (internal)            | Large C struct passed recursively through the AST traversal. Tracks the current block, symbol table, and error state. |
| `basicblock`         | `Python/flowgraph.c` (or `compile.c`)    | Represents a sequence of instructions. Contains a pointer to the next block and an array of `instruction` structs. |

---

## 7. Important Functions

| **Function**                     | **Location**          | **Purpose**                                                                                  |
|----------------------------------|-----------------------|----------------------------------------------------------------------------------------------|
| `_PyAST_Compile()`               | `Python/compile.c`    | Main entry point. Orchestrates symtable creation, AST walk, and assembly.                    |
| `compiler_visit_stmt()`          | `Python/compile.c`    | Massive `switch` over AST statement nodes (`If`, `While`, `Assign`). Routes to handlers.     |
| `compiler_visit_expr()`          | `Python/compile.c`    | Similar for expression nodes (`BinOp`, `Call`, `Constant`).                                  |
| `_PySymtable_Build()`            | `Python/symtable.c`   | Walks the AST and builds the Symbol Table.                                                  |
| `assemble_jump_offsets()`        | `Python/assemble.c`   | Iterates over Basic Blocks and translates block pointers into integer bytecode offsets.      |

---

## 8. Important Macros

| **Macro**                | **Purpose**                                                                                 |
|--------------------------|---------------------------------------------------------------------------------------------|
| `VISIT(c, node)`         | Expands to a call to `compiler_visit_expr()` or `compiler_visit_stmt()` depending on node type. Simplifies recursive traversal. |
| `ADDOP(c, opcode)`       | Appends a parameter‑less instruction (like `RETURN_VALUE`) to the current Basic Block.       |
| `ADDOP_I(c, opcode, arg)`| Appends an instruction with an integer argument (like `LOAD_CONST 1`) to the current block. |

---

## 9. Source Code Exploration

1. **`Python/symtable.c`** – Search for `symtable_visit_stmt`. Notice that it **does not** generate bytecode. It only looks at names and flags them. For example, if it sees `global x`, it flags `x` with `DEF_GLOBAL`.

2. **`Python/compile.c`** – Search for `compiler_if()`. This perfectly demonstrates Basic Blocks. It creates two new blocks (`next` and `end`), emits a jump to `next`, compiles the `if` body, emits a jump to `end`, then compiles the `else` body.

3. **`Python/assemble.c`** – Search for `assemble_jump_offsets`. This is the C loop that iterates over the Basic Blocks and translates block pointers into integer bytecode jumps.

---

## 10. Execution Flow – Compiling `x = 42`

Let's trace how the compiler processes a simple assignment:

1. `_PyAST_Compile()` is called with an `Assign` AST node.

2. `_PySymtable_Build()` sees `x` on the left side of an assignment. It marks `x` as `DEF_LOCAL` in the Symbol Table.

3. `compiler_visit_stmt` is called. It switches on `Assign_kind` and routes to `compiler_assign()`.

4. `compiler_assign()` calls `VISIT(c, expr, 42)` – this visits the expression `42`.

5. `compiler_visit_expr` routes to the constant handler, which emits `ADDOP_LOAD_CONST(c, 42)`.

6. The handler returns to `compiler_assign()`.

7. `compiler_assign()` checks the Symbol Table: `x` is `DEF_LOCAL`, so it emits `ADDOP_I(c, STORE_FAST, 0)` (where `0` is the index of `x` in `co_varnames`).

8. The assembler later flattens this into the `PyCodeObject`'s `co_code_adaptive` array.

---

## 11. Real Python Example – `if` Statement

Consider this Python function:

```python
def check(y):
    if y:
        return 1
    return 0
```

When `compile.c` hits the `if y:` node (an `If_kind` AST node), it does the following:

1. **Creates two Basic Blocks:** an `end` block and an `else` block (if needed).
2. **Visits the expression `y`** – emits `LOAD_FAST 0` (since `y` is a local variable).
3. **Emits a conditional jump:** `POP_JUMP_IF_FALSE` pointing to the `end` block.
4. **Visits the `return 1` statement** – emits `LOAD_CONST 1` and `RETURN_VALUE`.
5. **Sets the compiler's current block** to the `end` block.
6. **Visits the `return 0` statement** – emits `LOAD_CONST 0` and `RETURN_VALUE`.

The resulting Basic Block graph looks like:

```
[Block A] -> LOAD_FAST 0
            POP_JUMP_IF_FALSE (to Block C)
            LOAD_CONST 1
            RETURN_VALUE
            (fall through to Block B?)

[Block B] -> (if no else, this is end)
            ...

[Block C] -> LOAD_CONST 0
            RETURN_VALUE
```

The assembler later resolves the `POP_JUMP_IF_FALSE` target to the byte offset of `Block C`.

---

## 12. Why This Design?

### Why a separate Symbol Table pass?
Python does **not** have variable declarations (`int x;`). If you have `x = 10` on line 500 of a function, `x` is a local variable for the *entire* function, even on line 1. The compiler cannot know this if it generates bytecode on the first pass – it must map the entire scope first.

### Why use C Macros (`VISIT`, `ADDOP`) so heavily?
The AST traversal requires extreme amounts of error checking. If memory allocation fails halfway through compiling a function, CPython must abort and return `NULL`. The macros hide the repetitive `if (ret < 0) return 0;` boilerplate, making the C code readable and maintainable.

### Why Basic Blocks instead of direct bytecode emission?
- **Forward jumps** – you don't know the target offset until the body is compiled.
- **Optimisation** – Basic Blocks enable peephole optimisations (like constant folding) and, more recently, the adaptive interpreter's inline caches.

---

## 13. Common Beginner Mistakes

- **Mistake:** Thinking `compile.c` outputs a `PyCodeObject` directly during the AST traversal.  
  **Correction:** It outputs a graph of Basic Blocks. `assemble.c` is responsible for building the actual `PyCodeObject`.

- **Mistake:** Confusing `LOAD_FAST` (local) with `LOAD_GLOBAL` (global).  
  **Correction:** The difference is entirely decided by `symtable.c`. If `symtable.c` sees an assignment to a variable anywhere in the function, it becomes a local, and `compile.c` emits `LOAD_FAST`.

- **Mistake:** Modifying the AST structure without regenerating the C headers.  
  **Correction:** If you change `Parser/Python.asdl`, you must run `make regen-ast` to regenerate the AST C structs before compiling.

---

## 14. Summary

The CPython compiler is a **multi‑pass system**:

1. **`symtable.c`** identifies variable scopes.
2. **`compile.c`** walks the AST and emits instructions into a control‑flow graph of Basic Blocks.
3. **`assemble.c`** resolves jump offsets, builds the constants/names tuples, and packages everything into an immutable `PyCodeObject`.

---

## 15. Mental Model to Remember

```
Pass 1: Read the map  (Symtable – Who is local? Who is global?)
Pass 2: Draw the graph (Compile – Emit instructions into Basic Blocks)
Pass 3: Flatten the road (Assemble – Calculate jumps, create PyCodeObject)
```

---

## 16. Important Functions (Quick Reference)

| Function                 | Purpose                                                       |
|--------------------------|---------------------------------------------------------------|
| `_PySymtable_Build()`    | Builds the Symbol Table from the AST.                         |
| `_PyAST_Compile()`       | Main entry point – orchestrates all phases.                   |
| `compiler_visit_stmt()`  | AST visitor for statements.                                   |
| `compiler_visit_expr()`  | AST visitor for expressions.                                  |

---

## 17. Important Structs

| Struct           | Purpose                                                               |
|------------------|-----------------------------------------------------------------------|
| `struct symtable`| Holds scope information for all namespaces.                           |
| `struct compiler`| Context passed during AST walk – holds current block, symbol table.   |
| `basicblock`     | A sequence of instructions with no internal jumps.                    |

---

## 18. Important Files (By Phase)

| Phase               | Files                                      |
|---------------------|--------------------------------------------|
| Symbol Table        | `Python/symtable.c`                        |
| Code Generation     | `Python/compile.c`                         |
| Assembly            | `Python/assemble.c` (or `flowgraph.c`)     |
| AST definitions     | `Parser/Python.asdl` (source), generated headers in `Include/internal/` |

---

## 19. Code‑Reading Exercises

1. Open `Python/symtable.c`. Search for `symtable_visit_expr` and look at how it handles a `Name_kind` node. Notice how it flags the name based on the current context, but generates no execution code.

2. Open `Python/compile.c`. Search for `#define ADDOP`. Notice how it wraps a call to a function that adds an instruction to the current Basic Block, and includes error checking.

3. Open `Python/compile.c` and find `compiler_while()`. Observe the creation of the `loop` and `end` blocks, and how `ADDOP_JUMP` is used to create the control flow graph.

---

## 20. Understanding Questions

1. If Python required variable declarations (e.g., `let x = 10;`), **would the `symtable.c` pass still be strictly necessary**, or could we compile in one pass?

2. During the AST walk, the compiler emits a jump to a Basic Block. **Why can't it just emit the integer byte‑offset immediately?**

3. Who owns the memory for the Basic Blocks? Is it leaked, garbage collected, or manually freed after the `PyCodeObject` is created?  
   *(Hint: The AST and Basic Blocks are intermediate representations – they are freed after assembly.)*
