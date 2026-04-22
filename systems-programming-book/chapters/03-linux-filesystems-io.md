# Chapter 3: File Systems, VFS & I/O

> *"On a UNIX system, everything is a file; if something is not a file, it is a process."*
> — Classic UNIX wisdom (with a grain of salt)

Linux's power as a server operating system comes largely from its sophisticated file system
and I/O stack. This chapter takes you deep into the kernel's Virtual File System layer,
file descriptors, I/O multiplexing with epoll, the revolutionary io_uring interface,
zero-copy techniques, and much more — all with hands-on Go code that talks directly
to the kernel via syscalls.

By the end of this chapter, you will understand:

- How the VFS abstraction lets Linux support dozens of file system types through one interface
- The three-level file descriptor architecture that underpins all I/O
- How the page cache makes your programs fast without you even knowing
- Why epoll replaced select/poll and how Go's runtime uses it
- How io_uring is changing the game for high-performance I/O
- Zero-copy techniques that eliminate unnecessary data copies
- File locking, inotify, overlayfs, and I/O scheduling

---

## 3.1 The Virtual File System (VFS)

### 3.1.1 Why VFS Exists

Linux supports over 70 file system types: ext4, XFS, Btrfs, ZFS, NFS, procfs, sysfs,
tmpfs, FUSE, overlayfs, and many more. Yet every user-space program uses the same
system calls — `open()`, `read()`, `write()`, `stat()` — regardless of the underlying
file system. How?

The answer is the **Virtual File System (VFS)** — an abstraction layer inside the kernel
that provides a uniform interface to all file system implementations. VFS defines a set
of data structures and function pointers that every file system driver must implement.
When you call `read()`, the VFS dispatches to the correct file-system-specific
`read` implementation.

```text
┌─────────────────────────────────────────────────────────────────────┐
│                        User Space                                   │
│                                                                     │
│   open()    read()    write()    stat()    readdir()    close()     │
│     │         │         │          │          │           │          │
└─────┼─────────┼─────────┼──────────┼──────────┼───────────┼─────────┘
      │         │         │          │          │           │
══════╪═════════╪═════════╪══════════╪══════════╪═══════════╪══════════
      │         │         │          │          │           │
┌─────▼─────────▼─────────▼──────────▼──────────▼───────────▼─────────┐
│                    System Call Interface                              │
│                    (sys_open, sys_read, ...)                         │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────────────┐
│                                                                     │
│                    Virtual File System (VFS)                        │
│                                                                     │
│   ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐      │
│   │ Superblock │  │   Inode   │  │  Dentry   │  │   File    │      │
│   │  Object    │  │  Object   │  │  Object   │  │  Object   │      │
│   └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘      │
│         │              │              │              │               │
│         ▼              ▼              ▼              ▼               │
│   super_operations  inode_ops    dentry_ops    file_operations      │
│                                                                     │
└───┬──────────┬──────────┬──────────┬──────────┬──────────┬─────────┘
    │          │          │          │          │          │
    ▼          ▼          ▼          ▼          ▼          ▼
 ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
 │ ext4 │  │ XFS  │  │procfs│  │sysfs │  │tmpfs │  │ NFS  │
 └──────┘  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘
```

### 3.1.2 The Four Core VFS Objects

Every file system in Linux revolves around four kernel data structures. Understanding
these is fundamental to understanding how Linux manages files.

#### Superblock Object (`struct super_block`)

The superblock represents a **mounted file system**. There is one superblock per mounted
file system instance. It contains:

| Field | Description |
|---|---|
| `s_type` | Pointer to the file_system_type (ext4, xfs, etc.) |
| `s_blocksize` | Block size in bytes (typically 4096) |
| `s_maxbytes` | Maximum file size supported |
| `s_op` | Pointer to `super_operations` (alloc_inode, write_inode, etc.) |
| `s_root` | The root dentry of this file system |
| `s_inodes` | List of all inodes belonging to this superblock |
| `s_dirty` | List of dirty (modified) inodes needing writeback |
| `s_dev` | Device identifier |

The `super_operations` struct contains function pointers like:

```c
// Key superblock operations (simplified from kernel source)
struct super_operations {
    struct inode *(*alloc_inode)(struct super_block *sb);
    void (*destroy_inode)(struct inode *);
    void (*dirty_inode)(struct inode *, int flags);
    int (*write_inode)(struct inode *, struct writeback_control *wbc);
    void (*put_super)(struct super_block *);
    int (*statfs)(struct dentry *, struct kstatfs *);
    int (*remount_fs)(struct super_block *, int *, char *);
};
```

#### Inode Object (`struct inode`)

The inode (index node) represents a **specific file or directory** on disk. Every file
has exactly one inode, regardless of how many names (hard links) point to it.

```
┌─────────────────────────────────────────────┐
│              struct inode                     │
├─────────────────────────────────────────────┤
│  i_mode      : file type + permissions       │
│               (regular, directory, symlink,   │
│                socket, pipe, block, char)     │
│  i_uid       : owner user ID                 │
│  i_gid       : owner group ID                │
│  i_ino       : inode number (unique per fs)  │
│  i_nlink     : hard link count               │
│  i_size      : file size in bytes            │
│  i_atime     : last access time              │
│  i_mtime     : last modification time        │
│  i_ctime     : last status change time       │
│  i_blocks    : number of 512-byte blocks     │
│  i_blksize   : preferred I/O block size      │
│  i_op        : pointer to inode_operations   │
│  i_fop       : pointer to file_operations    │
│  i_sb        : pointer to superblock         │
│  i_mapping   : pointer to address_space      │
│               (page cache for this file)      │
│  i_data      : block pointers / extents      │
│               (where data lives on disk)      │
└─────────────────────────────────────────────┘
```

**Critical insight**: The inode does NOT contain the file name. Names are stored
in directory entries (dentries). This is what makes hard links possible — multiple
names pointing to the same inode.

#### Dentry Object (`struct dentry`)

The dentry (directory entry) represents a **component in a path**. For the path
`/home/user/hello.txt`, there are four dentries: `/`, `home`, `user`, and `hello.txt`.

```
Path: /home/user/hello.txt

dentry "/"
  └── dentry "home"
        └── dentry "user"
              └── dentry "hello.txt" ──→ inode #42
```

Dentries are cached aggressively in the **dentry cache (dcache)** — one of the most
important caches in the kernel. Path lookups are extremely common, and the dcache
turns O(n) directory scans into O(1) hash lookups.

A dentry can be in one of three states:

