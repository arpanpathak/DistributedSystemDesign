# Chapter 1: Linux Process Model & System Calls

> *"In Unix, everything is a file. In Linux, everything is a process — or serves one."*

This chapter takes you on a deep dive into how Linux creates, manages, schedules, and
destroys processes. We will explore the kernel data structures that back every running
program, the system calls that control process lifecycles, and the scheduler that decides
who runs next. Along the way, you will write real Go code that interacts with these
primitives directly — not through high-level abstractions, but through raw syscalls, the
`/proc` filesystem, and the `syscall` package.

**Why should a Go developer care?** Go's runtime sits *on top of* the Linux process model.
Goroutines are scheduled onto OS threads. The garbage collector interacts with signals.
`os/exec` calls `clone()` and `execve()`. Understanding the layer beneath your language
runtime transforms you from someone who *uses* Go into someone who *understands* Go.

**Prerequisites:** You should be comfortable writing Go. You should have access to a Linux
machine (a VM or WSL2 works fine). We will use `strace`, read from `/proc`, and make raw
syscalls — all of which require Linux.

---

## 1.1 What Is a Process?

### Program vs. Process

A **program** is a passive entity — a file on disk containing machine instructions (an ELF
binary on Linux). A **process** is an active entity — a program in execution, complete with
its own address space, open file descriptors, signal handlers, and scheduling state.

Think of a program as a recipe and a process as a chef actively cooking that recipe. The
same recipe (program) can be followed by multiple chefs (processes) simultaneously, each
with their own set of ingredients (memory), kitchen timers (signals), and open drawers
(file descriptors).

When you run `./myapp`, the kernel:

1. Allocates a new `task_struct` (the Process Control Block).
2. Creates a virtual address space and loads the ELF segments into it.
3. Sets up the initial stack with `argc`, `argv`, `envp`, and the auxiliary vector.
4. Points the instruction pointer to the ELF entry point (or the dynamic linker).
5. Marks the task as `TASK_RUNNING` and places it on the scheduler's run queue.

### The Process Control Block: `task_struct`

Every process in Linux is represented by a `task_struct` — a massive C structure defined in
`include/linux/sched.h`. It is over **600 fields** and occupies several kilobytes of kernel
memory. Here is a simplified view of the most important fields:

```text
┌─────────────────────────────────────────────────────────────┐
│                      task_struct                            │
├─────────────────────────────────────────────────────────────┤
│  pid_t pid                  // Process ID                   │
│  pid_t tgid                 // Thread Group ID (= PID of   │
│                             //   the main thread)           │
├─────────────────────────────────────────────────────────────┤
│  volatile long state        // TASK_RUNNING, etc.           │
│  int exit_state             // EXIT_ZOMBIE, EXIT_DEAD       │
│  int exit_code              // exit status                  │
├─────────────────────────────────────────────────────────────┤
│  struct mm_struct *mm       // Memory descriptor            │
│    ├── pgd_t *pgd           //   Page Global Directory      │
│    ├── unsigned long start_code, end_code                   │
│    ├── unsigned long start_brk, brk                         │
│    └── unsigned long start_stack                            │
├─────────────────────────────────────────────────────────────┤
│  struct files_struct *files // Open file descriptors        │
│    └── struct fdtable *fdt  //   fd -> struct file mapping  │
├─────────────────────────────────────────────────────────────┤
│  struct fs_struct *fs       // Filesystem info (cwd, root)  │
│  struct nsproxy *nsproxy    // Namespaces (pid, net, mnt..) │
├─────────────────────────────────────────────────────────────┤
│  struct signal_struct *signal   // Shared signal data       │
│  struct sighand_struct *sighand // Signal handlers          │
│  sigset_t blocked               // Blocked signal mask      │
├─────────────────────────────────────────────────────────────┤
│  struct sched_entity se     // CFS scheduling entity        │
│    ├── u64 vruntime         //   Virtual runtime            │
│    └── struct rb_node run_node  // Red-black tree node      │
│  int prio, static_prio     // Dynamic and static priority  │
│  unsigned int policy        // SCHED_NORMAL, SCHED_FIFO..  │
├─────────────────────────────────────────────────────────────┤
│  struct task_struct *parent // Parent process               │
│  struct list_head children  // Child processes              │
│  struct list_head sibling   // Sibling list                 │
├─────────────────────────────────────────────────────────────┤
│  u64 utime, stime          // User/system CPU time         │
│  struct thread_struct thread // CPU-specific state (regs)   │
└─────────────────────────────────────────────────────────────┘
```

Notice how `pid` and `tgid` are separate. In Linux, threads are just processes that share
resources. A "thread" has its own unique `pid` but shares the `tgid` of the thread group
leader (what userspace calls the "process"). When you call `getpid()` in userspace, the
kernel actually returns `tgid`, not `pid`. This will become important when we discuss
`clone()`.

### Process States

A process transitions through several states during its lifetime:

```text
                    ┌──────────────┐
        fork()      │              │  schedule()
   ──────────────►  │ TASK_RUNNING │ ◄──────────────┐
                    │  (runnable)  │                 │
                    └──────┬───────┘                 │
                           │                         │
                    schedule()                wake_up()
                           │                         │
                           ▼                         │
                    ┌──────────────┐          ┌──────┴───────┐
                    │ TASK_RUNNING │          │    TASK_      │
                    │  (running)   │────────► │INTERRUPTIBLE  │
                    │  [on CPU]    │  sleep   │  (waiting)    │
                    └──────┬───────┘          └──────────────┘
                           │
                           │ do_exit()
                           ▼
                    ┌──────────────┐   wait()  ┌─────────────┐
                    │  EXIT_ZOMBIE │ ────────► │  EXIT_DEAD   │
                    │  (zombie)    │  by parent│  (removed)   │
                    └──────────────┘           └─────────────┘
```

| State | Value | Description |
|---|---|---|
| `TASK_RUNNING` | 0 | Either currently executing on a CPU or waiting in the run queue. |
| `TASK_INTERRUPTIBLE` | 1 | Sleeping, waiting for an event. Can be woken by a signal. |
| `TASK_UNINTERRUPTIBLE` | 2 | Sleeping, cannot be interrupted by signals. Used for critical I/O. This causes the dreaded "D" state in `ps`. |
| `__TASK_STOPPED` | 4 | Stopped by a signal (`SIGSTOP`, `SIGTSTP`). |
| `__TASK_TRACED` | 8 | Being traced by `ptrace` (e.g., by a debugger). |
| `EXIT_ZOMBIE` | 16 | Process has exited but its parent has not yet called `wait()`. |
| `EXIT_DEAD` | 32 | Final state. The `task_struct` is being deallocated. |

**The Zombie Problem:** When a process exits, it enters `EXIT_ZOMBIE` state. The kernel
keeps a minimal `task_struct` alive so the parent can retrieve the exit status via
`wait()`/`waitpid()`. If the parent never calls `wait()`, zombies accumulate. They consume
a PID and a small amount of kernel memory. If a parent dies, `init` (PID 1) adopts the
orphaned children and reaps them.

### The `/proc` Filesystem: A Window Into Kernel State

The `/proc` filesystem is a pseudo-filesystem — it does not reside on disk. Instead, every
read from a `/proc` file triggers a kernel function that dynamically generates the content
from in-kernel data structures. It is your primary tool for inspecting process state.

Key files for any process with PID `<pid>`:

| Path | Contents |
|---|---|
| `/proc/<pid>/status` | Human-readable process status (name, state, PIDs, memory, signals). |
| `/proc/<pid>/maps` | Virtual memory mappings (address ranges, permissions, mapped files). |
| `/proc/<pid>/fd/` | Directory of symlinks to open file descriptors. |
| `/proc/<pid>/cmdline` | The command line arguments (null-separated). |
| `/proc/<pid>/stat` | Machine-readable process statistics (used by `ps` and `top`). |
| `/proc/<pid>/syscall` | The syscall currently being executed (if any). |
| `/proc/<pid>/stack` | Kernel stack trace (requires `CAP_SYS_PTRACE`). |
| `/proc/self/` | Magic symlink that always points to the current process. |

### Hands-On: Inspecting Process State from Go

Let us write a Go program that reads its own `/proc/self` entries to demonstrate how
userspace programs can introspect their own kernel state:

```go
// procreader.go
//
// This program demonstrates how to read process state from the /proc
// filesystem. It reads three key files:
//   - /proc/self/status: Human-readable process information including
//     the process name, state, PID, memory usage, and signal masks.
//   - /proc/self/maps:   The virtual memory layout showing every mapped
//     region, its permissions, and what file (if any) backs it.
//   - /proc/self/fd:     The set of currently open file descriptors,
//     each a symlink to the underlying file or socket.
//
// By reading /proc/self (a symlink to /proc/<our-pid>), we avoid
// needing to know our own PID ahead of time.
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"strings"
)

func main() {
	// --- Section 1: Read /proc/self/status ---
	//
	// The status file contains one key-value pair per line.
	// Notable fields include:
	//   Name:   The executable name (truncated to 15 chars).
	//   State:  The current scheduling state (R=running, S=sleeping, etc.).
	//   Pid:    The kernel pid (actually the tgid in kernel terms).
	//   VmRSS:  Resident Set Size — physical memory currently used.
	//   Threads: Number of threads (OS-level, not goroutines).
	fmt.Println("=== /proc/self/status (selected fields) ===")
	statusData, err := os.ReadFile("/proc/self/status")
	if err != nil {
		fmt.Fprintf(os.Stderr, "error reading status: %v\n", err)
		os.Exit(1)
	}
	// Print only the most interesting fields to keep output manageable.
	interestingFields := map[string]bool{
		"Name": true, "State": true, "Pid": true, "Tgid": true,
		"PPid": true, "Threads": true, "VmRSS": true, "VmSize": true,
	}
	for _, line := range strings.Split(string(statusData), "\n") {
		parts := strings.SplitN(line, ":", 2)
		if len(parts) == 2 && interestingFields[strings.TrimSpace(parts[0])] {
			fmt.Println(line)
		}
	}

	// --- Section 2: Read /proc/self/maps (first 10 lines) ---
	//
	// Each line in maps describes one virtual memory area (VMA):
	//   address_range  perms  offset  dev  inode  pathname
	//
	// Example:
	//   00400000-00452000 r-xp 00000000 08:01 12345  /usr/bin/myapp
	//
	// Permissions: r=read, w=write, x=execute, p=private (COW), s=shared
	fmt.Println("\n=== /proc/self/maps (first 10 lines) ===")
	mapsData, err := os.ReadFile("/proc/self/maps")
	if err != nil {
		fmt.Fprintf(os.Stderr, "error reading maps: %v\n", err)
		os.Exit(1)
	}
	lines := strings.Split(string(mapsData), "\n")
	for i, line := range lines {
		if i >= 10 {
			fmt.Printf("... (%d more lines)\n", len(lines)-10)
			break
		}
		fmt.Println(line)
	}

	// --- Section 3: Read /proc/self/fd ---
	//
	// The fd directory contains one symlink per open file descriptor.
	// The name of each entry is the fd number (0, 1, 2, ...),
	// and the symlink target reveals what it points to:
	//   - A real file path (e.g., /dev/pts/0)
	//   - A socket (e.g., socket:[12345])
	//   - A pipe (e.g., pipe:[67890])
	//   - An anon_inode (e.g., anon_inode:[eventpoll])
	fmt.Println("\n=== /proc/self/fd ===")
	fdDir := "/proc/self/fd"
	entries, err := os.ReadDir(fdDir)
	if err != nil {
		fmt.Fprintf(os.Stderr, "error reading fd dir: %v\n", err)
		os.Exit(1)
	}
	for _, entry := range entries {
		// Readlink resolves the symlink to show the actual target.
		target, err := os.Readlink(filepath.Join(fdDir, entry.Name()))
		if err != nil {
			target = "(error reading link)"
		}
		fmt.Printf("  fd %s -> %s\n", entry.Name(), target)
	}
}
```

**What to observe when you run this:**

1. The `State` field will show `R (running)` because the process is actively executing.
2. The `Threads` count will be greater than 1 — Go spawns multiple OS threads for its runtime (GC, scheduler, etc.), even for a simple program.
3. The `maps` output will show the Go binary's text segment, the heap, the stack, and the `[vdso]` mapping (we will cover vDSO in Section 1.4).
4. File descriptors 0, 1, 2 will point to your terminal (`/dev/pts/N`).



---

## 1.2 The Process Lifecycle

### fork(), exec(), wait() — The Unix Trinity

The process lifecycle in Unix/Linux is built on three fundamental operations:

```
  Parent Process                         Child Process
  ┌──────────────┐                      ┌──────────────┐
  │              │    fork()            │  (exact copy  │
  │  Running     │ ──────────────────►  │   of parent)  │
  │  program A   │                      │  program A    │
  │              │                      │              │
  └──────┬───────┘                      └──────┬───────┘
         │                                     │
         │                              exec("/bin/ls")
         │                                     │
         │                              ┌──────▼───────┐
         │                              │  Now running  │
         │  waitpid()                   │  program B    │
         │  (blocks)                    │  (/bin/ls)    │
         │                              └──────┬───────┘
         │                                     │
         │                                  exit(0)
         │                                     │
         │                              ┌──────▼───────┐
         │◄─────── status ─────────────│    ZOMBIE     │
         │                              └──────────────┘
  ┌──────▼───────┐
  │  Continues   │
  │  execution   │
  └──────────────┘
```

**`fork()`** creates a near-exact copy of the calling process. The child gets:
- A copy of the parent's virtual address space (with COW — see below).
- Copies of all open file descriptors (sharing the same underlying file descriptions).
- The same signal handlers, environment variables, and current working directory.
- A **new PID** and a **new `task_struct`**.

The return value of `fork()` is the *only* way to distinguish parent from child:
- In the parent: `fork()` returns the child's PID (a positive integer).
- In the child: `fork()` returns 0.
- On error: `fork()` returns -1.

**`exec()`** (actually a family: `execve`, `execl`, `execp`, etc.) replaces the current
process image with a new program. The PID stays the same, but:
- The text, data, BSS, and stack segments are replaced.
- File descriptors marked `O_CLOEXEC` are closed.
- Signal handlers are reset to defaults (since the handler code no longer exists).
- The `mm_struct` is replaced entirely.

**`wait()`/`waitpid()`** allows a parent to block until a child changes state (exits,
is stopped by a signal, etc.) and retrieves the child's exit status. This is what reaps
zombie processes.

### Copy-on-Write (COW): The Performance Trick

Naively copying the entire address space on `fork()` would be extremely expensive. A
process might have hundreds of megabytes of memory mapped. Instead, Linux uses
**Copy-on-Write**:

```
  BEFORE fork():                 AFTER fork():

  Parent                         Parent          Child
  ┌──────────┐                   ┌──────────┐   ┌──────────┐
  │ Virtual  │                   │ Virtual  │   │ Virtual  │
  │ Page     │──────┐            │ Page     │   │ Page     │
  │ Tables   │      │            │ Tables   │   │ Tables   │
  └──────────┘      │            └────┬─────┘   └────┬─────┘
                    │                 │               │
                    ▼                 │    shared     │
              ┌──────────┐           ▼    (R/O)      ▼
              │ Physical │      ┌──────────────────────────┐
              │ Page     │      │ Physical Pages           │
              │ Frames   │      │ (marked read-only,       │
              └──────────┘      │  ref_count incremented)  │
                                └──────────────────────────┘

  AFTER child writes to a page:

  Parent          Child
  ┌──────────┐   ┌──────────┐
  │ Page     │   │ Page     │
  │ Tables   │   │ Tables   │
  └────┬─────┘   └────┬─────┘
       │               │
       ▼               ▼
  ┌─────────┐   ┌─────────┐     ◄── Kernel allocates a NEW
  │ Original│   │  COPY   │         physical page and copies
  │  Page   │   │ of Page │         content on first write.
  └─────────┘   └─────────┘         Only the written page is
                                     copied — not the entire
                                     address space.
```

The kernel marks all pages in both parent and child as **read-only** after `fork()`. When
either process tries to *write* to a page, a **page fault** occurs. The kernel's page fault
handler detects that this is a COW page (by checking the reference count), allocates a new
physical page, copies the content, and updates the faulting process's page table to point to
the new page with write permission.

This makes `fork()` extremely fast — it only needs to copy the page table entries, not the
actual memory. This is why the `fork() + exec()` pattern is efficient: most of the parent's
pages are never written to by the child before `exec()` replaces them entirely.

### `vfork()` and `clone()` — Variations on a Theme

**`vfork()`** was created before COW existed. It creates a child that *shares* the parent's
address space (no copy at all). The parent is suspended until the child calls `exec()` or
`_exit()`. Using `vfork()` incorrectly (e.g., returning from the function that called it)
causes undefined behavior. Modern Linux still supports `vfork()` but COW makes it mostly
unnecessary.

**`clone()`** is the Swiss Army knife. Both `fork()` and `vfork()` are implemented as
wrappers around `clone()` with different flags:

| Operation | Equivalent `clone()` flags |
|---|---|
| `fork()` | `SIGCHLD` |
| `vfork()` | `CLONE_VM \| CLONE_VFORK \| SIGCHLD` |
| Thread creation | `CLONE_VM \| CLONE_FS \| CLONE_FILES \| CLONE_SIGHAND \| CLONE_THREAD \| CLONE_SYSVSEM \| CLONE_SETTLS \| CLONE_PARENT_SETTID \| CLONE_CHILD_CLEARTID` |

We will explore clone flags in detail in Section 1.3.

### Go's `os/exec` Package: What Happens Under the Hood

When you write:

```go
cmd := exec.Command("/bin/ls", "-la")
cmd.Run()
```

Go does **not** call `fork()`. Instead, it calls `clone()` with careful flag management.
Here is what actually happens (simplified):

1. **`os/exec.Cmd.Start()`** is called.
2. This calls **`os.StartProcess()`** which calls **`syscall.StartProcess()`**.
3. `syscall.StartProcess()` calls **`syscall.forkAndExecInChild()`** — a function written
   in raw assembly (in `runtime/sys_linux_amd64.s` or the platform equivalent).
4. This function calls **`clone()`** with `CLONE_VFORK | CLONE_VM | SIGCHLD`:
   - `CLONE_VM`: Share address space with parent (no COW overhead).
   - `CLONE_VFORK`: Suspend parent until child calls `exec`.
   - This is safe because the child immediately calls `execve()`.
