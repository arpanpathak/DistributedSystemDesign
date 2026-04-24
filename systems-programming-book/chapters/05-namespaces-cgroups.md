# Chapter 5: Namespaces & Cgroups — The Container Primitives

> *"Containers are not a thing. Containers are a collection of things — namespaces,
> cgroups, and layered filesystems working together."*
> — Jérôme Petazzoni

---

## Introduction

Every time you run `docker run`, a deceptively simple command triggers an intricate
orchestration of Linux kernel primitives. No new kernel code executes. No hypervisor
is involved. Instead, two decades-old subsystems — **namespaces** and **cgroups** —
combine to create what we perceive as a "container."

This chapter tears apart that illusion. We will build, from scratch, every isolation
and resource-control mechanism that container runtimes like `runc` rely on. By the end
of this chapter, you will be able to construct a container without Docker, without
`runc`, and without any container runtime at all — using only Go and the Linux kernel
API.

**What you will learn:**

- All 8 Linux namespace types and how to create, enter, and manage them
- The three namespace syscalls: `clone()`, `unshare()`, and `setns()`
- PID namespace init process responsibilities and zombie reaping
- Network namespaces, veth pairs, and routing between namespaces
- Mount namespaces, `pivot_root`, and mount propagation
- User namespaces and rootless containers
- Cgroups v1 architecture: hierarchies, subsystems, and the cgroupfs interface
- Cgroups v2 unified hierarchy: controllers, delegation, and the subtree model
- Combining all primitives into a minimal container runtime

**Prerequisites:** Chapter 4 (processes), familiarity with Go's `os/exec` and `syscall` packages.

---

## 5.1 Linux Namespaces — The Isolation Mechanism

### 5.1.1 What Namespaces Do

A **namespace** wraps a global system resource in an abstraction that makes it appear
to processes within the namespace that they have their own isolated instance of that
resource. Processes in different namespaces see different views of the same system.

```text
┌─────────────────────────────────────────────────────────────────────┐
│                        LINUX KERNEL                                 │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Namespace A   │  │ Namespace B   │  │ Namespace C   │              │
│  │              │  │              │  │              │              │
│  │ PID 1: init  │  │ PID 1: nginx │  │ PID 1: redis │              │
│  │ PID 2: app   │  │ PID 2: worker│  │ PID 2: sentinel            │
│  │              │  │              │  │              │              │
│  │ eth0: 10.0.1 │  │ eth0: 10.0.2 │  │ eth0: 10.0.3 │              │
│  │ hostname: web│  │ hostname: api│  │ hostname: db │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                     │
│  Each "container" above is just a set of namespaces.                │
│  The kernel maintains ONE process table, ONE network stack, etc.    │
│  Namespaces provide VIEWS into those global resources.              │
└─────────────────────────────────────────────────────────────────────┘
```

Key properties of namespaces:

1. **They are kernel objects** — each namespace is a data structure inside the kernel.
2. **They are reference-counted** — a namespace lives as long as at least one process
   is inside it, or a bind mount holds it open.
3. **They can be nested** — PID and user namespaces support hierarchical nesting.
4. **They are composable** — a process can be in PID namespace A but network namespace B.

### 5.1.2 The Eight Namespace Types

Linux provides eight namespace types. Each isolates a different kernel resource:

| Flag             | Namespace | Isolates                              | Kernel Version |
|------------------|-----------|---------------------------------------|----------------|
| `CLONE_NEWPID`   | PID       | Process IDs                           | 3.8            |
| `CLONE_NEWNET`   | Network   | Network stack (interfaces, routes)    | 2.6.29         |
| `CLONE_NEWNS`    | Mount     | Filesystem mount points               | 2.4.19         |
| `CLONE_NEWUTS`   | UTS       | Hostname and NIS domain name          | 2.6.19         |
| `CLONE_NEWIPC`   | IPC       | System V IPC, POSIX message queues    | 2.6.19         |
| `CLONE_NEWUSER`  | User      | User and group IDs                    | 3.8            |
| `CLONE_NEWCGROUP`| Cgroup    | Cgroup root directory                 | 4.6            |
| `CLONE_NEWTIME`  | Time      | Boot and monotonic clocks             | 5.6            |

> **Historical note:** The mount namespace (`CLONE_NEWNS`) was the first namespace
> implemented (Linux 2.4.19, 2002). Its flag is `CLONE_NEWNS` — simply "new namespace" —
> because at the time, nobody imagined there would be more than one type.

Let us examine each in depth.

---

#### 5.1.2.1 PID Namespace — Isolated Process ID Trees

The PID namespace isolates the process ID number space. Processes in different PID
namespaces can have the same PID. Every PID namespace has its own PID 1 — the init
process for that namespace.

```text
┌─────────────── Host PID Namespace ───────────────────────┐
│                                                           │
│  PID 1 (systemd)                                         │
│  PID 1234 (dockerd)                                      │
│  PID 5678 ──────────────────────────────────────┐        │
│  PID 5679 ──────────────┐                       │        │
│                          │                       │        │
│  ┌── Container A PID NS ─┴──┐  ┌─ Container B PID NS ┴─┐│
│  │ PID 1 (= host PID 5679) │  │ PID 1 (= host PID 5678)││
│  │ PID 2 (= host PID 5680) │  │ PID 2 (= host PID 5681)││
│  │ PID 3 (= host PID 5682) │  │ PID 3 (= host PID 5683)││
│  └──────────────────────────┘  └─────────────────────────┘│
│                                                           │
│  From host: all PIDs visible (5678-5683)                  │
│  From Container A: only PIDs 1,2,3 visible                │
│  From Container B: only PIDs 1,2,3 visible                │
└───────────────────────────────────────────────────────────┘
```

**PID 1 behavior in a PID namespace:**

- PID 1 is special: the kernel sends `SIGCHLD` to PID 1 when any orphan process dies.
- PID 1 **must** reap zombie children — if it doesn't, zombies accumulate.
- If PID 1 exits, the kernel sends `SIGKILL` to all remaining processes in that namespace.
- Signals: only signals for which PID 1 has registered a handler can be sent from outside
  the namespace. `SIGKILL` and `SIGSTOP` from the parent namespace always work.

**Nested PID namespaces:**

PID namespaces can be nested. A process has a PID in every ancestor namespace:

```text
Host NS:       PID 8000
  └─ NS Level 1: PID 50
       └─ NS Level 2: PID 1
```

The process "sees" itself as PID 1 but the host sees it as PID 8000.

---

#### 5.1.2.2 Network Namespace — Isolated Network Stack

Each network namespace has its own:

- Network interfaces (lo, eth0, etc.)
- IPv4 and IPv6 protocol stacks
- Routing tables
- Firewall rules (iptables/nftables)
- `/proc/net`, `/sys/class/net`
- Port numbers (two containers can both listen on port 80)
- Unix domain sockets (abstract namespace ones are isolated)

```text
┌────────── Host Network Namespace ──────────────────────────────┐
│                                                                 │
│  eth0: 192.168.1.100           docker0: 172.17.0.1             │
│  lo: 127.0.0.1                 (bridge)                        │
│       │                            │                            │
│       │                     ┌──────┴───────┐                    │
│       │                     │              │                    │
│       │                   veth-a         veth-b                 │
│       │                     │              │                    │
│  ┌────┼─── Container A NS ──┼──┐  ┌───────┼── Container B NS ─┐│
│  │    │   eth0: 172.17.0.2  │  │  │ eth0: 172.17.0.3          ││
│  │    │   lo: 127.0.0.1     │  │  │ lo: 127.0.0.1             ││
│  │    │   (own routes)       │  │  │ (own routes)              ││
│  │    │   (own iptables)     │  │  │ (own iptables)            ││
│  └────┼──────────────────────┘  │  └───────────────────────────┘│
│       │                         │                               │
└───────┼─────────────────────────┼───────────────────────────────┘
```

Network namespaces communicate via:
- **veth pairs** — virtual Ethernet cables connecting two namespaces
- **bridges** — L2 switching between multiple veth endpoints
- **macvlan/ipvlan** — exposing a physical NIC directly into a namespace

---

#### 5.1.2.3 Mount Namespace — Isolated Filesystem View

The mount namespace isolates the list of mount points seen by a process. Changes to
mount points (mount, umount, pivot_root) in one mount namespace are invisible to others.

**Mount propagation** controls whether mounts in one namespace propagate to others:

| Propagation Type | Behavior                                                     |
|------------------|--------------------------------------------------------------|
| **shared**       | Mounts propagate bidirectionally between peer namespaces     |
| **private**      | No propagation — mounts are completely isolated              |
| **slave**        | One-directional — master propagates to slave, not reverse    |
| **unbindable**   | Like private, but cannot be bind-mounted                     |

```text
Mount Propagation Example:

     shared mount at /data
    ┌──────────┐
    │ NS-A     │──── mount /data/new ───► visible in NS-B (shared)
    └──────────┘
         │
    ┌──────────┐
    │ NS-B     │──── mount /data/other ──► visible in NS-A (shared)
    └──────────┘

     slave mount at /data
    ┌──────────┐
    │ NS-A     │──── mount /data/new ───► visible in NS-B (slave)
    └──────────┘
         │
    ┌──────────┐
    │ NS-B     │──── mount /data/other ──► NOT visible in NS-A
    └──────────┘
```

---

#### 5.1.2.4 UTS Namespace — Isolated Hostname and Domain Name

UTS stands for "UNIX Time-Sharing" — a historical name from the `utsname` struct.
This namespace isolates:

- **hostname** — `sethostname()` / `hostname` command
- **NIS domain name** — `setdomainname()`

Each container gets its own hostname. This is why `docker run --hostname myhost` works —
Docker creates a UTS namespace and sets the hostname inside it.

---

#### 5.1.2.5 IPC Namespace — Isolated Inter-Process Communication

The IPC namespace isolates:

- **System V IPC objects**: shared memory segments, message queues, semaphore sets
- **POSIX message queues** (under `/dev/mqueue`)

Processes in different IPC namespaces cannot access each other's IPC objects.
This prevents a container from interfering with another container's shared memory.

---

#### 5.1.2.6 User Namespace — UID/GID Mapping and Rootless Containers

The user namespace is the most powerful and most security-critical namespace. It isolates:

- User IDs (UIDs) and Group IDs (GIDs)
- Keys and keyrings
- Capabilities

A process can be UID 0 (root) inside a user namespace but map to an unprivileged
UID (e.g., 100000) on the host. This is the foundation of **rootless containers**.

```text
┌──── Host ─────────────────────────────────────┐
│                                                │
│  UID 1000 (arpan) starts container             │
│       │                                        │
│  ┌────┴─── User Namespace ───────────────┐     │
│  │                                       │     │
│  │  UID 0 (root inside)  ═══► maps to    │     │
│  │                           host UID 100000   │
│  │  UID 1 (daemon inside) ═══► maps to   │     │
│  │                           host UID 100001   │
│  │                                       │     │
│  │  Has CAP_SYS_ADMIN inside namespace   │     │
│  │  but NO capabilities on host          │     │
│  └───────────────────────────────────────┘     │
│                                                │
└────────────────────────────────────────────────┘
```

UID mapping is configured via `/proc/[pid]/uid_map` and `/proc/[pid]/gid_map`:
```
# Format: <inner-start> <outer-start> <count>
# Map inner UID 0 to outer UID 100000, for 65536 UIDs
0 100000 65536
```

---

#### 5.1.2.7 Cgroup Namespace — Virtualized Cgroup View

The cgroup namespace virtualizes the view of `/sys/fs/cgroup`. A process in a cgroup
namespace sees its own cgroup as the root of the hierarchy.

Without a cgroup namespace, a container could read `/proc/self/cgroup` and discover
its real position in the host's cgroup hierarchy — leaking host information.

```text
Host cgroup tree:               Container's view (cgroup NS):
/sys/fs/cgroup/                 /sys/fs/cgroup/
└── system.slice/               └── (root — appears as /)
    └── docker-abc123.scope/        ├── cpu.max
        ├── cpu.max                 ├── memory.max
        ├── memory.max              └── pids.max
        └── pids.max
```

---

#### 5.1.2.8 Time Namespace — Isolated Clocks (Linux 5.6+)

The time namespace, added in Linux 5.6 (March 2020), isolates:

- **CLOCK_BOOTTIME** — time since system boot
- **CLOCK_MONOTONIC** — monotonic clock

This allows a container to see different values for "time since boot" than the host.
Useful when migrating containers between hosts (CRIU checkpoint/restore), where the
boot time differs.

> **Note:** `CLOCK_REALTIME` (wall clock) is NOT isolated — it would break too many
> applications. NTP still manages the real-time clock globally.

---
## 5.2 Namespace Syscalls — clone(), unshare(), and setns()

Three syscalls form the namespace API:

| Syscall     | Purpose                                          |
|-------------|--------------------------------------------------|
| `clone()`   | Create a new process in new namespace(s)         |
| `unshare()` | Move the calling process into new namespace(s)   |
| `setns()`   | Join an existing namespace                       |

### 5.2.1 clone() — Create Process in New Namespaces

`clone()` is the general-purpose process creation syscall (the underlying
implementation of `fork()`). When combined with namespace flags, it creates the child
process in new namespaces:

```c
// C prototype
int clone(int (*fn)(void *), void *stack, int flags, void *arg);

// Namespace flags can be OR'd together:
int flags = CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS | CLONE_NEWUTS | SIGCHLD;
```

In Go, we use `syscall.SysProcAttr` to set clone flags:

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
	"syscall"
)

// createNamespacedProcess creates a child process that runs in its own
// PID, UTS, mount, and IPC namespaces. The child runs /bin/sh so we
// can interactively explore the isolated environment.
//
// This function demonstrates the fundamental mechanism that container
// runtimes use: setting clone flags via SysProcAttr before exec.
//
// Namespace flags used:
//   - CLONE_NEWPID:  new PID namespace (child becomes PID 1)
//   - CLONE_NEWUTS:  new UTS namespace (isolated hostname)
//   - CLONE_NEWNS:   new mount namespace (isolated mounts)
//   - CLONE_NEWIPC:  new IPC namespace (isolated IPC)
func createNamespacedProcess() error {
	// Create a command that will execute in the child process.
	// We use /proc/self/exe to re-exec the current binary when
	// building real container runtimes. Here we use /bin/sh for
	// simplicity.
	cmd := exec.Command("/bin/sh")

	// SysProcAttr allows us to set Linux-specific process attributes.
	// The Cloneflags field maps directly to the flags argument of
	// the clone(2) syscall.
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWPID | // New PID namespace
			syscall.CLONE_NEWUTS | // New UTS namespace
			syscall.CLONE_NEWNS | // New mount namespace
			syscall.CLONE_NEWIPC, // New IPC namespace
	}

	// Connect the child's stdio to our own so we can interact
	// with the shell inside the namespaces.
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	// Start the process. Under the hood, Go calls clone(2) with
	// our flags, creating the child in new namespaces.
	if err := cmd.Run(); err != nil {
		return fmt.Errorf("failed to run namespaced process: %w", err)
	}

	return nil
}

func main() {
	fmt.Println("[*] Creating process in new namespaces...")
	if err := createNamespacedProcess(); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
}
```

### 5.2.2 unshare() — Disassociate from Current Namespaces

`unshare()` moves the calling process (or thread) into a new namespace without creating
a new process. This is useful when you want the current process itself to enter a new
namespace.

```c
// C prototype
int unshare(int flags);

// Example: move current process into a new mount namespace
unshare(CLONE_NEWNS);
```

The `unshare` command-line tool wraps this syscall:

```bash
# Run a shell in a new UTS namespace
$ unshare --uts /bin/bash
$ hostname my-isolated-host    # Only visible inside this namespace
$ exit
$ hostname                     # Back to original hostname
```

In Go:

```go
package main

