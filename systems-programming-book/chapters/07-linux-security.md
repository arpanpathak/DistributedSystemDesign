# Chapter 7: Linux Security — Capabilities, Seccomp, AppArmor, SELinux

> *"Security is not a product, but a process."* — Bruce Schneier

Linux is the operating system that underpins nearly every container, every Kubernetes
cluster, and every cloud VM you will encounter. Understanding how Linux enforces
security — from the classic Unix permission bits all the way to modern mandatory
access controls — is not optional knowledge for a systems programmer. It is the
foundation upon which every later chapter in this book rests.

This chapter takes you on a deep dive through every layer of the Linux security
stack. We begin with the traditional Discretionary Access Control (DAC) model —
file permissions, UIDs, GIDs, and the dangers of root. We then move into the modern
world: Linux Capabilities (splitting root into granular privileges), Seccomp (syscall
filtering with BPF), AppArmor (path-based mandatory access control), and SELinux
(label-based mandatory access control). Along the way, we write Go programs that
interact with each of these subsystems through the `x/sys/unix` package.

By the end of this chapter, you will understand:

- Why root is dangerous, and how capabilities replace it
- How to restrict which system calls a process can make
- How AppArmor and SELinux enforce mandatory policies that even root cannot bypass
- How to layer these mechanisms for defense in depth
- What Docker and Kubernetes apply by default — and why

```text
┌─────────────────────────────────────────────────────────────────────┐
│                    LINUX SECURITY ARCHITECTURE                       │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                   User Space Process                          │   │
│  │   ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌─────────────┐  │   │
│  │   │   DAC   │  │  Caps   │  │ Seccomp  │  │ AppArmor/   │  │   │
│  │   │ (perms) │  │ (privs) │  │  (BPF)   │  │  SELinux    │  │   │
│  │   └────┬────┘  └────┬────┘  └────┬─────┘  └──────┬──────┘  │   │
│  └────────┼─────────────┼───────────┼────────────────┼─────────┘   │
│           │             │           │                │              │
│  ┌────────▼─────────────▼───────────▼────────────────▼─────────┐   │
│  │                     KERNEL SECURITY HOOKS                     │   │
│  │                                                               │   │
│  │  1. DAC check (file permissions, UID/GID)                    │   │
│  │  2. Capability check (does process have CAP_xxx?)            │   │
│  │  3. LSM hooks (AppArmor or SELinux policy check)             │   │
│  │  4. Seccomp-BPF filter (is this syscall allowed?)            │   │
│  │                                                               │   │
│  └───────────────────────────┬───────────────────────────────────┘   │
│                              │                                       │
│  ┌───────────────────────────▼───────────────────────────────────┐   │
│  │                    ACTUAL KERNEL OPERATION                     │   │
│  │              (open file, bind port, mount fs, etc.)           │   │
│  └───────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

**Every syscall passes through ALL of these checks.** A process must satisfy DAC
permissions, possess the required capability, pass any LSM policy, AND survive
seccomp filtering — failing any one layer blocks the operation.

---

## 7.1 The Linux Security Model

### 7.1.1 Discretionary Access Control (DAC) — Traditional Unix Permissions

The original Unix security model is called **Discretionary Access Control** (DAC)
because the *owner* of a resource has discretion over who can access it. This is
the permission model you interact with every time you run `ls -l`:

```text
-rwxr-xr-- 1 alice developers 8192 Jan 15 10:30 myapp
│├─┤├─┤├─┤   │     │
│ │  │  │    │     └── Group owner
│ │  │  │    └── File owner
│ │  │  └── Other permissions (r--)
│ │  └── Group permissions (r-x)
│ └── Owner permissions (rwx)
└── File type (- = regular file)
```

**The three permission classes:**

| Class | Meaning | Checked when... |
|-------|---------|----------------|
| **Owner (u)** | The user who owns the file | Process UID == file UID |
| **Group (g)** | Members of the file's group | Process GID == file GID (or supplementary groups) |
| **Other (o)** | Everyone else | Neither owner nor group member |

**The three basic permission bits:**

| Bit | On files | On directories |
|-----|----------|----------------|
| **r** (read, 4) | Read file contents | List directory contents |
| **w** (write, 2) | Modify file contents | Create/delete files in directory |
| **x** (execute, 1) | Execute as program | Enter (cd into) directory |

**How the kernel checks permissions (simplified):**

```text
┌─────────────────────────────────────────────┐
│           Process wants to open(file, O_RDWR) │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  Is process UID  │──── YES ──→ Skip all DAC checks
              │    == 0 (root)? │              (root bypasses DAC)
              └────────┬────────┘
                       │ NO
                       ▼
              ┌─────────────────┐
              │ Process UID ==  │──── YES ──→ Check owner bits (rw-)
              │   file UID?     │
              └────────┬────────┘
                       │ NO
                       ▼
              ┌─────────────────┐
              │ Process GID ==  │──── YES ──→ Check group bits (rw-)
              │   file GID?     │
              └────────┬────────┘
                       │ NO
                       ▼
              Check "other" bits (rw-)
```

> **Critical insight:** Notice that root (UID 0) bypasses DAC entirely. This is
> the fundamental problem that capabilities were designed to solve.


### 7.1.2 Special Permission Bits: setuid, setgid, and Sticky Bit

Beyond the basic rwx bits, Unix has three special permission bits that modify
execution behavior:

**setuid (Set User ID on execution) — mode 4xxx:**

When a setuid binary is executed, the process runs with the **file owner's UID**
rather than the caller's UID. The classic example is `/usr/bin/passwd`:

```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 68208 Jan 15 10:30 /usr/bin/passwd
#  ^--- 's' in owner execute position = setuid
```

When any user runs `passwd`, the process runs as root (UID 0), allowing it to
modify `/etc/shadow`. This is necessary because only root can write to the shadow
password file.

**setgid (Set Group ID on execution) — mode 2xxx:**

Similar to setuid, but for the group. When set on a directory, new files created
within inherit the directory's group (rather than the creator's primary group).

**Sticky bit — mode 1xxx:**

On directories, the sticky bit prevents users from deleting files they don't own.
`/tmp` is the classic example:

```bash
$ ls -ld /tmp
drwxrwxrwt 15 root root 4096 Jan 15 10:30 /tmp
#        ^--- 't' in other execute position = sticky bit
```

Without the sticky bit, any user with write permission to `/tmp` could delete any
other user's files.


### 7.1.3 The Root Problem — Why Running as Root Is Dangerous

The root user (UID 0) is the superuser in Unix/Linux. Root can:

- Read, write, and execute any file regardless of permissions
- Bind to privileged ports (below 1024)
- Load and unload kernel modules
- Mount and unmount filesystems
- Change any process's UID/GID
- Bypass all DAC checks
- Send signals to any process
- Configure network interfaces
- Access raw sockets
- Trace any process with ptrace
- Reboot the system

**This is an all-or-nothing model.** If a service needs to bind to port 80, it
traditionally had to run as root — even though it needed only *one* of root's
hundreds of privileges. If that service is compromised, the attacker gets ALL of
root's powers.

```
┌──────────────────────────────────────────────────┐
│              THE ROOT PROBLEM                      │
│                                                    │
│   What your web server needs:                      │
│     ✓ Bind to port 80                             │
│                                                    │
│   What root gives your web server:                 │
│     ✓ Bind to port 80                             │
│     ✓ Read /etc/shadow                            │
│     ✓ Load kernel modules                         │
│     ✓ Mount filesystems                           │
│     ✓ Kill any process                            │
│     ✓ Change any file's ownership                 │
│     ✓ Modify network routing                      │
│     ✓ Access raw devices                          │
│     ✓ ... hundreds more privileges                │
│                                                    │
│   This is why we need CAPABILITIES.               │
└──────────────────────────────────────────────────┘
```

### 7.1.4 Mandatory Access Control (MAC) — SELinux and AppArmor

DAC's fundamental weakness is that **the user controls the policy.** If a process
runs as root, it can change any file's permissions, ownership, or ACLs. There is
no way to constrain root under pure DAC.

**Mandatory Access Control (MAC)** solves this. Under MAC:

- A **central administrator** defines the security policy
- The policy is enforced by the kernel
- Even root cannot override the policy (without special administrative access)
- Every access decision is checked against the policy

Linux implements MAC through the **Linux Security Module (LSM)** framework.
The two most widely deployed LSMs are:

| Feature | AppArmor | SELinux |
|---------|----------|---------|
| **Model** | Path-based | Label-based |
| **Complexity** | Lower | Higher |
| **Default on** | Ubuntu, SUSE | RHEL, Fedora, CentOS |
| **Policy scope** | Per-program profiles | System-wide type enforcement |
| **Learning mode** | `complain` mode | `permissive` mode |
| **Container support** | Docker, K8s | Docker, K8s |

We cover both in depth later in this chapter.


### 7.1.5 Hands-On: Understanding File Permissions from Go Using syscall.Stat_t

Let's write our first Go program that inspects the full Unix security metadata of
a file — owner, group, permissions, and special bits:

```go
// file_permissions.go
//
// This program demonstrates how to inspect Unix file permissions,
// ownership, and special bits (setuid, setgid, sticky) from Go.
// It uses the syscall.Stat_t structure to access raw stat(2) data,
// giving us access to UID, GID, and the full mode bitmask — information
// that Go's os.FileInfo intentionally abstracts away.
//
// Understanding this is essential because Linux security decisions are
// made based on these exact fields: the kernel compares the process's
// UID/GID against the file's UID/GID and checks the permission bits
// on every file access.
//
// Usage:
//   go run file_permissions.go /etc/passwd /usr/bin/passwd /tmp
package main

import (
	"fmt"
	"os"
	"os/user"
	"strconv"
	"syscall"
)

// fileSecurityInfo holds the parsed security metadata for a file.
// We extract this from syscall.Stat_t, which maps directly to the
// kernel's struct stat returned by the stat(2) system call.
type fileSecurityInfo struct {
	Path       string      // Filesystem path of the file
	Mode       os.FileMode // Go's file mode (includes type + permissions)
	RawMode    uint32      // Raw mode bits from stat(2) — includes setuid/setgid/sticky
	UID        uint32      // Numeric user ID of the file owner
	GID        uint32      // Numeric group ID of the file's group
	UserName   string      // Resolved username (or "unknown" if lookup fails)
	GroupName  string      // Resolved group name (or "unknown" if lookup fails)
	IsSetuid   bool        // Whether the setuid bit (04000) is set
	IsSetgid   bool        // Whether the setgid bit (02000) is set
	IsSticky   bool        // Whether the sticky bit (01000) is set
	Size       int64       // File size in bytes
	LinkCount  uint64      // Number of hard links (nlink from stat)
}

// inspectFile performs a stat(2) on the given path and extracts all
// security-relevant metadata. This demonstrates the critical difference
// between Go's os.Stat (which returns an os.FileInfo with abstracted
	// permissions) and the raw syscall.Stat_t which gives us the actual
// UID, GID, and raw mode bits that the kernel uses for access decisions.
//
// The key insight: os.FileInfo.Mode() gives you permission bits, but
// Go intentionally does NOT expose UID/GID through the os package.
// You MUST use syscall.Stat_t or x/sys/unix.Stat_t for that.
func inspectFile(path string) (*fileSecurityInfo, error) {
	// os.Lstat does NOT follow symlinks — we want metadata about the
	// link itself, not its target. For security analysis, this matters
	// because a symlink and its target can have different owners.
	fi, err := os.Lstat(path)
	if err != nil {
		return nil, fmt.Errorf("lstat %s: %w", path, err)
	}

	// Extract the underlying syscall.Stat_t from the os.FileInfo.
	// Go's os package stores the raw stat result in the Sys() field.
	// This type assertion is platform-specific — it works on Linux
	// and other Unix-like systems but NOT on Windows.
	stat, ok := fi.Sys().(*syscall.Stat_t)
	if !ok {
		return nil, fmt.Errorf("not a unix system — cannot extract Stat_t")
	}

	info := &fileSecurityInfo{
		Path:      path,
		Mode:      fi.Mode(),
		RawMode:   stat.Mode,
		UID:       stat.Uid,
		GID:       stat.Gid,
		Size:      stat.Size,
		LinkCount: stat.Nlink,
	}

	// Check special permission bits using bitmasks on the raw mode.
	// These constants come from <sys/stat.h>:
	//   S_ISUID = 04000 (setuid)
	//   S_ISGID = 02000 (setgid)
	//   S_ISVTX = 01000 (sticky bit)
	info.IsSetuid = stat.Mode&syscall.S_ISUID != 0
	info.IsSetgid = stat.Mode&syscall.S_ISGID != 0
	info.IsSticky = stat.Mode&syscall.S_ISVTX != 0

	// Resolve UID to username. This calls getpwuid(3) under the hood.
	// In containers, /etc/passwd may not have the user, so we handle
	// the error gracefully.
	if u, err := user.LookupId(strconv.Itoa(int(stat.Uid))); err == nil {
		info.UserName = u.Username
	} else {
		info.UserName = "unknown"
	}

	// Resolve GID to group name. This calls getgrgid(3) under the hood.
	if g, err := user.LookupGroupId(strconv.Itoa(int(stat.Gid))); err == nil {
		info.GroupName = g.Name
	} else {
		info.GroupName = "unknown"
	}

	return info, nil
}

// formatPermissions produces a human-readable permission string like "rwsr-xr-t"
// that matches the output of ls -l, including setuid, setgid, and sticky bits.
// This is more complex than it appears because the special bits overlay the
// execute bits in the display format.
func formatPermissions(mode uint32) string {
	// Extract the 12 permission bits (3 special + 9 standard)
	perms := make([]byte, 10)

	// File type character
	switch {
	case mode&syscall.S_IFMT == syscall.S_IFDIR:
		perms[0] = 'd'
	case mode&syscall.S_IFMT == syscall.S_IFLNK:
		perms[0] = 'l'
	case mode&syscall.S_IFMT == syscall.S_IFCHR:
		perms[0] = 'c'
	case mode&syscall.S_IFMT == syscall.S_IFBLK:
		perms[0] = 'b'
	case mode&syscall.S_IFMT == syscall.S_IFIFO:
		perms[0] = 'p'
	case mode&syscall.S_IFMT == syscall.S_IFSOCK:
		perms[0] = 's'
	default:
		perms[0] = '-'
	}

	// Owner permissions
	perms[1] = bit(mode, 0400, 'r')
	perms[2] = bit(mode, 0200, 'w')
	if mode&syscall.S_ISUID != 0 {
		if mode&0100 != 0 {
			perms[3] = 's' // setuid + execute
		} else {
			perms[3] = 'S' // setuid without execute (unusual)
		}
	} else {
		perms[3] = bit(mode, 0100, 'x')
	}

	// Group permissions
	perms[4] = bit(mode, 040, 'r')
	perms[5] = bit(mode, 020, 'w')
	if mode&syscall.S_ISGID != 0 {
		if mode&010 != 0 {
			perms[6] = 's' // setgid + execute
		} else {
			perms[6] = 'S' // setgid without execute
		}
	} else {
		perms[6] = bit(mode, 010, 'x')
	}

	// Other permissions
	perms[7] = bit(mode, 04, 'r')
	perms[8] = bit(mode, 02, 'w')
	if mode&syscall.S_ISVTX != 0 {
		if mode&01 != 0 {
			perms[9] = 't' // sticky + execute
		} else {
			perms[9] = 'T' // sticky without execute
		}
	} else {
		perms[9] = bit(mode, 01, 'x')
	}

	return string(perms)
}

// bit is a helper that returns the character c if the given bit is set
// in mode, or '-' if it is not. This mirrors how ls formats permission strings.
func bit(mode uint32, mask uint32, c byte) byte {
	if mode&mask != 0 {
		return c
	}
	return '-'
}

func main() {
	// Default paths to inspect if none provided on the command line.
	// These are chosen to demonstrate different security scenarios:
	//   /etc/passwd   — world-readable, root-owned (DAC example)
	//   /usr/bin/passwd — setuid root binary (special bit example)
	//   /tmp          — sticky bit directory (sticky bit example)
	paths := []string{"/etc/passwd", "/usr/bin/passwd", "/tmp"}
	if len(os.Args) > 1 {
		paths = os.Args[1:]
	}

	for _, path := range paths {
		info, err := inspectFile(path)
		if err != nil {
			fmt.Fprintf(os.Stderr, "Error: %v\n", err)
			continue
		}

		fmt.Printf("━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n")
		fmt.Printf("Path:        %s\n", info.Path)
		fmt.Printf("Permissions: %s (octal: %04o)\n",
			formatPermissions(info.RawMode), info.RawMode&07777)
		fmt.Printf("Owner:       %s (UID: %d)\n", info.UserName, info.UID)
		fmt.Printf("Group:       %s (GID: %d)\n", info.GroupName, info.GID)
		fmt.Printf("Size:        %d bytes\n", info.Size)
		fmt.Printf("Hard links:  %d\n", info.LinkCount)
		fmt.Printf("Setuid:      %v\n", info.IsSetuid)
		fmt.Printf("Setgid:      %v\n", info.IsSetgid)
		fmt.Printf("Sticky:      %v\n", info.IsSticky)

		// Security analysis — highlight potential concerns
		if info.IsSetuid {
			fmt.Printf("⚠  WARNING: setuid bit is set — this binary runs as %s (UID %d)\n",
				info.UserName, info.UID)
			if info.UID == 0 {
				fmt.Printf("⚠  CRITICAL: setuid-root binary — compromise gives root access\n")
			}
		}
		if info.RawMode&0002 != 0 {
			fmt.Printf("⚠  WARNING: world-writable — any user can modify this\n")
		}
	}
}
```

**Sample output:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Path:        /etc/passwd
Permissions: -rw-r--r-- (octal: 0644)
Owner:       root (UID: 0)
Group:       root (GID: 0)
Size:        2847 bytes
Hard links:  1
Setuid:      false
Setgid:      false
Sticky:      false
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Path:        /usr/bin/passwd
Permissions: -rwsr-xr-x (octal: 4755)
Owner:       root (UID: 0)
Group:       root (GID: 0)
Size:        68208 bytes
Hard links:  1
Setuid:      true
Setgid:      false
Sticky:      false
⚠  WARNING: setuid bit is set — this binary runs as root (UID 0)
⚠  CRITICAL: setuid-root binary — compromise gives root access
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Path:        /tmp
Permissions: drwxrwxrwt (octal: 1777)
Owner:       root (UID: 0)
Group:       root (GID: 0)
Size:        4096 bytes
Hard links:  15
Setuid:      false
Setgid:      false
Sticky:      true
```

