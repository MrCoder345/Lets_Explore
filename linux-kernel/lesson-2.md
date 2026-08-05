
# Lesson 2: Linux Kernel Build System (Kbuild)

Every large C project needs a build system. You may already know tools like `Make`, `CMake`, or `Ninja`.

The Linux kernel uses its own build system called **Kbuild**.

To understand the kernel, you first need to understand how it is built. The Linux kernel is not a single fixed program. It is a highly configurable and modular system.

The same source code can produce:
- A small kernel for embedded devices.
- A large kernel for powerful servers with thousands of CPUs.

Kbuild solves the problem of **building a highly configurable kernel at a massive scale**.

---

# 1. Kbuild Overview

Kbuild is distributed across the entire Linux source tree.

It mainly depends on two types of files:

1. **`Kconfig`**
   - Defines available configuration options.
   - Controls what features can be enabled or disabled.

2. **`Makefile`**
   - Defines how source files are compiled based on the configuration.

The basic flow:

```text
Kconfig files
      |
      v
Configuration Tool (make menuconfig)
      |
      v
.config file
      |
      v
Makefiles
      |
      v
Kernel Build
      |
      +--> vmlinux
      |
      +--> Kernel Modules (*.ko)
````

---

# 2. The Tristate Configuration System

In normal C projects, a feature is usually either:

* Enabled
* Disabled

The Linux kernel adds a third option.

Kernel features can have three states:

| Option | Meaning                        |
| ------ | ------------------------------ |
| `Y`    | Built directly into the kernel |
| `M`    | Built as a loadable module     |
| `N`    | Disabled and not compiled      |

This is called a **tristate system**.

## `Y` - Built-in Feature

Example:

```text
CONFIG_FEATURE=y
```

The code becomes part of the main kernel image.

Advantages:

* Always available.
* Faster access.

Disadvantage:

* Increases kernel size.

---

## `M` - Kernel Module

Example:

```text
CONFIG_FEATURE=m
```

The code is compiled as a separate file:

```text
feature.ko
```

The module can be loaded when required.

Advantages:

* Keeps the kernel smaller.
* Allows drivers to be added or removed without rebuilding the kernel.

Example:

A network driver may only be loaded when the hardware exists.

---

## `N` - Disabled

Example:

```text
CONFIG_FEATURE is not set
```

The code is ignored.

The compiler never builds it.

---

# 3. Kbuild Architecture

The build process looks like this:

```text
        arch/*/Kconfig       drivers/*/Kconfig       net/Kconfig
              \                    |                    /
               ----------------------------------------
                              |
                              v
                    make menuconfig
                              |
                              v
                         .config
                  (Kernel configuration)
                              |
               --------------------------------
              /                |               \
     arch/*/Makefile   drivers/*/Makefile   net/Makefile
              \                |               /
               --------------------------------
                              |
                              v
                            make
                              |
              --------------------------------
             |                                |
             v                                v
        vmlinux                         *.ko files
   (Core kernel image)           (Kernel modules)
```

---

# 4. Kconfig Files

`Kconfig` files describe kernel configuration options.

Example:

```kconfig
config E1000
    tristate "Intel PRO/1000 Gigabit Ethernet support"
    depends on PCI
    help
      Enables support for Intel PRO/1000 ethernet adapters.
```

## Explanation

### `config E1000`

Creates a configuration symbol:

```text
CONFIG_E1000
```

This value is stored in `.config`.

Example:

```text
CONFIG_E1000=m
```

---

### `tristate`

Allows three choices:

```text
Y
M
N
```

If a feature cannot be a module, it uses:

```kconfig
bool
```

which only allows:

```text
Y
N
```

Example:

The scheduler must always exist inside the kernel, so it cannot be a module.

---

### `depends on PCI`

Defines dependencies.

The driver cannot be enabled unless PCI support exists.

Kbuild automatically handles these relationships.

---

# 5. Makefiles

After configuration, the Makefiles decide what gets compiled.

Example:

```make
obj-$(CONFIG_E1000) += e1000.o

