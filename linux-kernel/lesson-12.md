# Lesson 12 — Virtual File System (VFS)

## 1. Why VFS Exists

Imagine a Linux system with a complex storage setup: the main OS is on an `ext4` partition, a USB drive is formatted as Windows `FAT32`, a network share is mounted via `NFS`, and a specialized high-speed array uses `btrfs`.

If a user-space program wants to read a file, it shouldn't have to know whether the file is on a USB drive or a network server. It just calls the `read()` system call.

The **Virtual File System (VFS)** exists to solve the problem of **hardware and format fragmentation**. It is an abstraction layer that sits between the System Call interface and the actual low-level filesystem drivers. It provides a single, unified view of the entire directory tree and standardizes how every file is accessed.

## 2. Filesystem Abstraction (Object-Oriented C)

Linux is written in C, which is not an object-oriented language. However, the VFS is heavily object-oriented. How is this possible?

The kernel achieves OOP in C by using structures containing **function pointers**. The VFS defines standard interface structures (like "File Operations"). When a specific filesystem (like ext4) mounts, it populates these VFS structures with pointers to its own specific functions.

When you call `read()`, the VFS doesn't care what filesystem is underneath. It blindly follows the function pointer, effectively calling `filesystem->read()`.

## 3. Path Resolution

When you type `cat /usr/bin/gcc`, how does the kernel find the actual 1s and 0s on the hard drive? This process is called **Path Resolution**.

1. The kernel starts at the Root Directory (`/`).
2. It looks inside `/` to find the string `usr`.
3. It finds the disk location for `usr`, opens it, and looks for the string `bin`.
4. It finds `bin`, opens it, and looks for `gcc`.

Disk I/O is incredibly slow. If the kernel had to read the physical disk for every step of `/usr/bin/gcc` every time a program executed, the system would crawl. To fix this, VFS caches the directory tree in RAM using a specialized structure called a **dentry**.

## 4. `inode` (Index Node)

The `struct inode` is the heart of the VFS. **An inode represents a specific physical file or directory on the disk.**

- **What it contains:** File size, permissions (rwx), owner (UID/GID), timestamps (creation, modification), and pointers to the actual data blocks on the physical hardware.
- **What it DOES NOT contain:** The file's name! To the disk, a file is just a number (the inode number).

**Directory path:** `include/linux/fs.h`

## 5. `dentry` (Directory Entry)

The `struct dentry` glues a human-readable string name to a numerical `inode`.

- **What it contains:** A string name (e.g., "gcc"), a pointer to its parent dentry (e.g., "bin"), and a pointer to the `inode` it represents.
- **Where it lives:** Dentries exist **only in RAM**. They are not saved to the disk. The kernel builds them on-the-fly as you navigate directories and stores them in the **dcache** (Dentry Cache).
- **Hard Links explained:** Because the name (`dentry`) is separate from the file (`inode`), you can have two different dentries (e.g., "file1.txt" and "file2.txt") pointing to the exact same `inode`. This is a hard link!

## 6. `file`

A `struct file` represents an **open instance** of a file by a specific process.

If three different programs open `/var/log/syslog` simultaneously, there is only one `inode` in memory, but there are three separate `struct file` objects.

- **What it contains:** The current read/write offset (position) for that specific process, the mode it was opened in (Read-Only vs Read-Write), and a pointer to the `dentry`.

**Directory path:** `include/linux/fs.h`

## 7. `super_block`

The `struct super_block` represents an entire mounted filesystem.

- **What it contains:** Magic numbers to identify the filesystem type (e.g., ext4), the block size (e.g., 4096 bytes), total size, and a pointer to the root `dentry` of that specific filesystem.
- If you mount three USB drives, you will have three `super_block` structures in memory.

## 8. Mounts

Linux uses a single unified directory tree starting at `/`. You cannot access a Windows-style `D:\` drive. Instead, you attach a filesystem's `super_block` to an existing directory (like `/mnt/usb`).

The VFS manages this via a `struct vfsmount`. When Path Resolution hits the `/mnt/usb` dentry, the VFS intercepts it, looks at the mount table, and seamlessly redirects the path walk to the root dentry of the USB drive's `super_block`.

## 9. File Operations (The OOP Glue)

How does VFS talk to ext4? Through operation structs containing function pointers.

- **`struct file_operations`:** Functions acting on an open file (e.g., `read`, `write`, `mmap`, `llseek`).
- **`struct inode_operations`:** Functions acting on file metadata or directories (e.g., `create`, `lookup`, `mkdir`, `rename`).
- **`struct super_operations`:** Functions acting on the whole filesystem (e.g., `write_inode`, `umount_begin`).

```c
// Simplified excerpt from include/linux/fs.h
struct file_operations {
    ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
    ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
    int (*mmap) (struct file *, struct vm_area_struct *);
    int (*open) (struct inode *, struct file *);
    // ... many more ...
};
```

## 10. VFS Execution Flow

Here is how the four main VFS objects relate in memory:

```text
[ User Space Process ]
          |
    File Descriptor (integer like 3)
          |
[ Kernel Space ]
          |
    [ struct file ] (Tracks current read offset)
          |
          v
    [ struct dentry ] (Tracks the name and tree path in RAM)
          |
          v
    [ struct inode ] (Tracks permissions and disk blocks)
          |
          v
    [ struct super_block ] (Tracks the mounted partition)
          |
          v
