# Lesson 7: The Parser (Tokenizer + PEG)

We have worked our way backward from the Virtual Machine (`ceval`) to the Compiler, and now we arrive at the very front door of CPython: the **Parser**.

If you want to introduce a new operator or keyword to the Python language, this is exactly where you must start.

---

## 1. Lesson Overview

In this lesson, we study the **PEG (Parsing Expression Grammar) parser** and the **Tokenizer**. We learn how raw string text is:

- Chunked into logical **tokens** (identifiers, operators, indents, etc.) by the Tokenizer.
- Matched against grammar rules by the PEG parser to build the Abstract Syntax Tree (AST).

**Why it matters:**  
The parser dictates what is valid Python syntax and what raises a `SyntaxError`. Understanding this layer is essential if you want to understand:

- How Python handles indentation (the “invisible braces”).
- How to contribute to the language's grammar definition.

**Prerequisites:** You understand what an AST is, and that the compiler translates it into bytecode (Lesson 6).

---

## 2. Mental Model – Two‑Stage Frontend

Parsing in CPython is a **two‑step dance** between a **Tokenizer** and a **Parser**.

1. **The Tokenizer (hand‑written C)** – Acts as a lexer. It reads the raw C string character by character. Its most magical job is **tracking spaces**. When it sees an increase in leading spaces, it emits an invisible `INDENT` token. When spaces decrease, it emits a `DEDENT` token.

2. **The Parser (generated C)** – Consumes the token stream. It consists of hundreds of C functions, each representing one grammar rule (e.g., `assignment`). If a rule fails to match the tokens, the parser **“backtracks”** (rewinds the token stream) and tries the next rule.

```
"if x:\n    pass" (Raw text)
       │
       ▼
 [ Tokenizer ] ──► (IF, NAME('x'), COLON, NEWLINE, INDENT, PASS, DEDENT)
       │
       ▼
 [ PEG Parser ] ──(Tries rule A, fails) ──(Tries rule B, succeeds)
       │
       ▼
 [ AST Structs ] (e.g., If_kind node)
```

---

## 3. Where We Are in the Repository

| **Path / File**                       | **Role**                                                                 |
|---------------------------------------|--------------------------------------------------------------------------|
| `Grammar/python.gram`                 | **The source of truth** – human‑readable grammar definition.             |
| `Parser/tokenizer.c`                  | Hand‑written C lexer that produces tokens from characters.               |
| `Parser/parser.c` (generated)         | Auto‑generated C code that implements the PEG grammar.                   |
| `Tools/peg_generator/`                | Python scripts that read `python.gram` and produce `parser.c`.           |
| `Parser/pegen.h`                      | Data structures for the parser (tokens, state, arena).                   |

**Why they matter:**  
- `python.gram` is where the syntax is defined by humans.  
- `tokenizer.c` breaks text into chunks.  
- `parser.c` is the massive, auto‑generated C file that actually builds the AST.

---

## 4. Concepts We Need First

### 4.1 LL(1) vs PEG
Prior to Python 3.9, CPython used an **LL(1)** parser. LL(1) can only look ahead **one token**. This made complex grammar (like the `match` statement) nearly impossible.  
PEG parsers have **infinite lookahead**. They can read 50 tokens, realise a rule doesn't match, and backtrack to try something else.

### 4.2 Memoization (Packrat Parsing)
Infinite backtracking can cause exponential execution time. To fix this, PEG parsers use a **cache** (memoization). If the parser tries to evaluate the `expression` rule at token index 15, it caches the result. If it backtracks and tries `expression` at index 15 again later, it returns the cached C struct instantly in **O(1)** time.

### 4.3 Arena Allocation
Parsing allocates thousands of tiny C structs (AST nodes). Calling `malloc()` and `free()` for every node is too slow and risks memory leaks. Instead, CPython uses an **Arena Allocator** – it allocates a massive block of memory upfront, hands out pointers into it, and when compilation is done, it frees the entire block at once.

---

## 5. Architecture

1. **Grammar Definition** – A human edits `Grammar/python.gram`.

2. **Code Generation** – A Python script (`Tools/peg_generator/pegen`) reads `python.gram` and outputs `Parser/parser.c`.

3. **Runtime Initialization** – When `python` runs a script, it initialises a `tok_state` struct (the tokenizer) and a `Parser` struct (the parsing state).

