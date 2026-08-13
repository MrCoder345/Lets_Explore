# OpenJDK Internals: Day 9 – Execution Engine & Bytecode Interpreter

You have verified bytecode in Metaspace, memory is laid out, and an OS thread is ready. Now: **how does HotSpot actually execute the bytecodes?**

If you were to write a simple JVM interpreter in C++, you'd likely write a giant `while` loop with a `switch` statement over the opcodes. That's called a *naïve interpreter* – and it's slow, because modern CPUs hate unpredictable jumps.

HotSpot does something much smarter. It uses a **Template Interpreter**. During JVM startup, it dynamically generates **optimised machine code (assembly) stubs** for every single bytecode instruction, writes them into the Code Cache, and wires them together. When you run Java code in interpreted mode, you are actually running hand‑crafted assembly, not C++.

In this lesson, we’ll dissect the Template Interpreter, see how bytecodes are turned into machine code at startup, and trace how the interpreter falls back to C++ for complex operations.

---

## 1. The Big Picture (Mental Model)

The execution context relies on dedicated CPU registers pointing to key data structures, and a dispatch table generated during boot.

```
       1. The Interpreter Frame (on the thread's native OS stack)
┌────────────────────────────────────────────────────────────────────────┐
│ [ Local Variables Array ]  <-- CPU register (e.g., R14 on x86) points here
│  0: "this"                                                             │
│  1: int x (10)                                                         │
│                                                                        │
│ [ Operand Stack ]          <-- CPU register (e.g., RSP) points to top  │
│  (Top) int 20                                                          │
│  (Bot) int 10                                                          │
│                                                                        │
│ [ Frame State ]                                                        │
│  - Method* (pointer to C++ Method in Metaspace)                        │
│  - BCP (Bytecode Pointer)  <-- CPU register (e.g., R13) points here    │
└────────────────────────────────────────────────────────────────────────┘
                               │
            (BCP reads next byte: 0x60 'iadd')
                               ▼
       2. The Dispatch Table (generated in Code Cache at startup)
┌────────────────────────────────────────────────────────────────────────┐
│ [Opcode 0x00: nop ]  --> [ Asm stub: 0x90 ... Jmp to next ]            │
│ ...                                                                    │
│ [Opcode 0x60: iadd]  --> [ Asm stub: Pop RAX, Pop RBX, Add RAX, RBX,   │
│                                      Push RAX, Jmp to next ]           │
│ ...                                                                    │
│ [Opcode 0xBB: new ]  --> [ Asm stub: Call C++ InterpreterRuntime::_new ]
└────────────────────────────────────────────────────────────────────────┘
```

---

## 2. JVM Specification vs. HotSpot Implementation

| Aspect | Spec says... | HotSpot does... |
| :------ | :----------- | :-------------- |
| **Local variables & operand stack** | Must behave as a stack machine. | Implements the exact semantics, but via generated assembly. |
| **Bytecode effect** | Defined for each opcode. | HotSpot generates CPU‑specific assembly to match those effects. |
| **Execution model** | Not specified. | Uses **threaded dispatch** – each stub jumps directly to the next, no central loop. |

---

## 3. Where the Code Lives (Directories)

