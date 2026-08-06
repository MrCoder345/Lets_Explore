# Lesson 17 — Cgroups & Namespaces

## 1. Why Containers Exist

For decades, if you wanted to run two applications with conflicting dependencies (e.g., App A needs Python 2, App B needs Python 3) or limit their CPU usage, you had to run them in separate Virtual Machines (VMs). VMs are heavy—they require booting an entirely separate operating system, complete with its own kernel, inside a hypervisor.

**Containers** solve this problem. A container is *not* a Virtual Machine. There is no hypervisor, and there is no guest OS. A container is simply a standard Linux process that has been carefully isolated from the rest of the system using two massive kernel features:

1. **Namespaces:** Control what a process can **see** (Isolation).
2. **Cgroups:** Control what a process can **use** (Resource Limits).

Tools like Docker and Kubernetes do not do any "magic"; they are merely user-space wrappers that ask the Linux kernel to create processes wrapped in Namespaces and Cgroups.

## 2. Namespaces

Normally, the Linux kernel provides a single, global view of system resources. Every process can see every other process, the global network interfaces (`eth0`), and the single root filesystem (`/`).

**Namespaces** partition these global resources. When a process is put into a namespace, the kernel lies to it. The kernel creates a virtualized slice of the system and presents it to the process as if it were the entire world.

There are several types of namespaces, each isolating a specific subsystem.

## 3. PID Namespace (Process IDs)

In the global PID namespace, the `init` or `systemd` process is PID 1.
When you create a new PID Namespace, the first process inside it is given PID 1 *from its own perspective*.

If you run `ps` inside a container, you might see 3 running processes (PIDs 1, 2, and 3). But if you run `ps` on the host machine, you will see those exact same processes running as PIDs 1045, 1046, and 1047. The kernel simply maintains a mapping between the "Inside" PID and the "Outside" PID.

## 4. Mount Namespace (Filesystems)

The Mount namespace isolates the list of mounted filesystems.
It is the modern, secure successor to `chroot`. When a container is spawned, it is given its own Mount namespace. The container runtime mounts a specific Linux distribution image (like Ubuntu or Alpine) as the root directory (`/`) *only for that namespace*.

When the process looks at `/bin/bash`, it sees the container's version, completely oblivious to the host's `/bin/bash`.

## 5. Network Namespace

A Network namespace provides an entirely brand-new, empty network stack.
When created, the namespace has only a loopback interface (`lo`). It has no physical NICs, its own IP routing tables, and its own iptables/Netfilter rules.

To connect the container to the internet, the kernel uses a **veth pair** (Virtual Ethernet). Think of it as a virtual Ethernet cable. One end is plugged into the host's network namespace (often attached to a bridge like `docker0`), and the other end is plugged into the container's network namespace as `eth0`.

## 6. User Namespace

This is the holy grail of container security.
A User namespace isolates User IDs (UIDs) and Group IDs (GIDs).

It allows a process to be **Root (UID 0) inside the container**, but mapped to an unprivileged user (e.g., UID 100000) on the host machine. If a hacker breaks out of the container, they suddenly find themselves as a completely powerless user on the host kernel, unable to modify files or load modules.

## 7. IPC Namespace (Inter-Process Communication)

Historically, Linux allows processes to share memory and send messages using System V IPC or POSIX message queues.
The IPC namespace isolates these communication channels. A process in one IPC namespace cannot attach to a shared memory segment created by a process in another IPC namespace, preventing data leakage.

## 8. UTS Namespace (Hostname)

UTS stands for UNIX Time-Sharing System. This is the simplest namespace. It allows a container to have its own unique hostname and domain name, completely independent of the host machine's name.

## 9. Cgroups (Control Groups)

While Namespaces prevent processes from *seeing* each other, they do not prevent them from hoarding resources. A single container could consume 100% of the CPU and RAM, crashing the host.

**Control Groups (cgroups)** solve this. They organize processes into hierarchical groups and apply limits and accounting to them.

Unlike namespaces (which are passed as flags to system calls), cgroups are managed via a virtual filesystem usually mounted at `/sys/fs/cgroup`.

## 10. Resource Control (Cgroup Subsystems/Controllers)

The cgroup framework includes specific "controllers" that govern different hardware resources:

- **cpu:** Limits CPU time (e.g., "This group can only use 50% of 1 CPU core").
- **memory:** Limits RAM usage. If processes in the cgroup exceed this limit, the kernel's OOM (Out of Memory) Killer will selectively assassinate processes *inside that specific cgroup*, rather than crashing the whole system.
- **blkio:** Limits disk read/write bandwidth (IOPS).
- **pids:** Limits the maximum number of processes the group can fork (prevents fork-bomb denial of service attacks).

*(Note: Linux currently uses Cgroup v2, a unified hierarchy that fixes many architectural flaws of the original Cgroup v1 implementation).*

## 11. Container Creation Flow

How does Docker actually build this?