1. **Used** — associated with a valid inode, in active use (positive reference count)
2. **Unused** — associated with a valid inode, but not currently in use (cached for reuse)
3. **Negative** — not associated with any inode (the file doesn't exist), cached to speed up future lookups for non-existent files

#### File Object (`struct file`)

The file object represents an **open file** — it exists only when a process has the file
open. Multiple file objects can point to the same dentry/inode (multiple processes
opening the same file, or the same process opening it multiple times).

```
┌─────────────────────────────────────────────┐
│              struct file                     │
├─────────────────────────────────────────────┤
│  f_path      : dentry + mount point          │
│  f_inode     : pointer to inode              │
│  f_op        : pointer to file_operations    │
│  f_pos       : current read/write offset     │
│  f_flags     : open flags (O_RDONLY, etc.)   │
│  f_mode      : access mode                   │
│  f_count     : reference count               │
│  f_owner     : owner for async I/O signals   │
└─────────────────────────────────────────────┘
```

### 3.1.3 How VFS Unifies Different File Systems

When you run `cat /proc/cpuinfo`, here's what happens:

1. The VFS receives the `open("/proc/cpuinfo")` call
2. It walks the path using the dcache: `/` → `proc` → `cpuinfo`
3. At the mount point `/proc`, it discovers this is a **procfs** mount
4. It calls procfs's `open` implementation, which creates a kernel buffer
5. When you `read()`, VFS calls procfs's `read` which **generates** CPU info dynamically
6. There is **no disk involved** — procfs manufactures data from kernel structures

Compare this with `cat /etc/passwd`:

1. VFS receives `open("/etc/passwd")`
2. Walks the path: `/` → `etc` → `passwd`
3. `/etc` is on an ext4 partition
4. ext4's `open` locates the inode on disk
5. `read()` calls ext4's read, which fetches data from the page cache (or disk)

**Same user-space API, completely different implementations.** That's the power of VFS.

### 3.1.4 The Inode — In Full Detail

Let's examine what an inode actually contains on ext4:

```
On-disk inode (ext4, 256 bytes):
┌────────────────────────────────────────────────────────┐
│  Bytes 0-1:   i_mode        (file type + permissions)  │
│  Bytes 2-3:   i_uid_low     (owner UID, low 16 bits)  │
│  Bytes 4-7:   i_size_lo     (file size, low 32 bits)  │
│  Bytes 8-11:  i_atime       (last access time)         │
│  Bytes 12-15: i_ctime       (inode change time)        │
│  Bytes 16-19: i_mtime       (last modification time)   │
│  Bytes 20-23: i_dtime       (deletion time)            │
│  Bytes 24-25: i_gid_low     (group ID, low 16 bits)   │
│  Bytes 26-27: i_links_count (hard link count)          │
│  Bytes 28-31: i_blocks_lo   (512-byte block count)     │
│  Bytes 32-35: i_flags       (ext4 flags)               │
│  Bytes 40-99: i_block[15]   (block pointers/extents)   │
│  ...                                                    │
│  Bytes 128+:  Extended attributes (xattrs)              │
│  Bytes 152+:  nanosecond timestamps (since ext4)        │
└────────────────────────────────────────────────────────┘
```

**File types encoded in i_mode:**

| Bit Pattern | Type |
|---|---|
| `0100000` | Regular file |
| `0040000` | Directory |
| `0120000` | Symbolic link |
| `0140000` | Socket |
| `0010000` | FIFO (named pipe) |
| `0060000` | Block device |
| `0020000` | Character device |

### 3.1.5 Hard Links vs Symbolic Links — At the Inode Level

```
Hard Links:
                                    ┌───────────┐
  dentry "report.txt" ────────────→ │  inode #42 │
                                    │            │
  dentry "report_backup.txt" ─────→ │ i_nlink: 2 │
                                    │ data blocks│
                                    └───────────┘

  Both names point to the SAME inode. The inode's link count is 2.
  Deleting one name decrements i_nlink. Data is freed only when i_nlink == 0.
  Hard links CANNOT cross file system boundaries (same inode table).
  Hard links CANNOT point to directories (to prevent cycles).


Symbolic Links:
                                    ┌───────────┐         ┌───────────┐
  dentry "link.txt" ──────────────→ │  inode #99 │         │  inode #42 │
                                    │ i_nlink: 1 │         │            │
                                    │ data: path │────────→│ actual data│
                                    │ "/a/b.txt" │  lookup │            │
                                    └───────────┘         └───────────┘

  The symlink has its OWN inode (#99). Its data is the TARGET PATH string.
  The kernel resolves the path when you access the symlink.
  Symlinks CAN cross file systems, CAN be dangling (target deleted).
```

### 3.1.6 Hands-On: Exploring Inodes from Go

```go
// explore_inodes.go
//
// This program demonstrates how to inspect inode information from Go
// using the syscall package. It shows how hard links share inodes,
// how symbolic links have their own inodes, and how to read all the
// metadata that lives in the inode structure.
//
// Build and run:
//   go build -o explore_inodes explore_inodes.go
//   sudo ./explore_inodes    # (sudo for some /proc access)
package main

import (
	"fmt"
	"os"
	"syscall"
	"time"
)

// inodeInfo extracts and formats detailed inode information from
// a syscall.Stat_t structure. This mirrors what the `stat` command
// shows, but accessed programmatically from Go.
//
// The Stat_t structure maps directly to the kernel's stat buffer,
// giving us raw access to inode metadata without any abstraction.
type inodeInfo struct {
	Path       string      // File path we stat'd
	Ino        uint64      // Inode number — unique identifier within the filesystem
	Nlink      uint64      // Hard link count — how many directory entries point here
	Mode       os.FileMode // File type and permission bits
	UID        uint32      // Owner user ID
	GID        uint32      // Owner group ID
	Size       int64       // File size in bytes
	Blocks     int64       // Number of 512-byte blocks allocated
	BlkSize    int64       // Preferred I/O block size (usually 4096)
	Dev        uint64      // Device ID of the filesystem containing this file
	Rdev       uint64      // Device ID (only for device files)
	AccessTime time.Time   // Last access time (atime)
	ModifyTime time.Time   // Last data modification time (mtime)
	ChangeTime time.Time   // Last inode change time (ctime) — NOT creation time!
}

// getInodeInfo retrieves full inode information for a given path.
// It uses syscall.Stat (which calls the stat(2) system call) to
// get the raw Stat_t structure, then converts it to our friendlier
// inodeInfo format.
//
// Note: We use Stat (not Lstat) here, so symlinks are followed.
// To inspect the symlink's own inode, use syscall.Lstat instead.
func getInodeInfo(path string) (*inodeInfo, error) {
	var stat syscall.Stat_t

	// stat(2) follows symbolic links and returns info about the target.
	// lstat(2) returns info about the link itself.
	if err := syscall.Stat(path, &stat); err != nil {
		return nil, fmt.Errorf("stat %s: %w", path, err)
	}

	return &inodeInfo{
		Path:       path,
		Ino:        stat.Ino,
		Nlink:      stat.Nlink,
		Mode:       os.FileMode(stat.Mode),
		UID:        stat.Uid,
		GID:        stat.Gid,
		Size:       stat.Size,
		Blocks:     stat.Blocks,
		BlkSize:    stat.Blksize,
		Dev:        stat.Dev,
		Rdev:       stat.Rdev,
		AccessTime: time.Unix(stat.Atim.Sec, stat.Atim.Nsec),
		ModifyTime: time.Unix(stat.Mtim.Sec, stat.Mtim.Nsec),
		ChangeTime: time.Unix(stat.Ctim.Sec, stat.Ctim.Nsec),
	}, nil
}

// getLinkInodeInfo uses Lstat to get the inode of a symlink itself
// (not its target). This is crucial for understanding the difference
// between hard links and symbolic links at the inode level.
func getLinkInodeInfo(path string) (*inodeInfo, error) {
	var stat syscall.Stat_t

	// lstat(2) does NOT follow symbolic links.
	if err := syscall.Lstat(path, &stat); err != nil {
		return nil, fmt.Errorf("lstat %s: %w", path, err)
	}

	return &inodeInfo{
		Path:       path,
		Ino:        stat.Ino,
		Nlink:      stat.Nlink,
		Mode:       os.FileMode(stat.Mode),
		UID:        stat.Uid,
		GID:        stat.Gid,
		Size:       stat.Size,
		Blocks:     stat.Blocks,
		BlkSize:    stat.Blksize,
		Dev:        stat.Dev,
		Rdev:       stat.Rdev,
		AccessTime: time.Unix(stat.Atim.Sec, stat.Atim.Nsec),
		ModifyTime: time.Unix(stat.Mtim.Sec, stat.Mtim.Nsec),
		ChangeTime: time.Unix(stat.Ctim.Sec, stat.Ctim.Nsec),
	}, nil
}

// printInodeInfo displays formatted inode information, similar to
// the output of `stat filename` on the command line.
func printInodeInfo(info *inodeInfo) {
	fmt.Printf("  File: %s\n", info.Path)
	fmt.Printf("  Inode: %d\n", info.Ino)
	fmt.Printf("  Links: %d\n", info.Nlink)
	fmt.Printf("  Mode: %s (%o)\n", info.Mode, uint32(info.Mode))
	fmt.Printf("  UID/GID: %d/%d\n", info.UID, info.GID)
	fmt.Printf("  Size: %d bytes\n", info.Size)
	fmt.Printf("  Blocks: %d (512-byte) | BlkSize: %d\n", info.Blocks, info.BlkSize)
	fmt.Printf("  Device: %d:%d\n", info.Dev>>8, info.Dev&0xff) // major:minor
	fmt.Printf("  Access: %s\n", info.AccessTime.Format(time.RFC3339Nano))
	fmt.Printf("  Modify: %s\n", info.ModifyTime.Format(time.RFC3339Nano))
	fmt.Printf("  Change: %s\n", info.ChangeTime.Format(time.RFC3339Nano))
}

func main() {
	// Create a temporary file to experiment with
	tmpFile := "/tmp/inode_demo.txt"
	hardLink := "/tmp/inode_demo_hard.txt"
	symLink := "/tmp/inode_demo_sym.txt"

	// Clean up any previous runs
	os.Remove(hardLink)
	os.Remove(symLink)
	os.Remove(tmpFile)

	// Create the original file
	if err := os.WriteFile(tmpFile, []byte("Hello, inodes!\n"), 0644); err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create file: %v\n", err)
		os.Exit(1)
	}

	// Create a hard link — this creates a NEW directory entry (dentry)
	// pointing to the SAME inode. The inode's link count increases to 2.
	if err := os.Link(tmpFile, hardLink); err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create hard link: %v\n", err)
		os.Exit(1)
	}

	// Create a symbolic link — this creates a NEW inode whose data
	// contains the path to the target file.
	if err := os.Symlink(tmpFile, symLink); err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create symlink: %v\n", err)
		os.Exit(1)
	}

	// --- Inspect the original file ---
	fmt.Println("=== Original File ===")
	info, err := getInodeInfo(tmpFile)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	printInodeInfo(info)

	// --- Inspect the hard link ---
	fmt.Println("\n=== Hard Link ===")
	hlInfo, err := getInodeInfo(hardLink)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	printInodeInfo(hlInfo)

	// Verify: hard link shares the same inode!
	fmt.Printf("\n  ✓ Same inode? %v (original=%d, hardlink=%d)\n",
		info.Ino == hlInfo.Ino, info.Ino, hlInfo.Ino)
	fmt.Printf("  ✓ Link count: %d (both names point to same inode)\n", hlInfo.Nlink)

	// --- Inspect the symbolic link (following it) ---
	fmt.Println("\n=== Symbolic Link (stat — follows link) ===")
	slInfo, err := getInodeInfo(symLink)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	printInodeInfo(slInfo)
	fmt.Printf("\n  ✓ Same inode as original? %v (follows to target)\n",
		info.Ino == slInfo.Ino)

	// --- Inspect the symbolic link (NOT following it) ---
	fmt.Println("\n=== Symbolic Link (lstat — link itself) ===")
	slOwnInfo, err := getLinkInodeInfo(symLink)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	printInodeInfo(slOwnInfo)
	fmt.Printf("\n  ✓ Different inode from original? %v (symlink has own inode=%d)\n",
		info.Ino != slOwnInfo.Ino, slOwnInfo.Ino)
	fmt.Printf("  ✓ Symlink size: %d bytes (= length of target path string)\n",
		slOwnInfo.Size)

	// Clean up
	os.Remove(hardLink)
	os.Remove(symLink)
	os.Remove(tmpFile)
}
```

**Expected output:**
```
=== Original File ===
  File: /tmp/inode_demo.txt
  Inode: 1234567
  Links: 2            ← Link count is 2 because of the hard link!
  Mode: -rw-r--r-- (100644)
  ...

=== Hard Link ===
  File: /tmp/inode_demo_hard.txt
  Inode: 1234567       ← SAME inode number!
  Links: 2
  ...

  ✓ Same inode? true (original=1234567, hardlink=1234567)
  ✓ Link count: 2 (both names point to same inode)

=== Symbolic Link (lstat — link itself) ===
  File: /tmp/inode_demo_sym.txt
  Inode: 7654321       ← DIFFERENT inode — symlink has its own!
  Links: 1
  Size: 19 bytes       ← Length of "/tmp/inode_demo.txt"
```

---

## 3.2 File Descriptors — The Universal Handle

### 3.2.1 What is a File Descriptor?

A file descriptor (fd) is simply a **non-negative integer** — an index into a
per-process table of open files maintained by the kernel. When you call `open()`,
the kernel:

1. Creates a `struct file` object (the open file description)
2. Finds the lowest available index in the process's fd table
3. Points that index to the new file object
4. Returns the index to user space

That integer — 3, 7, 42 — is your file descriptor.

### 3.2.2 The Three-Level Architecture

This is one of the most important diagrams in systems programming:

```
Process A                          Process B
┌──────────────┐                   ┌──────────────┐
│ fd table     │                   │ fd table     │
│              │                   │              │
│ 0 ──→ ●─────┼───┐               │ 0 ──→ ●─────┼───┐
│ 1 ──→ ●─────┼───┼───┐           │ 1 ──→ ●─────┼───┼───┐
│ 2 ──→ ●─────┼───┼───┼───┐       │ 2 ──→ ●─────┼───┼───┼───┐
│ 3 ──→ ●─────┼─┐ │   │   │       │ 3 ──→ ●─────┼─┐ │   │   │
│ 4 ──→ ●─────┼─┼─┼───┼─┐ │       └──────────────┘ │ │   │   │
└──────────────┘ │ │   │ │ │                         │ │   │   │
                 │ │   │ │ │                         │ │   │   │
                 ▼ ▼   ▼ ▼ ▼                         ▼ ▼   ▼   ▼
        ┌──────────────────────────────────────────────────────────────┐
        │              System-Wide Open File Table                     │
        │                                                              │
        │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
        │  │ File Obj A   │  │ File Obj B   │  │ File Obj C   │      │
        │  │ offset: 0    │  │ offset: 1024 │  │ offset: 0    │      │
        │  │ flags: O_RD  │  │ flags: O_WR  │  │ flags: O_RW  │      │
        │  │ inode: ●─────┼──┼──────────────┼──┼──→┐          │      │
        │  └──────────────┘  └──────┼───────┘  └───┼──────────┘      │
        │                          │              │                    │
        └──────────────────────────┼──────────────┼────────────────────┘
                                   │              │
                                   ▼              ▼
        ┌──────────────────────────────────────────────────────────────┐
        │                    Inode Table (in memory)                    │
        │                                                              │
        │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
        │  │ Inode #42    │  │ Inode #99    │  │ Inode #101   │      │
        │  │ /etc/passwd  │  │ /var/log/sys │  │ /home/data   │      │
        │  │ links: 1     │  │ links: 1     │  │ links: 3     │      │
        │  │ size: 2048   │  │ size: 50000  │  │ size: 1024   │      │
        │  └──────────────┘  └──────────────┘  └──────────────┘      │
        │                                                              │
        └──────────────────────────────────────────────────────────────┘
```

**Key relationships:**

1. **fd table** (per-process): Maps integers to entries in the open file table.
   After `fork()`, the child gets a **copy** of the parent's fd table, but both
   point to the **same** file objects (shared offsets!).

2. **Open file table** (system-wide): Each entry has a file offset, access mode,
   and pointer to an inode. Created by `open()`. Shared after `fork()` or `dup()`.

3. **Inode table** (system-wide): Represents the actual file on disk. Multiple
   open file entries can point to the same inode (same file opened multiple times).

### 3.2.3 Standard File Descriptors

Every process starts with three open file descriptors:

| fd | Name | C Constant | Go Equivalent | Description |
|---|---|---|---|---|
| 0 | stdin | `STDIN_FILENO` | `os.Stdin` | Standard input |
| 1 | stdout | `STDOUT_FILENO` | `os.Stdout` | Standard output |
| 2 | stderr | `STDERR_FILENO` | `os.Stderr` | Standard error |

These are established by the shell before your program runs. When you type
`./myprogram < input.txt > output.txt 2>&1`, the shell opens `input.txt` on fd 0,
`output.txt` on fd 1, and duplicates fd 1 to fd 2.

### 3.2.4 dup() and dup2() — Redirecting File Descriptors

`dup(oldfd)` creates a copy of `oldfd` using the lowest available fd number.
`dup2(oldfd, newfd)` makes `newfd` a copy of `oldfd`, closing `newfd` first if needed.

Both result in two file descriptors pointing to the **same open file description**
(same offset, same flags).

### 3.2.5 Important Open Flags

| Flag | Description |
|---|---|
| `O_CLOEXEC` | Automatically close this fd when exec() is called. Essential for security — prevents leaking fds to child processes. |
| `O_NONBLOCK` | Non-blocking mode. read/write return EAGAIN instead of blocking. Critical for event-driven I/O. |
| `O_DIRECT` | Bypass the page cache. Writes go directly to disk. Reads come directly from disk. Requires aligned buffers. |
| `O_SYNC` | Synchronous I/O. write() doesn't return until data is on disk. Expensive but safe. |
| `O_APPEND` | Writes always go to end of file. Atomic even across processes. |
| `O_CREAT` | Create the file if it doesn't exist. |
| `O_EXCL` | Used with O_CREAT — fail if file already exists. Atomic create-if-not-exists. |
| `O_TRUNC` | Truncate file to zero length on open. |

### 3.2.6 Hands-On: Examining /proc/self/fd from Go

```go
// proc_fd_explorer.go
//
// This program demonstrates how to examine a process's file descriptors
// by reading /proc/self/fd. This directory contains a symbolic link for
// each open file descriptor, where the link name is the fd number and
// the link target is the file path (or a description like "pipe:[12345]").
//
// The /proc filesystem is a window into kernel data structures — these
// "files" don't exist on disk; they're generated on-the-fly by the kernel.
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"strconv"
	"strings"
	"syscall"
)

// fdInfo holds information about a single file descriptor, gathered
// from /proc/self/fd and the corresponding fdinfo entry.
type fdInfo struct {
	FD     int    // File descriptor number
	Target string // What the fd points to (file path, pipe, socket, etc.)
	Flags  int    // File status flags (O_RDONLY, O_WRONLY, etc.)
	Offset int64  // Current file offset (position for next read/write)
}

// listProcessFDs reads /proc/self/fd to enumerate all open file descriptors
// for the current process. For each fd, it reads the symlink target to
// determine what file or resource the fd refers to.
//
// /proc/self is a magic symlink that always points to the current process's
// proc directory (e.g., /proc/1234 for PID 1234).
func listProcessFDs() ([]fdInfo, error) {
	// Read all entries in /proc/self/fd
	// Each entry is a symlink named with the fd number
	entries, err := os.ReadDir("/proc/self/fd")
	if err != nil {
		return nil, fmt.Errorf("reading /proc/self/fd: %w", err)
	}

	var fds []fdInfo
	for _, entry := range entries {
		// Parse the fd number from the directory entry name
		fdNum, err := strconv.Atoi(entry.Name())
		if err != nil {
			continue // Skip non-numeric entries (shouldn't happen)
		}

		// Read the symlink to find out what this fd points to.
		// For regular files, this is the absolute path.
		// For pipes: "pipe:[inode]"
		// For sockets: "socket:[inode]"
		// For anonymous inodes: "anon_inode:[type]"
		target, err := os.Readlink(filepath.Join("/proc/self/fd", entry.Name()))
		if err != nil {
			target = fmt.Sprintf("(error: %v)", err)
		}

		// Read additional info from /proc/self/fdinfo/N
		// This gives us flags and offset
		info := fdInfo{
			FD:     fdNum,
			Target: target,
		}

		// Parse /proc/self/fdinfo/N for flags and position
		fdinfoPath := filepath.Join("/proc/self/fdinfo", entry.Name())
		if data, err := os.ReadFile(fdinfoPath); err == nil {
			for _, line := range strings.Split(string(data), "\n") {
				if strings.HasPrefix(line, "flags:") {
					fmt.Sscanf(strings.TrimPrefix(line, "flags:"), "%o", &info.Flags)
				}
				if strings.HasPrefix(line, "pos:") {
					fmt.Sscanf(strings.TrimSpace(strings.TrimPrefix(line, "pos:")), "%d", &info.Offset)
				}
			}
		}

		fds = append(fds, info)
	}

	return fds, nil
}

// flagsToString converts numeric file flags to human-readable strings.
// These are the O_* flags from fcntl.h, stored in the file object.
func flagsToString(flags int) string {
	var parts []string

	// Access mode is in the low 2 bits
	switch flags & syscall.O_ACCMODE {
	case syscall.O_RDONLY:
		parts = append(parts, "O_RDONLY")
	case syscall.O_WRONLY:
		parts = append(parts, "O_WRONLY")
	case syscall.O_RDWR:
		parts = append(parts, "O_RDWR")
	}

	// Check other flags
	if flags&syscall.O_APPEND != 0 {
		parts = append(parts, "O_APPEND")
	}
	if flags&syscall.O_NONBLOCK != 0 {
		parts = append(parts, "O_NONBLOCK")
	}
	if flags&syscall.O_CLOEXEC != 0 {
		parts = append(parts, "O_CLOEXEC")
	}
	if flags&syscall.O_SYNC != 0 {
		parts = append(parts, "O_SYNC")
	}
	if flags&syscall.O_DIRECT != 0 {
		parts = append(parts, "O_DIRECT")
	}

	return strings.Join(parts, "|")
}

func main() {
	fmt.Println("=== Open File Descriptors for Current Process ===")
	fmt.Printf("PID: %d\n\n", os.Getpid())

	// Open some extra files to make the output more interesting
	// These will show up alongside stdin/stdout/stderr
	f1, _ := os.Open("/etc/hosts")
	defer f1.Close()

	f2, _ := os.OpenFile("/dev/null", os.O_WRONLY, 0)
	defer f2.Close()

	// Create a pipe — this gives us two fds (read end and write end)
	r, w, _ := os.Pipe()
	defer r.Close()
	defer w.Close()

	fds, err := listProcessFDs()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}

	fmt.Printf("%-4s %-45s %-10s %-20s\n", "FD", "Target", "Offset", "Flags")
	fmt.Println(strings.Repeat("-", 85))

	for _, fd := range fds {
		fmt.Printf("%-4d %-45s %-10d %-20s\n",
			fd.FD,
			fd.Target,
			fd.Offset,
			flagsToString(fd.Flags),
		)
	}
}
```

### 3.2.7 Hands-On: Implementing I/O Redirection in Go

```go
// io_redirect.go
//
// This program demonstrates I/O redirection using dup2() — the same
// mechanism that shells use to implement >, <, 2>&1, and pipes.
//
// When a shell processes: ./program > output.txt 2>&1
// It does exactly what this program does:
//   1. Open output.txt
//   2. dup2(output_fd, 1)   — redirect stdout to the file
//   3. dup2(1, 2)           — redirect stderr to wherever stdout goes
//   4. exec(./program)
//
// After dup2(oldfd, newfd), newfd refers to the same open file description
// as oldfd. The original file that newfd pointed to is closed.
package main

import (
	"fmt"
	"os"
	"syscall"
)

// redirectFD duplicates oldFD onto newFD using the dup2 system call.
// After this call, newFD refers to the same open file description as oldFD.
// If newFD was already open, it is silently closed first.
//
// This is the fundamental building block of I/O redirection in Unix.
// Every time you write `>`, `<`, `2>&1`, or `|` in a shell command,
// the shell is using dup2 under the hood.
func redirectFD(oldFD, newFD int) error {
	// dup2(oldfd, newfd) is atomic:
	// - If newfd == oldfd, it's a no-op (returns newfd)
	// - Otherwise, close(newfd) if open, then duplicate oldfd to newfd
	// - The close and duplicate are atomic (no race condition)
	if err := syscall.Dup2(oldFD, newFD); err != nil {
		return fmt.Errorf("dup2(%d, %d): %w", oldFD, newFD, err)
	}
	return nil
}

// demonstrateOutputRedirection shows how to redirect stdout to a file.
// This is equivalent to running: ./program > output.txt
func demonstrateOutputRedirection() {
	fmt.Println("=== Output Redirection Demo ===")
	fmt.Println("This line goes to the original stdout (your terminal).")

	// Save the original stdout fd so we can restore it later.
	// dup() creates a copy of fd 1 at the lowest available fd number.
	savedStdout, err := syscall.Dup(1)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to save stdout: %v\n", err)
		return
	}
	defer syscall.Close(savedStdout)

	// Open a file for writing — this will be our new stdout target
	outFile, err := os.Create("/tmp/redirected_output.txt")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create file: %v\n", err)
		return
	}

	// dup2(file_fd, 1) — make fd 1 (stdout) point to our file.
	// After this, any write to fd 1 goes to the file.
	if err := redirectFD(int(outFile.Fd()), 1); err != nil {
		fmt.Fprintf(os.Stderr, "Failed to redirect: %v\n", err)
		return
	}
	outFile.Close() // Close the extra fd; stdout (fd 1) keeps the file open

	// These writes go to the FILE, not the terminal!
	fmt.Println("This line goes to redirected_output.txt")
	fmt.Println("So does this one.")
	// Even os.Stdout writes go to the file, because os.Stdout wraps fd 1
	os.Stdout.WriteString("And this one too.\n")

	// Restore original stdout
	if err := redirectFD(savedStdout, 1); err != nil {
		// Can't even print an error to stdout here!
		os.Exit(1)
	}

	// Back to the terminal
	fmt.Println("This line is back on the terminal.")

	// Verify: read and print the redirected file
	data, _ := os.ReadFile("/tmp/redirected_output.txt")
	fmt.Printf("File contents:\n%s", data)
	os.Remove("/tmp/redirected_output.txt")
}

// demonstratePipeRedirection shows how to connect two processes via a pipe.
// This is equivalent to: ls | wc -l
func demonstratePipeRedirection() {
	fmt.Println("\n=== Pipe Redirection Demo ===")

	// Create a pipe. pipe[0] is the read end, pipe[1] is the write end.
	// Data written to pipe[1] can be read from pipe[0].
	var pipeFDs [2]int
	if err := syscall.Pipe(pipeFDs[:]); err != nil {
		fmt.Fprintf(os.Stderr, "pipe: %v\n", err)
		return
	}

	fmt.Printf("Pipe created: read_fd=%d, write_fd=%d\n", pipeFDs[0], pipeFDs[1])

	// Write some data to the pipe
	message := []byte("Hello through the pipe!\n")
	n, err := syscall.Write(pipeFDs[1], message)
	if err != nil {
		fmt.Fprintf(os.Stderr, "write to pipe: %v\n", err)
		return
	}
	fmt.Printf("Wrote %d bytes to pipe write end (fd %d)\n", n, pipeFDs[1])

	// Close the write end so the read end gets EOF after reading
	syscall.Close(pipeFDs[1])

	// Read from the pipe
	buf := make([]byte, 256)
	n, err = syscall.Read(pipeFDs[0], buf)
	if err != nil {
		fmt.Fprintf(os.Stderr, "read from pipe: %v\n", err)
		return
	}
	fmt.Printf("Read %d bytes from pipe read end (fd %d): %s",
		n, pipeFDs[0], string(buf[:n]))

	syscall.Close(pipeFDs[0])
}

func main() {
	demonstrateOutputRedirection()
	demonstratePipeRedirection()
}
```

---

## 3.3 File I/O — The Syscall Level

### 3.3.1 The Life of a write() Call

When your Go program calls `os.File.Write()`, here's the full journey:

```
Go Application
     │
     ▼
os.File.Write(data)
     │
     ▼
syscall.Write(fd, data)     ← Go runtime wrapper
     │
     ▼
write(fd, buf, count)       ← libc wrapper (or direct syscall via SYSCALL instruction)
     │
     ▼
═══════════════════════════ User/Kernel Boundary (syscall) ═══════════════
     │
     ▼
sys_write(fd, buf, count)   ← Kernel syscall handler
     │
     ▼
Look up fd in process's fd table → get struct file
     │
     ▼
Check f_op->write (VFS dispatch) → calls filesystem's write method
     │
     ▼
┌────────────────────────────────────────────────────────────────┐
│ For ext4 with buffered I/O (the common case):                  │
│                                                                │
│  1. Find or create pages in the page cache                     │
│  2. Copy data from user buffer to page cache pages             │
│  3. Mark pages as "dirty"                                      │
│  4. Return to user space (write "complete" — data is in RAM!)  │
│                                                                │
│ Later, asynchronously:                                         │
│  5. Kernel writeback thread flushes dirty pages to disk        │
│  6. Journal transaction ensures filesystem consistency          │
└────────────────────────────────────────────────────────────────┘
```

**Critical insight**: A normal `write()` call does NOT write to disk. It copies
data to the page cache and returns. The kernel writes to disk later. This is why:
- `write()` is fast (just a memcpy)
- Power loss can lose data (it was only in RAM)
- `fsync()` exists (to force data to disk)

### 3.3.2 The Page Cache — Linux's Secret Weapon

The page cache is one of the most important subsystems in the Linux kernel. It
caches file data in memory, using **all available free RAM**.

```
┌──────────────────────────────────────────────────────────┐
│                      Physical RAM                         │
│                                                          │
│  ┌──────────┐  ┌──────────────────────┐  ┌───────────┐  │
│  │  Kernel   │  │     Page Cache       │  │  Process   │  │
│  │  Memory   │  │                      │  │  Memory    │  │
│  │           │  │  File A, pages 0-3   │  │  (heap,    │  │
│  │  (code,   │  │  File B, pages 0-99  │  │   stack,   │  │
│  │   slab,   │  │  File C, pages 0-1   │  │   mmap)    │  │
│  │   etc.)   │  │  ...                 │  │            │  │
│  │           │  │  Grows/shrinks based  │  │            │  │
│  │           │  │  on memory pressure   │  │            │  │
│  └──────────┘  └──────────────────────┘  └───────────┘  │
│       ~5%           ~60-80%                  ~15-35%     │
└──────────────────────────────────────────────────────────┘
```

You can see the page cache in action:

```bash
$ free -h
              total    used    free   shared  buff/cache  available
Mem:           32G     8G      2G     256M       22G        23G
```

That "buff/cache" column? Most of it is the page cache. Linux uses "free" RAM
to cache file data, which is why a Linux server that's been running for a while
shows very little "free" memory — that's **good**, not bad.

### 3.3.3 Readahead — Kernel Prefetching

When you read a file sequentially, the kernel notices the pattern and starts
prefetching data **ahead** of your reads. The default readahead window is
128 KB (32 pages), but it adapts dynamically.

```
Your reads:     [page 0] [page 1] [page 2] [page 3] ...
                    ↓
Kernel notices sequential pattern
                    ↓
Kernel prefetches:              [page 4] [page 5] [page 6] [page 7]
                                     (readahead window)
                    ↓
By the time you read page 4, it's already in the page cache!
```

### 3.3.4 pread/pwrite — Atomic Positioned I/O

Normal `read()` and `write()` use the file object's current offset (`f_pos`),
and they update it after each call. This is problematic for concurrent access —
two threads sharing an fd would race on the offset.

`pread(fd, buf, count, offset)` and `pwrite(fd, buf, count, offset)` solve this
by taking an explicit offset parameter and NOT updating `f_pos`. They're atomic
with respect to the offset — no race conditions.

### 3.3.5 Hands-On: Buffered vs Direct I/O Performance

```go
// buffered_vs_direct.go
//
// This program benchmarks buffered I/O (normal, through the page cache)
// against direct I/O (O_DIRECT, bypassing the page cache).
//
// Buffered I/O:
//   - Writes go to page cache, kernel flushes to disk later
//   - Reads come from page cache if cached, disk if not
//   - Great for most workloads — the kernel is smart about caching
//
// Direct I/O (O_DIRECT):
//   - Bypasses the page cache entirely
//   - Data goes directly between user buffer and disk
//   - Buffer must be aligned to filesystem block size (usually 512 or 4096)
//   - Used by databases (they have their own caching) and benchmarks
//
// Run: go build -o buffered_vs_direct buffered_vs_direct.go && ./buffered_vs_direct
package main

import (
	"fmt"
	"os"
	"syscall"
	"time"
	"unsafe"
)

const (
	// fileSize is the total amount of data to write/read in each benchmark.
	// 256 MB is large enough to see meaningful differences.
	fileSize = 256 * 1024 * 1024

	// blockSize is the I/O block size. For O_DIRECT, this must be
	// a multiple of the filesystem's logical block size (usually 512).
	// 4096 matches the typical page size and block size.
	blockSize = 4096

	// alignment is the memory alignment required for O_DIRECT buffers.
	// The buffer address must be a multiple of the filesystem block size.
	alignment = 4096
)

// alignedBuffer allocates a byte slice that is aligned to the given
// alignment boundary. O_DIRECT requires this — if the buffer is not
// aligned, the write/read syscall will return EINVAL.
//
// We allocate extra bytes and then find the aligned offset within
// the allocation. This is a common C technique ported to Go.
func alignedBuffer(size, align int) []byte {
	// Allocate extra space so we can find an aligned offset
	buf := make([]byte, size+align)

	// Calculate the offset needed to align the start of our usable region
	ptr := uintptr(unsafe.Pointer(&buf[0]))
	offset := (align - int(ptr%uintptr(align))) % align

	// Return a slice starting at the aligned offset
	return buf[offset : offset+size]
}

// benchmarkBufferedWrite measures the time to write fileSize bytes
// using normal buffered I/O (through the page cache).
func benchmarkBufferedWrite(path string) time.Duration {
	// Open with normal flags — writes go to page cache
	fd, err := syscall.Open(path, syscall.O_WRONLY|syscall.O_CREAT|syscall.O_TRUNC, 0644)
	if err != nil {
		fmt.Fprintf(os.Stderr, "open (buffered): %v\n", err)
		os.Exit(1)
	}
	defer syscall.Close(fd)

	buf := make([]byte, blockSize)
	// Fill buffer with a recognizable pattern
	for i := range buf {
		buf[i] = byte(i % 256)
	}

	iterations := fileSize / blockSize
	start := time.Now()

	for i := 0; i < iterations; i++ {
		if _, err := syscall.Write(fd, buf); err != nil {
			fmt.Fprintf(os.Stderr, "write (buffered): %v\n", err)
			os.Exit(1)
		}
	}

	// fsync to ensure data is actually on disk for a fair comparison.
	// Without this, buffered writes would appear instant (they're in page cache).
	syscall.Fsync(fd)

	return time.Since(start)
}

// benchmarkDirectWrite measures the time to write fileSize bytes
// using O_DIRECT, bypassing the page cache entirely.
func benchmarkDirectWrite(path string) time.Duration {
	// O_DIRECT tells the kernel to bypass the page cache.
	// Data goes directly from our buffer to the disk controller.
	fd, err := syscall.Open(path,
		syscall.O_WRONLY|syscall.O_CREAT|syscall.O_TRUNC|syscall.O_DIRECT, 0644)
	if err != nil {
		fmt.Fprintf(os.Stderr, "open (direct): %v\n", err)
		os.Exit(1)
	}
	defer syscall.Close(fd)

	// O_DIRECT requires aligned buffers. If the buffer is not aligned
	// to the filesystem block size, the syscall returns EINVAL.
	buf := alignedBuffer(blockSize, alignment)
	for i := range buf {
		buf[i] = byte(i % 256)
	}

	iterations := fileSize / blockSize
	start := time.Now()

	for i := 0; i < iterations; i++ {
		if _, err := syscall.Write(fd, buf); err != nil {
			fmt.Fprintf(os.Stderr, "write (direct): %v\n", err)
			os.Exit(1)
		}
	}

	// No fsync needed — O_DIRECT writes go to disk immediately
	// (though the disk's own write cache may still be involved)
	return time.Since(start)
}

// benchmarkBufferedRead measures the time to read fileSize bytes
// using normal buffered I/O.
func benchmarkBufferedRead(path string) time.Duration {
	// Drop the page cache first to get a fair cold-cache read benchmark.
	// Without this, the data we just wrote would still be in the page cache,
	// making buffered reads appear instant.
	dropCaches()

	fd, err := syscall.Open(path, syscall.O_RDONLY, 0)
	if err != nil {
		fmt.Fprintf(os.Stderr, "open (buffered read): %v\n", err)
		os.Exit(1)
	}
	defer syscall.Close(fd)

	buf := make([]byte, blockSize)
	start := time.Now()

	for {
		n, err := syscall.Read(fd, buf)
		if n == 0 || err != nil {
			break
		}
	}

	return time.Since(start)
}

// benchmarkDirectRead measures the time to read fileSize bytes
// using O_DIRECT, bypassing the page cache.
func benchmarkDirectRead(path string) time.Duration {
	// No need to drop caches — O_DIRECT bypasses them anyway
	fd, err := syscall.Open(path, syscall.O_RDONLY|syscall.O_DIRECT, 0)
	if err != nil {
		fmt.Fprintf(os.Stderr, "open (direct read): %v\n", err)
		os.Exit(1)
	}
	defer syscall.Close(fd)

	buf := alignedBuffer(blockSize, alignment)
	start := time.Now()

	for {
		n, err := syscall.Read(fd, buf)
		if n == 0 || err != nil {
			break
		}
	}

	return time.Since(start)
}

// dropCaches writes to /proc/sys/vm/drop_caches to force the kernel
// to free the page cache. This requires root privileges.
//
// Value 1: free page cache
// Value 2: free dentries and inodes
// Value 3: free all of the above
func dropCaches() {
	// First, sync all dirty data to disk
	syscall.Sync()

	// Then drop the caches
	f, err := os.OpenFile("/proc/sys/vm/drop_caches", os.O_WRONLY, 0)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Warning: Cannot drop caches (need root): %v\n", err)
		return
	}
	defer f.Close()
	f.WriteString("3")
}

func main() {
	testFile := "/tmp/io_benchmark.dat"
	defer os.Remove(testFile)

	fmt.Printf("Benchmark: Buffered vs Direct I/O\n")
	fmt.Printf("File size: %d MB, Block size: %d bytes\n\n",
		fileSize/(1024*1024), blockSize)

	// --- Write benchmarks ---
	fmt.Println("--- Write Benchmarks ---")

	bufferedWriteTime := benchmarkBufferedWrite(testFile)
	bufferedWriteMBs := float64(fileSize) / (1024 * 1024) / bufferedWriteTime.Seconds()
	fmt.Printf("Buffered write: %v (%.1f MB/s)\n", bufferedWriteTime, bufferedWriteMBs)

	os.Remove(testFile)

	directWriteTime := benchmarkDirectWrite(testFile)
	directWriteMBs := float64(fileSize) / (1024 * 1024) / directWriteTime.Seconds()
	fmt.Printf("Direct write:   %v (%.1f MB/s)\n", directWriteTime, directWriteMBs)

	// --- Read benchmarks (use the direct-written file) ---
	fmt.Println("\n--- Read Benchmarks ---")

	bufferedReadTime := benchmarkBufferedRead(testFile)
	bufferedReadMBs := float64(fileSize) / (1024 * 1024) / bufferedReadTime.Seconds()
	fmt.Printf("Buffered read:  %v (%.1f MB/s)\n", bufferedReadTime, bufferedReadMBs)

	directReadTime := benchmarkDirectRead(testFile)
	directReadMBs := float64(fileSize) / (1024 * 1024) / directReadTime.Seconds()
	fmt.Printf("Direct read:    %v (%.1f MB/s)\n", directReadTime, directReadMBs)

	fmt.Println("\n--- Analysis ---")
	fmt.Println("Buffered I/O goes through the page cache (memcpy + async flush).")
	fmt.Println("Direct I/O bypasses the cache (goes straight to disk controller).")
	fmt.Println("For sequential I/O, buffered is often faster due to write coalescing.")
	fmt.Println("Databases use O_DIRECT because they manage their own cache.")
}
```

### 3.3.6 Hands-On: Using pread/pwrite for Concurrent File Access

```go
// pread_pwrite.go
//
// This program demonstrates pread() and pwrite() — positioned I/O syscalls
// that read/write at a specific offset WITHOUT modifying the file's current
// position (f_pos in the kernel's struct file).
//
// Why this matters:
//   - Normal read()/write() update f_pos, creating race conditions when
//     multiple goroutines share a file descriptor
//   - pread()/pwrite() are atomic with respect to offset — no races
//   - This is how databases do concurrent I/O on a single file
//
// Build: go build -o pread_pwrite pread_pwrite.go
package main

import (
	"encoding/binary"
	"fmt"
	"os"
	"sync"
	"syscall"
)

// record represents a fixed-size record in our file-based "database".
// Each record is exactly 64 bytes, making offset calculation trivial:
//   offset = recordNumber * 64
type record struct {
	ID    uint64   // 8 bytes: unique record identifier
	Value uint64   // 8 bytes: the data value
	Name  [48]byte // 48 bytes: fixed-length name field
}

const recordSize = 64 // sizeof(record) — must match the struct layout

// writeRecord writes a record at a specific position in the file using
// pwrite(). Multiple goroutines can call this concurrently on the same
// fd without any locking — pwrite is atomic with respect to the offset.
//
// The key advantage over write() + lseek():
//   - lseek(fd, offset, SEEK_SET); write(fd, data, len)  ← NOT ATOMIC (race!)
//   - pwrite(fd, data, len, offset)                       ← ATOMIC
func writeRecord(fd int, rec *record, recordNum int) error {
	// Calculate the byte offset for this record number
	offset := int64(recordNum) * recordSize

	// Serialize the record to bytes
	buf := make([]byte, recordSize)
	binary.LittleEndian.PutUint64(buf[0:8], rec.ID)
	binary.LittleEndian.PutUint64(buf[8:16], rec.Value)
	copy(buf[16:64], rec.Name[:])

	// pwrite: write at the specified offset without changing f_pos.
	// Multiple goroutines can call this simultaneously on the same fd
	// with different offsets — no data corruption.
	n, err := syscall.Pwrite(fd, buf, offset)
	if err != nil {
		return fmt.Errorf("pwrite at offset %d: %w", offset, err)
	}
	if n != recordSize {
		return fmt.Errorf("short pwrite: wrote %d of %d bytes", n, recordSize)
	}
	return nil
}

// readRecord reads a record from a specific position using pread().
// Like pwrite, this is atomic and doesn't affect the file offset.
func readRecord(fd int, recordNum int) (*record, error) {
	offset := int64(recordNum) * recordSize
	buf := make([]byte, recordSize)

	// pread: read at the specified offset without changing f_pos.
	n, err := syscall.Pread(fd, buf, offset)
	if err != nil {
		return nil, fmt.Errorf("pread at offset %d: %w", offset, err)
	}
	if n != recordSize {
		return nil, fmt.Errorf("short pread: read %d of %d bytes", n, recordSize)
	}

	// Deserialize the record from bytes
	rec := &record{
		ID:    binary.LittleEndian.Uint64(buf[0:8]),
		Value: binary.LittleEndian.Uint64(buf[8:16]),
	}
	copy(rec.Name[:], buf[16:64])
	return rec, nil
}

func main() {
	const numRecords = 1000
	const numWriters = 10
	dbFile := "/tmp/pread_pwrite_demo.db"

	// Open the file with read-write access
	fd, err := syscall.Open(dbFile,
		syscall.O_RDWR|syscall.O_CREAT|syscall.O_TRUNC, 0644)
	if err != nil {
		fmt.Fprintf(os.Stderr, "open: %v\n", err)
		os.Exit(1)
	}
	defer syscall.Close(fd)
	defer os.Remove(dbFile)

	// Pre-allocate the file to hold all records.
	// fallocate is more efficient than writing zeros.
	syscall.Fallocate(fd, 0, 0, int64(numRecords*recordSize))

	fmt.Printf("Writing %d records concurrently with %d goroutines using pwrite...\n",
		numRecords, numWriters)

	// Launch multiple writers concurrently. Each writer handles a range
	// of record numbers. They all share the same fd but use pwrite,
	// so there are no race conditions on the file offset.
	var wg sync.WaitGroup
	recordsPerWriter := numRecords / numWriters

	for w := 0; w < numWriters; w++ {
		wg.Add(1)
		go func(writerID int) {
			defer wg.Done()

			startRec := writerID * recordsPerWriter
			endRec := startRec + recordsPerWriter

			for i := startRec; i < endRec; i++ {
				rec := &record{
					ID:    uint64(i),
					Value: uint64(i * 100),
				}
				name := fmt.Sprintf("record-%04d-by-writer-%d", i, writerID)
				copy(rec.Name[:], name)

				if err := writeRecord(fd, rec, i); err != nil {
					fmt.Fprintf(os.Stderr, "writer %d: %v\n", writerID, err)
					return
				}
			}
		}(w)
	}

	wg.Wait()
	fmt.Println("All writes complete.")

	// Verify: read back some records using pread
	fmt.Println("\n--- Verifying random records with pread ---")
	for _, recNum := range []int{0, 42, 500, 999} {
		rec, err := readRecord(fd, recNum)
		if err != nil {
			fmt.Fprintf(os.Stderr, "Error reading record %d: %v\n", recNum, err)
			continue
		}
		// Trim null bytes from the name
		name := string(rec.Name[:])
		for i, b := range rec.Name {
			if b == 0 {
				name = string(rec.Name[:i])
				break
			}
		}
		fmt.Printf("Record %4d: ID=%d, Value=%d, Name=%q\n",
			recNum, rec.ID, rec.Value, name)
	}

	fmt.Println("\nKey takeaway: All goroutines shared ONE file descriptor.")
	fmt.Println("pwrite/pread made concurrent access safe without any mutexes.")
}
```

---

## 3.4 I/O Multiplexing — Handling Many Connections

### 3.4.1 The Problem

Imagine a server handling 10,000 concurrent TCP connections. Without I/O multiplexing,
you'd need 10,000 threads — each blocked in `read()`, waiting for data. That's 10,000
stacks (~80 MB of memory) and massive context-switching overhead.

I/O multiplexing lets a **single thread** monitor many file descriptors and be notified
when any of them is ready for reading or writing.

### 3.4.2 select() — The Ancient Way

```c
// select() API (simplified)
int select(int nfds,
           fd_set *readfds,    // set of fds to watch for readability
           fd_set *writefds,   // set of fds to watch for writability
           fd_set *exceptfds,  // set of fds to watch for exceptions
           struct timeval *timeout);
```

**Why select() is terrible for high-performance servers:**

1. **FD_SETSIZE limit** — typically 1024. Cannot monitor more than 1024 fds.
2. **Linear scan** — kernel must check every bit in the fd_set on every call.
3. **Copy overhead** — entire fd_set is copied to/from kernel on every call.
4. **Destroys input** — the fd_set is modified in place; must rebuild it each time.

### 3.4.3 poll() — Slightly Better

```c
int poll(struct pollfd *fds, nfds_t nfds, int timeout);

struct pollfd {
    int   fd;         // file descriptor
    short events;     // requested events (POLLIN, POLLOUT)
    short revents;    // returned events
};
```

poll() fixes the FD_SETSIZE limit (uses an array instead of a bitmap) and doesn't
destroy its input. But it still has the **linear scan problem** — the kernel must
check every fd in the array on every call. O(n) per call where n is the number
of monitored fds.

### 3.4.4 epoll — The Linux Way

epoll solves all of select/poll's problems with a fundamentally different architecture:

```
┌──────────────────────────────────────────────────────────────────┐
│                        Kernel Space                               │
│                                                                  │
│   epoll_create() creates:                                        │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │              epoll instance                              │    │
│   │                                                         │    │
│   │   ┌─────────────────────┐    ┌──────────────────┐       │    │
│   │   │  Interest List      │    │  Ready List       │       │    │
│   │   │  (Red-Black Tree)   │    │  (Linked List)    │       │    │
│   │   │                     │    │                    │       │    │
│   │   │  fd=4: EPOLLIN     │    │  fd=7 → fd=42 →   │       │    │
│   │   │  fd=7: EPOLLIN|OUT │    │  (fds with events) │       │    │
│   │   │  fd=9: EPOLLIN     │    │                    │       │    │
│   │   │  fd=42: EPOLLOUT   │    │                    │       │    │
│   │   │  ...               │    │                    │       │    │
│   │   └─────────────────────┘    └──────────────────┘       │    │
│   │                                                         │    │
│   │   When data arrives on fd=7:                             │    │
│   │   1. Network stack receives packet                       │    │
│   │   2. Wakes up socket's wait queue                        │    │
│   │   3. Callback moves fd=7 to the ready list              │    │
│   │   4. epoll_wait() returns immediately with fd=7          │    │
│   │                                                         │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**The three epoll syscalls:**

```c
// 1. Create an epoll instance. Returns an fd for the epoll object.
int epoll_create1(int flags);

// 2. Add/modify/remove fds from the interest list.
//    op: EPOLL_CTL_ADD, EPOLL_CTL_MOD, EPOLL_CTL_DEL
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

// 3. Wait for events. Returns the number of ready fds.
//    Only returns fds that ACTUALLY have events — O(ready), not O(total)!
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

**Why epoll is O(1) per event:**

- `epoll_ctl` adds/removes fds from a red-black tree — O(log n) per operation,
  but you only call it when connections arrive/depart, not on every wait.
- When data arrives, the kernel's network stack calls a **callback** that adds
  the fd to the ready list — O(1).
- `epoll_wait` just sweeps the ready list — O(ready fds), not O(total fds).

For a server with 100,000 connections where 50 are active at any moment:
- select/poll: check 100,000 fds → O(100,000)
- epoll: return 50 ready fds → O(50)

### 3.4.5 Level-Triggered vs Edge-Triggered epoll

**Level-triggered (LT)** — the default:
- `epoll_wait` returns a fd as ready **as long as** the condition is true
- If there's data to read, the fd is reported as ready on every `epoll_wait` call
- Simpler to program — you can read partial data and come back later

**Edge-triggered (ET)** — with `EPOLLET` flag:
- `epoll_wait` returns a fd **only when the state changes** (e.g., new data arrives)
- You must read ALL available data when notified, or you'll miss events
- More efficient — fewer epoll_wait wakeups
- Requires non-blocking I/O and careful handling

```
Time →

Data arrives:        ████████████████████████████████
                          (8 KB of data in socket buffer)

Level-Triggered:
  epoll_wait #1:     READY  (8 KB available)
  read 2 KB
  epoll_wait #2:     READY  (6 KB still available) ← reported again!
  read 6 KB
  epoll_wait #3:     (blocks — no data)

Edge-Triggered:
  epoll_wait #1:     READY  (state changed: data arrived)
  read 2 KB
  epoll_wait #2:     (blocks! state hasn't changed) ← NOT reported again!
  ...data sits unread until MORE data arrives...
  
  Correct ET usage: read ALL data until EAGAIN when notified
```

### 3.4.6 How Go's netpoller Uses epoll Under the Hood

Go's `net` package appears to use goroutine-per-connection (one goroutine per
`net.Conn`), but under the hood, it uses epoll (on Linux). Here's how:

```
Go Application Layer:
  goroutine 1: conn.Read()  → blocks (from goroutine's perspective)
  goroutine 2: conn.Read()  → blocks
  goroutine 3: conn.Write() → blocks
  ...10,000 goroutines...

Go Runtime Layer (netpoller):
  ┌─────────────────────────────────────────────────┐
  │  runtime.netpoll()                               │
  │                                                 │
  │  1. Registers all waiting goroutines' fds with  │
  │     epoll via epoll_ctl(EPOLL_CTL_ADD)          │
  │                                                 │
  │  2. Calls epoll_wait() — blocks until events    │
  │                                                 │
  │  3. For each ready fd:                          │
  │     - Find the goroutine waiting on this fd     │
  │     - Mark it as runnable                       │
  │     - Schedule it on a P (processor)            │
  │                                                 │
  │  Only 1 actual thread doing epoll_wait!         │
  └─────────────────────────────────────────────────┘
```

This is why Go can handle millions of goroutines doing network I/O — they're all
multiplexed onto one (or a few) epoll instances by the runtime. The goroutine model
gives you the **simplicity** of blocking I/O with the **performance** of event-driven I/O.

### 3.4.7 Hands-On: Raw epoll-based TCP Server in Go

```go
// epoll_server.go
//
// This program builds a TCP echo server using raw epoll syscalls,
// WITHOUT using Go's net package. This demonstrates exactly what
// happens under the hood of Go's network stack and event-driven
// servers like nginx.
//
// Architecture:
//   1. Create a listening socket (socket → bind → listen)
//   2. Create an epoll instance
//   3. Add the listener to epoll
//   4. Event loop: epoll_wait → handle events
//      - Listener ready: accept new connections, add to epoll
//      - Connection ready: read data, echo it back
//
// Run: go build -o epoll_server epoll_server.go && ./epoll_server
// Test: echo "hello" | nc localhost 8080
//
// NOTE: This is Linux-only (epoll is a Linux-specific API).
package main

import (
	"fmt"
	"os"
	"syscall"
	"unsafe"
)

const (
	// maxEvents is the maximum number of events epoll_wait will return
	// per call. This limits how many ready fds we process per iteration.
	// In production, 128 or 256 is common.
	maxEvents = 128

	// listenBacklog is the maximum length of the pending connections queue.
	// When this is full, new SYN packets are dropped (client sees timeout).
	listenBacklog = 128

	// listenPort is the TCP port we bind to.
	listenPort = 8080

	// bufSize is the read buffer size per read() call.
	bufSize = 4096
)

// setNonBlocking sets a file descriptor to non-blocking mode.
// In non-blocking mode, read() returns EAGAIN instead of blocking
// when there's no data available. This is ESSENTIAL for epoll
// edge-triggered mode and highly recommended for level-triggered too.
func setNonBlocking(fd int) error {
	// Get current flags
	flags, _, errno := syscall.Syscall(syscall.SYS_FCNTL,
		uintptr(fd), uintptr(syscall.F_GETFL), 0)
	if errno != 0 {
		return fmt.Errorf("fcntl F_GETFL: %v", errno)
	}

	// Set O_NONBLOCK flag
	_, _, errno = syscall.Syscall(syscall.SYS_FCNTL,
		uintptr(fd), uintptr(syscall.F_SETFL), flags|uintptr(syscall.O_NONBLOCK))
	if errno != 0 {
		return fmt.Errorf("fcntl F_SETFL: %v", errno)
	}

	return nil
}

// createListenSocket creates a TCP listening socket bound to the given port.
// This is the low-level equivalent of net.Listen("tcp", ":8080").
//
// Steps:
//   1. socket()  — create a socket fd
//   2. setsockopt() — set SO_REUSEADDR (allows immediate rebind after restart)
//   3. bind()    — associate the socket with an address and port
//   4. listen()  — mark the socket as passive (accepts connections)
func createListenSocket(port int) (int, error) {
	// Create a TCP socket (AF_INET = IPv4, SOCK_STREAM = TCP)
	// SOCK_NONBLOCK makes the socket non-blocking from creation
	// SOCK_CLOEXEC sets the close-on-exec flag (security best practice)
	fd, err := syscall.Socket(
		syscall.AF_INET,
		syscall.SOCK_STREAM|syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC,
		0,
	)
	if err != nil {
		return -1, fmt.Errorf("socket: %w", err)
	}

	// SO_REUSEADDR allows binding to an address that's in TIME_WAIT state.
	// Without this, restarting the server immediately after stopping it
	// would fail with "address already in use" for ~60 seconds.
	err = syscall.SetsockoptInt(fd, syscall.SOL_SOCKET, syscall.SO_REUSEADDR, 1)
	if err != nil {
		syscall.Close(fd)
		return -1, fmt.Errorf("setsockopt SO_REUSEADDR: %w", err)
	}

	// Bind to 0.0.0.0:port (all interfaces)
	addr := syscall.SockaddrInet4{Port: port}
	copy(addr.Addr[:], []byte{0, 0, 0, 0})

	if err := syscall.Bind(fd, &addr); err != nil {
		syscall.Close(fd)
		return -1, fmt.Errorf("bind: %w", err)
	}

	// Mark as a passive socket (listening for connections)
	if err := syscall.Listen(fd, listenBacklog); err != nil {
		syscall.Close(fd)
		return -1, fmt.Errorf("listen: %w", err)
	}

	return fd, nil
}

func main() {
	// Step 1: Create the listening socket
	listenFD, err := createListenSocket(listenPort)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create listener: %v\n", err)
		os.Exit(1)
	}
	defer syscall.Close(listenFD)
	fmt.Printf("Listening on :%d (fd=%d)\n", listenPort, listenFD)

	// Step 2: Create an epoll instance.
	// epoll_create1(EPOLL_CLOEXEC) returns an fd that represents the epoll instance.
	// This fd is itself a file descriptor that can be polled, closed, etc.
	epollFD, err := syscall.EpollCreate1(syscall.EPOLL_CLOEXEC)
	if err != nil {
		fmt.Fprintf(os.Stderr, "epoll_create1: %v\n", err)
		os.Exit(1)
	}
	defer syscall.Close(epollFD)
	fmt.Printf("Created epoll instance (fd=%d)\n", epollFD)

	// Step 3: Add the listener to epoll's interest list.
	// We watch for EPOLLIN (ready to accept) events.
	// EPOLLET enables edge-triggered mode for maximum performance.
	listenerEvent := syscall.EpollEvent{
		Events: syscall.EPOLLIN | syscall.EPOLLET,
		Fd:     int32(listenFD),
	}
	if err := syscall.EpollCtl(epollFD, syscall.EPOLL_CTL_ADD, listenFD, &listenerEvent); err != nil {
		fmt.Fprintf(os.Stderr, "epoll_ctl ADD listener: %v\n", err)
		os.Exit(1)
	}

	// Allocate the events buffer outside the loop to avoid GC pressure
	events := make([]syscall.EpollEvent, maxEvents)
	buf := make([]byte, bufSize)

	// Track active connections for statistics
	activeConns := 0

	fmt.Println("Entering event loop...")

	// Step 4: The event loop — the heart of every epoll-based server
	for {
		// epoll_wait blocks until at least one fd has events.
		// Returns the number of fds with events (up to maxEvents).
		// Timeout of -1 means wait indefinitely.
		nEvents, err := syscall.EpollWait(epollFD, events, -1)
		if err != nil {
			// EINTR happens when a signal interrupts the wait — just retry
			if err == syscall.EINTR {
				continue
			}
			fmt.Fprintf(os.Stderr, "epoll_wait: %v\n", err)
			break
		}

		// Process each ready event
		for i := 0; i < nEvents; i++ {
			fd := int(events[i].Fd)

			// Check for error/hangup conditions
			if events[i].Events&(syscall.EPOLLERR|syscall.EPOLLHUP) != 0 {
				fmt.Printf("  Connection fd=%d: error/hangup, closing\n", fd)
				syscall.Close(fd)
				activeConns--
				continue
			}

			if fd == listenFD {
				// Listener is ready — accept ALL pending connections
				// (edge-triggered: must drain the accept queue)
				for {
					connFD, sa, err := syscall.Accept4(listenFD,
						syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC)
					if err != nil {
						// EAGAIN/EWOULDBLOCK: no more pending connections
						if err == syscall.EAGAIN || err == syscall.EWOULDBLOCK {
							break
						}
						fmt.Fprintf(os.Stderr, "accept: %v\n", err)
						break
					}

					// Log the new connection
					if addr, ok := sa.(*syscall.SockaddrInet4); ok {
						fmt.Printf("  Accepted connection fd=%d from %d.%d.%d.%d:%d\n",
							connFD,
							addr.Addr[0], addr.Addr[1], addr.Addr[2], addr.Addr[3],
							addr.Port)
					}

					// Add the new connection to epoll's interest list.
					// EPOLLIN: notify when data arrives
					// EPOLLET: edge-triggered mode
					connEvent := syscall.EpollEvent{
						Events: syscall.EPOLLIN | syscall.EPOLLET,
						Fd:     int32(connFD),
					}
					if err := syscall.EpollCtl(epollFD, syscall.EPOLL_CTL_ADD, connFD, &connEvent); err != nil {
						fmt.Fprintf(os.Stderr, "epoll_ctl ADD conn: %v\n", err)
						syscall.Close(connFD)
						continue
					}
					activeConns++
				}
			} else {
				// A connected socket is ready — read and echo data.
				// Edge-triggered: must read ALL available data until EAGAIN.
				for {
					n, err := syscall.Read(fd, buf)
					if err != nil {
						if err == syscall.EAGAIN || err == syscall.EWOULDBLOCK {
							// No more data available right now — done for this fd
							break
						}
						// Real error — close the connection
						fmt.Printf("  Connection fd=%d: read error: %v\n", fd, err)
						syscall.Close(fd)
						activeConns--
						break
					}

					if n == 0 {
						// EOF — client closed the connection
						fmt.Printf("  Connection fd=%d: client disconnected\n", fd)
						syscall.Close(fd)
						activeConns--
						break
					}

					// Echo the data back to the client.
					// In production, you'd handle partial writes here.
					written := 0
					for written < n {
						w, err := syscall.Write(fd, buf[written:n])
						if err != nil {
							fmt.Printf("  Connection fd=%d: write error: %v\n", fd, err)
							break
						}
						written += w
					}
				}
			}
		}

		// Suppress "unused" warning for unsafe import needed for some platforms
		_ = unsafe.Sizeof(0)
	}

	fmt.Printf("Server stopped. Active connections at shutdown: %d\n", activeConns)
}
```

