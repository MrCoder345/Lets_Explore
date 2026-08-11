# OpenJDK Internals: Day 2 – Building OpenJDK from Source

You can't hack on the JVM if you can't compile it.  
Building OpenJDK is uniquely tricky because it's a **bootstrapping** system: the repository contains Java code that needs `javac` to compile, but `javac` itself is part of what you're building. So you need an *existing* JDK to start with.

In this lesson, we’ll break down the build system, understand the `configure` + `make` dance, compare `fastdebug` vs `release` builds, and follow the journey from raw source files to a working `java` command.

---

## 1. The Big Picture (Mental Model)

Imagine a factory that uses an old machine to build parts for a new, better machine, while simultaneously casting the metal frame for that new machine.

```
                          [Your Workstation]
                                 │
         ┌───────────────────────┴───────────────────────┐
         │                                               │
   [Boot JDK (N-1)]                               [C/C++ Compiler]
   (Older Java version)                          (GCC / Clang / MSVC)
         │                                               │
         ├─ Compiles ───┐                    ┌── Compiles─┘
         │              ▼                    ▼
         │        [JDK Libraries]     [HotSpot JVM]
         │       (java.base, etc.)     (libjvm.so)
         │              │                    │
         └──────────────┼──────(Link)────────┘
                        ▼
               [Generated JDK Image]
               (build/.../images/jdk)
```

- **Boot JDK** provides the initial `javac` and `java` to start the build.
- **C/C++ compiler** builds the native HotSpot VM.
- Both streams converge into the final **JDK image** you can run.

---

## 2. JVM Specification vs. HotSpot (Build Context)

The JVM Specification says *nothing* about how to build or package a JVM.  
The entire `make/` system and `configure` script are purely OpenJDK infrastructure.  
Other JVM implementations (like Eclipse OpenJ9 or GraalVM) use totally different build systems (CMake, Gradle, etc.).

---

## 3. Key Files & Directories (Your Toolbox)

| Path | Purpose |
| :--- | :--- |
| `configure` (top-level) | Main entry script that probes your system and sets up the build. |
| `make/` | All GNU Make orchestration files. |
| `make/hotspot/` | Makefiles specifically for building the C++ VM. |
| `build/` | **Generated** after configure – contains all objects, libs, and the final JDK image. |
| `doc/building.md` | The *official, definitive* build guide. Read it first! |

---

## 4. Essential Concepts Before You Start

### The Boot JDK
To build OpenJDK 21, you need an existing JDK 20 (or 21) installed. The build uses its `javac` to compile the Java sources of the *new* JDK. Without it, you're stuck in a chicken-and-egg problem.

### Autoconf + Make Workflow
1. **`configure`** – probes your OS, finds the Boot JDK, checks your C++ compiler, detects missing libraries (like `libfreetype`). It then generates a `spec.gmk` file with all environment variables.
2. **`make`** – reads `spec.gmk` and executes the actual compilation and linking commands.

---

## 5. Build Architecture (Phases of `make images`)

When you run `make images`, the build executes in a strict order:

1. **Interim Build** – uses the Boot JDK to build a temporary, "interim" `javac` from the source tree. This ensures the new Java code is compiled with the *latest* compiler rules, not the Boot JDK's older ones.
2. **Generate Sources** – runs small Java/C programs that generate platform-specific C++ headers and extra Java code before the main compile.
3. **Compile HotSpot** – invokes GCC/Clang/MSVC to turn `src/hotspot/**/*.cpp` into object files, then links them into `libjvm.so` (Linux) or `jvm.dll` (Windows).
4. **Compile JDK Libraries** – uses the interim `javac` to compile `src/java.base` and all other modules.
5. **Create Image** – assembles the compiled Java classes, native libraries, and launcher into a fully structured JDK directory under `build/<config>/images/jdk/`.

---

## 6. Key Build Targets (Your "Structs")

| Make Target | What it builds |
| :--- | :--- |
| `hotspot` | **Only** the native C++ VM (fastest, good for C++ changes). |
| `java.base` | **Only** the core Java module. |
| `jdk` | All Java modules (libraries). |
| `images` | The **complete** runnable JDK image (the target you want 99% of the time). |
| `clean` | Remove all build artifacts. |

---

## 7. Critical Commands (Your "Functions")

```bash
# 1. Configure for development (with assertions!)
bash configure --with-debug-level=fastdebug

# 2. Build the full JDK (use multiple cores)
make JOBS=8 images

# 3. If you only changed C++ code:
make hotspot

# 4. If you only changed Java library code:
make jdk

# 5. Build a specific configuration (if you have multiple)
make CONF=linux-x86_64-server-fastdebug images

# 6. See the exact compiler commands (debug the build)
make LOG=debug images
```

---

## 8. Debug vs. Release Builds (Crucial!)