5. In the child, before `execve()`:
   - File descriptors are set up (`dup2` for stdin/stdout/stderr).
   - The working directory is changed if specified.
   - `setpgid()`, `setsid()` are called if configured.
   - Environment variables are prepared.
6. **`execve()`** replaces the child's process image with the target program.
7. The parent is resumed (because `CLONE_VFORK` + `exec` = parent unblocks).

**Why `CLONE_VM | CLONE_VFORK` instead of plain `fork()`?** Performance. Even with COW,
`fork()` must copy the page table entries, which can be significant for Go processes with
large heaps. `CLONE_VM | CLONE_VFORK` avoids this entirely. The downside is that the child
runs in the parent's address space, so it must not call any Go runtime functions (no
allocations, no goroutine scheduling). The `forkAndExecInChild` function is written in
assembly precisely to avoid these pitfalls.

### Hands-On: fork-exec in Go with `syscall.ForkExec`

```go
// forkexec.go
//
// Demonstrates using syscall.ForkExec to launch a child process.
// This is the low-level interface that os/exec.Command uses internally.
//
// syscall.ForkExec atomically forks and execs in a single operation,
// which avoids the complexities of managing a child process between
// fork() and exec() in a multithreaded Go runtime.
//
// The SysProcAttr struct gives fine-grained control over:
//   - Credential: run the child as a different UID/GID.
//   - Chroot:     change the child's root directory.
//   - Setpgid:    put the child in a new process group.
//   - Setsid:     make the child a session leader.
//   - Ptrace:     launch the child under ptrace.
//   - Cloneflags: pass raw clone flags to the kernel.
package main

import (
	"fmt"
	"os"
	"syscall"
)

func main() {
	// ForkExec takes the path to the executable, argv (where argv[0]
		// is conventionally the program name), and a ProcAttr that controls
	// the child's environment, working directory, and file descriptors.
	//
	// The Files field maps child fd numbers to parent fd numbers:
	//   Files[0] = parent's stdin  -> child's fd 0
	//   Files[1] = parent's stdout -> child's fd 1
	//   Files[2] = parent's stderr -> child's fd 2
	pid, err := syscall.ForkExec("/bin/echo", []string{"echo", "Hello from child process!"}, &syscall.ProcAttr{
			Dir:   "/",
			Env:   os.Environ(),
			Files: []uintptr{
				uintptr(syscall.Stdin),
				uintptr(syscall.Stdout),
				uintptr(syscall.Stderr),
			},
		})
	if err != nil {
		fmt.Fprintf(os.Stderr, "ForkExec failed: %v\n", err)
		os.Exit(1)
	}

	fmt.Printf("Parent: launched child with PID %d\n", pid)

	// Wait for the child to finish. Wait4 fills a WaitStatus that
	// we can inspect for the exit code, signal information, etc.
	var ws syscall.WaitStatus
	var rusage syscall.Rusage
	wpid, err := syscall.Wait4(pid, &ws, 0, &rusage)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Wait4 failed: %v\n", err)
		os.Exit(1)
	}

	fmt.Printf("Parent: child %d exited with status %d\n", wpid, ws.ExitStatus())
	fmt.Printf("Parent: child used %d µs of user CPU time\n", rusage.Utime.Usec)
}
```

### Hands-On: Building a Simple Shell in Go

This is one of the most instructive exercises in systems programming. A shell is simply a
loop that reads input, parses it, and fork-execs the requested command:

```go
// minishell.go
//
// A minimal Unix shell implemented in Go. This demonstrates the
// fundamental read-parse-fork-exec-wait loop that every shell uses.
//
// Features:
//   - Executes external commands by searching PATH.
//   - Handles the "cd" built-in (which must be handled in-process
	//     because changing the working directory of a child has no effect
	//     on the parent).
//   - Handles the "exit" built-in.
//   - Displays the current working directory in the prompt.
//
// Limitations (intentional — this is a teaching example):
//   - No pipes, redirections, or job control.
//   - No quoting or escaping.
//   - No signal handling (Ctrl+C kills the shell, not just the child).
package main

import (
	"bufio"
	"fmt"
	"os"
	"os/exec"
	"strings"
)

func main() {
	reader := bufio.NewReader(os.Stdin)

	for {
		// Print a prompt showing the current working directory.
		cwd, _ := os.Getwd()
		fmt.Printf("minishell:%s$ ", cwd)

		// Read one line of input.
		input, err := reader.ReadString('\n')
		if err != nil {
			// EOF (Ctrl+D) exits the shell gracefully.
			fmt.Println("\nexit")
			break
		}
		input = strings.TrimSpace(input)
		if input == "" {
			continue
		}

		// Parse the input into command and arguments.
		// A real shell would handle quoting, escaping, and expansion here.
		args := strings.Fields(input)
		command := args[0]

		// Handle built-in commands.
		// "cd" must be a built-in because changing the directory in a
		// child process would not affect the shell (parent) process.
		switch command {
		case "exit":
			fmt.Println("Goodbye!")
			os.Exit(0)
		case "cd":
			if len(args) < 2 {
				home, _ := os.UserHomeDir()
				os.Chdir(home)
			} else {
				if err := os.Chdir(args[1]); err != nil {
					fmt.Fprintf(os.Stderr, "cd: %v\n", err)
				}
			}
			continue
		}

		// For external commands, use os/exec which handles the
		// fork-exec-wait cycle for us. Under the hood, this calls
		// clone(CLONE_VFORK|CLONE_VM) + execve().
		//
		// exec.Command searches PATH for the binary, sets up pipes
		// for stdin/stdout/stderr, and manages the child lifecycle.
		cmd := exec.Command(command, args[1:]...)
		cmd.Stdin = os.Stdin
		cmd.Stdout = os.Stdout
		cmd.Stderr = os.Stderr

		if err := cmd.Run(); err != nil {
			fmt.Fprintf(os.Stderr, "minishell: %v\n", err)
		}
	}
}
```



---

## 1.3 Threads vs. Processes

### Linux's Unified Model

Here is a fact that surprises many developers coming from Windows: **Linux does not have a
separate concept of a "thread" at the kernel level.** Both processes and threads are
represented by `task_struct`. The difference is entirely determined by what resources are
*shared* between them.

When you create a "process" with `fork()`, you get a new `task_struct` that has its *own*
copies of the address space, file descriptor table, filesystem info, and signal handlers.

When you create a "thread" with `pthread_create()` (which calls `clone()`), you get a new
`task_struct` that *shares* the address space, file descriptors, filesystem info, and signal
handlers with the creator.

The `clone()` system call controls this with flags:

```
clone(flags, stack, parent_tidptr, child_tidptr, tls)
```

Each flag controls one dimension of sharing:

| Flag | What is shared | Process | Thread |
|---|---|---|---|
| `CLONE_VM` | Virtual memory (address space) | ✗ (copy) | ✓ (shared) |
| `CLONE_FILES` | File descriptor table | ✗ (copy) | ✓ (shared) |
| `CLONE_FS` | Filesystem info (cwd, root, umask) | ✗ (copy) | ✓ (shared) |
| `CLONE_SIGHAND` | Signal handlers | ✗ (copy) | ✓ (shared) |
| `CLONE_THREAD` | Thread group (same TGID) | ✗ (new TGID) | ✓ (same TGID) |
| `CLONE_SYSVSEM` | SysV semaphore undo values | ✗ | ✓ |
| `CLONE_SETTLS` | Thread-Local Storage | — | ✓ (new TLS) |

This means you can create exotic combinations. Want a "process" that shares file descriptors
but has its own address space? Pass `CLONE_FILES` without `CLONE_VM`. Linux's model is
fundamentally more flexible than the traditional "process vs thread" dichotomy.

**`CLONE_THREAD`** deserves special attention. When set, the new task joins the same
*thread group* as the caller. This means:
- The new task's `tgid` is set to the caller's `tgid`.
- The new task shares the same PID as seen by userspace (`getpid()` returns `tgid`).
- Signals sent to the "process" (i.e., to the TGID) are delivered to any thread in the group.
- `exit_group()` terminates all threads in the group.
- The new task's `pid` (the kernel-internal one) is still unique.

### Go's Goroutine Model: M:N Scheduling

Go uses an **M:N scheduling** model where M goroutines are multiplexed onto N OS threads.
This is fundamentally different from:

- **1:1 model** (C with pthreads): Each user thread maps to exactly one kernel thread.
- **N:1 model** (green threads): All user threads run on a single kernel thread.

Go's model gives you the best of both worlds: goroutines are cheap (a few KB of stack,
microseconds to create) while still being able to utilize multiple CPU cores.

The Go scheduler is built around three entities, commonly called **G, M, P**:

```
┌──────────────────────────────────────────────────────────────────┐
│                       Go Scheduler Model                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  G = Goroutine                                                   │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐             │
│  │ G1 │ │ G2 │ │ G3 │ │ G4 │ │ G5 │ │ G6 │ │ G7 │ ...         │
│  └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘             │
│    │      │      │      │      │      │      │                   │
│    │      │      │      │      │      │      │                   │
│    ▼      ▼      ▼      ▼      ▼      ▼      ▼                   │
│  P = Processor (logical CPU)                                     │
│  ┌─────────────────┐   ┌─────────────────┐                      │
│  │       P0        │   │       P1        │                      │
│  │  ┌───────────┐  │   │  ┌───────────┐  │                      │
│  │  │ Local Run │  │   │  │ Local Run │  │                      │
│  │  │   Queue   │  │   │  │   Queue   │  │                      │
│  │  │ [G1,G2,G3]│  │   │  │ [G4,G5]  │  │                      │
│  │  └───────────┘  │   │  └───────────┘  │                      │
│  │  current: G1    │   │  current: G4    │                      │
│  └────────┬────────┘   └────────┬────────┘                      │
│           │                     │                                │
│           ▼                     ▼                                │
│  M = Machine (OS Thread)                                         │
│  ┌─────────────────┐   ┌─────────────────┐   ┌──────────────┐  │
│  │       M0        │   │       M1        │   │     M2       │  │
│  │  (bound to P0)  │   │  (bound to P1)  │   │  (idle or    │  │
│  │  syscall:clone  │   │  syscall:futex  │   │   in syscall)│  │
│  └─────────────────┘   └─────────────────┘   └──────────────┘  │
│           │                     │                                │
│           ▼                     ▼                                │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              Linux Kernel Scheduler (CFS)                   │ │
│  │   Schedules M0, M1, M2 onto physical CPU cores              │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  Global Run Queue: [G6, G7]  (overflow / unassigned goroutines) │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**G (Goroutine):** The unit of work. Each goroutine has a small stack (starting at 2-8 KB,
growable), a program counter, and a reference to the function it is executing. Goroutines
are not visible to the kernel — they exist entirely in userspace.

**M (Machine/OS Thread):** A real OS thread backed by a `task_struct` in the kernel. Ms are
created via `clone()` with thread flags. An M must be attached to a P to run goroutines.
Ms that are blocked in syscalls detach from their P.

**P (Processor):** A logical processor — a scheduling context. The number of Ps equals
`GOMAXPROCS` (defaults to the number of CPU cores). Each P has a local run queue of
goroutines. A P must be attached to an M to execute goroutines.

**The scheduling loop:**

1. An M attached to a P picks the next G from P's local run queue.
2. If the local queue is empty, it tries to *steal* from another P's queue.
3. If all local queues are empty, it checks the global run queue.
4. If there is nothing to do, the M parks itself and the P is released.

**When a goroutine makes a blocking syscall:**

1. The Go runtime detects the syscall entry.
2. The M (OS thread) detaches from its P.
3. The P is handed off to another M (or a new M is created).
4. The original M blocks in the kernel, holding only the G that made the syscall.
5. When the syscall returns, the M tries to reacquire a P. If no P is available, the G
   is put on the global run queue and the M parks.

This is why Go can handle thousands of concurrent I/O operations without thousands of
threads: only goroutines that are actually blocked in syscalls need a dedicated OS thread.

**`GOMAXPROCS`:** Sets the number of Ps. With `GOMAXPROCS=4`, at most 4 goroutines can
execute user-level Go code simultaneously (but more Ms may exist for syscall-blocked
goroutines).

### Hands-On: Tracing Goroutine-to-Thread Mapping

```go
// goroutine_threads.go
//
// This program demonstrates how goroutines map to OS threads.
// It launches several goroutines and has each one report:
//   - Its goroutine ID (extracted via runtime.Stack, as Go does not
	//     expose goroutine IDs directly — by design).
//   - The OS thread ID it is currently running on (via syscall.Gettid).
//
// Key observations:
//   1. Multiple goroutines may report the same thread ID, proving
//      that goroutines are multiplexed onto OS threads.
//   2. The number of unique thread IDs will be at most GOMAXPROCS
//      (for CPU-bound work without blocking syscalls).
//   3. Goroutines may migrate between threads between observations.
package main

import (
	"fmt"
	"runtime"
	"sync"
	"syscall"
)

func main() {
	// Set GOMAXPROCS to 2 so we can clearly see the multiplexing.
	// With 2 Ps, at most 2 goroutines run simultaneously.
	runtime.GOMAXPROCS(2)

	var wg sync.WaitGroup
	const numGoroutines = 8

	fmt.Printf("GOMAXPROCS = %d\n", runtime.GOMAXPROCS(0))
	fmt.Printf("NumCPU     = %d\n", runtime.NumCPU())
	fmt.Println()

	for i := 0; i < numGoroutines; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()

			// LockOSThread ensures this goroutine stays on the same
			// OS thread for the duration. Without this, the goroutine
			// could migrate between observations.
			//
			// Note: LockOSThread is NOT normally needed. We use it
			// here for demonstration clarity. In production, it is
			// used for C library interop, thread-local state, etc.
			runtime.LockOSThread()
			defer runtime.UnlockOSThread()

			// Gettid returns the Linux thread ID (the kernel pid).
			// This is different from getpid() which returns the tgid.
			tid := syscall.Gettid()

			fmt.Printf("Goroutine %d: running on OS thread (tid=%d)\n", id, tid)
		}(i)
	}

	wg.Wait()

	fmt.Printf("\nTotal OS threads used by Go runtime: %d\n", runtime.NumGoroutine())
}
```



---

## 1.4 System Calls — The Kernel Gateway

### What Is a System Call?

A system call is the **only** mechanism by which userspace code can request services from
the kernel. Every file read, every network packet sent, every process created — they all go
through system calls. Understanding syscalls is understanding the boundary between your code
and the operating system.

When your Go program calls `os.Open()`, here is what happens at the machine level:

```
  Your Go Code              Go Runtime             Kernel
  ┌──────────┐          ┌──────────────┐      ┌───────────────┐
  │os.Open() │ ──────►  │syscall.Open()│      │               │
  │          │          │              │      │               │
  └──────────┘          │ RAX = 2      │      │ sys_open()    │
                        │ RDI = path   │      │               │
                        │ RSI = flags  │      │ - permission  │
                        │ RDX = mode   │      │   check       │
                        │              │      │ - allocate fd │
                        │ SYSCALL instr│─────►│ - create      │
                        │              │      │   struct file  │
                        │ (trap to     │      │ - install in  │
                        │  kernel mode)│      │   fd table    │
                        │              │◄─────│               │
                        │ RAX = fd (or │      │ return fd     │
                        │       -errno)│      │               │
                        └──────────────┘      └───────────────┘
```

On x86_64 Linux, the mechanism works like this:

1. **Load the syscall number** into the `RAX` register. Each syscall has a unique number
   (e.g., `read` = 0, `write` = 1, `open` = 2, `close` = 3).
2. **Load arguments** into registers `RDI`, `RSI`, `RDX`, `R10`, `R8`, `R9` (up to 6 args).
3. **Execute the `SYSCALL` instruction.** This is a special CPU instruction that:
   - Switches from Ring 3 (user mode) to Ring 0 (kernel mode).
   - Saves the user-mode RIP (instruction pointer) and RFLAGS.
   - Jumps to the kernel's syscall entry point (`entry_SYSCALL_64`).
4. **The kernel dispatches** through the syscall table (`sys_call_table`) using RAX as index.
5. **The kernel function executes** and places the return value in `RAX`.
6. **`SYSRET` instruction** returns to user mode.

The older mechanism used `INT 0x80` (a software interrupt), which is slower because it goes
through the full interrupt descriptor table lookup. Modern x86_64 systems always use the
`SYSCALL`/`SYSRET` instruction pair.

### The Syscall Table

The syscall table is a simple array of function pointers in the kernel:

```c
// arch/x86/entry/syscall_64.c (simplified)
const sys_call_ptr_t sys_call_table[] = {
    [0] = sys_read,        // #define __NR_read 0
    [1] = sys_write,       // #define __NR_write 1
    [2] = sys_open,        // #define __NR_open 2
    [3] = sys_close,       // #define __NR_close 3
    // ... hundreds more
    [56] = sys_clone,
    [57] = sys_fork,
    [59] = sys_execve,
    [61] = sys_wait4,
    // ...
    [435] = sys_clone3,    // newest additions get higher numbers
};
```

You can see the full table on your system:

```bash
$ ausyscall --dump | head -20
# or
$ cat /usr/include/asm/unistd_64.h
```

### How Go Makes System Calls

Go has **three layers** for making syscalls, each with different trade-offs:

**Layer 1: `runtime/internal/syscall`** — Used by the Go runtime itself. These are raw
syscall wrappers that do not interact with the Go scheduler. Used for critical runtime
operations (memory management, thread creation, signal handling). You cannot import this
package.

**Layer 2: `syscall` package** — The standard library's syscall interface. Functions like
`syscall.Open()`, `syscall.Read()`, `syscall.Write()`. These are "scheduler-aware": before
entering a blocking syscall, they notify the Go runtime so it can hand off the P to another
M. This package is frozen — new syscalls are not added.

**Layer 3: `golang.org/x/sys/unix`** — The maintained, comprehensive syscall package.
Contains wrappers for virtually every Linux syscall. Use this for new code.

```go
// Three ways to call getpid():

import "syscall"
pid1 := syscall.Getpid()

import "golang.org/x/sys/unix"
pid2 := unix.Getpid()