1. **Syscall:** The container runtime calls `clone()`.
2. **Namespace Flags:** It passes flags like `CLONE_NEWPID | CLONE_NEWNS | CLONE_NEWNET`.
3. **Kernel Forking:** The kernel's `copy_process()` function (from Lesson 4) creates the new `task_struct`, detects the `CLONE_NEW*` flags, and allocates new namespace structures instead of copying the parent's.
4. **Cgroup Setup:** The runtime writes the new process's PID into a file like `/sys/fs/cgroup/my_container/cgroup.procs`.
5. **Resource Limits:** The runtime writes `500M` into `/sys/fs/cgroup/my_container/memory.max`.
6. **Execution:** The containerized process calls `execve()` and begins running, completely trapped in its virtualized box.

## 12. Important APIs

- `clone(..., flags)`: The standard process creation syscall, extended to accept `CLONE_NEW*` flags to spawn the child into new namespaces.
- `unshare(flags)`: Allows a currently running process to detach from its current namespace and create a new one for itself.
- `setns(fd, nstype)`: Allows a process to enter an *already existing* namespace (This is how `docker exec` works! It asks the kernel to join the namespaces of a running container).

## 13. Important Structs

- `struct nsproxy`: Embedded in `task_struct`. It acts as a container for pointers to all the different namespaces a task belongs to.
- `struct pid_namespace`: The internal kernel representation of a PID namespace.
- `struct cgroup`: Represents a node in the cgroup hierarchy.
- `struct css_set` (Cgroup Subsystem State): Maps a `task_struct` to its associated cgroups.

## 14. Source Files to Explore

- **`kernel/nsproxy.c`**: The core management of the `nsproxy` struct and namespace copying.
- **`kernel/cgroup/cgroup.c`**: The massive core of the Control Groups v2 framework.
- **`include/linux/nsproxy.h`**: The header defining how namespaces are attached to a task.
- **`include/linux/cgroup.h`**: The master header for cgroup state.
- **`fs/namespace.c`**: The implementation of Mount namespaces.

## 15. Mental Model

- **Namespaces (The Illusion):** Imagine putting a person inside a windowless room, painting the walls to look like a vast landscape, and telling them they are the king of the world. They believe it, but the kernel knows they are just in Room 4B.
- **Cgroups (The Utilities):** The building manager controls the pipes leading to Room 4B. If the person inside turns on all the faucets to flood the room (use all the RAM), the manager automatically restricts the water pressure to 2 gallons a minute so the rest of the building doesn't lose water.

## 16. Summary

Containers are an illusion created by the Linux kernel. By extending the `clone()` system call with Namespace flags, the kernel provides absolute isolation for PIDs, mounts, and networks, tricking processes into thinking they own the system. By exposing Cgroup controllers via the `sysfs` filesystem, the kernel enforces strict physical hardware limits on groups of processes. Together, these two mechanisms form the bedrock of modern cloud computing.

## 17. Exercises

1. **Explore Cgroups:** On any Linux machine, run `cat /sys/fs/cgroup/cgroup.controllers`. This will show you exactly which hardware resources your kernel is capable of limiting.
2. **Explore Namespaces:** Run `ls -l /proc/$$/ns`. This will list all the namespaces your current shell belongs to. Notice how each namespace is represented by a unique inode number.
3. **Read the Kernel Source:** Open `include/linux/sched.h` and find `struct task_struct`. Search within it for `struct nsproxy *nsproxy;` and `struct css_set __rcu *cgroups;`. This is the exact mathematical link between a process and its container!

## 18. Questions

1. If you run a web server inside a container on port 80, why does it not conflict with a web server running on port 80 on the host machine? Which namespace is responsible for this?

2. When you use the `docker exec -it <container> /bin/sh` command to get a terminal inside a running container, which kernel system call is Docker using to inject your new shell process into the container's isolated world?

3. Why is the User Namespace considered the most critical namespace for preventing container breakout security vulnerabilities?

---

## 💡 Answer Key (For Your Reference)

**Q1:** The **Network Namespace** is responsible. Each network namespace has its own independent set of network interfaces, IP addresses, routing tables, and port mappings. Port 80 inside the container's network namespace is completely isolated from port 80 on the host's network namespace. The container runtime maps the container's port 80 to a different port on the host (e.g., 8080) using NAT/port forwarding.

**Q2:** Docker uses the `setns()` system call. This allows a process to join an existing namespace by passing a file descriptor (obtained from `/proc/<pid>/ns/`) that references the target namespace. The new shell process is created (via `clone()` or `fork()`) and then attached to the container's namespaces using `setns()` before executing the shell binary.

**Q3:** The User Namespace maps container-internal root (UID 0) to an unprivileged user on the host (e.g., UID 100000). This means even if an attacker compromises the container and gains root privileges *inside* the container, those privileges do not translate to the host system. On the host, they are just an unprivileged user with no access to host files, kernel modules, or critical system resources. This creates a **security boundary** that makes container breakouts significantly harder.