import (
	"fmt"
	"os"
	"syscall"
)

// unshareNamespace calls the unshare(2) syscall to move the current
// process into a new namespace specified by the flags parameter.
//
// Unlike clone(), which creates a NEW process in a new namespace,
// unshare() moves the CURRENT process into a new namespace.
//
// Parameters:
//   - flags: one or more CLONE_NEW* flags OR'd together
//
// Common usage: unshare mount namespace before pivot_root,
// or unshare user namespace to gain capabilities for further
// namespace operations.
//
// Important caveat for PID namespaces: unshare(CLONE_NEWPID) does
// NOT move the calling process into the new PID namespace. Instead,
// the NEXT child process created will be PID 1 in the new namespace.
// The calling process remains in the old PID namespace.
func unshareNamespace(flags uintptr) error {
	// syscall.RawSyscall provides direct access to Linux syscalls.
	// SYS_UNSHARE is the syscall number for unshare(2).
	_, _, errno := syscall.RawSyscall(
		syscall.SYS_UNSHARE, // syscall number
		flags,               // namespace flags
		0,                   // unused
		0,                   // unused
	)
	if errno != 0 {
		return fmt.Errorf("unshare failed: %v", errno)
	}
	return nil
}

func main() {
	fmt.Println("[*] Unsharing UTS namespace...")

	// Move into a new UTS namespace. After this call, changes to
	// hostname will not affect the parent namespace.
	if err := unshareNamespace(syscall.CLONE_NEWUTS); err != nil {
		fmt.Fprintf(os.Stderr, "unshare UTS: %v\n", err)
		os.Exit(1)
	}

	// Now we can set a new hostname visible only to us.
	if err := syscall.Sethostname([]byte("isolated-host")); err != nil {
		fmt.Fprintf(os.Stderr, "sethostname: %v\n", err)
		os.Exit(1)
	}

	// Verify the hostname change.
	hostname, _ := os.Hostname()
	fmt.Printf("[+] Hostname inside new UTS namespace: %s\n", hostname)
}
```

### 5.2.3 setns() — Join an Existing Namespace

`setns()` allows a process to join an **existing** namespace by passing a file
descriptor to the namespace. Namespace file descriptors are found in `/proc/[pid]/ns/`.

```c
// C prototype
int setns(int fd, int nstype);
```

This is how `docker exec` works: it opens the namespace file descriptors of the
container's init process and calls `setns()` for each one.

```
/proc/[pid]/ns/ contains symlinks to namespace inodes:

$ ls -la /proc/1/ns/
lrwxrwxrwx 1 root root 0 Jan  1 00:00 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Jan  1 00:00 ipc    -> 'ipc:[4026531839]'
lrwxrwxrwx 1 root root 0 Jan  1 00:00 mnt    -> 'mnt:[4026531841]'
lrwxrwxrwx 1 root root 0 Jan  1 00:00 net    -> 'net:[4026531840]'
lrwxrwxrwx 1 root root 0 Jan  1 00:00 pid    -> 'pid:[4026531836]'
lrwxrwxrwx 1 root root 0 Jan  1 00:00 user   -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Jan  1 00:00 uts    -> 'uts:[4026531838]'

The number in brackets is the inode number — unique per namespace instance.
Two processes in the same namespace will have the same inode number.
```

In Go:

```go
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"runtime"
	"syscall"
)

// nsType maps human-readable namespace names to their /proc/[pid]/ns/ filenames
// and the corresponding CLONE_NEW* flag used by setns() for type validation.
//
// When calling setns(), the nstype parameter acts as a safety check:
// the kernel verifies that the file descriptor actually refers to a
// namespace of the specified type. Passing 0 skips this check.
var nsType = map[string]struct {
	file string
	flag int
}{
	"pid":    {"pid", syscall.CLONE_NEWPID},
	"net":    {"net", syscall.CLONE_NEWNET},
	"mnt":    {"mnt", syscall.CLONE_NEWNS},
	"uts":    {"uts", syscall.CLONE_NEWUTS},
	"ipc":    {"ipc", syscall.CLONE_NEWIPC},
	"user":   {"user", syscall.CLONE_NEWUSER},
	"cgroup": {"cgroup", 0x02000000}, // CLONE_NEWCGROUP
}

// enterNamespace opens the namespace file for the given PID and namespace
// type, then calls setns(2) to move the current thread into that namespace.
//
// Parameters:
//   - pid:    the target process whose namespace we want to join
//   - nsName: one of "pid", "net", "mnt", "uts", "ipc", "user", "cgroup"
//
// Important: setns() affects only the calling THREAD, not the entire process.
// In Go, we must lock the OS thread first with runtime.LockOSThread() to
// prevent the Go scheduler from migrating the goroutine to a different thread
// (which would still be in the old namespace).
//
// This function implements the core mechanism behind `docker exec` and
// `nsenter` — joining an existing container's namespaces.
func enterNamespace(pid int, nsName string) error {
	// Lock this goroutine to its current OS thread. Without this,
	// the Go scheduler could move us to another thread after setns(),
	// and that thread would still be in the old namespace.
	runtime.LockOSThread()
	// Note: we deliberately do NOT call runtime.UnlockOSThread()
	// because we want ALL subsequent work on this goroutine to
	// remain in the new namespace.

	ns, ok := nsType[nsName]
	if !ok {
		return fmt.Errorf("unknown namespace type: %s", nsName)
	}

	// Construct the path to the namespace file descriptor.
	// /proc/[pid]/ns/[type] is a magic symlink that, when opened,
	// returns a file descriptor referring to that namespace.
	nsPath := filepath.Join("/proc", fmt.Sprintf("%d", pid), "ns", ns.file)

	// Open the namespace file descriptor.
	fd, err := syscall.Open(nsPath, syscall.O_RDONLY, 0)
	if err != nil {
		return fmt.Errorf("open %s: %w", nsPath, err)
	}
	defer syscall.Close(fd)

	// Call setns(2) to join the namespace.
	// The second argument is the expected namespace type — the kernel
	// will verify the fd actually refers to this type of namespace.
	_, _, errno := syscall.RawSyscall(
		308,              // SYS_SETNS on x86_64
		uintptr(fd),      // namespace file descriptor
		uintptr(ns.flag), // expected namespace type (0 = any)
		0,
	)
	if errno != 0 {
		return fmt.Errorf("setns %s: %v", nsName, errno)
	}

	fmt.Printf("[+] Joined %s namespace of PID %d\n", nsName, pid)
	return nil
}

func main() {
	if len(os.Args) < 3 {
		fmt.Fprintf(os.Stderr, "Usage: %s <pid> <namespace-type>\n", os.Args[0])
		fmt.Fprintf(os.Stderr, "  namespace-type: pid, net, mnt, uts, ipc, user, cgroup\n")
		os.Exit(1)
	}

	pid := 0
	fmt.Sscanf(os.Args[1], "%d", &pid)
	nsName := os.Args[2]

	if err := enterNamespace(pid, nsName); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}

	// After joining the namespace, exec a shell so the user can
	// explore the namespace interactively.
	syscall.Exec("/bin/sh", []string{"sh"}, os.Environ())
}
```

### 5.2.4 Hands-On: Building a Namespace Explorer Tool in Go

Let us build a comprehensive tool that lists all namespaces for a given process
and compares them with the current process's namespaces:

```go
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"strings"
)

// NamespaceInfo holds metadata about a single namespace instance.
// The Inode field uniquely identifies a namespace — two processes
// sharing the same namespace will have the same inode number.
type NamespaceInfo struct {
	// Type is the namespace type name (e.g., "pid", "net", "mnt").
	Type string

	// Path is the full path to /proc/[pid]/ns/[type].
	Path string

	// Inode is the namespace inode number extracted from the symlink
	// target (e.g., "pid:[4026531836]" → inode 4026531836).
	// Two processes in the same namespace share the same inode.
	Inode string
}

// allNamespaceTypes lists every namespace type available in /proc/[pid]/ns/.
// This includes the 8 standard types plus "pid_for_children" and
// "user" (which may differ from the process's own namespace).
var allNamespaceTypes = []string{
	"cgroup",
	"ipc",
	"mnt",
	"net",
	"pid",
	"pid_for_children",
	"time",
	"time_for_children",
	"user",
	"uts",
}

// getNamespaceInfo reads the namespace symlink for a given PID and
// namespace type. The symlink target has the format "type:[inode]",
// e.g., "pid:[4026531836]".
//
// Returns a NamespaceInfo struct with the extracted inode, or an error
// if the namespace file cannot be read (e.g., insufficient permissions
// or the namespace type doesn't exist on this kernel).
func getNamespaceInfo(pid int, nsType string) (NamespaceInfo, error) {
	nsPath := filepath.Join("/proc", fmt.Sprintf("%d", pid), "ns", nsType)

	// Readlink returns the symlink target, e.g., "pid:[4026531836]"
	target, err := os.Readlink(nsPath)
	if err != nil {
		return NamespaceInfo{}, fmt.Errorf("readlink %s: %w", nsPath, err)
	}

	return NamespaceInfo{
		Type:  nsType,
		Path:  nsPath,
		Inode: target,
	}, nil
}

// getAllNamespaces retrieves namespace information for all namespace types
// of a given process. It silently skips namespace types that don't exist
// (e.g., "time" on kernels < 5.6).
func getAllNamespaces(pid int) []NamespaceInfo {
	var namespaces []NamespaceInfo
	for _, nsType := range allNamespaceTypes {
		info, err := getNamespaceInfo(pid, nsType)
		if err != nil {
			continue // Skip unavailable namespace types
		}
		namespaces = append(namespaces, info)
	}
	return namespaces
}

// compareNamespaces compares the namespaces of two processes and prints
// a formatted table showing which namespaces are shared and which differ.
//
// This is useful for understanding container isolation — a containerized
// process should differ from the host PID 1 in most namespace types.
func compareNamespaces(pid1, pid2 int) {
	fmt.Printf("\n%-20s %-30s %-30s %s\n",
		"Namespace", fmt.Sprintf("PID %d", pid1),
		fmt.Sprintf("PID %d", pid2), "Same?")
	fmt.Println(strings.Repeat("─", 100))

	for _, nsType := range allNamespaceTypes {
		info1, err1 := getNamespaceInfo(pid1, nsType)
		info2, err2 := getNamespaceInfo(pid2, nsType)

		if err1 != nil || err2 != nil {
			fmt.Printf("%-20s %-30s %-30s %s\n",
				nsType, "(unavailable)", "(unavailable)", "?")
			continue
		}

		same := "✓ YES"
		if info1.Inode != info2.Inode {
			same = "✗ NO"
		}

		fmt.Printf("%-20s %-30s %-30s %s\n",
			nsType, info1.Inode, info2.Inode, same)
	}
}

func main() {
	// Get current process's PID for self-comparison.
	selfPID := os.Getpid()

	fmt.Printf("=== Namespace Explorer ===\n")
	fmt.Printf("Current PID: %d\n\n", selfPID)

	// List namespaces for PID 1 (init/systemd)
	fmt.Println("--- Namespaces for PID 1 (init) ---")
	for _, ns := range getAllNamespaces(1) {
		fmt.Printf("  %-20s %s\n", ns.Type, ns.Inode)
	}

	// List namespaces for current process
	fmt.Printf("\n--- Namespaces for PID %d (self) ---\n", selfPID)
	for _, ns := range getAllNamespaces(selfPID) {
		fmt.Printf("  %-20s %s\n", ns.Type, ns.Inode)
	}

	// Compare — if we're in a container, namespaces will differ from PID 1
	fmt.Println("\n--- Comparison: PID 1 vs Self ---")
	compareNamespaces(1, selfPID)
}
```

---
## 5.3 PID Namespace Deep Dive

### 5.3.1 PID 1 Responsibilities

When you create a PID namespace, the first process becomes PID 1. This process has
special kernel-enforced responsibilities:

1. **Zombie reaping**: When any process in the namespace becomes an orphan (its parent
   dies), it is re-parented to PID 1. PID 1 must call `wait()` to reap these processes,
   otherwise they become zombies consuming kernel resources.

2. **Signal handling**: The kernel protects PID 1 from signals it hasn't registered
   handlers for. Only `SIGKILL` and `SIGSTOP` from the parent namespace can kill PID 1.
   Inside the namespace, PID 1 is unkillable unless it has registered a handler for
   the signal.

3. **Namespace lifecycle**: When PID 1 dies, the kernel sends `SIGKILL` to every
   remaining process in the namespace. The namespace is then destroyed (unless held
   open by a bind mount).

```
PID 1 as Zombie Reaper:

  PID 1 (init)
    │
    ├── PID 2 (app)
    │     └── PID 3 (worker)
    │
    └── PID 4 (logger)

  If PID 2 dies while PID 3 is still running:
  1. PID 3 becomes an orphan
  2. Kernel re-parents PID 3 to PID 1
  3. PID 1 must wait() on PID 3 to prevent zombie

  PID 1 (init)
    │
    ├── PID 3 (worker) ← re-parented from dead PID 2
    │
    └── PID 4 (logger)
```

### 5.3.2 Hands-On: Creating a PID Namespace with Init Process

This example creates a proper PID 1 init process that handles signal forwarding
and zombie reaping — exactly what tini, dumb-init, or the Docker --init flag provides:

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
	"os/signal"
	"syscall"
)

// runAsInit is the code path executed when this binary is re-invoked
// as PID 1 inside the new PID namespace. It sets up signal forwarding,
// spawns the real workload, and reaps zombie processes.
//
// Why re-exec? When Go creates a child process with CLONE_NEWPID, the
// child gets a PID in the new namespace, but /proc still shows the old
// namespace's view. By re-exec'ing, we ensure the entire Go runtime
// reinitializes inside the namespace with the correct PID 1 identity.
//
// This function implements the "init process" pattern used by:
//   - tini (https://github.com/krallin/tini)
//   - dumb-init (https://github.com/Yelp/dumb-init)
//   - Docker's --init flag
func runAsInit() {
	fmt.Printf("[init] Running as PID %d inside namespace\n", os.Getpid())

	// Step 1: Mount a new /proc so that tools like `ps` show only
	// processes in this PID namespace, not the host's processes.
	//
	// We must mount a new /proc because the kernel populates it
	// based on the PID namespace of the mounting process.
	if err := syscall.Mount("proc", "/proc", "proc", 0, ""); err != nil {
		fmt.Fprintf(os.Stderr, "[init] mount /proc: %v\n", err)
	}

	// Step 2: Set hostname to demonstrate UTS namespace isolation.
	syscall.Sethostname([]byte("container"))

	// Step 3: Start the real workload as a child process.
	// In a real container runtime, this would be the user's entrypoint
	// (e.g., nginx, python app.py, etc.)
	cmd := exec.Command("/bin/sh", "-c", "echo 'Hello from PID NS!' && ps aux && exec sleep 5")
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Start(); err != nil {
		fmt.Fprintf(os.Stderr, "[init] start workload: %v\n", err)
		os.Exit(1)
	}

	// Step 4: Set up signal forwarding. PID 1 must forward signals
	// to the child workload process because:
	//   - The kernel won't deliver unhandled signals to PID 1
	//   - If we receive SIGTERM, we should gracefully stop the child
	//
	// This goroutine catches signals and forwards them to the child.
	sigCh := make(chan os.Signal, 10)
	signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT, syscall.SIGHUP)
	go func() {
		for sig := range sigCh {
			// Forward the signal to the child process.
			// If the child has already exited, this is a no-op.
			cmd.Process.Signal(sig)
		}
	}()

	// Step 5: Zombie reaping loop. As PID 1, we are responsible for
	// reaping ALL orphaned child processes, not just our direct child.
	//
	// We use waitpid(-1, ..., WNOHANG) in a loop to reap any process
	// that has exited. This is critical — without this, zombie processes
	// would accumulate in the namespace.
	go func() {
		var ws syscall.WaitStatus
		for {
			// Wait for ANY child process (-1) to change state.
			// WNOHANG: return immediately if no child has exited.
			// This prevents blocking if no zombies are available.
			pid, err := syscall.Wait4(-1, &ws, syscall.WNOHANG, nil)
			if err != nil || pid <= 0 {
				// No more zombies to reap. Sleep briefly and try again.
				// In production, you'd use signalfd or SIGCHLD instead of polling.
				syscall.Nanosleep(&syscall.Timespec{Sec: 0, Nsec: 100000000}, nil)
				continue
			}
			fmt.Printf("[init] Reaped zombie PID %d (status: %d)\n", pid, ws.ExitStatus())
		}
	}()

	// Step 6: Wait for our primary workload to exit.
	if err := cmd.Wait(); err != nil {
		fmt.Fprintf(os.Stderr, "[init] workload exited: %v\n", err)
	}

	fmt.Println("[init] Workload finished. Init process exiting.")
}

func main() {
	// Check if we're being re-invoked as the init process inside
	// the new PID namespace. We use an environment variable as a
	// sentinel to distinguish the parent invocation from the child.
	if os.Getenv("_CONTAINER_INIT") == "1" {
		runAsInit()
		return
	}

	fmt.Println("[parent] Creating new PID + UTS + Mount namespaces...")

	// Re-execute ourselves inside new namespaces. The child process
	// will detect _CONTAINER_INIT=1 and run runAsInit() instead.
	cmd := exec.Command("/proc/self/exe")
	cmd.Env = append(os.Environ(), "_CONTAINER_INIT=1")
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	// Set clone flags to create new namespaces for the child.
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWPID | // Child becomes PID 1
			syscall.CLONE_NEWUTS | // Isolated hostname
			syscall.CLONE_NEWNS, // Isolated mounts (for /proc)
	}

	if err := cmd.Run(); err != nil {
		fmt.Fprintf(os.Stderr, "[parent] Error: %v\n", err)
		os.Exit(1)
	}

	fmt.Println("[parent] Child namespace terminated.")
}
```