---
## 7.2 Linux Capabilities

### 7.2.1 What Capabilities Replace — Splitting Root Into Granular Units

Before capabilities, the Linux kernel made a binary distinction:

- **UID 0 (root):** All privilege checks pass
- **UID != 0 (everyone else):** Normal permission checks apply

Linux capabilities (introduced in kernel 2.2, significantly extended in 2.6.24+)
break root's monolithic power into **individual units of privilege** called
capabilities. Each capability grants one specific type of privileged operation.

```
┌───────────────────────────────────────────────────────────┐
│                  BEFORE CAPABILITIES                       │
│                                                           │
│  Process needs to bind port 80?                           │
│    → Must run as root                                     │
│    → Gets ALL privileges                                  │
│    → Can also: load modules, mount fs, kill processes...  │
│                                                           │
├───────────────────────────────────────────────────────────┤
│                  WITH CAPABILITIES                         │
│                                                           │
│  Process needs to bind port 80?                           │
│    → Give it ONLY CAP_NET_BIND_SERVICE                    │
│    → Cannot: load modules, mount fs, kill processes...    │
│    → Principle of least privilege!                         │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### 7.2.2 Key Capabilities Reference

The kernel defines approximately 40 capabilities (as of kernel 6.x). Here are
the most important ones for systems programmers and container security:

| Capability | What it grants | Danger level |
|-----------|---------------|-------------|
| `CAP_NET_BIND_SERVICE` | Bind to privileged ports (< 1024) | Low |
| `CAP_NET_RAW` | Use raw sockets (ping, packet capture) | Medium |
| `CAP_NET_ADMIN` | Configure network interfaces, routing, firewall | High |
| `CAP_SYS_ADMIN` | Mount, swapon, sethostname, many sysfs ops — the "new root" | Critical |
| `CAP_SYS_PTRACE` | Trace any process with ptrace(2) | Critical |
| `CAP_SYS_MODULE` | Load/unload kernel modules | Critical |
| `CAP_SYS_RAWIO` | Perform raw I/O (ioperm, iopl) | Critical |
| `CAP_SYS_CHROOT` | Call chroot(2) | Medium |
| `CAP_SYS_TIME` | Set system clock | Low |
| `CAP_SYS_RESOURCE` | Override resource limits (ulimit) | Medium |
| `CAP_CHOWN` | Change file ownership (chown) | Medium |
| `CAP_DAC_OVERRIDE` | Bypass file read/write/execute permission checks | Critical |
| `CAP_DAC_READ_SEARCH` | Bypass file read permission and directory read/search | High |
| `CAP_FOWNER` | Bypass permission checks on operations that require file owner match | High |
| `CAP_FSETID` | Don't clear setuid/setgid bits on file modification | Medium |
| `CAP_KILL` | Send signals to any process (bypass permission checks) | High |
| `CAP_SETUID` | Manipulate process UIDs (setuid, setreuid, etc.) | Critical |
| `CAP_SETGID` | Manipulate process GIDs | Critical |
| `CAP_SETFCAP` | Set file capabilities | High |
| `CAP_MKNOD` | Create special files (device nodes) | Medium |
| `CAP_AUDIT_WRITE` | Write to the kernel audit log | Low |
| `CAP_SYS_BOOT` | Reboot the system | Medium |
| `CAP_LEASE` | Establish leases on files | Low |
| `CAP_IPC_LOCK` | Lock memory (mlock, mlockall) | Low |

> **`CAP_SYS_ADMIN` is the "junk drawer" capability.** It covers over 30 different
> privilege checks in the kernel — mount, pivot_root, sethostname, perf_event_open,
> BPF program loading, and many more. It is nearly as dangerous as full root.
> Container runtimes should almost NEVER grant it.


### 7.2.3 Capability Sets — The Five Sets

Every thread (not just process — Linux capabilities are per-thread) has **five**
capability sets. Understanding how they interact is crucial:

```
┌──────────────────────────────────────────────────────────┐
│               CAPABILITY SETS PER THREAD                  │
│                                                          │
│  ┌────────────┐                                          │
│  │ Effective  │  The set actually checked by the kernel  │
│  │   (E)      │  when a privileged operation is          │
│  │            │  attempted. If the cap is NOT in E,      │
│  │            │  the operation is denied.                │
│  └────────────┘                                          │
│                                                          │
│  ┌────────────┐                                          │
│  │ Permitted  │  The maximum set of capabilities the     │
│  │   (P)      │  thread MAY use. A cap can be moved      │
│  │            │  from P to E, but never added to P       │
│  │            │  once dropped.                           │
│  └────────────┘                                          │
│                                                          │
│  ┌────────────┐                                          │
│  │ Inheritable│  Caps that can be inherited across       │
│  │   (I)      │  execve(2). Combined with file caps      │
│  │            │  to determine the new process's caps.    │
│  └────────────┘                                          │
│                                                          │
│  ┌────────────┐                                          │
│  │ Bounding   │  Upper limit on caps that can be gained  │
│  │   (B)      │  through file capabilities or            │
│  │            │  inheritable sets. Can only be dropped,  │
│  │            │  never raised.                           │
│  └────────────┘                                          │
│                                                          │
│  ┌────────────┐                                          │
│  │ Ambient    │  Caps automatically added to P and E     │
│  │   (A)      │  across execve(2), even for non-setuid   │
│  │            │  programs without file caps. Added in    │
│  │            │  kernel 4.3 to fix inheritance.          │
│  └────────────┘                                          │
│                                                          │
│  INVARIANT: E ⊆ P  (effective is always subset of       │
│                      permitted)                          │
│  INVARIANT: A ⊆ P ∩ I (ambient must be in both          │
│                          permitted and inheritable)      │
└──────────────────────────────────────────────────────────┘
```

### 7.2.4 How Capabilities Are Inherited Across fork/exec

Understanding capability inheritance is critical for writing secure services:

**On fork(2):**
The child inherits all five capability sets from the parent — unchanged. This is
simple: fork creates an exact copy.

**On execve(2):**
This is where it gets complex. The new program's capabilities are computed as:

```
New P' = (Old P ∩ F_Inheritable ∩ Old I) | (F_Permitted ∩ B) | Old A
New E' = F_Effective ? New P' : Old A
New I' = Old I
New A' = (no-new-privs or F_Inheritable != 0) ? 0 : Old A
```

Where:
- `F_Permitted`, `F_Inheritable`, `F_Effective` = file capability sets
- `B` = bounding set
- `A` = ambient set

**In plain English:**

1. **File capabilities dominate.** If the binary has file capabilities set (via
   `setcap`), those largely determine the new capabilities.
2. **Ambient capabilities are the escape hatch.** They allow a parent to pass caps
   to a child without the child binary having file capabilities. This is how
   container runtimes grant capabilities to containerized processes.
3. **The bounding set is the ceiling.** Even file capabilities can't exceed it.

```
┌────────────────────────────────────────────────────────┐
│            CAPABILITY INHERITANCE ON EXEC               │
│                                                        │
│  Parent Process          File Caps        New Process  │
│  ┌───────────┐          ┌──────────┐     ┌──────────┐ │
│  │ P: {A,B,C}│   exec   │ FP: {B,D}│  →  │ P': ???  │ │
│  │ I: {A,B}  │──binary──│ FI: {B}  │     │ E': ???  │ │
│  │ E: {A,B,C}│  with    │ FE: yes  │     │ I': {A,B}│ │
│  │ A: {A}    │  file    │          │     │ A': {A}  │ │
│  │ B: {A..Z} │  caps    │          │     │ B: {A..Z}│ │
│  └───────────┘          └──────────┘     └──────────┘ │
│                                                        │
│  P' = (P ∩ FI ∩ I) | (FP ∩ B) | A                    │
│     = ({A,B,C} ∩ {B} ∩ {A,B}) | ({B,D} ∩ {A..Z}) | {A}│
│     = {B} | {B,D} | {A}                               │
│     = {A, B, D}                                        │
│                                                        │
│  E' = (FE is set) ? P' : A  = P' = {A, B, D}         │
└────────────────────────────────────────────────────────┘
```


### 7.2.5 File Capabilities — setcap and getcap

File capabilities are extended attributes (xattrs) stored on executable files
that grant capabilities when the file is executed. They are the capability
system's replacement for setuid.

```bash
# Set file capabilities — give a binary the ability to bind privileged ports
$ sudo setcap 'cap_net_bind_service=+ep' ./myserver

# Read file capabilities
$ getcap ./myserver
./myserver cap_net_bind_service=ep

# Remove all file capabilities
$ sudo setcap -r ./myserver

# List ALL files with capabilities on the system
$ getcap -r / 2>/dev/null
/usr/bin/ping cap_net_raw=ep
/usr/bin/traceroute6.iputils cap_net_raw=ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper cap_net_bind_service,cap_net_admin=ep
```

The `=+ep` syntax means:
- `e` = Add to the **Effective** set flag (caps become effective immediately on exec)
- `p` = Add to the **Permitted** file set

Without the `e` flag, the binary must explicitly raise capabilities from permitted
to effective using `capset(2)` — this is called a "capability-aware" binary.


### 7.2.6 Hands-On: Dropping Capabilities in Go Using x/sys/unix

A secure service should start with only the capabilities it needs and drop
everything else. Here's how to do it in Go:

```go
// drop_caps.go
//
// This program demonstrates how to drop Linux capabilities from a Go
// process using the x/sys/unix package. This is a critical security
// practice: any service that starts with elevated capabilities should
// immediately drop the ones it doesn't need.
//
// The program follows the principle of least privilege:
// 1. Start with whatever capabilities the process was given
// 2. Define the minimal set of capabilities actually needed
// 3. Drop everything else from the permitted, effective, and bounding sets
//
// This is exactly what container runtimes (Docker, containerd) do before
// executing the container's entrypoint — they compute the needed caps and
// drop the rest.
//
// Build and test:
//   go build -o drop_caps drop_caps.go
//   sudo setcap 'cap_net_bind_service,cap_net_raw,cap_sys_admin=+ep' ./drop_caps
//   ./drop_caps
package main

import (
	"fmt"
	"log"
	"os"
	"strings"

	"golang.org/x/sys/unix"
)

// capName maps capability numbers to human-readable names.
// The kernel defines these in <linux/capability.h>. We include
// all capabilities up to CAP_LAST_CAP (currently 40 in kernel 6.x).
// This mapping is essential for debugging — raw capability numbers
// are not meaningful to humans.
var capName = map[uintptr]string{
	0:  "CAP_CHOWN",
	1:  "CAP_DAC_OVERRIDE",
	2:  "CAP_DAC_READ_SEARCH",
	3:  "CAP_FOWNER",
	4:  "CAP_FSETID",
	5:  "CAP_KILL",
	6:  "CAP_SETGID",
	7:  "CAP_SETUID",
	8:  "CAP_SETPCAP",
	9:  "CAP_LINUX_IMMUTABLE",
	10: "CAP_NET_BIND_SERVICE",
	11: "CAP_NET_BROADCAST",
	12: "CAP_NET_ADMIN",
	13: "CAP_NET_RAW",
	14: "CAP_IPC_LOCK",
	15: "CAP_IPC_OWNER",
	16: "CAP_SYS_MODULE",
	17: "CAP_SYS_RAWIO",
	18: "CAP_SYS_CHROOT",
	19: "CAP_SYS_PTRACE",
	20: "CAP_SYS_PACCT",
	21: "CAP_SYS_ADMIN",
	22: "CAP_SYS_BOOT",
	23: "CAP_SYS_NICE",
	24: "CAP_SYS_RESOURCE",
	25: "CAP_SYS_TIME",
	26: "CAP_SYS_TTY_CONFIG",
	27: "CAP_MKNOD",
	28: "CAP_LEASE",
	29: "CAP_AUDIT_WRITE",
	30: "CAP_AUDIT_CONTROL",
	31: "CAP_SETFCAP",
	32: "CAP_MAC_OVERRIDE",
	33: "CAP_MAC_ADMIN",
	34: "CAP_SYSLOG",
	35: "CAP_WAKE_ALARM",
	36: "CAP_BLOCK_SUSPEND",
	37: "CAP_AUDIT_READ",
	38: "CAP_PERFMON",
	39: "CAP_BPF",
	40: "CAP_CHECKPOINT_RESTORE",
}

// lastCap is the highest capability number we know about.
// In practice, you can read /proc/sys/kernel/cap_last_cap to get
// the kernel's actual maximum. We hardcode 40 for clarity.
const lastCap = 40

// listCaps reads the current thread's capability sets by reading
// /proc/self/status. This is the simplest way to inspect caps —
// the alternative is the capget(2) syscall, which requires
// constructing the __user_cap_header_struct manually.
//
// The /proc/self/status file contains lines like:
//   CapInh: 0000000000000000
//   CapPrm: 0000003fffffffff
//   CapEff: 0000003fffffffff
//   CapBnd: 0000003fffffffff
//   CapAmb: 0000000000000000
//
// Each value is a hex bitmask where bit N corresponds to capability N.
func listCaps() {
	data, err := os.ReadFile("/proc/self/status")
	if err != nil {
		log.Fatalf("Failed to read /proc/self/status: %v", err)
	}

	for _, line := range strings.Split(string(data), "\n") {
		for _, prefix := range []string{"CapInh:", "CapPrm:", "CapEff:", "CapBnd:", "CapAmb:"} {
			if strings.HasPrefix(line, prefix) {
				fmt.Printf("  %s\n", strings.TrimSpace(line))
			}
		}
	}
}

// decodeCaps parses a hex capability bitmask string (like "0000003fffffffff")
// and returns a list of human-readable capability names that are set.
// This is useful for debugging — seeing "CAP_NET_BIND_SERVICE" is much
// more meaningful than seeing bit 10 in a hex mask.
func decodeCaps(hexMask string) []string {
	var val uint64
	fmt.Sscanf(hexMask, "%x", &val)

	var caps []string
	for i := uintptr(0); i <= lastCap; i++ {
		if val&(1<<i) != 0 {
			if name, ok := capName[i]; ok {
				caps = append(caps, name)
			} else {
				caps = append(caps, fmt.Sprintf("CAP_%d", i))
			}
		}
	}
	return caps
}

// dropBoundingCap removes a single capability from the bounding set.
// The bounding set is a ceiling — once a cap is removed from the bounding
// set, no future execve(2) can add it back. This is irreversible within
// the process tree.
//
// We use prctl(PR_CAPBSET_DROP, cap) which is the standard mechanism.
// This requires CAP_SETPCAP in the caller's effective set.
func dropBoundingCap(cap uintptr) error {
	return unix.Prctl(unix.PR_CAPBSET_DROP, cap, 0, 0, 0)
}

// setCaps sets the thread's permitted and effective capability sets
// using the capset(2) system call. The inheritable set is cleared
// to prevent capability leakage to child processes.
//
// This is the core capability manipulation function. The unix.CapUserHeader
// and unix.CapUserData structures map directly to the kernel's
// __user_cap_header_struct and __user_cap_data_struct.
//
// Note: Linux capabilities use two 32-bit words to represent the full
// 64-bit bitmask (for historical reasons — the original API only
	// supported 32 capabilities). We must set both words correctly.
func setCaps(capMask uint64) error {
	// Version 0x20080522 is _LINUX_CAPABILITY_VERSION_3, which supports
	// up to 64 capabilities using two data structures.
	hdr := unix.CapUserHeader{
		Version: unix.LINUX_CAPABILITY_VERSION_3,
		Pid:     0, // 0 means "this thread"
	}

	// Split the 64-bit mask into two 32-bit words.
	// Word 0 = capabilities 0-31, Word 1 = capabilities 32-63.
	data := [2]unix.CapUserData{
		{
			Effective:   uint32(capMask & 0xFFFFFFFF),
			Permitted:   uint32(capMask & 0xFFFFFFFF),
			Inheritable: 0, // Clear inheritable — defense in depth
		},
		{
			Effective:   uint32(capMask >> 32),
			Permitted:   uint32(capMask >> 32),
			Inheritable: 0,
		},
	}

	return unix.Capset(&hdr, &data[0])
}