### 3.4.8 Hands-On: Comparing with Go's net.Listener

```go
// go_net_echo.go
//
// This is the SAME echo server built with Go's standard library.
// Compare the simplicity with the raw epoll version above.
//
// Under the hood, net.Listen and conn.Read use the EXACT same epoll
// mechanism — Go's runtime netpoller handles it transparently.
//
// You get goroutine-per-connection simplicity with epoll performance.
//
// Run: go run go_net_echo.go
// Test: echo "hello" | nc localhost 8081
package main

import (
	"fmt"
	"io"
	"net"
	"os"
)

func main() {
	// net.Listen creates a socket, binds, and listens — all in one call.
	// Internally, it also registers with the netpoller (epoll).
	ln, err := net.Listen("tcp", ":8081")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Listen: %v\n", err)
		os.Exit(1)
	}
	defer ln.Close()
	fmt.Println("Listening on :8081")

	for {
		// Accept blocks the current goroutine (not the OS thread!).
		// The runtime puts this goroutine to sleep and uses epoll_wait
		// to be notified when a new connection arrives.
		conn, err := ln.Accept()
		if err != nil {
			fmt.Fprintf(os.Stderr, "Accept: %v\n", err)
			continue
		}

		// One goroutine per connection — this looks like it would
		// create millions of OS threads, but goroutines are multiplexed
		// onto a small number of OS threads by the Go scheduler.
		go func(c net.Conn) {
			defer c.Close()
			// io.Copy reads from c and writes back to c (echo).
			// Each Read() call internally uses epoll to avoid blocking
			// an OS thread when no data is available.
			io.Copy(c, c)
		}(conn)
	}
}
```