4. **Parsing** – The parser calls `_PyPegen_fill_token()` to pull tokens from `tok_state`. It recursively calls generated C functions (like `assignment_rule(p)`) to match tokens.

5. **AST Creation** – When a rule completely matches, the generated C code calls a helper function to allocate the appropriate AST struct in the memory Arena.

---

## 6. Important Data Structures

| **Struct**          | **Defined In**         | **Purpose**                                                                                  |
|---------------------|------------------------|----------------------------------------------------------------------------------------------|
| `struct tok_state`  | `Parser/tokenizer.h`   | Lexer state – holds current character buffer, current line number, and indentation stack.    |
| `Token`             | `Parser/pegen.h`       | Represents a single token – contains integer type (`NAME`, `NUMBER`, etc.), string value, and line/column numbers. |
| `Parser`            | `Parser/pegen.h`       | PEG parser state – holds fetched tokens, current index, memory arena, and memoization cache. |

---

## 7. Important Functions

| **Function**                           | **Location**           | **Purpose**                                                                       |
|----------------------------------------|------------------------|-----------------------------------------------------------------------------------|
| `PyTokenizer_Get(struct tok_state *)`   | `Parser/tokenizer.c`   | Core tokenizer loop – reads characters and returns the next `Token`.              |
| `_PyPegen_run_parser(Parser *p)`       | `Parser/parser.c`      | Entry point for the generated parser.                                             |
| `assignment_rule(Parser *p)`           | Generated in `parser.c`| Implements the logic: “Try to match a NAME, then an EQUAL, then an EXPRESSION.”   |

---

## 8. Important Macros

The generated `parser.c` relies heavily on C macros to handle backtracking cleanly. As a C developer, you don’t need to memorise them. The key idea is:

- If a token doesn't match, the macro resets the `p->mark` (token index) and returns `NULL`.

---

## 9. Source Code Exploration

1. **`Grammar/python.gram`** – This is where you actually read the grammar.  
   Look at the `assignment` rule. You will see something like:
   ```
   assignment: NAME '=' expression { _PyAST_Assign(...) }
   ```
   This means: “If you see a name, an equals sign, and an expression, execute this C code to build an Assign AST node.”

2. **`Parser/tokenizer.c`** – Look at `tok_get()`. This is a massive, hand‑written C state machine that processes characters. Search for where it handles `\n` and spaces to emit `INDENT` or `DEDENT`.

3. **`Parser/parser.c`** – **Do not edit this file.** Open it just to see what the Python generator produced. Find `assignment_rule`. You will see it translates the PEG grammar into a sequence of C `if` statements and token lookups.

---

## 10. Execution Flow – Parsing `x = 10`

Let's trace the path:

1. CPython calls `PyParser_ASTFromStringObject()`.
2. An arena is allocated for the AST.
3. `tok_state` is initialised with the string `"x = 10"`.
4. The parser calls the top‑level grammar rule (`file_rule`).
5. `file_rule` calls `statement_rule`, which calls `assignment_rule`.
6. `assignment_rule` asks for the next token. `tokenizer.c` reads `x` and returns `NAME`.
7. `assignment_rule` asks for the next token. `tokenizer.c` reads `=` and returns `EQUAL`.
8. `assignment_rule` calls `expression_rule`, which matches the `NUMBER` 10.
9. Because all parts matched, `assignment_rule` allocates an `Assign` C struct in the arena and returns its pointer up the call stack.

---

## 11. Real Python Example – Syntax Error

Consider:

```python
x = = 10
```

1. The tokenizer emits: `NAME('x')`, `EQUAL`, `EQUAL`, `NUMBER(10)`.
2. The parser enters `assignment_rule`.
3. It matches `NAME` and the first `EQUAL`.
4. It calls `expression_rule`.
5. `expression_rule` looks at the next token (`EQUAL`). It expects a number, string, or name. It fails and returns `NULL`.
6. The parser backtracks, tries every other possible rule, and ultimately fails.
7. CPython looks at the furthest token the parser reached before failing, and raises a `SyntaxError` pointing to the second `=`.

---

## 12. Why This Design?