func main() {
	fmt.Println("=== Current capabilities ===")
	listCaps()

	// Define the MINIMAL set of capabilities this service needs.
	// In this example, we only need CAP_NET_BIND_SERVICE (bit 10)
	// to bind to port 80. Everything else is unnecessary and
	// should be dropped.
	keepCaps := uint64(1 << 10) // Only CAP_NET_BIND_SERVICE

	fmt.Println("\n=== Dropping to minimal capability set ===")
	fmt.Println("Keeping only: CAP_NET_BIND_SERVICE")

	// Step 1: Drop all capabilities from the bounding set except
	// the ones we want to keep. This prevents any future exec'd
	// binary from gaining these capabilities.
	for cap := uintptr(0); cap <= lastCap; cap++ {
		if keepCaps&(1<<cap) == 0 {
			if err := dropBoundingCap(cap); err != nil {
				// EPERM means we don't have CAP_SETPCAP — expected
				// if we're not running with full caps
				fmt.Printf("  Note: could not drop bounding cap %s: %v\n",
					capName[cap], err)
			}
		}
	}

	// Step 2: Set the permitted and effective sets to only our
	// desired capabilities.
	if err := setCaps(keepCaps); err != nil {
		log.Fatalf("Failed to set capabilities: %v", err)
	}

	fmt.Println("\n=== Capabilities after dropping ===")
	listCaps()

	fmt.Println("\n✓ Process now has minimal capabilities")
	fmt.Println("  It can bind to port 80, but cannot:")
	fmt.Println("  - Load kernel modules")
	fmt.Println("  - Mount filesystems")
	fmt.Println("  - Change file ownership")
	fmt.Println("  - Trace other processes")
	fmt.Println("  - ... or do anything else requiring root")
}
```


### 7.2.7 Hands-On: Binding Port 80 Without Root (CAP_NET_BIND_SERVICE)

This is one of the most practical capability use cases — running a web server
on port 80 without running as root:

```go
// bind_port80.go
//
// This program demonstrates binding to a privileged port (port 80)
// without running as root, using only the CAP_NET_BIND_SERVICE capability.
//
// Traditional approach (DANGEROUS):
//   sudo ./webserver  ← runs entire process as root
//
// Capability approach (SAFE):
//   go build -o webserver bind_port80.go
//   sudo setcap 'cap_net_bind_service=+ep' ./webserver
//   ./webserver  ← runs as normal user, can bind port 80
//
// This is the same technique used by production web servers like nginx
// and caddy. The capability is checked in the kernel's inet_bind()
// function — if the port number is < 1024 and the process has
// CAP_NET_BIND_SERVICE in its effective set, the bind is allowed.
//
// Security benefit: If the web server is compromised, the attacker
// runs as a normal user with ONE extra capability — NOT as root.
package main

import (
	"context"
	"fmt"
	"log"
	"net"
	"net/http"
	"os"
	"os/signal"
	"strings"
	"syscall"
	"time"
)

// getCapStatus reads /proc/self/status and extracts the capability
// lines, returning them as a formatted string. This lets us display
// the process's current capabilities at startup for verification.
//
// In production, you would log this at startup to confirm your
// capability setup is correct — a misconfigured capability set
// is a common source of security bugs.
func getCapStatus() string {
	data, err := os.ReadFile("/proc/self/status")
	if err != nil {
		return fmt.Sprintf("(cannot read caps: %v)", err)
	}

	var result []string
	for _, line := range strings.Split(string(data), "\n") {
		if strings.HasPrefix(line, "Cap") {
			result = append(result, strings.TrimSpace(line))
		}
	}
	return strings.Join(result, "\n    ")
}

func main() {
	// Display current capabilities at startup.
	// In production, log these at INFO level so you can verify
	// the capability setup in your deployment.
	fmt.Println("=== Port 80 Binding Demo ===")
	fmt.Printf("PID:  %d\n", os.Getpid())
	fmt.Printf("UID:  %d\n", os.Getuid())
	fmt.Printf("GID:  %d\n", os.Getgid())
	fmt.Printf("Caps:\n    %s\n\n", getCapStatus())

	if os.Getuid() == 0 {
		fmt.Println("⚠  Running as root — this demo works, but defeats the purpose!")
		fmt.Println("   Try: go build -o server && sudo setcap 'cap_net_bind_service=+ep' ./server && ./server")
	}

	// Create a simple HTTP handler that reports the server's security context.
	// This is useful for verifying the setup from a browser or curl.
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
			fmt.Fprintf(w, "Hello from port 80!\n")
			fmt.Fprintf(w, "Running as UID: %d\n", os.Getuid())
			fmt.Fprintf(w, "PID: %d\n", os.Getpid())
		})

	// Bind to port 80.
	// Without CAP_NET_BIND_SERVICE, this would fail with:
	//   listen tcp :80: bind: permission denied
	//
	// The kernel checks CAP_NET_BIND_SERVICE in net/ipv4/af_inet.c:
	//   if (snum && inet_port_requires_bind_service(net, snum) &&
		//       !ns_capable(net->user_ns, CAP_NET_BIND_SERVICE))
	//       goto out;
	listener, err := net.Listen("tcp", ":80")
	if err != nil {
		log.Fatalf("Failed to bind port 80: %v\n"+
			"  Make sure you have CAP_NET_BIND_SERVICE:\n"+
			"  sudo setcap 'cap_net_bind_service=+ep' <binary>\n", err)
	}
	defer listener.Close()

	fmt.Println("✓ Successfully bound to port 80 without root!")
	fmt.Println("  Visit http://localhost:80/ to test")

	server := &http.Server{
		Handler:      mux,
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
	}

	// Graceful shutdown on SIGINT/SIGTERM.
	// Container runtimes send SIGTERM when stopping containers.
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)

	go func() {
		<-sigCh
		fmt.Println("\nShutting down...")
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		defer cancel()
		server.Shutdown(ctx)
	}()

	if err := server.Serve(listener); err != http.ErrServerClosed {
		log.Fatalf("Server error: %v", err)
	}
}
```


### 7.2.8 Why Containers Use Capabilities — Docker's Default Capability Set

Docker and other container runtimes use capabilities as a primary security
mechanism. When you run `docker run`, the container starts with a **restricted
set** of capabilities — not full root:

```
Docker's Default Capabilities (as of Docker 24.x):
─────────────────────────────────────────────────
GRANTED (14 capabilities):
  CAP_CHOWN            — Change file ownership
  CAP_DAC_OVERRIDE     — Bypass file permission checks
  CAP_FSETID           — Don't clear setuid/setgid on write
  CAP_FOWNER           — Bypass ownership checks
  CAP_MKNOD            — Create device special files
  CAP_NET_RAW          — Raw sockets (ping)
  CAP_SETGID           — Manipulate GIDs
  CAP_SETUID           — Manipulate UIDs
  CAP_SETFCAP          — Set file capabilities
  CAP_SETPCAP          — Transfer capabilities
  CAP_NET_BIND_SERVICE — Bind privileged ports
  CAP_SYS_CHROOT       — Use chroot
  CAP_KILL             — Send signals to any process
  CAP_AUDIT_WRITE      — Write to audit log

DROPPED (everything else, including):
  CAP_SYS_ADMIN        — Mount, BPF, many admin ops
  CAP_NET_ADMIN        — Network configuration
  CAP_SYS_MODULE       — Load kernel modules
  CAP_SYS_RAWIO        — Raw I/O access
  CAP_SYS_PTRACE       — Process tracing
  CAP_SYS_TIME         — Set system clock
  CAP_SYS_BOOT         — Reboot
  ... and ~20 more
```

You can modify the capability set with `--cap-add` and `--cap-drop`:

```bash
# Drop all capabilities, then add only what's needed
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myimage

# Add a dangerous capability (be very careful!)
docker run --cap-add=SYS_ADMIN myimage

# Run with literally ALL capabilities (DANGEROUS — equivalent to --privileged)
docker run --cap-add=ALL myimage
```

> **Security best practice:** Always start with `--cap-drop=ALL` and add back
> only what your application needs. This is the principle of least privilege
> applied to container security.

---
## 7.3 Seccomp — Syscall Filtering

### 7.3.1 What Seccomp Does — Restricting Which Syscalls a Process Can Make

**Seccomp** (Secure Computing Mode) is a Linux kernel facility that restricts
which system calls a process is allowed to make. If capabilities answer "does
this process have the *privilege* to do X?", seccomp answers "is this process
*allowed to attempt* X at all?"

```
┌─────────────────────────────────────────────────────────────┐
│              SECCOMP IN THE SYSCALL PATH                      │
│                                                             │
│  User Process                                               │
│       │                                                     │
│       ▼                                                     │
│  syscall(SYS_open, ...)                                     │
│       │                                                     │
│       ▼                                                     │
│  ┌──────────────────────┐                                   │
│  │   SECCOMP BPF FILTER │  ← Runs BEFORE any other check   │
│  │                      │                                   │
│  │  Is SYS_open allowed? │                                  │
│  │  Check args if needed │                                  │
│  └──────────┬───────────┘                                   │
│             │                                               │
│      ┌──────┴──────┐                                        │
│      │             │                                        │
│   ALLOW         KILL/ERRNO/TRAP/LOG                         │
│      │             │                                        │
│      ▼             ▼                                        │
│  Continue to    Process killed,                             │
│  DAC → Caps →   or syscall returns                          │
│  LSM → actual   error, or signal                            │
│  operation      sent, or logged                             │
└─────────────────────────────────────────────────────────────┘
```

> **Key insight:** Seccomp filters run *before* capability checks and LSM hooks.
> This makes seccomp the first line of defense — a denied syscall never even
> reaches the permission checking code.


### 7.3.2 Seccomp Strict Mode vs Seccomp-BPF

Linux supports two seccomp modes:

**Strict mode (seccomp mode 1):**
The original, extremely restrictive mode. The process can ONLY use four syscalls:
`read()`, `write()`, `_exit()`, and `sigreturn()`. Any other syscall immediately
kills the process with SIGKILL. This was designed for running untrusted
computation — pure number crunching that only reads input and writes output.

```go
// Enabling strict mode — almost never used in practice because
// it's too restrictive. The process can't even open files or
// allocate memory.
unix.Prctl(unix.PR_SET_SECCOMP, unix.SECCOMP_MODE_STRICT, 0, 0, 0)
```

**Seccomp-BPF (seccomp mode 2):**
The modern and useful mode, added in kernel 3.5. Instead of a fixed whitelist,
you provide a **BPF (Berkeley Packet Filter) program** that examines each syscall
and decides what to do. The BPF program can:

- **SECCOMP_RET_ALLOW** — Allow the syscall to proceed
- **SECCOMP_RET_KILL_PROCESS** — Kill the entire process
- **SECCOMP_RET_KILL_THREAD** — Kill just the calling thread
- **SECCOMP_RET_TRAP** — Send SIGSYS to the process (for debugging)
- **SECCOMP_RET_ERRNO** — Make the syscall return an error (e.g., EPERM)
- **SECCOMP_RET_TRACE** — Notify a ptrace tracer
- **SECCOMP_RET_LOG** — Allow but log the syscall
- **SECCOMP_RET_USER_NOTIF** — Notify a userspace supervisor (kernel 5.0+)


### 7.3.3 BPF Filter Programs — How They Work

Seccomp-BPF uses "classic BPF" (cBPF), not the newer eBPF. A BPF filter program
is a sequence of instructions that operates on a read-only data structure
representing the syscall:

```c
// The kernel provides this struct to the BPF program:
struct seccomp_data {
    int   nr;                    // Syscall number (e.g., __NR_open = 2)
    __u32 arch;                  // Architecture (e.g., AUDIT_ARCH_X86_64)
    __u64 instruction_pointer;   // Address of the syscall instruction
    __u64 args[6];               // Syscall arguments (arg0 through arg5)
};
```

A BPF program is an array of `sock_filter` structures:

```
┌──────────────────────────────────────────────────────┐
│              BPF FILTER STRUCTURE                      │
│                                                      │
│  Each instruction:                                   │
│  ┌─────────┬─────┬─────┬──────────────┐             │
│  │  code   │ jt  │ jf  │      k       │             │
│  │ (opcode)│(jump│(jump│  (constant)  │             │
│  │         │true)│false│              │             │
│  └─────────┴─────┴─────┴──────────────┘             │
│                                                      │
│  Example: "Allow read, write, exit; kill on others"  │
│                                                      │
│  BPF_STMT(BPF_LD|BPF_W|BPF_ABS, 0)  // Load nr     │
│  BPF_JUMP(BPF_JMP|BPF_JEQ, __NR_read,  2, 0)       │
│  BPF_JUMP(BPF_JMP|BPF_JEQ, __NR_write, 1, 0)       │
│  BPF_JUMP(BPF_JMP|BPF_JEQ, __NR_exit,  0, 1)       │
│  BPF_STMT(BPF_RET, SECCOMP_RET_ALLOW)               │
│  BPF_STMT(BPF_RET, SECCOMP_RET_KILL_PROCESS)        │
└──────────────────────────────────────────────────────┘
```

> **Why classic BPF and not eBPF?** Classic BPF programs are simpler, have
> a bounded execution time (no loops), and are easier for the kernel to verify
> as safe. eBPF is more powerful but that power is unnecessary for syscall
> filtering and would expand the attack surface.


### 7.3.4 Writing Seccomp Profiles in JSON (for Containers)

Docker and Kubernetes use JSON seccomp profiles. Here's the anatomy of one:

```json
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "comment": "Custom seccomp profile for a Go web server",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_X86",
        "SCMP_ARCH_AARCH64"
    ],
    "syscalls": [
        {
            "comment": "Core process operations — needed by any program",
            "names": [
                "read", "write", "close", "fstat", "lseek",
                "mmap", "mprotect", "munmap", "brk",
                "rt_sigaction", "rt_sigprocmask", "rt_sigreturn",
                "ioctl", "pread64", "pwrite64",
                "access", "pipe", "pipe2",
                "select", "sched_yield",
                "clone", "fork", "vfork", "execve",
                "exit", "exit_group", "wait4",
                "kill", "tgkill",
                "uname", "getpid", "getppid",
                "getuid", "geteuid", "getgid", "getegid",
                "gettid", "setsid"
            ],
            "action": "SCMP_ACT_ALLOW"
        },
        {
            "comment": "Network operations — needed by web server",
            "names": [
                "socket", "bind", "listen", "accept", "accept4",
                "connect", "sendto", "recvfrom", "sendmsg", "recvmsg",
                "shutdown", "getsockname", "getpeername",
                "setsockopt", "getsockopt",
                "epoll_create1", "epoll_ctl", "epoll_wait", "epoll_pwait"
            ],
            "action": "SCMP_ACT_ALLOW"
        },
        {
            "comment": "File operations — needed for reading configs, certs",
            "names": [
                "open", "openat", "stat", "lstat", "readlink",
                "getcwd", "chdir", "rename", "unlink", "mkdir",
                "readlinkat", "newfstatat", "statx"
            ],
            "action": "SCMP_ACT_ALLOW"
        },
        {
            "comment": "Go runtime needs — futex for goroutine scheduling",
            "names": [
                "futex", "nanosleep", "clock_gettime", "clock_nanosleep",
                "sigaltstack", "set_tid_address", "set_robust_list",
                "getrlimit", "prlimit64",
                "arch_prctl", "prctl",
                "getrandom"
            ],
            "action": "SCMP_ACT_ALLOW"
        },
        {
            "comment": "Block dangerous syscalls explicitly for clarity",
            "names": [
                "mount", "umount2", "pivot_root",
                "init_module", "finit_module", "delete_module",
                "reboot", "settimeofday", "stime",
                "swapon", "swapoff",
                "ptrace",
                "kexec_load", "kexec_file_load",
                "keyctl", "request_key", "add_key",
                "bpf",
                "userfaultfd",
                "perf_event_open"
            ],
            "action": "SCMP_ACT_KILL"
        }
    ]
}
```

Use this profile with Docker:

```bash
docker run --security-opt seccomp=./my-profile.json myimage
```


### 7.3.5 The seccomp() Syscall

The `seccomp()` system call (syscall number 317 on x86_64) is the modern
interface for installing seccomp filters:

```c
int seccomp(unsigned int operation, unsigned int flags, void *args);
```

**Operations:**
- `SECCOMP_SET_MODE_STRICT` — Enable strict mode
- `SECCOMP_SET_MODE_FILTER` — Install a BPF filter
- `SECCOMP_GET_ACTION_AVAIL` — Check if an action is supported
- `SECCOMP_GET_NOTIF_SIZES` — Get notification sizes (for user_notif)

**Flags for SECCOMP_SET_MODE_FILTER:**
- `SECCOMP_FILTER_FLAG_TSYNC` — Synchronize filter across all threads
- `SECCOMP_FILTER_FLAG_LOG` — Log all filter results
- `SECCOMP_FILTER_FLAG_SPEC_ALLOW` — Disable speculative execution mitigations
- `SECCOMP_FILTER_FLAG_NEW_LISTENER` — Get a notification FD

> **Important:** Once a seccomp filter is installed, it **cannot be removed.**
> Additional filters can be added (they are AND'd together — the most restrictive
> result wins), but you can never loosen a filter. This is a security feature.


### 7.3.6 Hands-On: Applying a Seccomp Filter from Go Using x/sys/unix

Here's a complete Go program that installs a seccomp-BPF filter:

```go
// seccomp_filter.go
//
// This program demonstrates how to install a seccomp-BPF filter from Go
// using the x/sys/unix package. The filter restricts which system calls
// the process is allowed to make — any blocked syscall will cause the
// process to be killed with SIGSYS.
//
// This is the same mechanism that Docker and Kubernetes use to restrict
// container syscalls, except they install the filter before exec'ing the
// container process.
//
// The program:
// 1. Displays the syscalls it will allow
// 2. Sets PR_SET_NO_NEW_PRIVS (required for non-root seccomp)
// 3. Installs a BPF filter that allows only essential syscalls
// 4. Demonstrates that allowed syscalls work
// 5. Attempts a blocked syscall (which kills the process)
//
// Build and run:
//   go build -o seccomp_filter seccomp_filter.go
//   ./seccomp_filter
//
// NOTE: The process will be killed by SIGSYS when it attempts the
// blocked syscall — this is expected behavior.
package main

import (
	"encoding/binary"
	"fmt"
	"log"
	"os"
	"unsafe"

	"golang.org/x/sys/unix"
)