**The key insight**: Both versions use epoll. The raw version shows you exactly what
happens; the net package version hides it behind a beautiful abstraction. In production,
use the net package — but now you know what's underneath.

---

## 3.5 io_uring — The Future of Linux I/O

### 3.5.1 Why io_uring Was Created

Traditional Linux I/O (even epoll) has fundamental overhead:

1. **System call overhead** — every I/O operation requires a user→kernel→user transition
2. **Data copying** — data often passes through kernel buffers
3. **Synchronous completion** — even with epoll, the actual read/write is a blocking syscall

io_uring (introduced in Linux 5.1, 2019) solves all three problems with a
**shared memory ring buffer** between user space and kernel space:

```
┌──────────────────────────────────────────────────────────────────────┐
│                          User Space                                   │
│                                                                      │
│   Application                                                        │
│       │                                                              │
│       │  1. Write SQE to SQ        4. Read CQE from CQ              │
│       ▼                                   ▲                          │
│   ┌───────────────────┐   ┌───────────────────┐                      │
│   │  Submission Queue  │   │  Completion Queue  │                     │
│   │  (SQ Ring)        │   │  (CQ Ring)         │                     │
│   │                   │   │                    │                     │
│   │  ┌─────┐ ┌─────┐ │   │  ┌─────┐ ┌─────┐  │                     │
│   │  │SQE 0│ │SQE 1│ │   │  │CQE 0│ │CQE 1│  │                     │
│   │  │read │ │write│ │   │  │done! │ │done! │  │                     │
│   │  │fd=5 │ │fd=7 │ │   │  │n=4096│ │n=512 │  │                     │
│   │  └─────┘ └─────┘ │   │  └─────┘ └─────┘  │                     │
│   │  tail →          │   │  head →            │                     │
│   └────────┬──────────┘   └──────────▲────────┘                      │
│            │     Shared Memory       │                               │
═════════════╪═════(mmap'd)════════════╪═══════════════════════════════│
│            │                         │                               │
│   ┌────────▼─────────────────────────┴────────┐                      │
│   │              Kernel I/O Engine             │                     │
│   │                                            │                     │
│   │  2. Kernel reads SQEs                      │                     │
│   │  3. Performs I/O, writes CQEs              │                     │
│   │                                            │                     │
│   │  Can batch multiple SQEs per syscall!      │                     │
│   │  Can even poll without ANY syscalls!        │                     │
│   └────────────────────────────────────────────┘                      │
│                          Kernel Space                                 │
└──────────────────────────────────────────────────────────────────────┘
```

**Key advantages over epoll + read/write:**

| Feature | epoll + read/write | io_uring |
|---|---|---|
| Syscalls per I/O | 2 (epoll_wait + read) | 0 (with SQPOLL mode) |
| Data copies | User→Kernel or Kernel→User | Zero-copy with registered buffers |
| Batching | No (one read per call) | Yes (submit many SQEs at once) |
| File I/O | Must use threads for non-blocking | Native async file I/O |
| Supported ops | Network only (epoll) | Read, write, send, recv, accept, connect, fsync, openat, statx, and 60+ more |

### 3.5.2 SQE and CQE Structures

```c
// Submission Queue Entry (SQE) — what you submit to the kernel
struct io_uring_sqe {
    __u8    opcode;      // Operation: IORING_OP_READ, IORING_OP_WRITE, etc.
    __u8    flags;       // IOSQE_FIXED_FILE, IOSQE_IO_LINK, etc.
    __u16   ioprio;      // I/O priority
    __s32   fd;          // File descriptor (or index into registered files)
    __u64   off;         // Offset in file (like pread/pwrite)
    __u64   addr;        // User buffer address
    __u32   len;         // Buffer length
    // ... more fields for specific operations
    __u64   user_data;   // Opaque value returned in CQE (your cookie!)
};

// Completion Queue Entry (CQE) — what the kernel returns to you
struct io_uring_cqe {
    __u64   user_data;   // Copied from SQE — identifies which request completed
    __s32   res;         // Result (bytes transferred, or negative errno)
    __u32   flags;       // IORING_CQE_F_BUFFER, etc.
};
```

### 3.5.3 io_uring Syscalls

```c
// Set up an io_uring instance. Returns the SQ and CQ ring parameters.
int io_uring_setup(unsigned entries, struct io_uring_params *p);

// Submit SQEs and/or wait for CQEs.
// min_complete: block until this many CQEs are ready
// With SQPOLL, you may never need to call this!
int io_uring_enter(int ring_fd, unsigned to_submit,
                   unsigned min_complete, unsigned flags);
```

### 3.5.4 Hands-On: Using io_uring from Go

```go
// iouring_demo.go
//
// This program demonstrates using io_uring from Go via the
// iceber/iouring-go library. io_uring provides true asynchronous I/O
// for both files and network operations, with zero-copy capability
// and minimal system call overhead.
//
// We'll perform asynchronous file reads — something that's impossible
// with regular epoll (epoll only works for network/pipe fds, not
	// regular files on most systems).
//
// Prerequisites:
//   go get github.com/iceber/iouring-go
//   Linux 5.1+ kernel
//
// Build: go build -o iouring_demo iouring_demo.go
package main

import (
	"fmt"
	"os"
	"unsafe"

	"github.com/iceber/iouring-go"
)

// asyncReadFile demonstrates reading a file asynchronously using io_uring.
// Unlike traditional read() which blocks until data is available,
// io_uring submits the request and returns immediately. The kernel
// completes the I/O in the background and notifies us via the CQ.
//
// This is particularly powerful for:
//   - Reading multiple files in parallel without threads
//   - High-throughput file I/O (databases, file servers)
//   - Mixing file and network I/O in the same event loop
func asyncReadFile(ring *iouring.IOURing, path string) ([]byte, error) {
	// Open the file (we could also do this via io_uring's OPENAT op!)
	f, err := os.Open(path)
	if err != nil {
		return nil, fmt.Errorf("open %s: %w", path, err)
	}
	defer f.Close()

	// Get the file size so we know how much to read
	stat, err := f.Stat()
	if err != nil {
		return nil, fmt.Errorf("stat %s: %w", path, err)
	}

	// Allocate a buffer for the file contents
	size := int(stat.Size())
	if size == 0 {
		size = 4096 // For procfs files that report size 0
	}
	buf := make([]byte, size)

	// Create a read request (SQE) and submit it.
	// The PrepareRead function fills in an SQE with:
	//   opcode: IORING_OP_READ
	//   fd:     the file descriptor
	//   addr:   pointer to our buffer
	//   len:    buffer size
	//   off:    0 (read from beginning)
	request, err := ring.PrepareRead(
		int(f.Fd()),           // File descriptor
		buf,                   // Buffer to read into
		uint64(0),             // Offset (0 = beginning of file)
	)
	if err != nil {
		return nil, fmt.Errorf("prepare read: %w", err)
	}

	// Wait for the request to complete.
	// In a real server, you'd submit many requests and wait for
	// any of them to complete — that's where io_uring shines.
	<-request.Done()

	// Check the result (equivalent to the return value of read())
	result := request.Result()
	if result < 0 {
		return nil, fmt.Errorf("io_uring read: error code %d", result)
	}

	return buf[:result], nil
}

func main() {
	// Create an io_uring instance with 32 SQ entries.
	// The kernel may allocate more CQ entries (typically 2x SQ entries).
	ring, err := iouring.New(32)
	if err != nil {
		fmt.Fprintf(os.Stderr, "io_uring setup failed: %v\n", err)
		fmt.Fprintf(os.Stderr, "(Requires Linux 5.1+ kernel)\n")
		os.Exit(1)
	}
	defer ring.Close()

	fmt.Println("io_uring instance created successfully")
	fmt.Printf("SQ entries: %d\n", 32)

	// Read /proc/version asynchronously
	fmt.Println("\n--- Async read of /proc/version ---")
	data, err := asyncReadFile(ring, "/proc/version")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
	} else {
		fmt.Printf("Contents (%d bytes): %s", len(data), string(data))
	}

	// Read multiple files concurrently using io_uring batching
	fmt.Println("\n--- Batch async reads ---")
	files := []string{
		"/proc/uptime",
		"/proc/loadavg",
		"/proc/meminfo",
	}

	for _, path := range files {
		data, err := asyncReadFile(ring, path)
		if err != nil {
			fmt.Fprintf(os.Stderr, "Error reading %s: %v\n", path, err)
			continue
		}
		// Show just the first line of each file
		firstLine := string(data)
		for i, c := range firstLine {
			if c == '\n' {
				firstLine = firstLine[:i]
				break
			}
		}
		fmt.Printf("%-20s → %s\n", path, firstLine)
	}

	// Demonstrate the zero-copy concept
	fmt.Println("\n--- io_uring advantages ---")
	fmt.Println("1. Submissions via shared memory (no syscall needed with SQPOLL)")
	fmt.Println("2. Batch multiple I/O operations in one submission")
	fmt.Println("3. True async file I/O (epoll can't do this for regular files)")
	fmt.Println("4. Registered buffers avoid repeated page pinning")
	fmt.Printf("5. SQE size: %d bytes, CQE size: %d bytes (cache-friendly)\n",
		unsafe.Sizeof(iouring.SubmissionQueueEntry{}),
		unsafe.Sizeof(iouring.CompletionQueueEntry{}))
}
```

### 3.5.5 io_uring vs epoll — When to Use What