[ Physical Block Device / Disk ]
```

## 11. System Call to Filesystem Flow

Let's trace a `read()` system call from user-space down to the hardware:

1. **User Space:** A C program calls `read(fd, buffer, 1024)`.
2. **Syscall Entry:** Traps into the kernel (`sys_read` in `fs/read_write.c`).
3. **VFS Layer:** The kernel uses the `fd` integer to look up the `struct file`. It calls the generic `vfs_read()`.
4. **The OOP Jump:** `vfs_read()` checks permissions, then executes the function pointer: `file->f_op->read(...)`.
5. **Specific Filesystem:** If this is an ext4 drive, that pointer points to `ext4_file_read_iter()`. Ext4 figures out which physical blocks contain the data.
6. **Block Layer / Page Cache:** Ext4 asks the memory management subsystem/block layer to fetch the physical data.
7. **Return:** Data is copied to the user's buffer using `copy_to_user()`.

## 12. Important Structs

- `struct inode`: The physical file abstraction.
- `struct dentry`: The name/path abstraction.
- `struct file`: The open-file context for a process.
- `struct super_block`: The mounted filesystem abstraction.
- `struct file_operations`: The function pointers for reading/writing.

*(All found in `include/linux/fs.h`)*

## 13. Important APIs

- `vfs_read()` / `vfs_write()`: The core generic VFS functions called by system calls.
- `path_lookup()`: The complex function that walks the directory tree to turn a string path into a `dentry`.
- `d_alloc()` / `d_add()`: Functions for managing the dcache.
- `register_filesystem()`: Called by drivers (like ext4 or FAT) during boot or module load to tell VFS, "I know how to read this format."

## 14. Source Files

- `fs/`: The root directory for the VFS and all filesystem drivers.
- `fs/read_write.c`: Contains the actual system call implementations for `read` and `write`.
- `fs/namei.c`: Contains the incredibly complex path resolution code (turning `/a/b/c` into an inode).
- `fs/dcache.c`: The Dentry Cache implementation.
- `include/linux/fs.h`: The master header file containing all the structs mentioned today.
- `Documentation/filesystems/`: Contains excellent official documentation, starting with `vfs.rst`.

## 15. Mental Model

Think of VFS as a massive, bureaucratic library system.

- **`super_block`:** The specific library branch building (e.g., the Downtown branch vs the Eastside branch).
- **`inode`:** The actual physical book on the shelf. It has a barcode (inode number), pages (data), and a history of who checked it out, but the book itself doesn't know its title.
- **`dentry`:** The card catalog cabinet. It maps a title ("Moby Dick") to a specific barcode. The library keeps the most popular cards on the front desk (dcache) so they don't have to walk to the cabinet every time.
- **`file`:** A bookmark. If you and I both check out "Moby Dick" to read in the lobby, there is only one book (`inode`), but we each have our own bookmark (`struct file`) telling us what page we are currently reading.

## 16. Summary

The Virtual File System (VFS) is an ingenious object-oriented abstraction layer built in C. It separates the volatile, name-based directory structure (Dentries) from the physical, metadata-heavy file objects (Inodes). By forcing all filesystems to implement standardized `file_operations` and `inode_operations`, the VFS allows user-space programs to access local disks, network shares, and temporary RAM disks using the exact same system calls, completely ignorant of the underlying hardware.

## 17. Exercises

1. Open `include/linux/fs.h` and find `struct file_operations`. Scroll through the massive list of function pointers. Find the ones you recognize from user-space (like `mmap`, `fsync`, `splice`).
2. Open `include/linux/fs.h` and find `struct inode`. Look for the `i_mode` (permissions), `i_uid` (owner ID), and `i_size` (file size in bytes). Notice how `inode` contains NO string fields for a filename.
3. Navigate to `fs/ext4/` and open `file.c`. Look at the bottom of the file for `const struct file_operations ext4_file_operations`. Notice how ext4 populates the VFS struct with its own custom functions, like `.read_iter = ext4_file_read_iter`.

## 18. Questions

1. If a program calls `open("/home/user/test.txt")` and then calls `fork()`, the parent and child processes share the same `struct file`. If the child reads 100 bytes, what happens to the parent's file offset?

2. Why does the kernel separate the `dentry` from the `inode` instead of just putting the filename string directly inside the `inode` structure?

3. When you unplug a USB drive without safely ejecting it, what happens to the `super_block` and `dentries` currently sitting in the kernel's RAM?

---

## 💡 Answer Key (For Your Reference)

**Q1:** The parent's file offset **also advances by 100 bytes**. Since both processes share the same `struct file` object (file descriptors are duplicated during `fork()`), they also share the same `f_pos` (file position). This is why parent and child need to coordinate when reading from the same file descriptor.

**Q2:** Because **hard links** exist. A single file (one `inode`) can have multiple names (multiple `dentries`) in different directories. If the filename were stored in the `inode`, you couldn't have two different names for the same file. Separating them also allows the `dentry` cache to keep path lookups fast while the `inode` manages physical disk data.

**Q3:** The `super_block` and `dentries` become **stale/dangling references** in kernel memory. If you try to access the mount point, the VFS will return errors (like `EIO` or `ESTALE`). The kernel cannot safely flush these structures to disk because the device is gone. Eventually, if you try to unmount, the kernel may complain that the filesystem is "busy" or force a lazy unmount. This is why safe ejection is important—it ensures all cached data is written to disk before the device disconnects.