### 5.3.3 Nested PID Namespaces and /proc

PID namespaces can be nested to arbitrary depth. Each level adds a new PID to
every process within it:

```
Level 0 (Host):     PID 42000
Level 1 (Container): PID 100
Level 2 (Nested):    PID 1

The process with PID 1 at Level 2 has THREE PIDs:
  - PID 1     (visible at Level 2)
  - PID 100   (visible at Level 1)
  - PID 42000 (visible at Level 0 / host)

Each level's /proc shows only processes in that namespace and below.
Level 0 sees all processes.
Level 1 sees PIDs 1-100+ (its own subtree).
Level 2 sees only PID 1 (and its children).
```

This nesting is used by Kubernetes for pod sandboxing and by Kata Containers
for nested isolation.

---

## 5.4 Network Namespace Deep Dive

### 5.4.1 Anatomy of Network Namespace Isolation

A new network namespace starts completely empty — it has only a `lo` (loopback)
interface, and even that is initially DOWN. To provide connectivity, you must:

1. Create a **veth pair** (virtual Ethernet cable)
2. Move one end into the network namespace
3. Assign IP addresses to both ends
4. Set up routing
5. Optionally configure NAT for internet access

```
Complete Network Namespace Setup:

  ┌─────────── Host Namespace ──────────────────────────────────┐
  │                                                              │
  │  eth0: 192.168.1.100/24 (physical NIC)                      │
  │    │                                                         │
  │    │  ┌─────────┐                                            │
  │    └──│ routing │── default gw 192.168.1.1                   │
  │       │  table  │                                            │
  │       └─────────┘                                            │
  │            │                                                  │
  │       ┌─────────┐                                            │
  │       │  br0    │ 10.0.0.1/24  (bridge / virtual switch)    │
  │       └────┬────┘                                            │
  │            │                                                  │
  │     ┌──────┼────────┐                                        │
  │     │      │        │                                        │
  │   veth-a0 veth-b0  veth-c0  (host-side veth endpoints)     │
  │     │      │        │                                        │
  ├─────┼──────┼────────┼────────────────────────────────────────┤
  │     │      │        │                                        │
  │   veth-a1 veth-b1  veth-c1  (container-side veth endpoints)│
  │     │      │        │                                        │
  │  ┌──┴──┐ ┌─┴──┐ ┌──┴──┐                                    │
  │  │NS-A │ │NS-B│ │NS-C │                                    │
  │  │.0.2 │ │.0.3│ │.0.4 │                                    │
  │  └─────┘ └────┘ └─────┘                                    │
  │                                                              │
  │  iptables -t nat -A POSTROUTING -s 10.0.0.0/24             │
  │           -o eth0 -j MASQUERADE   (NAT for internet)        │
  └──────────────────────────────────────────────────────────────┘
```

### 5.4.2 Hands-On: Full Network Namespace Setup in Go

This example creates a network namespace with full connectivity using the
`vishvananda/netlink` library — the Go equivalent of the `ip` command:

```go
package main

import (
	"fmt"
	"net"
	"os"
	"os/exec"
	"runtime"
	"syscall"

	"github.com/vishvananda/netlink"
	"github.com/vishvananda/netns"
)

// NetworkNamespaceConfig holds the configuration parameters needed to
// set up an isolated network namespace with connectivity back to the host.
//
// This configuration mirrors what Docker and CNI plugins create when
// setting up container networking.
type NetworkNamespaceConfig struct {
	// HostVethName is the name of the veth endpoint that remains in the
	// host namespace. Convention: "veth" + short container ID.
	HostVethName string

	// ContainerVethName is the name of the veth endpoint moved into the
	// container namespace. Typically renamed to "eth0" inside.
	ContainerVethName string

	// HostIP is the IP address assigned to the host-side veth endpoint.
	// This serves as the default gateway for the container.
	HostIP net.IPNet

	// ContainerIP is the IP address assigned to the container-side veth.
	ContainerIP net.IPNet
}

// setupNetworkNamespace creates a new network namespace and configures
// full network connectivity using a veth pair.
//
// Steps performed:
//  1. Create a veth pair (two linked virtual Ethernet interfaces)
//  2. Create a new network namespace
//  3. Move one veth endpoint into the new namespace
//  4. Configure IP addresses on both endpoints
//  5. Bring both interfaces UP
//  6. Set up routing in the container namespace
//
// This is the exact sequence that Docker, Kubernetes CNI plugins, and
// other container runtimes follow to set up container networking.
//
// Returns the network namespace file descriptor for later use with setns().
func setupNetworkNamespace(cfg NetworkNamespaceConfig) (netns.NsHandle, error) {
	// Lock the OS thread because network namespace operations affect
	// the calling thread. We don't want the Go scheduler to move
	// this goroutine to a different thread mid-operation.
	runtime.LockOSThread()
	defer runtime.UnlockOSThread()

	// Save the current (host) network namespace so we can return to it.
	hostNS, err := netns.Get()
	if err != nil {
		return 0, fmt.Errorf("get host netns: %w", err)
	}
	defer hostNS.Close()

	// Step 1: Create a veth pair. A veth pair is like a virtual Ethernet
	// cable — packets sent into one end come out the other end. This is
	// the fundamental building block of container networking.
	vethAttrs := netlink.NewLinkAttrs()
	vethAttrs.Name = cfg.HostVethName
	veth := &netlink.Veth{
		LinkAttrs: vethAttrs,
		PeerName:  cfg.ContainerVethName,
	}

	if err := netlink.LinkAdd(veth); err != nil {
		return 0, fmt.Errorf("create veth pair: %w", err)
	}
	fmt.Printf("[+] Created veth pair: %s <-> %s\n",
		cfg.HostVethName, cfg.ContainerVethName)

	// Step 2: Create a new network namespace.
	newNS, err := netns.New()
	if err != nil {
		return 0, fmt.Errorf("create netns: %w", err)
	}

	// Switch back to host namespace to configure the host-side veth.
	if err := netns.Set(hostNS); err != nil {
		return 0, fmt.Errorf("set host netns: %w", err)
	}

	// Step 3: Move the container-side veth into the new namespace.
	// After this, the container veth is only visible inside the new NS.
	containerVeth, err := netlink.LinkByName(cfg.ContainerVethName)
	if err != nil {
		return 0, fmt.Errorf("find container veth: %w", err)
	}

	if err := netlink.LinkSetNsFd(containerVeth, int(newNS)); err != nil {
		return 0, fmt.Errorf("move veth to netns: %w", err)
	}
	fmt.Printf("[+] Moved %s into new network namespace\n", cfg.ContainerVethName)

	// Step 4a: Configure the host-side veth with an IP address.
	hostVeth, err := netlink.LinkByName(cfg.HostVethName)
	if err != nil {
		return 0, fmt.Errorf("find host veth: %w", err)
	}

	hostAddr := &netlink.Addr{IPNet: &cfg.HostIP}
	if err := netlink.AddrAdd(hostVeth, hostAddr); err != nil {
		return 0, fmt.Errorf("add host IP: %w", err)
	}

	// Bring the host-side veth UP.
	if err := netlink.LinkSetUp(hostVeth); err != nil {
		return 0, fmt.Errorf("set host veth up: %w", err)
	}
	fmt.Printf("[+] Host veth %s: %s UP\n", cfg.HostVethName, cfg.HostIP.String())

	// Step 4b: Switch to container namespace to configure its veth.
	if err := netns.Set(newNS); err != nil {
		return 0, fmt.Errorf("enter container netns: %w", err)
	}

	// Find the container veth (now inside the new namespace).
	cVeth, err := netlink.LinkByName(cfg.ContainerVethName)
	if err != nil {
		return 0, fmt.Errorf("find container veth in ns: %w", err)
	}

	// Assign IP address to the container-side veth.
	containerAddr := &netlink.Addr{IPNet: &cfg.ContainerIP}
	if err := netlink.AddrAdd(cVeth, containerAddr); err != nil {
		return 0, fmt.Errorf("add container IP: %w", err)
	}

	// Bring the container-side veth UP.
	if err := netlink.LinkSetUp(cVeth); err != nil {
		return 0, fmt.Errorf("set container veth up: %w", err)
	}

	// Also bring up the loopback interface inside the namespace.
	// New network namespaces have a lo interface, but it starts DOWN.
	lo, err := netlink.LinkByName("lo")
	if err == nil {
		netlink.LinkSetUp(lo)
	}

	// Step 5: Add a default route pointing to the host-side veth IP.
	// This allows the container to reach the host and (with NAT)
	// the external network.
	defaultRoute := &netlink.Route{
		Dst: nil, // nil Dst = default route (0.0.0.0/0)
		Gw:  cfg.HostIP.IP,
	}
	if err := netlink.RouteAdd(defaultRoute); err != nil {
		return 0, fmt.Errorf("add default route: %w", err)
	}
	fmt.Printf("[+] Container veth %s: %s UP, gateway: %s\n",
		cfg.ContainerVethName, cfg.ContainerIP.String(), cfg.HostIP.IP)

	// Return to host namespace.
	if err := netns.Set(hostNS); err != nil {
		return 0, fmt.Errorf("return to host netns: %w", err)
	}

	return newNS, nil
}

func main() {
	// Configuration for our network namespace — similar to Docker's
	// default bridge network configuration.
	cfg := NetworkNamespaceConfig{
		HostVethName:      "veth-host",
		ContainerVethName: "eth0",
		HostIP: net.IPNet{
			IP:   net.ParseIP("10.0.0.1"),
			Mask: net.CIDRMask(24, 32),
		},
		ContainerIP: net.IPNet{
			IP:   net.ParseIP("10.0.0.2"),
			Mask: net.CIDRMask(24, 32),
		},
	}

	fmt.Println("=== Network Namespace Setup ===")

	nsHandle, err := setupNetworkNamespace(cfg)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	defer nsHandle.Close()

	fmt.Println("\n[+] Network namespace created and configured!")
	fmt.Println("[*] Testing connectivity...")

	// Test: run ping inside the container namespace.
	cmd := exec.Command("ip", "netns", "exec", "test", "ping", "-c", "2", "10.0.0.1")
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	cmd.Run()
}
```

### 5.4.3 Network Namespace with iptables and NAT

For containers to reach the internet, we need NAT (Network Address Translation)
on the host. This is what Docker's bridge networking does:

```bash
# Enable IP forwarding on the host
echo 1 > /proc/sys/net/ipv4/ip_forward

# Add MASQUERADE rule: packets from the container subnet going out
# through the host's physical interface get their source IP rewritten
# to the host's IP (NAT).
iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

# Allow forwarding between the bridge and the physical interface
iptables -A FORWARD -i veth-host -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o veth-host -m state --state RELATED,ESTABLISHED -j ACCEPT
```

---
## 5.5 Mount Namespace & Filesystem Isolation

### 5.5.1 pivot_root vs chroot — Why pivot_root Is Correct

Both `chroot` and `pivot_root` change the root filesystem, but they work very
differently:

| Feature            | chroot                          | pivot_root                      |
|--------------------|---------------------------------|---------------------------------|
| Changes root for   | The calling process only        | The entire mount namespace      |
| Old root           | Still accessible via ../../../  | Moved to a specified mountpoint |
| Security           | Escapable (chroot escape)       | Secure (old root is unmounted)  |
| Used by            | Build systems, rescue modes     | Container runtimes (runc, crun) |

**Why chroot is insecure for containers:**

```
chroot jail escape (classic technique):

1. Process is chroot'd to /container-root
2. Process opens a file descriptor to "." (current dir = /container-root)
3. Process calls chroot(".") again (creates a new chroot)
4. Process uses the saved fd to fchdir() back to the real root
5. Process calls chroot(".") — now at real root, jail escaped!

pivot_root prevents this by:
1. Moving the old root to a subdirectory
2. The process unmounts the old root completely
3. There is no reference to the old root left
```

### 5.5.2 Mount Propagation Types Explained

Mount propagation controls how mount events flow between mount namespaces.
Understanding this is critical for container storage drivers and volume mounts.

```
Mount Propagation Relationships:

   ┌─────────────────────────────────────────────────────────────┐
   │                    SHARED                                    │
   │                                                              │
   │  NS-A ◄─────── mount events ────────► NS-B                 │
   │        (bidirectional propagation)                           │
   └─────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────┐
   │                    SLAVE                                     │
   │                                                              │
   │  NS-A (master) ──── mount events ────► NS-B (slave)        │
   │  NS-B ─────────── mount events ────╳── NS-A (blocked)      │
   └─────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────┐
   │                   PRIVATE                                    │
   │                                                              │
   │  NS-A ──────── mount events ────╳──── NS-B                 │
   │  NS-B ──────── mount events ────╳──── NS-A                 │
   │        (no propagation in either direction)                  │
   └─────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────┐
   │                  UNBINDABLE                                  │
   │                                                              │
   │  Like private, PLUS: cannot be bind-mounted.                │
   │  Prevents recursive bind mount loops.                        │
   └─────────────────────────────────────────────────────────────┘
```

Setting propagation from the command line:
```bash
# Make a mount shared (propagates to peers)
mount --make-shared /mnt/data

# Make a mount private (no propagation)
mount --make-private /mnt/data

# Make a mount slave (receives from master, doesn't send)
mount --make-slave /mnt/data

# Make a mount unbindable (private + can't be bind-mounted)
mount --make-unbindable /mnt/data

# Make all mounts recursively private (common for containers)
mount --make-rprivate /
```

### 5.5.3 Bind Mounts

A bind mount makes a file or directory visible at another location. Unlike symlinks,
bind mounts work at the VFS level and are transparent to applications:

```bash
# Bind mount: make /data visible at /container-root/data
mount --bind /data /container-root/data

# Read-only bind mount (two-step process):
mount --bind /data /container-root/data
mount -o remount,ro,bind /container-root/data
```

Bind mounts are how Docker volumes work — `docker run -v /host/path:/container/path`
creates a bind mount from the host path into the container's mount namespace.

### 5.5.4 Hands-On: Isolated Filesystem with pivot_root in Go

This example sets up a complete isolated filesystem using `pivot_root`, exactly as
`runc` does when creating containers:

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"syscall"
)

// setupRootfs prepares an isolated root filesystem for the container.
// This involves:
//  1. Making all existing mounts private (preventing propagation)
//  2. Bind-mounting the new root onto itself (required by pivot_root)
//  3. Creating necessary mount points inside the new root
//  4. Mounting essential filesystems (proc, sys, dev)
//  5. Executing pivot_root to switch to the new root
//  6. Unmounting the old root
//
// After this function completes, the process sees only the new
// filesystem — the host's filesystem is completely inaccessible.
//
// Parameters:
//   - newRoot: path to the container's root filesystem (e.g., an
//     extracted Docker image layer or a minimal rootfs like Alpine)
func setupRootfs(newRoot string) error {
	// Step 1: Make the entire mount tree private. This prevents any
	// mount/unmount operations we perform from propagating back to
	// the host's mount namespace.
	//
	// MS_REC: apply recursively to all submounts
	// MS_PRIVATE: no mount propagation
	if err := syscall.Mount("", "/", "", syscall.MS_REC|syscall.MS_PRIVATE, ""); err != nil {
		return fmt.Errorf("make root private: %w", err)
	}

	// Step 2: Bind-mount the new root onto itself. This is a
	// requirement of pivot_root(2) — the new root must be a
	// mount point, and a bind mount is the simplest way to ensure this.
	if err := syscall.Mount(newRoot, newRoot, "", syscall.MS_BIND|syscall.MS_REC, ""); err != nil {
		return fmt.Errorf("bind mount new root: %w", err)
	}

	// Step 3: Create the directory where we'll put the old root.
	// After pivot_root, the old root filesystem will be accessible
	// here (temporarily — we unmount it in step 6).
	oldRoot := filepath.Join(newRoot, ".pivot_root")
	if err := os.MkdirAll(oldRoot, 0700); err != nil {
		return fmt.Errorf("mkdir old root: %w", err)
	}

	// Step 4: pivot_root(new_root, put_old)
	//
	// This syscall atomically:
	//   a) Makes new_root the new root filesystem (/)
	//   b) Moves the old root filesystem to put_old
	//
	// After this call:
	//   - "/" points to what was previously newRoot
	//   - "/.pivot_root" contains the old root
	if err := syscall.PivotRoot(newRoot, oldRoot); err != nil {
		return fmt.Errorf("pivot_root: %w", err)
	}

	// Step 5: Change working directory to the new root.
	// After pivot_root, our cwd might be invalid.
	if err := os.Chdir("/"); err != nil {
		return fmt.Errorf("chdir /: %w", err)
	}

	// Step 6: Unmount the old root. This is the critical security
	// step — without this, the host filesystem would still be
	// accessible at /.pivot_root.
	//
	// MNT_DETACH: perform a lazy unmount. The mount point is
	// immediately disconnected from the filesystem tree, and
	// cleanup happens when it's no longer in use.
	if err := syscall.Unmount("/.pivot_root", syscall.MNT_DETACH); err != nil {
		return fmt.Errorf("unmount old root: %w", err)
	}

	// Remove the now-empty mount point directory.
	os.Remove("/.pivot_root")

	// Step 7: Mount essential filesystems.
	// /proc: process information filesystem (needed for ps, /proc/self, etc.)
	os.MkdirAll("/proc", 0755)
	if err := syscall.Mount("proc", "/proc", "proc", 0, ""); err != nil {
		return fmt.Errorf("mount /proc: %w", err)
	}

	// /sys: sysfs (kernel objects, device info, cgroup interface)
	os.MkdirAll("/sys", 0755)
	if err := syscall.Mount("sysfs", "/sys", "sysfs",
		syscall.MS_RDONLY|syscall.MS_NOSUID|syscall.MS_NODEV|syscall.MS_NOEXEC, ""); err != nil {
		return fmt.Errorf("mount /sys: %w", err)
	}

	// /dev: minimal device nodes. In production, container runtimes
	// create individual device nodes (null, zero, random, etc.)
	// rather than mounting the full devtmpfs.
	os.MkdirAll("/dev", 0755)
	if err := syscall.Mount("tmpfs", "/dev", "tmpfs",
		syscall.MS_NOSUID|syscall.MS_STRICTATIME, "mode=755,size=65536k"); err != nil {
		return fmt.Errorf("mount /dev: %w", err)
	}

	fmt.Println("[+] Root filesystem isolated with pivot_root")
	return nil
}

// runContainer launches the container init process with a fully
// isolated filesystem. This is called after clone(CLONE_NEWNS) has
// created a new mount namespace.
func runContainer() {
	// Use an Alpine Linux rootfs (extract with: mkdir rootfs && tar -C rootfs -xf alpine-rootfs.tar.gz)
	rootfsPath := "/var/lib/containers/alpine-rootfs"

	if err := setupRootfs(rootfsPath); err != nil {
		fmt.Fprintf(os.Stderr, "setupRootfs: %v\n", err)
		os.Exit(1)
	}

	// Execute a shell inside the isolated filesystem.
	fmt.Println("[+] Entering isolated filesystem...")
	syscall.Exec("/bin/sh", []string{"sh"}, []string{
		"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
		"HOME=/root",
		"TERM=xterm",
	})
}

func main() {
	if os.Getenv("_CONTAINER_CHILD") == "1" {
		runContainer()
		return
	}

	fmt.Println("[*] Creating container with isolated filesystem...")

	cmd := exec.Command("/proc/self/exe")
	cmd.Env = append(os.Environ(), "_CONTAINER_CHILD=1")
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWPID |
			syscall.CLONE_NEWNS |
			syscall.CLONE_NEWUTS,
	}

	if err := cmd.Run(); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
}
```

---

## 5.6 User Namespace & Rootless Containers

### 5.6.1 How User Namespaces Enable Rootless Containers

User namespaces are the key technology enabling **rootless containers** — running
containers without any root privileges on the host. Here's how it works:

```
Without user namespaces (traditional containers):
  Host: root → docker daemon (root) → container process (root)
  Risk: container escape = root on host!

With user namespaces (rootless containers):
  Host: arpan (UID 1000) → podman (UID 1000) → container process (UID 0 → mapped to 100000)
  Risk: container escape = UID 100000 on host (unprivileged!)
```

### 5.6.2 UID/GID Mapping In Detail

The kernel maintains a mapping table for each user namespace. When a process in a user
namespace performs an operation, the kernel translates UIDs through this table:

```
/proc/[pid]/uid_map format:
  <inside-id>  <outside-id>  <count>

Example: Map container root (UID 0) to host UID 100000, with 65536 UIDs:
  0 100000 65536

This means:
  Container UID 0     → Host UID 100000
  Container UID 1     → Host UID 100001
  Container UID 65535 → Host UID 165535

/proc/[pid]/gid_map uses the same format for group IDs.
```

**Writing UID/GID maps — the rules:**

1. You can only write to `uid_map`/`gid_map` **once** per namespace.
2. Before writing `gid_map`, you must write "deny" to `/proc/[pid]/setgroups` to
   prevent setgroups() from being used (security requirement).
3. Without the `newuidmap`/`newgidmap` setuid helpers, an unprivileged user can only
   map their own UID. The helpers consult `/etc/subuid` and `/etc/subgid` for
   allowed ranges.

### 5.6.3 /etc/subuid and /etc/subgid

These files define subordinate UID/GID ranges that each user is allowed to map:

```
# /etc/subuid
# Format: username:start:count
arpan:100000:65536
# User "arpan" can map UIDs 100000 through 165535

# /etc/subgid (same format)
arpan:100000:65536
```

The `newuidmap` and `newgidmap` setuid binaries read these files and perform the
mapping on behalf of the unprivileged user.

### 5.6.4 Hands-On: Creating a User Namespace with UID Mapping in Go

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
	"syscall"
)

// createUserNamespace creates a new process inside a user namespace with
// UID/GID mapping configured. The child process appears to be root (UID 0)
// inside the namespace but maps to an unprivileged UID on the host.
//
// This is the foundation of rootless containers as implemented by:
//   - Podman (rootless mode)
//   - Docker (rootless mode, experimental)
//   - Buildah
//   - Singularity/Apptainer
//
// The mapping is configured via SysProcAttr.UidMappings and GidMappings,
// which Go writes to /proc/[pid]/uid_map and /proc/[pid]/gid_map
// after creating the child process.
func createUserNamespace() error {
	// Re-execute ourselves inside the new user namespace.
	cmd := exec.Command("/proc/self/exe")
	cmd.Env = append(os.Environ(), "_IN_USER_NS=1")
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	// Get current user's UID and GID for the mapping.
	uid := os.Getuid()
	gid := os.Getgid()

	cmd.SysProcAttr = &syscall.SysProcAttr{
		// Create new User, PID, UTS, and Mount namespaces.
		// CLONE_NEWUSER is special: it can be used by unprivileged users!
		// This is what makes rootless containers possible.
		Cloneflags: syscall.CLONE_NEWUSER |
			syscall.CLONE_NEWPID |
			syscall.CLONE_NEWUTS |
			syscall.CLONE_NEWNS,

		// UidMappings defines the UID translation table.
		// ContainerID 0 (root inside) maps to HostID (our UID) for Size 1.
		//
		// This means: inside the namespace, UID 0 (root) is actually
		// our unprivileged UID on the host. The child process THINKS
		// it's root but has no real root privileges on the host.
		UidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,   // UID 0 inside namespace (root)
				HostID:      uid, // Maps to our real UID outside
				Size:        1,   // Only one UID mapped
			},
		},

		// GidMappings follows the same pattern for group IDs.
		GidMappings: []syscall.SysProcIDMap{
			{
				ContainerID: 0,   // GID 0 inside namespace (root)
				HostID:      gid, // Maps to our real GID outside
				Size:        1,   // Only one GID mapped
			},
		},
	}

	fmt.Printf("[parent] Creating user namespace (host UID %d → container UID 0)\n", uid)

	if err := cmd.Run(); err != nil {
		return fmt.Errorf("run in user namespace: %w", err)
	}

	return nil
}

// childInUserNamespace runs inside the user namespace. It demonstrates
// that it appears to be root (UID 0) despite running as an unprivileged
// user on the host.
func childInUserNamespace() {
	fmt.Printf("[child] PID: %d\n", os.Getpid())
	fmt.Printf("[child] UID: %d (should be 0 = root)\n", os.Getuid())
	fmt.Printf("[child] GID: %d (should be 0 = root)\n", os.Getgid())

	// Set hostname — this works because we have CAP_SYS_ADMIN
	// inside our user namespace (and we created a UTS namespace).
	if err := syscall.Sethostname([]byte("rootless-container")); err != nil {
		fmt.Printf("[child] sethostname failed: %v\n", err)
	} else {
		hostname, _ := os.Hostname()
		fmt.Printf("[child] Hostname: %s\n", hostname)
	}

	// Try to read a file only readable by root — this will fail
	// because our UID 0 inside the namespace maps to an unprivileged
	// UID on the host. User namespaces provide the ILLUSION of root
	// but not actual host root privileges.
	_, err := os.ReadFile("/etc/shadow")
	if err != nil {
		fmt.Printf("[child] Cannot read /etc/shadow (expected): %v\n", err)
	}

	fmt.Println("[child] Running shell as 'root' inside user namespace...")
	// Launch an interactive shell so we can explore.
	sh := exec.Command("/bin/sh")
	sh.Stdin = os.Stdin
	sh.Stdout = os.Stdout
	sh.Stderr = os.Stderr
	sh.Run()
}

func main() {
	if os.Getenv("_IN_USER_NS") == "1" {
		childInUserNamespace()
		return
	}

	if err := createUserNamespace(); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
}
```

### 5.6.5 Security Implications of User Namespaces

User namespaces are both a powerful security feature and a historically controversial
attack surface:

**Benefits:**
- Enable rootless containers (no setuid binaries needed)
- Limit blast radius of container escapes
- Allow unprivileged users to use features that normally require root

**Historical concerns:**
- New user namespace = new capability set, expanding kernel attack surface
- Several kernel exploits have used user namespaces as the first step
- Some distributions disable unprivileged user namespaces by default:
  ```bash
  # Check if unprivileged user namespaces are allowed:
  sysctl kernel.unprivileged_userns_clone
  # 0 = disabled, 1 = enabled
  ```

---
## 5.7 Cgroups v1 — Resource Control

### 5.7.1 What Are Cgroups?

**Control groups (cgroups)** are a Linux kernel mechanism for organizing processes
into hierarchical groups and applying resource limits, priorities, accounting, and
control to those groups.

If namespaces answer "what can a process **see**?", cgroups answer "what can a
process **use**?"

```
Namespaces vs. Cgroups:

  ┌─────────────────────────────────────────────────────┐
  │                    NAMESPACES                        │
  │                                                      │
  │  "What can this process SEE?"                        │
  │                                                      │
  │  ✓ Its own PID tree                                  │
  │  ✓ Its own network interfaces                        │
  │  ✓ Its own mount points                              │
  │  ✓ Its own hostname                                  │
  │  ✗ Other containers' processes                       │
  │  ✗ Other containers' network                         │
  └─────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────┐
  │                     CGROUPS                          │
  │                                                      │
  │  "What can this process USE?"                        │
  │                                                      │
  │  ✓ Max 512MB RAM                                     │
  │  ✓ Max 50% of one CPU core                           │
  │  ✓ Max 100MB/s disk I/O                              │
  │  ✓ Max 100 PIDs (fork bomb protection)               │
  │  ✗ Cannot exceed these limits                        │
  └─────────────────────────────────────────────────────┘
```

### 5.7.2 Cgroups v1 Architecture

Cgroups v1 uses **multiple independent hierarchies**, one per controller (subsystem):

```
Cgroups v1 Hierarchy (multiple trees, one per controller):

/sys/fs/cgroup/
├── cpu/                              ← CPU controller hierarchy
│   ├── docker/
│   │   ├── container-abc/
│   │   │   ├── cpu.shares            (relative CPU weight)
│   │   │   ├── cpu.cfs_quota_us      (hard CPU limit)
│   │   │   ├── cpu.cfs_period_us     (quota period)
│   │   │   ├── tasks                 (PIDs in this cgroup)
│   │   │   └── cgroup.procs          (TGIDs in this cgroup)
│   │   └── container-def/
│   │       └── ...
│   ├── cpu.shares
│   ├── tasks
│   └── cgroup.procs
│
├── memory/                           ← Memory controller hierarchy
│   ├── docker/
│   │   ├── container-abc/
│   │   │   ├── memory.limit_in_bytes (hard memory limit)
│   │   │   ├── memory.usage_in_bytes (current usage)
│   │   │   ├── memory.max_usage_in_bytes
│   │   │   ├── memory.oom_control    (OOM killer settings)
│   │   │   └── tasks
│   │   └── container-def/
│   │       └── ...
│   └── ...
│
├── pids/                             ← PID controller hierarchy
│   ├── docker/
│   │   ├── container-abc/
│   │   │   ├── pids.max             (max number of processes)
│   │   │   ├── pids.current         (current count)
│   │   │   └── tasks
│   │   └── ...
│   └── ...
│
├── blkio/                            ← Block I/O controller
├── cpuacct/                          ← CPU accounting
├── cpuset/                           ← CPU/memory node pinning
├── devices/                          ← Device access control
├── freezer/                          ← Freeze/thaw processes
├── hugetlb/                          ← Huge page limits
├── net_cls/                          ← Network packet classification
├── net_prio/                         ← Network priority
└── perf_event/                       ← Perf monitoring
```