| Use Case | Recommendation |
|---|---|
| Network server (many connections) | **epoll** (via Go's net package) — mature, well-tested |
| High-throughput file I/O | **io_uring** — true async file reads/writes |
| Database engine | **io_uring** — batch submissions, zero-copy, async fsync |
| Mixed file + network I/O | **io_uring** — unified interface for both |
| Compatibility (older kernels) | **epoll** — works on Linux 2.6+ |
| Simple Go server | **net package** — uses epoll automatically |

---

## 3.6 sendfile, splice, tee — Zero-Copy I/O

### 3.6.1 The Copy Problem

Consider a simple file server that reads a file and sends it over a network socket.
With normal I/O, data is copied **four times**:

```
Traditional file serving (4 copies!):

Disk → [DMA copy] → Kernel Page Cache → [CPU copy] → User Buffer
                                                          │
User Buffer → [CPU copy] → Kernel Socket Buffer → [DMA copy] → NIC

1. DMA copy:  Disk → Kernel buffer (page cache)
2. CPU copy:  Kernel buffer → User space buffer     (read syscall)
3. CPU copy:  User space buffer → Socket buffer     (write syscall)
4. DMA copy:  Socket buffer → Network card

Also: 4 context switches (read: user→kernel→user, write: user→kernel→user)
```

### 3.6.2 sendfile() — Kernel-to-Kernel Transfer

`sendfile()` eliminates copies #2 and #3 by keeping data in the kernel:

```
sendfile() optimization (2 copies + 0 user-space involvement):

Disk → [DMA copy] → Kernel Page Cache → [DMA scatter/gather] → NIC

1. DMA copy:     Disk → Page cache (if not already cached)
2. DMA S/G copy: Page cache → NIC (with scatter/gather DMA)

Data NEVER enters user space! Only 2 context switches (one syscall).
```

```c
// sendfile: transfer data between file descriptors entirely in kernel space
// Returns the number of bytes transferred, or -1 on error
ssize_t sendfile(int out_fd,      // destination (must be a socket)
                 int in_fd,       // source (must support mmap — regular file)
                 off_t *offset,   // position in source file (updated on return)
                 size_t count);   // number of bytes to transfer
```

### 3.6.3 splice() and tee()

`splice()` is more general than `sendfile()` — it moves data between a pipe and
any file descriptor, **without copying through user space**.

```c
// splice: move data between a pipe and a file descriptor
ssize_t splice(int fd_in, off_t *off_in,
               int fd_out, off_t *off_out,
               size_t len, unsigned int flags);

// tee: duplicate data in a pipe (zero-copy)
// After tee(), both the original pipe and the duplicate have the same data
ssize_t tee(int fd_in, int fd_out, size_t len, unsigned int flags);
```

**splice() data flow:**

```
File → splice → Pipe → splice → Socket

The data flows through the pipe's kernel buffer.
No user-space copies at all. The pipe acts as a kernel relay.

Useful for:
  - Proxies (splice from incoming socket → pipe → outgoing socket)
  - Log tee (tee the pipe, splice one copy to log, other to destination)
```

### 3.6.4 Hands-On: Zero-Copy File Server in Go

```go
// zerocopy_server.go
//
// This program implements a simple HTTP file server that uses sendfile()
// for zero-copy file transfers. When a client requests a file, the data
// flows directly from the page cache to the network card without ever
// entering user space.
//
// This is the technique used by nginx, Apache, and other high-performance
// web servers. Go's net/http package also uses sendfile under the hood
// when you serve files with http.ServeFile() or http.FileServer().
//
// Architecture:
//   Client ←──[sendfile]──→ Kernel ←──[page cache]──→ Disk
//   (No user-space data copies!)
//
// Run: go build -o zerocopy_server zerocopy_server.go && ./zerocopy_server
// Test: curl http://localhost:8082/etc/hosts
package main

import (
	"fmt"
	"net"
	"os"
	"strconv"
	"strings"
	"syscall"
)

// sendFileZeroCopy uses the sendfile(2) system call to transfer
// data from a regular file to a network socket without copying
// data through user space.
//
// Parameters:
//   - socketFD: the destination socket file descriptor
//   - fileFD:   the source file file descriptor
//   - offset:   starting position in the source file
//   - count:    number of bytes to transfer
//
// Returns the total number of bytes transferred.
//
// Under the hood, sendfile:
//   1. Checks if the source file's pages are in the page cache
//   2. If not, triggers a page fault to bring them in from disk
//   3. Sets up DMA scatter/gather to transfer directly from
//      page cache pages to the network card's buffer
//   4. No CPU copy involved (with modern hardware/kernels)
func sendFileZeroCopy(socketFD, fileFD int, offset *int64, count int) (int, error) {
	totalSent := 0

	for totalSent < count {
		// sendfile may transfer less than requested (like write).
		// We loop until all data is sent or an error occurs.
		n, err := syscall.Sendfile(socketFD, fileFD, offset, count-totalSent)
		if err != nil {
			return totalSent, fmt.Errorf("sendfile: %w", err)
		}
		if n == 0 {
			break // EOF
		}
		totalSent += n
	}

	return totalSent, nil
}

// handleConnection processes a single HTTP request using zero-copy sendfile.
// This is a minimal HTTP implementation — just enough to demonstrate sendfile.
func handleConnection(conn net.Conn) {
	defer conn.Close()

	// Read the HTTP request (simplified — just get the path)
	buf := make([]byte, 4096)
	n, err := conn.Read(buf)
	if err != nil {
		return
	}

	request := string(buf[:n])
	lines := strings.Split(request, "\r\n")
	if len(lines) == 0 {
		return
	}

	// Parse "GET /path HTTP/1.1"
	parts := strings.Fields(lines[0])
	if len(parts) < 2 || parts[0] != "GET" {
		sendHTTPError(conn, 400, "Bad Request")
		return
	}

	// Map URL path to file path (simplified — don't do this in production!)
	filePath := parts[1]
	if filePath == "/" {
		filePath = "/etc/hostname"
	}

	// Open the requested file
	f, err := os.Open(filePath)
	if err != nil {
		sendHTTPError(conn, 404, "Not Found")
		return
	}
	defer f.Close()

	// Get file size for Content-Length header
	stat, err := f.Stat()
	if err != nil || stat.IsDir() {
		sendHTTPError(conn, 403, "Forbidden")
		return
	}
	fileSize := stat.Size()

	// Send HTTP response headers (these go through normal write)
	header := fmt.Sprintf("HTTP/1.1 200 OK\r\n"+
		"Content-Length: %d\r\n"+
		"Content-Type: application/octet-stream\r\n"+
		"X-Transfer-Method: sendfile\r\n"+
		"\r\n", fileSize)
	conn.Write([]byte(header))

	// Get the raw socket fd from the net.Conn.
	// We need this for the sendfile syscall.
	rawConn, err := conn.(*net.TCPConn).SyscallConn()
	if err != nil {
		fmt.Fprintf(os.Stderr, "SyscallConn: %v\n", err)
		return
	}

	// Use sendfile for the response body — zero-copy!
	var sendErr error
	var bytesSent int
	rawConn.Control(func(socketFD uintptr) {
			offset := int64(0)
			bytesSent, sendErr = sendFileZeroCopy(
				int(socketFD),
				int(f.Fd()),
				&offset,
				int(fileSize),
			)
		})

	if sendErr != nil {
		fmt.Fprintf(os.Stderr, "sendfile error: %v\n", sendErr)
		return
	}

	fmt.Printf("Served %s: %d bytes via sendfile (zero-copy)\n",
		filePath, bytesSent)
}

// sendHTTPError sends a simple HTTP error response.
func sendHTTPError(conn net.Conn, code int, message string) {
	body := fmt.Sprintf("<h1>%d %s</h1>", code, message)
	response := "HTTP/1.1 " + strconv.Itoa(code) + " " + message + "\r\n" +
	"Content-Length: " + strconv.Itoa(len(body)) + "\r\n" +
	"Content-Type: text/html\r\n" +
	"\r\n" +
	body
	conn.Write([]byte(response))
}

func main() {
	ln, err := net.Listen("tcp", ":8082")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Listen: %v\n", err)
		os.Exit(1)
	}
	defer ln.Close()

	fmt.Println("Zero-copy file server listening on :8082")
	fmt.Println("Try: curl http://localhost:8082/etc/hosts")
	fmt.Println("     curl http://localhost:8082/proc/cpuinfo")

	for {
		conn, err := ln.Accept()
		if err != nil {
			fmt.Fprintf(os.Stderr, "Accept: %v\n", err)
			continue
		}
		go handleConnection(conn)
	}
}
```

**How Go's standard library uses sendfile:**

When you call `http.ServeFile()` or use `http.FileServer()`, Go detects that the
source is a regular file and the destination is a TCP socket, and automatically
uses `sendfile()`. You get zero-copy for free! The relevant code is in
`net/http/fs.go` and `internal/poll/sendfile_linux.go`.

---

## 3.7 Filesystem Types Deep Dive

### 3.7.1 ext4 — The Linux Workhorse

ext4 (Fourth Extended Filesystem) is the default filesystem for most Linux
distributions. It evolved from ext2 → ext3 → ext4 over two decades.

**Key features:**

| Feature | Description |
|---|---|
| **Journaling** | Metadata changes are written to a journal before the actual filesystem. On crash, replay the journal to restore consistency. |
| **Extents** | Contiguous blocks described by (start_block, count) instead of individual block pointers. Reduces metadata for large files dramatically. |
| **Delayed allocation** | Don't allocate disk blocks until data is actually flushed. Allows the allocator to see the full picture and reduce fragmentation. |
| **Multi-block allocator** | Allocates many blocks at once, improving sequential write performance. |
| **Max file size** | 16 TiB (with 4K blocks) |
| **Max filesystem size** | 1 EiB |
| **Max filename** | 255 bytes |

**ext4 on-disk layout:**

```
┌──────┬──────────┬──────────┬──────────┬──────────┬─────────────────┐
│Boot  │  Block   │  Block   │  Block   │  Block   │                 │
│Sector│ Group 0  │ Group 1  │ Group 2  │ Group 3  │  ...            │
│(1KB) │          │          │          │          │                 │
└──────┴──────────┴──────────┴──────────┴──────────┴─────────────────┘

Each Block Group:
┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│  Super   │  Group   │  Block   │  Inode   │  Inode   │  Data    │
│  Block   │  Desc.   │  Bitmap  │  Bitmap  │  Table   │  Blocks  │
│  (copy)  │  Table   │          │          │          │          │
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

**Extents vs block pointers (ext2/3 vs ext4):**

```
ext2/ext3: indirect block pointers
  inode → [blk0][blk1]...[blk11] → [indirect block] → [blocks]
                                  → [double indirect] → [indirect] → [blocks]
                                  → [triple indirect] → ...

  For a 1 GB file: thousands of block pointer lookups!

ext4: extents
  inode → extent: (start=1000, count=262144)   ← one extent = 1 GB contiguous!

  For a 1 GB contiguous file: ONE extent lookup!
```

### 3.7.2 XFS — High-Performance at Scale

XFS was developed by SGI for IRIX and ported to Linux. It excels at:

- **Large files** — designed for files in the terabyte range
- **Parallel I/O** — allocation groups allow concurrent allocations
- **Metadata scalability** — B+ trees for everything

**Key XFS concepts:**

```
XFS Filesystem Layout:
┌────────────────┬────────────────┬────────────────┬────────────────┐
│ Allocation     │ Allocation     │ Allocation     │ Allocation     │
│ Group 0        │ Group 1        │ Group 2        │ Group 3        │
│                │                │                │                │
│ ┌────────────┐ │ ┌────────────┐ │ ┌────────────┐ │ ┌────────────┐ │
│ │ AG Header  │ │ │ AG Header  │ │ │ AG Header  │ │ │ AG Header  │ │
│ │ Free Space │ │ │ Free Space │ │ │ Free Space │ │ │ Free Space │ │
│ │ B+ Tree    │ │ │ B+ Tree    │ │ │ B+ Tree    │ │ │ B+ Tree    │ │
│ │ Inode B+   │ │ │ Inode B+   │ │ │ Inode B+   │ │ │ Inode B+   │ │
│ │ Tree       │ │ │ Tree       │ │ │ Tree       │ │ │ Tree       │ │
│ │ Data Blocks│ │ │ Data Blocks│ │ │ Data Blocks│ │ │ Data Blocks│ │
│ └────────────┘ │ └────────────┘ │ └────────────┘ │ └────────────┘ │
└────────────────┴────────────────┴────────────────┴────────────────┘

Each AG is independently managed → concurrent operations on different AGs
don't contend for the same locks!
```

### 3.7.3 tmpfs — RAM-Backed Filesystem

tmpfs lives entirely in memory (and swap). No disk I/O at all.

```bash
# tmpfs is already mounted on most Linux systems:
$ mount | grep tmpfs
tmpfs on /run type tmpfs (rw,nosuid,nodev,size=3291812k,mode=755)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev)
tmpfs on /tmp type tmpfs (rw,nosuid,nodev)

# Create your own tmpfs:
$ mount -t tmpfs -o size=1G mytmpfs /mnt/ramdisk
```

**When to use tmpfs:**
- Build directories (compile artifacts — fast writes, don't need persistence)
- Temporary file processing
- `/dev/shm` for POSIX shared memory
- Test fixtures that need filesystem semantics but not durability

### 3.7.4 overlayfs — Union Filesystem (Containers!)

overlayfs is the filesystem behind Docker/container images. It merges multiple
directory trees ("layers") into a single unified view:

```
Container's view (merged):
┌─────────────────────────────────────────┐
│  /bin/sh  /etc/passwd  /app/server      │
│  (from    (from        (from            │
│  lower)   upper)       upper)           │
└─────────────────────────────────────────┘
                │
                │ overlayfs merges
                │
     ┌──────────┴──────────┐
     │                     │
┌────▼─────┐         ┌────▼─────┐
│  Upper   │         │  Lower   │  (read-only image layer)
│  Layer   │         │  Layer   │
│ (r/w)    │         │          │
│          │         │ /bin/sh  │
│ /etc/    │         │ /etc/    │
│  passwd  │         │  passwd  │  ← shadowed by upper
│ /app/    │         │ /lib/    │
│  server  │         │  ...     │
└──────────┘         └──────────┘

Read: Check upper first, then lower
Write: Copy-on-write to upper layer
Delete: "Whiteout" file in upper layer
```

### 3.7.5 Hands-On: Creating and Mounting an overlayfs in Go

```go
// overlayfs_demo.go
//
// This program demonstrates creating and using an overlayfs mount
// from Go. Overlayfs is the filesystem technology that makes Docker
// container images efficient — multiple containers can share the same
// read-only base image, with each container having its own writable layer.
//
// How overlayfs works:
//   - Lower layer: read-only base (e.g., Ubuntu base image)
//   - Upper layer: writable layer for changes (container's modifications)
//   - Work dir: internal scratch space for overlayfs
//   - Merged dir: the unified view that users/containers see
//
// REQUIRES: root privileges (mount is a privileged operation)
//
// Run: sudo go run overlayfs_demo.go
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"syscall"
)

// setupOverlayFS creates the directory structure and mounts an overlayfs.
// Returns the path to the merged directory.
//
// The mount options for overlayfs are:
//   lowerdir=<path>  — read-only base layer(s), separated by ':'
//   upperdir=<path>  — writable layer for modifications
//   workdir=<path>   — internal directory used by overlayfs (must be empty,
	//                       on the same filesystem as upperdir)
func setupOverlayFS(baseDir string) (string, error) {
	// Create the four directories needed for overlayfs
	dirs := map[string]string{
		"lower":  filepath.Join(baseDir, "lower"),   // Read-only base layer
		"upper":  filepath.Join(baseDir, "upper"),   // Writable layer
		"work":   filepath.Join(baseDir, "work"),    // Internal scratch space
		"merged": filepath.Join(baseDir, "merged"),  // The unified mount point
	}

	for name, path := range dirs {
		if err := os.MkdirAll(path, 0755); err != nil {
			return "", fmt.Errorf("mkdir %s (%s): %w", name, path, err)
		}
	}

	// Populate the lower (base) layer with some files.
	// In Docker, this would be the image layers.
	lowerFiles := map[string]string{
		"base_config.txt": "# Base configuration\nversion=1.0\nmode=production\n",
		"README.md":       "# Base Image\nThis is the read-only base layer.\n",
		"data/info.txt":   "Original data from base layer.\n",
	}

	for name, content := range lowerFiles {
		path := filepath.Join(dirs["lower"], name)
		os.MkdirAll(filepath.Dir(path), 0755)
		if err := os.WriteFile(path, []byte(content), 0644); err != nil {
			return "", fmt.Errorf("write %s: %w", name, err)
		}
	}

	// Mount the overlayfs.
	// This is equivalent to:
	//   mount -t overlay overlay -o lowerdir=lower,upperdir=upper,workdir=work merged
	mountOpts := fmt.Sprintf("lowerdir=%s,upperdir=%s,workdir=%s",
		dirs["lower"], dirs["upper"], dirs["work"])

	err := syscall.Mount("overlay", dirs["merged"], "overlay", 0, mountOpts)
	if err != nil {
		return "", fmt.Errorf("mount overlayfs: %w (are you root?)", err)
	}

	return dirs["merged"], nil
}

func main() {
	baseDir := "/tmp/overlayfs_demo"

	// Clean up from any previous run
	syscall.Unmount(filepath.Join(baseDir, "merged"), 0)
	os.RemoveAll(baseDir)

	fmt.Println("=== OverlayFS Demo ===\n")

	// Step 1: Set up and mount the overlayfs
	fmt.Println("1. Setting up overlayfs mount...")
	mergedDir, err := setupOverlayFS(baseDir)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Setup failed: %v\n", err)
		os.Exit(1)
	}
	defer func() {
		syscall.Unmount(mergedDir, 0)
		os.RemoveAll(baseDir)
	}()
	fmt.Printf("   Mounted at: %s\n", mergedDir)

	// Step 2: Read files from the merged view (comes from lower layer)
	fmt.Println("\n2. Reading files from merged view (from lower layer):")
	data, _ := os.ReadFile(filepath.Join(mergedDir, "base_config.txt"))
	fmt.Printf("   base_config.txt: %s", string(data))

	data, _ = os.ReadFile(filepath.Join(mergedDir, "README.md"))
	fmt.Printf("   README.md: %s", string(data))

	// Step 3: Modify a file — triggers copy-on-write!
	// The original file in lower is untouched; a modified copy goes to upper.
	fmt.Println("\n3. Modifying base_config.txt (triggers copy-on-write)...")
	newConfig := "# Modified configuration\nversion=2.0\nmode=development\ncustom=true\n"
	os.WriteFile(filepath.Join(mergedDir, "base_config.txt"), []byte(newConfig), 0644)

	// Step 4: Create a new file — goes directly to upper layer
	fmt.Println("4. Creating new file app.log...")
	os.WriteFile(filepath.Join(mergedDir, "app.log"), []byte("Application started\n"), 0644)

	// Step 5: Verify the state of each layer
	fmt.Println("\n5. Examining layer contents:")

	// Lower layer should be UNCHANGED (read-only)
	lowerData, _ := os.ReadFile(filepath.Join(baseDir, "lower", "base_config.txt"))
	fmt.Printf("   Lower layer (unchanged): %s", string(lowerData))

	// Upper layer has the modified file (copy-on-write)
	upperData, _ := os.ReadFile(filepath.Join(baseDir, "upper", "base_config.txt"))
	fmt.Printf("   Upper layer (modified):  %s", string(upperData))

	// Merged view shows the upper layer's version
	mergedData, _ := os.ReadFile(filepath.Join(mergedDir, "base_config.txt"))
	fmt.Printf("   Merged view:             %s", string(mergedData))

	// Step 6: List all files in each layer
	fmt.Println("\n6. File listing comparison:")
	for _, layer := range []struct {
		name string
		path string
	}{
		{"Lower (base)", filepath.Join(baseDir, "lower")},
		{"Upper (changes)", filepath.Join(baseDir, "upper")},
		{"Merged (unified)", mergedDir},
	} {
		fmt.Printf("   %s:\n", layer.name)
		filepath.Walk(layer.path, func(path string, info os.FileInfo, err error) error {
				if err != nil || path == layer.path {
					return nil
				}
				rel, _ := filepath.Rel(layer.path, path)
				if info.IsDir() {
					fmt.Printf("     📁 %s/\n", rel)
				} else {
					fmt.Printf("     📄 %s (%d bytes)\n", rel, info.Size())
				}
				return nil
			})
	}

	fmt.Println("\n=== Key Takeaways ===")
	fmt.Println("• Lower layer was NEVER modified (read-only)")
	fmt.Println("• Modified files were copied to upper layer (copy-on-write)")
	fmt.Println("• New files went directly to upper layer")
	fmt.Println("• Merged view shows upper files shadowing lower files")
	fmt.Println("• This is exactly how Docker container layers work!")
}
```

### 3.7.6 procfs and sysfs — Kernel Interfaces

**procfs** (`/proc`) — Process and kernel information:
```
/proc/
├── 1/              ← Process 1 (init/systemd)
│   ├── cmdline     ← Command line arguments
│   ├── maps        ← Memory mappings
│   ├── fd/         ← Open file descriptors
│   ├── status      ← Process status (memory, signals, etc.)
│   └── ...
├── self/           ← Symlink to current process's directory
├── cpuinfo         ← CPU information
├── meminfo         ← Memory statistics
├── interrupts      ← Interrupt counters
├── sys/            ← Tunable kernel parameters
│   ├── vm/         ← Virtual memory tunables
│   ├── net/        ← Network tunables
│   └── fs/         ← Filesystem tunables
└── ...
```

**sysfs** (`/sys`) — Device and driver model:
```
/sys/
├── block/          ← Block devices (sda, nvme0n1, etc.)
├── bus/            ← Bus types (pci, usb, etc.)
├── class/          ← Device classes (net, block, tty, etc.)
├── devices/        ← Device tree (physical hierarchy)
├── fs/             ← Filesystem parameters
├── kernel/         ← Kernel objects
├── module/         ← Loaded kernel modules
└── power/          ← Power management
```

Both are "virtual" filesystems — files are generated dynamically by the kernel
when you read them, and writing to certain files changes kernel behavior.

---

## 3.8 File Locking

### 3.8.1 Why File Locking?

When multiple processes need to access the same file, you need coordination.
File locking provides mutual exclusion at the file or byte-range level.

### 3.8.2 flock() — Advisory Locking

`flock()` provides **whole-file** advisory locks:

```c
int flock(int fd, int operation);
// operation: LOCK_SH (shared/read), LOCK_EX (exclusive/write),
//            LOCK_UN (unlock), LOCK_NB (non-blocking)
```

**Advisory** means the lock is only enforced between cooperating processes that
also call `flock()`. A process that doesn't call `flock()` can freely read/write
the locked file. This is how most Unix file locking works.

### 3.8.3 fcntl() — POSIX Record Locking

`fcntl()` provides **byte-range** locks — you can lock specific portions of a file:

```c
struct flock {
    short l_type;    // F_RDLCK, F_WRLCK, F_UNLCK
    short l_whence;  // SEEK_SET, SEEK_CUR, SEEK_END
    off_t l_start;   // Offset of start of lock region
    off_t l_len;     // Length of lock region (0 = to EOF)
    pid_t l_pid;     // PID of lock owner (set by F_GETLK)
};

