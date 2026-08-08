# CPython  Lesson 1

## Repository Structure 

Welcome to CPython internals.

The challenge of understanding Python is not learning Python syntax. The challenge is understanding:

* how source code becomes executable instructions
* how the interpreter manages objects
* how memory is allocated and released
* how C implements dynamic language features
* how the runtime communicates with the operating system

This course approaches CPython as an **architectural exploration**, not as a collection of implementation details.

The goal is to understand:

* where things live
* how execution flows
* which files control each subsystem
* how Python's high-level features map to C code

---

# 1. What Is CPython?

Python is a language specification.

CPython is the most widely used implementation of that specification.

**CPython Source Repository:** [CPython GitHub Repository](https://github.com/python/cpython?utm_source=chatgpt.com)

CPython is written mainly in:

* C
* Python
* a small amount of platform-specific code

It transforms:

```
Human Python Code
        |
        |
        v
  CPython Runtime
        |
        |
        v
   Machine Execution
```

---

# 2. The Two Main Engines of CPython

CPython can be understood as two major systems working together.

```
              Python Source (.py)
                       |
                       |
                  Compiler
                       |
                       |
              PyCodeObject
              (Bytecode)
                       |
                       |
                       v
             Virtual Machine
          (Evaluation Loop)
                       |
                       |
                    CPU
```

---

# Engine 1: The Compiler

The compiler converts Python source code into bytecode.

It performs:

* tokenization
* parsing
* AST generation
* bytecode generation

Example:

Python:

```python
x = 1 + 2
```

becomes:

```
LOAD_CONST 1
LOAD_CONST 2
BINARY_OP +
STORE_NAME x
```

---

# Engine 2: The Virtual Machine

The virtual machine executes bytecode.

Main component:

```
Python/ceval.c
```

Important function:

```c
_PyEval_EvalFrameDefault()
```

Responsibilities:

* fetch instructions
* manipulate the stack
* call C implementations
* manage execution frames
* handle exceptions

---

# 3. Complete Python Execution Pipeline

The complete journey of Python code:

```
                Python Source
                    (.py)
                      |
                      |
              Tokenizer
                      |
                      |
                  Tokens
                      |
                      |
              PEG Parser
                      |
                      |
                    AST
                      |
                      |
                 Compiler
                      |
                      |
              PyCodeObject
              (Bytecode)
                      |
                      |
              Evaluation Loop
              (Virtual Machine)
                      |
        -----------------------------
        |                           |
        |                           |
 Built-in Objects             C Extensions
 Objects/*.c                  Modules/*.c
```

---

# 4. Why Does CPython Have This Structure?

CPython implements a high-level dynamic language on top of low-level hardware.

Python provides:

* dynamic typing
* automatic memory management
* objects everywhere
* reflection
* exceptions
* garbage collection

The CPU only understands:

* machine instructions
* memory addresses
* registers

CPython creates a bridge:

```
Python Concepts
      |
      |
CPython Runtime
      |
      |
C Structures
      |
      |
Operating System
      |
      |
Hardware
```

---

# 5. CPython Repository Structure

The repository is divided by responsibility.

```
cpython/
|
|-- Include/
|-- Objects/
|-- Python/
|-- Parser/
|-- Modules/
|-- Programs/
|-- Lib/
```

---

# 6. Important Top-Level Directories

---

## `Include/`

Location:

```
Include/
```

Purpose:

Contains CPython header files.

Contains:

* public C API
* internal structures
* macros
* type definitions

Important files:

```
Include/Python.h
Include/object.h
```

---

## `Include/Python.h`

The master header file.

Used by:

* C extensions
* embedding applications

It provides access to:

* Python objects
* interpreter APIs
* runtime functions

Example:

```c
#include <Python.h>
```

---

## `Include/object.h`

Defines the foundation of Python objects.

Important structure:

```c
PyObject
```

Every Python object begins with this structure.

Conceptually:

```c
typedef struct {
    Py_ssize_t ob_refcnt;
    PyTypeObject *ob_type;
} PyObject;
```

It contains:

* reference count
* pointer to object type

---

# `Objects/`

Location:

```
Objects/
```

Contains implementations of Python's built-in objects.

Examples:

```
Objects/listobject.c
Objects/dictobject.c
Objects/longobject.c
Objects/unicodeobject.c
```

Responsible for:

* integers
* strings
* lists
* dictionaries
* tuples

---

Example:

Python:

```python
x = 10
```

Internally:

```
10
 |
 |
PyLongObject
 |
 |
Objects/longobject.c
```

---

# `Python/`

The runtime core.

This is the heart of CPython.

Contains:

```
Python/ceval.c
Python/compile.c
Python/pythonrun.c
Python/gcmodule.c
```

Responsibilities:

* interpreter startup
* bytecode execution
* compiler
* garbage collector
* thread handling

---

# `Parser/`

Contains:

* tokenizer
* PEG parser
* grammar definitions

Important files:

```
Parser/tokenizer.c
Grammar/python.gram
```

Flow:

```
Source Code
      |
      |
Tokenizer
      |
      |
Tokens
      |
      |
PEG Parser
      |
      |
AST
```

---

# `Modules/`

Contains built-in C extensions.

Examples:

```
Modules/mathmodule.c
Modules/_io/
Modules/posixmodule.c
```

These connect Python with:

* operating system APIs
* mathematics libraries
* file systems
* networking

Example:

Python:

```python
import math
```

may load:

```
Modules/mathmodule.c
```

---

# `Programs/`

Contains executable entry points.

Important file:

```
Programs/python.c
```

This contains:

```c
main()
```

The operating system starts here.

---

# `Lib/`

Python standard library.

Written mostly in Python.

Examples:

```
Lib/os.py
Lib/json/
Lib/asyncio/
Lib/threading.py
```

Unlike:

```
Objects/
Python/
```

this is not the interpreter core.

It is Python code running on top of the interpreter.

---

# 7. Important CPython Structures

---

# `PyObject`

Location:

```
Include/object.h
```

The base structure of all Python objects.

Everything is an object:

```
int
str
list
function
class
module
```

Concept:

```
          PyObject
              |
   ----------------------
   |          |         |
 PyLong   PyUnicode   PyList
```

---

# `PyTypeObject`

Represents a Python type.

Examples:

```
int
str
list
dict
```

It defines:

* methods
* memory behavior
* operations

Example:

```
PyLong_Type
      |
      |
 integer behavior
```

---

# `PyCodeObject`

Represents compiled Python code.

Contains:

* bytecode
* constants
* variable names
* source information

Flow:

```
Python Code
      |
      |
Compiler
      |
      |
PyCodeObject
```

---

# `PyFrameObject`

Represents an execution frame.

Every running function has a frame.

Example:

```python
def hello():
    print("hi")
```

Execution:

```
hello()
  |
  |
PyFrameObject
  |
  |
Bytecode execution
```

Contains:

* local variables
* stack
* instruction position

---

# 8. Internal Architecture Overview

```
                  User Python Code
                         |
                         |
                  Interpreter Startup
                         |
                         |
                 Programs/python.c
                         |
                         |
                Python/pythonrun.c
                         |
                         |
                    Compiler
                         |
                         |
                 PyCodeObject
                         |
                         |
                Evaluation Loop
                         |
                         |
                 Python/ceval.c
                         |
 ------------------------------------------------
 |                     |                         |
Objects/            Modules/                  Memory
Built-ins           C Extensions              Allocator
 |                     |                         |
int                 OS APIs                 obmalloc.c
str                 Files                   GC
list
dict
 ------------------------------------------------
                         |
                         |
                    Operating System
```

---

# 9. Python Startup Flow

When you type:

```bash
python program.py
```

the flow begins.

```
Operating System
        |
        |
Programs/python.c
        |
        |
Py_BytesMain()
        |
        |
Python/pythonrun.c
        |
        |
Py_Initialize()
        |
        |
Create Interpreter
        |
        |
Execute Python Code
```

---

# 10. Entry Point: `Programs/python.c`

File:

```
Programs/python.c
```

The executable starts here.

Simplified:

```c
int
main(int argc, char **argv)
{
    return Py_BytesMain(argc, argv);
}
```

Purpose:

* receive command-line arguments
* initialize Python
* start execution

---

# 11. Interpreter Initialization

File:

```
Python/pythonrun.c
```

Important function:

```c
Py_Initialize()
```

Responsibilities:

```
Py_Initialize()
        |
        |
        +-- Initialize memory system
        |
        +-- Create interpreter state
        |
        +-- Create thread state
        |
        +-- Initialize built-in types
        |
        +-- Setup sys module
        |
        +-- Setup builtins
```

Before user code runs, CPython creates its own world:

```
int type
str type
list type
dict type
```

---

# 12. The Heart of CPython: Evaluation Loop

File:

```
Python/ceval.c
```

Function:

```c
_PyEval_EvalFrameDefault()
```

This executes bytecode.

Conceptual loop:

```c
while (running) {

    opcode = FETCH_OPCODE();

    switch(opcode) {

        case LOAD_CONST:
            PUSH(value);
            break;

        case BINARY_OP:
            result = PyNumber_Add(a,b);
            PUSH(result);
            break;
    }
}
```

The VM repeatedly:

1. fetches instruction
2. executes operation
3. updates stack
4. moves forward

---

# 13. Example Execution

Python code:

```python
x = 1 + 2
```

---

## Step 1: Tokenization

Input:

```
x = 1 + 2
```

Tokens:

```
NAME
EQUAL
NUMBER
PLUS
NUMBER
```

---

## Step 2: Parsing

Tokens become:

```
AST

Assign
 |
 +-- Name(x)
 |
 +-- BinOp(+)
       |
       +-- 1
       +-- 2
```

---

## Step 3: Compilation

Compiler produces:

```
PyCodeObject
```

Bytecode:

```
LOAD_CONST 1
LOAD_CONST 2
BINARY_OP +
STORE_NAME x
```

---

## Step 4: Virtual Machine Execution

Stack:

Initial:

```
[]
```

---

Execute:

```
LOAD_CONST 1
```

Stack:

```
[1]
```

---

Execute:

```
LOAD_CONST 2
```

Stack:

```
[1,2]
```

---

Execute:

```
BINARY_OP +
```

Internally:

```c
PyNumber_Add(1,2)
```

Result:

```
[3]
```

---

Execute:

```
STORE_NAME x
```

Namespace:

```
x -> 3
```

Python has executed.

---

# 14. Memory Management

CPython manages memory automatically.

Major components:

```
Memory Management
        |
 -----------------------
 |                     |
Reference Counting    Garbage Collector
 |                     |
Immediate cleanup     Cyclic cleanup
```

---

# Reference Counting

Every object has:

```c
ob_refcnt
```

Example:

```
x = []
```

creates:

```
PyListObject

refcount = 1
```

When references disappear:

```
refcount = 0
```

object is freed.

---

# pymalloc

Small objects use:

```
Objects/obmalloc.c
```

Instead of:

```
malloc()
free()
```

CPython uses:

```
Arena
 |
 +-- Pools
       |
       +-- Blocks
```

Benefits:

* faster allocation
* reduced fragmentation
* fewer system calls

---

# 15. How Experts Navigate CPython

Reading CPython requires navigation tools.

---

# Tool 1: clangd + LSP

Generate:

```
compile_commands.json
```

Benefits:

* jump to definitions
* find references
* inspect structures
* navigate C code

Works with:

* VS Code
* Vim
* Neovim
* Emacs

---

# Tool 2: ripgrep

Fast source searching.

Install:

```bash
sudo apt install ripgrep
```

Find:

```
PyObject
```

Command:

```bash
rg "typedef struct.*PyObject"
```

Find evaluation loop:

```bash
rg "_PyEval_EvalFrameDefault"
```

---

# Tool 3: ctags

Generate:

```bash
ctags -R .
```

Navigate:

* functions
* structs
* macros

---

# 16. First Files To Explore

Start with:

---

## 1. Program Entry

```
Programs/python.c
```

Understand:

* executable startup

---

## 2. Interpreter Startup

```
Python/pythonrun.c
```

Understand:

* initialization sequence

---

## 3. Object System

```
Include/object.h
```

Understand:

* PyObject
* reference counting

---

## 4. Evaluation Loop

```
Python/ceval.c
```

Understand:

* bytecode execution

---

## 5. Built-in Types

```
Objects/
```

Understand:

* int
* str
* list
* dict

---

# Exercises

## Exercise 1

Open:

```
Include/object.h
```

Find:

```c
PyObject
```

Observe:

* reference counting
* type pointer
* why everything starts from this structure

---

## Exercise 2

Open:

```
Python/ceval.c
```

Find:

```c
_PyEval_EvalFrameDefault()
```

Understand:

* instruction fetching
* stack operations
* dispatch mechanism

---

## Exercise 3

Open:

```
Programs/python.c
```

Find:

```c
main()
```

Trace:

```
main()
 |
 Py_BytesMain()
 |
 Py_Initialize()
```

---

# Understanding Questions

## Question 1

Why does CPython need bytecode?

Think about:

* portability
* execution speed
* separating parsing from execution

---

## Question 2

Why is `PyObject` important?

Think about:

* every Python value
* reference counting
* dynamic typing

---

## Question 3

Where would you start if you want to understand:

"How does Python execute a function call?"

Answer:

```
Python/ceval.c
```

---