### 5.7.3 Cgroups v1 Controllers Explained

**cpu** — CPU bandwidth control using CFS (Completely Fair Scheduler):
```bash
# Relative weight (shares) — proportional CPU time when competing
echo 1024 > cpu.shares    # Default is 1024
# A container with shares=512 gets half the CPU of one with shares=1024
# (but only when both are competing for CPU)

# Hard limit (CFS bandwidth) — absolute maximum CPU time
echo 50000 > cpu.cfs_quota_us     # 50ms out of every...
echo 100000 > cpu.cfs_period_us   # ...100ms period = 50% of one CPU
# quota/period = 50000/100000 = 0.5 CPUs
# For 2 CPUs: echo 200000 > cpu.cfs_quota_us
```

**memory** — Memory usage limits and accounting:
```bash
# Hard memory limit (OOM killer triggers when exceeded)
echo 536870912 > memory.limit_in_bytes    # 512 MB

# Soft memory limit (kernel reclaims memory when under pressure)
echo 268435456 > memory.soft_limit_in_bytes  # 256 MB

# Swap limit (total = memory + swap)
echo 1073741824 > memory.memsw.limit_in_bytes  # 1 GB total

# Disable OOM killer (container pauses instead of getting killed)
echo 1 > memory.oom_control

# Read current usage
cat memory.usage_in_bytes
cat memory.max_usage_in_bytes
```

**pids** — Limit number of processes (fork bomb protection):
```bash
echo 100 > pids.max        # Maximum 100 processes
cat pids.current            # Current number of processes
```

**blkio** — Block I/O throttling:
```bash
# Limit read bandwidth to 10MB/s on device 8:0 (sda)
echo "8:0 10485760" > blkio.throttle.read_bps_device

# Limit write IOPS to 100 on device 8:0
echo "8:0 100" > blkio.throttle.write_iops_device
```

**devices** — Device access control:
```bash
# Deny all device access
echo "a" > devices.deny

# Allow read/write to /dev/null (1:3), /dev/zero (1:5), etc.
echo "c 1:3 rwm" > devices.allow    # /dev/null
echo "c 1:5 rwm" > devices.allow    # /dev/zero
echo "c 1:8 rwm" > devices.allow    # /dev/random
echo "c 1:9 rwm" > devices.allow    # /dev/urandom
echo "c 5:0 rwm" > devices.allow    # /dev/tty
echo "c 5:2 rwm" > devices.allow    # /dev/ptmx
```

**freezer** — Freeze and thaw processes:
```bash
echo FROZEN > freezer.state     # Freeze all processes in cgroup
echo THAWED > freezer.state     # Unfreeze them
cat freezer.state               # FROZEN, FREEZING, or THAWED
```

### 5.7.4 Hands-On: Creating Cgroups and Setting Limits from Go

```go
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"strconv"
)

// CgroupV1Config defines the resource limits for a cgroup v1 hierarchy.
// Each field corresponds to a cgroup controller and the value to write
// to its control file.
//
// Zero values mean "no limit" — the corresponding control file is not written.
type CgroupV1Config struct {
	// Name is the cgroup name (becomes a directory under each controller).
	// Example: "mycontainer" creates /sys/fs/cgroup/memory/mycontainer/
	Name string

	// MemoryLimitBytes is the hard memory limit in bytes.
	// Written to: memory.limit_in_bytes
	// When a process exceeds this limit, the OOM killer is invoked.
	MemoryLimitBytes int64

	// MemorySoftLimitBytes is the soft memory limit in bytes.
	// Written to: memory.soft_limit_in_bytes
	// The kernel tries to reclaim memory when a cgroup exceeds this
	// limit and the system is under memory pressure.
	MemorySoftLimitBytes int64

	// CPUShares is the relative CPU weight (default 1024).
	// Written to: cpu.shares
	// Only meaningful when multiple cgroups compete for CPU time.
	CPUShares int64

	// CPUQuotaUS is the CPU bandwidth quota in microseconds.
	// Written to: cpu.cfs_quota_us
	// Combined with CPUPeriodUS, this sets an absolute CPU limit.
	// Example: 50000 with period 100000 = 50% of one CPU.
	CPUQuotaUS int64

	// CPUPeriodUS is the CFS scheduler period in microseconds.
	// Written to: cpu.cfs_period_us
	// Default: 100000 (100ms). Typical range: 1000-1000000.
	CPUPeriodUS int64

	// PidsMax is the maximum number of processes in this cgroup.
	// Written to: pids.max
	// Protects against fork bombs.
	PidsMax int64
}

// cgroupV1BasePath is the standard mount point for cgroup v1 hierarchies.
// Each controller is mounted as a separate filesystem under this path.
const cgroupV1BasePath = "/sys/fs/cgroup"

// writeFile is a helper that writes a value to a cgroup control file.
// Cgroup control files accept values written as text strings.
//
// Parameters:
//   - path: full path to the control file (e.g., /sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes)
//   - value: the value to write (e.g., "536870912" for 512MB)
//
// Returns an error if the write fails (e.g., invalid value, permission denied,
// or the cgroup controller rejected the value).
func writeFile(path string, value string) error {
	if err := os.WriteFile(path, []byte(value), 0644); err != nil {
		return fmt.Errorf("write %s=%s: %w", path, value, err)
	}
	fmt.Printf("  [cgroup] %s = %s\n", path, value)
	return nil
}

// CreateCgroupV1 creates a cgroup v1 hierarchy for the given configuration.
// It creates directories under each controller and writes the limit values
// to the corresponding control files.
//
// This function creates:
//   - /sys/fs/cgroup/memory/<name>/   (if MemoryLimitBytes > 0)
//   - /sys/fs/cgroup/cpu/<name>/      (if CPUShares > 0 or CPUQuotaUS > 0)
//   - /sys/fs/cgroup/pids/<name>/     (if PidsMax > 0)
//
// After creating the cgroup, call AddProcessToCgroupV1() to add a PID.
func CreateCgroupV1(cfg CgroupV1Config) error {
	fmt.Printf("[+] Creating cgroup v1: %s\n", cfg.Name)

	// Memory controller limits
	if cfg.MemoryLimitBytes > 0 {
		memDir := filepath.Join(cgroupV1BasePath, "memory", cfg.Name)
		if err := os.MkdirAll(memDir, 0755); err != nil {
			return fmt.Errorf("create memory cgroup: %w", err)
		}

		// Set hard memory limit
		if err := writeFile(
			filepath.Join(memDir, "memory.limit_in_bytes"),
			strconv.FormatInt(cfg.MemoryLimitBytes, 10),
		); err != nil {
			return err
		}

		// Set soft memory limit (if configured)
		if cfg.MemorySoftLimitBytes > 0 {
			if err := writeFile(
				filepath.Join(memDir, "memory.soft_limit_in_bytes"),
				strconv.FormatInt(cfg.MemorySoftLimitBytes, 10),
			); err != nil {
				return err
			}
		}

		// Disable swap to prevent the cgroup from using swap space.
		// This is equivalent to docker run --memory-swap=<same as memory>.
		writeFile(filepath.Join(memDir, "memory.swappiness"), "0")
	}

	// CPU controller limits
	if cfg.CPUShares > 0 || cfg.CPUQuotaUS > 0 {
		cpuDir := filepath.Join(cgroupV1BasePath, "cpu", cfg.Name)
		if err := os.MkdirAll(cpuDir, 0755); err != nil {
			return fmt.Errorf("create cpu cgroup: %w", err)
		}

		if cfg.CPUShares > 0 {
			if err := writeFile(
				filepath.Join(cpuDir, "cpu.shares"),
				strconv.FormatInt(cfg.CPUShares, 10),
			); err != nil {
				return err
			}
		}

		if cfg.CPUQuotaUS > 0 {
			period := cfg.CPUPeriodUS
			if period == 0 {
				period = 100000 // Default 100ms
			}
			if err := writeFile(
				filepath.Join(cpuDir, "cpu.cfs_period_us"),
				strconv.FormatInt(period, 10),
			); err != nil {
				return err
			}
			if err := writeFile(
				filepath.Join(cpuDir, "cpu.cfs_quota_us"),
				strconv.FormatInt(cfg.CPUQuotaUS, 10),
			); err != nil {
				return err
			}
		}
	}

	// PID controller limit
	if cfg.PidsMax > 0 {
		pidsDir := filepath.Join(cgroupV1BasePath, "pids", cfg.Name)
		if err := os.MkdirAll(pidsDir, 0755); err != nil {
			return fmt.Errorf("create pids cgroup: %w", err)
		}

		if err := writeFile(
			filepath.Join(pidsDir, "pids.max"),
			strconv.FormatInt(cfg.PidsMax, 10),
		); err != nil {
			return err
		}
	}

	return nil
}

// AddProcessToCgroupV1 adds a process to all controllers of a cgroup v1 group.
// This is done by writing the PID to the `cgroup.procs` file in each
// controller's directory.
//
// When a PID is written to cgroup.procs:
//   - The process and all its threads are moved to the cgroup
//   - The process is removed from its previous cgroup (for that controller)
//   - Child processes inherit the cgroup membership
//
// Parameters:
//   - name: the cgroup name (directory name under each controller)
//   - pid: the process ID to add
func AddProcessToCgroupV1(name string, pid int) error {
	pidStr := strconv.Itoa(pid)
	controllers := []string{"memory", "cpu", "pids"}

	for _, ctrl := range controllers {
		procsFile := filepath.Join(cgroupV1BasePath, ctrl, name, "cgroup.procs")
		if _, err := os.Stat(procsFile); err != nil {
			continue // Skip controllers that weren't created
		}
		if err := writeFile(procsFile, pidStr); err != nil {
			return fmt.Errorf("add PID %d to %s: %w", pid, ctrl, err)
		}
	}

	return nil
}

// CleanupCgroupV1 removes the cgroup directories. A cgroup directory can
// only be removed when it has no processes and no child cgroups.
//
// Parameters:
//   - name: the cgroup name to remove
func CleanupCgroupV1(name string) error {
	controllers := []string{"memory", "cpu", "pids"}
	for _, ctrl := range controllers {
		dir := filepath.Join(cgroupV1BasePath, ctrl, name)
		if err := os.Remove(dir); err != nil && !os.IsNotExist(err) {
			return fmt.Errorf("remove %s cgroup: %w", ctrl, err)
		}
	}
	fmt.Printf("[+] Cleaned up cgroup v1: %s\n", name)
	return nil
}

func main() {
	cgroupName := "chapter5-demo"

	// Create a cgroup with:
	//   - 256 MB memory limit
	//   - 50% of one CPU
	//   - Maximum 50 processes
	cfg := CgroupV1Config{
		Name:             cgroupName,
		MemoryLimitBytes: 256 * 1024 * 1024, // 256 MB
		CPUQuotaUS:       50000,             // 50ms per 100ms = 50% CPU
		CPUPeriodUS:      100000,            // 100ms period
		PidsMax:          50,                // Max 50 processes
	}

	if err := CreateCgroupV1(cfg); err != nil {
		fmt.Fprintf(os.Stderr, "Create cgroup: %v\n", err)
		os.Exit(1)
	}

	// Add current process to the cgroup (for demonstration).
	if err := AddProcessToCgroupV1(cgroupName, os.Getpid()); err != nil {
		fmt.Fprintf(os.Stderr, "Add process: %v\n", err)
	}

	fmt.Printf("\n[+] Process %d is now in cgroup %s\n", os.Getpid(), cgroupName)
	fmt.Println("[*] Try allocating > 256MB RAM — the OOM killer will intervene!")

	// In a real container runtime, you would:
	// 1. Create the cgroup
	// 2. Start the container process
	// 3. Add the container's PID to the cgroup
	// The process inherits the limits immediately.

	// Cleanup
	defer CleanupCgroupV1(cgroupName)
}
```

---
## 5.8 Cgroups v2 — The Unified Hierarchy

### 5.8.1 Why Cgroups v2 Was Needed

Cgroups v1 had several fundamental design problems that made it increasingly difficult
to manage in production:

**Problem 1: Multiple Hierarchies**
In v1, each controller has its own independent hierarchy. A process could be in
`/docker/abc` in the memory hierarchy but `/system/xyz` in the CPU hierarchy. This
made holistic resource management nearly impossible.

**Problem 2: Thread vs. Process Granularity**
v1 allowed individual threads to be in different cgroups via the `tasks` file. This
created complex bookkeeping and race conditions.

**Problem 3: Inconsistent Interfaces**
Each controller had its own file names, semantics, and behavior. There was no
standardized way to enable/disable controllers or delegate them.

**Problem 4: No Safe Delegation**
v1 had no clean mechanism for unprivileged delegation — letting a user manage their
own cgroup subtree without granting them control over the entire system.

```
Cgroups v1 (Multiple Hierarchies):          Cgroups v2 (Unified Hierarchy):

/sys/fs/cgroup/cpu/                         /sys/fs/cgroup/
├── docker/                                 └── system.slice/
│   ├── abc/                                    ├── docker-abc.scope/
│   └── def/                                    │   ├── cgroup.controllers
/sys/fs/cgroup/memory/                          │   ├── cpu.max
├── docker/                                     │   ├── memory.max
│   ├── abc/     ← Same process,                │   ├── pids.max
│   └── def/       different path!              │   ├── io.max
/sys/fs/cgroup/pids/                            │   └── cgroup.procs
├── docker/                                     └── docker-def.scope/
│   ├── abc/     ← And again!                       ├── cpu.max
│   └── def/                                        ├── memory.max
                                                    └── cgroup.procs
Problem: 3 separate trees                  Solution: ONE tree, all controllers
to manage for same container!              in the same directory!
```

### 5.8.2 Cgroups v2 Architecture

Cgroups v2 has a **single unified hierarchy** mounted at `/sys/fs/cgroup`. All
controllers are managed through the same tree.

Key concepts:

**1. Single hierarchy**: There is exactly one cgroup tree. All controllers attach to it.

**2. Top-down constraint**: A cgroup can only enable a controller if its parent has
enabled it. Resources flow top-down through the tree.

**3. No-internal-process rule**: A cgroup that has child cgroups cannot have its own
processes (except the root cgroup). This prevents resource accounting ambiguity.

**4. Subtree control**: Each cgroup has a `cgroup.subtree_control` file that determines
which controllers are available to its children.

```
Cgroups v2 File Structure:

/sys/fs/cgroup/                              ← Root cgroup
├── cgroup.controllers                       ← Available controllers (read-only)
│   → "cpu io memory pids"
├── cgroup.subtree_control                   ← Controllers enabled for children
│   → "+cpu +memory +pids +io"               (writable)
├── cgroup.procs                             ← PIDs in root cgroup
├── cgroup.type                              ← "domain" or "threaded"
│
├── system.slice/                            ← systemd creates this
│   ├── cgroup.controllers                   ← Inherited from parent
│   ├── cgroup.subtree_control
│   ├── cpu.max                              ← "max 100000" (no limit)
│   ├── cpu.weight                           ← 100 (default weight)
│   ├── memory.max                           ← "max" (no limit)
│   ├── memory.current                       ← Current usage (read-only)
│   ├── memory.high                          ← Throttling threshold
│   ├── memory.low                           ← Best-effort protection
│   ├── pids.max                             ← "max" (no limit)
│   ├── pids.current                         ← Current count (read-only)
│   ├── io.max                               ← Per-device I/O limits
│   └── cgroup.procs
│
└── user.slice/                              ← Per-user cgroups
    └── user-1000.slice/                     ← User UID 1000
        └── session-1.scope/                 ← Login session
            └── cgroup.procs
```