| Build Type | C++ Assertions | Use Case |
| :--- | :--- | :--- |
| `release` | **Disabled** | For production. If you make a mistake, it crashes silently or corrupts memory. |
| `fastdebug` | **Enabled** | **Always use this for development.** C++ `assert()` macros will catch your bugs immediately with a clear error message. |
| `slowdebug` | Enabled + extra checks | Very slow. Only used for the most stubborn bugs. |

**Rule of thumb:** As a new developer, **never** build `release`. Always use `fastdebug`.

---

## 9. Execution Flow (What happens when you type)

```text
$ bash configure --with-debug-level=fastdebug
    │
    ├── Detects OS, CPU, and C++ compiler.
    ├── Finds Boot JDK (must be present).
    ├── Writes config file: build/linux-x86_64-server-fastdebug/spec.gmk
    └── Exits.

$ make images
    │
    ├── Reads spec.gmk.
    ├── Compiles thread.cpp → thread.o
    ├── Links thread.o + others → libjvm.so
    ├── Compiles String.java → String.class
    ├── Copies launcher (java) and libjvm.so into build/.../images/jdk/bin/ and lib/
    └── Done!  You can now run:
        build/linux-x86_64-server-fastdebug/images/jdk/bin/java -version
```

---

## 10. Real-World Workflow (C++ Change)

Let's say you add a `printf("Hello from Thread!\n");` in `src/hotspot/share/runtime/thread.cpp`.

- **Do NOT** run `make images` (that takes 5–10 minutes).
- **Do** run `make hotspot` – it detects the single changed file, recompiles `thread.o`, relinks `libjvm.so`, and finishes in **seconds**.
- Then test immediately with your local `java` binary.

---

## 11. Real-World Workflow (Java Library Change)

If you change `src/java.base/share/classes/java/lang/String.java`:

- `make hotspot` will do **nothing** (it only builds C++).
- You must run `make jdk` (or `make images`) to recompile the Java bytecode.

---

## 12. Why This Design? (The Why Behind the Mess)

**Why not a simple Makefile?**  
OpenJDK supports Linux (gcc/clang), Windows (MSVC), macOS (Apple Clang), and dozens of CPU architectures (x86, ARM, RISC-V, PowerPC). A single Makefile would be an unreadable swamp of `ifdef` conditions.  
The `configure` script abstracts all OS/compiler differences into clean variables, so the Makefiles stay simple and portable.

**Why a full "Image"?**  
The JVM isn't a standalone executable. It expects a specific folder structure to find core classes (now packed in `lib/modules`), security policies, timezone data, etc. The `images` target builds that exact directory tree so you can simply `bin/java` without any `-cp` or `-Xbootclasspath` hacks.

---

## 13. Common Beginner Mistakes (Avoid These!)

| Mistake | Correction |
| :--- | :--- |
| Building `release` while coding. | Always use `--with-debug-level=fastdebug`. |
| Modifying Java code and running `make hotspot`. | `make hotspot` is for C++ only. Use `make jdk` or `make images`. |
| Running `make` from inside `src/hotspot/`. | Always run `make` from the top-level root. |
| Forgetting to set `JOBS=8` or similar. | Builds will be very slow. Use `JOBS=$(nproc)` to use all cores. |

---

## 14. Summary

- You **must** have a Boot JDK to compile the Java parts.
- You **must** have a native C/C++ compiler to build HotSpot.
- `configure` probes your system and creates a build configuration.
- `make` executes the actual compilation in stages: interim tools → HotSpot → JDK libraries → final image.
- Use `fastdebug` for development, `release` only for production builds.
- The final output lives in `build/<config>/images/jdk/` – that's your runnable JDK.

---

## 15. Code-Reading Exercises

*(Try these with your cloned repo.)*

1. **Read the official build guide**  
   Open `doc/building.md` and find the "Boot JDK Requirements" section. Check which version you need.

2. **Trace the `images` target**  
   Open `make/Main.gmk` and search for the `images:` target. Follow its dependencies to see what gets built before it.

3. **Examine HotSpot compilation flags**  
   Open `make/hotspot/lib/CompileJvm.gmk` and search for `WARNINGS_AS_ERRORS`. Notice how the build forces strict C++ warnings to keep the code clean.

4. **Find the launcher source**  
   Look for `src/java.base/share/native/libjli/java.c` – this is the C code for the `java` executable launcher.

---

## 16. Self-Check Questions

1. Why does OpenJDK require an existing Java installation (Boot JDK) instead of just using a C++ compiler alone?

2. You modify a GC file in `src/hotspot/share/gc/g1/` and run `make images` in `fastdebug` mode. Your mistake triggers a C++ `assert()`. Why does this crash with a clear error in `fastdebug`, but would cause silent heap corruption in `release`?

3. You just ran `make hotspot` successfully, but your change to `PrintStream.println()` (in Java) doesn't appear when you run `java HelloWorld`. Why?