// Raw syscall (the hard way):
pid3, _, _ := syscall.RawSyscall(syscall.SYS_GETPID, 0, 0, 0)
```

**Important distinction:** `syscall.Syscall()` vs `syscall.RawSyscall()`:
- `Syscall()` calls `runtime.entersyscall()` before and `runtime.exitsyscall()` after.
  This allows the Go scheduler to detach the P and give it to another M.
- `RawSyscall()` does NOT notify the scheduler. If the syscall blocks, the entire P
  (and all goroutines queued on it) stalls. Use only for syscalls guaranteed to be fast
  and non-blocking (like `getpid()`).

### vDSO — The Fast Path

Some syscalls are called extremely frequently (e.g., `clock_gettime()`, `gettimeofday()`)
but only need to *read* kernel data, not *modify* anything. For these, the overhead of a
full user→kernel→user transition is wasteful.

The **vDSO** (virtual Dynamic Shared Object) solves this. It is a small shared library
that the kernel maps into every process's address space. It contains userspace
implementations of selected syscalls that read from a shared memory page the kernel keeps
updated.

```
  Without vDSO:                  With vDSO:

  clock_gettime()                clock_gettime()
       │                              │
       ▼                              ▼
  SYSCALL instruction            Read from shared
       │                         memory page
       ▼                         (mapped by kernel)
  Ring 0 transition                   │
       │                              ▼
       ▼                         Return to caller
  Kernel handler                 ── No mode switch!
       │                         ── No context switch!
       ▼                         ── ~20ns vs ~200ns
  SYSRET to Ring 3
       │
       ▼
  Return to caller
```

You can see the vDSO in your process's memory map:

```bash
$ cat /proc/self/maps | grep vdso
7ffd4b1fe000-7ffd4b200000 r-xp 00000000 00:00 0  [vdso]
```

Go's `time.Now()` uses the vDSO path via `clock_gettime(CLOCK_REALTIME)` for maximum
performance.

### Hands-On: Making Raw Syscalls from Go

```go
// rawsyscall.go
//
// Demonstrates making raw system calls from Go using both the
// syscall package and the x/sys/unix package. We call several
// syscalls directly to show the mechanics:
//
//   1. SYS_GETPID  - Get our process ID.
//   2. SYS_GETUID  - Get our user ID.
//   3. SYS_UNAME   - Get system information (kernel version, etc.).
//   4. SYS_WRITE   - Write directly to stdout via syscall, bypassing
//                     Go's buffered I/O entirely.
//
// This illustrates that at the lowest level, everything in your
// program ultimately reduces to numbered syscall invocations.
package main

import (
	"fmt"
	"syscall"
	"unsafe"
)

func main() {
	// --- SYS_GETPID ---
	// getpid() takes no arguments and returns the process ID.
	// We use RawSyscall because getpid is guaranteed to never block.
	pid, _, _ := syscall.RawSyscall(syscall.SYS_GETPID, 0, 0, 0)
	fmt.Printf("PID (via raw syscall):  %d\n", pid)
	fmt.Printf("PID (via syscall pkg):  %d\n", syscall.Getpid())

	// --- SYS_GETUID ---
	// Similarly, getuid() returns the real user ID of the calling process.
	uid, _, _ := syscall.RawSyscall(syscall.SYS_GETUID, 0, 0, 0)
	fmt.Printf("UID (via raw syscall):  %d\n", uid)

	// --- SYS_UNAME ---
	// uname() fills a struct with system information.
	// The Go syscall package provides a Utsname struct matching the
	// kernel's struct utsname layout.
	var utsname syscall.Utsname
	_, _, errno := syscall.RawSyscall(
		syscall.SYS_UNAME,
		uintptr(unsafe.Pointer(&utsname)),
		0, 0,
	)
	if errno != 0 {
		fmt.Printf("uname failed: %v\n", errno)
	} else {
		// The fields are [65]byte arrays (null-terminated C strings).
		// We convert them to Go strings for display.
		fmt.Printf("System:   %s\n", byteSliceToString(utsname.Sysname[:]))
		fmt.Printf("Hostname: %s\n", byteSliceToString(utsname.Nodename[:]))
		fmt.Printf("Release:  %s\n", byteSliceToString(utsname.Release[:]))
		fmt.Printf("Machine:  %s\n", byteSliceToString(utsname.Machine[:]))
	}

	// --- SYS_WRITE ---
	// Write directly to file descriptor 1 (stdout) using the write syscall.
	// This completely bypasses Go's fmt, bufio, and os packages.
	msg := []byte("Hello directly from SYS_WRITE!\n")
	syscall.Syscall(
		syscall.SYS_WRITE,
		uintptr(1), // fd 1 = stdout
		uintptr(unsafe.Pointer(&msg[0])),
		uintptr(len(msg)),
	)
}

// byteSliceToString converts a null-terminated byte array (C string)
// to a Go string by finding the first null byte.
func byteSliceToString(b []byte) string {
	// The Utsname fields are int8 arrays on some platforms and byte
	// arrays on others, but always null-terminated.
	for i, v := range b {
		if v == 0 {
			return string(b[:i])
		}
	}
	return string(b)
}
```

### Hands-On: Using strace to Trace Go Program Syscalls

`strace` is your most powerful tool for understanding what a program does at the kernel
level. It intercepts every syscall a process makes using `ptrace()`.

```bash
# Trace all syscalls made by a Go program.
# -f: follow child threads (important for Go, which uses multiple threads)
# -e trace=process: only show process-related syscalls (clone, execve, exit, wait)
$ strace -f -e trace=process go run hello.go

# Show a summary of syscall counts and timing:
$ strace -f -c ./myprogram

# Trace specific syscalls with timestamps:
$ strace -f -T -e trace=open,read,write,close ./myprogram

# Trace a running process by PID:
$ strace -f -p $(pidof myprogram)
```

**What to look for in Go program strace output:**

```
# You will see something like this:
clone(child_stack=0x7f..., flags=CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|
      CLONE_THREAD|CLONE_SYSVSEM|CLONE_SETTLS|CLONE_PARENT_SETTID|
      CLONE_CHILD_CLEARTID, ...) = 12345

# ^ This is the Go runtime creating OS threads. Notice the CLONE_THREAD flag.

futex(0xc000032148, FUTEX_WAIT_PRIVATE, 0, NULL) = 0
futex(0xc000032148, FUTEX_WAKE_PRIVATE, 1) = 1

# ^ The Go scheduler uses futexes for thread synchronization.

# When using os/exec:
clone(child_stack=NULL, flags=CLONE_VM|CLONE_VFORK|SIGCHLD) = 12346
# In child:
execve("/bin/ls", ["ls", "-la"], [...]) = 0

# ^ CLONE_VM|CLONE_VFORK is Go's optimized fork-exec pattern.
```



---

## 1.5 Process Scheduling

### The Completely Fair Scheduler (CFS)

Linux's default scheduler since kernel 2.6.23 is the **Completely Fair Scheduler (CFS)**.
Its design goal is elegant: give every runnable task an equal share of CPU time,
proportional to its weight (priority).

The core idea is **virtual runtime** (`vruntime`). Each task accumulates virtual runtime as
it runs. Tasks with lower `vruntime` get priority — they have received less than their fair
share. The scheduler always picks the task with the smallest `vruntime`.

```
  CFS Red-Black Tree (ordered by vruntime)
  ─────────────────────────────────────────

                    ┌─────────┐
                    │ Task C  │
                    │ vrt=500 │
                    └────┬────┘
                   ╱             ╲
          ┌─────────┐       ┌─────────┐
          │ Task A  │       │ Task E  │
          │ vrt=300 │       │ vrt=700 │
          └────┬────┘       └────┬────┘
         ╱         ╲       ╱         ╲
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │ Task B  │  │ Task D  │  │ Task F  │
    │ vrt=100 │  │ vrt=400 │  │ vrt=900 │
    └─────────┘  └─────────┘  └─────────┘
         ▲
         │
    LEFTMOST NODE
    (next to run)

  ◄─── smaller vruntime ──── larger vruntime ───►
  ◄─── gets scheduled next                      ───►