e1000-objs := e1000_main.o e1000_hw.o e1000_ethtool.o
```

The important pattern is:

```make
obj-$(CONFIG_NAME)
```

---

## When CONFIG_E1000 = y

The line becomes:

```make
obj-y += e1000.o
```

Meaning:

* Compile the code.
* Link it directly into the kernel.

---

## When CONFIG_E1000 = m

The line becomes:

```make
obj-m += e1000.o
```

Meaning:

* Build it as a kernel module.

Output:

```text
e1000.ko
```

---

## When CONFIG_E1000 = n

The variable is empty:

```make
obj- += e1000.o
```

Kbuild ignores it.

The source files are not compiled.

---

This simple mechanism avoids large amounts of complicated `if/else` logic inside Makefiles.

---

# 6. Kernel Build Outputs

After compilation, important files are generated.

## vmlinux

`vmlinux` is:

* The raw kernel ELF binary.
* Uncompressed.
* Contains symbols and debugging information.

Usually, it is not directly booted.

---

## bzImage / Image

The bootable kernel image.

Process:

```text
vmlinux
   |
   v
Remove debug information
   |
   v
Compress
   |
   v
Add boot code
   |
   v
bzImage / Image
```

Examples:

* x86 → `bzImage`
* ARM → `Image`

This is what the bootloader loads.

---

## System.map

A map of kernel symbols.

Example:

```text
function_name -> memory_address
```

Useful for debugging:

* Kernel crashes.
* Kernel panics.

---

## Kernel Modules

Files ending with:

```text
.ko
```

Example:

```text
e1000.ko
```

These are dynamically loadable kernel components.

---

# 7. Important Files to Remember

| File                | Purpose                      |
| ------------------- | ---------------------------- |
| `.config`           | Kernel configuration choices |
| `vmlinux`           | Raw kernel ELF binary        |
| `bzImage` / `Image` | Bootable kernel image        |
| `*.ko`              | Loadable kernel modules      |
| `System.map`        | Kernel symbol address map    |

---

# 8. Mental Model

Think of Kbuild like a restaurant.

* **Kconfig** → The menu.

  * Shows available features.
  * Defines requirements.

* **`.config`** → Your order.

  * Contains your selected options.

* **Makefiles** → The kitchen.

  * Reads your order.
  * Builds only the required parts.

* **Kernel image** → Final meal.

---

# 9. Code Reading Exercises

## Exercise 1

Open:

```text
init/Kconfig
```

Find:

```text
config EXPERT
```

Notice it uses:

```kconfig
bool
```

Question:

Why can't core kernel features be built as modules?

---

## Exercise 2

Open:

```text
net/ipv4/Makefile
```

Find:

```make
obj-y
obj-$(CONFIG_...)
```

Identify:

* One built-in feature.
* One modular feature.

---

## Exercise 3

Open:

```text
Makefile
```

Search:

```text
vmlinux:
```

Look at how the final kernel binary is linked.

---

# 10. Questions for Understanding

### 1. Adding a New Driver

If you add:

```text
my_driver.c
```

to:

```text
drivers/misc/
```

which files need changes?

Answer:

* `drivers/misc/Kconfig`

  * Add the configuration option.

* `drivers/misc/Makefile`

  * Add the build rule.

---

### 2. Why Both vmlinux and bzImage?

`vmlinux`:

* Full kernel binary.
* Used during linking and debugging.

`bzImage`:

* Compressed bootable image.
* Loaded by the bootloader.

---

### 3. Multiple Source Files in a Module

Example:

```make
e1000-objs := e1000_main.o e1000_hw.o
```

Kbuild knows that these object files must be combined into:

```text
e1000.o
```

and then packaged as:

```text
e1000.ko
```
