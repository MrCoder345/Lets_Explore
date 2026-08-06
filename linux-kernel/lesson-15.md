# Lesson 15 — Device Model & Driver Framework

## 1. Why Device Model Exists

In the early days of Linux (before the 2.6 kernel), drivers were chaotic. A driver would load, probe random hardware ports, and claim whatever it found. The kernel had no centralized idea of what hardware was actually plugged into the motherboard.

This created massive problems:

- **Power Management:** How do you put a laptop to sleep if the kernel doesn't know which devices need to be powered down first, and in what order? (You can't power down the USB controller *before* powering down the USB mouse).
- **Hotplugging:** How do you handle a user yanking a USB drive out while the system is running?
- **User-Space Visibility:** How do user-space programs (like `udev` or hardware monitors) know what is happening?

To solve this, kernel developers created the **Linux Device Model (LDM)**. It is a unified, object-oriented framework that tracks every single piece of hardware in a strict, centralized hierarchy.

## 2. Device Hierarchy

The device model represents your computer as a tree, physically radiating outward from the CPU.

```text
[ CPU / Root ]
      |
[ PCI Host Bridge ]
      |
      +---> [ PCI Graphics Card ]
      |
      +---> [ USB Host Controller ]
                   |
                   +---> [ USB Root Hub ]
                                |
                                +---> [ USB Keyboard ]
                                |
                                +---> [ USB Wi-Fi Dongle ]
```

The kernel uses this exact tree to suspend your system: it walks from the bottom up (leaf nodes first), powering down the keyboard and Wi-Fi before powering down the USB controller.

## 3. Bus (`struct bus_type`)

A **Bus** is a communication channel that hardware devices sit on.
Examples include physical buses like PCI, USB, I2C, and SPI, as well as virtual buses like the `platform` bus (used for System-on-a-Chip peripherals integrated directly into the silicon).

The Bus acts as the **Matchmaker**. Its primary job is to maintain a list of all devices plugged into it, and a list of all drivers registered to it, and pair them together.

## 4. Driver (`struct device_driver`)

A **Driver** is a piece of software that knows how to initialize, communicate with, and shut down a specific piece of hardware.
A single driver can manage multiple devices of the same type. For example, if you plug in three identical USB webcams, the kernel loads the `uvcvideo` driver once, but binds it to three separate device instances.

## 5. Device (`struct device`)

A **Device** represents a physical or logical entity on a bus.
When you plug in a USB mouse, the USB subsystem detects the electrical signal and dynamically allocates a `struct device`. It populates it with hardware IDs (like Vendor ID and Product ID) and registers it with the USB bus.

## 6. Class (`struct class`)

While a Bus describes *how a device is connected*, a Class describes *what a device does*.

- **Scenario A:** A Wi-Fi card connected via PCI.
- **Scenario B:** A Wi-Fi dongle connected via USB.

They are on entirely different buses, but to user-space (like your web browser), they do the exact same thing. Therefore, both devices belong to the `net` (Networking) class. This allows user-space programs to ignore the complex hardware topology and just say, "Give me a list of all network interfaces."

## 7. `kobject` (Kernel Object)

Linux is written in C, which lacks classes, inheritance, and reference counting. The kernel implements these object-oriented features using `struct kobject`.

A `kobject` is the fundamental building block of the Device Model.
Instead of existing on its own, a `kobject` is **embedded** inside larger structures (like `struct device` and `struct cdev`).

It provides:

1. **Reference Counting:** (Via `kref`). It ensures a `struct device` is not freed from memory while a user-space program is actively reading from it.
2. **Hierarchy:** `kobject`s have parent pointers, creating the tree we saw in Section 2.
3. **Sysfs representation:** Every `kobject` automatically gets a directory in `/sys`.

## 8. `sysfs`

`sysfs` is a virtual filesystem mounted at `/sys`.
It is not a real disk. **It is a real-time, visual representation of the `kobject` tree in RAM.**