### Why is the tokenizer hand‑written C, but the parser generated C?
Tokenizing characters is highly performance‑sensitive and involves Python‑specific quirks (indentation logic, f‑string nesting). Hand‑writing it allows for extreme optimisation. The grammar, however, changes frequently – writing a PEG parser by hand in C is error‑prone and unmaintainable. Generating it from a clean `.gram` file gives the best of both worlds.

### Why Arena Allocation?
A simple script can generate 10,000 AST nodes. `malloc`ing 10,000 times fragments the heap and is slow. By allocating one 1 MB chunk (an arena) and just advancing a pointer to hand out structs, node creation is nearly instant. When the bytecode compiler finishes, CPython frees the single 1 MB chunk, instantly reclaiming all 10,000 nodes without walking the tree.

---

## 13. Common Beginner Mistakes

- **Mistake:** Modifying `Parser/parser.c` to add a new Python keyword.  
  **Correction:** Any `make` command will instantly overwrite your changes. You must modify `Grammar/python.gram` and run `make regen-pegen`.

- **Mistake:** Trying to define how indentation works inside the PEG grammar.  
  **Correction:** The PEG parser doesn't know what a space is. `tokenizer.c` converts raw indentation spaces into discrete `INDENT` and `DEDENT` tokens. To the parser, an `INDENT` is just a structural token like a `{` brace in C.

---

## 14. Summary

The frontend of CPython consists of:

- A hand‑written C **Tokenizer** (`tokenizer.c`) that handles characters and indentation.
- A generated **PEG parser** (`parser.c`) that matches token streams against grammar rules.

The parser uses infinite lookahead with memoization to remain fast, and allocates AST structs in a fast memory Arena that is completely thrown away once compilation to bytecode is finished.

---

## 15. Mental Model to Remember

```
Source Text 
    │
    ▼
Tokenizer.c (Hand-written C state machine, counts spaces)
    │
  Tokens (INDENT, NAME, EQUAL)
    │
    ▼
Parser.c (Generated C functions, tries rules, backtracks)
    │
    ▼
Arena Memory (AST structs created in bulk, freed in bulk)
```

---

## 16. Important Functions (Quick Reference)

| Function                    | Purpose                                               |
|-----------------------------|-------------------------------------------------------|
| `PyTokenizer_Get()`         | Fetches the next token from the input stream.         |
| `_PyPegen_run_parser()`     | Entry point that starts the parsing process.          |

---

## 17. Important Structs

| Struct            | Purpose                                                |
|-------------------|--------------------------------------------------------|
| `struct tok_state`| Tokenizer state – character buffer, line number, etc.  |
| `Parser`          | PEG parser state – token stream, arena, memoization.   |
| `Token`           | Single token with type, value, and line/column info.   |

---

## 18. Important Files

| File                                | Role                                                       |
|-------------------------------------|------------------------------------------------------------|
| `Grammar/python.gram`               | The human‑readable grammar definition (source of truth).   |
| `Parser/tokenizer.c`                | The lexer – turns characters into tokens.                  |
| `Tools/peg_generator/`              | Python scripts that generate `parser.c` from the grammar.  |
| `Parser/parser.c` (generated)       | The actual C code that performs parsing – do not edit.     |

---

## 19. Code‑Reading Exercises

1. Open `Grammar/python.gram`. Search for the `while_stmt` rule. See how it clearly defines that a `while` statement consists of the keyword `while`, an expression, a colon, and a block.

2. Open `Parser/tokenizer.c`. Search for the `tok_get()` function. Scroll through the massive `switch(c)` statement to see how it identifies integers versus strings versus operators.

3. Open `Parser/pegen.h`. Find the `Parser` struct. Identify the `arena` field (which manages memory) and the array used for memoization.

---

## 20. Understanding Questions

1. If a PEG parser has infinite lookahead, **what prevents it from taking exponential time ($O(2^N)$)** to parse a deeply nested expression?

2. Why does CPython use an **Arena Allocator** for AST nodes instead of calling `PyMem_Malloc` for each node?

3. If you wanted to add a new operator like `<=>` to Python, **would you need to modify the Tokenizer, the Grammar, or both**?

---

## 21. Suggested Next Files to Read

- **`Include/object.h`** – Now that we have covered the entire compiler pipeline from text to execution, it is time to look at the data the VM is actually manipulating – the Python object system itself.

---