// BPF instruction constants.
// These map directly to the kernel's BPF instruction set defined in
// <linux/filter.h> and <linux/bpf_common.h>. Classic BPF (cBPF) uses
// these opcodes — this is NOT the same as eBPF.
const (
	// Instruction classes
	bpfLD  = 0x00 // Load instruction
	bpfJMP = 0x05 // Jump instruction
	bpfRET = 0x06 // Return instruction

	// Load sizes
	bpfW = 0x00 // Word (32-bit)

	// Load modes
	bpfABS = 0x20 // Absolute offset into seccomp_data

	// Jump types
	bpfJEQ = 0x10 // Jump if equal

	// Source operand
	bpfK = 0x00 // Constant (immediate value)

	// Architecture for validation
	auditArchX86_64 = 0xC000003E // AUDIT_ARCH_X86_64
)

// Offsets into the seccomp_data structure.
// The kernel maps the syscall info into this structure before running
// the BPF filter. We use these offsets to load specific fields.
const (
	// offsetNR is the offset of the syscall number (nr) field in
	// struct seccomp_data. This is what we check to identify which
	// syscall is being attempted.
	offsetNR = 0

	// offsetArch is the offset of the architecture field.
	// We MUST check this first to prevent syscall confusion attacks
	// where a 32-bit syscall number maps to a different 64-bit call.
	offsetArch = 4
)

// bpfStmt creates a BPF statement instruction (no conditional jumps).
// Statements are used for LOAD and RETURN operations.
//
// Examples:
//   bpfStmt(bpfLD|bpfW|bpfABS, offsetNR) — Load syscall number
//   bpfStmt(bpfRET|bpfK, SECCOMP_RET_ALLOW) — Return ALLOW
func bpfStmt(code uint16, k uint32) unix.SockFilter {
	return unix.SockFilter{Code: code, Jt: 0, Jf: 0, K: k}
}

// bpfJump creates a BPF jump instruction for conditional branching.
// If the comparison is true, jump forward 'jt' instructions.
// If false, jump forward 'jf' instructions.
//
// Example:
//   bpfJump(bpfJMP|bpfJEQ|bpfK, SYS_READ, 0, 1)
//   — If accumulator == SYS_READ, continue (jt=0),
//     else skip 1 instruction (jf=1)
func bpfJump(code uint16, k uint32, jt, jf uint8) unix.SockFilter {
	return unix.SockFilter{Code: code, Jt: jt, Jf: jf, K: k}
}

// buildSeccompFilter constructs a BPF filter program that:
// 1. Validates the architecture (prevents syscall confusion attacks)
// 2. Allows a whitelist of syscalls needed by a Go program
// 3. Kills the process on any other syscall
//
// This is a "whitelist" (default-deny) approach, which is more secure
// than a "blacklist" (default-allow) approach because new/unknown
// syscalls are automatically blocked.
func buildSeccompFilter() []unix.SockFilter {
	// Syscalls to allow. These are the MINIMUM set needed for a basic
	// Go program. The Go runtime needs:
	//   - futex: goroutine synchronization
	//   - mmap/munmap/madvise: memory management
	//   - clone: creating goroutine threads
	//   - sigaltstack: signal handling
	//   - rt_sigaction/rt_sigprocmask: signal management
	//   - gettid/getpid: thread/process identification
	//   - tgkill: sending signals to specific threads
	//   - write: output (stdout/stderr)
	//   - read: input (stdin)
	//   - exit_group: clean process exit
	allowedSyscalls := []uint32{
		unix.SYS_READ,
		unix.SYS_WRITE,
		unix.SYS_CLOSE,
		unix.SYS_FSTAT,
		unix.SYS_MMAP,
		unix.SYS_MPROTECT,
		unix.SYS_MUNMAP,
		unix.SYS_BRK,
		unix.SYS_RT_SIGACTION,
		unix.SYS_RT_SIGPROCMASK,
		unix.SYS_RT_SIGRETURN,
		unix.SYS_CLONE,
		unix.SYS_EXIT_GROUP,
		unix.SYS_FUTEX,
		unix.SYS_SIGALTSTACK,
		unix.SYS_GETTID,
		unix.SYS_GETPID,
		unix.SYS_TGKILL,
		unix.SYS_NANOSLEEP,
		unix.SYS_CLOCK_GETTIME,
		unix.SYS_MADVISE,
		unix.SYS_SET_TID_ADDRESS,
		unix.SYS_SET_ROBUST_LIST,
		unix.SYS_PRLIMIT64,
		unix.SYS_GETRANDOM,
		unix.SYS_ARCH_PRCTL,
		unix.SYS_SCHED_YIELD,
		unix.SYS_OPENAT,
		unix.SYS_NEWFSTATAT,
		unix.SYS_FCNTL,
	}

	// Number of allowed syscalls determines the jump offsets.
	// Each allowed syscall adds one BPF_JUMP instruction.
	n := uint8(len(allowedSyscalls))

	filter := []unix.SockFilter{
		// === Step 1: Validate architecture ===
		// Load the arch field from seccomp_data
		bpfStmt(bpfLD|bpfW|bpfABS, offsetArch),
		// If arch != X86_64, kill the process.
		// This prevents "syscall confusion" attacks where a process
		// uses int 0x80 to invoke 32-bit syscalls (which have
			// different numbers than 64-bit syscalls).
		bpfJump(bpfJMP|bpfJEQ|bpfK, auditArchX86_64, 1, 0),
		bpfStmt(bpfRET|bpfK, 0x00000000), // SECCOMP_RET_KILL_PROCESS

		// === Step 2: Load the syscall number ===
		bpfStmt(bpfLD|bpfW|bpfABS, offsetNR),
	}

	// === Step 3: Check each allowed syscall ===
	// For each allowed syscall, add a jump-if-equal instruction.
	// If the syscall matches, jump to the ALLOW return.
	// The jump targets are calculated relative to the current instruction:
	//   jt = (remaining syscalls to check) = distance to ALLOW
	//   jf = 0 = fall through to next check
	for i, sc := range allowedSyscalls {
		// Distance from this instruction to the ALLOW return:
		// There are (n - i - 1) more check instructions after this one,
		// then the KILL instruction, then the ALLOW instruction.
		// So jump forward by (n - i - 1) + 1 = (n - i) to reach ALLOW.
		// But actually we skip over remaining checks + the kill = jt = n-1-i+1
		distToAllow := n - uint8(i) - 1
		filter = append(filter,
			bpfJump(bpfJMP|bpfJEQ|bpfK, sc, distToAllow, 0))
	}

	// === Step 4: Default action — KILL ===
	// If we fall through all checks without matching, kill the process.
	filter = append(filter,
		bpfStmt(bpfRET|bpfK, 0x00000000)) // SECCOMP_RET_KILL_PROCESS

	// === Step 5: ALLOW action ===
	// This is the jump target for matched syscalls.
	filter = append(filter,
		bpfStmt(bpfRET|bpfK, 0x7FFF0000)) // SECCOMP_RET_ALLOW

	return filter
}

func main() {
	fmt.Println("=== Seccomp-BPF Filter Demo ===")
	fmt.Printf("PID: %d\n\n", os.Getpid())

	// Step 1: Set PR_SET_NO_NEW_PRIVS.
	// This is REQUIRED to install seccomp filters as a non-root user.
	// It tells the kernel: "this process (and its children) can never
	// gain new privileges through execve — no setuid, no file caps."
	// This is a security prerequisite because seccomp filters could
	// otherwise be used to subvert setuid programs.
	fmt.Println("Setting PR_SET_NO_NEW_PRIVS...")
	if err := unix.Prctl(unix.PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0); err != nil {
		log.Fatalf("PR_SET_NO_NEW_PRIVS: %v", err)
	}

	// Step 2: Build the BPF filter.
	filter := buildSeccompFilter()
	fmt.Printf("Built BPF filter with %d instructions\n", len(filter))

	// Step 3: Install the filter using the seccomp(2) system call.
	// We construct a sock_fprog structure that points to our filter array.
	// The SECCOMP_FILTER_FLAG_TSYNC flag synchronizes the filter across
	// all threads — important for Go programs which use multiple OS threads.
	prog := &unix.SockFprog{
		Len:    uint16(len(filter)),
		Filter: &filter[0],
	}

	// Use RawSyscall to invoke seccomp(2) directly.
	// unix.SECCOMP_SET_MODE_FILTER = 1
	// SECCOMP_FILTER_FLAG_TSYNC = 1
	_, _, errno := unix.RawSyscall(
		unix.SYS_SECCOMP,
		1, // SECCOMP_SET_MODE_FILTER
		1, // SECCOMP_FILTER_FLAG_TSYNC
		uintptr(unsafe.Pointer(prog)),
	)
	if errno != 0 {
		log.Fatalf("seccomp(SET_MODE_FILTER): %v", errno)
	}

	fmt.Println("✓ Seccomp filter installed!")
	fmt.Println()

	// Step 4: Demonstrate allowed syscalls.
	// write(2) is in our whitelist, so this works:
	fmt.Println("Testing allowed syscall (write to stdout)... OK!")

	// Reading /proc/self/status is allowed because we permitted
	// openat, read, and close.
	data, err := os.ReadFile("/proc/self/status")
	if err == nil {
		// Find and print the Seccomp line to prove our filter is active.
		for _, line := range splitLines(data) {
			if len(line) > 7 && string(line[:7]) == "Seccomp" {
				fmt.Printf("  %s\n", string(line))
			}
		}
	}

	fmt.Println("\nAll allowed operations succeeded!")
	fmt.Println("The process is now restricted to only essential syscalls.")
	fmt.Println("Attempting a blocked syscall would kill the process with SIGSYS.")

	// Step 5: If you uncomment the following, the process will be killed:
	// unix.Reboot(unix.LINUX_REBOOT_CMD_CAD_OFF) // BLOCKED! → SIGSYS
}

// splitLines splits a byte slice into lines without importing strings
// (to minimize syscall surface for the demo).
func splitLines(data []byte) [][]byte {
	var lines [][]byte
	start := 0
	for i, b := range data {
		if b == '\n' {
			lines = append(lines, data[start:i])
			start = i + 1
		}
	}
	if start < len(data) {
		lines = append(lines, data[start:])
	}
	return lines
}

// Verify the size of SockFilter at compile time.
// The kernel expects each BPF instruction to be exactly 8 bytes.
var _ = [1]struct{}{}[unsafe.Sizeof(unix.SockFilter{})-8]
```

> **Compile-time size check:** The last line `var _ = [1]struct{}{}[unsafe.Sizeof(unix.SockFilter{})-8]`
> is a compile-time assertion that `SockFilter` is exactly 8 bytes. If the size
> differs, this produces a compile error. This is critical because the kernel
> expects exactly 8-byte BPF instructions.


### 7.3.7 Hands-On: Creating a Seccomp Profile That Blocks Dangerous Syscalls

A practical "hardened" seccomp profile for a Go HTTP server. This profile uses
the blacklist approach (default allow, block dangerous calls) which is easier to
deploy but less secure than the whitelist approach above:

```json
{
    "comment": "Hardened seccomp profile for Go HTTP services",
    "defaultAction": "SCMP_ACT_ALLOW",
    "architectures": [
        "SCMP_ARCH_X86_64",
        "SCMP_ARCH_AARCH64"
    ],
    "syscalls": [
        {
            "comment": "Block kernel module operations — no legitimate container use",
            "names": ["init_module", "finit_module", "delete_module"],
            "action": "SCMP_ACT_ERRNO",
            "errnoRet": 1
        },
        {
            "comment": "Block mount/unmount — containers should not modify mounts",
            "names": ["mount", "umount2", "pivot_root"],
            "action": "SCMP_ACT_ERRNO",
            "errnoRet": 1
        },
        {
            "comment": "Block reboot — container should not reboot host",
            "names": ["reboot"],
            "action": "SCMP_ACT_ERRNO",
            "errnoRet": 1
        },
        {
            "comment": "Block ptrace — prevents container escape via process debugging",
            "names": ["ptrace", "process_vm_readv", "process_vm_writev"],
            "action": "SCMP_ACT_ERRNO",
            "errnoRet": 1
        },
        {
            "comment": "Block dangerous namespace operations",
            "names": ["unshare", "setns"],
            "action": "SCMP_ACT_ERRNO",
            "errnoRet": 1
        },
        {
            "comment": "Block clock manipulation",
            "names": ["settimeofday", "clock_settime", "adjtimex"],
            "action": "SCMP_ACT_ERRNO",
            "errnoRet": 1
        },
        {
            "comment": "Block BPF and performance monitoring — attack surface reduction",
            "names": ["bpf", "perf_event_open"],
            "action": "SCMP_ACT_ERRNO",
            "errnoRet": 1
        },
        {
            "comment": "Block keyring operations — not needed by most services",
            "names": ["keyctl", "request_key", "add_key"],
            "action": "SCMP_ACT_ERRNO",
            "errnoRet": 1
        },
        {
            "comment": "Block userfaultfd — used in container escape exploits",
            "names": ["userfaultfd"],
            "action": "SCMP_ACT_ERRNO",
            "errnoRet": 1
        },
        {
            "comment": "Block kexec — loading a new kernel is never needed in containers",
            "names": ["kexec_load", "kexec_file_load"],
            "action": "SCMP_ACT_ERRNO",
            "errnoRet": 1
        },
        {
            "comment": "Block raw I/O port access",
            "names": ["ioperm", "iopl"],
            "action": "SCMP_ACT_ERRNO",
            "errnoRet": 1
        }
    ]
}
```


### 7.3.8 Hands-On: Debugging Seccomp Violations with Audit Logs

When a seccomp filter blocks a syscall, the kernel can log it via the audit
subsystem. Here's how to debug:

```bash
# Step 1: Modify your seccomp profile to LOG instead of KILL
# Change "action": "SCMP_ACT_KILL" to "action": "SCMP_ACT_LOG"
# or use SCMP_ACT_ERRNO which returns an error instead of killing

# Step 2: Check the audit log for seccomp violations
# On systems with auditd:
sudo ausearch -m seccomp --start recent

# On systems using journald:
sudo journalctl -k | grep seccomp

# Step 3: Parse the audit log entry
# type=SECCOMP msg=audit(1705312800.123:4567):
#   auid=1000 uid=1000 gid=1000 ses=2
#   subj=unconfined pid=12345 comm="myapp"
#   exe="/home/user/myapp" sig=0 arch=c000003e
#   syscall=35 compat=0 ip=0x7f1234567890 code=0x7ffc0000
#
# Key fields:
#   syscall=35  → SYS_NANOSLEEP (look up in /usr/include/asm/unistd_64.h)
#   sig=0       → No signal sent (ERRNO action, not KILL)
#   code=0x7ffc0000 → SECCOMP_RET_ERRNO (the filter returned an error)
#   arch=c000003e → x86_64

# Step 4: Map syscall number to name
python3 -c "
import ctypes, ctypes.util
libc = ctypes.CDLL(ctypes.util.find_library('c'))
# Or simply:
"
ausyscall x86_64 35
# Output: nanosleep

# Step 5: Decide whether to allow or investigate
# If the syscall is legitimate, add it to your profile.
# If it's unexpected, investigate why your app needs it.
```

**Debugging from Go:**

```go
// seccomp_debug.go
//
// This helper function installs a seccomp filter that LOGS blocked
// syscalls instead of killing the process. This is invaluable during
// development — you can run your application normally and then check
// the audit log to see which syscalls it actually uses.
//
// Workflow:
// 1. Install this logging filter with a very permissive whitelist
// 2. Run your application through its normal operations
// 3. Check /var/log/audit/audit.log for SECCOMP entries
// 4. Add any missing syscalls to your whitelist
// 5. Switch from SCMP_ACT_LOG to SCMP_ACT_KILL for production
package main

import (
	"bufio"
	"fmt"
	"os"
	"strconv"
	"strings"
)

// syscallNames maps x86_64 syscall numbers to their names.
// This is a subset — the full table has 300+ entries.
// Generated from /usr/include/asm/unistd_64.h.
var syscallNames = map[int]string{
	0: "read", 1: "write", 2: "open", 3: "close",
	4: "stat", 5: "fstat", 6: "lstat", 7: "poll",
	8: "lseek", 9: "mmap", 10: "mprotect", 11: "munmap",
	12: "brk", 13: "rt_sigaction", 14: "rt_sigprocmask",
	15: "rt_sigreturn", 16: "ioctl", 17: "pread64",
	35: "nanosleep", 39: "getpid", 56: "clone",
	57: "fork", 59: "execve", 60: "exit", 62: "kill",
	72: "fcntl", 87: "unlink", 89: "readlink",
	158: "arch_prctl", 165: "mount", 166: "umount2",
	175: "init_module", 176: "delete_module",
	186: "gettid", 202: "futex", 218: "set_tid_address",
	231: "exit_group", 257: "openat", 262: "newfstatat",
	302: "prlimit64", 318: "getrandom",
}

// parseSeccompAuditLog reads audit log entries and decodes seccomp
// violations into human-readable format. This helps identify which
// syscalls your application needs in its seccomp profile.
//
// Each audit log line for seccomp looks like:
//   type=SECCOMP ... syscall=NNN comm="myapp" ...
func parseSeccompAuditLog(logPath string) error {
	f, err := os.Open(logPath)
	if err != nil {
		return fmt.Errorf("open audit log: %w", err)
	}
	defer f.Close()

	scanner := bufio.NewScanner(f)
	for scanner.Scan() {
		line := scanner.Text()
		if !strings.Contains(line, "type=SECCOMP") {
			continue
		}

		// Extract key fields from the audit log line
		fields := map[string]string{}
		for _, part := range strings.Fields(line) {
			if kv := strings.SplitN(part, "=", 2); len(kv) == 2 {
				fields[kv[0]] = kv[1]
			}
		}

		syscallNum, _ := strconv.Atoi(fields["syscall"])
		name := syscallNames[syscallNum]
		if name == "" {
			name = "unknown"
		}

		fmt.Printf("BLOCKED: syscall=%d (%s) pid=%s comm=%s\n",
			syscallNum, name, fields["pid"], fields["comm"])
	}

	return scanner.Err()
}