fcntl(fd, F_SETLK, &lock);   // Try to set lock (non-blocking)
fcntl(fd, F_SETLKW, &lock);  // Set lock (blocking — waits)
fcntl(fd, F_GETLK, &lock);   // Test for lock
```

### 3.8.4 Hands-On: Implementing a File-Based Mutex in Go

```go
// file_mutex.go
//
// This program implements a cross-process mutex using file locking.
// Multiple instances of this program can run simultaneously, and the
// file lock ensures only one enters the critical section at a time.
//
// This is a common pattern for:
//   - Ensuring only one instance of a daemon runs (PID files)
//   - Coordinating access to shared resources between processes
//   - Implementing distributed locks on shared filesystems (NFS)
//
// Run two instances simultaneously:
//   go run file_mutex.go &
//   go run file_mutex.go &
//
// You'll see them take turns entering the critical section.
package main

import (
	"fmt"
	"os"
	"syscall"
	"time"
)

// FileMutex implements a cross-process mutual exclusion lock using
// the flock(2) system call. Unlike sync.Mutex which only works within
// a single process, FileMutex coordinates between separate processes.
//
// The lock is associated with the file's inode (not the file descriptor
	// or path), so any process that opens the same file can participate in
// the locking protocol.
//
// The lock is automatically released when:
//   1. Unlock() is called explicitly
//   2. The file descriptor is closed
//   3. The process exits (clean or crash)
// Property #3 makes file locks crash-safe — no stale locks after crashes.
type FileMutex struct {
	path string   // Path to the lock file
	fd   int      // File descriptor for the lock file
}

// NewFileMutex creates a new file-based mutex. The lock file is created
// if it doesn't exist. The file itself is just a coordination point —
// its contents don't matter (though we write the PID for debugging).
func NewFileMutex(path string) (*FileMutex, error) {
	// Open or create the lock file.
	// O_CLOEXEC: don't leak this fd to child processes
	fd, err := syscall.Open(path,
		syscall.O_CREAT|syscall.O_RDWR|syscall.O_CLOEXEC, 0644)
	if err != nil {
		return nil, fmt.Errorf("open lock file %s: %w", path, err)
	}

	return &FileMutex{path: path, fd: fd}, nil
}

// Lock acquires an exclusive lock on the file. If another process holds
// the lock, this call blocks until the lock is released.
//
// Under the hood, flock(LOCK_EX) adds an entry to the inode's lock list
// in the kernel. If the lock is already held exclusively, the calling
// process is added to a wait queue and put to sleep.
func (m *FileMutex) Lock() error {
	// LOCK_EX: exclusive lock (only one holder at a time)
	// This blocks until the lock is available
	if err := syscall.Flock(m.fd, syscall.LOCK_EX); err != nil {
		return fmt.Errorf("flock LOCK_EX: %w", err)
	}

	// Write our PID to the lock file (for debugging — not required)
	syscall.Ftruncate(m.fd, 0)
	pid := fmt.Sprintf("%d\n", os.Getpid())
	syscall.Write(m.fd, []byte(pid))

	return nil
}

// TryLock attempts to acquire the lock without blocking.
// Returns true if the lock was acquired, false if it's held by another process.
//
// This is useful for "check if another instance is running" patterns:
//   if !mutex.TryLock() {
	//       log.Fatal("Another instance is already running")
	//   }
func (m *FileMutex) TryLock() (bool, error) {
	// LOCK_NB: non-blocking — return immediately if lock is held
	err := syscall.Flock(m.fd, syscall.LOCK_EX|syscall.LOCK_NB)
	if err != nil {
		if err == syscall.EWOULDBLOCK {
			return false, nil // Lock is held by another process
		}
		return false, fmt.Errorf("flock LOCK_EX|LOCK_NB: %w", err)
	}

	pid := fmt.Sprintf("%d\n", os.Getpid())
	syscall.Ftruncate(m.fd, 0)
	syscall.Write(m.fd, []byte(pid))

	return true, nil
}

// RLock acquires a shared (read) lock. Multiple processes can hold
// shared locks simultaneously, but a shared lock blocks exclusive locks
// and vice versa.
//
// Use pattern:
//   - Readers take LOCK_SH (can be concurrent)
//   - Writers take LOCK_EX (exclusive, blocks all others)
func (m *FileMutex) RLock() error {
	return syscall.Flock(m.fd, syscall.LOCK_SH)
}

// Unlock releases the lock, allowing other waiting processes to proceed.
func (m *FileMutex) Unlock() error {
	return syscall.Flock(m.fd, syscall.LOCK_UN)
}

// Close releases the lock (if held) and closes the file descriptor.
// The lock is automatically released when the fd is closed.
func (m *FileMutex) Close() error {
	return syscall.Close(m.fd)
}

func main() {
	pid := os.Getpid()
	lockPath := "/tmp/demo.lock"

	mutex, err := NewFileMutex(lockPath)
	if err != nil {
		fmt.Fprintf(os.Stderr, "[%d] Failed to create mutex: %v\n", pid, err)
		os.Exit(1)
	}
	defer mutex.Close()

	// Perform 5 iterations of: acquire lock → critical section → release lock
	for i := 0; i < 5; i++ {
		fmt.Printf("[%d] Waiting for lock (iteration %d)...\n", pid, i+1)

		if err := mutex.Lock(); err != nil {
			fmt.Fprintf(os.Stderr, "[%d] Lock failed: %v\n", pid, err)
			continue
		}

		// === Critical Section ===
		// Only one process at a time can be here
		fmt.Printf("[%d] 🔒 LOCKED — entering critical section\n", pid)
		time.Sleep(500 * time.Millisecond) // Simulate work
		fmt.Printf("[%d] 🔓 UNLOCKING — leaving critical section\n", pid)

		if err := mutex.Unlock(); err != nil {
			fmt.Fprintf(os.Stderr, "[%d] Unlock failed: %v\n", pid, err)
		}

		// Small delay to give other processes a chance to acquire the lock
		time.Sleep(100 * time.Millisecond)
	}

	fmt.Printf("[%d] Done.\n", pid)
}
```

---

## 3.9 inotify — File System Event Monitoring

### 3.9.1 What is inotify?

inotify is a Linux kernel subsystem that provides file system event notifications.
When files or directories change, inotify delivers events to watching processes.

This is how `inotifywait`, file sync tools, and IDEs detect file changes.

```
Application
     │
     │  inotify_init1() → creates inotify instance (returns fd)
     │  inotify_add_watch(fd, "/path", IN_MODIFY|IN_CREATE) → adds watch
     │
     │  read(fd, buf) → blocks until events occur
     │       │
     │       ▼
     │  struct inotify_event {
     │      int      wd;      // Watch descriptor
     │      uint32_t mask;    // Event type (IN_MODIFY, IN_CREATE, etc.)
     │      uint32_t cookie;  // For rename: links IN_MOVED_FROM/TO events
     │      uint32_t len;     // Length of name field
     │      char     name[];  // Filename (for directory watches)
     │  };
     │
     ▼
Kernel inotify subsystem
     │
     │  Hooks into VFS layer — gets callbacks when files change
     │  Maintains per-watch queues of events
     │  Coalesces duplicate events (same file, same event)
     │
     ▼
VFS operations trigger events:
  vfs_write()   → IN_MODIFY
  vfs_create()  → IN_CREATE
  vfs_unlink()  → IN_DELETE
  vfs_rename()  → IN_MOVED_FROM + IN_MOVED_TO
  vfs_open()    → IN_OPEN
  vfs_close()   → IN_CLOSE_WRITE or IN_CLOSE_NOWRITE
```

### 3.9.2 inotify Event Types

| Event | Description |
|---|---|
| `IN_ACCESS` | File was read |
| `IN_MODIFY` | File was written |
| `IN_ATTRIB` | Metadata changed (permissions, timestamps, etc.) |
| `IN_CLOSE_WRITE` | Writable file was closed |
| `IN_CLOSE_NOWRITE` | Read-only file was closed |
| `IN_OPEN` | File was opened |
| `IN_MOVED_FROM` | File was moved out of watched directory |
| `IN_MOVED_TO` | File was moved into watched directory |
| `IN_CREATE` | File/directory created in watched directory |
| `IN_DELETE` | File/directory deleted from watched directory |
| `IN_DELETE_SELF` | Watched file/directory itself was deleted |
| `IN_MOVE_SELF` | Watched file/directory itself was moved |

### 3.9.3 Hands-On: Building a File Watcher with inotify Syscalls

```go
// inotify_watcher.go
//
// This program implements a file system watcher using raw inotify
// syscalls. It watches a directory for changes and prints events
// in real-time.
//
// inotify is the kernel API that powers tools like:
//   - fsnotify (Go library)
//   - inotifywait (command-line tool)
//   - IDE file watchers (VS Code, IntelliJ)
//   - File sync tools (rsync --watch, syncthing)
//
// Limitations of inotify:
//   - Watches are per-directory (not recursive by default)
//   - Limited watch count (/proc/sys/fs/inotify/max_user_watches)
//   - Events can be lost if the queue overflows
//   - Doesn't work across network filesystems (NFS, CIFS)
//
// Run: go build -o watcher inotify_watcher.go && ./watcher /path/to/watch
// Then make changes in another terminal and watch events appear.
package main

import (
	"bytes"
	"encoding/binary"
	"fmt"
	"os"
	"path/filepath"
	"syscall"
	"unsafe"
)

// inotifyEvent mirrors the kernel's struct inotify_event.
// Each event read from the inotify fd has this structure followed
// by a variable-length filename.
type inotifyEvent struct {
	WD     int32  // Watch descriptor (identifies which watch triggered)
	Mask   uint32 // Event type bitmask (IN_MODIFY, IN_CREATE, etc.)
	Cookie uint32 // Unique cookie for rename events (pairs FROM/TO)
	Len    uint32 // Length of the name field that follows this struct
}

// inotifyEventSize is the size of the fixed part of an inotify event.
// The actual event is this size plus the variable-length name field.
const inotifyEventSize = int(unsafe.Sizeof(inotifyEvent{}))

// maskToString converts an inotify event mask to human-readable strings.
// An event can have multiple bits set (e.g., IN_MODIFY|IN_ISDIR).
func maskToString(mask uint32) string {
	var events []string

	eventNames := map[uint32]string{
		syscall.IN_ACCESS:        "ACCESS",
		syscall.IN_MODIFY:        "MODIFY",
		syscall.IN_ATTRIB:        "ATTRIB",
		syscall.IN_CLOSE_WRITE:   "CLOSE_WRITE",
		syscall.IN_CLOSE_NOWRITE: "CLOSE_NOWRITE",
		syscall.IN_OPEN:          "OPEN",
		syscall.IN_MOVED_FROM:    "MOVED_FROM",
		syscall.IN_MOVED_TO:      "MOVED_TO",
		syscall.IN_CREATE:        "CREATE",
		syscall.IN_DELETE:        "DELETE",
		syscall.IN_DELETE_SELF:   "DELETE_SELF",
		syscall.IN_MOVE_SELF:     "MOVE_SELF",
	}

	for bit, name := range eventNames {
		if mask&bit != 0 {
			events = append(events, name)
		}
	}

	if mask&syscall.IN_ISDIR != 0 {
		events = append(events, "ISDIR")
	}

	if len(events) == 0 {
		return fmt.Sprintf("UNKNOWN(0x%x)", mask)
	}

	result := events[0]
	for _, e := range events[1:] {
		result += "|" + e
	}
	return result
}

// Watcher wraps an inotify file descriptor and provides a higher-level
// API for watching file system events.
type Watcher struct {
	fd      int            // inotify file descriptor
	watches map[int32]string // wd → path mapping for human-readable output
}

// NewWatcher creates a new inotify instance.
// The returned file descriptor is used for all subsequent operations.
//
// inotify_init1(IN_CLOEXEC) is preferred over inotify_init() because
// it sets the close-on-exec flag, preventing fd leaks to child processes.
func NewWatcher() (*Watcher, error) {
	// Create the inotify instance. This returns a file descriptor
	// that we'll read events from.
	fd, err := syscall.InotifyInit1(syscall.IN_CLOEXEC)
	if err != nil {
		return nil, fmt.Errorf("inotify_init1: %w", err)
	}

	return &Watcher{
		fd:      fd,
		watches: make(map[int32]string),
	}, nil
}

// AddWatch adds a watch on the specified path for the given events.
// Returns a watch descriptor that identifies this watch in future events.
//
// The mask specifies which events to watch for. Common combinations:
//   IN_ALL_EVENTS: everything
//   IN_MODIFY | IN_CREATE | IN_DELETE: file content changes
//   IN_CLOSE_WRITE: file write completed (good for triggering rebuilds)
func (w *Watcher) AddWatch(path string, mask uint32) (int32, error) {
	wd, err := syscall.InotifyAddWatch(w.fd, path, mask)
	if err != nil {
		return -1, fmt.Errorf("inotify_add_watch(%s): %w", path, err)
	}

	w.watches[int32(wd)] = path
	return int32(wd), nil
}

// AddRecursiveWatch adds watches on a directory and all its subdirectories.
// This is needed because inotify watches are NOT recursive by default —
// a watch on /foo does NOT watch /foo/bar/baz.
func (w *Watcher) AddRecursiveWatch(root string, mask uint32) error {
	return filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
			if err != nil {
				return nil // Skip inaccessible directories
			}
			if info.IsDir() {
				_, err := w.AddWatch(path, mask)
				if err != nil {
					fmt.Fprintf(os.Stderr, "Warning: cannot watch %s: %v\n", path, err)
					return nil
				}
			}
			return nil
		})
}

// ReadEvents reads and processes inotify events in a loop.
// This blocks until events are available, then processes them.
//
// The inotify fd is a special file — reading from it returns a stream
// of inotify_event structs. Multiple events can be returned in a single
// read() call for efficiency.
func (w *Watcher) ReadEvents() error {
	// Buffer for reading events. 4096 bytes is enough for many events.
	// Each event is at least 16 bytes + filename length.
	buf := make([]byte, 4096)

	for {
		// read() on the inotify fd blocks until events are available.
		// Returns one or more inotify_event structs concatenated together.
		n, err := syscall.Read(w.fd, buf)
		if err != nil {
			return fmt.Errorf("read inotify: %w", err)
		}

		if n < inotifyEventSize {
			continue // Shouldn't happen, but be defensive
		}

		// Parse the events from the buffer
		offset := 0
		for offset < n {
			// Parse the fixed-size event header
			var event inotifyEvent
			eventBytes := buf[offset : offset+inotifyEventSize]
			binary.Read(bytes.NewReader(eventBytes), binary.LittleEndian, &event)

			// Extract the filename (if present — only for directory watches)
			var name string
			if event.Len > 0 {
				nameBytes := buf[offset+inotifyEventSize : offset+inotifyEventSize+int(event.Len)]
				// Name is null-terminated — find the first null byte
				for i, b := range nameBytes {
					if b == 0 {
						name = string(nameBytes[:i])
						break
					}
				}
			}

			// Build the full path from the watch descriptor + event name
			dir := w.watches[event.WD]
			fullPath := dir
			if name != "" {
				fullPath = filepath.Join(dir, name)
			}

			// Print the event
			fmt.Printf("%-50s %s\n", fullPath, maskToString(event.Mask))

			// If a new directory was created, add a watch for it too
			// (for recursive watching)
			if event.Mask&syscall.IN_CREATE != 0 && event.Mask&syscall.IN_ISDIR != 0 {
				w.AddWatch(fullPath, syscall.IN_ALL_EVENTS)
				fmt.Printf("  → Added recursive watch on new directory: %s\n", fullPath)
			}

			// Advance to the next event in the buffer
			offset += inotifyEventSize + int(event.Len)
		}
	}
}