### 5.8.3 Cgroups v2 Controllers Deep Dive

#### Memory Controller

The v2 memory controller provides fine-grained memory management with four key knobs:

```
Memory Controller Knobs:

  ┌────────────────────────────────────────────────────────────┐
  │                                                            │
  │  memory.max ─────────── Hard limit (OOM kill above this)  │
  │  ═══════════════════════════════════════════ 512 MB       │
  │                                                            │
  │  memory.high ────────── Throttle threshold                │
  │  ════════════════════════════════════ 400 MB              │
  │  (Process is slowed down above this — reclaim pressure)   │
  │                                                            │
  │  memory.low ─────────── Best-effort memory protection     │
  │  ══════════════ 200 MB                                    │
  │  (Kernel avoids reclaiming below this if possible)        │
  │                                                            │
  │  memory.min ─────────── Hard memory protection            │
  │  ══════ 100 MB                                            │
  │  (Kernel guarantees this memory — never reclaimed)        │
  │                                                            │
  │  0 MB ─────────────────────────────────────────────────   │
  └────────────────────────────────────────────────────────────┘
```

```bash
# Hard limit: process is OOM-killed above this
echo 536870912 > memory.max           # 512 MB
# Or: echo "512M" > memory.max       (human-readable format in v2!)

# Throttling threshold: process is slowed (memory pressure) above this
echo 419430400 > memory.high          # 400 MB

# Protection: kernel tries not to reclaim memory below this
echo 209715200 > memory.low           # 200 MB

# Hard protection: guaranteed minimum memory
echo 104857600 > memory.min           # 100 MB

# Swap limit (separate from memory in v2!)
echo 0 > memory.swap.max             # Disable swap entirely

# Read current memory usage
cat memory.current                    # Current RSS usage in bytes
cat memory.swap.current               # Current swap usage in bytes

# Memory events (OOM kills, high threshold hits, etc.)
cat memory.events
# → low 0
# → high 42
# → max 1
# → oom 0
# → oom_kill 0
```

#### CPU Controller

```bash
# cpu.max: absolute bandwidth limit
# Format: "$MAX $PERIOD" (both in microseconds)
echo "50000 100000" > cpu.max         # 50ms per 100ms = 50% of one CPU
echo "200000 100000" > cpu.max        # 200ms per 100ms = 2 CPUs
echo "max 100000" > cpu.max           # No limit (default)

# cpu.weight: relative weight (replaces cpu.shares from v1)
# Range: 1-10000, default: 100
echo 100 > cpu.weight                 # Normal weight
echo 200 > cpu.weight                 # Twice the weight of default
echo 50 > cpu.weight                  # Half the weight of default
# Note: weight only matters when cgroups COMPETE for CPU time

# cpu.weight.nice: nice-value based weight (alternative to cpu.weight)
# Maps nice values (-20 to 19) to weights
echo 0 > cpu.weight.nice             # Nice 0 = normal priority

# Read CPU usage statistics
cat cpu.stat
# → usage_usec 123456789
# → user_usec 100000000
# → system_usec 23456789
# → nr_periods 1234
# → nr_throttled 56
# → throttled_usec 7890000
```

#### I/O Controller

```bash
# io.max: per-device bandwidth and IOPS limits
# Format: "MAJ:MIN rbps=BYTES wbps=BYTES riops=OPS wiops=OPS"
echo "8:0 rbps=10485760 wbps=5242880" > io.max    # sda: 10MB/s read, 5MB/s write
echo "8:0 riops=1000 wiops=500" > io.max           # sda: 1000 read IOPS, 500 write IOPS

# io.latency: latency-based I/O control
# Target latency in microseconds — kernel throttles other cgroups
# to ensure this cgroup meets its target.
echo "8:0 target=5000" > io.latency               # 5ms target latency

# io.weight: proportional I/O weight
# Range: 1-10000, default: 100
echo "default 100" > io.weight

# Read I/O statistics
cat io.stat
# → 8:0 rbytes=1234567 wbytes=7654321 rios=100 wios=50 dbytes=0 dios=0
```

#### PID Controller

```bash
# Limit maximum number of tasks (processes + threads)
echo 100 > pids.max

# Read current count
cat pids.current                     # Current number of tasks
cat pids.events
# → max 3                            # Number of times fork was denied
```

### 5.8.4 Delegation — Unprivileged Cgroup Management

Cgroups v2 introduces a clean delegation model. A privileged process can delegate
a subtree to an unprivileged user by:

1. Creating a cgroup directory
2. Setting ownership with `chown`
3. Enabling controllers via `cgroup.subtree_control`

```bash
# As root: create and delegate a cgroup subtree to user 1000
mkdir /sys/fs/cgroup/user.slice/user-1000.slice/delegate
chown -R 1000:1000 /sys/fs/cgroup/user.slice/user-1000.slice/delegate

# Enable controllers for the delegate subtree
echo "+cpu +memory +pids" > /sys/fs/cgroup/user.slice/user-1000.slice/cgroup.subtree_control

# Now user 1000 can create sub-cgroups and set limits within the delegate tree
# without any root privileges!
```

This is how systemd delegates cgroups to user sessions, and how rootless Podman
manages container resource limits.

### 5.8.5 Hands-On: Full Cgroup v2 Setup from Go

```go
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"strconv"
	"strings"
)

// CgroupV2Config defines resource limits for a cgroup v2 group.
// All limits are optional — zero/empty values mean "no limit."
//
// Cgroups v2 uses a unified hierarchy with all controllers in a
// single tree, unlike v1 which had separate hierarchies per controller.
type CgroupV2Config struct {
	// Name is the cgroup path relative to the cgroup v2 root.
	// Example: "myapp" creates /sys/fs/cgroup/myapp/
	// Example: "myapp/worker" creates /sys/fs/cgroup/myapp/worker/
	Name string

	// CPUMax is the CPU bandwidth limit in "quota period" format.
	// Written to: cpu.max
	// Example: "50000 100000" = 50% of one CPU
	// Example: "200000 100000" = 2 CPUs
	// Example: "max 100000" = unlimited (default)
	CPUMax string

	// CPUWeight is the relative CPU weight (1-10000, default 100).
	// Written to: cpu.weight
	// Only applies when multiple cgroups compete for CPU time.
	CPUWeight int

	// MemoryMax is the hard memory limit in bytes.
	// Written to: memory.max
	// The OOM killer is invoked when usage exceeds this limit.
	MemoryMax int64

	// MemoryHigh is the memory throttling threshold in bytes.
	// Written to: memory.high
	// Processes are throttled (slowed down) when usage exceeds this.
	// This provides a "soft" limit that avoids OOM kills.
	MemoryHigh int64

	// MemoryLow is the best-effort memory protection in bytes.
	// Written to: memory.low
	// The kernel tries to avoid reclaiming memory below this amount.
	MemoryLow int64

	// MemorySwapMax is the swap limit in bytes.
	// Written to: memory.swap.max
	// Set to 0 to disable swap usage for this cgroup.
	MemorySwapMax int64

	// PidsMax is the maximum number of processes (and threads).
	// Written to: pids.max
	PidsMax int64

	// IOMax is a map of "MAJ:MIN" device IDs to bandwidth limits.
	// Written to: io.max
	// Example: {"8:0": "rbps=10485760 wbps=5242880"}
	IOMax map[string]string
}

// cgroupV2Root is the mount point for the unified cgroup v2 hierarchy.
const cgroupV2Root = "/sys/fs/cgroup"

// writeCgroupFile writes a value to a cgroup v2 control file.
// Returns nil if the file doesn't exist (controller not enabled).
//
// All cgroup v2 control files accept text values. The kernel parses
// the text and applies the setting atomically.
func writeCgroupFile(cgroupPath, filename, value string) error {
	path := filepath.Join(cgroupPath, filename)

	// Check if the control file exists — it won't if the controller
	// isn't enabled for this cgroup.
	if _, err := os.Stat(path); os.IsNotExist(err) {
		return nil // Controller not enabled, skip silently
	}

	if err := os.WriteFile(path, []byte(value), 0644); err != nil {
		return fmt.Errorf("write %s = %q: %w", path, value, err)
	}
	fmt.Printf("  [cgroupv2] %s = %s\n", filename, value)
	return nil
}

// enableControllers enables the specified controllers for children
// of the given cgroup. This is the "top-down constraint" — a controller
// must be enabled at each level of the hierarchy.
//
// Parameters:
//   - cgroupPath: absolute path to the cgroup directory
//   - controllers: list of controller names (e.g., "cpu", "memory", "pids")
//
// This writes to cgroup.subtree_control, e.g., "+cpu +memory +pids".
// The controllers must already be listed in cgroup.controllers (available
// from the parent).
func enableControllers(cgroupPath string, controllers []string) error {
	// Build the "+controller" string for each controller.
	var enables []string
	for _, c := range controllers {
		enables = append(enables, "+"+c)
	}
	value := strings.Join(enables, " ")

	return writeCgroupFile(cgroupPath, "cgroup.subtree_control", value)
}

// CreateCgroupV2 creates a cgroup v2 group and configures its resource limits.
//
// The function:
//  1. Enables required controllers at the parent level
//  2. Creates the cgroup directory
//  3. Writes resource limit values to the control files
//
// Important: Due to the "no-internal-process" rule, a cgroup that has
// child cgroups cannot directly contain processes. This means you should
// either use leaf cgroups for processes or create a specific "leaf" child.
func CreateCgroupV2(cfg CgroupV2Config) error {
	cgroupPath := filepath.Join(cgroupV2Root, cfg.Name)

	// Determine which controllers we need to enable.
	var controllers []string
	if cfg.CPUMax != "" || cfg.CPUWeight > 0 {
		controllers = append(controllers, "cpu")
	}
	if cfg.MemoryMax > 0 || cfg.MemoryHigh > 0 {
		controllers = append(controllers, "memory")
	}
	if cfg.PidsMax > 0 {
		controllers = append(controllers, "pids")
	}
	if len(cfg.IOMax) > 0 {
		controllers = append(controllers, "io")
	}

	// Enable controllers at the parent level.
	// Walk up from our target path to enable controllers at each level.
	parentPath := filepath.Dir(cgroupPath)
	if parentPath != cgroupV2Root {
		// For nested cgroups, enable controllers at the grandparent too.
		grandparent := filepath.Dir(parentPath)
		enableControllers(grandparent, controllers)
	}
	enableControllers(parentPath, controllers)

	// Create the cgroup directory.
	fmt.Printf("[+] Creating cgroup v2: %s\n", cfg.Name)
	if err := os.MkdirAll(cgroupPath, 0755); err != nil {
		return fmt.Errorf("create cgroup dir: %w", err)
	}

	// Write CPU limits.
	if cfg.CPUMax != "" {
		if err := writeCgroupFile(cgroupPath, "cpu.max", cfg.CPUMax); err != nil {
			return err
		}
	}
	if cfg.CPUWeight > 0 {
		if err := writeCgroupFile(cgroupPath, "cpu.weight",
			strconv.Itoa(cfg.CPUWeight)); err != nil {
			return err
		}
	}

	// Write memory limits.
	if cfg.MemoryMax > 0 {
		if err := writeCgroupFile(cgroupPath, "memory.max",
			strconv.FormatInt(cfg.MemoryMax, 10)); err != nil {
			return err
		}
	}
	if cfg.MemoryHigh > 0 {
		if err := writeCgroupFile(cgroupPath, "memory.high",
			strconv.FormatInt(cfg.MemoryHigh, 10)); err != nil {
			return err
		}
	}
	if cfg.MemoryLow > 0 {
		if err := writeCgroupFile(cgroupPath, "memory.low",
			strconv.FormatInt(cfg.MemoryLow, 10)); err != nil {
			return err
		}
	}
	if cfg.MemorySwapMax >= 0 && cfg.MemoryMax > 0 {
		if err := writeCgroupFile(cgroupPath, "memory.swap.max",
			strconv.FormatInt(cfg.MemorySwapMax, 10)); err != nil {
			return err
		}
	}

	// Write PID limit.
	if cfg.PidsMax > 0 {
		if err := writeCgroupFile(cgroupPath, "pids.max",
			strconv.FormatInt(cfg.PidsMax, 10)); err != nil {
			return err
		}
	}

	// Write I/O limits.
	for device, limit := range cfg.IOMax {
		value := fmt.Sprintf("%s %s", device, limit)
		if err := writeCgroupFile(cgroupPath, "io.max", value); err != nil {
			return err
		}
	}

	return nil
}

// AddProcessToCgroupV2 adds a process to a cgroup v2 group by writing
// its PID to the cgroup.procs file.
//
// The process is moved atomically — it leaves its current cgroup and
// enters the new one. All threads of the process move together (unlike
// cgroups v1 which allowed per-thread placement).
func AddProcessToCgroupV2(name string, pid int) error {
	procsFile := filepath.Join(cgroupV2Root, name, "cgroup.procs")
	return writeCgroupFile(filepath.Dir(procsFile), "cgroup.procs",
		strconv.Itoa(pid))
}

// ReadCgroupStats reads common statistics from a cgroup v2 group.
// These files are read-only and maintained by the kernel.
func ReadCgroupStats(name string) {
	cgroupPath := filepath.Join(cgroupV2Root, name)

	fmt.Printf("\n=== Cgroup Stats: %s ===\n", name)

	// Memory statistics
	readAndPrint := func(file, label string) {
		data, err := os.ReadFile(filepath.Join(cgroupPath, file))
		if err == nil {
			fmt.Printf("  %-25s %s", label, string(data))
		}
	}

	readAndPrint("memory.current", "Memory usage:")
	readAndPrint("memory.max", "Memory limit:")
	readAndPrint("cpu.stat", "CPU stats:")
	readAndPrint("pids.current", "PIDs current:")
	readAndPrint("pids.max", "PIDs max:")
	readAndPrint("io.stat", "I/O stats:")
}

func main() {
	cgroupName := "ch5-demo"

	// Create a cgroup with comprehensive resource limits.
	cfg := CgroupV2Config{
		Name:          cgroupName,
		CPUMax:        "50000 100000",    // 50% of one CPU
		CPUWeight:     100,               // Default weight
		MemoryMax:     256 * 1024 * 1024, // 256 MB hard limit
		MemoryHigh:    200 * 1024 * 1024, // 200 MB throttle point
		MemoryLow:     64 * 1024 * 1024,  // 64 MB protection
		MemorySwapMax: 0,                 // No swap
		PidsMax:       50,                // Max 50 processes
	}

	if err := CreateCgroupV2(cfg); err != nil {
		fmt.Fprintf(os.Stderr, "Error creating cgroup: %v\n", err)
		os.Exit(1)
	}

	// Add ourselves to the cgroup for demonstration.
	if err := AddProcessToCgroupV2(cgroupName, os.Getpid()); err != nil {
		fmt.Fprintf(os.Stderr, "Error adding process: %v\n", err)
	}

	// Read and display stats.
	ReadCgroupStats(cgroupName)

	fmt.Println("\n[+] Cgroup v2 setup complete!")
	fmt.Println("[*] This process is now resource-limited.")
}
```

### 5.8.6 Hands-On: Resource-Limited Process Launcher

This tool launches any command with cgroup v2 resource limits — a simplified version
of `systemd-run --scope`:

```go
package main

import (
	"flag"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"strconv"
	"strings"
	"syscall"
	"time"
)

// ResourceLimits holds the resource constraints for a launched process.
// These map directly to cgroup v2 control file values.
type ResourceLimits struct {
	// MemoryMax in human-readable format: "256M", "1G", etc.
	// Parsed and written to memory.max in bytes.
	MemoryMax string

	// CPUPercent is the CPU limit as a percentage of one core.
	// 50 = 50% of one core, 200 = two full cores.
	// Converted to cpu.max format: "$QUOTA $PERIOD".
	CPUPercent int

	// MaxPIDs is the maximum number of processes and threads.
	// Written to pids.max.
	MaxPIDs int
}

// parseMemory converts human-readable memory strings to bytes.
//
// Supported formats:
//   - "256M" or "256m" → 256 * 1024 * 1024
//   - "1G" or "1g"     → 1024 * 1024 * 1024
//   - "512K" or "512k" → 512 * 1024
//   - "1234"           → 1234 (raw bytes)
func parseMemory(s string) (int64, error) {
	s = strings.TrimSpace(s)
	if s == "" || s == "max" {
		return 0, nil
	}

	multiplier := int64(1)
	suffix := s[len(s)-1]
	switch suffix {
	case 'K', 'k':
		multiplier = 1024
		s = s[:len(s)-1]
	case 'M', 'm':
		multiplier = 1024 * 1024
		s = s[:len(s)-1]
	case 'G', 'g':
		multiplier = 1024 * 1024 * 1024
		s = s[:len(s)-1]
	}

	val, err := strconv.ParseInt(s, 10, 64)
	if err != nil {
		return 0, fmt.Errorf("parse memory %q: %w", s, err)
	}
	return val * multiplier, nil
}

// launchWithLimits creates a cgroup v2 group, applies resource limits,
// and launches the specified command within that cgroup.
//
// This function demonstrates the complete lifecycle of cgroup-based
// resource control:
//  1. Generate a unique cgroup name
//  2. Create the cgroup directory
//  3. Enable required controllers at the parent level
//  4. Write resource limit values
//  5. Fork the child process
//  6. Move the child into the cgroup
//  7. Wait for the child to exit
//  8. Clean up the cgroup
//
// Parameters:
//   - limits: the resource constraints to apply
//   - args: the command and arguments to execute
func launchWithLimits(limits ResourceLimits, args []string) error {
	// Generate a unique cgroup name using a timestamp.
	cgroupName := fmt.Sprintf("rlimit-%d", time.Now().UnixNano())
	cgroupPath := filepath.Join("/sys/fs/cgroup", cgroupName)

	fmt.Printf("[*] Creating cgroup: %s\n", cgroupName)

	// Create the cgroup directory.
	if err := os.MkdirAll(cgroupPath, 0755); err != nil {
		return fmt.Errorf("mkdir cgroup: %w", err)
	}
	// Ensure cleanup on exit.
	defer func() {
		os.Remove(cgroupPath)
		fmt.Printf("[*] Cleaned up cgroup: %s\n", cgroupName)
	}()

	// Enable controllers at the root level.
	var controllers []string
	if limits.MemoryMax != "" {
		controllers = append(controllers, "memory")
	}
	if limits.CPUPercent > 0 {
		controllers = append(controllers, "cpu")
	}
	if limits.MaxPIDs > 0 {
		controllers = append(controllers, "pids")
	}

	if len(controllers) > 0 {
		enables := make([]string, len(controllers))
		for i, c := range controllers {
			enables[i] = "+" + c
		}
		os.WriteFile(
			filepath.Join("/sys/fs/cgroup", "cgroup.subtree_control"),
			[]byte(strings.Join(enables, " ")),
			0644,
		)
	}

	// Apply memory limit.
	if limits.MemoryMax != "" {
		memBytes, err := parseMemory(limits.MemoryMax)
		if err != nil {
			return err
		}
		if memBytes > 0 {
			os.WriteFile(
				filepath.Join(cgroupPath, "memory.max"),
				[]byte(strconv.FormatInt(memBytes, 10)),
				0644,
			)
			fmt.Printf("[+] Memory limit: %s (%d bytes)\n", limits.MemoryMax, memBytes)
		}
	}

	// Apply CPU limit.
	// Convert percentage to CFS quota: percentage% of 100ms period.
	if limits.CPUPercent > 0 {
		period := 100000 // 100ms in microseconds
		quota := limits.CPUPercent * period / 100
		cpuMax := fmt.Sprintf("%d %d", quota, period)
		os.WriteFile(
			filepath.Join(cgroupPath, "cpu.max"),
			[]byte(cpuMax),
			0644,
		)
		fmt.Printf("[+] CPU limit: %d%% (quota=%dµs period=%dµs)\n",
			limits.CPUPercent, quota, period)
	}

	// Apply PID limit.
	if limits.MaxPIDs > 0 {
		os.WriteFile(
			filepath.Join(cgroupPath, "pids.max"),
			[]byte(strconv.Itoa(limits.MaxPIDs)),
			0644,
		)
		fmt.Printf("[+] PID limit: %d\n", limits.MaxPIDs)
	}

	// Start the child process.
	cmd := exec.Command(args[0], args[1:]...)
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Start(); err != nil {
		return fmt.Errorf("start command: %w", err)
	}

	// Move the child process into our cgroup.
	// Writing the PID to cgroup.procs atomically moves the process.
	pid := cmd.Process.Pid
	os.WriteFile(
		filepath.Join(cgroupPath, "cgroup.procs"),
		[]byte(strconv.Itoa(pid)),
		0644,
	)
	fmt.Printf("[+] Process %d added to cgroup %s\n", pid, cgroupName)

	// Wait for the process to complete.
	err := cmd.Wait()
	if err != nil {
		if exitErr, ok := err.(*exec.ExitError); ok {
			ws := exitErr.Sys().(syscall.WaitStatus)
			if ws.Signaled() {
				fmt.Printf("[!] Process killed by signal: %v\n", ws.Signal())
				if ws.Signal() == syscall.SIGKILL {
					fmt.Println("[!] Likely OOM-killed — exceeded memory.max!")
				}
			}
		}
		return err
	}

	return nil
}

func main() {
	// Parse command-line flags for resource limits.
	memLimit := flag.String("memory", "", "Memory limit (e.g., 256M, 1G)")
	cpuLimit := flag.Int("cpu", 0, "CPU limit as percentage of one core (e.g., 50)")
	pidLimit := flag.Int("pids", 0, "Maximum number of processes")
	flag.Parse()

	args := flag.Args()
	if len(args) == 0 {
		fmt.Fprintf(os.Stderr, "Usage: %s [--memory 256M] [--cpu 50] [--pids 100] -- <command> [args...]\n",
			os.Args[0])
		os.Exit(1)
	}

	limits := ResourceLimits{
		MemoryMax:  *memLimit,
		CPUPercent: *cpuLimit,
		MaxPIDs:    *pidLimit,
	}

	fmt.Printf("=== Resource-Limited Process Launcher ===\n")
	fmt.Printf("Command: %s\n", strings.Join(args, " "))

	if err := launchWithLimits(limits, args); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
}
```

Usage:
```bash
# Run a process with 256MB memory, 50% CPU, and max 20 PIDs
$ sudo go run rlimit.go --memory 256M --cpu 50 --pids 20 -- stress --vm 1 --vm-bytes 512M --timeout 10s
[*] Creating cgroup: rlimit-1234567890
[+] Memory limit: 256M (268435456 bytes)
[+] CPU limit: 50% (quota=50000µs period=100000µs)
[+] PID limit: 20
[+] Process 12345 added to cgroup rlimit-1234567890
[!] Process killed by signal: killed
[!] Likely OOM-killed — exceeded memory.max!
```

---
## 5.9 Combining Namespaces & Cgroups — The Container Foundation

### 5.9.1 This Is What a Container Runtime Does

A container is simply a process (or group of processes) that runs with:
1. **Namespaces** — for isolation (what the process can see)
2. **Cgroups** — for resource limits (what the process can use)
3. **A root filesystem** — an isolated filesystem view (overlay, bind mount, etc.)
4. **Security policies** — seccomp, AppArmor, SELinux, capabilities

```
What "docker run --memory 256m --cpus 0.5 -p 8080:80 nginx" does:

  1. Pull & extract image layers → overlay filesystem
  2. Create namespaces:
     ├── PID namespace    (nginx gets PID 1)
     ├── Network namespace (own eth0, routing)
     ├── Mount namespace   (pivot_root to overlay rootfs)
     ├── UTS namespace     (container hostname)
     ├── IPC namespace     (isolated IPC)
     └── Cgroup namespace  (virtualized /sys/fs/cgroup)
  3. Create cgroup:
     ├── memory.max = 268435456  (256 MB)
     ├── cpu.max = "50000 100000" (50% CPU)
     └── pids.max = 4096
  4. Set up networking:
     ├── Create veth pair
     ├── Attach to docker0 bridge
     ├── Assign IP address
     └── Set up port mapping (iptables DNAT 8080→80)
  5. Apply security:
     ├── Drop unnecessary capabilities
     ├── Apply seccomp profile (block dangerous syscalls)
     └── Apply AppArmor/SELinux profile
  6. pivot_root to overlay filesystem
  7. exec() nginx → becomes PID 1 in namespace
```

### 5.9.2 Hands-On: Minimal Container — Namespaces + Cgroups Combined

This is the capstone example of the chapter. We create a fully isolated process
with namespaces for isolation and cgroups for resource control:

```go
package main

import (
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"strconv"
	"syscall"
)

// ContainerConfig holds all configuration for our minimal container.
// This struct combines namespace isolation settings with cgroup
// resource limits — the two pillars of container technology.
type ContainerConfig struct {
	// Hostname is the container's hostname (set via UTS namespace).
	Hostname string

	// RootFS is the path to the container's root filesystem.
	// This should be a directory containing a complete Linux rootfs
	// (e.g., extracted from an Alpine Docker image).
	RootFS string

	// Command is the entrypoint command to run inside the container.
	Command []string

	// MemoryLimit is the hard memory limit in bytes.
	MemoryLimit int64

	// CPUQuota is the CPU limit as "quota period" for cpu.max.
	// Example: "50000 100000" for 50% of one CPU.
	CPUQuota string

	// PidsLimit is the maximum number of processes.
	PidsLimit int64

	// Env is the environment variables for the container process.
	Env []string
}

// containerInit is the code that runs as PID 1 inside the container.
// It is called after clone() creates the new namespaces.
//
// Responsibilities:
//  1. Set up the isolated filesystem (pivot_root)
//  2. Mount essential filesystems (/proc, /sys, /dev)
//  3. Set hostname
//  4. Drop to the container's entrypoint command
//
// This function implements the "container init" phase of the OCI
// runtime specification — the same steps runc performs between
// creating namespaces and exec'ing the user's command.
func containerInit(cfg ContainerConfig) error {
	fmt.Printf("[container] PID %d starting initialization...\n", os.Getpid())

	// --- Filesystem Isolation ---

	// Make all mounts private to prevent propagation to host.
	if err := syscall.Mount("", "/", "", syscall.MS_REC|syscall.MS_PRIVATE, ""); err != nil {
		return fmt.Errorf("make root private: %w", err)
	}

	// Bind-mount rootfs onto itself (required by pivot_root).
	if err := syscall.Mount(cfg.RootFS, cfg.RootFS, "", syscall.MS_BIND|syscall.MS_REC, ""); err != nil {
		return fmt.Errorf("bind mount rootfs: %w", err)
	}

	// Create and execute pivot_root.
	putOld := filepath.Join(cfg.RootFS, ".old_root")
	os.MkdirAll(putOld, 0700)

	if err := syscall.PivotRoot(cfg.RootFS, putOld); err != nil {
		return fmt.Errorf("pivot_root: %w", err)
	}

	if err := os.Chdir("/"); err != nil {
		return fmt.Errorf("chdir: %w", err)
	}

	// Unmount old root — completely severs access to host filesystem.
	if err := syscall.Unmount("/.old_root", syscall.MNT_DETACH); err != nil {
		return fmt.Errorf("unmount old root: %w", err)
	}
	os.Remove("/.old_root")

	// --- Mount Essential Filesystems ---

	// Mount /proc (PID-namespace-aware — only shows our processes).
	os.MkdirAll("/proc", 0755)
	syscall.Mount("proc", "/proc", "proc", 0, "")

	// Mount /sys (read-only for security).
	os.MkdirAll("/sys", 0755)
	syscall.Mount("sysfs", "/sys", "sysfs",
		syscall.MS_RDONLY|syscall.MS_NOSUID|syscall.MS_NODEV|syscall.MS_NOEXEC, "")

	// Mount tmpfs on /dev for device nodes.
	os.MkdirAll("/dev", 0755)
	syscall.Mount("tmpfs", "/dev", "tmpfs",
		syscall.MS_NOSUID|syscall.MS_STRICTATIME, "mode=755,size=65536k")

	// Create essential device nodes.
	// In a full container runtime, these are created with mknod().
	// For simplicity, we mount /dev/null etc. from the host.
	devNodes := map[string]string{
		"/dev/null":    "/dev/null",
		"/dev/zero":    "/dev/zero",
		"/dev/random":  "/dev/random",
		"/dev/urandom": "/dev/urandom",
	}
	for containerDev, hostDev := range devNodes {
		f, _ := os.Create(containerDev)
		if f != nil {
			f.Close()
		}
		syscall.Mount(hostDev, containerDev, "", syscall.MS_BIND, "")
	}

	// Mount /dev/pts for pseudo-terminals.
	os.MkdirAll("/dev/pts", 0755)
	syscall.Mount("devpts", "/dev/pts", "devpts",
		syscall.MS_NOSUID|syscall.MS_NOEXEC, "newinstance,ptmxmode=0666,mode=0620")

	// --- Set Hostname ---
	if cfg.Hostname != "" {
		syscall.Sethostname([]byte(cfg.Hostname))
	}

	fmt.Printf("[container] Filesystem and hostname configured.\n")
	fmt.Printf("[container] Executing: %v\n", cfg.Command)

	// --- Execute the Container Entrypoint ---
	// syscall.Exec replaces the current process image with the
	// specified command. After this call, this Go code is gone —
	// the container's entrypoint is now PID 1.
	return syscall.Exec(cfg.Command[0], cfg.Command, cfg.Env)
}

// setupCgroup creates a cgroup v2 group and returns a cleanup function.
//
// This function encapsulates the cgroup setup that container runtimes
// perform before starting the container process. The cgroup is created
// BEFORE the container starts so the process can be added immediately.
func setupCgroup(name string, cfg ContainerConfig) (cleanup func(), err error) {
	cgroupPath := filepath.Join("/sys/fs/cgroup", name)

	// Create the cgroup directory.
	if err := os.MkdirAll(cgroupPath, 0755); err != nil {
		return nil, fmt.Errorf("create cgroup: %w", err)
	}

	// Enable controllers at root level.
	os.WriteFile(
		"/sys/fs/cgroup/cgroup.subtree_control",
		[]byte("+cpu +memory +pids"),
		0644,
	)

	// Write resource limits.
	if cfg.MemoryLimit > 0 {
		os.WriteFile(
			filepath.Join(cgroupPath, "memory.max"),
			[]byte(strconv.FormatInt(cfg.MemoryLimit, 10)),
			0644,
		)
		// Also set swap to 0 to prevent swapping.
		os.WriteFile(
			filepath.Join(cgroupPath, "memory.swap.max"),
			[]byte("0"),
			0644,
		)
		fmt.Printf("[cgroup] memory.max = %d bytes\n", cfg.MemoryLimit)
	}

	if cfg.CPUQuota != "" {
		os.WriteFile(
			filepath.Join(cgroupPath, "cpu.max"),
			[]byte(cfg.CPUQuota),
			0644,
		)
		fmt.Printf("[cgroup] cpu.max = %s\n", cfg.CPUQuota)
	}

	if cfg.PidsLimit > 0 {
		os.WriteFile(
			filepath.Join(cgroupPath, "pids.max"),
			[]byte(strconv.FormatInt(cfg.PidsLimit, 10)),
			0644,
		)
		fmt.Printf("[cgroup] pids.max = %d\n", cfg.PidsLimit)
	}

	// Return a cleanup function that removes the cgroup.
	cleanup = func() {
		os.Remove(cgroupPath)
		fmt.Printf("[cgroup] Cleaned up: %s\n", name)
	}

	return cleanup, nil
}

func main() {
	// Check if we're the child (container init) or parent.
	if os.Getenv("_MINI_CONTAINER_INIT") == "1" {
		cfg := ContainerConfig{
			Hostname: os.Getenv("_CONTAINER_HOSTNAME"),
			RootFS:   os.Getenv("_CONTAINER_ROOTFS"),
			Command:  []string{"/bin/sh"},
			Env: []string{
				"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
				"HOME=/root",
				"TERM=xterm-256color",
			},
		}

		if err := containerInit(cfg); err != nil {
			fmt.Fprintf(os.Stderr, "[container] Init failed: %v\n", err)
			os.Exit(1)
		}
		return
	}

	// --- Parent: Orchestrate container creation ---

	cfg := ContainerConfig{
		Hostname:    "mini-container",
		RootFS:      "/var/lib/containers/alpine-rootfs",
		Command:     []string{"/bin/sh"},
		MemoryLimit: 256 * 1024 * 1024, // 256 MB
		CPUQuota:    "50000 100000",    // 50% CPU
		PidsLimit:   64,                // Max 64 processes
	}

	fmt.Println("╔═══════════════════════════════════════════════════════╗")
	fmt.Println("║      Mini Container Runtime — Chapter 5 Demo         ║")
	fmt.Println("╚═══════════════════════════════════════════════════════╝")
	fmt.Printf("  Hostname:  %s\n", cfg.Hostname)
	fmt.Printf("  RootFS:    %s\n", cfg.RootFS)
	fmt.Printf("  Memory:    %d MB\n", cfg.MemoryLimit/1024/1024)
	fmt.Printf("  CPU:       %s\n", cfg.CPUQuota)
	fmt.Printf("  PIDs:      %d\n", cfg.PidsLimit)
	fmt.Println()

	// Step 1: Create cgroup.
	cgroupName := "mini-container"
	cleanupCgroup, err := setupCgroup(cgroupName, cfg)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Setup cgroup: %v\n", err)
		os.Exit(1)
	}
	defer cleanupCgroup()

	// Step 2: Create the container process in new namespaces.
	cmd := exec.Command("/proc/self/exe")
	cmd.Env = []string{
		"_MINI_CONTAINER_INIT=1",
		"_CONTAINER_HOSTNAME=" + cfg.Hostname,
		"_CONTAINER_ROOTFS=" + cfg.RootFS,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	// Set ALL namespace flags — full isolation.
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWPID | // Isolated PIDs
			syscall.CLONE_NEWNS | // Isolated mounts
			syscall.CLONE_NEWUTS | // Isolated hostname
			syscall.CLONE_NEWIPC | // Isolated IPC
			syscall.CLONE_NEWNET | // Isolated network
			0x02000000, // CLONE_NEWCGROUP
	}

	if err := cmd.Start(); err != nil {
		fmt.Fprintf(os.Stderr, "Start container: %v\n", err)
		os.Exit(1)
	}

	// Step 3: Add the container process to the cgroup.
	// This MUST happen after Start() so we have the PID,
	// but the process is paused (hasn't exec'd yet) so limits
	// are in effect before any user code runs.
	pid := cmd.Process.Pid
	cgroupProcs := filepath.Join("/sys/fs/cgroup", cgroupName, "cgroup.procs")
	os.WriteFile(cgroupProcs, []byte(strconv.Itoa(pid)), 0644)
	fmt.Printf("[parent] Container PID %d added to cgroup %s\n", pid, cgroupName)

	// Step 4: Wait for the container to exit.
	if err := cmd.Wait(); err != nil {
		fmt.Fprintf(os.Stderr, "[parent] Container exited: %v\n", err)
	}

	fmt.Println("[parent] Container terminated. Cleaning up...")
}
```