func main() {
	logPath := "/var/log/audit/audit.log"
	if len(os.Args) > 1 {
		logPath = os.Args[1]
	}

	fmt.Printf("Parsing seccomp violations from: %s\n\n", logPath)
	if err := parseSeccompAuditLog(logPath); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
}
```


### 7.3.9 Docker and Kubernetes Default Seccomp Profiles

**Docker's default seccomp profile** (as of Docker 24.x) blocks approximately
44 syscalls out of ~300+ available. Key blocked syscalls include:

```
Blocked by Docker's default seccomp profile:
─────────────────────────────────────────────
Syscall                Reason
──────────────────     ──────────────────────────────
acct                   Process accounting
add_key                Kernel keyring manipulation
bpf                    BPF program loading
clock_adjtime          Clock manipulation
clock_settime          Clock manipulation
create_module          Kernel module creation
delete_module          Kernel module deletion
finit_module           Kernel module loading
get_kernel_syms        Kernel symbol table access
init_module            Kernel module loading
ioperm                 Raw I/O port access
iopl                   Raw I/O port access
kcmp                   Process comparison
kexec_file_load        Load new kernel
kexec_load             Load new kernel
keyctl                 Kernel keyring manipulation
mount                  Filesystem mounting
move_mount             Filesystem mounting (newer)
open_tree              Filesystem mounting (newer)
perf_event_open        Performance monitoring
personality            Process execution domain
pivot_root             Root filesystem pivot
ptrace                 Process debugging/tracing
query_module           Kernel module query
reboot                 System reboot
request_key            Kernel keyring
setns                  Namespace switching
stime                  Clock manipulation
swapon                 Swap space management
swapoff                Swap space management
sysfs                  Sysfs access
_sysctl                Deprecated sysctl
umount2                Filesystem unmounting
unshare                Namespace creation
userfaultfd            Page fault handling (exploit vector)
ustat                  Deprecated filesystem stat
vm86                   x86 virtual mode
vm86old                x86 virtual mode
```

**Kubernetes** added seccomp support progressively:
- **v1.19+**: Seccomp GA with `securityContext.seccompProfile`
- **v1.22+**: `SeccompDefault` feature gate (alpha)
- **v1.25+**: `SeccompDefault` feature gate (beta/stable)
- **v1.27+**: `RuntimeDefault` seccomp profile recommended

```yaml
# Kubernetes Pod with seccomp profile
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault  # Use container runtime's default profile
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      seccompProfile:
        type: Localhost
        localhostProfile: profiles/my-custom-profile.json