```

**Why a red-black tree?** Finding the minimum element is O(log n), but the kernel caches
the leftmost node pointer, making it O(1). Insertion and deletion are O(log n). This gives
predictable scheduling overhead regardless of the number of runnable tasks.

**How `vruntime` accumulates:** It is NOT just wall-clock time. It is weighted:

```
vruntime += actual_runtime × (NICE_0_WEIGHT / task_weight)
```

A task with a higher `nice` value (lower priority) has a smaller weight, so its `vruntime`
increases faster. It "uses up" its fair share quicker and gets preempted sooner. A task with
a lower `nice` value (higher priority) accumulates `vruntime` more slowly, getting more
actual CPU time before the scheduler picks someone else.

### Nice Values and Scheduling Policies

**Nice values** range from -20 (highest priority) to +19 (lowest priority), with 0 as the
default. Each nice level corresponds to a weight that affects `vruntime` accumulation. The
difference between each nice level is approximately 10% CPU time — a task at nice -1 gets
roughly 10% more CPU than a task at nice 0.

**Scheduling policies** determine which scheduler class handles a task:

| Policy | Class | Description |
|---|---|---|
| `SCHED_OTHER` (= `SCHED_NORMAL`) | CFS | Default time-sharing policy. Uses nice values. |
| `SCHED_BATCH` | CFS | For CPU-intensive batch jobs. Like NORMAL but with a scheduling penalty to avoid starving interactive tasks. |
| `SCHED_IDLE` | CFS | Ultra-low priority. Only runs when no other CFS task is runnable. |
| `SCHED_FIFO` | RT | Real-time, first-in-first-out. Runs until it voluntarily yields or a higher-priority RT task preempts it. Requires `CAP_SYS_NICE`. |
| `SCHED_RR` | RT | Real-time, round-robin. Like FIFO but with a time quantum — tasks at the same priority level take turns. |
| `SCHED_DEADLINE` | DL | Earliest Deadline First. For tasks with explicit period/runtime/deadline parameters. The highest-priority class. |

**Real-time scheduling** (`SCHED_FIFO` and `SCHED_RR`) always preempts CFS tasks. A
real-time task at priority 1 beats any CFS task. Real-time priorities range from 1 to 99.
**Warning:** A runaway `SCHED_FIFO` task can lock up your system because it will never be
preempted by normal tasks. Always have a safety net (SSH access on another core, or use
`SCHED_DEADLINE` with enforced budgets).

### CPU Affinity

CPU affinity controls which CPU cores a process/thread is allowed to run on. By default,
a task can run on any core. Setting affinity restricts it to specific cores.

**Why would you set CPU affinity?**

1. **Cache optimization:** Keeping a thread on the same core improves L1/L2 cache hit rates.
2. **NUMA awareness:** On multi-socket systems, you want threads to run near their memory.
3. **Isolation:** Dedicate cores to latency-sensitive tasks, keeping other work off them.
4. **Benchmarking:** Pin a benchmark to a single core for reproducible results.

The syscalls are `sched_setaffinity()` and `sched_getaffinity()`.

### Hands-On: Setting CPU Affinity from Go

```go
// cpuaffinity.go
//
// Demonstrates setting CPU affinity from Go using golang.org/x/sys/unix.
// We pin specific goroutines to specific CPU cores, then verify by
// checking which core we are actually running on.
//
// This technique is useful for:
//   - Performance-critical code that benefits from cache locality.
//   - Benchmarks that need reproducible, jitter-free measurements.
//   - NUMA-aware applications on multi-socket servers.
//
// IMPORTANT: CPU affinity is a per-thread attribute. In Go, you must
// call runtime.LockOSThread() first to ensure the goroutine stays
// on the same OS thread. Otherwise, the Go scheduler might migrate
// the goroutine to a different (unpinned) thread.
package main

import (
	"fmt"
	"os"
	"runtime"

	"golang.org/x/sys/unix"
)

func main() {
	fmt.Printf("Number of CPUs: %d\n", runtime.NumCPU())
	fmt.Printf("GOMAXPROCS:     %d\n\n", runtime.GOMAXPROCS(0))

	// First, get the current CPU affinity mask.
	// This shows which CPUs we are currently allowed to run on.
	var currentMask unix.CPUSet
	err := unix.SchedGetaffinity(0, &currentMask) // 0 = current process
	if err != nil {
		fmt.Fprintf(os.Stderr, "SchedGetaffinity: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("Current affinity mask: CPUs ")
	for i := 0; i < runtime.NumCPU(); i++ {
		if currentMask.IsSet(i) {
			fmt.Printf("%d ", i)
		}
	}
	fmt.Println()

	// Now pin this thread to CPU 0 only.
	// Step 1: Lock the goroutine to its current OS thread.
	runtime.LockOSThread()
	defer runtime.UnlockOSThread()

	// Step 2: Create a CPU set with only CPU 0.
	var pinMask unix.CPUSet
	pinMask.Set(0) // Only allow CPU 0

	// Step 3: Apply the affinity mask.
	// The 0 argument means "apply to the calling thread."
	err = unix.SchedSetaffinity(0, &pinMask)
	if err != nil {
		fmt.Fprintf(os.Stderr, "SchedSetaffinity: %v\n", err)
		os.Exit(1)
	}

	fmt.Println("\nAfter pinning to CPU 0:")

	// Verify: read back the affinity.
	var verifyMask unix.CPUSet
	unix.SchedGetaffinity(0, &verifyMask)
	fmt.Printf("New affinity mask: CPUs ")
	for i := 0; i < runtime.NumCPU(); i++ {
		if verifyMask.IsSet(i) {
			fmt.Printf("%d ", i)
		}
	}
	fmt.Println()

	// Do some CPU-bound work on the pinned core.
	sum := 0
	for i := 0; i < 100_000_000; i++ {
		sum += i
	}
	fmt.Printf("Computed sum=%d on CPU 0\n", sum)
}
```

### Hands-On: Observing Scheduler Behavior

You can observe scheduler statistics for any process via `/proc/[pid]/sched`:

```bash
# View scheduler stats for a running process:
$ cat /proc/$(pidof myprogram)/sched

# Key fields:
#   se.sum_exec_runtime   : Total CPU time in nanoseconds.
#   se.vruntime           : Current virtual runtime.
#   nr_switches           : Total context switches (voluntary + involuntary).
#   nr_voluntary_switches : Process voluntarily gave up CPU (e.g., sleep, I/O).
#   nr_involuntary_switches: Preempted by scheduler (time slice expired).
```

A high ratio of involuntary-to-voluntary switches indicates a CPU-bound workload being
preempted frequently — potentially too many competing tasks for the available cores.



---

## 1.6 Process Groups, Sessions, and Terminals

### The Process Hierarchy

Linux organizes processes into a three-level hierarchy:

```
  Session (SID = 1000)
  │
  ├── Process Group (PGID = 1000) ── "foreground group"
  │   ├── bash (PID 1000, session leader, group leader)
  │   └── (controlling terminal: /dev/pts/0)
  │
  ├── Process Group (PGID = 1500) ── "foreground job"
  │   ├── cat file.txt (PID 1500, group leader)
  │   └── grep pattern (PID 1501)
  │
  └── Process Group (PGID = 1600) ── "background job"
      └── make -j8 (PID 1600, group leader)
          ├── make[1] (PID 1601)
          ├── gcc ... (PID 1602)
          └── gcc ... (PID 1603)
```

**Process Group:** A collection of related processes, typically a pipeline. When you run
`cat file.txt | grep pattern`, both `cat` and `grep` are placed in the same process group.
Signals like `SIGINT` (Ctrl+C) are sent to the entire foreground process group, not to an
individual process. This is why Ctrl+C kills the whole pipeline.

**Session:** A collection of process groups, typically associated with a single login.
The **session leader** is usually the shell. The session owns a **controlling terminal**
(`/dev/pts/N` for pseudo-terminals, `/dev/ttyN` for virtual consoles).

**Controlling Terminal:** The terminal that can send signals to the session's foreground
process group. Only one process group per session is the "foreground" group at any time.
The terminal sends:
- `SIGINT` (Ctrl+C) to the foreground group.
- `SIGTSTP` (Ctrl+Z) to the foreground group.
- `SIGHUP` when the terminal is closed (e.g., SSH disconnection).

### Key System Calls

**`setsid()`**: Creates a new session. The calling process becomes the session leader and
process group leader. The process has no controlling terminal (must explicitly acquire one).
Used by daemons to detach from the parent's session.

**`setpgid(pid, pgid)`**: Moves a process into a different process group, or creates a new
one. Used by shells to set up pipelines in their own process groups.

**`tcsetpgrp(fd, pgid)`**: Sets the foreground process group for the terminal. This is how
shells implement `fg` and `bg`.

### Daemon Processes: The Double-Fork Pattern

A **daemon** is a long-running background process that has no controlling terminal. The
classic way to create a daemon is the **double-fork** pattern:

```
  Original Process
       │
       ├── fork() ──────► First Child
       │                      │
     exit()                   ├── setsid()     ← becomes session leader
                              │                   (no controlling terminal)
                              ├── fork() ──────► Second Child (the daemon)
                              │                      │
                            exit()                   ├── chdir("/")
                                                     ├── umask(0)
                                                     ├── close(0,1,2)
                                                     └── ... do work ...
```

**Why the double fork?**

1. **First `fork()`**: The parent exits, so the child is orphaned and adopted by `init`.
   This ensures the child is not a process group leader (needed for `setsid()`).

2. **`setsid()`**: Creates a new session. The child becomes the session leader. But — as a
   session leader, it *could* acquire a controlling terminal if it opens one.

3. **Second `fork()`**: The session leader exits. The grandchild is NOT a session leader,
   so it **cannot** accidentally acquire a controlling terminal. This is the final daemon.

4. **`chdir("/")`**: Prevents the daemon from holding a mount point busy.

5. **`umask(0)`**: Ensures the daemon can create files with any permissions.

6. **Close stdin/stdout/stderr**: The daemon has no terminal, so these are meaningless.
   Redirect them to `/dev/null`.

### Hands-On: Creating a Daemon Process in Go

```go
// daemon.go
//
// Demonstrates creating a daemon process in Go. We use a simplified
// approach: instead of the traditional double-fork (which is tricky in
	// Go due to the multi-threaded runtime), we use os.StartProcess with
// SysProcAttr.Setsid to create a new session.
//
// The daemon:
//   1. Detaches from the controlling terminal via Setsid.
//   2. Changes its working directory to /.
//   3. Writes a PID file so we can find it later.
//   4. Logs to a file (since it has no terminal).
//   5. Runs a simple loop writing timestamps to the log.
//
// This is a teaching example. In production, consider using systemd
// (which handles daemonization, logging, restart, and cleanup for you).
//
// Usage:
//   go build -o daemon daemon.go
//   ./daemon                    # Starts the daemon
//   cat /var/tmp/daemon.pid     # Shows the daemon PID
//   cat /var/tmp/daemon.log     # Shows the daemon output
//   kill $(cat /var/tmp/daemon.pid)  # Stops the daemon
package main

import (
	"fmt"
	"os"
	"strconv"
	"syscall"
	"time"
)

const (
	pidFile = "/var/tmp/daemon.pid"
	logFile = "/var/tmp/daemon.log"
)

func main() {
	// If the DAEMON_CHILD environment variable is set, we are the
	// daemon child — skip re-launching and go straight to daemon work.
	if os.Getenv("DAEMON_CHILD") == "1" {
		runDaemon()
		return
	}

	// Otherwise, we are the parent. Re-exec ourselves as a daemon.
	// This is Go's idiomatic replacement for fork(): launch a new
	// process with the same binary, using environment variables to
	// distinguish parent from child.
	execPath, err := os.Executable()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Cannot determine executable path: %v\n", err)
		os.Exit(1)
	}

	// Open /dev/null to use as stdin/stdout/stderr for the child.
	devNull, err := os.OpenFile("/dev/null", os.O_RDWR, 0)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Cannot open /dev/null: %v\n", err)
		os.Exit(1)
	}
	defer devNull.Close()

	// Set up the child's environment with our marker variable.
	env := append(os.Environ(), "DAEMON_CHILD=1")

	// Launch the daemon child process.
	// Setsid: true creates a new session, detaching from the terminal.
	attr := &os.ProcAttr{
		Dir:   "/",
		Env:   env,
		Files: []*os.File{devNull, devNull, devNull},
		Sys: &syscall.SysProcAttr{
			Setsid: true, // Create new session — key for daemonization.
		},
	}

	child, err := os.StartProcess(execPath, []string{execPath}, attr)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to start daemon: %v\n", err)
		os.Exit(1)
	}

	// Release the child process — we don't want to wait for it.
	child.Release()
	fmt.Printf("Daemon started with PID %d\n", child.Pid)
	fmt.Printf("Log: %s\n", logFile)
	fmt.Printf("PID file: %s\n", pidFile)
}