When you navigate to `/sys/bus/usb/devices/`, you are literally browsing the kernel's internal RAM structures. If you `cat` a file in `sysfs`, it calls a C function inside a kernel driver to read a hardware register and print it as text.

## 9. Probe / Remove (Lifecycle Hooks)

When a driver registers itself, it provides two critical function pointers:

- **`.probe()`:** Called by the kernel when a matching device is found. The driver uses this to initialize the hardware, allocate memory, map I/O ports, and register interrupt handlers.
- **`.remove()`:** Called when the device is unplugged or the driver module is unloaded. The driver must undo everything it did in `probe()` to prevent memory leaks.

## 10. Driver Binding (The Matchmaking Process)

How does a driver find its device?

1. A new USB mouse is plugged in. The USB bus creates a `struct device` with Vendor ID: `0x046D` (Logitech).
2. The USB bus loops through every registered `struct device_driver` on the USB bus.
3. It calls the bus's `.match()` function.
4. The driver has a table of IDs it supports (`struct usb_device_id`). The bus compares `0x046D` against this table.
5. **MATCH!**
6. The bus then calls the driver's `.probe()` function, passing the `struct device` as an argument. The driver takes control of the hardware.

## 11. Hotplug and Uevents

When a device is added or removed, the kernel generates a **uevent** (User-space Event).
This is a message sent over a Netlink socket to user-space, usually intercepted by a daemon called `udevd` (or `systemd-udevd`).

`udevd` reads the uevent (e.g., "A new USB device with Vendor ID 0x1234 appeared!") and looks at its rules. It can then automatically run `modprobe` to load the correct kernel module from the hard drive, and create the character device node (like `/dev/ttyUSB0`) so user programs can talk to it.

## 12. Important APIs

- `bus_register()` / `bus_unregister()`: Creates a new bus type.
- `driver_register()` / `driver_unregister()`: Registers a driver with the core device model.
- `device_add()` / `device_del()`: Adds a physical device to the hierarchy.
- `class_create()` / `class_destroy()`: Creates a high-level view (like `/sys/class/net`).
- `kobject_init_and_add()`: Initializes a kobject and exposes it to `sysfs`.

## 13. Important Structs

```c
// Simplified representations from include/linux/device.h

struct bus_type {
    const char *name;
    int (*match)(struct device *dev, struct device_driver *drv);
    int (*probe)(struct device *dev);
    // ...
};

struct device_driver {
    const char *name;
    struct bus_type *bus;
    int (*probe)(struct device *dev);
    int (*remove)(struct device *dev);
    // ...
};

struct device {
    struct kobject kobj;             // The OOP base class
    struct device *parent;           // Hierarchy
    struct bus_type *bus;            // Which bus it sits on
    struct device_driver *driver;    // Which driver bound to it
    void *driver_data;               // Private data for the driver to use
    // ...
};
```

## 14. Execution Flow

Here is the lifecycle of inserting a USB drive:

```text
[ Hardware ] -> User plugs in USB Drive. Electrical interrupt fires.
      |
[ USB Hub Driver ] -> Detects port change, enumerates the device, gets IDs.
      |
[ USB Bus Core ] -> Allocates 'struct device'. Sets IDs. 
      |             Calls device_add().
      |             Generates 'uevent' to user-space (udev).
      |
[ Bus Matcher ] -> Loops through all registered USB drivers. 
      |            Matches IDs with 'usb-storage' driver.
      |
[ usb-storage Driver ] -> .probe() is called. 
      |                   Driver initializes the USB endpoint.
      |                   Registers as a Block Device (Lesson 13) and SCSI device.
      |
[ sysfs ] -> Automatically populated at /sys/bus/usb/devices/...
```

## 15. Source Files to Explore