This is the **foundation** for Chapter 8's container runtime. The differences between
this code and a production runtime like `runc` are:
- Error handling and edge cases
- OCI spec compliance (config.json parsing)
- Seccomp and capability dropping
- Proper console/TTY setup
- Rootless mode support
- Pre-start and post-stop hooks

---

## 5.10 systemd and Cgroups

### 5.10.1 How systemd Manages Cgroups

systemd is the default init system on most modern Linux distributions, and it takes
full ownership of the cgroup hierarchy. Understanding systemd's cgroup management
is essential because:

1. All container runtimes must cooperate with systemd's cgroup management
2. Kubernetes uses systemd as its cgroup driver
3. systemd provides the delegation mechanism for rootless containers

```
systemd Cgroup Hierarchy (cgroups v2):

/sys/fs/cgroup/
├── init.scope/                          ← systemd (PID 1) itself
│   └── cgroup.procs → 1
│
├── system.slice/                        ← System services
│   ├── sshd.service/                    ← Each service gets a cgroup
│   │   ├── cgroup.procs → 1234
│   │   ├── memory.max → max
│   │   └── cpu.weight → 100
│   ├── docker.service/
│   │   └── cgroup.procs → 5678
│   ├── nginx.service/
│   │   └── cgroup.procs → 9012
│   └── ...
│
├── user.slice/                          ← User sessions
│   ├── user-1000.slice/                 ← User UID 1000
│   │   ├── session-1.scope/             ← Login session
│   │   │   └── cgroup.procs → 3456
│   │   └── user@1000.service/           ← User's systemd instance
│   │       └── ...
│   └── user-0.slice/                    ← Root user
│       └── ...
│
└── machine.slice/                       ← VMs and containers
    ├── docker-abc123.scope/             ← Docker container
    └── libpod-def456.scope/             ← Podman container
```

**systemd unit types and cgroups:**

| Unit Type  | Cgroup Behavior                                         |
|------------|---------------------------------------------------------|
| `.service` | Creates a cgroup for the service's processes            |
| `.scope`   | Groups externally-started processes (e.g., containers)  |
| `.slice`   | Hierarchical grouping for resource control              |

### 5.10.2 systemd Resource Control

systemd provides a high-level interface for cgroup resource control through unit
file directives:

```ini
# /etc/systemd/system/myapp.service
[Service]
ExecStart=/usr/bin/myapp

# Memory limits (maps to memory.max)
MemoryMax=512M
# Memory throttling (maps to memory.high)
MemoryHigh=400M

# CPU limits (maps to cpu.max)
CPUQuota=50%
# CPU weight (maps to cpu.weight)
CPUWeight=100

# PID limit (maps to pids.max)
TasksMax=100

# I/O limits (maps to io.max)
IOReadBandwidthMax=/dev/sda 10M
IOWriteBandwidthMax=/dev/sda 5M

# I/O weight (maps to io.weight)
IOWeight=100
```

### 5.10.3 Useful systemd Cgroup Tools

```bash
# --- systemd-cgls: Display the cgroup hierarchy as a tree ---
$ systemd-cgls
Control group /:
-.slice
├─init.scope
│ └─1 /sbin/init
├─system.slice
│ ├─sshd.service
│ │ └─1234 sshd: /usr/sbin/sshd -D
│ ├─docker.service
│ │ └─5678 /usr/bin/dockerd
│ └─nginx.service
│   ├─9012 nginx: master process
│   └─9013 nginx: worker process
└─user.slice
  └─user-1000.slice
    └─session-1.scope
      ├─3456 -bash
      └─3457 systemd-cgls

# --- systemd-cgtop: Real-time cgroup resource usage (like top) ---
$ systemd-cgtop
Control Group                Tasks   %CPU   Memory  Input/s Output/s
/                               89    3.2   1.2G        -       -
/system.slice                   35    2.1   800M        -       -
/system.slice/docker.service     8    1.5   500M        -       -
/system.slice/nginx.service      3    0.4   120M        -       -
/user.slice                     12    0.3   200M        -       -

# --- systemd-run: Run a transient unit with resource limits ---
$ systemd-run --scope --property=MemoryMax=256M --property=CPUQuota=50% \
    stress --vm 1 --vm-bytes 512M --timeout 10s
# This creates a temporary cgroup scope with the specified limits.

# --- Show cgroup of a service ---
$ systemctl show docker.service --property=ControlGroup
ControlGroup=/system.slice/docker.service

# --- Show resource usage of a service ---
$ systemctl show docker.service --property=MemoryCurrent,CPUUsageNSec
MemoryCurrent=524288000
CPUUsageNSec=12345678901234
```

### 5.10.4 systemd Slice Hierarchy

Slices provide hierarchical resource partitioning. Think of them as "cgroup budget
envelopes" — a parent slice's limits cap the total for all children:

```
Resource Distribution with Slices:

  Total System: 4 CPUs, 16 GB RAM
  ─────────────────────────────────────────────────────
  ┌─ system.slice (CPUWeight=100, MemoryMax=8G) ──────┐
  │                                                     │
  │  ┌─ docker.service (CPUQuota=200%, Mem=4G) ──┐     │
  │  └───────────────────────────────────────────┘     │
  │  ┌─ nginx.service (CPUQuota=100%, Mem=2G) ───┐     │
  │  └───────────────────────────────────────────┘     │
  │  ┌─ postgres.service (CPUQuota=100%, Mem=2G) ┐     │
  │  └───────────────────────────────────────────┘     │
  └─────────────────────────────────────────────────────┘

  ┌─ user.slice (CPUWeight=100, MemoryMax=4G) ────────┐
  │                                                     │
  │  ┌─ user-1000.slice (MemoryMax=2G) ───────────┐    │
  │  │  session-1.scope, session-2.scope           │    │
  │  └─────────────────────────────────────────────┘    │
  │  ┌─ user-1001.slice (MemoryMax=2G) ───────────┐    │
  │  │  session-3.scope                            │    │
  │  └─────────────────────────────────────────────┘    │
  └─────────────────────────────────────────────────────┘

  ┌─ machine.slice (CPUWeight=100, MemoryMax=4G) ─────┐
  │                                                     │
  │  VMs and containers managed by libvirt/systemd-nspawn│
  └─────────────────────────────────────────────────────┘
```

---

## 5.11 Summary — The Complete Picture

This chapter covered the two fundamental Linux primitives that make containers
possible:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      CONTAINER = NS + CGROUPS                        │
│                                                                      │
│  ┌── NAMESPACES (Isolation) ────────┐  ┌── CGROUPS (Limits) ──────┐│
│  │                                   │  │                          ││
│  │  PID   → isolated process tree   │  │  cpu.max    → CPU limit  ││
│  │  NET   → isolated network stack  │  │  memory.max → RAM limit  ││
│  │  MNT   → isolated filesystem     │  │  pids.max   → fork bomb  ││
│  │  UTS   → isolated hostname       │  │  io.max     → disk I/O   ││
│  │  IPC   → isolated IPC            │  │  cpu.weight → CPU share  ││
│  │  USER  → UID/GID mapping         │  │  memory.high→ throttle   ││
│  │  CGROUP→ virtualized cgroupfs    │  │  memory.low → protect    ││
│  │  TIME  → isolated clocks         │  │                          ││
│  └───────────────────────────────────┘  └──────────────────────────┘│
│                                                                      │
│  Syscalls: clone(), unshare(), setns()  Interface: cgroupfs (files) │
│  Files: /proc/[pid]/ns/*                Mount: /sys/fs/cgroup/       │
│                                                                      │
│  These primitives are composed by container runtimes (runc, crun)   │
│  and orchestrated by Kubernetes (kubelet → CRI → OCI runtime)       │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Takeaways

1. **Namespaces provide isolation** — they control what a process can see. There are
   eight types, each isolating a different kernel resource.

2. **Cgroups provide resource control** — they control what a process can use. Cgroups v2
   is the modern unified hierarchy that fixes v1's design problems.

3. **Three syscalls** — `clone()`, `unshare()`, and `setns()` — are the entire
   namespace API. All container runtimes use these same syscalls.

4. **PID 1 is special** — the init process in a PID namespace must handle zombie
   reaping and signal forwarding. Ignoring this causes zombie accumulation.

5. **pivot_root, not chroot** — container runtimes use `pivot_root` because it
   completely severs access to the host filesystem.

6. **User namespaces enable rootless containers** — UID mapping lets unprivileged users
   appear as root inside a container without any host root privileges.

7. **Cgroups v2 unified hierarchy** — single tree, consistent interface, safe delegation,
   and the `memory.high` throttling knob that prevents OOM kills.

8. **systemd manages cgroups** — it creates the hierarchy, delegates subtrees, and
   provides tooling (`systemd-cgls`, `systemd-cgtop`, `systemd-run`).

### What's Next

In **Chapter 6**, we explore the Linux networking stack in depth — the foundations
that underpin both the network namespace connectivity we built here and the CNI
(Container Network Interface) plugins that Kubernetes uses for pod networking.

In **Chapter 8**, we build a complete OCI-compliant container runtime using everything
from this chapter — namespaces, cgroups, pivot_root, and security policies — into
a production-quality implementation.

---

## Exercises

### Exercise 5.1: Namespace Inspector
Write a Go tool that accepts a PID and prints all of its namespaces, including the
inode numbers. Compare them with PID 1's namespaces to determine which namespaces
are shared with the host.

### Exercise 5.2: Fork Bomb Protection
Create a cgroup v2 group with `pids.max=10`, add your shell's PID, and then try
running a fork bomb (`:(){ :|:& };:`). Observe how the pids controller prevents
runaway process creation.

### Exercise 5.3: Memory Limit Stress Test
Write a Go program that allocates memory in a loop (1 MB at a time). Run it inside
a cgroup with `memory.max=64M` and `memory.swap.max=0`. Observe the OOM kill.
Then set `memory.high=48M` and run again — observe the throttling behavior before
the OOM kill threshold.

### Exercise 5.4: Network Namespace from Scratch
Without using any container runtime, create a network namespace using `ip netns`,
set up a veth pair, configure IP addresses and routing, enable NAT, and verify
you can ping an external host from inside the namespace.

### Exercise 5.5: Rootless Container
Create a user namespace mapping your UID to root (UID 0) inside the namespace.
Inside the namespace, set a hostname, mount /proc, and run `id` to verify you
appear as root. Then try reading `/etc/shadow` — explain why it fails.

### Exercise 5.6: CPU Throttling Measurement
Write a Go program that does CPU-intensive work (e.g., computing SHA-256 hashes
in a tight loop) and measures its throughput. Run it with and without `cpu.max`
limits. Measure the actual throughput reduction and compare it to the expected
value from the cpu.max quota.

### Exercise 5.7: systemd Slice Design
Design a systemd slice hierarchy for a web application with three tiers:
- Frontend (2 nginx instances): 30% CPU, 2GB RAM combined
- Backend (4 API servers): 50% CPU, 8GB RAM combined
- Database (1 PostgreSQL): 20% CPU, 6GB RAM

Write the `.slice` and `.service` unit files with appropriate resource directives.

### Exercise 5.8: Mount Propagation
Create two mount namespaces sharing a mount point with `shared` propagation.
Mount a tmpfs in one namespace and verify it appears in the other. Then change
the propagation to `private` and verify mounts no longer propagate.

---

## Further Reading

- **man 7 namespaces** — comprehensive documentation of all namespace types
- **man 7 cgroups** — cgroups v1 and v2 documentation
- **man 2 clone** — clone() syscall reference
- **man 2 unshare** — unshare() syscall reference
- **man 2 setns** — setns() syscall reference
- **man 2 pivot_root** — pivot_root() syscall reference
- **Linux kernel documentation**: `Documentation/admin-guide/cgroup-v2.rst`
- **systemd resource control**: `man 5 systemd.resource-control`
- **OCI Runtime Specification**: https://github.com/opencontainers/runtime-spec
- **runc source code**: https://github.com/opencontainers/runc
- **Michael Kerrisk's namespace articles** on LWN.net (definitive reference)