// runDaemon is the actual daemon logic, executed in the child process.
func runDaemon() {
	// Write our PID to the PID file so external tools can find us.
	pid := os.Getpid()
	os.WriteFile(pidFile, []byte(strconv.Itoa(pid)+"\n"), 0644)

	// Open a log file since we have no terminal.
	f, err := os.OpenFile(logFile, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		os.Exit(1)
	}
	defer f.Close()

	// Simple daemon loop: write a timestamp every 5 seconds.
	for i := 0; i < 60; i++ { // Run for 5 minutes, then exit.
		fmt.Fprintf(f, "[%s] Daemon (PID %d) is alive, iteration %d\n",
			time.Now().Format(time.RFC3339), pid, i)
		f.Sync()
		time.Sleep(5 * time.Second)
	}

	// Clean up PID file on exit.
	os.Remove(pidFile)
}
```



---

## 1.7 The /proc Filesystem Deep Dive

The `/proc` filesystem is a goldmine of kernel information. Every tool you use daily —
`ps`, `top`, `htop`, `lsof`, `free`, `vmstat` — reads from `/proc`. Understanding its
structure lets you build your own monitoring tools and debug problems that high-level
tools cannot reach.

### Key `/proc/[pid]` Files

**`/proc/[pid]/maps` — Virtual Memory Layout**

Each line describes one Virtual Memory Area (VMA):

```
address           perms offset  dev   inode  pathname
00400000-00452000 r-xp 00000000 08:01 12345  /usr/bin/myapp     ← text segment
00651000-00652000 r--p 00051000 08:01 12345  /usr/bin/myapp     ← rodata
00652000-00653000 rw-p 00052000 08:01 12345  /usr/bin/myapp     ← data segment
01a00000-01a21000 rw-p 00000000 00:00 0      [heap]             ← heap
7f8a1c000000-7f8a1c021000 rw-p 00000000 00:00 0                 ← anonymous mmap
7ffd4b1fe000-7ffd4b200000 r-xp 00000000 00:00 0  [vdso]         ← vDSO
7ffd4b200000-7ffd4b202000 r--p 00000000 00:00 0  [vvar]         ← vDSO data
7fff5c800000-7fff5c821000 rw-p 00000000 00:00 0  [stack]        ← stack
```

Permissions: `r`=read, `w`=write, `x`=execute, `p`=private (COW), `s`=shared.

For Go programs, you will also see many anonymous `rw-p` mappings — these are goroutine
stacks and the Go heap, managed by the Go runtime's memory allocator.

**`/proc/[pid]/status` — Human-Readable Status**

Contains dozens of fields. The most useful:

```
Name:    myapp              ← Executable name (truncated to 15 chars)
Umask:   0022               ← File creation mask
State:   S (sleeping)       ← Process state
Tgid:    12345              ← Thread Group ID (= PID in userspace)
Pid:     12345              ← Kernel PID (= TID for threads)
PPid:    1000               ← Parent PID
Threads: 5                  ← Number of threads
VmPeak:  220000 kB          ← Peak virtual memory size
VmSize:  200000 kB          ← Current virtual memory size
VmRSS:   15000 kB           ← Resident set size (physical memory)
VmData:  50000 kB           ← Data segment size
VmStk:   8192 kB            ← Stack size
VmExe:   2000 kB            ← Text segment size
voluntary_ctxt_switches: 100 ← Voluntarily gave up CPU
nonvoluntary_ctxt_switches: 5← Preempted by scheduler
```

**`/proc/[pid]/fd/` — Open File Descriptors**

A directory containing numbered symlinks. Each symlink name is the fd number, and the
target reveals the underlying resource:

```
0 -> /dev/pts/0                    ← stdin (terminal)
1 -> /dev/pts/0                    ← stdout (terminal)
2 -> /dev/pts/0                    ← stderr (terminal)
3 -> /home/user/data.txt           ← regular file
4 -> socket:[12345]                ← network socket
5 -> pipe:[67890]                  ← pipe
6 -> anon_inode:[eventpoll]        ← epoll fd
7 -> anon_inode:[eventfd]          ← eventfd
```

**`/proc/[pid]/syscall` — Current System Call**

Shows the syscall number and arguments if the process is currently blocked in a syscall:

```
0 0x3 0x7f8a1c000000 0x1000 0x0 0x0 0x0 0x7ffd12345678 0x7f8a1c001234
│  │      │            │
│  │      │            └── count (4096 bytes)
│  │      └── buf pointer
│  └── fd (3)
└── syscall number (0 = read)
```

If the process is running (not in a syscall), this file contains `running`.

**`/proc/stat` — System-Wide Statistics**

The first line shows aggregate CPU time:

```
cpu  12345 678 9012 345678 901 234 567 0 0 0
     │     │   │    │      │   │   │
     │     │   │    │      │   │   └── steal (VM)
     │     │   │    │      │   └── softirq
     │     │   │    │      └── irq (hardware interrupts)
     │     │   │    └── idle
     │     │   └── system (kernel mode)
     │     └── nice (user mode, low priority)
     └── user (user mode, normal priority)
```

**`/proc/meminfo` — System Memory**

```
MemTotal:       16384000 kB    ← Total physical RAM
MemFree:         2000000 kB    ← Truly unused memory
MemAvailable:    8000000 kB    ← Available for allocation (includes reclaimable)
Buffers:          500000 kB    ← Block device cache
Cached:          5000000 kB    ← Page cache
SwapTotal:       4000000 kB    ← Total swap space
SwapFree:        3800000 kB    ← Unused swap
```

### Hands-On: Building a Mini `top` in Go

```go
// minitop.go
//
// A simplified process monitor that reads /proc to display information
// similar to the `top` command. This demonstrates how monitoring tools
// work under the hood — they all read from /proc.
//
// Features:
//   - Lists all running processes with PID, state, name, and memory.
//   - Shows system-wide CPU and memory statistics.
//   - Refreshes every 2 seconds.
//
// This program MUST be run on Linux (it reads /proc).
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"runtime"
	"sort"
	"strconv"
	"strings"
	"time"
)

// ProcessInfo holds the parsed information about a single process,
// extracted from various files under /proc/[pid]/.
type ProcessInfo struct {
	PID     int    // Process ID
	Name    string // Executable name from /proc/[pid]/status
	State   string // Process state (R, S, D, Z, T, etc.)
	VmRSS   int    // Resident set size in kB
	Threads int    // Number of threads
}

// SystemInfo holds system-wide statistics parsed from /proc/stat
// and /proc/meminfo.
type SystemInfo struct {
	MemTotal     int // Total physical memory in kB
	MemAvailable int // Available memory in kB
	MemFree      int // Free memory in kB
	NumProcesses int // Total number of processes
}