| Path | Purpose |
| :--- | :--- |
| `src/hotspot/share/interpreter/` | Platform‑independent C++ orchestration. |
| `src/hotspot/cpu/x86/` | Platform‑specific assembly generators (we'll use x86). |
| `src/hotspot/share/interpreter/interpreterRuntime.cpp` | C++ "slow path" helpers. |
| `src/hotspot/share/runtime/frame.hpp` | C++ view over the native stack. |

---

## 4. Key Concepts You Need to Know

### Threaded Dispatch
In a `switch`‑based interpreter, after each instruction you jump back to the loop, fetch the next opcode, and re‑enter the `switch`. This causes branch prediction misses.  
HotSpot uses **threaded dispatch**: at the end of each assembly stub, it loads the *next* opcode from the BCP (Bytecode Pointer), looks up its stub address in the Dispatch Table, and does an indirect `JMP` directly to that stub. No central loop – just straight‑line jumps.

### MacroAssembler
HotSpot has a C++ subsystem (the `MacroAssembler`) that emits raw binary CPU instructions into memory. When you see C++ code like `__ movl(rax, rbx);`, that's **not** executing a move – it's writing the bytes `0x89 0xD8` into the Code Cache. The `__` is a macro that expands to `_masm->`.

---

## 5. Architecture – How the Interpreter Works

### Boot Phase (Code Generation)
- `TemplateInterpreterGenerator::generate_all()` loops over all bytecodes.
- For each opcode, it calls a platform‑specific function (e.g., `TemplateTable::iadd()`).
- That function emits x86 assembly into the Code Cache and records the starting address in the Dispatch Table.

### Invocation (Running a Method)
- A Java method is called; HotSpot allocates an **Interpreter Frame** on the C stack.
- It sets CPU registers:
  - **R13** (on x86) = BCP (pointer to the bytecode array in Metaspace).
  - **R14** = Local Variables pointer.
- Then it jumps to the assembly stub for the first bytecode.

### Execution (Fast Path)
- The stub executes the bytecode purely in assembly.
- It updates the BCP, then uses threaded dispatch to jump to the next stub.

### Slow Path (C++ Fallback)
- For complex bytecodes (`new`, `monitorenter`, `athrow`), the stub calls a C++ function in `InterpreterRuntime.cpp`.
- That function does the heavy lifting (GC allocation, locking, exception handling).

---

## 6. Important C++ Classes / Structs

| Class / Struct | File | Role |
| :------------- | :--- | :--- |
| `TemplateTable` | `interpreter/templateTable.cpp` | Maps bytecodes to C++ generator functions. |
| `InterpreterMacroAssembler` | `cpu/.../interpreterMacroAssembler.hpp` | Emits CPU instructions into the Code Cache. |
| `InterpreterRuntime` | `interpreter/interpreterRuntime.hpp` | Contains static C++ methods for slow‑path operations. |
| `frame` | `runtime/frame.hpp` | C++ struct that parses the native stack to inspect Java frames. |

---

## 7. Critical Functions

- `TemplateInterpreterGenerator::generate_all()` – boot‑time code generator.
- `TemplateTable::initialize()` – wires opcodes to generation functions.
- `InterpreterRuntime::_new(JavaThread*, ConstantPool*, int index)` – C++ helper for `new`.

---

## 8. Important Macros

- **`__`** – expands to `_masm->`, used to emit instructions. Example: `__ push(rax);`
- **`def(...)`** – used in `TemplateTable::initialize()` to define each bytecode's properties.

---

## 9. Source Code Exploration (Guided Tour)

### Tour 1: The Template Table
- **Open:** `src/hotspot/share/interpreter/templateTable.cpp`
- **Look for:** `TemplateTable::initialize()`. You'll see a long list of macro calls:
```cpp
def(Bytecodes::_iadd, ____|____|____|____, itos, itos, iadd, _)
```
This says: for `iadd`, the stack before is integer (`itos`), after is integer, and the generator is `TemplateTable::iadd`.

### Tour 2: The x86 Generator
- **Open:** `src/hotspot/cpu/x86/templateTable_x86.cpp`
- **Search for:** `void TemplateTable::iadd()`. You'll see:
```cpp
__ pop_i(rdx);         // emit: pop from stack into RDX
__ addl(rax, rdx);     // emit: RAX += RDX
```
This C++ function *generates* the assembly code that will run when `iadd` is executed.

### Tour 3: The C++ Runtime Bridge
- **Open:** `src/hotspot/share/interpreter/interpreterRuntime.cpp`
- **Find:** `JRT_ENTRY(void, InterpreterRuntime::_new(JavaThread*, ConstantPool*, int))`
This is standard C++ – called from assembly when the fast‑path allocation fails.

---

## 10. Execution Flow – `new Object()`

1. BCP points to opcode `0xBB` (`new`).
2. Threaded dispatch jumps to the stub generated by `TemplateTable::_new()`.
3. The stub tries a **fast‑path allocation** (bump‑pointer in the TLAB – Thread Local Allocation Buffer). If it succeeds:
   - It initialises the `markWord` and `klass` pointer.
   - Pushes the new `oop` onto the operand stack.
   - Dispatches to the next bytecode – **all in assembly**.
4. If the TLAB is full, the stub falls back:
   - It pushes the Constant Pool index and issues a `call` to `InterpreterRuntime::_new()`.
   - That C++ function resolves the class, calls the GC, handles errors, and returns the `oop`.
   - Back in assembly, the stub pushes the result onto the operand stack and dispatches.

---

## 11. Real Java Example – `addTwo`

```java
public int addTwo(int a) {
    return a + 2;
}
```
Bytecode:
```
0: iload_1     // push local variable 1
1: iconst_2    // push constant 2
2: iadd        // pop two ints, add, push result
3: ireturn     // return top of stack
```
Runtime (simplified):
- `iload_1` – R14 points to locals; load offset 1 into RAX; dispatch to `iconst_2`.
- `iconst_2` – put literal `2` into a register; dispatch to `iadd`.
- `iadd` – RAX += RDX; dispatch to `ireturn`.
- `ireturn` – tear down frame, restore caller, set return value.

---

## 12. Why This Design? (The "Why")

**Why not a C++ switch statement?**
- A switch compiles to a jump table, but each iteration returns to the top of the loop → branch mispredictions.
- Threaded dispatch (direct JMP to next stub) gives the CPU a predictable flow, often avoiding mispredictions.

**Why not JIT‑compile everything at startup?**
- JIT compilation costs CPU time and memory. If every method was compiled, "Hello World" would take seconds.
- The interpreter starts execution instantly; JIT only kicks in for hot methods later.

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Reality |
| :------ | :------ |
| Putting a breakpoint in `TemplateTable::iadd()` to debug running Java code. | That C++ function runs **only once** – during startup to generate the assembly. Debug the generated machine code instead. |
| Thinking local variable 1 is a CPU register. | Local variables live in memory on the thread's stack; the interpreter moves them into registers as needed. |

---

## 14. Summary

HotSpot doesn't "interpret" bytecode using C++ logic loops. Instead, at startup it generates **optimised assembly stubs** for every bytecode, stores them in the Code Cache, and chains them together via threaded dispatch. For complex operations (allocation, locking), the assembly stubs call into C++ helpers (`InterpreterRuntime`). This gives both fast startup and good performance for interpreted code.

---

## 15. Mental Model to Remember

```
Startup:  C++ code  →  generates assembly stubs  →  Code Cache
Runtime:  Execute stub  →  update BCP  →  direct JMP to next stub
Slow path:  stub  →  calls  →  InterpreterRuntime (C++)
```

---

## 16. Important Classes / Structs

- `TemplateTable`
- `InterpreterMacroAssembler`
- `InterpreterRuntime`
- `frame`

---

## 17. Important Functions / Methods

- `TemplateTable::initialize()`
- `InterpreterRuntime::_new()`
- `TemplateInterpreterGenerator::generate_all()`

---

## 18. Important Files

- `src/hotspot/share/interpreter/templateTable.cpp`
- `src/hotspot/cpu/x86/templateTable_x86.cpp`
- `src/hotspot/share/interpreter/interpreterRuntime.cpp`

---

## 19. Code‑Reading Exercises

1. **Assembly generator** – open `src/hotspot/cpu/x86/templateTable_x86.cpp`, find `void TemplateTable::iadd()`. Notice the `__ addl(rax, rdx);` line – that's the assembly addition.

2. **Slow path** – open `src/hotspot/share/interpreter/interpreterRuntime.cpp` and search for `InterpreterRuntime::_new`. Observe the C++ logic for class resolution and GC allocation.

3. **Frame inspection** – open `src/hotspot/share/runtime/frame.hpp` and look at `sender()` and `local_at()`. This is how HotSpot walks the native stack.

---

## 20. Self‑Check Questions

1. You invent a new bytecode `fast_math`. Name the three main HotSpot files you'd modify to support it in the x86 interpreter.

2. Why does the interpreter try to allocate objects (`new`) in assembly with a bump‑pointer, rather than always calling the C++ runtime?

3. If the interpreter executes raw machine code, what's the fundamental difference between it and the JIT compiler? (Hint: does the interpreter optimise across multiple bytecodes?)

---

## 21. What to Read Next

| File | Why |
| :--- | :--- |
| `src/hotspot/share/runtime/deoptimization.cpp` | How the JIT and interpreter interact. |
| `src/hotspot/share/compiler/compileBroker.cpp` | The coordinator that decides when to JIT‑compile. |

---

## 22. Coming Up Next

**Lesson 10 – JIT Compilation & Code Cache**  
The interpreter is fast to start, but it can't do deep optimisations (inlining, loop unrolling). Once a method becomes “hot”, HotSpot brings out the heavy artillery – the C1 and C2 compilers. We'll explore how they generate highly optimised machine code and manage the Code Cache.