- **Directory:** `drivers/base/` (The heart of the Linux Device Model).
- **Directory:** `include/linux/device.h` (The most important header file for driver authors).
- **Directory:** `include/linux/kobject.h` (The OOP building block).
- **Directory:** `Documentation/driver-api/` (Fantastic official documentation).
- **File:** `drivers/base/core.c` (Contains `device_add` and core hierarchy logic).
- **File:** `drivers/base/dd.c` (Contains the Driver Binding logic: `driver_probe_device`).

## 16. Mental Model

Think of the Device Model as a Corporate Employment Agency.

- **The Agency Building (Kernel/sysfs):** Where all the records are kept.
- **The Departments (Buses):** The IT Department (USB), the Maintenance Department (PCI).
- **The Job Openings (Devices):** A new computer arrives and needs someone to operate it.
- **The Job Seekers (Drivers):** Software waiting for a job.
- **The Recruiter (Bus Matcher):** Looks at the Job Opening's requirements (Vendor ID) and matches it to a Job Seeker's resume (Device ID Table).
- **The Interview/Hiring (Probe):** The Recruiter introduces them. The Seeker sets up their desk.
- **The Firing (Remove):** The computer is thrown away; the Seeker packs up their desk.

## 17. Summary

Before the Device Model, the kernel was blind to its own hardware topology. By introducing an object-oriented framework based on `kobject`, the kernel can now perfectly map the physical hierarchy of buses and devices. This enables reliable power management, seamless plug-and-play (hotplugging), and a dynamic user-space interface via `sysfs`. Writing a Linux device driver today is primarily an exercise in filling out standard structures (`device_driver`) and implementing lifecycle hooks (`probe`, `remove`).

## 18. Exercises

1. **Explore Sysfs:** Open a terminal on any Linux machine. Run `ls -l /sys/class/net/`. You will see your network interfaces. Notice that they are actually *symlinks* pointing deep into `/sys/devices/pci...` demonstrating the difference between Class and Bus!
2. **Read the Header:** Open `include/linux/device.h`. Find `struct device_driver`. Look at the function signatures for `probe` and `remove`.
3. **Trace the Bind:** Open `drivers/base/dd.c`. Search for the function `really_probe()`. This is the exact function where the kernel finally hands control over to a driver.

## 19. Questions

1. If a driver's `.probe()` function encounters an error (e.g., the device fails to respond to an initialization command), what should the function return, and what happens to the `struct device`?

2. Why does `struct device` embed a `struct kobject` inside it, rather than just using a pointer to a `kobject`? *(Hint: Think about `container_of` from Lesson 4).*

3. When a system goes into Suspend (Sleep), why is it critical that the kernel traverses the `device` tree from the leaf nodes (bottom) up to the root, rather than top-down?

---

## 💡 Answer Key (For Your Reference)

**Q1:** The `.probe()` function should return a **negative error code** (e.g., `-ENODEV` or `-EIO`). The kernel will then **clean up and destroy** the `struct device` binding attempt. The device remains registered on the bus, but no driver binds to it. The kernel logs the error via `dev_err()`, and user-space may see the device listed but without a driver attached (in `/sys` the `driver` symlink will be missing).

**Q2:** Embedding the `kobject` (composition) rather than using a pointer provides **better cache locality** and eliminates an extra memory allocation. More importantly, it enables the use of `container_of()`: given a pointer to the embedded `kobject`, the kernel can safely calculate the memory address of the containing `struct device` using pointer arithmetic. If it were a pointer, you couldn't reliably get back to the parent structure because the pointer could be NULL or point elsewhere. This is a classic C OOP pattern used throughout the kernel.

**Q3:** **Dependencies matter.** A child device (like a USB mouse) depends on its parent device (the USB host controller) to function. If you power down the parent *first*, the child loses power abruptly—potentially causing data loss or corruption for devices with pending writes (like a USB drive). By going **bottom-up**, you ensure all children are safely shut down and their data is flushed *before* the power supply to the parent bus is cut. This is called **ordered power management**.