func main() {
	if runtime.GOOS != "linux" {
		fmt.Fprintf(os.Stderr, "This program requires Linux (/proc filesystem)\n")
		os.Exit(1)
	}

	for {
		// Clear screen using ANSI escape codes.
		fmt.Print("\033[2J\033[H")

		// Gather system-wide information.
		sysInfo := getSystemInfo()
		fmt.Println("╔══════════════════════════════════════════════════════════╗")
		fmt.Println("║                    Mini Top (Go)                        ║")
		fmt.Println("╠══════════════════════════════════════════════════════════╣")
		fmt.Printf("║ Memory: %d MB total, %d MB available, %d MB free       \n",
			sysInfo.MemTotal/1024, sysInfo.MemAvailable/1024, sysInfo.MemFree/1024)
		fmt.Printf("║ Processes: %d total                                     \n",
			sysInfo.NumProcesses)
		fmt.Println("╠══════════════════════════════════════════════════════════╣")
		fmt.Printf("║ %-8s %-5s %-20s %8s %8s ║\n",
			"PID", "STATE", "NAME", "RSS(kB)", "THREADS")
		fmt.Println("╠══════════════════════════════════════════════════════════╣")

		// Gather per-process information.
		procs := getAllProcesses()

		// Sort by RSS (memory usage) descending — show the biggest first.
		sort.Slice(procs, func(i, j int) bool {
				return procs[i].VmRSS > procs[j].VmRSS
			})

		// Display top 20 processes.
		displayed := 0
		for _, p := range procs {
			if displayed >= 20 {
				break
			}
			name := p.Name
			if len(name) > 20 {
				name = name[:20]
			}
			fmt.Printf("║ %-8d %-5s %-20s %8d %8d ║\n",
				p.PID, p.State, name, p.VmRSS, p.Threads)
			displayed++
		}
		fmt.Println("╚══════════════════════════════════════════════════════════╝")
		fmt.Println("Press Ctrl+C to exit. Refreshing every 2 seconds...")

		time.Sleep(2 * time.Second)
	}
}

// getAllProcesses scans /proc for all numeric directories (each
	// representing a process) and parses their status files.
func getAllProcesses() []ProcessInfo {
	entries, err := os.ReadDir("/proc")
	if err != nil {
		return nil
	}

	var procs []ProcessInfo
	for _, entry := range entries {
		// Only look at numeric directory names (PIDs).
		pid, err := strconv.Atoi(entry.Name())
		if err != nil {
			continue
		}

		proc := parseProcessStatus(pid)
		if proc != nil {
			procs = append(procs, *proc)
		}
	}
	return procs
}

// parseProcessStatus reads /proc/[pid]/status and extracts key fields.
// Returns nil if the process no longer exists or cannot be read.
func parseProcessStatus(pid int) *ProcessInfo {
	statusPath := filepath.Join("/proc", strconv.Itoa(pid), "status")
	data, err := os.ReadFile(statusPath)
	if err != nil {
		return nil // Process may have exited between readdir and readfile.
	}

	info := &ProcessInfo{PID: pid}

	for _, line := range strings.Split(string(data), "\n") {
		parts := strings.SplitN(line, ":\t", 2)
		if len(parts) != 2 {
			continue
		}
		key := strings.TrimSpace(parts[0])
		val := strings.TrimSpace(parts[1])

		switch key {
		case "Name":
			info.Name = val
		case "State":
			// State field looks like "S (sleeping)" — grab just the letter.
			if len(val) > 0 {
				info.State = string(val[0])
			}
		case "VmRSS":
			// Format: "12345 kB" — parse the numeric part.
			fields := strings.Fields(val)
			if len(fields) > 0 {
				info.VmRSS, _ = strconv.Atoi(fields[0])
			}
		case "Threads":
			info.Threads, _ = strconv.Atoi(val)
		}
	}

	return info
}

// getSystemInfo reads /proc/meminfo for system-wide memory statistics.
func getSystemInfo() SystemInfo {
	var info SystemInfo

	// Count processes.
	entries, _ := os.ReadDir("/proc")
	for _, e := range entries {
		if _, err := strconv.Atoi(e.Name()); err == nil {
			info.NumProcesses++
		}
	}

	// Read memory info.
	data, err := os.ReadFile("/proc/meminfo")
	if err != nil {
		return info
	}

	for _, line := range strings.Split(string(data), "\n") {
		parts := strings.SplitN(line, ":", 2)
		if len(parts) != 2 {
			continue
		}
		key := strings.TrimSpace(parts[0])
		val := strings.TrimSpace(parts[1])
		fields := strings.Fields(val)
		if len(fields) == 0 {
			continue
		}
		num, _ := strconv.Atoi(fields[0])

		switch key {
		case "MemTotal":
			info.MemTotal = num
		case "MemAvailable":
			info.MemAvailable = num
		case "MemFree":
			info.MemFree = num
		}
	}

	return info
}
```

**What this teaches you:**

1. Every monitoring tool reads the same `/proc` files. `top`, `htop`, `ps` — they are all
   just parsers for `/proc`.
2. Race conditions are inherent: a process can exit between when you see its directory and
   when you read its files. Always handle `ENOENT` / `ESRCH` gracefully.
3. The `/proc` filesystem is generated on-the-fly by the kernel. Reading `/proc/[pid]/status`
   triggers `proc_pid_status()` in the kernel, which reads the live `task_struct`.



---

## 1.8 Summary and Key Takeaways

This chapter covered the foundational layer that everything in Linux sits on — the process
model. Here are the key concepts you should carry forward:

### Core Concepts

| Concept | Key Insight |
|---|---|
| **Process vs. Program** | A program is code on disk. A process is a running instance with its own `task_struct`, address space, and kernel state. |
| **`task_struct`** | The kernel's representation of every process/thread. Over 600 fields covering memory, scheduling, signals, file descriptors, and more. |
| **Process States** | `TASK_RUNNING` → `TASK_INTERRUPTIBLE` → `EXIT_ZOMBIE` → `EXIT_DEAD`. Understanding states explains `ps` output and debugging hangs (D state = uninterruptible I/O). |
| **fork + exec + wait** | The Unix trinity. `fork()` copies, `exec()` replaces, `wait()` reaps. Go uses `clone(CLONE_VM\|CLONE_VFORK)` + `execve()` for efficiency. |
| **Copy-on-Write** | `fork()` is cheap because pages are shared until written. Page faults trigger the actual copy. |
| **clone() flags** | Linux has no separate "thread" concept. `clone()` with `CLONE_VM\|CLONE_THREAD\|...` creates threads; without those flags, it creates processes. |
| **Go's M:N scheduler** | Goroutines (G) run on OS threads (M) mediated by logical processors (P). `GOMAXPROCS` sets the number of Ps. |
| **System calls** | The `SYSCALL` instruction transitions to Ring 0. Syscall number in RAX, args in registers, return in RAX. |
| **vDSO** | Kernel maps a shared library into userspace for fast read-only syscalls (`clock_gettime`). No mode switch needed. |
| **CFS Scheduler** | Red-black tree ordered by `vruntime`. Tasks with less CPU time get priority. Nice values adjust the accumulation rate. |
| **CPU Affinity** | Pin threads to cores for cache locality. In Go, use `runtime.LockOSThread()` + `unix.SchedSetaffinity()`. |
| **Process Groups & Sessions** | Shells use process groups for job control. `SIGINT` goes to the foreground group. Daemons use `setsid()` to detach. |
| **`/proc` filesystem** | Pseudo-filesystem that exposes live kernel state. Every monitoring tool reads from here. |

### The Syscall Layers in Go

```
  Your Go Code
       │
       ▼
  os, net, time packages      ◄── High-level, portable
       │
       ▼
  syscall package              ◄── Scheduler-aware, frozen API
       │
       ▼
  golang.org/x/sys/unix       ◄── Comprehensive, maintained
       │
       ▼
  runtime/internal/syscall     ◄── Raw, runtime-internal only
       │
       ▼
  SYSCALL instruction          ◄── CPU instruction, Ring 0 transition
       │
       ▼
  sys_call_table[NR]           ◄── Kernel function dispatch
```

### What to Explore Next

- **Chapter 2: Memory Management** — How virtual memory works, page tables, mmap, the Go
  garbage collector's interaction with the kernel.
- **Chapter 3: File I/O and the VFS Layer** — File descriptors, the Virtual Filesystem
  Switch, buffered vs. direct I/O, io_uring.
- **Chapter 4: Signals and Inter-Process Communication** — Signal handling in Go, pipes,
  Unix domain sockets, shared memory.

### Exercises

1. **Zombie Factory:** Write a Go program that intentionally creates zombie processes by
   forking children and not calling `wait()`. Observe them in `ps aux`. Then fix it.

2. **Process Tree:** Write a Go program that reads `/proc` to build and display the
   complete process tree (like `pstree`), showing parent-child relationships.

3. **Syscall Counter:** Use `strace -c` on various Go programs (a simple hello world, an
   HTTP server, a file copier) and compare the syscall profiles. What dominates?

4. **CPU Affinity Benchmark:** Write a benchmark that performs the same computation with
   and without CPU affinity pinning. Measure the difference. Try it on a NUMA system.

5. **Shell Enhancements:** Extend the mini shell from Section 1.2 to support:
   - Pipe (`|`) — create a pipe, fork two children, connect stdout→stdin.
   - Background execution (`&`) — don't wait for the child.
   - `Ctrl+C` handling — forward SIGINT to the child, not the shell.

---

*Next chapter: [Chapter 2: Memory Management & Virtual Address Spaces →](02-memory-management.md)*