// Close cleans up the inotify instance.
// All watches are automatically removed when the fd is closed.
func (w *Watcher) Close() error {
	return syscall.Close(w.fd)
}

func main() {
	// Default to watching the current directory
	watchPath := "."
	if len(os.Args) > 1 {
		watchPath = os.Args[1]
	}

	watcher, err := NewWatcher()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create watcher: %v\n", err)
		os.Exit(1)
	}
	defer watcher.Close()

	// Watch for all events, recursively
	fmt.Printf("Setting up recursive watches on %s...\n", watchPath)
	if err := watcher.AddRecursiveWatch(watchPath, syscall.IN_ALL_EVENTS); err != nil {
		fmt.Fprintf(os.Stderr, "Failed to add watches: %v\n", err)
		os.Exit(1)
	}

	fmt.Printf("Watching for events (Ctrl+C to stop)...\n\n")
	fmt.Printf("%-50s %s\n", "PATH", "EVENTS")
	fmt.Println("──────────────────────────────────────────────────────────────────────")

	// This blocks forever, printing events as they occur
	if err := watcher.ReadEvents(); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
}
```

### 3.9.4 Comparison: Raw inotify vs fsnotify Library

```go
// fsnotify_example.go
//
// The same file watcher using the fsnotify library, which provides:
//   - Cross-platform support (Linux inotify, macOS kqueue, Windows ReadDirectoryChanges)
//   - Cleaner API with Go channels
//   - Automatic event deduplication
//
// The tradeoff: you lose fine-grained control over inotify flags and
// can't do things like rename cookie tracking.
//
// Install: go get github.com/fsnotify/fsnotify
package main

import (
	"fmt"
	"log"
	"os"

	"github.com/fsnotify/fsnotify"
)

func main() {
	// Create a watcher — internally calls inotify_init1() on Linux
	watcher, err := fsnotify.NewWatcher()
	if err != nil {
		log.Fatal(err)
	}
	defer watcher.Close()

	// Start a goroutine to process events from the channel
	go func() {
		for {
			select {
			case event, ok := <-watcher.Events:
				if !ok {
					return
				}
				// event.Op is a bitmask: Create, Write, Remove, Rename, Chmod
				fmt.Printf("%-50s %s\n", event.Name, event.Op)

			case err, ok := <-watcher.Errors:
				if !ok {
					return
				}
				fmt.Fprintf(os.Stderr, "Error: %v\n", err)
			}
		}
	}()

	// Add a watch — internally calls inotify_add_watch()
	watchPath := "."
	if len(os.Args) > 1 {
		watchPath = os.Args[1]
	}

	err = watcher.Add(watchPath)
	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("Watching %s for changes...\n", watchPath)

	// Block forever
	select {}
}
```

---

## 3.10 Disk I/O Scheduling

### 3.10.1 Why I/O Scheduling Matters

When multiple processes submit I/O requests to a disk, the kernel's I/O scheduler
decides the **order** in which requests are sent to the hardware. This is critical
for spinning disks (HDDs) where seek time dominates, but also matters for SSDs.

```
Process A: read sector 1000
Process B: read sector 5000000
Process C: read sector 1001
Process D: read sector 4999999

Without scheduling (FIFO):
  Head moves: 1000 → 5000000 → 1001 → 4999999
  Seeks: 4,999,000 + 4,998,999 + 4,998,998 = terrible!

With elevator scheduling (sorted):
  Head moves: 1000 → 1001 → 4999999 → 5000000
  Seeks: 1 + 4,998,998 + 1 = much better!
```

### 3.10.2 Linux I/O Schedulers

| Scheduler | Type | Best For | Description |
|---|---|---|---|
| **none/noop** | Pass-through | NVMe SSDs | No reordering. SSDs have no seek penalty. |
| **mq-deadline** | Deadline | SSDs, general | Batches requests by direction, ensures no request starves beyond a deadline. Default for many distros. |
| **BFQ** | Fair queuing | Desktop, mixed workloads | Budget Fair Queueing — gives each process a fair share of I/O bandwidth. Good for interactive responsiveness. |
| **kyber** | Token-based | Fast SSDs | Lightweight, low-overhead. Targets latency by throttling dispatch. |
| **CFQ** | Fair queuing | DEPRECATED (HDD era) | Was the default for years. Removed in Linux 5.0. |

```bash
# Check the current scheduler for a disk:
$ cat /sys/block/sda/queue/scheduler
[mq-deadline] kyber bfq none

# The bracketed one is active. Change it:
$ echo bfq > /sys/block/sda/queue/scheduler

# For NVMe drives, 'none' is typical and optimal:
$ cat /sys/block/nvme0n1/queue/scheduler
[none] mq-deadline kyber bfq
```

### 3.10.3 ionice — I/O Priority

Linux supports three I/O scheduling classes:

| Class | Description |
|---|---|
| **Real-time (1)** | Highest priority. Gets first access to disk. Can starve other processes. |
| **Best-effort (2)** | Default class. Priority levels 0-7 (0 = highest). |
| **Idle (3)** | Only gets I/O time when no other process needs the disk. |

```bash
# Run a backup at idle I/O priority (won't impact other processes):
$ ionice -c 3 rsync -a /data /backup

# Run a database at high best-effort priority:
$ ionice -c 2 -n 0 postgres

# Check a process's I/O priority:
$ ionice -p $(pidof postgres)
```

### 3.10.4 Hands-On: Observing I/O with Go and /proc

```go
// io_stats.go
//
// This program reads I/O statistics from /proc and /sys to demonstrate
// how to monitor disk I/O behavior from Go. This is similar to what
// iostat, iotop, and other monitoring tools do.
//
// Key files:
//   /proc/diskstats     — per-disk I/O counters
//   /proc/self/io       — per-process I/O counters
//   /sys/block/*/stat   — per-device statistics
//   /sys/block/*/queue/ — scheduler configuration
//
// Run: go run io_stats.go
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"strings"
	"time"
)

// diskStats holds parsed statistics from /proc/diskstats.
// These counters are cumulative since boot.
type diskStats struct {
	Device      string
	ReadsCompleted  uint64
	ReadsMerged     uint64
	SectorsRead     uint64
	ReadTimeMs      uint64
	WritesCompleted uint64
	WritesMerged    uint64
	SectorsWritten  uint64
	WriteTimeMs     uint64
	IOsInProgress   uint64
	IOTimeMs        uint64
	WeightedIOMs    uint64
}

// parseDiskStats reads /proc/diskstats and returns parsed statistics
// for all block devices. Each line has 14+ fields:
//   major minor name reads_completed reads_merged sectors_read ms_reading
//   writes_completed writes_merged sectors_written ms_writing
//   ios_in_progress ms_io weighted_ms_io
func parseDiskStats() ([]diskStats, error) {
	data, err := os.ReadFile("/proc/diskstats")
	if err != nil {
		return nil, err
	}

	var stats []diskStats
	for _, line := range strings.Split(string(data), "\n") {
		fields := strings.Fields(line)
		if len(fields) < 14 {
			continue
		}

		var s diskStats
		s.Device = fields[2]
		fmt.Sscanf(fields[3], "%d", &s.ReadsCompleted)
		fmt.Sscanf(fields[4], "%d", &s.ReadsMerged)
		fmt.Sscanf(fields[5], "%d", &s.SectorsRead)
		fmt.Sscanf(fields[6], "%d", &s.ReadTimeMs)
		fmt.Sscanf(fields[7], "%d", &s.WritesCompleted)
		fmt.Sscanf(fields[8], "%d", &s.WritesMerged)
		fmt.Sscanf(fields[9], "%d", &s.SectorsWritten)
		fmt.Sscanf(fields[10], "%d", &s.WriteTimeMs)
		fmt.Sscanf(fields[11], "%d", &s.IOsInProgress)
		fmt.Sscanf(fields[12], "%d", &s.IOTimeMs)
		fmt.Sscanf(fields[13], "%d", &s.WeightedIOMs)

		// Filter to actual devices (skip partitions for cleaner output)
		if s.ReadsCompleted > 0 || s.WritesCompleted > 0 {
			stats = append(stats, s)
		}
	}
	return stats, nil
}

// getProcessIO reads the current process's I/O counters from /proc/self/io.
// This file shows how much I/O the current process has performed.
func getProcessIO() (map[string]uint64, error) {
	data, err := os.ReadFile("/proc/self/io")
	if err != nil {
		return nil, err
	}

	result := make(map[string]uint64)
	for _, line := range strings.Split(string(data), "\n") {
		parts := strings.SplitN(line, ": ", 2)
		if len(parts) == 2 {
			var val uint64
			fmt.Sscanf(strings.TrimSpace(parts[1]), "%d", &val)
			result[parts[0]] = val
		}
	}
	return result, nil
}

// getSchedulerInfo reads the I/O scheduler configuration for a device.
func getSchedulerInfo(device string) string {
	// Strip partition numbers to get the base device
	baseDev := device
	for _, suffix := range []string{"1", "2", "3", "4", "5"} {
		baseDev = strings.TrimSuffix(baseDev, suffix)
	}

	schedPath := filepath.Join("/sys/block", baseDev, "queue/scheduler")
	data, err := os.ReadFile(schedPath)
	if err != nil {
		return "unknown"
	}
	return strings.TrimSpace(string(data))
}

func main() {
	fmt.Println("=== Disk I/O Statistics ===\n")

	// --- System-wide disk stats ---
	fmt.Println("--- Per-Device Statistics (from /proc/diskstats) ---")
	stats, err := parseDiskStats()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error reading diskstats: %v\n", err)
	} else {
		fmt.Printf("%-10s %10s %10s %12s %12s %10s\n",
			"Device", "Reads", "Writes", "Read MB", "Write MB", "IO ms")
		fmt.Println(strings.Repeat("-", 70))
		for _, s := range stats {
			readMB := float64(s.SectorsRead) * 512 / (1024 * 1024)
			writeMB := float64(s.SectorsWritten) * 512 / (1024 * 1024)
			fmt.Printf("%-10s %10d %10d %10.1f MB %10.1f MB %10d\n",
				s.Device, s.ReadsCompleted, s.WritesCompleted,
				readMB, writeMB, s.IOTimeMs)
		}
	}

	// --- Scheduler info ---
	fmt.Println("\n--- I/O Scheduler Configuration ---")
	blockDevs, _ := filepath.Glob("/sys/block/*/queue/scheduler")
	for _, path := range blockDevs {
		dev := strings.Split(path, "/")[3]
		data, _ := os.ReadFile(path)
		fmt.Printf("%-15s scheduler: %s", dev, string(data))
	}

	// --- Per-process I/O ---
	fmt.Println("\n--- Current Process I/O (from /proc/self/io) ---")
	ioStats, err := getProcessIO()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error reading process IO: %v (need root?)\n", err)
	} else {
		descriptions := map[string]string{
			"rchar":                  "Bytes read (including page cache)",
			"wchar":                  "Bytes written (including page cache)",
			"syscr":                  "Read syscalls count",
			"syscw":                  "Write syscalls count",
			"read_bytes":             "Bytes actually read from disk",
			"write_bytes":            "Bytes actually written to disk",
			"cancelled_write_bytes":  "Bytes cancelled (truncated files)",
		}
		for key, val := range ioStats {
			desc := descriptions[key]
			if desc == "" {
				desc = key
			}
			fmt.Printf("  %-30s %12d  (%s)\n", key+":", val, desc)
		}
	}

	// --- Live monitoring demo ---
	fmt.Println("\n--- Live I/O Rate (5-second sample) ---")
	before, _ := parseDiskStats()
	time.Sleep(5 * time.Second)
	after, _ := parseDiskStats()

	if before != nil && after != nil {
		// Build a map for quick lookup
		beforeMap := make(map[string]diskStats)
		for _, s := range before {
			beforeMap[s.Device] = s
		}

		fmt.Printf("%-10s %10s %10s %10s %10s\n",
			"Device", "Read/s", "Write/s", "Read MB/s", "Write MB/s")
		fmt.Println(strings.Repeat("-", 55))

		for _, s := range after {
			b, ok := beforeMap[s.Device]
			if !ok {
				continue
			}
			readOps := float64(s.ReadsCompleted-b.ReadsCompleted) / 5.0
			writeOps := float64(s.WritesCompleted-b.WritesCompleted) / 5.0
			readMBs := float64(s.SectorsRead-b.SectorsRead) * 512 / (1024 * 1024 * 5)
			writeMBs := float64(s.SectorsWritten-b.SectorsWritten) * 512 / (1024 * 1024 * 5)

			if readOps > 0 || writeOps > 0 {
				fmt.Printf("%-10s %10.1f %10.1f %8.2f MB %8.2f MB\n",
					s.Device, readOps, writeOps, readMBs, writeMBs)
			}
		}
	}
}
```

---

## 3.11 Summary

This chapter covered the complete Linux file system and I/O stack:

| Topic | Key Takeaway |
|---|---|
| **VFS** | Abstracts all filesystems behind one interface (superblock, inode, dentry, file) |
| **File Descriptors** | Integers indexing a three-level table: fd table → open file table → inode table |
| **File I/O** | write() goes to page cache, not disk. fsync() forces disk write. pread/pwrite are atomic. |
| **epoll** | O(ready) event notification using callback-driven ready list. Go's net uses this. |
| **io_uring** | Shared-memory ring buffers for zero-syscall I/O. The future of Linux I/O. |
| **Zero-copy** | sendfile/splice eliminate user-space copies. Used by Go's http.ServeFile. |
| **Filesystems** | ext4 (journaling+extents), XFS (parallel AGs), tmpfs (RAM), overlayfs (containers) |
| **File Locking** | flock() for whole-file, fcntl() for byte-range. Advisory by default. |
| **inotify** | Kernel event notification for file changes. Powers fsnotify library. |
| **I/O Scheduling** | mq-deadline (general), BFQ (desktop), none (NVMe). ionice for priorities. |

### What's Next

In Chapter 4, we'll dive into **Networking & Sockets** — the TCP/IP stack, raw
sockets, socket options, and building network tools from syscalls. You'll see
how the I/O multiplexing techniques from this chapter apply to real network
programming.

---

## 3.12 Exercises

1. **Inode Explorer**: Write a Go program that takes a file path and prints
   everything that `stat(2)` reveals, including decoding the file type from
   the mode bits. Handle all 7 file types (regular, directory, symlink,
   socket, FIFO, block device, character device).

2. **fd Leak Detector**: Write a program that opens 100 files, "forgets" to
   close some, then examines `/proc/self/fd` to detect the leaks. Add
   `O_CLOEXEC` and verify it works after fork+exec.

3. **Page Cache Observer**: Write a Go program that reads a large file twice.
   First read should be slow (cold cache), second read should be fast (warm
   cache). Use `/proc/meminfo` to observe the page cache growing after the
   first read.

4. **epoll Chat Server**: Extend the epoll echo server into a multi-user chat
   server. When any client sends a message, broadcast it to all other connected
   clients. Track clients in a map[fd]*Client structure.

5. **Zero-Copy Proxy**: Build a TCP proxy that uses `splice()` to forward data
   between two sockets without copying through user space. Benchmark it against
   a naive `Read()`/`Write()` proxy.

6. **Container Layer Simulator**: Build a program that creates a multi-layer
   overlayfs (3+ layers) simulating a Docker image with base layer, package
   layer, and application layer. Demonstrate copy-on-write behavior at each level.

7. **File Lock Benchmark**: Compare the performance of `flock()` vs `fcntl()`
   locks under contention from multiple processes. How do they behave when
   a lock holder crashes?

8. **Build System Watcher**: Use inotify to watch a source directory. When any
   `.go` file changes, automatically run `go build` and report success/failure.
   Handle the common inotify pitfalls (duplicate events, editor temp files).

9. **I/O Scheduler Benchmark**: Write a Go program that creates random I/O
   patterns (random reads across a large file) and sequential I/O patterns.
   Run it under different I/O schedulers (`none`, `mq-deadline`, `bfq`) and
   compare throughput and latency using `/proc/diskstats`.

10. **io_uring File Copier**: Implement a file copy program using io_uring
    that submits batches of read requests, waits for completions, then submits
    batches of write requests. Compare performance with a simple `io.Copy()`
    implementation.