```

---
## 7.4 AppArmor — Path-Based Mandatory Access Control

### 7.4.1 What AppArmor Does

AppArmor is a **Linux Security Module (LSM)** that provides path-based mandatory
access control. Unlike SELinux (which labels every object), AppArmor profiles
define what a specific *program* can access by filesystem path.

```
┌────────────────────────────────────────────────────────────┐
│                 APPARMOR ARCHITECTURE                        │
│                                                            │
│  ┌─────────────┐    ┌─────────────────────┐               │
│  │  /usr/bin/   │    │  AppArmor Profile    │               │
│  │  myapp       │────│  for /usr/bin/myapp  │               │
│  │             │    │                     │               │
│  └──────┬──────┘    │  ALLOW:             │               │
│         │           │    /etc/myapp/** r  │               │
│         │           │    /var/log/myapp/** rw│              │
│         │           │    /tmp/** rw        │               │
│         │           │    network tcp       │               │
│         │           │                     │               │
│         │           │  DENY:              │               │
│         │           │    everything else   │               │
│         │           └─────────────────────┘               │
│         │                                                  │
│         ▼                                                  │
│  syscall(open("/etc/shadow"))                              │
│         │                                                  │
│         ▼                                                  │
│  ┌──────────────────┐                                      │
│  │  Kernel LSM Hook │                                      │
│  │  → Check AppArmor│                                      │
│  │  → /etc/shadow   │                                      │
│  │    not in profile │                                      │
│  │  → DENIED!       │                                      │
│  └──────────────────┘                                      │
└────────────────────────────────────────────────────────────┘
```

**Key characteristics of AppArmor:**

1. **Path-based** — Rules reference filesystem paths, not labels
2. **Per-program** — Each program gets its own profile
3. **Default-deny** — If a path isn't explicitly allowed, it's blocked
4. **Transparent** — Programs don't need modification to work with AppArmor
5. **Stackable** — Multiple profiles can be applied (kernel 5.1+)


### 7.4.2 Profile Modes: Enforce, Complain, Unconfined

AppArmor profiles operate in three modes:

| Mode | Behavior | Use case |
|------|----------|----------|
| **Enforce** | Violations are blocked and logged | Production |
| **Complain** | Violations are logged but allowed | Development/profiling |
| **Unconfined** | No restrictions | Default (no profile loaded) |

```bash
# Check which profiles are loaded and their modes
sudo aa-status

# Set a profile to complain mode (for testing)
sudo aa-complain /etc/apparmor.d/usr.bin.myapp

# Set a profile to enforce mode (for production)
sudo aa-enforce /etc/apparmor.d/usr.bin.myapp

# Disable a profile entirely
sudo aa-disable /etc/apparmor.d/usr.bin.myapp
```


### 7.4.3 Profile Syntax — File Rules, Network Rules, Capability Rules

An AppArmor profile is a text file that defines what a program can do:

```
# /etc/apparmor.d/usr.bin.mygoserver
#
# AppArmor profile for a Go HTTP server.
# This profile restricts the server to only the resources it needs:
# - Read its configuration and TLS certificates
# - Write to its log directory
# - Listen on network sockets
# - Use specific Linux capabilities
#
# Profile naming convention: the path to the binary with '/' replaced by '.'

#include <tunables/global>

# The profile name matches the binary's absolute path.
# This tells AppArmor to apply this profile whenever /usr/bin/mygoserver
# is executed.
/usr/bin/mygoserver {

  # ──────────────────────────────────────────────
  # Include standard abstractions
  # ──────────────────────────────────────────────
  
  # base: fundamental libc and system file access
  #include <abstractions/base>
  
  # nameservice: DNS resolution (/etc/resolv.conf, /etc/hosts, etc.)
  #include <abstractions/nameservice>
  
  # openssl: TLS library file access
  #include <abstractions/openssl>

  # ──────────────────────────────────────────────
  # Capability rules
  # ──────────────────────────────────────────────
  
  # Allow binding to privileged ports (< 1024).
  # This maps to the Linux capability CAP_NET_BIND_SERVICE.
  capability net_bind_service,
  
  # Allow setting resource limits for the Go runtime.
  # Some Go programs call setrlimit to adjust stack sizes.
  capability sys_resource,

  # ──────────────────────────────────────────────
  # Network rules
  # ──────────────────────────────────────────────
  
  # Allow TCP network access (for HTTP server).
  # Format: network <domain> <type>
  network inet stream,      # IPv4 TCP
  network inet6 stream,     # IPv6 TCP
  network inet dgram,       # IPv4 UDP (for DNS)
  network inet6 dgram,      # IPv6 UDP (for DNS)
  
  # Deny raw sockets — the server doesn't need them.
  # Explicit denials are logged with higher priority.
  deny network raw,

  # ──────────────────────────────────────────────
  # File rules
  # ──────────────────────────────────────────────
  
  # SYNTAX: <path> <permissions>,
  # Permissions:
  #   r  = read
  #   w  = write
  #   a  = append (write-only, no truncate)
  #   x  = execute
  #   m  = memory map executable (mmap PROT_EXEC)
  #   k  = file locking
  #   l  = create hard links
  #
  # Globbing:
  #   *  = matches any characters except /
  #   ** = matches any characters including /
  #   ?  = matches any single character
  #   {a,b} = alternation

  # The binary itself must be readable and executable
  /usr/bin/mygoserver                  mr,

  # Configuration files — read only
  /etc/mygoserver/                     r,
  /etc/mygoserver/**                   r,
  
  # TLS certificates and keys — read only
  /etc/ssl/certs/**                    r,
  /etc/mygoserver/tls/*.pem            r,
  /etc/mygoserver/tls/*.key            r,

  # Log directory — append only (prevents log tampering)
  /var/log/mygoserver/                 r,
  /var/log/mygoserver/**               a,

  # Runtime data directory — read/write
  /var/lib/mygoserver/                 r,
  /var/lib/mygoserver/**               rw,

  # Temporary files
  /tmp/mygoserver-*                    rw,

  # Go runtime needs
  /proc/sys/net/core/somaxconn         r,
  /sys/kernel/mm/transparent_hugepage/hpage_pmd_size r,

  # Deny access to sensitive files explicitly.
  # While the default-deny policy would block these anyway,
  # explicit denials generate log entries with higher severity
  # and make the policy's intent clearer.
  deny /etc/shadow                     rwx,
  deny /etc/sudoers                    rwx,
  deny /root/**                        rwx,
  deny /proc/kcore                     rwx,
  deny /proc/sysrq-trigger            rwx,

  # Deny all mount operations
  deny mount,
  deny umount,
  deny pivot_root,

  # ──────────────────────────────────────────────
  # Signal rules (AppArmor 2.9+)
  # ──────────────────────────────────────────────
  
  # Allow receiving SIGTERM and SIGINT (for graceful shutdown)
  signal receive set=(term, int) peer=unconfined,
  
  # Allow sending signals to self (Go runtime uses SIGURG)
  signal send set=(urg) peer=/usr/bin/mygoserver,
}
```


### 7.4.4 Hands-On: Writing an AppArmor Profile for a Go Application

Let's walk through the process of creating an AppArmor profile from scratch:

```bash
# Step 1: Generate a skeleton profile
# aa-genprof runs the program in complain mode and watches what
# resources it accesses, then suggests rules.
sudo aa-genprof /usr/bin/mygoserver

# Step 2: In another terminal, exercise the application
# Do everything the app normally does: serve requests, read configs,
# write logs, handle signals.
curl http://localhost:8080/
curl http://localhost:8080/api/health
kill -SIGTERM $(pidof mygoserver)

# Step 3: Back in the aa-genprof terminal, press 'S' to scan logs
# and 'A' to allow suggested rules, or 'D' to deny them.

# Step 4: Save and enforce the profile
# The profile is saved to /etc/apparmor.d/
```

For a **complete, tested** AppArmor profile for a Go application, here's what we
need to account for:

```go
// apparmor_check.go
//
// This program checks whether AppArmor is active and which profile
// is applied to the current process. This is useful as a startup
// diagnostic — your Go service should verify its security context
// at launch.
//
// AppArmor exposes the current profile via /proc/self/attr/apparmor/current
// (kernel 5.6+) or /proc/self/attr/current (older kernels).
//
// In production, log this at startup:
//   "AppArmor profile: mygoserver (enforce)"
// This confirms the security policy is active and correctly applied.
//
// Usage:
//   go run apparmor_check.go
package main

import (
	"fmt"
	"os"
	"strings"
)

// getAppArmorProfile reads the current process's AppArmor profile
// from the proc filesystem. The kernel exposes this information in
// two possible locations:
//
//   /proc/self/attr/apparmor/current — preferred (kernel 5.6+)
//   /proc/self/attr/current — fallback (older kernels)
//
// The content is a string like:
//   "mygoserver (enforce)"
//   "mygoserver (complain)"
//   "unconfined"
//
// Returns the profile name and mode, or "unconfined" if no profile
// is applied.
func getAppArmorProfile() (name string, mode string, err error) {
	// Try the newer, AppArmor-specific path first.
	// This path was added in kernel 5.6 to avoid conflicts with
	// SELinux, which also uses /proc/self/attr/current.
	paths := []string{
		"/proc/self/attr/apparmor/current",
		"/proc/self/attr/current",
	}

	var data []byte
	for _, path := range paths {
		data, err = os.ReadFile(path)
		if err == nil {
			break
		}
	}
	if err != nil {
		return "", "", fmt.Errorf("cannot read AppArmor profile: %w", err)
	}

	profile := strings.TrimSpace(string(data))
	// Remove the null byte that the kernel sometimes appends
	profile = strings.TrimRight(profile, "\x00")

	if profile == "unconfined" {
		return "unconfined", "unconfined", nil
	}

	// Parse "profile_name (mode)" format
	if idx := strings.LastIndex(profile, " ("); idx != -1 {
			name = profile[:idx]
			mode = strings.Trim(profile[idx+2:], ")")
		return name, mode, nil
	}

	return profile, "unknown", nil
}

// checkAppArmorEnabled checks if AppArmor is enabled at the kernel level
// by reading /sys/module/apparmor/parameters/enabled.
// Returns true if AppArmor is compiled into the kernel and enabled.
func checkAppArmorEnabled() bool {
	data, err := os.ReadFile("/sys/module/apparmor/parameters/enabled")
	if err != nil {
		return false
	}
	return strings.TrimSpace(string(data)) == "Y"
}

func main() {
	fmt.Println("=== AppArmor Status Check ===")
	fmt.Printf("PID: %d\n", os.Getpid())

	// Check if AppArmor is enabled in the kernel
	if checkAppArmorEnabled() {
		fmt.Println("AppArmor: ENABLED")
	} else {
		fmt.Println("AppArmor: DISABLED (or not available)")
		fmt.Println("  On Ubuntu/SUSE, install: sudo apt install apparmor apparmor-utils")
		return
	}

	// Get current profile
	name, mode, err := getAppArmorProfile()
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}

	fmt.Printf("Profile: %s\n", name)
	fmt.Printf("Mode:    %s\n", mode)

	switch mode {
	case "enforce":
		fmt.Println("✓ Process is confined by AppArmor in enforce mode")
		fmt.Println("  Violations will be BLOCKED and logged")
	case "complain":
		fmt.Println("⚠ Process is in complain mode — violations are LOGGED but ALLOWED")
		fmt.Println("  Switch to enforce for production: sudo aa-enforce <profile>")
	case "unconfined":
		fmt.Println("⚠ Process is UNCONFINED — no AppArmor restrictions!")
		fmt.Println("  Load a profile: sudo apparmor_parser -r /etc/apparmor.d/<profile>")
	}
}
```


### 7.4.5 Hands-On: Loading and Enforcing an AppArmor Profile

```bash
# ──────────────────────────────────────────────────────
# Complete workflow: Create → Load → Test → Enforce
# ──────────────────────────────────────────────────────

# Step 1: Write the profile (save as /etc/apparmor.d/usr.bin.mygoserver)
sudo vim /etc/apparmor.d/usr.bin.mygoserver

# Step 2: Parse and load the profile in complain mode first
sudo apparmor_parser -C /etc/apparmor.d/usr.bin.mygoserver
# -C = complain mode (log violations but don't block)

# Step 3: Verify the profile is loaded
sudo aa-status | grep mygoserver
# Should show: /usr/bin/mygoserver (complain)

# Step 4: Run your application and exercise all code paths
/usr/bin/mygoserver &
curl http://localhost:8080/
# ... run all tests ...

# Step 5: Check for violations in the log
sudo journalctl -k | grep apparmor | grep DENIED
# OR
sudo cat /var/log/kern.log | grep apparmor

# Step 6: If there are legitimate violations, update the profile
# and reload:
sudo apparmor_parser -r /etc/apparmor.d/usr.bin.mygoserver

# Step 7: Switch to enforce mode for production
sudo aa-enforce /etc/apparmor.d/usr.bin.mygoserver

# Step 8: Verify enforce mode
sudo aa-status | grep mygoserver
# Should show: /usr/bin/mygoserver (enforce)
```

### 7.4.6 AppArmor in Kubernetes — Annotations and Pod Security

Kubernetes supports AppArmor through container annotations (and natively in
newer versions):

```yaml
# Kubernetes Pod with AppArmor profile
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod
  annotations:
    # Format: container.apparmor.security.beta.kubernetes.io/<container-name>
    # Values:
    #   runtime/default — use container runtime's default profile
    #   localhost/<profile-name> — use a profile loaded on the node
    #   unconfined — no AppArmor restrictions
    container.apparmor.security.beta.kubernetes.io/app: localhost/mygoserver
spec:
  containers:
  - name: app
    image: mygoserver:latest
    ports:
    - containerPort: 8080
---
# Kubernetes 1.30+ native AppArmor support (no longer beta annotation)
apiVersion: v1
kind: Pod
metadata:
  name: apparmor-pod-native
spec:
  containers:
  - name: app
    image: mygoserver:latest
    securityContext:
      appArmorProfile:
        type: Localhost
        localhostProfile: mygoserver
```

> **Prerequisite:** The AppArmor profile must be loaded on every node where
> the pod might be scheduled. Use a DaemonSet or node provisioning to deploy
> profiles across your cluster.

---

## 7.5 SELinux — Label-Based Mandatory Access Control

### 7.5.1 What SELinux Does

**SELinux (Security-Enhanced Linux)** is a Mandatory Access Control system
developed by the NSA and Red Hat. Unlike AppArmor's path-based approach,
SELinux assigns **labels** (called security contexts) to every object in the
system — files, processes, ports, devices — and uses a policy to control which
labels can interact.

```
┌──────────────────────────────────────────────────────────────┐
│                   SELINUX ARCHITECTURE                        │
│                                                              │
│  Every object has a LABEL (security context):                │
│                                                              │
│  Process label:  system_u:system_r:httpd_t:s0               │
│  File label:     system_u:object_r:httpd_sys_content_t:s0   │
│  Port label:     system_u:object_r:http_port_t:s0           │
│                                                              │
│  ┌─────────────┐     ┌────────────┐    ┌─────────────┐     │
│  │   Process    │     │   SELinux   │    │    File     │     │
│  │  httpd_t     │────▶│   Policy   │───▶│ httpd_sys_  │     │
│  │             │     │            │    │ content_t   │     │
│  └─────────────┘     │ httpd_t    │    └─────────────┘     │
│                      │ can READ   │                         │
│                      │ httpd_sys_ │                         │
│                      │ content_t  │    ┌─────────────┐     │
│                      │            │    │    File     │     │
│                      │ httpd_t    │───▶│ shadow_t    │     │
│                      │ CANNOT     │    │ (/etc/shadow)│     │
│                      │ access     │    └─────────────┘     │
│                      │ shadow_t   │                         │
│                      └────────────┘                         │
│                                                              │
│  FORMAT: user:role:type:level                                │
│          │     │    │     │                                   │
│          │     │    │     └── MLS/MCS sensitivity level      │
│          │     │    └── Type (most important for enforcement)│
│          │     └── Role (which types a user can enter)       │
│          └── SELinux user (mapped from Linux user)           │
└──────────────────────────────────────────────────────────────┘
```


### 7.5.2 Contexts: user:role:type:level

Every SELinux label has four components:

**1. User (e.g., `system_u`, `unconfined_u`):**
SELinux users are separate from Linux users. They control which roles a
login user can assume. Common SELinux users:
- `system_u` — System processes and files
- `unconfined_u` — Unrestricted users (default on targeted policy)
- `staff_u` — Staff who can use sudo
- `user_u` — Restricted users

**2. Role (e.g., `system_r`, `object_r`):**
Roles are an RBAC layer between users and types. They restrict which
types a user can transition to.
- `system_r` — For system processes (daemons)
- `object_r` — For files and other objects (not processes)
- `unconfined_r` — Unrestricted role
- `staff_r` — Staff role (can sudo)

**3. Type (e.g., `httpd_t`, `shadow_t`):**
The type is the **most important** part of the label. Almost all SELinux
policy rules are based on types. This is called **Type Enforcement (TE).**
- Process types are called "domains" (e.g., `httpd_t`, `sshd_t`)
- File types are called "types" (e.g., `httpd_sys_content_t`, `shadow_t`)

**4. Level (e.g., `s0`, `s0:c0.c1023`):**
Used for Multi-Level Security (MLS) and Multi-Category Security (MCS).
In the common "targeted" policy, most objects have `s0`.
MCS categories (like `c0.c1023`) are used by container runtimes to
isolate containers from each other.


### 7.5.3 Policy Types: Targeted, Strict, MLS

| Policy | Description | Where used |
|--------|------------|-----------|
| **Targeted** | Only confines specific network-facing daemons; everything else runs "unconfined" | Default on RHEL/Fedora |
| **Strict** | Confines ALL processes — nothing runs unconfined | High-security environments |
| **MLS** | Adds Multi-Level Security (top secret, secret, classified, etc.) | Military/government systems |

In practice, you'll almost always work with the **targeted** policy. It
confines services like httpd, sshd, named, etc., while leaving user
sessions unconfined.


### 7.5.4 Booleans, Type Enforcement, and Role-Based Access

**Booleans** are runtime switches that modify policy behavior without
recompiling the policy:

```bash
# List all booleans
sudo getsebool -a

# Common booleans:
sudo getsebool httpd_can_network_connect
# httpd_can_network_connect --> off

# Enable a boolean (persistent across reboots)
sudo setsebool -P httpd_can_network_connect on

# Common use cases:
# - httpd_can_network_connect: Allow web server to make outbound connections
# - httpd_can_sendmail: Allow web server to send email
# - container_manage_cgroup: Allow containers to manage cgroups
# - selinuxuser_execmod: Allow users to run code from writable memory
```

**Type Enforcement rules** are the core of SELinux policy:

```
# Allow rule format:
# allow <source_type> <target_type>:<class> { <permissions> };

# Example: Allow httpd to read web content files
allow httpd_t httpd_sys_content_t:file { read open getattr };
allow httpd_t httpd_sys_content_t:dir  { search open getattr };

# Example: Allow httpd to bind to HTTP ports
allow httpd_t http_port_t:tcp_socket { name_bind };

# Example: Allow httpd to connect to database ports
allow httpd_t postgresql_port_t:tcp_socket { name_connect };
```


### 7.5.5 Hands-On: Examining SELinux Contexts

```bash
# ──────────────────────────────────────────────────────
# Examining SELinux labels on a running system
# ──────────────────────────────────────────────────────

# Check SELinux status
getenforce
# Output: Enforcing, Permissive, or Disabled

sestatus
# SELinux status:                 enabled
# SELinuxfs mount:                /sys/fs/selinux
# SELinux root directory:         /etc/selinux
# Loaded policy name:             targeted
# Current mode:                   enforcing
# Mode from config file:          enforcing
# Policy MLS status:              enabled
# Policy deny_unknown status:     allowed
# Memory protection checking:     actual (secure)
# Max kernel policy version:      33

# View file contexts
ls -Z /etc/passwd
# system_u:object_r:passwd_file_t:s0 /etc/passwd

ls -Z /etc/shadow
# system_u:object_r:shadow_t:s0 /etc/shadow

ls -Z /var/www/html/
# system_u:object_r:httpd_sys_content_t:s0 index.html

# View process contexts
ps -eZ | grep httpd
# system_u:system_r:httpd_t:s0   1234 ?  00:00:00 httpd

ps -eZ | grep sshd
# system_u:system_r:sshd_t:s0-s0:c0.c1023 1100 ? 00:00:00 sshd

# View your own context
id -Z
# unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023

# View port contexts
sudo semanage port -l | grep http
# http_port_t     tcp  80, 81, 443, 488, 8008, 8009, 8443, 9000
```

**Examining SELinux contexts from Go:**

```go
// selinux_context.go
//
// This program reads and displays the SELinux security context of the
// current process and specified files. SELinux labels are the foundation
// of all access control decisions on RHEL/Fedora systems, and
// understanding them is essential for debugging "Permission denied"
// errors that are caused by SELinux policy rather than DAC permissions.
//
// The program reads contexts from:
//   /proc/self/attr/current — process's own SELinux label
//   Extended attributes — file SELinux labels (security.selinux xattr)
//
// Usage:
//   go run selinux_context.go /etc/passwd /etc/shadow /var/www/html
package main

import (
	"fmt"
	"os"
	"strings"

	"golang.org/x/sys/unix"
)

// SELinuxContext represents the four components of an SELinux
// security context. Every object in an SELinux system has this
// label, and all access control decisions are based on comparing
// the source (process) and target (file/port/etc.) labels.
type SELinuxContext struct {
	User  string // SELinux user (e.g., system_u, unconfined_u)
	Role  string // RBAC role (e.g., system_r, object_r)
	Type  string // Type/domain (e.g., httpd_t, shadow_t) — MOST IMPORTANT
	Level string // MLS/MCS level (e.g., s0, s0:c0.c1023)
	Raw   string // The full unparsed context string
}

// parseContext splits an SELinux context string into its four components.
// The format is always user:role:type:level, where level may contain colons
// (e.g., s0:c0.c1023). We split on the first three colons.
func parseContext(raw string) SELinuxContext {
	raw = strings.TrimSpace(raw)
	raw = strings.TrimRight(raw, "\x00") // Remove null terminator

	parts := strings.SplitN(raw, ":", 4)
	ctx := SELinuxContext{Raw: raw}

	if len(parts) >= 1 {
		ctx.User = parts[0]
	}
	if len(parts) >= 2 {
		ctx.Role = parts[1]
	}
	if len(parts) >= 3 {
		ctx.Type = parts[2]
	}
	if len(parts) >= 4 {
		ctx.Level = parts[3]
	}

	return ctx
}

// getProcessContext reads the current process's SELinux context from
// /proc/self/attr/current. The kernel populates this file with the
// process's security context on every read.
func getProcessContext() (SELinuxContext, error) {
	data, err := os.ReadFile("/proc/self/attr/current")
	if err != nil {
		return SELinuxContext{}, fmt.Errorf("read process context: %w", err)
	}
	return parseContext(string(data)), nil
}

// getFileContext reads a file's SELinux context from its extended
// attributes. SELinux stores the label in the "security.selinux"
// xattr, which is set when the file is created (based on the parent
	// directory's context and the policy) and can be changed with chcon
// or restorecon.
//
// We use unix.Lgetxattr (not Getxattr) to avoid following symlinks —
// the symlink itself may have a different context than its target.
func getFileContext(path string) (SELinuxContext, error) {
	// First call with nil buffer to get the required buffer size.
	// This is the standard pattern for xattr operations.
	size, err := unix.Lgetxattr(path, "security.selinux", nil)
	if err != nil {
		return SELinuxContext{}, fmt.Errorf("lgetxattr %s: %w", path, err)
	}

	buf := make([]byte, size)
	_, err = unix.Lgetxattr(path, "security.selinux", buf)
	if err != nil {
		return SELinuxContext{}, fmt.Errorf("lgetxattr %s: %w", path, err)
	}

	return parseContext(string(buf)), nil
}

// isSELinuxEnabled checks if SELinux is enabled by reading
// /sys/fs/selinux/enforce. If this file exists and is readable,
// SELinux is enabled. The content is "1" for enforcing or "0"
// for permissive mode.
func isSELinuxEnabled() (enabled bool, enforcing bool) {
	data, err := os.ReadFile("/sys/fs/selinux/enforce")
	if err != nil {
		return false, false // SELinux not available
	}
	return true, strings.TrimSpace(string(data)) == "1"
}

func main() {
	fmt.Println("=== SELinux Context Inspector ===")

	// Check SELinux status
	enabled, enforcing := isSELinuxEnabled()
	if !enabled {
		fmt.Println("SELinux: DISABLED or not available")
		fmt.Println("  On RHEL/Fedora, enable with: sudo setenforce 1")
		fmt.Println("  Permanent: edit /etc/selinux/config → SELINUX=enforcing")
		return
	}

	if enforcing {
		fmt.Println("SELinux: ENFORCING")
	} else {
		fmt.Println("SELinux: PERMISSIVE (logging only)")
	}

	// Display process context
	pctx, err := getProcessContext()
	if err != nil {
		fmt.Printf("Process context error: %v\n", err)
	} else {
		fmt.Printf("\nProcess context: %s\n", pctx.Raw)
		fmt.Printf("  User:  %s\n", pctx.User)
		fmt.Printf("  Role:  %s\n", pctx.Role)
		fmt.Printf("  Type:  %s (this is the 'domain')\n", pctx.Type)
		fmt.Printf("  Level: %s\n", pctx.Level)
	}

	// Display file contexts for command-line arguments
	paths := []string{"/etc/passwd", "/etc/shadow"}
	if len(os.Args) > 1 {
		paths = os.Args[1:]
	}

	fmt.Println()
	for _, path := range paths {
		fctx, err := getFileContext(path)
		if err != nil {
			fmt.Printf("%-30s ERROR: %v\n", path, err)
			continue
		}

		fmt.Printf("%-30s %s\n", path, fctx.Raw)
		fmt.Printf("  Type: %s", fctx.Type)

		// Highlight interesting types
		switch fctx.Type {
		case "shadow_t":
			fmt.Printf(" ← SENSITIVE: password hashes")
		case "httpd_sys_content_t":
			fmt.Printf(" ← web server content")
		case "container_file_t":
			fmt.Printf(" ← container-managed file")
		}
		fmt.Println()
	}
}
```


### 7.5.6 Hands-On: Creating a Custom SELinux Policy for a Go Service

Creating a custom SELinux policy module involves writing type enforcement
(`.te`), file context (`.fc`), and interface (`.if`) files:

```bash
# ──────────────────────────────────────────────────────
# Step 1: Create the Type Enforcement policy (.te file)
# ──────────────────────────────────────────────────────
cat > mygoserver.te << 'EOF'
# mygoserver.te — SELinux Type Enforcement policy for our Go server
#
# This policy creates a new domain (mygoserver_t) for our Go server
# and defines exactly what it can access. Any access not explicitly
# allowed here is DENIED — this is the power of MAC.

# Policy module declaration
# The version number should be incremented with each policy change.
policy_module(mygoserver, 1.0.0)

# ────── Type declarations ──────

# Declare the domain (process type) for our server.
# gen_require pulls in types defined by the base policy.
type mygoserver_t;
type mygoserver_exec_t;

# Declare types for our server's files.
type mygoserver_conf_t;    # Configuration files
type mygoserver_log_t;     # Log files
type mygoserver_data_t;    # Runtime data

# ────── Domain transition ──────

# When a process in initrc_t (init/systemd) executes a file labeled
# mygoserver_exec_t, transition to the mygoserver_t domain.
# This is how SELinux ensures our server always runs in its confined domain.
init_daemon_domain(mygoserver_t, mygoserver_exec_t)

# ────── Permissions ──────

# Allow the server to read its own configuration
allow mygoserver_t mygoserver_conf_t:file { read open getattr };
allow mygoserver_t mygoserver_conf_t:dir  { search open getattr };

# Allow the server to write logs
allow mygoserver_t mygoserver_log_t:file { create write append open getattr setattr };
allow mygoserver_t mygoserver_log_t:dir  { search write add_name open getattr };

# Allow the server to read/write runtime data
allow mygoserver_t mygoserver_data_t:file { create read write open getattr setattr unlink };
allow mygoserver_t mygoserver_data_t:dir  { search write add_name remove_name open getattr };

# Network access — bind to HTTP port and make outbound connections
allow mygoserver_t self:tcp_socket { create listen accept bind connect getopt setopt shutdown };
allow mygoserver_t self:udp_socket { create connect getopt setopt };

# Allow binding to HTTP ports (80, 443)
# corenet_tcp_bind_http_port is a macro that expands to the proper
# allow rule for http_port_t.
corenet_tcp_bind_http_port(mygoserver_t)
corenet_tcp_connect_http_port(mygoserver_t)

# DNS resolution
sysnet_dns_name_resolve(mygoserver_t)

# Read system certificates (for TLS)
miscfiles_read_generic_certs(mygoserver_t)

# Read /etc/passwd and /etc/group (for user lookup)
auth_read_passwd(mygoserver_t)

# Use /dev/urandom for crypto random (Go runtime)
dev_read_urand(mygoserver_t)

# Logging to syslog
logging_send_syslog_msg(mygoserver_t)

# Allow the Go runtime to manage its threads and signals
allow mygoserver_t self:process { signal sigchld fork getsched setsched };

# Temporary file access
files_tmp_filetrans(mygoserver_t, mygoserver_data_t, file)
EOF

# ──────────────────────────────────────────────────────
# Step 2: Create the File Context policy (.fc file)
# ──────────────────────────────────────────────────────
cat > mygoserver.fc << 'EOF'
# mygoserver.fc — File context definitions
#
# This file tells SELinux which labels to apply to our server's files.
# These labels are applied by restorecon and during file creation.

# The server binary
/usr/bin/mygoserver    --    gen_context(system_u:object_r:mygoserver_exec_t,s0)

# Configuration directory and files
/etc/mygoserver(/.*)?         gen_context(system_u:object_r:mygoserver_conf_t,s0)

# Log directory and files
/var/log/mygoserver(/.*)?     gen_context(system_u:object_r:mygoserver_log_t,s0)

# Runtime data directory
/var/lib/mygoserver(/.*)?     gen_context(system_u:object_r:mygoserver_data_t,s0)
EOF

# ──────────────────────────────────────────────────────
# Step 3: Compile and install the policy
# ──────────────────────────────────────────────────────

# Compile the policy module
make -f /usr/share/selinux/devel/Makefile mygoserver.pp

# Install the policy module
sudo semodule -i mygoserver.pp

# Apply file contexts
sudo restorecon -Rv /usr/bin/mygoserver
sudo restorecon -Rv /etc/mygoserver/
sudo restorecon -Rv /var/log/mygoserver/
sudo restorecon -Rv /var/lib/mygoserver/

# Verify the labels
ls -Z /usr/bin/mygoserver
# system_u:object_r:mygoserver_exec_t:s0 /usr/bin/mygoserver
```


### 7.5.7 SELinux in Kubernetes

Kubernetes supports SELinux through the `seLinuxOptions` field in the
security context:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: selinux-pod
spec:
  securityContext:
    seLinuxOptions:
      # These labels are applied to the container process.
      # The container runtime (CRI-O, containerd) handles the
      # actual label application.
      level: "s0:c123,c456"  # MCS categories for isolation
  containers:
  - name: app
    image: mygoserver:latest
    securityContext:
      seLinuxOptions:
        # Type determines which SELinux domain the container runs in.
        # container_t — standard container (default in RHEL/Fedora)
        # spc_t — "super privileged container" (no SELinux restrictions)
        type: "container_t"
        level: "s0:c123,c456"
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

**How container runtimes use SELinux MCS:**

```
┌──────────────────────────────────────────────────────┐
│         SELINUX MCS CONTAINER ISOLATION               │
│                                                      │
│  Container A: system_u:system_r:container_t:s0:c1,c2 │
│  Container B: system_u:system_r:container_t:s0:c3,c4 │
│                                                      │
│  Both containers run as container_t (same type),     │
│  but have DIFFERENT MCS categories (c1,c2 vs c3,c4). │
│                                                      │
│  Result: Container A CANNOT access Container B's     │
│  files, even though they share the same type, because│
│  the MCS levels don't match.                         │
│                                                      │
│  This is how container runtimes isolate containers   │
│  from each other using SELinux on RHEL/Fedora.       │
└──────────────────────────────────────────────────────┘
```

---
## 7.6 Defense in Depth — Seccomp + Capabilities + AppArmor/SELinux Together

### 7.6.1 Layering Security Mechanisms

No single security mechanism is sufficient. Each layer catches different types
of attacks, and layering them creates a defense-in-depth strategy where an
attacker must bypass ALL layers to succeed:

```
┌──────────────────────────────────────────────────────────────────┐
│              DEFENSE IN DEPTH — LAYERED SECURITY                  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ Layer 4: AppArmor / SELinux (MAC)                          │  │
│  │   "This program can only access /etc/myapp/ and /var/log/" │  │
│  │   Even root cannot bypass this policy                      │  │
│  ├────────────────────────────────────────────────────────────┤  │
│  │ Layer 3: Seccomp-BPF (Syscall filtering)                   │  │
│  │   "This process can only use these 50 syscalls"            │  │
│  │   mount(), ptrace(), kexec_load() are impossible           │  │
│  ├────────────────────────────────────────────────────────────┤  │
│  │ Layer 2: Linux Capabilities                                │  │
│  │   "This process has only CAP_NET_BIND_SERVICE"             │  │
│  │   Cannot load modules, cannot mount, cannot chown          │  │
│  ├────────────────────────────────────────────────────────────┤  │
│  │ Layer 1: DAC (File permissions + non-root UID)             │  │
│  │   "Running as UID 1000, not root"                          │  │
│  │   Standard Unix permission checks apply                    │  │
│  ├────────────────────────────────────────────────────────────┤  │
│  │ Layer 0: User Namespaces + no-new-privileges               │  │
│  │   "Process is in its own user namespace"                   │  │
│  │   "Cannot gain privileges through exec"                    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  An attacker who exploits a vulnerability in the application     │
│  must bypass ALL layers. Each layer blocks different attack       │
│  vectors:                                                        │
│                                                                  │
│  DAC:      Prevents reading files owned by other users           │
│  Caps:     Prevents using root-like operations                   │
│  Seccomp:  Prevents calling dangerous syscalls entirely          │
│  MAC:      Prevents accessing paths/types not in the policy      │
│  Namespace: Prevents escalating privileges                       │
└──────────────────────────────────────────────────────────────────┘
```


### 7.6.2 What Docker/containerd Applies by Default

When you run a container with Docker (using containerd as the runtime), here's
what security is applied by default — without any configuration:

```
┌──────────────────────────────────────────────────────────────┐
│          DOCKER DEFAULT SECURITY SETTINGS                     │
│                                                              │
│  1. CAPABILITIES                                             │
│     Drops ~27 capabilities, keeps 14                         │
│     Notable drops: SYS_ADMIN, NET_ADMIN, SYS_MODULE,        │
│                    SYS_PTRACE, SYS_RAWIO                     │
│                                                              │
│  2. SECCOMP                                                  │
│     Default profile blocks ~44 dangerous syscalls            │
│     mount, umount, ptrace, kexec_load, reboot, etc.         │
│                                                              │
│  3. APPARMOR (on Ubuntu/SUSE)                                │
│     docker-default profile restricts:                        │
│     - Mount operations                                       │
│     - Write to /proc and /sys                               │
│     - Signal sending to other containers                    │
│     - ptrace                                                 │
│                                                              │
│  4. NAMESPACES                                               │
│     PID, NET, MNT, UTS, IPC namespaces                      │
│     (NOT user namespace by default)                          │
│                                                              │
│  5. CGROUPS                                                  │
│     Resource limits (CPU, memory, I/O)                       │
│                                                              │
│  6. READONLY ROOTFS                                          │
│     Not by default, but recommended (--read-only)            │
│                                                              │
│  7. /proc and /sys MASKING                                   │
│     Sensitive paths masked or read-only                      │
│                                                              │
│  8. NO-NEW-PRIVILEGES                                        │
│     Not set by default (but recommended)                     │
│     Enable with: --security-opt no-new-privileges            │
└──────────────────────────────────────────────────────────────┘
```

### 7.6.3 What Kubernetes Adds

Kubernetes adds Pod Security Standards on top of the container runtime's defaults:

```yaml
# Restricted Pod Security Standards (Kubernetes 1.25+)
# This is the "gold standard" for pod security
apiVersion: v1
kind: Pod
metadata:
  name: hardened-pod
spec:
  securityContext:
    # Run as non-root
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    
    # Seccomp
    seccompProfile:
      type: RuntimeDefault
    
    # SELinux (on RHEL nodes)
    seLinuxOptions:
      type: container_t
  
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      # Drop ALL capabilities — most restrictive setting
      capabilities:
        drop: ["ALL"]
      
      # Prevent privilege escalation
      allowPrivilegeEscalation: false
      
      # Read-only root filesystem
      readOnlyRootFilesystem: true
      
      # Don't run as root
      runAsNonRoot: true
      
      # AppArmor profile (on Ubuntu nodes)
      # (via annotation on older K8s versions)
      appArmorProfile:
        type: RuntimeDefault
    
    # Writable directories via tmpfs/volumes only
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache
  
  volumes:
  - name: tmp
    emptyDir:
      medium: Memory
      sizeLimit: 64Mi
  - name: cache
    emptyDir:
      sizeLimit: 128Mi
```

---

## 7.7 Namespaced Security

### 7.7.1 User Namespaces for Privilege Isolation

User namespaces allow a process to have different UID/GID mappings inside vs
outside the namespace. A process can be "root" (UID 0) inside the namespace
while mapping to an unprivileged user (e.g., UID 100000) on the host.

```
┌──────────────────────────────────────────────────────────┐
│               USER NAMESPACE MAPPING                      │
│                                                          │
│  Host System                Container (User Namespace)   │
│  ┌─────────────┐           ┌─────────────┐              │
│  │ UID 100000  │ ────────→ │ UID 0 (root)│              │
│  │ UID 100001  │ ────────→ │ UID 1       │              │
│  │ UID 100002  │ ────────→ │ UID 2       │              │
│  │ ...         │ ────────→ │ ...         │              │
│  │ UID 165535  │ ────────→ │ UID 65535   │              │
│  └─────────────┘           └─────────────┘              │
│                                                          │
│  Inside the container, the process thinks it's root.     │
│  On the host, it's UID 100000 — an unprivileged user.   │
│  If the container is compromised:                        │
│    - The attacker is "root" in the container             │
│    - But UID 100000 on the host — harmless!              │
│    - Cannot access other users' files on the host        │
│    - Cannot use privileged operations on the host        │
└──────────────────────────────────────────────────────────┘
```


### 7.7.2 The no-new-privileges Flag

The `PR_SET_NO_NEW_PRIVS` flag is a critical security feature that prevents a
process (and all its descendants) from gaining new privileges through
`execve(2)`. When set:

- **setuid/setgid bits are ignored** — exec'ing a setuid-root binary does NOT
  give root privileges
- **File capabilities are ignored** — exec'ing a binary with `setcap` caps
  does NOT grant those capabilities
- **The flag is inherited** — all child processes inherit no-new-privileges
- **Irreversible** — once set, it cannot be unset

This flag is required for non-root processes to install seccomp filters (without
it, a seccomp filter could be used to subvert setuid programs).

### 7.7.3 Hands-On: Using PR_SET_NO_NEW_PRIVS from Go

```go
// no_new_privs.go
//
// This program demonstrates the PR_SET_NO_NEW_PRIVS flag, which is
// one of the most important security primitives for containers and
// sandboxed applications.
//
// When no-new-privileges is set:
// 1. execve() will NOT honor setuid/setgid bits
// 2. execve() will NOT apply file capabilities
// 3. The flag is inherited by all child processes
// 4. The flag cannot be unset (irreversible for the process tree)
//
// This prevents privilege escalation attacks where a confined process
// exec's a setuid binary to escape its sandbox.
//
// Container runtimes should ALWAYS set this flag. Kubernetes exposes
// it as: securityContext.allowPrivilegeEscalation: false
//
// Usage:
//   go run no_new_privs.go
//   # Then try: exec'ing /usr/bin/passwd → it won't gain root
package main

import (
	"fmt"
	"log"
	"os"
	"os/exec"

	"golang.org/x/sys/unix"
)

// getNoNewPrivs checks whether PR_SET_NO_NEW_PRIVS is currently set
// for this thread using prctl(PR_GET_NO_NEW_PRIVS).
// Returns true if the flag is set (no privilege escalation possible).
func getNoNewPrivs() (bool, error) {
	// prctl(PR_GET_NO_NEW_PRIVS, 0, 0, 0, 0) returns:
	//   1 if no-new-privs is set
	//   0 if not set
	//   -1 on error (shouldn't happen for this call)
	val, err := unix.PrctlRetInt(unix.PR_GET_NO_NEW_PRIVS, 0, 0, 0, 0)
	if err != nil {
		return false, fmt.Errorf("prctl(PR_GET_NO_NEW_PRIVS): %w", err)
	}
	return val != 0, nil
}

// setNoNewPrivs enables the no-new-privileges flag for this process.
// This is IRREVERSIBLE — once set, it cannot be cleared, and all
// child processes will inherit it.
//
// This is the Go equivalent of:
//   prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0)
//
// In Docker, this is enabled with:
//   docker run --security-opt no-new-privileges myimage
//
// In Kubernetes:
//   securityContext:
//     allowPrivilegeEscalation: false
func setNoNewPrivs() error {
	return unix.Prctl(unix.PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0)
}

func main() {
	fmt.Println("=== PR_SET_NO_NEW_PRIVS Demo ===")
	fmt.Printf("PID: %d, UID: %d\n\n", os.Getpid(), os.Getuid())

	// Check current status
	nnp, err := getNoNewPrivs()
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("no-new-privileges before: %v\n", nnp)

	// Set the flag
	fmt.Println("\nSetting PR_SET_NO_NEW_PRIVS...")
	if err := setNoNewPrivs(); err != nil {
		log.Fatalf("Failed to set no-new-privs: %v", err)
	}

	// Verify it's set
	nnp, _ = getNoNewPrivs()
	fmt.Printf("no-new-privileges after:  %v\n\n", nnp)

	// Demonstrate the effect: try to exec a setuid binary
	// With no-new-privs set, the setuid bit is IGNORED.
	fmt.Println("Attempting to exec /usr/bin/passwd (setuid root)...")
	fmt.Println("With no-new-privs, the setuid bit will be IGNORED.")
	fmt.Println("The process will run as our UID, not as root.\n")

	// In a real scenario, you'd exec the binary and observe that
	// it runs as the current UID instead of root.
	cmd := exec.Command("id")
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Run(); err != nil {
		fmt.Printf("exec failed: %v\n", err)
	}

	fmt.Println("\n✓ no-new-privileges is set")
	fmt.Println("  Effects:")
	fmt.Println("  - setuid/setgid bits ignored on exec")
	fmt.Println("  - File capabilities ignored on exec")
	fmt.Println("  - Seccomp filters can be installed without CAP_SYS_ADMIN")
	fmt.Println("  - All child processes inherit this restriction")
	fmt.Println("  - This flag CANNOT be unset (irreversible)")
}
```

---

## 7.8 Linux Audit Framework

### 7.8.1 auditd, auditctl, ausearch

The Linux Audit Framework provides comprehensive system call monitoring.
It's the kernel's built-in intrusion detection system — every syscall can
be logged with full context.

```
┌──────────────────────────────────────────────────────────────┐
│               LINUX AUDIT ARCHITECTURE                        │
│                                                              │
│  ┌──────────────┐     ┌──────────────┐                      │
│  │   Process     │     │   auditd     │                      │
│  │  makes a      │     │  (daemon)    │                      │
│  │  syscall      │     │              │                      │
│  └──────┬───────┘     │  Receives    │                      │
│         │             │  audit events│                      │
│         ▼             │  from kernel │                      │
│  ┌──────────────┐     │              │                      │
│  │ Kernel Audit │     │  Writes to   │                      │
│  │  Subsystem   │────▶│  /var/log/   │                      │
│  │              │     │  audit/      │                      │
│  │  Checks      │     │  audit.log   │                      │
│  │  audit rules │     └──────────────┘                      │
│  │  against     │                                            │
│  │  syscall     │     ┌──────────────┐                      │
│  └──────────────┘     │  ausearch    │                      │
│                       │  aureport    │                      │
│  ┌──────────────┐     │  (query      │                      │
│  │  auditctl    │     │   tools)     │                      │
│  │  (manage     │     └──────────────┘                      │
│  │   rules)     │                                            │
│  └──────────────┘                                            │
└──────────────────────────────────────────────────────────────┘
```

Key tools:

| Tool | Purpose |
|------|---------|
| `auditd` | Daemon that receives and logs audit events |
| `auditctl` | Add/remove/list audit rules at runtime |
| `ausearch` | Search audit logs by various criteria |
| `aureport` | Generate summary reports from audit logs |
| `autrace` | Trace a single program (like strace but via audit) |


### 7.8.2 Audit Rules for Syscall Monitoring

Audit rules define which events to log. There are three types:

**1. File/directory watches (simplest):**
```bash
# Watch /etc/shadow for any access
sudo auditctl -w /etc/shadow -p rwxa -k shadow_access
#  -w  = watch this path
#  -p  = permissions to watch: r(ead), w(rite), e(x)ecute, (a)ttribute change
#  -k  = key (tag) for searching later

# Watch /etc/passwd for modifications
sudo auditctl -w /etc/passwd -p wa -k passwd_changes

# Watch the Docker socket
sudo auditctl -w /var/run/docker.sock -p rwxa -k docker_socket

# Watch sudoers file
sudo auditctl -w /etc/sudoers -p wa -k sudoers_changes
```

**2. Syscall rules (most powerful):**
```bash
# Log all mount operations
sudo auditctl -a always,exit -F arch=b64 -S mount -S umount2 -k mounts
#  -a always,exit = audit on syscall exit
#  -F arch=b64    = 64-bit syscalls only
#  -S mount       = the mount syscall
#  -k mounts      = search key

# Log all execve calls (command execution)
sudo auditctl -a always,exit -F arch=b64 -S execve -k commands

# Log privilege escalation attempts
sudo auditctl -a always,exit -F arch=b64 -S setuid -S setgid \
  -S setreuid -S setregid -k privilege_escalation

# Log module loading
sudo auditctl -a always,exit -F arch=b64 -S init_module \
  -S finit_module -S delete_module -k kernel_modules

# Log ptrace (process debugging/injection)
sudo auditctl -a always,exit -F arch=b64 -S ptrace -k ptrace_attempt

# Log network socket creation by non-root users
sudo auditctl -a always,exit -F arch=b64 -S socket -F a0=2 \
  -F euid!=0 -k user_network
```

**3. Searching audit logs:**
```bash
# Search by key
sudo ausearch -k shadow_access --start today

# Search by syscall
sudo ausearch -sc mount --start recent

# Search by process name
sudo ausearch -c mygoserver --start today

# Search for failed operations (potential attacks)
sudo ausearch --success no --start today

# Generate a summary report
sudo aureport --summary

# Report on all command executions
sudo aureport -x --summary
```


### 7.8.3 Hands-On: Setting Up Audit Rules for Sensitive Operations

```go
// audit_setup.go
//
// This program demonstrates how to programmatically interact with
// the Linux audit system from Go. While audit rules are typically
// managed via auditctl, understanding the underlying netlink protocol
// is valuable for building security monitoring tools.
//
// The Linux audit system uses a netlink socket (NETLINK_AUDIT) for
// communication between userspace and the kernel audit subsystem.
//
// This program reads the current audit status and displays it.
// For a production system, you'd use the go-libaudit library for
// full audit rule management.
//
// Usage:
//   sudo go run audit_setup.go
package main

import (
	"encoding/binary"
	"fmt"
	"log"
	"os"
	"os/exec"
	"strings"
)

// auditRule represents a single audit rule that we want to install.
// We use auditctl for actual rule installation since the netlink
// protocol for audit rules is complex and fragile.
type auditRule struct {
	// Description explains what this rule monitors and why.
	Description string

	// Args are the arguments to pass to auditctl.
	// For example: ["-w", "/etc/shadow", "-p", "rwxa", "-k", "shadow"]
	Args []string
}

// securityAuditRules returns a comprehensive set of audit rules
// for monitoring security-sensitive operations on a Linux system.
// These rules are based on CIS benchmarks and common security
// monitoring best practices.
//
// Categories:
// - File integrity monitoring (sensitive config files)
// - Privilege escalation detection
// - Kernel module monitoring
// - Network socket monitoring
// - Container-specific monitoring
func securityAuditRules() []auditRule {
	return []auditRule{
		// ── File integrity monitoring ──
		{
			Description: "Monitor shadow password file access",
			Args:        []string{"-w", "/etc/shadow", "-p", "rwxa", "-k", "shadow_access"},
		},
		{
			Description: "Monitor passwd file changes",
			Args:        []string{"-w", "/etc/passwd", "-p", "wa", "-k", "passwd_changes"},
		},
		{
			Description: "Monitor group file changes",
			Args:        []string{"-w", "/etc/group", "-p", "wa", "-k", "group_changes"},
		},
		{
			Description: "Monitor sudoers changes",
			Args:        []string{"-w", "/etc/sudoers", "-p", "wa", "-k", "sudoers_changes"},
		},
		{
			Description: "Monitor SSH configuration changes",
			Args:        []string{"-w", "/etc/ssh/sshd_config", "-p", "wa", "-k", "ssh_config"},
		},

		// ── Privilege escalation detection ──
		{
			Description: "Monitor setuid/setgid calls (privilege changes)",
			Args: []string{"-a", "always,exit", "-F", "arch=b64",
				"-S", "setuid", "-S", "setgid",
				"-S", "setreuid", "-S", "setregid",
				"-k", "privilege_escalation"},
		},

		// ── Kernel module monitoring ──
		{
			Description: "Monitor kernel module operations",
			Args: []string{"-a", "always,exit", "-F", "arch=b64",
				"-S", "init_module", "-S", "finit_module",
				"-S", "delete_module",
				"-k", "kernel_modules"},
		},

		// ── Mount monitoring ──
		{
			Description: "Monitor filesystem mount/unmount",
			Args: []string{"-a", "always,exit", "-F", "arch=b64",
				"-S", "mount", "-S", "umount2",
				"-k", "filesystem_mounts"},
		},

		// ── Process tracing ──
		{
			Description: "Monitor ptrace (debugging/injection attempts)",
			Args: []string{"-a", "always,exit", "-F", "arch=b64",
				"-S", "ptrace",
				"-k", "ptrace_attempts"},
		},

		// ── Container-specific ──
		{
			Description: "Monitor Docker socket access",
			Args: []string{"-w", "/var/run/docker.sock", "-p", "rwxa", "-k", "docker_socket"},
		},
		{
			Description: "Monitor containerd socket access",
			Args: []string{"-w", "/run/containerd/containerd.sock", "-p", "rwxa", "-k", "containerd_socket"},
		},
	}
}

// installAuditRules installs the given audit rules using auditctl.
// Returns the number of successfully installed rules and any errors.
func installAuditRules(rules []auditRule) (installed int, errors []string) {
	for _, rule := range rules {
		args := append([]string{}, rule.Args...)
		cmd := exec.Command("auditctl", args...)
		output, err := cmd.CombinedOutput()
		if err != nil {
			errors = append(errors,
				fmt.Sprintf("  ✗ %s: %v (%s)",
					rule.Description, err, strings.TrimSpace(string(output))))
		} else {
			installed++
			fmt.Printf("  ✓ %s\n", rule.Description)
		}
	}
	return installed, errors
}

func main() {
	fmt.Println("=== Linux Audit Framework Setup ===")

	// Check if we're root (required for auditctl)
	if os.Getuid() != 0 {
		log.Fatal("This program must be run as root (sudo)")
	}

	// Check if auditd is running
	if _, err := exec.LookPath("auditctl"); err != nil {
		log.Fatal("auditctl not found — install auditd: sudo apt install auditd")
	}

	// Display current audit status
	fmt.Println("\n── Current audit status ──")
	cmd := exec.Command("auditctl", "-s")
	cmd.Stdout = os.Stdout
	cmd.Run()

	// List current rules
	fmt.Println("\n── Current audit rules ──")
	cmd = exec.Command("auditctl", "-l")
	cmd.Stdout = os.Stdout
	cmd.Run()

	// Install security audit rules
	rules := securityAuditRules()
	fmt.Printf("\n── Installing %d audit rules ──\n", len(rules))
	installed, errs := installAuditRules(rules)

	fmt.Printf("\n── Summary ──\n")
	fmt.Printf("Installed: %d/%d rules\n", installed, len(rules))
	if len(errs) > 0 {
		fmt.Printf("Errors (%d):\n", len(errs))
		for _, e := range errs {
			fmt.Println(e)
		}
	}

	fmt.Println("\n── Verify with ──")
	fmt.Println("  sudo auditctl -l                    # List all rules")
	fmt.Println("  sudo ausearch -k shadow_access       # Search by key")
	fmt.Println("  sudo aureport --summary              # Summary report")

	// Suppress unused import warning
	_ = binary.LittleEndian
}
```

---

## 7.9 /dev, /proc, /sys Security

### 7.9.1 Why These Must Be Restricted in Containers

The `/dev`, `/proc`, and `/sys` pseudo-filesystems expose kernel internals
to userspace. In a container context, unrestricted access to these filesystems
can lead to container escapes and host compromise:

```
┌──────────────────────────────────────────────────────────────┐
│      DANGEROUS PATHS IN /proc AND /sys                        │
│                                                              │
│  /proc/sysrq-trigger                                         │
│    Writing to this can reboot the host, crash it,            │
│    or dump memory. MUST be masked in containers.             │
│                                                              │
│  /proc/kcore                                                 │
│    Physical memory of the host. Reading this leaks           │
│    ALL host memory including secrets and keys.               │
│                                                              │
│  /proc/kmsg                                                  │
│    Kernel message buffer. Can leak host information.         │
│                                                              │
│  /proc/kallsyms                                              │
│    Kernel symbol addresses — useful for building             │
│    kernel exploits.                                          │
│                                                              │
│  /sys/firmware/efi/efivars                                   │
│    Writing here can BRICK the hardware (yes, really).        │
│                                                              │
│  /sys/kernel/debug                                           │
│    Kernel debug interfaces — powerful and dangerous.         │
│                                                              │
│  /sys/fs/cgroup (writable)                                   │
│    If writable, can modify resource limits or escape.        │
│                                                              │
│  /dev/mem, /dev/kmem                                         │
│    Direct physical/kernel memory access.                     │
│    Complete host compromise.                                 │
│                                                              │
│  /dev/sd*, /dev/nvme*                                        │
│    Raw block device access — read any disk on the host.      │
└──────────────────────────────────────────────────────────────┘
```


### 7.9.2 Masking Sensitive Paths

Container runtimes "mask" dangerous paths by bind-mounting `/dev/null` or
a read-only tmpfs over them, making them inaccessible:

```bash
# How Docker masks /proc paths (using bind mounts):
mount --bind /dev/null /proc/kcore
mount --bind /dev/null /proc/sysrq-trigger
mount --bind /dev/null /proc/timer_list
mount --bind /dev/null /proc/timer_stats
mount --bind /dev/null /proc/sched_debug
mount --bind /dev/null /proc/scsi

# How Docker makes /proc paths read-only:
mount -o bind,ro /proc/bus
mount -o bind,ro /proc/fs
mount -o bind,ro /proc/irq
mount -o bind,ro /proc/sys
mount -o bind,ro /proc/sysrq-trigger

# How Docker restricts /sys:
mount -o bind,ro /sys
```


### 7.9.3 Hands-On: Examining What Docker Masks in /proc

```go
// proc_masking.go
//
// This program inspects the /proc filesystem to determine which paths
// have been masked or made read-only by the container runtime. This is
// a diagnostic tool for verifying that your container's /proc is properly
// restricted.
//
// When running inside a Docker container, you'll see that many /proc
// paths are masked (inaccessible) or read-only. When running on a
// regular host, all paths will be accessible.
//
// How masking works:
// Docker bind-mounts /dev/null over sensitive /proc files, making them
// appear to be empty. It bind-mounts a read-only tmpfs over /proc
// directories that should not be writable.
//
// Usage:
//   # On host (everything accessible):
//   go run proc_masking.go
//
//   # In Docker (paths masked):
//   docker run --rm -v $(pwd):/app -w /app golang:1.22 go run proc_masking.go
package main

import (
	"fmt"
	"os"
	"strings"
	"syscall"
)

// procPath represents a sensitive /proc or /sys path that should
// be restricted in containers. We check both readability and
// writability to determine the level of restriction.
type procPath struct {
	Path        string // Filesystem path to check
	Description string // What this path exposes
	Danger      string // Why this is dangerous in containers
}

// sensitivePaths returns the list of /proc and /sys paths that
// container runtimes should mask or make read-only.
// This list is based on Docker's default masking behavior and
// OCI runtime spec recommendations.
func sensitivePaths() []procPath {
	return []procPath{
		{
			Path:        "/proc/kcore",
			Description: "Physical memory (ELF core format)",
			Danger:      "Leaks ALL host memory — passwords, keys, data",
		},
		{
			Path:        "/proc/sysrq-trigger",
			Description: "Magic SysRq key interface",
			Danger:      "Can reboot, crash, or memory-dump the host",
		},
		{
			Path:        "/proc/kmsg",
			Description: "Kernel message buffer",
			Danger:      "Leaks kernel log messages from the host",
		},
		{
			Path:        "/proc/kallsyms",
			Description: "Kernel symbol table with addresses",
			Danger:      "Provides addresses for kernel exploits (KASLR bypass)",
		},
		{
			Path:        "/proc/timer_list",
			Description: "Active kernel timers",
			Danger:      "Leaks host timing information",
		},
		{
			Path:        "/proc/sched_debug",
			Description: "Scheduler debug info for ALL processes",
			Danger:      "Reveals all host processes (escapes PID namespace)",
		},
		{
			Path:        "/proc/sys",
			Description: "Kernel tunable parameters",
			Danger:      "Writable access can modify host kernel settings",
		},
		{
			Path:        "/proc/bus",
			Description: "Bus information (PCI, USB, etc.)",
			Danger:      "Reveals host hardware topology",
		},
		{
			Path:        "/proc/irq",
			Description: "Interrupt request information",
			Danger:      "Reveals host interrupt configuration",
		},
		{
			Path:        "/proc/acpi",
			Description: "ACPI power management",
			Danger:      "Can control host power state",
		},
		{
			Path:        "/sys/firmware",
			Description: "System firmware interfaces",
			Danger:      "Writing to EFI vars can brick the machine",
		},
		{
			Path:        "/sys/kernel/debug",
			Description: "Kernel debug filesystem",
			Danger:      "Powerful debugging interfaces — information leak",
		},
		{
			Path:        "/sys/fs/cgroup",
			Description: "Cgroup filesystem",
			Danger:      "Writable access enables cgroup escape attacks",
		},
	}
}

// checkPath examines a /proc or /sys path to determine its
// accessibility status. It performs three checks:
//   1. Can we stat the path? (Does it exist / is it masked?)
//   2. Can we read the path? (Is it readable?)
//   3. Can we write to the path? (Is it writable? — most dangerous)
//
// Returns a status string: MASKED, READ-ONLY, ACCESSIBLE, or ERROR.
func checkPath(path string) (status string, detail string) {
	// Check if the path exists at all
	fi, err := os.Stat(path)
	if err != nil {
		if os.IsNotExist(err) {
			return "NOT FOUND", "Path does not exist"
		}
		if os.IsPermission(err) {
			return "DENIED", "Permission denied (good — restricted)"
		}
		return "ERROR", err.Error()
	}

	// For files: try to read them
	if !fi.IsDir() {
		f, err := os.Open(path)
		if err != nil {
			if os.IsPermission(err) {
				return "DENIED", "Cannot open for reading"
			}
			return "MASKED", "Open failed (likely masked with /dev/null bind mount)"
		}
		defer f.Close()

		// Read a small amount to check if it's a /dev/null bind mount
		buf := make([]byte, 1)
		n, err := f.Read(buf)
		if n == 0 && err != nil {
			return "MASKED", "Empty read (bind-mounted from /dev/null)"
		}
		// Check if the file is writable
		if canWrite(path) {
			return "⚠ WRITABLE", "Readable AND writable — DANGEROUS"
		}
		return "READABLE", fmt.Sprintf("Size: %d bytes", fi.Size())
	}

	// For directories: check if writable
	if canWrite(path) {
		return "⚠ WRITABLE", "Directory is writable"
	}

	return "READ-ONLY", "Directory exists, read-only"
}

// canWrite checks if a path is writable by attempting to open it
// with O_WRONLY. We do NOT actually write anything.
// This uses syscall.Access which checks based on the real UID/GID.
func canWrite(path string) bool {
	return syscall.Access(path, syscall.O_WRONLY) == nil
}

// isInContainer performs a best-effort check to determine if we're
// running inside a container. We check several indicators:
// 1. /.dockerenv file exists (Docker-specific)
// 2. /proc/1/cgroup contains container-related strings
// 3. /proc/self/mountinfo contains overlay filesystem
func isInContainer() bool {
	// Docker creates this file in every container
	if _, err := os.Stat("/.dockerenv"); err == nil {
		return true
	}

	// Check cgroup for container indicators
	data, err := os.ReadFile("/proc/1/cgroup")
	if err == nil {
		content := string(data)
		if strings.Contains(content, "docker") ||
		strings.Contains(content, "containerd") ||
		strings.Contains(content, "kubepods") ||
		strings.Contains(content, "lxc") {
			return true
		}
	}

	// Check mount info for overlay filesystem (common in containers)
	data, err = os.ReadFile("/proc/self/mountinfo")
	if err == nil {
		if strings.Contains(string(data), "overlay") {
			return true
		}
	}

	return false
}

func main() {
	fmt.Println("=== /proc and /sys Security Audit ===")
	fmt.Printf("PID: %d, UID: %d\n", os.Getpid(), os.Getuid())

	if isInContainer() {
		fmt.Println("Environment: CONTAINER (some paths should be masked)")
	} else {
		fmt.Println("Environment: HOST (all paths likely accessible)")
		fmt.Println("  Run in Docker to see masking in action:")
		fmt.Println("  docker run --rm -v $(pwd):/app -w /app golang:1.22 go run proc_masking.go")
	}

	fmt.Println()
	paths := sensitivePaths()

	// Track statistics
	var masked, readable, writable, denied int

	for _, p := range paths {
		status, detail := checkPath(p.Path)

		// Color-code the status for terminal output
		var indicator string
		switch status {
		case "MASKED":
			indicator = "✓ MASKED    "
			masked++
		case "READ-ONLY":
			indicator = "~ READ-ONLY "
			readable++
		case "READABLE":
			indicator = "~ READABLE  "
			readable++
		case "⚠ WRITABLE":
			indicator = "✗ WRITABLE  "
			writable++
		case "DENIED":
			indicator = "✓ DENIED    "
			denied++
		default:
			indicator = "? " + status + " "
		}

		fmt.Printf("%s %-30s %s\n", indicator, p.Path, detail)
		if status == "⚠ WRITABLE" {
			fmt.Printf("             ⚠ DANGER: %s\n", p.Danger)
		}
	}

	fmt.Printf("\n── Summary ──\n")
	fmt.Printf("Masked/Denied: %d (good — restricted)\n", masked+denied)
	fmt.Printf("Read-only:     %d (acceptable for most paths)\n", readable)
	fmt.Printf("Writable:      %d", writable)
	if writable > 0 {
		fmt.Printf(" ⚠ SECURITY CONCERN — these should be restricted!")
	}
	fmt.Println()

	if writable > 0 && isInContainer() {
		fmt.Println("\n⚠ WARNING: Some sensitive paths are writable in this container!")
		fmt.Println("  This may indicate --privileged mode or a misconfigured runtime.")
		fmt.Println("  Review your container security settings.")
	}
}
```

---

## 7.10 Summary and Security Checklist

This chapter covered the full depth of Linux security mechanisms. Here is a
concise checklist for securing a Go application on Linux:

```
┌──────────────────────────────────────────────────────────────┐
│           LINUX SECURITY CHECKLIST FOR GO SERVICES            │
│                                                              │
│  □ DAC                                                       │
│    □ Run as non-root user (UID != 0)                        │
│    □ Minimize file permissions (no world-writable)          │
│    □ No setuid binaries unless absolutely necessary         │
│                                                              │
│  □ CAPABILITIES                                              │
│    □ Drop ALL capabilities except those needed              │
│    □ Use file capabilities (setcap) instead of setuid       │
│    □ Document which capabilities your service requires      │
│    □ Drop bounding set to prevent future escalation         │
│                                                              │
│  □ SECCOMP                                                   │
│    □ Apply a seccomp profile (whitelist preferred)          │
│    □ Block: mount, ptrace, kexec, module loading            │
│    □ Test in LOG mode before switching to KILL              │
│    □ Use TSYNC flag for multi-threaded Go programs          │
│                                                              │
│  □ APPARMOR / SELINUX                                        │
│    □ Create a custom profile/policy for your service        │
│    □ Test in complain/permissive mode first                 │
│    □ Switch to enforce mode for production                  │
│    □ Deny access to sensitive paths explicitly              │
│                                                              │
│  □ NAMESPACES                                                │
│    □ Use user namespaces for rootless operation             │
│    □ Set PR_SET_NO_NEW_PRIVS                                │
│    □ Isolate PID, NET, MNT, IPC namespaces                 │
│                                                              │
│  □ AUDIT                                                     │
│    □ Monitor sensitive file access                          │
│    □ Log privilege escalation attempts                      │
│    □ Monitor kernel module operations                       │
│    □ Set up alerts for security-relevant events             │
│                                                              │
│  □ CONTAINER-SPECIFIC                                        │
│    □ Mask /proc and /sys sensitive paths                    │
│    □ Read-only root filesystem                              │
│    □ No --privileged containers                             │
│    □ Use --cap-drop=ALL --cap-add=<needed>                  │
│    □ Use seccomp=RuntimeDefault at minimum                  │
│    □ Set allowPrivilegeEscalation: false in K8s             │
│                                                              │
│  REMEMBER: Security is defense in depth.                     │
│  Each layer catches what the others miss.                    │
└──────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **DAC is necessary but not sufficient.** File permissions are the foundation,
   but they can be bypassed by root. MAC (AppArmor/SELinux) provides
   restrictions that even root cannot override.

2. **Capabilities replace root.** Instead of running as root, use capabilities
   to grant only the specific privileges your service needs. `CAP_NET_BIND_SERVICE`
   is the most common example.

3. **Seccomp reduces the attack surface.** By filtering which syscalls a process
   can make, seccomp prevents exploitation of kernel vulnerabilities in syscalls
   your application never uses.

4. **AppArmor and SELinux enforce mandatory policies.** Choose AppArmor for
   path-based simplicity (Ubuntu/SUSE) or SELinux for label-based power
   (RHEL/Fedora). Both provide MAC that constrains even root.

5. **Layer everything.** The combination of DAC + capabilities + seccomp + MAC
   + namespaces creates a defense-in-depth posture that is extremely difficult
   to bypass.

6. **Containers are not magic.** Docker and Kubernetes apply reasonable defaults,
   but you should explicitly configure capabilities, seccomp, and MAC profiles
   for your specific workloads.

7. **Audit for visibility.** The Linux audit framework gives you visibility into
   what processes are doing. Without audit logs, you're flying blind.

In the next chapter, we'll build on this security foundation to explore
**container networking** — how Linux namespaces and virtual networking combine
to create isolated network environments for containers.

---

*Further reading:*
- `man 7 capabilities` — Complete capability documentation
- `man 2 seccomp` — Seccomp system call documentation
- `man 5 apparmor.d` — AppArmor profile syntax reference
- Red Hat SELinux documentation — Comprehensive SELinux guide
- OCI Runtime Specification — What container runtimes must enforce
- Kubernetes Pod Security Standards — Pod hardening guidelines