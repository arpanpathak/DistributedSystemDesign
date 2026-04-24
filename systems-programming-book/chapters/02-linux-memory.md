# Chapter 2: Memory Management — Virtual Memory, Paging, mmap

> *"Memory is the treasury and guardian of all things."* — Cicero

Every Go program you write runs atop a sophisticated memory management system that spans
hardware, kernel, and runtime. When you write `make([]byte, 1<<30)` and the allocation
succeeds instantly on a machine with only 512 MB of free RAM, that is virtual memory at
work. When your Go garbage collector pauses for under 100 microseconds while managing
gigabytes of heap, that is decades of OS and runtime engineering cooperating with hardware.

This chapter takes you from silicon to `runtime.MemStats`. We will trace every layer: how
the CPU translates virtual addresses through page tables, how the kernel handles page faults
and implements copy-on-write, how `mmap` and `brk` carve out address space, how Go's
allocator organizes that space into size classes and spans, and how the garbage collector
reclaims it. Along the way, you will write Go programs that peer into `/proc`, measure page
faults, memory-map files, build shared-memory IPC, and tune the garbage collector.

---

## 2.1 Virtual Memory Fundamentals

### 2.1.1 Why Virtual Memory Exists

Before virtual memory, programs accessed physical RAM directly. This created three
insurmountable problems:

1. **No isolation.** Any program could read or write any other program's memory — or the
   OS kernel's memory. A bug in one program could corrupt the entire system.

2. **No flexibility.** Programs had to be loaded at specific physical addresses. If two
   programs both expected to use address `0x400000`, they could not coexist.

3. **No overcommit.** The sum of all programs' memory usage could never exceed physical
   RAM. Even if most pages were never touched, they still consumed real memory.

Virtual memory solves all three problems through a single elegant abstraction: **every
process sees its own private, contiguous address space that maps to physical memory through
a translation table managed by the kernel.** The CPU's Memory Management Unit (MMU)
performs this translation on every single memory access, in hardware, at full speed.

The key insight is that the mapping is **sparse and lazy**:

- A process can have a 128 TB virtual address space while using only 50 MB of physical
  memory.
- Pages can be **demand-allocated**: the kernel records the mapping but allocates no
  physical memory until the page is actually accessed, triggering a page fault.
- Pages can be **shared**: two processes can map the same physical frame (e.g., shared
  libraries), saving memory.
- Pages can be **swapped**: the kernel can evict rarely-used pages to disk, reclaiming
  physical memory for active pages.

For Go developers, virtual memory is especially relevant because:

- **Go's garbage collector** relies on virtual address space being cheap. The runtime
  reserves large virtual arenas upfront without consuming physical memory.
- **`mmap`** is how Go's runtime (and your programs) interact with the OS for file I/O,
  shared memory, and large allocations.
- **Stack growth** uses virtual memory tricks — Go allocates small initial stacks and grows
  them by allocating new virtual pages.

### 2.1.2 Virtual Address Space Layout

On x86_64 Linux, a process has a 48-bit virtual address space (256 TB), split between
user space and kernel space. Here is the canonical layout:

```text
    Virtual Address Space Layout (x86_64 Linux, 48-bit addressing)

    0xFFFF_FFFF_FFFF_FFFF  ┌─────────────────────────────────┐
                            │                                 │
                            │         Kernel Space            │
                            │    (direct-mapped physical      │
                            │     memory, vmalloc, modules,   │
                            │     per-CPU data, etc.)         │
                            │                                 │
    0xFFFF_8000_0000_0000  ├─────────────────────────────────┤
                            │                                 │
                            │    Non-canonical hole           │
                            │    (addresses with bits 47-63   │
                            │     not all-same cause #GP)     │
                            │                                 │
    0x0000_7FFF_FFFF_FFFF  ├─────────────────────────────────┤
                            │         Stack                   │
                            │    (grows downward ↓)           │
                            │    [RSP]                        │
                            ├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
                            │                                 │
                            │    (random gap — ASLR)          │
                            │                                 │
                            ├─────────────────────────────────┤
                            │    Memory-mapped region          │
                            │    (mmap, shared libraries,     │
                            │     file mappings — grows ↓)    │
                            │                                 │
                            │    e.g., libc.so, ld-linux.so   │
                            │    e.g., Go runtime mmap arenas │
                            ├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┤
                            │                                 │
                            │    (unmapped gap)               │
                            │                                 │
                            ├─────────────────────────────────┤
                            │    Heap                          │
                            │    (grows upward ↑ via brk)     │
                            │    [program break]              │
                            ├─────────────────────────────────┤
                            │    BSS Segment                  │
                            │    (uninitialized globals,      │
                            │     zero-filled on demand)      │
                            ├─────────────────────────────────┤
                            │    Data Segment                 │
                            │    (initialized globals,        │
                            │     static variables)           │
                            ├─────────────────────────────────┤
                            │    Text Segment                 │
                            │    (executable code, read-only) │
    0x0000_0000_0040_0000  ├─────────────────────────────────┤
                            │    (unmapped — null ptr guard)  │
    0x0000_0000_0000_0000  └─────────────────────────────────┘
```

**Segment details:**

| Segment | Contents | Permissions | Notes |
|---------|----------|-------------|-------|
| **Text** | Machine code (`.text`) | `r-x` | Shared across `fork()` children; read-only to prevent code injection |
| **Data** | Initialized globals (`.data`) | `rw-` | `var x = 42` in Go ends up here |
| **BSS** | Zero-initialized globals (`.bss`) | `rw-` | `var buf [4096]byte` — no space in binary, zero-filled at load |
| **Heap** | Dynamic allocations | `rw-` | Managed by `brk()`/`sbrk()`; Go rarely uses this directly |
| **mmap region** | File mappings, shared libs, anon maps | varies | Go's runtime allocates most memory here via `mmap` |
| **Stack** | Function call frames, locals | `rw-` | Main goroutine OS stack; goroutine stacks are on the heap |
| **Kernel** | Kernel code and data | `---` (user) | Mapped into every process for fast syscalls; inaccessible from user mode |

**Important note for Go developers:** Unlike C programs, Go does not heavily use the
traditional heap (managed by `brk`). Instead, Go's runtime uses large `mmap` allocations
to create its own heap, which it then subdivides using its TCMalloc-inspired allocator.
Goroutine stacks are also allocated from this mmap-based heap, not from the OS stack.

### 2.1.3 Page Tables — The Translation Machinery

The CPU does not understand virtual addresses natively. Every memory access goes through
the MMU, which walks a **page table** — a tree structure in physical memory — to translate
virtual addresses to physical addresses.

On x86_64, Linux uses **4-level page tables** (5-level with LA57, but 4-level is standard):

```text
    4-Level Page Table Walk (x86_64)

    Virtual Address (48 bits used of 64):
    ┌────────┬────────┬────────┬────────┬──────────────┐
    │ PML4   │  PDPT  │   PD   │   PT   │   Offset     │
    │ [47:39]│ [38:30]│ [29:21]│ [20:12]│   [11:0]     │
    │ 9 bits │ 9 bits │ 9 bits │ 9 bits │  12 bits     │
    └───┬────┴───┬────┴───┬────┴───┬────┴──────┬───────┘
        │        │        │        │           │
        │   ┌────┘        │        │           │
        │   │    ┌────────┘        │           │
        │   │    │    ┌────────────┘           │
        ▼   ▼    ▼    ▼                        │
    ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐    │
    │ CR3  │─▶│PML4  │─▶│PDPT  │─▶│ PD   │─▶│ PT   │
    │(reg) │  │Entry │  │Entry │  │Entry │  │Entry │
    └──────┘  │[512] │  │[512] │  │[512] │  │[512] │
              └──────┘  └──────┘  └──────┘  └──┬───┘
                                                │
                                                ▼
                                          Physical Frame
                                          Address + Offset
                                               │
                                               ▼
                                        ┌──────────────┐
                                        │Physical Addr │
                                        └──────────────┘
```

Each level contains **512 entries** (9 bits of index × 8 bytes per entry = 4 KB per table
page — exactly one page). A Page Table Entry (PTE) at the lowest level has this format:

```text
    x86_64 Page Table Entry (PTE) Format — 64 bits

    Bit(s)   Name              Description
    ──────   ────              ───────────
    0        Present (P)       1 = page is in physical memory
                               0 = page fault on access
    1        Read/Write (R/W)  1 = writable, 0 = read-only
    2        User/Super (U/S)  1 = user-mode accessible
    3        Write-Through     1 = write-through caching
    4        Cache Disable     1 = disable caching for this page
    5        Accessed (A)      Set by CPU when page is read
    6        Dirty (D)         Set by CPU when page is written
    7        PAT / Page Size   Page Attribute Table index
    8        Global (G)        1 = don't flush from TLB on CR3 switch
    11:9     Available         OS can use these bits freely
    12:51    Physical Address  Physical frame number (shifted left by 12)
    62:52    Available         More OS-usable bits
    63       NX (No Execute)   1 = page is not executable (W^X enforcement)
```

**Key bits for systems programmers:**

- **Present bit (0):** When clear, any access causes a page fault. This is how demand
  paging works — the kernel sets up the PTE with present=0, and only allocates a physical
  frame when the page is first accessed.

- **Dirty bit (6):** The CPU sets this automatically when a write occurs. The kernel uses
  this to know which pages need to be written back to disk (for file-backed mappings or
  swap).

- **Accessed bit (5):** The CPU sets this on any access (read or write). The kernel uses
  this for page reclamation — pages not recently accessed are candidates for eviction.

- **NX bit (63):** Prevents code execution from data pages. This is critical for security
  (W^X policy: a page is either Writable or eXecutable, never both).

**Cost of page table walks:** A full 4-level walk requires 4 memory accesses just to
translate one virtual address. At ~100 ns per memory access, this would make every memory
operation take 400+ ns — unacceptable. This is where the TLB comes in.

### 2.1.4 TLB — Translation Lookaside Buffer

The TLB is a small, fast cache inside the CPU that stores recent virtual-to-physical
translations. It is typically:

- **L1 dTLB:** 64 entries, 4-way set associative, 1-cycle lookup
- **L1 iTLB:** 128 entries for instructions
- **L2 TLB:** 1536 entries, 12-way set associative, ~7 cycles

When the CPU accesses a virtual address:

```text
    TLB Lookup Flow

    Virtual Address
         │
         ▼
    ┌──────────┐     Hit
    │   TLB    │──────────▶ Physical Address (1 cycle)
    │  Lookup  │
    └────┬─────┘
         │ Miss
         ▼
    ┌──────────────┐
    │  Page Table   │
    │  Walk (HW)    │──────▶ Physical Address (~100-400 cycles)
    │  4 memory     │         + TLB entry populated
    │  accesses     │
    └──────────────┘
```

**TLB shootdown** is one of the most expensive operations in a multiprocessor system.
When one CPU modifies a page table entry (e.g., unmapping a page), it must ensure that
all other CPUs flush that entry from their TLBs. This is done via Inter-Processor
Interrupts (IPIs), which force remote CPUs to invalidate their TLB entries. This is why
`munmap` and `mprotect` on shared mappings can be surprisingly expensive on many-core
machines.

**Go relevance:** Go's garbage collector must update page metadata and sometimes change
page protections. The cost of TLB shootdowns is one reason the GC tries to minimize
the number of page-level protection changes.


### 2.1.5 Hands-On: Examining /proc/self/maps from Go

Let us write a Go program that reads and parses its own memory map, revealing the virtual
address space layout we discussed above.

```go
// cmd/procmaps/main.go
//
// procmaps reads /proc/self/maps to display the current process's virtual
// memory layout. Each line in /proc/self/maps describes one Virtual Memory
// Area (VMA) — a contiguous range of virtual addresses with uniform
// permissions and backing.
//
// The format of each line is:
//
//	address           perms offset  dev   inode   pathname
//	00400000-0040b000 r-xp 00000000 08:01 1234567 /usr/bin/foo
//
// This program parses each field and categorizes the region by type
// (stack, heap, mmap, vDSO, etc.) to help you understand what occupies
// your process's address space.
package main

import (
	"bufio"
	"fmt"
	"os"
	"strconv"
	"strings"
)

// VMA represents a single Virtual Memory Area parsed from /proc/self/maps.
// Each VMA is a contiguous range of pages with the same permissions and backing.
type VMA struct {
	// StartAddr is the starting virtual address of this region (inclusive).
	StartAddr uint64
	// EndAddr is the ending virtual address of this region (exclusive).
	EndAddr uint64
	// Perms encodes the page permissions: r=read, w=write, x=execute, p=private/s=shared.
	Perms string
	// Offset is the offset into the file (for file-backed mappings) or 0 for anonymous.
	Offset uint64
	// Device is the major:minor device number of the backing file.
	Device string
	// Inode is the inode number of the backing file (0 for anonymous mappings).
	Inode uint64
	// Pathname is the file path, or a special label like [stack], [heap], [vdso].
	Pathname string
}

// Size returns the size of this VMA in bytes.
func (v VMA) Size() uint64 {
	return v.EndAddr - v.StartAddr
}

// SizeHuman returns a human-readable size string (e.g., "4 KB", "2 MB").
func (v VMA) SizeHuman() string {
	size := v.Size()
	switch {
	case size >= 1<<30:
		return fmt.Sprintf("%.1f GB", float64(size)/float64(1<<30))
	case size >= 1<<20:
		return fmt.Sprintf("%.1f MB", float64(size)/float64(1<<20))
	case size >= 1<<10:
		return fmt.Sprintf("%.1f KB", float64(size)/float64(1<<10))
	default:
		return fmt.Sprintf("%d B", size)
	}
}

// Category returns a human-readable classification of this VMA based on its
// pathname and permissions. This helps identify what each region is used for.
func (v VMA) Category() string {
	switch {
	case v.Pathname == "[stack]":
		return "STACK"
	case v.Pathname == "[heap]":
		return "HEAP"
	case v.Pathname == "[vdso]":
		return "vDSO (virtual dynamic shared object)"
	case v.Pathname == "[vvar]":
		return "vVAR (kernel variables exported to userspace)"
	case v.Pathname == "[vsyscall]":
		return "VSYSCALL (legacy fast syscall page)"
	case strings.HasSuffix(v.Pathname, ".so") || strings.Contains(v.Pathname, ".so."):
		return "SHARED LIBRARY"
	case v.Pathname != "" && v.Inode != 0:
		return "FILE MAPPING"
	case v.Pathname == "" && v.Inode == 0:
		return "ANONYMOUS (mmap or runtime allocation)"
	default:
		return "OTHER"
	}
}

// parseMapsLine parses a single line from /proc/self/maps into a VMA struct.
// Returns an error if the line format is unexpected.
func parseMapsLine(line string) (VMA, error) {
	// Split by whitespace. The pathname field may contain spaces or be absent.
	fields := strings.Fields(line)
	if len(fields) < 5 {
		return VMA{}, fmt.Errorf("too few fields in line: %s", line)
	}

	// Parse the address range "start-end".
	addrParts := strings.Split(fields[0], "-")
	if len(addrParts) != 2 {
		return VMA{}, fmt.Errorf("invalid address range: %s", fields[0])
	}
	start, err := strconv.ParseUint(addrParts[0], 16, 64)
	if err != nil {
		return VMA{}, fmt.Errorf("invalid start address: %v", err)
	}
	end, err := strconv.ParseUint(addrParts[1], 16, 64)
	if err != nil {
		return VMA{}, fmt.Errorf("invalid end address: %v", err)
	}

	// Parse offset.
	offset, err := strconv.ParseUint(fields[2], 16, 64)
	if err != nil {
		return VMA{}, fmt.Errorf("invalid offset: %v", err)
	}

	// Parse inode.
	inode, err := strconv.ParseUint(fields[4], 10, 64)
	if err != nil {
		return VMA{}, fmt.Errorf("invalid inode: %v", err)
	}

	// Pathname is optional — join remaining fields if present.
	pathname := ""
	if len(fields) > 5 {
		pathname = strings.Join(fields[5:], " ")
	}

	return VMA{
		StartAddr: start,
		EndAddr:   end,
		Perms:     fields[1],
		Offset:    offset,
		Device:    fields[3],
		Inode:     inode,
		Pathname:  pathname,
	}, nil
}

func main() {
	// Open our own memory map. Every Linux process can read /proc/self/maps
	// to discover its virtual memory layout.
	f, err := os.Open("/proc/self/maps")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to open /proc/self/maps: %v\n", err)
		fmt.Fprintf(os.Stderr, "This program must run on Linux.\n")
		os.Exit(1)
	}
	defer f.Close()

	// Summary counters: track total virtual memory by category.
	categorySizes := make(map[string]uint64)
	var totalVirtual uint64

	fmt.Println("=== Virtual Memory Map of Current Process ===")
	fmt.Println()
	fmt.Printf("%-34s %-5s %10s  %-s\n", "Address Range", "Perms", "Size", "Category / Path")
	fmt.Println(strings.Repeat("-", 100))

	scanner := bufio.NewScanner(f)
	for scanner.Scan() {
		vma, err := parseMapsLine(scanner.Text())
		if err != nil {
			fmt.Fprintf(os.Stderr, "Warning: %v\n", err)
			continue
		}

		category := vma.Category()
		categorySizes[category] += vma.Size()
		totalVirtual += vma.Size()

		// Format the address range.
		addrRange := fmt.Sprintf("0x%012x-0x%012x", vma.StartAddr, vma.EndAddr)

		// Print this VMA.
		path := vma.Pathname
		if path == "" {
			path = "(anonymous)"
		}
		fmt.Printf("%-34s %-5s %10s  [%s] %s\n",
			addrRange, vma.Perms, vma.SizeHuman(), category, path)
	}

	fmt.Println()
	fmt.Println("=== Summary by Category ===")
	for cat, size := range categorySizes {
		fmt.Printf("  %-45s %10.1f MB\n", cat, float64(size)/float64(1<<20))
	}
	fmt.Printf("  %-45s %10.1f MB\n", "TOTAL VIRTUAL", float64(totalVirtual)/float64(1<<20))
}
```

When you run this program on Linux, you will see output similar to:

```
=== Virtual Memory Map of Current Process ===

Address Range                      Perms       Size  Category / Path
----------------------------------------------------------------------------------------------------
0x000000400000-0x000000480000      r-xp     512.0 KB  [FILE MAPPING] /path/to/procmaps
0x000000480000-0x0000004e0000      r--p     384.0 KB  [FILE MAPPING] /path/to/procmaps
0x0000004e0000-0x000000500000      rw-p     128.0 KB  [FILE MAPPING] /path/to/procmaps
0x000000c00000-0x000000c21000      rw-p     132.0 KB  [HEAP]
0x00007f1234000000-0x00007f1234200000 rw-p   2.0 MB  [ANONYMOUS (mmap or runtime allocation)]
...
0x00007fff12300000-0x00007fff12321000 rw-p   132.0 KB  [STACK]
0x00007fff12321000-0x00007fff12323000 r--p     8.0 KB  [vVAR (kernel variables exported to userspace)]
0x00007fff12323000-0x00007fff12325000 r-xp     8.0 KB  [vDSO (virtual dynamic shared object)]
```

Note that Go programs typically show large anonymous `mmap` regions — these are the Go
runtime's heap arenas. The Go runtime pre-reserves virtual address space in large chunks
(often hundreds of megabytes) but only commits physical memory as needed.

---

## 2.2 Page Faults — The Heart of Virtual Memory

Page faults are the mechanism that makes virtual memory work. Without page faults, every
virtual page would need a physical frame allocated upfront, defeating the purpose of
virtual memory. Page faults are **not errors** — they are a normal, essential part of the
memory management system.

### 2.2.1 Minor vs Major Page Faults

There are two types of page faults:

**Minor page faults** occur when:
- The page exists in physical memory but the page table entry has not been set up yet.
- The kernel just needs to update the page table — no disk I/O required.
- Example: First access to a page in an anonymous mmap region. The kernel allocates a
  zeroed physical frame and maps it.
- Typical cost: **1–10 microseconds**.

**Major page faults** occur when:
- The page is not in physical memory and must be read from disk.
- Example: Accessing a page that was swapped out, or first accessing a page in a
  file-backed mmap region.
- Typical cost: **1–10 milliseconds** (1000x slower than minor faults due to disk I/O).

```
    Page Fault Classification

    CPU accesses virtual address
              │
              ▼
    ┌───────────────────┐
    │  PTE Present = 0  │
    │  → Page Fault!    │
    └────────┬──────────┘
             │
             ▼
    ┌───────────────────────┐
    │  Is page in physical  │──── Yes ──▶ MINOR FAULT
    │  memory (page cache   │           (update PTE, ~1-10 μs)
    │  or just not mapped)? │
    └────────┬──────────────┘
             │ No
             ▼
    ┌───────────────────────┐
    │  Is page on swap or   │──── Yes ──▶ MAJOR FAULT
    │  in a file on disk?   │           (read from disk, ~1-10 ms)
    └────────┬──────────────┘
             │ No
             ▼
    ┌───────────────────────┐
    │  Is address valid in  │──── No ──▶ SIGSEGV (segfault)
    │  any VMA?             │           (process killed)
    └────────┬──────────────┘
             │ Yes
             ▼
    ┌───────────────────────┐
    │  Permission check:    │──── Fail ──▶ SIGSEGV
    │  Does VMA allow this  │            (e.g., write to read-only)
    │  access type?         │
    └────────┬──────────────┘
             │ Pass
             ▼
        MINOR FAULT
    (allocate frame, map it)
```

### 2.2.2 Demand Paging — Lazy Allocation

When you call `mmap` to create an anonymous mapping, the kernel does **not** allocate any
physical memory. It merely creates a VMA (Virtual Memory Area) data structure recording
that this range of virtual addresses is valid and describes its permissions and backing.

Physical memory is only allocated when you actually **access** a page within that range,
one page at a time. This is demand paging:

1. Program calls `mmap(NULL, 1GB, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)`
2. Kernel creates a VMA: "addresses 0x7f0000000000 to 0x7f0040000000 are valid, read-write,
   anonymous, private." No physical memory allocated. Returns instantly.
3. Program writes to address `0x7f0000001000` (page 1 of the mapping).
4. MMU walks the page table, finds no valid PTE → page fault.
5. Kernel's page fault handler:
   a. Looks up the VMA for this address — found, permissions match.
   b. Allocates one physical frame (4 KB).
   c. Zeroes the frame (security: prevent data leaks from previous processes).
   d. Creates a PTE mapping virtual page → physical frame.
   e. Returns to userspace — the instruction retries and succeeds.
6. The other 262,143 pages in the 1 GB mapping remain unmapped and use zero physical memory.

This is why Go's runtime can reserve huge virtual address spaces cheaply. The `GOGC` and
memory arena system relies on the fact that reserving virtual space is essentially free.

### 2.2.3 Copy-on-Write (COW)

Copy-on-Write is one of the most important optimizations in the kernel. When `fork()`
creates a child process, the kernel does **not** copy the parent's memory. Instead:

1. Both parent and child share the same physical pages.
2. All shared pages are marked **read-only** in both processes' page tables.
3. When either process writes to a shared page, a page fault occurs.
4. The kernel allocates a new physical frame, copies the old page's contents, maps the new
   frame into the writing process's page table as read-write, and the original page stays
   with the other process.

```
    Copy-on-Write Mechanism

    After fork():
    ┌──────────┐     ┌──────────┐
    │  Parent   │     │  Child   │
    │  PTE: RO  │────▶│  PTE: RO │
    └──────────┘  │  └──────────┘
                  │
                  ▼
            ┌──────────┐
            │ Physical  │  refcount = 2
            │  Frame    │
            └──────────┘

    After parent writes to the page:
    ┌──────────┐     ┌──────────┐
    │  Parent   │     │  Child   │
    │  PTE: RW  │     │  PTE: RO │
    └────┬─────┘     └────┬─────┘
         │                │
         ▼                ▼
    ┌──────────┐    ┌──────────┐
    │  New      │    │ Original │  refcount = 1
    │  Frame    │    │  Frame   │
    │ (copy)    │    │          │
    └──────────┘    └──────────┘
```

COW is especially important for Go programs that use `exec.Command` — under the hood,
Go calls `fork()` then `exec()`. Without COW, forking a Go process using 2 GB of memory
would require another 2 GB just for the fork.

### 2.2.4 Page Fault Handler Flow in the Kernel

When a page fault occurs, the CPU saves the faulting address in the `CR2` register and
transfers control to the kernel's page fault handler (`do_page_fault` in the Linux kernel).
Here is the simplified flow:

```
    do_page_fault(regs, error_code)
        │
        ├─▶ Was fault in kernel mode?
        │   └─ Yes: check exception tables → fixup or oops/panic
        │
        ├─▶ find_vma(mm, address)
        │   └─ No VMA found? → SIGSEGV
        │
        ├─▶ Is fault address < vma->vm_start?
        │   └─ Yes: Is it a stack VMA that can expand? → expand_stack()
        │         └─ No: SIGSEGV
        │
        ├─▶ Check permissions (vma->vm_flags vs fault type)
        │   └─ Mismatch? → SIGSEGV
        │
        └─▶ handle_mm_fault(vma, address, flags)
            │
            ├─▶ Walk page table levels, allocating intermediate tables as needed
            │
            ├─▶ Is this an anonymous page (first access)?
            │   └─ alloc_zeroed_user_highpage() → map it in
            │
            ├─▶ Is this a COW fault (write to shared page)?
            │   └─ Copy page, remap as writable
            │
            └─▶ Is this a file-backed page?
                └─ Call the filesystem's ->fault() handler
                   └─ Read page from disk into page cache → map it in
```

### 2.2.5 Hands-On: Measuring Page Faults with getrusage from Go

The `getrusage` system call reports resource usage statistics for the calling process,
including page fault counts. Let us write a Go program that deliberately triggers page
faults and measures them.

```go
// cmd/pagefaults/main.go
//
// pagefaults demonstrates how to measure minor and major page faults using
// the getrusage(2) system call. It allocates a large block of memory via
// mmap and then touches each page, triggering demand-paged minor faults.
//
// This program shows the direct relationship between touching pages and
// the minor fault counter incrementing — each new page touched causes
// exactly one minor fault.
//
// Usage:
//
//	go build -o pagefaults ./cmd/pagefaults
//	./pagefaults
//
// For more detail, run under perf:
//
//	perf stat -e page-faults,minor-faults,major-faults ./pagefaults
package main

import (
	"fmt"
	"os"
	"runtime"
	"syscall"
	"unsafe"
)

// getPageFaults returns the current minor and major page fault counts
// for this process by calling getrusage(RUSAGE_SELF).
//
// Minor faults occur when the kernel maps a page without disk I/O.
// Major faults occur when the kernel must read from disk (swap or file).
func getPageFaults() (minor, major int64, err error) {
	var rusage syscall.Rusage
	// RUSAGE_SELF (0) reports usage for the calling process.
	err = syscall.Getrusage(syscall.RUSAGE_SELF, &rusage)
	if err != nil {
		return 0, 0, fmt.Errorf("getrusage failed: %w", err)
	}
	// Minflt = minor faults (page in memory, just not mapped)
	// Majflt = major faults (page had to be fetched from disk)
	return int64(rusage.Minflt), int64(rusage.Majflt), nil
}

func main() {
	// Lock this goroutine to a single OS thread to get consistent
	// page fault measurements (rusage counts are per-thread on Linux).
	runtime.LockOSThread()

	const (
		// Allocate 64 MB of memory. At 4 KB per page, that is 16,384 pages.
		allocSize = 64 * 1024 * 1024
		pageSize  = 4096
		numPages  = allocSize / pageSize
	)

	// Snapshot page faults BEFORE allocation.
	minBefore, majBefore, err := getPageFaults()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("Before mmap: minor=%d, major=%d\n", minBefore, majBefore)

	// Allocate memory using mmap. This creates a virtual mapping but does
	// NOT allocate any physical memory yet (demand paging).
	data, err := syscall.Mmap(
		-1,                                   // fd: -1 for anonymous mapping
		0,                                    // offset: 0
		allocSize,                            // length: 64 MB
		syscall.PROT_READ|syscall.PROT_WRITE, // permissions
		syscall.MAP_PRIVATE|syscall.MAP_ANONYMOUS, // flags
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "mmap failed: %v\n", err)
		os.Exit(1)
	}
	defer syscall.Munmap(data)

	// Snapshot page faults AFTER mmap but BEFORE touching pages.
	// mmap itself should cause zero or very few additional page faults
	// because demand paging means no physical memory is allocated yet.
	minAfterMmap, majAfterMmap, err := getPageFaults()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("After mmap (before touch): minor=%d (+%d), major=%d (+%d)\n",
		minAfterMmap, minAfterMmap-minBefore,
		majAfterMmap, majAfterMmap-majBefore)

	// Now touch every page by writing one byte per page. Each first write
	// to a page triggers a minor page fault, causing the kernel to:
	//   1. Allocate a physical frame
	//   2. Zero-fill it
	//   3. Map it into our page table
	//   4. Return to our code (the write instruction retries and succeeds)
	//
	// We use unsafe.Slice to avoid bounds checking overhead on the inner loop
	// (the mmap returned a []byte, but we want to stride by pageSize).
	ptr := unsafe.Pointer(&data[0])
	for i := 0; i < numPages; i++ {
		// Write to the first byte of each page.
		*(*byte)(unsafe.Add(ptr, i*pageSize)) = 0xFF
	}

	// Snapshot page faults AFTER touching all pages.
	minAfterTouch, majAfterTouch, err := getPageFaults()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("After touching %d pages: minor=%d (+%d), major=%d (+%d)\n",
		numPages,
		minAfterTouch, minAfterTouch-minAfterMmap,
		majAfterTouch, majAfterTouch-majAfterMmap)

	// The number of new minor faults should be very close to numPages (16,384),
	// because each page touch causes exactly one minor fault.
	fmt.Printf("\nExpected ~%d minor faults from page touches, got %d\n",
		numPages, minAfterTouch-minAfterMmap)
}
```

**Expected output:**
```
Before mmap: minor=1234, major=0
After mmap (before touch): minor=1235 (+1), major=0 (+0)
After touching 16384 pages: minor=17620 (+16385), major=0 (+0)

Expected ~16384 minor faults from page touches, got 16385
```

The extra fault or two comes from the mmap metadata itself. The key observation is:
**16,384 pages touched = ~16,384 minor faults.** Each page fault costs approximately
1–5 microseconds on modern hardware.

### 2.2.6 Hands-On: Triggering and Observing Page Faults with perf stat

For even more detailed analysis, use `perf stat` to count hardware and software events:

```bash
# Build the program
go build -o pagefaults ./cmd/pagefaults

# Run with perf stat to count page fault events
sudo perf stat -e page-faults,minor-faults,major-faults,dTLB-load-misses,dTLB-store-misses \
    ./pagefaults
```

**Expected perf output:**
```
 Performance counter stats for './pagefaults':

            16,512      page-faults
            16,512      minor-faults
                 0      major-faults
           185,234      dTLB-load-misses
            42,108      dTLB-store-misses

       0.028374521 seconds time elapsed
```

Notice that TLB misses exceed page faults — the TLB is small (typically 64–1536 entries),
so accessing 16,384 pages causes many TLB evictions and refills even after the pages are
mapped.

---

## 2.3 Memory Allocation

### 2.3.1 brk() and sbrk() — The Traditional Heap

The oldest way to allocate dynamic memory on Unix is `brk()`/`sbrk()`, which move the
**program break** — the boundary between the heap and unmapped memory:

```
    brk() / sbrk() Mechanism

    Before allocation:
    ┌─────────┬──────────┬───────────────────────┐
    │  Text   │   Data   │   Heap   │ break ▼    │
    │  (.text)│  (.data) │          │(unmapped)   │
    └─────────┴──────────┴──────────┴────────────┘
                                     ↑
                                  program break

    After sbrk(4096):
    ┌─────────┬──────────┬────────────────────────┐
    │  Text   │   Data   │   Heap (expanded) │brk▼│
    │  (.text)│  (.data) │                   │    │
    └─────────┴──────────┴───────────────────┴────┘
                                              ↑
                                          new program break
```

`brk()` is simple but limited:
- It can only grow or shrink the heap from one end.
- It cannot create mappings at arbitrary addresses.
- It cannot create file-backed mappings.
- It cannot release memory in the middle of the heap (fragmentation).

**Go does not use brk() for its heap.** Go's runtime uses `mmap` exclusively, which gives
it far more flexibility. However, understanding `brk` helps you understand legacy C
programs and tools like `strace` output.

### 2.3.2 mmap() — The Swiss Army Knife

`mmap()` is the most important memory allocation syscall in modern Linux. It creates a new
mapping in the calling process's virtual address space:

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
```

**Parameters:**
- `addr`: Hint for where to place the mapping (usually `NULL` to let the kernel choose).
- `length`: Size of the mapping in bytes (rounded up to page size).
- `prot`: Protection flags (`PROT_READ`, `PROT_WRITE`, `PROT_EXEC`, `PROT_NONE`).
- `flags`: Type and behavior of the mapping.
- `fd`: File descriptor for file-backed mappings (-1 for anonymous).
- `offset`: Offset into the file.

### 2.3.3 mmap Flags Deep Dive

| Flag | Value | Description |
|------|-------|-------------|
| `MAP_PRIVATE` | | Changes are private to the process (COW). Writes are not reflected in the underlying file. |
| `MAP_SHARED` | | Changes are shared with other processes mapping the same file. Writes propagate to the file. |
| `MAP_ANONYMOUS` | | No file backing — memory is initialized to zero. Used for general-purpose memory allocation. |
| `MAP_FIXED` | | Place mapping at exactly `addr`. Dangerous: silently unmaps anything already there. |
| `MAP_HUGETLB` | | Use huge pages (2 MB or 1 GB). Requires kernel configuration. |
| `MAP_POPULATE` | | Pre-fault all pages immediately. Avoids minor faults on first access but increases mmap latency. |
| `MAP_NORESERVE` | | Don't reserve swap space. The mapping may cause OOM on access if memory is tight. |
| `MAP_LOCKED` | | Lock pages in memory (equivalent to `mlock` after mapping). Prevents swapping. |
| `MAP_GROWSDOWN` | | Used for stack-like mappings that grow downward. |

**Common combinations:**

```
Anonymous private memory (like malloc):
  MAP_PRIVATE | MAP_ANONYMOUS

Shared memory between processes:
  MAP_SHARED | MAP_ANONYMOUS

Memory-mapped file (read-only, shared, e.g., shared libraries):
  MAP_SHARED  with fd pointing to the file

Memory-mapped file (private copy, e.g., reading a config file):
  MAP_PRIVATE with fd pointing to the file

Pre-faulted anonymous memory (avoid page fault latency):
  MAP_PRIVATE | MAP_ANONYMOUS | MAP_POPULATE
```

### 2.3.4 mprotect() — Changing Page Permissions

`mprotect()` changes the protection on a region of memory after it has been mapped:

```c
int mprotect(void *addr, size_t len, int prot);
```

This is used for:
- **Guard pages**: Set `PROT_NONE` on a page to detect buffer overflows. Any access causes
  a segfault.
- **JIT compilation**: Allocate memory as `PROT_READ|PROT_WRITE`, write machine code into
  it, then `mprotect` to `PROT_READ|PROT_EXEC` before jumping to it.
- **Garbage collectors**: Some GCs use `mprotect` to implement write barriers in hardware.

Go's runtime uses guard pages between goroutine stacks to detect stack overflow.

### 2.3.5 madvise() — Advising the Kernel

`madvise()` lets you hint to the kernel about your memory access patterns:

```c
int madvise(void *addr, size_t length, int advice);
```

| Advice | Effect |
|--------|--------|
| `MADV_NORMAL` | Default behavior — no special treatment. |
| `MADV_SEQUENTIAL` | Expect sequential access. Kernel aggressively reads ahead. |
| `MADV_RANDOM` | Expect random access. Kernel disables readahead. |
| `MADV_WILLNEED` | Will need this range soon. Kernel starts readahead now. |
| `MADV_DONTNEED` | Don't need this range. Kernel may reclaim the pages immediately. For anonymous private mappings, the pages are freed and will be zero-filled on next access. |
| `MADV_HUGEPAGE` | Enable transparent huge pages for this range. |
| `MADV_NOHUGEPAGE` | Disable transparent huge pages for this range. |
| `MADV_FREE` | Pages can be reclaimed when memory is needed (lazy DONTNEED). Kernel reclaims only under memory pressure. |

**Go's runtime uses `MADV_DONTNEED` (or `MADV_FREE` on Linux 4.5+) to release physical
memory back to the OS without unmapping the virtual address range.** This is how Go can
show reduced RSS (Resident Set Size) after a GC cycle without calling `munmap`.


### 2.3.6 Hands-On: Using mmap from Go with syscall.Mmap

Go's `syscall` package provides direct access to `mmap`. Let us explore various mapping
types:

```go
// cmd/mmapbasic/main.go
//
// mmapbasic demonstrates the fundamental mmap operations available from Go:
// anonymous private mappings, file-backed mappings, and shared mappings.
//
// Each example shows a different use case for mmap and explains the kernel
// behavior behind the scenes.
package main

import (
	"fmt"
	"os"
	"syscall"
	"unsafe"
)

// demonstrateAnonymousMmap creates an anonymous private mapping — the most
// common type used by memory allocators. No file is involved; the kernel
// provides zero-filled pages on demand.
//
// This is equivalent to what Go's runtime does internally when it needs
// more memory from the OS for its heap arenas.
func demonstrateAnonymousMmap() {
	fmt.Println("=== Anonymous Private mmap ===")

	// Request 1 MB of anonymous private memory.
	// - fd=-1, offset=0 because there is no backing file.
	// - MAP_PRIVATE means writes are not shared.
	// - MAP_ANONYMOUS means no file backing.
	//
	// After this call, 1 MB of virtual address space is reserved,
	// but ZERO physical memory is consumed (demand paging).
	data, err := syscall.Mmap(
		-1, 0,
		1024*1024, // 1 MB
		syscall.PROT_READ|syscall.PROT_WRITE,
		syscall.MAP_PRIVATE|syscall.MAP_ANONYMOUS,
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "mmap failed: %v\n", err)
		return
	}
	// Always unmap when done to release the virtual address range and any
	// physical pages. Go's GC does NOT track mmap allocations.
	defer syscall.Munmap(data)

	// The first read should return zero (anonymous pages are zero-filled).
	fmt.Printf("First byte (should be 0): %d\n", data[0])

	// Write a pattern. This triggers a page fault on first access to each
	// page, causing the kernel to allocate a physical frame.
	for i := range data {
		data[i] = byte(i % 256)
	}

	fmt.Printf("Wrote %d bytes. Last byte: %d\n", len(data), data[len(data)-1])
	fmt.Printf("Virtual address of mapping: %p\n", unsafe.Pointer(&data[0]))
	fmt.Println()
}

// demonstrateFileMmap maps a file into memory, allowing you to access file
// contents as if they were a byte array. The kernel handles all I/O — reads
// are satisfied from the page cache, and there is no need for explicit
// read()/write() calls.
//
// This is the basis of "zero-copy I/O" — data goes directly from the page
// cache to your process's virtual address space without any copying.
func demonstrateFileMmap() {
	fmt.Println("=== File-backed mmap (read-only) ===")

	// Open /proc/self/maps as a demonstration file.
	// (On a real system, you might map a large data file.)
	f, err := os.Open("/proc/self/maps")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Cannot open file: %v\n", err)
		return
	}
	defer f.Close()

	// Get file size for the mapping.
	info, err := f.Stat()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Cannot stat file: %v\n", err)
		return
	}
	size := int(info.Size())
	if size == 0 {
		// /proc files report size 0; use a fixed buffer instead.
		fmt.Println("Proc file reports size 0; skipping mmap demo for this file.")
		fmt.Println("On a real file, mmap would map the file contents directly.")
		return
	}

	// Map the file into memory. MAP_PRIVATE means writes (if allowed) create
	// private copies and do NOT modify the underlying file.
	data, err := syscall.Mmap(
		int(f.Fd()), 0,
		size,
		syscall.PROT_READ,
		syscall.MAP_PRIVATE,
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "mmap failed: %v\n", err)
		return
	}
	defer syscall.Munmap(data)

	fmt.Printf("Mapped %d bytes of file into memory at %p\n",
		size, unsafe.Pointer(&data[0]))

	// Read the first 200 bytes as a string.
	preview := size
	if preview > 200 {
		preview = 200
	}
	fmt.Printf("First %d bytes:\n%s\n", preview, string(data[:preview]))
	fmt.Println()
}

func main() {
	demonstrateAnonymousMmap()
	demonstrateFileMmap()
}
```

### 2.3.7 Hands-On: Memory-Mapped File I/O in Go — Reading a Large File via mmap

For high-performance file processing, `mmap` can be significantly faster than `read()`
because it avoids copying data from the kernel's page cache to a userspace buffer. The
pages are shared directly.

```go
// cmd/mmapfile/main.go
//
// mmapfile demonstrates high-performance file processing using mmap.
// It maps a file into memory and performs analysis (counting lines and
// bytes) without any explicit read() system calls.
//
// The kernel handles all I/O transparently through page faults:
// when we access a byte that is not yet in memory, a page fault occurs,
// the kernel reads the corresponding 4 KB page from disk into the page
// cache, maps it into our address space, and our code continues without
// ever knowing a page fault happened.
//
// Advantages over read():
//   - Zero-copy: data is accessed directly in the page cache
//   - No userspace buffer management
//   - Kernel handles readahead automatically
//   - Multiple accesses to the same region reuse cached pages
//
// Disadvantages:
//   - Not suitable for files larger than virtual address space
//   - Error handling is harder (SIGBUS on I/O errors instead of read() returning -1)
//   - Cannot easily handle files that grow while being read
//
// Usage:
//
//	go build -o mmapfile ./cmd/mmapfile
//	./mmapfile /path/to/large/file
package main

import (
	"fmt"
	"os"
	"syscall"
	"time"
)

// mmapFile maps the entire contents of the file at the given path into
// memory and returns the mapped byte slice. The caller must call
// syscall.Munmap on the returned slice when done.
//
// The mapping is read-only and private (MAP_PRIVATE), meaning:
//   - We can only read, not write (PROT_READ only).
//   - Even if we could write, changes would not affect the file.
//
// We also call madvise(MADV_SEQUENTIAL) to hint that we will read the
// file sequentially, which enables aggressive kernel readahead.
func mmapFile(path string) ([]byte, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, fmt.Errorf("open %s: %w", path, err)
	}
	defer f.Close()

	info, err := f.Stat()
	if err != nil {
		return nil, fmt.Errorf("stat %s: %w", path, err)
	}
	size := int(info.Size())
	if size == 0 {
		return nil, fmt.Errorf("file %s is empty", path)
	}

	// Map the file into memory.
	data, err := syscall.Mmap(
		int(f.Fd()), 0, size,
		syscall.PROT_READ,
		syscall.MAP_PRIVATE,
	)
	if err != nil {
		return nil, fmt.Errorf("mmap %s: %w", path, err)
	}

	// Advise the kernel that we will access this mapping sequentially.
	// This tells the kernel to aggressively prefetch pages ahead of our
	// current read position, significantly reducing page fault latency.
	//
	// MADV_SEQUENTIAL typically causes the kernel to:
	//   1. Double the normal readahead window.
	//   2. Free pages behind the current access point sooner.
	_ = syscall.Madvise(data, syscall.MADV_SEQUENTIAL)

	return data, nil
}

func main() {
	if len(os.Args) < 2 {
		fmt.Fprintf(os.Stderr, "Usage: %s <file>\n", os.Args[0])
		os.Exit(1)
	}
	path := os.Args[1]

	// Map the file.
	start := time.Now()
	data, err := mmapFile(path)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
	defer syscall.Munmap(data)
	mapTime := time.Since(start)

	// Count lines and bytes by scanning the mapped memory.
	// No read() calls — we just iterate over the byte slice.
	start = time.Now()
	lines := 0
	for _, b := range data {
		if b == '\n' {
			lines++
		}
	}
	scanTime := time.Since(start)

	fmt.Printf("File:       %s\n", path)
	fmt.Printf("Size:       %d bytes (%.1f MB)\n", len(data), float64(len(data))/1e6)
	fmt.Printf("Lines:      %d\n", lines)
	fmt.Printf("mmap time:  %v\n", mapTime)
	fmt.Printf("Scan time:  %v\n", scanTime)
	fmt.Printf("Throughput: %.0f MB/s\n", float64(len(data))/scanTime.Seconds()/1e6)
}
```

### 2.3.8 Hands-On: Implementing a Simple Memory-Mapped Ring Buffer in Go

A ring buffer (circular buffer) is a fundamental data structure in systems programming,
used for inter-process communication, logging, and buffering. Using `mmap`, we can
implement a clever trick: map the same physical memory twice, back-to-back, so that
wrapping reads and writes become simple linear operations.

```go
// cmd/mmapring/main.go
//
// mmapring implements a ring buffer backed by mmap. The key insight is that
// we map the same shared memory region twice at consecutive virtual addresses,
// creating a "mirror" that eliminates wrap-around logic.
//
// Traditional ring buffer (wrapping required):
//
//	Physical:  [ A B C _ _ X Y Z ]
//	                     ^       ^
//	                    write   read
//	Reading "XYZ" + "ABC" requires two separate copy operations because
//	the data wraps around the end of the buffer.
//
// mmap double-mapping trick:
//
//	Virtual:   [ A B C _ _ X Y Z | A B C _ _ X Y Z ]
//	                               ^               ^
//	                    Same physical memory mapped twice!
//	Now reading "XYZABC" is a single contiguous read from the second 'X'
//	to the second 'C', even though it logically wraps around.
//
// This technique is used in production by logging systems (e.g., the Linux
// kernel's perf ring buffer) and high-performance message queues.
package main

import (
	"fmt"
	"os"
	"syscall"
	"unsafe"
)

// RingBuffer is a circular buffer backed by a double-mapped mmap region.
// The double mapping means that reads and writes that cross the end of
// the buffer appear contiguous in virtual memory, eliminating the need
// for wrap-around logic.
type RingBuffer struct {
	// data is the double-mapped region: data[0:size] and data[size:2*size]
	// map to the same physical memory.
	data []byte
	// size is the actual buffer capacity in bytes (must be a multiple of page size).
	size int
	// writePos is the next write position (monotonically increasing).
	writePos uint64
	// readPos is the next read position (monotonically increasing).
	readPos uint64
	// fd is the memfd file descriptor backing the shared memory.
	fd int
}

// NewRingBuffer creates a ring buffer of the given size (rounded up to page size).
//
// Implementation:
//  1. Create a memfd (anonymous file in memory) of the desired size.
//  2. mmap the memfd at a chosen address for size*2 bytes (to reserve space).
//  3. mmap the memfd again at address+size, overlapping the second half.
//     This makes both halves of the virtual mapping point to the same physical pages.
//
// Note: This simplified version uses a basic approach. A production implementation
// would use MAP_FIXED to ensure the two mappings are adjacent.
func NewRingBuffer(requestedSize int) (*RingBuffer, error) {
	// Round up to page size.
	pageSize := os.Getpagesize()
	size := (requestedSize + pageSize - 1) &^ (pageSize - 1)

	// Create an anonymous file in memory using memfd_create equivalent.
	// We use a temporary file in /dev/shm for portability.
	name := fmt.Sprintf("/dev/shm/ringbuf-%d", os.Getpid())
	f, err := os.Create(name)
	if err != nil {
		return nil, fmt.Errorf("create shm: %w", err)
	}
	// Unlink immediately — the file stays open via fd but is invisible in the filesystem.
	os.Remove(name)

	fd := int(f.Fd())

	// Set the file size. This allocates the physical pages in the tmpfs.
	if err := syscall.Ftruncate(fd, int64(size)); err != nil {
		f.Close()
		return nil, fmt.Errorf("ftruncate: %w", err)
	}

	// Step 1: Reserve a contiguous virtual address range of 2*size.
	// We use MAP_ANONYMOUS|MAP_PRIVATE to get the address, then overwrite
	// with our file-backed mappings.
	reserved, err := syscall.Mmap(-1, 0, size*2,
		syscall.PROT_NONE,
		syscall.MAP_PRIVATE|syscall.MAP_ANONYMOUS)
	if err != nil {
		f.Close()
		return nil, fmt.Errorf("mmap reserve: %w", err)
	}
	baseAddr := uintptr(unsafe.Pointer(&reserved[0]))

	// Step 2: Map the first half to our memfd.
	_, _, errno := syscall.Syscall6(syscall.SYS_MMAP,
		baseAddr, uintptr(size),
		uintptr(syscall.PROT_READ|syscall.PROT_WRITE),
		uintptr(syscall.MAP_SHARED|syscall.MAP_FIXED),
		uintptr(fd), 0)
	if errno != 0 {
		syscall.Munmap(reserved)
		f.Close()
		return nil, fmt.Errorf("mmap first half: %v", errno)
	}

	// Step 3: Map the second half to the SAME memfd at offset 0.
	// This creates the "mirror" — both halves map to the same physical pages.
	_, _, errno = syscall.Syscall6(syscall.SYS_MMAP,
		baseAddr+uintptr(size), uintptr(size),
		uintptr(syscall.PROT_READ|syscall.PROT_WRITE),
		uintptr(syscall.MAP_SHARED|syscall.MAP_FIXED),
		uintptr(fd), 0)
	if errno != 0 {
		syscall.Munmap(reserved)
		f.Close()
		return nil, fmt.Errorf("mmap second half: %v", errno)
	}

	// Build the double-mapped slice spanning both halves.
	data := unsafe.Slice((*byte)(unsafe.Pointer(baseAddr)), size*2)

	return &RingBuffer{
		data: data,
		size: size,
		fd:   fd,
	}, nil
}

// Write writes data into the ring buffer. Returns the number of bytes written.
// If the buffer is full, it overwrites the oldest data.
func (rb *RingBuffer) Write(p []byte) int {
	n := len(p)
	if n > rb.size {
		// If the write is larger than the buffer, only keep the last 'size' bytes.
		p = p[n-rb.size:]
		n = rb.size
	}
	// Thanks to the double mapping, this copy never wraps.
	// We write starting at writePos % size, and if the data extends past
	// the end of the first mapping, it seamlessly continues in the second
	// mapping which is the same physical memory — so it appears as if
	// we wrapped around to the beginning.
	offset := int(rb.writePos % uint64(rb.size))
	copy(rb.data[offset:offset+n], p)
	rb.writePos += uint64(n)
	return n
}

// Read reads up to len(p) bytes from the ring buffer.
// Returns the number of bytes read.
func (rb *RingBuffer) Read(p []byte) int {
	available := int(rb.writePos - rb.readPos)
	if available <= 0 {
		return 0
	}
	n := len(p)
	if n > available {
		n = available
	}
	// Again, the double mapping means this copy is always contiguous.
	offset := int(rb.readPos % uint64(rb.size))
	copy(p[:n], rb.data[offset:offset+n])
	rb.readPos += uint64(n)
	return n
}

// Close releases all resources (unmaps memory, closes fd).
func (rb *RingBuffer) Close() error {
	// Munmap the double-sized region.
	if err := syscall.Munmap(rb.data); err != nil {
		return err
	}
	return syscall.Close(rb.fd)
}

func main() {
	// Create a 4 KB ring buffer (1 page).
	rb, err := NewRingBuffer(4096)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create ring buffer: %v\n", err)
		os.Exit(1)
	}
	defer rb.Close()

	// Write data that wraps around.
	message1 := []byte("Hello, ring buffer! This is a test of the double-mapped mmap trick. ")
	message2 := []byte("Data written past the end wraps around seamlessly. ")

	for i := 0; i < 100; i++ {
		rb.Write(message1)
		rb.Write(message2)
	}

	// Read the last chunk of data.
	buf := make([]byte, 200)
	n := rb.Read(buf)
	fmt.Printf("Read %d bytes: %s\n", n, string(buf[:n]))
	fmt.Printf("Total bytes written: %d\n", rb.writePos)
	fmt.Printf("Buffer size: %d bytes\n", rb.size)
	fmt.Println("Double-mapped ring buffer working correctly!")
}
```

---

## 2.4 Go's Memory Allocator (TCMalloc-Inspired)

Go's memory allocator is inspired by TCMalloc (Thread-Caching Malloc) from Google. It is
designed for high concurrency: each P (processor, in Go's scheduler terminology) has its
own local cache, minimizing lock contention.

### 2.4.1 mcache, mcentral, mheap Hierarchy

```
    Go Memory Allocator Hierarchy

    ┌─────────────────────────────────────────────────────┐
    │                    OS (Linux Kernel)                 │
    │              mmap / madvise / munmap                 │
    └───────────────────────┬─────────────────────────────┘
                            │
                            ▼
    ┌─────────────────────────────────────────────────────┐
    │                      mheap                          │
    │  (Global heap — one per process)                    │
    │                                                     │
    │  - Manages large spans of pages                     │
    │  - Requests memory from OS via mmap                 │
    │  - Divides memory into spans (mspan)                │
    │  - Each span is a contiguous run of pages           │
    │  - Free spans tracked in treap by size              │
    │  - Lock: mheap.lock (global, relatively rare)       │
    └───────────────┬─────────────────────────────────────┘
                    │
                    ▼
    ┌─────────────────────────────────────────────────────┐
    │                   mcentral                          │
    │  (One per size class — ~70 size classes)            │
    │                                                     │
    │  - Manages spans for a specific size class          │
    │  - Has two span lists:                              │
    │    • partial: spans with free objects               │
    │    • full: spans with no free objects               │
    │  - Lock: per-mcentral (one lock per size class)     │
    │  - When mcache needs objects, it gets a span        │
    │    from mcentral (or requests from mheap)           │
    └───────────────┬─────────────────────────────────────┘
                    │
                    ▼
    ┌────────────┐ ┌────────────┐ ┌────────────┐
    │  mcache    │ │  mcache    │ │  mcache    │
    │  (per-P)   │ │  (per-P)   │ │  (per-P)   │
    │            │ │            │ │            │
    │  - No locks│ │  - No locks│ │  - No locks│
    │  - Array of│ │  - Array of│ │  - Array of│
    │   ~70 size │ │   ~70 size │ │   ~70 size │
    │   class    │ │   class    │ │   class    │
    │   free     │ │   free     │ │   free     │
    │   lists    │ │   lists    │ │   lists    │
    └────────────┘ └────────────┘ └────────────┘
         P0             P1             P2
```

**Allocation flow for small objects (≤ 32 KB):**

1. Determine the **size class** for the requested size. Go has ~70 size classes ranging
   from 8 bytes to 32 KB, with each class wasting at most ~12.5% memory.
2. Check the **mcache** for the current P. Each mcache has a free list for each size class.
   If there is a free object, return it immediately. **No lock required.**
3. If the mcache's free list is empty for this size class, refill it from the **mcentral**
   for that size class. This requires a lock, but only on that specific mcentral (not a
   global lock).
4. If the mcentral has no free spans, request a new span from the **mheap**. This requires
   the global heap lock.
5. If the mheap has no free spans of the right size, request pages from the OS via `mmap`.

**Allocation flow for large objects (> 32 KB):**
- Allocate directly from the **mheap**, bypassing mcache and mcentral.
- The mheap finds a suitable span (contiguous pages) or requests memory from the OS.

### 2.4.2 Size Classes and Span Management

An **mspan** is a contiguous run of pages managed by the Go allocator. Each span is
dedicated to a single size class:

```
    mspan Example: Size Class 5 (48 bytes)

    ┌──────────────────────────────────────────┐
    │                mspan                     │
    │  startAddr: 0xc000100000                 │
    │  npages:    1 (4096 bytes)               │
    │  elemsize:  48 bytes                     │
    │  nelems:    85 (4096/48 = 85.3, so 85)   │
    │  freeindex: 12 (next free slot to check) │
    │  allocBits: bitmap tracking used slots   │
    └──────────────────────────────────────────┘

    Memory layout within the span:
    ┌────┬────┬────┬────┬────┬────┬─...─┬────┬─────┐
    │ 48B│ 48B│ 48B│ 48B│ 48B│ 48B│     │ 48B│waste│
    │ #0 │ #1 │ #2 │ #3 │ #4 │ #5 │     │#84 │16B  │
    └────┴────┴────┴────┴────┴────┴─...─┴────┴─────┘
    ▲ used  ▲ used  ▲ free  ▲ used  ...
```

The size classes are carefully chosen to minimize internal fragmentation. Here are some
examples:

| Size Class | Object Size | Span Size | Objects per Span | Waste |
|------------|-------------|-----------|------------------|-------|
| 1 | 8 B | 8 KB | 1024 | 0% |
| 2 | 16 B | 8 KB | 512 | 0% |
| 5 | 48 B | 8 KB | 170 | ~0.4% |
| 10 | 128 B | 8 KB | 64 | 0% |
| 25 | 1 KB | 8 KB | 8 | 0% |
| 35 | 4 KB | 8 KB | 2 | 0% |
| 67 | 32 KB | 32 KB | 1 | 0% |

### 2.4.3 How Go Requests Memory from the OS

Go's mheap requests memory from the Linux kernel exclusively through `mmap`:

```go
// Simplified version of how Go requests memory from the OS.
// (runtime/mem_linux.go)
//
// sysAlloc allocates a new chunk of memory from the operating system.
// It uses mmap with MAP_ANONYMOUS | MAP_PRIVATE to get zero-filled pages.
//
// The hint parameter suggests a virtual address — the runtime tries to
// keep its heap in a contiguous region for efficiency.
func sysAlloc(n uintptr, hint *uintptr) unsafe.Pointer {
	p, err := mmap(unsafe.Pointer(*hint), n,
		_PROT_READ|_PROT_WRITE,
		_MAP_ANON|_MAP_PRIVATE,
		-1, 0)
	if err != 0 {
		return nil
	}
	return p
}
```

The runtime also uses:
- `madvise(MADV_DONTNEED)` to release physical memory without unmapping (after GC).
- `madvise(MADV_HUGEPAGE)` to request transparent huge pages for heap arenas.
- `mprotect(PROT_NONE)` for guard pages and unused arena space.

### 2.4.4 Stack Growth — Contiguous Stacks

Go goroutines start with a small stack (currently **2 KB on Go 1.22+**, was 8 KB earlier).
When a function's prologue detects that the stack is too small (the "stack check"),
Go allocates a new, larger stack (typically 2x the old size), copies the old stack contents,
updates all pointers, and frees the old stack.

```
    Goroutine Stack Growth

    1. Initial state (2 KB stack):
    ┌──────────────────────┐
    │  main()              │  ← SP (stack pointer)
    │  foo()               │
    │  (free space)        │
    │  ──── stack guard ───│  ← stack limit check
    └──────────────────────┘
    2 KB

    2. foo() calls bar() which needs more space:
       Stack check: SP < stackguard → need to grow!

    3. Runtime allocates new 4 KB stack:
    ┌──────────────────────────────────────────────┐
    │  main()              │  ← copied from old     │
    │  foo()               │  ← copied from old     │
    │  bar()               │  ← new frame           │
    │  (free space)        │                        │
    │  (free space)        │                        │
    │  ──── stack guard ───│                        │
    └──────────────────────────────────────────────┘
    4 KB

    4. All pointers into the old stack are updated to point into the new stack.
       Old stack is freed.
```

**History:** Before Go 1.4, Go used **segmented stacks** — each new segment was linked
to the previous one. This caused the "hot split" problem: if a function oscillated between
needing and not needing an extra segment, the overhead of allocating/freeing segments on
every call was devastating. Go 1.4 switched to contiguous stacks, eliminating this problem.

### 2.4.5 Escape Analysis — What Goes on Heap vs Stack

Go's compiler performs **escape analysis** to determine whether a variable can be allocated
on the stack (cheap, freed automatically when the function returns) or must be allocated on
the heap (requires GC to reclaim).

**A variable escapes to the heap if:**
- Its address is returned from the function.
- Its address is stored in a heap-allocated object.
- It is captured by a closure that outlives the function.
- It is sent to a channel.
- It is assigned to an interface value (in some cases).
- It is too large for the stack.

**Stack allocation is always preferred** because:
- No GC overhead — the entire stack frame is freed at once when the function returns.
- Better locality — stack data is hot in the CPU cache.
- No allocation overhead — just decrementing the stack pointer.

### 2.4.6 Hands-On: Using `go build -gcflags="-m"` to See Escape Analysis

```go
// cmd/escape/main.go
//
// escape demonstrates Go's escape analysis. Run this program's build with:
//
//	go build -gcflags="-m -m" ./cmd/escape
//
// to see the compiler's escape analysis decisions, including WHY each
// variable escapes or stays on the stack.
//
// The double -m flag produces verbose output explaining the reasoning.
package main

import "fmt"

// stackOnly returns an integer by value. The local variable 'x' never
// escapes — it lives entirely on the stack and is freed when the function
// returns. This is the ideal case.
//
// Escape analysis output: "x does not escape"
func stackOnly() int {
	x := 42
	return x // value copy — x stays on stack
}

// escapesToHeap returns a pointer to a local variable. Since the caller
// receives a pointer that outlives this function's stack frame, the
// compiler must allocate 'x' on the heap instead.
//
// Escape analysis output: "moved to heap: x"
func escapesToHeap() *int {
	x := 42
	return &x // &x escapes — x must be heap-allocated
}

// interfaceEscape demonstrates that assigning to an interface can cause
// escape. The compiler cannot always prove that the interface value won't
// be stored somewhere long-lived.
//
// Escape analysis output: "y escapes to heap"
func interfaceEscape() interface{} {
	y := "hello"
	return y // y escapes because interface{} might be stored anywhere
}

// closureCapture demonstrates that variables captured by closures escape
// if the closure itself escapes (e.g., is returned or stored).
//
// Escape analysis output: "moved to heap: counter"
func closureCapture() func() int {
	counter := 0
	return func() int {
		counter++ // counter is captured by the closure
		return counter
	}
}

// noEscapeClosure shows that a closure that does NOT outlive the
// enclosing function does not cause its captured variables to escape.
//
// Escape analysis output: "counter does not escape"
func noEscapeClosure() int {
	counter := 0
	increment := func() {
		counter++
	}
	increment()
	increment()
	return counter // closure does not escape, so counter stays on stack
}

// largeStackAlloc shows that very large allocations may be forced to the
// heap even without escaping, because the stack has limited space.
//
// The exact threshold depends on the Go version, but generally objects
// larger than ~64 KB are moved to the heap.
func largeStackAlloc() {
	// This 1 MB array is too large for the stack → heap allocated.
	var big [1024 * 1024]byte
	big[0] = 1
	_ = big
}

func main() {
	a := stackOnly()
	b := escapesToHeap()
	c := interfaceEscape()
	d := closureCapture()
	e := noEscapeClosure()

	fmt.Println(a, *b, c, d(), e)

	largeStackAlloc()
}
```

**Build with escape analysis output:**

```bash
$ go build -gcflags="-m -m" ./cmd/escape 2>&1 | head -30

./cmd/escape/main.go:28:2: x does not escape
./cmd/escape/main.go:36:2: moved to heap: x
./cmd/escape/main.go:36:9: &x escapes to heap
./cmd/escape/main.go:36:9:   flow: ~r0 = &x:
./cmd/escape/main.go:36:9:     from return &x (return) at ./cmd/escape/main.go:36:2
./cmd/escape/main.go:44:2: moved to heap: y
./cmd/escape/main.go:55:2: moved to heap: counter
./cmd/escape/main.go:56:9: func literal escapes to heap
./cmd/escape/main.go:64:2: counter does not escape
./cmd/escape/main.go:65:15: func literal does not escape
```

### 2.4.7 Hands-On: runtime.MemStats — Monitoring Go's Memory Usage

```go
// cmd/memstats/main.go
//
// memstats demonstrates how to use runtime.MemStats to monitor Go's
// memory allocator and garbage collector in detail. This is the primary
// tool for understanding your Go program's memory behavior.
//
// runtime.MemStats provides over 30 fields covering:
//   - Total allocations and frees
//   - Heap size and utilization
//   - Stack usage
//   - GC statistics (pause times, cycles, CPU usage)
//   - Per-size-class allocation counts
//
// Usage:
//
//	go run ./cmd/memstats
package main

import (
	"fmt"
	"runtime"
	"strings"
)

// printMemStats reads and displays the current runtime.MemStats.
// Each field is explained with its significance for performance analysis.
func printMemStats(label string) {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)

	fmt.Printf("\n=== MemStats: %s ===\n", label)
	fmt.Println(strings.Repeat("-", 60))

	// --- General Statistics ---
	fmt.Println("General:")
	// Sys = total bytes of memory obtained from the OS.
	// This includes everything: heap, stacks, GC metadata, etc.
	fmt.Printf("  Sys          = %10d bytes (%6.1f MB) — total from OS\n",
		m.Sys, float64(m.Sys)/1e6)
	// TotalAlloc = cumulative bytes allocated (monotonically increasing).
	// Does NOT decrease when memory is freed. Useful for allocation rate.
	fmt.Printf("  TotalAlloc   = %10d bytes (%6.1f MB) — cumulative allocs\n",
		m.TotalAlloc, float64(m.TotalAlloc)/1e6)
	// Mallocs and Frees = cumulative count of allocations and frees.
	// Mallocs - Frees = number of live objects.
	fmt.Printf("  Mallocs      = %10d — cumulative allocation count\n", m.Mallocs)
	fmt.Printf("  Frees        = %10d — cumulative free count\n", m.Frees)
	fmt.Printf("  LiveObjects  = %10d — currently live (Mallocs-Frees)\n",
		m.Mallocs-m.Frees)

	// --- Heap Statistics ---
	fmt.Println("\nHeap:")
	// HeapAlloc = bytes of live objects on the heap (reachable + not yet swept).
	// This is the most useful metric for "how much memory is my app using?"
	fmt.Printf("  HeapAlloc    = %10d bytes (%6.1f MB) — live heap objects\n",
		m.HeapAlloc, float64(m.HeapAlloc)/1e6)
	// HeapSys = bytes of heap memory obtained from the OS.
	// This is the virtual address space reserved for the heap.
	fmt.Printf("  HeapSys      = %10d bytes (%6.1f MB) — heap virtual space\n",
		m.HeapSys, float64(m.HeapSys)/1e6)
	// HeapIdle = bytes in idle (unused) spans. These spans have no objects
	// and could be returned to the OS.
	fmt.Printf("  HeapIdle     = %10d bytes (%6.1f MB) — idle spans\n",
		m.HeapIdle, float64(m.HeapIdle)/1e6)
	// HeapInuse = bytes in spans that have at least one live object.
	// HeapInuse - HeapAlloc = fragmentation + span overhead.
	fmt.Printf("  HeapInuse    = %10d bytes (%6.1f MB) — in-use spans\n",
		m.HeapInuse, float64(m.HeapInuse)/1e6)
	// HeapReleased = bytes of memory returned to the OS (via madvise DONTNEED).
	// This memory is still mapped but the OS can reclaim the physical pages.
	fmt.Printf("  HeapReleased = %10d bytes (%6.1f MB) — returned to OS\n",
		m.HeapReleased, float64(m.HeapReleased)/1e6)
	// HeapObjects = number of allocated heap objects.
	fmt.Printf("  HeapObjects  = %10d — allocated objects\n", m.HeapObjects)

	// --- Stack ---
	fmt.Println("\nStacks:")
	fmt.Printf("  StackInuse   = %10d bytes (%6.1f MB) — goroutine stacks\n",
		m.StackInuse, float64(m.StackInuse)/1e6)
	fmt.Printf("  StackSys     = %10d bytes (%6.1f MB) — stack system bytes\n",
		m.StackSys, float64(m.StackSys)/1e6)

	// --- GC Statistics ---
	fmt.Println("\nGC:")
	fmt.Printf("  NumGC        = %10d — completed GC cycles\n", m.NumGC)
	fmt.Printf("  NumForcedGC  = %10d — forced GC cycles (runtime.GC())\n", m.NumForcedGC)
	fmt.Printf("  GCCPUFraction= %10.4f%% — fraction of CPU used by GC\n",
		m.GCCPUFraction*100)
	// PauseTotalNs = cumulative nanoseconds in GC STW pauses.
	fmt.Printf("  PauseTotalNs = %10d ns (%.3f ms) — total STW pause\n",
		m.PauseTotalNs, float64(m.PauseTotalNs)/1e6)
	// Last pause time.
	if m.NumGC > 0 {
		lastPause := m.PauseNs[(m.NumGC+255)%256]
		fmt.Printf("  LastPauseNs  = %10d ns (%.3f ms) — last STW pause\n",
			lastPause, float64(lastPause)/1e6)
	}

	fmt.Println(strings.Repeat("-", 60))
}

func main() {
	printMemStats("Initial")

	// Allocate a bunch of small objects to exercise the allocator.
	// These 1 million 64-byte objects will use the size class allocator
	// (mcache → mcentral → mheap path).
	var ptrs []*[64]byte
	for i := 0; i < 1_000_000; i++ {
		p := new([64]byte)
		ptrs = append(ptrs, p)
	}
	printMemStats("After 1M allocations")

	// Release references and force GC.
	ptrs = nil
	runtime.GC()
	printMemStats("After GC")

	// Allocate large objects (> 32 KB each) — these bypass the size class
	// allocator and go directly to the mheap.
	var bigPtrs []*[1024 * 1024]byte
	for i := 0; i < 100; i++ {
		p := new([1024 * 1024]byte) // 1 MB each
		bigPtrs = append(bigPtrs, p)
	}
	printMemStats("After 100 × 1MB allocations")

	bigPtrs = nil
	runtime.GC()
	printMemStats("After final GC")
}
```

---

## 2.5 Go's Garbage Collector

Go uses a **concurrent, tri-color, mark-and-sweep** garbage collector. It is designed to
minimize stop-the-world (STW) pauses while providing predictable latency — critical for
server applications.

### 2.5.1 Tri-Color Mark-and-Sweep

The GC uses three colors to classify objects:

```
    Tri-Color Marking

    WHITE = not yet visited (potentially garbage)
    GRAY  = visited but children not yet scanned
    BLACK = visited and all children scanned (definitely live)

    Start:  All objects are WHITE
            Root set → mark as GRAY

    Step 1: Pick a GRAY object
    Step 2: Scan all its pointers
            - If pointer → WHITE object, mark it GRAY
    Step 3: Mark the scanned object BLACK
    Repeat until no GRAY objects remain

    End:    BLACK objects = live (keep)
            WHITE objects = garbage (sweep/free)

    Example progression:

    Roots: [A, B]

    Initial:    A(W) → C(W) → E(W)
                B(W) → D(W)

    After marking roots gray:
                A(G) → C(W) → E(W)
                B(G) → D(W)

    After scanning A:
                A(B) → C(G) → E(W)
                B(G) → D(W)

    After scanning B:
                A(B) → C(G) → E(W)
                B(B) → D(G)

    After scanning C:
                A(B) → C(B) → E(G)
                B(B) → D(G)

    After scanning D, E:
                A(B) → C(B) → E(B)
                B(B) → D(B)

    Any remaining WHITE objects are garbage.
```

### 2.5.2 Write Barriers

The key challenge of **concurrent** GC is that the program (the "mutator") continues
running while the GC is scanning. If the mutator modifies pointers during marking, the
GC might miss reachable objects.

Go solves this with a **write barrier** — a small piece of code injected by the compiler
at every pointer write. When GC is active, the write barrier ensures that if a pointer is
written that might hide an object from the GC, that object is marked gray.

Go uses a **hybrid write barrier** (since Go 1.8) that combines:
- **Dijkstra-style insertion barrier**: When storing a new pointer, shade the new referent.
- **Yuasa-style deletion barrier**: When overwriting a pointer, shade the old referent.

The hybrid barrier eliminates the need for stack re-scanning, which was a major source of
STW pause time in earlier Go versions.

```go
// Pseudocode for Go's hybrid write barrier.
// This runs on every pointer store: *slot = ptr
//
// writePointer is the compiler-generated write barrier that ensures
// GC correctness during concurrent marking. Without this, the mutator
// could hide live objects from the GC by moving pointers between
// already-scanned (black) and not-yet-scanned (white) objects.
func writePointer(slot *unsafe.Pointer, ptr unsafe.Pointer) {
	// Shade the object being overwritten (deletion barrier).
	// This prevents the "lost object" problem where:
	//   1. GC scans object A (marks it black)
	//   2. Mutator copies A.ptr to B.ptr (B is white)
	//   3. Mutator sets A.ptr = nil
	//   4. GC scans B, but B.ptr was set after A was scanned
	//   → The object that A.ptr used to point to is now unreachable
	//     from the GC's perspective, even though B.ptr points to it.
	shade(*slot)

	// Shade the new object (insertion barrier).
	// This handles the case where a new pointer is stored into
	// an already-scanned (black) object.
	if isStackPointer(slot) {
		shade(ptr) // For stack slots, shade the new value
	}

	// Perform the actual pointer write.
	*slot = ptr
}
```

### 2.5.3 GC Pacing — GOGC and GOMEMLIMIT

Go's GC uses a **pacing algorithm** to decide when to start a GC cycle. The goal is to
complete the GC cycle before the heap grows too much, while not wasting too much CPU on
GC work.

**GOGC** (default: 100) controls the **heap growth ratio**:
- `GOGC=100` means: start a new GC cycle when heap has grown by 100% since the last cycle.
- `GOGC=200` means: allow the heap to triple before GC (less GC, more memory).
- `GOGC=50` means: allow the heap to grow by 50% (more GC, less memory).
- `GOGC=off` disables GC entirely (dangerous in production!).

```
    GOGC Pacing Example (GOGC=100)

    After GC #1: live heap = 100 MB
    Goal:         next GC at 200 MB (100% growth = 100 MB + 100 MB)
    GC #2 starts when heap reaches ~200 MB

    After GC #2: live heap = 80 MB  (some objects were freed)
    Goal:         next GC at 160 MB
    ...
```

**GOMEMLIMIT** (Go 1.19+) sets a **soft memory limit** for the entire Go runtime:
- The GC will work harder (run more frequently) to stay under this limit.
- It will NOT crash if the limit is exceeded — it is advisory.
- When the heap approaches GOMEMLIMIT, the GC may run almost continuously.

**Best practice:** Set `GOMEMLIMIT` to ~80-90% of your container's memory limit, and set
`GOGC=off` or a very high value. This tells Go: "use as much memory as you want up to the
limit, and only GC when you need to."

### 2.5.4 STW Phases — How Go Minimizes Them

Go's GC has two brief STW (Stop-The-World) phases:

1. **Mark Setup** (~10-30 μs): Enable the write barrier, start background mark workers.
2. **Mark Termination** (~10-30 μs): Disable the write barrier, finalize marking.

All the actual marking (scanning objects, following pointers) happens **concurrently** — the
program continues running while dedicated GC goroutines scan the heap.

```
    GC Cycle Timeline

    ──── Mutator (application) ────────────────────────────────────────────
                    │                                              │
    ────────────────┼──── Concurrent Mark ─────────────────────────┼───────
                    │                                              │
    ──STW──┐        │                                     ┌──STW──┐
    Mark   │        │                                     │Mark   │
    Setup  │        │         GC Workers Scanning         │Term   │
    ~30μs  │        │         (concurrent with app)       │~30μs  │
    ───────┘        │                                     └───────┘
                    │                                              │
    ────────────────┼──── Sweep (concurrent) ──────────────────────┼───────
                    │  (freeing white objects, lazy,                │
                    │   happens on allocation)                     │
```

### 2.5.5 Hands-On: Tuning GC with GOGC and GOMEMLIMIT

```go
// cmd/gctune/main.go
//
// gctune demonstrates the effects of GOGC and GOMEMLIMIT on GC behavior.
// Run it with different settings to see how they affect GC frequency,
// pause times, and memory usage:
//
//	GOGC=100 go run ./cmd/gctune              # default
//	GOGC=50 go run ./cmd/gctune               # more frequent GC
//	GOGC=200 go run ./cmd/gctune              # less frequent GC
//	GOGC=off GOMEMLIMIT=256MiB go run ./cmd/gctune  # memory-limit-only mode
//
// This pattern (GOGC=off + GOMEMLIMIT) is recommended for containerized
// services where you know the exact memory budget.
package main

import (
	"fmt"
	"os"
	"runtime"
	"runtime/debug"
	"time"
)

// allocateWork simulates a workload that allocates and discards objects.
// It creates a mix of short-lived and long-lived objects to stress the GC.
//
// The function tracks GC events and reports statistics.
func allocateWork() {
	// Long-lived objects — these survive across GC cycles and form
	// the "live heap" that determines GC pacing.
	longLived := make([]*[1024]byte, 0, 50000)

	var totalAllocated uint64
	startTime := time.Now()

	for round := 0; round < 20; round++ {
		// Allocate 10,000 short-lived objects (1 KB each = 10 MB per round).
		// These will be collected by the next GC cycle.
		for i := 0; i < 10000; i++ {
			p := new([1024]byte)
			p[0] = byte(i) // Prevent optimization
			_ = p          // Short-lived: no reference kept
			totalAllocated += 1024
		}

		// Allocate 2,500 long-lived objects per round.
		// These accumulate and increase the live heap.
		for i := 0; i < 2500; i++ {
			p := new([1024]byte)
			p[0] = byte(i)
			longLived = append(longLived, p)
			totalAllocated += 1024
		}

		// Print GC stats every 5 rounds.
		if (round+1)%5 == 0 {
			var m runtime.MemStats
			runtime.ReadMemStats(&m)
			fmt.Printf("Round %2d: HeapAlloc=%6.1f MB, NumGC=%d, "+
				"PauseTotalMs=%.2f, LiveObjects=%d\n",
				round+1,
				float64(m.HeapAlloc)/1e6,
				m.NumGC,
				float64(m.PauseTotalNs)/1e6,
				m.Mallocs-m.Frees,
			)
		}
	}

	elapsed := time.Since(startTime)

	// Final stats.
	var m runtime.MemStats
	runtime.ReadMemStats(&m)

	fmt.Println("\n=== Final Statistics ===")
	fmt.Printf("Elapsed:        %v\n", elapsed)
	fmt.Printf("Total allocated: %.1f MB\n", float64(totalAllocated)/1e6)
	fmt.Printf("Live heap:      %.1f MB\n", float64(m.HeapAlloc)/1e6)
	fmt.Printf("GC cycles:      %d\n", m.NumGC)
	fmt.Printf("Total GC pause: %.2f ms\n", float64(m.PauseTotalNs)/1e6)
	fmt.Printf("GC CPU%%:        %.2f%%\n", m.GCCPUFraction*100)
	fmt.Printf("Long-lived objs: %d\n", len(longLived))

	_ = longLived // Keep alive until here
}

func main() {
	// Print current GC settings.
	fmt.Printf("GOGC:       %s\n", os.Getenv("GOGC"))
	fmt.Printf("GOMEMLIMIT: %s\n", os.Getenv("GOMEMLIMIT"))
	fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))

	// Read the actual GOGC value from the runtime.
	gcPercent := debug.SetGCPercent(-1) // Read without changing
	debug.SetGCPercent(gcPercent)       // Restore
	fmt.Printf("Effective GOGC: %d\n\n", gcPercent)

	allocateWork()
}
```

### 2.5.6 Hands-On: runtime/trace — Visualizing GC Behavior

```go
// cmd/gctrace/main.go
//
// gctrace generates a runtime trace file that can be visualized with
// `go tool trace` to see GC behavior graphically. The trace shows:
//   - When GC cycles start and stop
//   - STW pause durations
//   - GC worker goroutine activity
//   - Heap size over time
//   - Goroutine scheduling events
//
// Usage:
//
//	go run ./cmd/gctrace
//	go tool trace trace.out
//	# Opens a browser with an interactive timeline visualization
//
// You can also use GODEBUG=gctrace=1 for text-based GC logging:
//
//	GODEBUG=gctrace=1 go run ./cmd/gctrace
package main

import (
	"fmt"
	"os"
	"runtime"
	"runtime/trace"
)

func main() {
	// Create the trace output file.
	f, err := os.Create("trace.out")
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create trace file: %v\n", err)
		os.Exit(1)
	}
	defer f.Close()

	// Start tracing. All goroutine events, GC events, syscalls, and
	// scheduling decisions will be recorded to the file.
	if err := trace.Start(f); err != nil {
		fmt.Fprintf(os.Stderr, "Failed to start trace: %v\n", err)
		os.Exit(1)
	}
	defer trace.Stop()

	// Generate some GC activity.
	fmt.Println("Generating allocations to trigger GC cycles...")

	// Create a workload that generates garbage.
	var keep []*[]byte
	for i := 0; i < 50; i++ {
		// Short-lived allocations (garbage).
		for j := 0; j < 10000; j++ {
			b := make([]byte, 1024)
			_ = b
		}
		// Long-lived allocation.
		b := make([]byte, 1024*1024)
		keep = append(keep, &b)
	}

	// Force a final GC to see the sweep phase.
	runtime.GC()

	fmt.Printf("Trace written to trace.out\n")
	fmt.Printf("View with: go tool trace trace.out\n")
	fmt.Printf("Kept %d long-lived allocations\n", len(keep))
}
```

**GODEBUG=gctrace=1 output explained:**

```
gc 1 @0.012s 2%: 0.011+1.2+0.004 ms clock, 0.089+0.15/1.1/0+0.036 ms cpu, 4->4->2 MB, 4 MB goal, 0 MB stacks, 0 MB globals, 8 P

│   │          │                                              │                       │
│   │          │    wall-clock times:                         │   heap sizes:          │
│   │          │    STW mark setup + concurrent + STW term    │   before→after→live    │
│   │          │                                              │                       │
│   │          └─ CPU% spent in GC                            └─ heap goal for next GC│
│   └─ time since program start                                                       │
└─ GC cycle number                                                        GOMAXPROCS ─┘
```

---

## 2.6 Huge Pages

### 2.6.1 Regular Pages vs Huge Pages

Standard x86_64 pages are 4 KB. For a process using 1 GB of memory, that is 262,144 page
table entries. The TLB can only hold ~1,536 entries, so TLB misses are frequent.

**Huge pages** reduce this overhead:

| Page Size | Pages for 1 GB | TLB Entries Needed | TLB Coverage with 1536 entries |
|-----------|---------------|-------------------|-------------------------------|
| 4 KB | 262,144 | 262,144 | 6 MB (0.6%) |
| 2 MB | 512 | 512 | 3 GB (300%) |
| 1 GB | 1 | 1 | 1.5 TB |

With 2 MB huge pages, 1,536 TLB entries cover 3 GB — more than enough for most workloads.

### 2.6.2 Transparent Huge Pages (THP) vs Explicit hugetlbfs

**Transparent Huge Pages (THP):**
- Enabled by default on most Linux distributions.
- The kernel automatically promotes 4 KB pages to 2 MB pages when possible.
- No application changes required.
- Can cause latency spikes due to background page compaction.
- Controlled via `/sys/kernel/mm/transparent_hugepage/enabled`.

**Explicit hugetlbfs:**
- Application explicitly requests huge pages via `MAP_HUGETLB` or `hugetlbfs`.
- Pages are pre-reserved at boot time or via `sysctl`.
- More predictable performance — no compaction delays.
- Requires explicit configuration.

### 2.6.3 When to Use and When to Avoid

**Use huge pages when:**
- Your application has a large working set (> 100 MB) with random access patterns.
- TLB misses are a bottleneck (check with `perf stat -e dTLB-load-misses`).
- You need predictable, low-latency memory access (databases, caches).

**Avoid huge pages when:**
- Your application has a small working set.
- Memory is fragmented and huge pages cannot be formed.
- You are running in containers with strict memory limits (THP compaction can cause
  unexpected memory spikes).
- Your workload is fork-heavy (COW on huge pages copies 2 MB instead of 4 KB).

### 2.6.4 Hands-On: Allocating Huge Pages from Go

```go
// cmd/hugepages/main.go
//
// hugepages demonstrates how to allocate memory backed by huge pages (2 MB)
// from Go using the mmap system call with MAP_HUGETLB flag.
//
// Prerequisites:
//
//	# Check available huge pages
//	cat /proc/meminfo | grep HugePages
//
//	# Allocate 100 huge pages (200 MB) if not already available
//	echo 100 | sudo tee /proc/sys/vm/nr_hugepages
//
//	# Verify
//	cat /proc/meminfo | grep HugePages
//	# HugePages_Total:     100
//	# HugePages_Free:      100
//
// Usage:
//
//	go build -o hugepages ./cmd/hugepages
//	./hugepages
//
// If you get "cannot allocate memory", ensure huge pages are configured
// as shown above.
package main

import (
	"fmt"
	"os"
	"runtime"
	"syscall"
	"time"
	"unsafe"
)

const (
	// hugePageSize is the default huge page size on x86_64 Linux (2 MB).
	hugePageSize = 2 * 1024 * 1024

	// MAP_HUGETLB is the mmap flag requesting huge pages.
	// It may not be defined in older versions of the syscall package.
	mapHugeTLB = 0x40000

	// Total memory to allocate: 64 MB = 32 huge pages.
	allocSize = 64 * 1024 * 1024
)

// allocateHugePages attempts to allocate the given number of bytes using
// huge pages via mmap with MAP_HUGETLB. Returns the mapped byte slice
// or an error.
//
// The allocation uses MAP_ANONYMOUS (no file backing) and MAP_PRIVATE
// (private to this process). The MAP_HUGETLB flag tells the kernel to
// use 2 MB huge pages instead of standard 4 KB pages.
//
// Each 2 MB huge page requires only ONE page table entry and ONE TLB
// entry, compared to 512 entries for the same memory with 4 KB pages.
func allocateHugePages(size int) ([]byte, error) {
	data, err := syscall.Mmap(
		-1, 0, size,
		syscall.PROT_READ|syscall.PROT_WRITE,
		syscall.MAP_PRIVATE|syscall.MAP_ANONYMOUS|mapHugeTLB,
	)
	if err != nil {
		return nil, fmt.Errorf("mmap with MAP_HUGETLB failed: %w "+
			"(ensure huge pages are configured: echo 100 > /proc/sys/vm/nr_hugepages)", err)
	}
	return data, nil
}

// allocateRegularPages allocates the given number of bytes using standard
// 4 KB pages. Used for comparison with huge pages.
func allocateRegularPages(size int) ([]byte, error) {
	data, err := syscall.Mmap(
		-1, 0, size,
		syscall.PROT_READ|syscall.PROT_WRITE,
		syscall.MAP_PRIVATE|syscall.MAP_ANONYMOUS,
	)
	if err != nil {
		return nil, fmt.Errorf("mmap failed: %w", err)
	}
	return data, nil
}

// benchmarkAccess performs random reads across the mapped region to
// stress the TLB. With huge pages, TLB coverage is much better,
// leading to fewer TLB misses and faster access times.
//
// The benchmark accesses every page in a pseudo-random order by
// stepping through the region with a stride that is relatively prime
// to the number of pages, ensuring all pages are visited.
func benchmarkAccess(data []byte, label string) {
	runtime.LockOSThread()
	n := len(data)
	stride := 4099 * 4096 // ~16 MB stride (prime × page size)
	if stride >= n {
		stride = 4096
	}

	start := time.Now()
	var sum byte
	offset := 0
	for i := 0; i < 10_000_000; i++ {
		sum += data[offset]
		offset = (offset + stride) % n
	}
	elapsed := time.Since(start)

	fmt.Printf("  %-20s: %v for 10M accesses (%.0f ns/access), sum=%d\n",
		label, elapsed, float64(elapsed.Nanoseconds())/10_000_000, sum)
}

func main() {
	fmt.Printf("Attempting to allocate %d MB with huge pages...\n", allocSize/1024/1024)

	// Try huge pages first.
	hugeData, err := allocateHugePages(allocSize)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Huge page allocation failed: %v\n", err)
		fmt.Fprintf(os.Stderr, "Continuing with regular pages only.\n\n")
	} else {
		defer syscall.Munmap(hugeData)
		fmt.Printf("Huge page allocation succeeded at %p\n",
			unsafe.Pointer(&hugeData[0]))

		// Touch all pages to fault them in.
		for i := 0; i < len(hugeData); i += hugePageSize {
			hugeData[i] = 0xFF
		}
	}

	// Allocate with regular pages for comparison.
	regularData, err := allocateRegularPages(allocSize)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Regular allocation failed: %v\n", err)
		os.Exit(1)
	}
	defer syscall.Munmap(regularData)

	// Touch all regular pages.
	for i := 0; i < len(regularData); i += 4096 {
		regularData[i] = 0xFF
	}

	fmt.Println("\n=== Access Benchmark ===")
	benchmarkAccess(regularData, "Regular (4 KB)")
	if hugeData != nil {
		benchmarkAccess(hugeData, "Huge (2 MB)")
	}
}
```

---

## 2.7 Memory-Mapped I/O for Systems Programming

### 2.7.1 Using mmap for Zero-Copy I/O

Traditional file I/O with `read()` involves two copies:

```
    Traditional read() Path:
    Disk → Kernel Page Cache → User Buffer (2 copies, 1 syscall)

    ┌──────┐    ┌──────────────┐    ┌──────────────┐
    │ Disk │───▶│ Page Cache   │───▶│ User Buffer  │
    │      │    │ (kernel)     │    │ (userspace)  │
    └──────┘    └──────────────┘    └──────────────┘
       DMA         memcpy from         your []byte
                   kernel to user

    mmap() Path:
    Disk → Page Cache → User accesses directly (1 copy, 0 syscalls per access)

    ┌──────┐    ┌──────────────┐
    │ Disk │───▶│ Page Cache   │◀── User reads/writes directly
    │      │    │ (shared)     │    via virtual address mapping
    └──────┘    └──────────────┘
       DMA         no memcpy!
```

With `mmap`, the user-space virtual address maps directly to the page cache. There is no
copy from kernel to user — the same physical frame is shared. This is true zero-copy I/O.

### 2.7.2 Shared Memory Between Processes via mmap

Two processes can share memory by mapping the same file (or anonymous shared mapping)
with `MAP_SHARED`. This is the fastest form of inter-process communication because there
is no copying, no syscalls, and no kernel involvement for reads and writes — the processes
read and write directly to shared physical frames.

### 2.7.3 Hands-On: IPC via Shared Memory Using mmap in Go

```go
// cmd/shmwriter/main.go
//
// shmwriter creates a shared memory region via mmap and writes messages
// into it. A companion program (shmreader) can read the messages.
//
// The shared memory is backed by a file in /dev/shm (a tmpfs filesystem),
// which means it resides entirely in RAM — no disk I/O. Both processes
// map the same file, so writes by the writer are instantly visible to
// the reader without any system calls.
//
// This is the fastest possible IPC mechanism on Linux — there is no
// kernel involvement in the data transfer. The only synchronization
// overhead is cache coherency between CPU cores (typically ~50-100 ns).
//
// Memory layout of the shared region:
//
//	Bytes 0-7:    uint64 sequence number (writer increments atomically)
//	Bytes 8-263:  message payload (256 bytes, null-terminated)
//	Bytes 264+:   unused
//
// Usage:
//
//	go build -o shmwriter ./cmd/shmwriter
//	go build -o shmreader ./cmd/shmreader
//	./shmwriter &
//	./shmreader
package main

import (
	"encoding/binary"
	"fmt"
	"os"
	"sync/atomic"
	"syscall"
	"time"
	"unsafe"
)

const (
	// shmPath is the shared memory file path. /dev/shm is a tmpfs
	// filesystem that lives entirely in RAM.
	shmPath = "/dev/shm/go-ipc-demo"
	// shmSize is the size of the shared memory region.
	shmSize = 4096
	// seqOffset is the byte offset of the sequence number.
	seqOffset = 0
	// msgOffset is the byte offset of the message payload.
	msgOffset = 8
	// msgMaxLen is the maximum message length.
	msgMaxLen = 256
)

func main() {
	// Create (or open) the shared memory file.
	f, err := os.OpenFile(shmPath, os.O_RDWR|os.O_CREATE|os.O_TRUNC, 0666)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create shm file: %v\n", err)
		os.Exit(1)
	}
	defer f.Close()

	// Set the file size to our shared region size.
	if err := f.Truncate(shmSize); err != nil {
		fmt.Fprintf(os.Stderr, "Failed to truncate: %v\n", err)
		os.Exit(1)
	}

	// Map the file into memory with MAP_SHARED so that writes are visible
	// to other processes mapping the same file.
	data, err := syscall.Mmap(
		int(f.Fd()), 0, shmSize,
		syscall.PROT_READ|syscall.PROT_WRITE,
		syscall.MAP_SHARED,
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "mmap failed: %v\n", err)
		os.Exit(1)
	}
	defer syscall.Munmap(data)

	// Get a pointer to the sequence number for atomic operations.
	seqPtr := (*uint64)(unsafe.Pointer(&data[seqOffset]))

	fmt.Println("Writer: starting to write messages to shared memory...")
	fmt.Printf("Writer: shared memory at %s (%d bytes)\n", shmPath, shmSize)

	for i := 1; i <= 100; i++ {
		// Write the message payload.
		msg := fmt.Sprintf("Message #%d at %s", i, time.Now().Format(time.RFC3339Nano))
		if len(msg) > msgMaxLen {
			msg = msg[:msgMaxLen]
		}

		// Copy message into shared memory.
		copy(data[msgOffset:msgOffset+msgMaxLen], make([]byte, msgMaxLen)) // Clear old message
		copy(data[msgOffset:], []byte(msg))

		// Update the sequence number atomically. The reader watches this
		// to detect new messages. We use StoreUint64 for atomic visibility
		// across processes sharing the same physical memory.
		atomic.StoreUint64(seqPtr, uint64(i))

		// Ensure the write is flushed from CPU write buffers to the
		// shared cache line.
		// (On x86, stores are already ordered, but atomic.Store provides
		// the memory ordering guarantee we need.)

		if i%10 == 0 {
			fmt.Printf("Writer: wrote message #%d\n", i)
		}
		time.Sleep(50 * time.Millisecond)
	}

	// Write final sequence number with a zero-length message to signal done.
	binary.LittleEndian.PutUint64(data[seqOffset:], 0xFFFFFFFFFFFFFFFF)

	fmt.Println("Writer: done. Cleaning up...")
	// Note: we don't remove the shm file here so the reader can still access it.
}
```

```go
// cmd/shmreader/main.go
//
// shmreader opens the shared memory region created by shmwriter and
// reads messages from it. It uses atomic operations on the sequence
// number to detect new messages without any system calls.
//
// This demonstrates true zero-copy, zero-syscall IPC: the reader
// accesses the writer's data directly through shared physical memory,
// mediated only by CPU cache coherency.
//
// Usage:
//
//	# In terminal 1:
//	./shmwriter
//
//	# In terminal 2:
//	./shmreader
package main

import (
	"fmt"
	"os"
	"runtime"
	"strings"
	"sync/atomic"
	"syscall"
	"time"
	"unsafe"
)

const (
	shmPath   = "/dev/shm/go-ipc-demo"
	shmSize   = 4096
	seqOffset = 0
	msgOffset = 8
	msgMaxLen = 256
)

func main() {
	// Open the shared memory file created by the writer.
	f, err := os.OpenFile(shmPath, os.O_RDONLY, 0)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to open shm file: %v\n", err)
		fmt.Fprintf(os.Stderr, "Is the writer running? (./shmwriter)\n")
		os.Exit(1)
	}
	defer f.Close()

	// Map the file read-only with MAP_SHARED.
	data, err := syscall.Mmap(
		int(f.Fd()), 0, shmSize,
		syscall.PROT_READ,
		syscall.MAP_SHARED,
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "mmap failed: %v\n", err)
		os.Exit(1)
	}
	defer syscall.Munmap(data)

	// Get a pointer to the sequence number.
	seqPtr := (*uint64)(unsafe.Pointer(&data[seqOffset]))

	fmt.Println("Reader: watching shared memory for messages...")

	var lastSeq uint64
	deadline := time.Now().Add(30 * time.Second)

	for time.Now().Before(deadline) {
		// Atomically load the sequence number. If it has changed since
		// our last read, there is a new message.
		seq := atomic.LoadUint64(seqPtr)

		if seq == 0xFFFFFFFFFFFFFFFF {
			fmt.Println("Reader: writer signaled done.")
			break
		}

		if seq != lastSeq && seq != 0 {
			// Extract the null-terminated message string.
			msgBytes := data[msgOffset : msgOffset+msgMaxLen]
			msg := string(msgBytes[:strings.IndexByte(string(msgBytes), 0)])

			fmt.Printf("Reader: [seq=%d] %s\n", seq, msg)
			lastSeq = seq
		}

		// Spin briefly. In production, you would use a futex or eventfd
		// for efficient waiting, but this demonstrates the shared memory
		// data path.
		runtime.Gosched()
		time.Sleep(10 * time.Millisecond)
	}

	fmt.Println("Reader: exiting.")
}
```

### 2.7.4 Hands-On: Building a Simple Memory-Mapped Key-Value Store in Go

```go
// cmd/mmapkv/main.go
//
// mmapkv implements a simple persistent key-value store backed by a
// memory-mapped file. It demonstrates how databases use mmap to provide
// fast, persistent storage.
//
// Design:
//   - Fixed-size header (magic number, version, entry count)
//   - Fixed-size entries (key: 64 bytes, value: 256 bytes, used flag)
//   - Linear probing hash table for lookups
//   - Changes are immediately visible in the file (via MAP_SHARED)
//   - msync ensures durability to disk
//
// This is a teaching implementation — a production KV store would need:
//   - Variable-length keys and values
//   - Better hash collision handling
//   - Concurrency control (locks or lock-free)
//   - Write-ahead logging for crash safety
//   - Compaction / garbage collection
//
// File layout:
//
//	┌──────────────────────────────────────────┐
//	│ Header (64 bytes)                        │
//	│   magic:   uint32 (0x4D4D4B56 = "MMKV") │
//	│   version: uint32                        │
//	│   count:   uint64 (number of entries)    │
//	│   cap:     uint64 (max entries)          │
//	│   padding: 40 bytes                      │
//	├──────────────────────────────────────────┤
//	│ Entry 0 (324 bytes)                      │
//	│   used:  uint32 (0=empty, 1=used)        │
//	│   key:   [64]byte                        │
//	│   value: [256]byte                       │
//	├──────────────────────────────────────────┤
//	│ Entry 1 (324 bytes)                      │
//	│   ...                                    │
//	├──────────────────────────────────────────┤
//	│ ...                                      │
//	└──────────────────────────────────────────┘
//
// Usage:
//
//	go run ./cmd/mmapkv
package main

import (
	"encoding/binary"
	"fmt"
	"hash/fnv"
	"os"
	"syscall"
	"unsafe"
)

const (
	// headerSize is the size of the file header in bytes.
	headerSize = 64
	// keySize is the maximum key length in bytes.
	keySize = 64
	// valueSize is the maximum value length in bytes.
	valueSize = 256
	// entrySize is the size of one key-value entry in bytes.
	// 4 (used flag) + 64 (key) + 256 (value) = 324 bytes
	entrySize = 4 + keySize + valueSize
	// magic is the file format magic number ("MMKV" in hex).
	magic = 0x4D4D4B56
	// version is the file format version.
	version = 1
)

// MmapKV is a memory-mapped key-value store. All data is stored in a
// memory-mapped file, so reads and writes go directly to the page cache
// without any serialization/deserialization overhead.
type MmapKV struct {
	// data is the raw memory-mapped byte slice.
	data []byte
	// cap is the maximum number of entries.
	cap uint64
	// path is the backing file path.
	path string
}

// OpenKV opens (or creates) a memory-mapped key-value store at the given
// path with the specified capacity (maximum number of entries).
//
// If the file exists and has a valid header, it is opened as-is.
// If the file does not exist, a new store is created.
//
// The entire store is mapped into memory with MAP_SHARED, so all writes
// are immediately reflected in the file (though not necessarily flushed
// to disk until msync or munmap).
func OpenKV(path string, capacity uint64) (*MmapKV, error) {
	totalSize := headerSize + int(capacity)*entrySize

	// Open or create the file.
	f, err := os.OpenFile(path, os.O_RDWR|os.O_CREATE, 0666)
	if err != nil {
		return nil, fmt.Errorf("open %s: %w", path, err)
	}
	defer f.Close()

	// Ensure the file is the right size.
	info, err := f.Stat()
	if err != nil {
		return nil, fmt.Errorf("stat: %w", err)
	}
	if info.Size() < int64(totalSize) {
		if err := f.Truncate(int64(totalSize)); err != nil {
			return nil, fmt.Errorf("truncate: %w", err)
		}
	}

	// Map the file into memory.
	data, err := syscall.Mmap(
		int(f.Fd()), 0, totalSize,
		syscall.PROT_READ|syscall.PROT_WRITE,
		syscall.MAP_SHARED,
	)
	if err != nil {
		return nil, fmt.Errorf("mmap: %w", err)
	}

	kv := &MmapKV{data: data, cap: capacity, path: path}

	// Check or initialize the header.
	headerMagic := binary.LittleEndian.Uint32(data[0:4])
	if headerMagic == 0 {
		// New file — initialize header.
		binary.LittleEndian.PutUint32(data[0:4], magic)
		binary.LittleEndian.PutUint32(data[4:8], version)
		binary.LittleEndian.PutUint64(data[8:16], 0) // count = 0
		binary.LittleEndian.PutUint64(data[16:24], capacity)
	} else if headerMagic != magic {
		syscall.Munmap(data)
		return nil, fmt.Errorf("invalid magic number: %x (expected %x)", headerMagic, magic)
	}

	return kv, nil
}

// entryOffset returns the byte offset of the entry at the given index.
func (kv *MmapKV) entryOffset(index uint64) int {
	return headerSize + int(index)*entrySize
}

// hashKey returns a hash of the key used for linear probing.
func hashKey(key string) uint64 {
	h := fnv.New64a()
	h.Write([]byte(key))
	return h.Sum64()
}

// Put stores a key-value pair in the store. Uses linear probing to
// handle hash collisions.
//
// Returns an error if the store is full or the key/value exceeds the
// maximum size.
func (kv *MmapKV) Put(key, value string) error {
	if len(key) > keySize {
		return fmt.Errorf("key too long: %d > %d", len(key), keySize)
	}
	if len(value) > valueSize {
		return fmt.Errorf("value too long: %d > %d", len(value), valueSize)
	}

	startIdx := hashKey(key) % kv.cap

	// Linear probe: search for an empty slot or existing key.
	for i := uint64(0); i < kv.cap; i++ {
		idx := (startIdx + i) % kv.cap
		off := kv.entryOffset(idx)

		used := binary.LittleEndian.Uint32(kv.data[off : off+4])

		if used == 0 {
			// Empty slot — insert here.
			binary.LittleEndian.PutUint32(kv.data[off:off+4], 1) // mark used

			// Write key (zero-padded).
			keySlice := kv.data[off+4 : off+4+keySize]
			copy(keySlice, make([]byte, keySize)) // clear
			copy(keySlice, []byte(key))

			// Write value (zero-padded).
			valSlice := kv.data[off+4+keySize : off+4+keySize+valueSize]
			copy(valSlice, make([]byte, valueSize)) // clear
			copy(valSlice, []byte(value))

			// Increment count in header.
			count := binary.LittleEndian.Uint64(kv.data[8:16])
			binary.LittleEndian.PutUint64(kv.data[8:16], count+1)

			return nil
		}

		// Check if this slot has the same key (update case).
		existingKey := string(kv.data[off+4 : off+4+keySize])
		existingKey = existingKey[:clen(existingKey)]
		if existingKey == key {
			// Update value.
			valSlice := kv.data[off+4+keySize : off+4+keySize+valueSize]
			copy(valSlice, make([]byte, valueSize))
			copy(valSlice, []byte(value))
			return nil
		}
	}

	return fmt.Errorf("store is full (capacity: %d)", kv.cap)
}

// Get retrieves the value associated with the given key.
// Returns the value and true if found, or ("", false) if not found.
func (kv *MmapKV) Get(key string) (string, bool) {
	startIdx := hashKey(key) % kv.cap

	for i := uint64(0); i < kv.cap; i++ {
		idx := (startIdx + i) % kv.cap
		off := kv.entryOffset(idx)

		used := binary.LittleEndian.Uint32(kv.data[off : off+4])
		if used == 0 {
			return "", false // Empty slot → key not found
		}

		existingKey := string(kv.data[off+4 : off+4+keySize])
		existingKey = existingKey[:clen(existingKey)]
		if existingKey == key {
			val := string(kv.data[off+4+keySize : off+4+keySize+valueSize])
			val = val[:clen(val)]
			return val, true
		}
	}

	return "", false
}

// Sync flushes the memory-mapped region to disk using msync.
// This ensures durability — without this, data could be lost on crash.
func (kv *MmapKV) Sync() error {
	_, _, errno := syscall.Syscall(
		syscall.SYS_MSYNC,
		uintptr(unsafe.Pointer(&kv.data[0])),
		uintptr(len(kv.data)),
		uintptr(syscall.MS_SYNC),
	)
	if errno != 0 {
		return fmt.Errorf("msync: %v", errno)
	}
	return nil
}

// Close unmaps the memory and releases resources.
func (kv *MmapKV) Close() error {
	return syscall.Munmap(kv.data)
}

// Count returns the number of entries in the store.
func (kv *MmapKV) Count() uint64 {
	return binary.LittleEndian.Uint64(kv.data[8:16])
}

// clen returns the length of a null-terminated string.
func clen(s string) int {
	for i := 0; i < len(s); i++ {
		if s[i] == 0 {
			return i
		}
	}
	return len(s)
}

func main() {
	kvPath := "test.mmkv"

	// Open a store with capacity for 1024 entries.
	kv, err := OpenKV(kvPath, 1024)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to open KV store: %v\n", err)
		os.Exit(1)
	}
	defer func() {
		kv.Close()
		os.Remove(kvPath)
	}()

	fmt.Printf("Opened mmap KV store: %s (capacity: 1024)\n\n", kvPath)

	// Insert some key-value pairs.
	entries := map[string]string{
		"name":     "Go Systems Programming",
		"author":   "Chapter 2 Demo",
		"topic":    "Memory Management",
		"os":       "Linux",
		"arch":     "x86_64",
		"pagesize": "4096",
	}

	for k, v := range entries {
		if err := kv.Put(k, v); err != nil {
			fmt.Fprintf(os.Stderr, "Put(%q) failed: %v\n", k, err)
		}
	}
	fmt.Printf("Inserted %d entries (count in store: %d)\n\n", len(entries), kv.Count())

	// Read them back.
	fmt.Println("Reading entries:")
	for k := range entries {
		if v, ok := kv.Get(k); ok {
			fmt.Printf("  %s = %s\n", k, v)
		} else {
			fmt.Printf("  %s = (not found!)\n", k)
		}
	}

	// Test a missing key.
	if _, ok := kv.Get("nonexistent"); !ok {
		fmt.Println("\n  'nonexistent' correctly not found")
	}

	// Update a value.
	kv.Put("name", "Updated: Go Systems Programming Book")
	v, _ := kv.Get("name")
	fmt.Printf("\n  After update: name = %s\n", v)

	// Sync to disk.
	if err := kv.Sync(); err != nil {
		fmt.Fprintf(os.Stderr, "Sync failed: %v\n", err)
	}
	fmt.Println("\nData synced to disk successfully.")
}
```

---

## 2.8 NUMA (Non-Uniform Memory Access)

### 2.8.1 NUMA Topology

On modern multi-socket servers, memory is not uniformly accessible. Each CPU socket has
its own local memory, and accessing remote memory (attached to another socket) is slower:

```
    NUMA Topology (2-socket server)

    ┌─────────────────────────────┐     ┌─────────────────────────────┐
    │         Node 0              │     │         Node 1              │
    │                             │     │                             │
    │  ┌─────┐  ┌─────┐          │     │  ┌─────┐  ┌─────┐          │
    │  │CPU 0│  │CPU 1│          │     │  │CPU 4│  │CPU 5│          │
    │  └──┬──┘  └──┬──┘          │     │  └──┬──┘  └──┬──┘          │
    │     │        │              │     │     │        │              │
    │  ┌─────┐  ┌─────┐          │     │  ┌─────┐  ┌─────┐          │
    │  │CPU 2│  │CPU 3│          │     │  │CPU 6│  │CPU 7│          │
    │  └──┬──┘  └──┬──┘          │     │  └──┬──┘  └──┬──┘          │
    │     │        │              │     │     │        │              │
    │  ┌──┴────────┴──┐          │     │  ┌──┴────────┴──┐          │
    │  │  Memory      │          │     │  │  Memory      │          │
    │  │  Controller  │          │     │  │  Controller  │          │
    │  └──────┬───────┘          │     │  └──────┬───────┘          │
    │         │                  │     │         │                  │
    │  ┌──────┴───────┐          │     │  ┌──────┴───────┐          │
    │  │ Local DRAM   │          │     │  │ Local DRAM   │          │
    │  │ (64 GB)      │          │     │  │ (64 GB)      │          │
    │  │ ~100ns       │          │     │  │ ~100ns       │          │
    │  └──────────────┘          │     │  └──────────────┘          │
    │                             │     │                             │
    └──────────────┬──────────────┘     └──────────────┬──────────────┘
                   │                                   │
                   └─────── QPI / UPI Link ────────────┘
                         ~150-300ns remote access
```

**Local access:** ~100 ns
**Remote access:** ~150-300 ns (1.5x-3x slower)

### 2.8.2 NUMA Tools and Syscalls

- **`numactl`**: Command-line tool to control NUMA policy for a process.
  - `numactl --membind=0 ./myapp` — allocate all memory on node 0.
  - `numactl --interleave=all ./myapp` — spread memory across all nodes.
  - `numactl --cpunodebind=0 ./myapp` — run only on node 0's CPUs.

- **`set_mempolicy()`**: Set the NUMA memory policy for the calling process.
- **`mbind()`**: Set the NUMA policy for a specific memory region.
- **`get_mempolicy()`**: Query the current NUMA policy.

### 2.8.3 Go and NUMA Awareness

Go's runtime is **not NUMA-aware** as of Go 1.22. The scheduler does not consider NUMA
topology when scheduling goroutines, and the allocator does not try to allocate memory
local to the CPU running the goroutine.

For NUMA-sensitive workloads, you can:
1. Use `numactl` to bind your Go process to a single NUMA node.
2. Use `runtime.LockOSThread()` + OS-level CPU affinity to pin goroutines.
3. Use `mbind()` via syscall to bind specific allocations to a node.

### 2.8.4 Hands-On: Checking NUMA Topology from Go

```go
// cmd/numainfo/main.go
//
// numainfo reads NUMA topology information from /sys/devices/system/node/
// and displays it. This helps you understand the memory architecture of
// the machine your Go program is running on.
//
// On a NUMA system, this will show:
//   - Number of NUMA nodes
//   - Which CPUs belong to each node
//   - Memory available on each node
//   - Inter-node distances (latency ratios)
//
// On a single-socket system (most laptops/desktops), there is typically
// only one NUMA node (node0) containing all CPUs and memory.
//
// Usage:
//
//	go run ./cmd/numainfo
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"strings"
)

// readFile is a helper to read a file and return its trimmed contents.
func readFile(path string) (string, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return "", err
	}
	return strings.TrimSpace(string(data)), nil
}

func main() {
	nodeBase := "/sys/devices/system/node"

	// Find all NUMA nodes.
	entries, err := os.ReadDir(nodeBase)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Cannot read %s: %v\n", nodeBase, err)
		fmt.Fprintf(os.Stderr, "This system may not expose NUMA info via sysfs.\n")
		os.Exit(1)
	}

	var nodes []string
	for _, e := range entries {
		if strings.HasPrefix(e.Name(), "node") && e.IsDir() {
			nodes = append(nodes, e.Name())
		}
	}

	fmt.Printf("=== NUMA Topology ===\n")
	fmt.Printf("Number of NUMA nodes: %d\n\n", len(nodes))

	for _, node := range nodes {
		nodePath := filepath.Join(nodeBase, node)
		fmt.Printf("--- %s ---\n", node)

		// Read CPU list for this node.
		// This tells us which logical CPUs are directly connected to
		// this node's memory controller.
		cpuList, err := readFile(filepath.Join(nodePath, "cpulist"))
		if err == nil {
			fmt.Printf("  CPUs:     %s\n", cpuList)
		}

		// Read memory info for this node.
		memInfo, err := readFile(filepath.Join(nodePath, "meminfo"))
		if err == nil {
			// Parse out total and free memory lines.
			for _, line := range strings.Split(memInfo, "\n") {
				if strings.Contains(line, "MemTotal") ||
					strings.Contains(line, "MemFree") ||
					strings.Contains(line, "MemUsed") {
					fmt.Printf("  %s\n", strings.TrimSpace(line))
				}
			}
		}
		fmt.Println()
	}

	// Read the distance matrix.
	// The distance file shows relative latency between nodes.
	// A distance of 10 = local, 20 = one hop, 30 = two hops, etc.
	distPath := filepath.Join(nodeBase, nodes[0], "distance")
	if dist, err := readFile(distPath); err == nil {
		fmt.Println("NUMA Distance Matrix (relative latency):")
		fmt.Printf("         ")
		for _, n := range nodes {
			fmt.Printf("%-8s", n)
		}
		fmt.Println()
		for i, n := range nodes {
			d, _ := readFile(filepath.Join(nodeBase, n, "distance"))
			fmt.Printf("  %s: %s\n", nodes[i], d)
		}
		_ = dist
	}

	// Bonus: read the online/possible/present CPU info.
	fmt.Println()
	if possible, err := readFile("/sys/devices/system/cpu/possible"); err == nil {
		fmt.Printf("CPUs possible: %s\n", possible)
	}
	if online, err := readFile("/sys/devices/system/cpu/online"); err == nil {
		fmt.Printf("CPUs online:   %s\n", online)
	}
}
```

---

## 2.9 OOM Killer

### 2.9.1 How Linux Decides What to Kill

When the system runs out of memory (both RAM and swap are exhausted), the Linux kernel
invokes the **OOM (Out-Of-Memory) Killer**. It must choose a process to kill to free
memory, and it uses a heuristic scoring system:

**OOM Score Calculation:**
1. Start with the process's RSS (Resident Set Size) as a proportion of total memory.
2. Add or subtract the `oom_score_adj` value (-1000 to +1000).
3. Consider the process's age, CPU time, and number of children.

```
    OOM Score Range:

    -1000          0                                  +1000
      │            │                                    │
      ▼            ▼                                    ▼
    Never kill     Default                          Kill first
    (oom_score_adj  (oom_score_adj = 0)              (oom_score_adj = +1000)
     = -1000)

    The kernel picks the process with the HIGHEST oom_score.

    Special values:
      oom_score_adj = -1000  →  Process is exempt from OOM killer
      oom_score_adj = -999   →  Very unlikely to be killed
      oom_score_adj = 0      →  Default
      oom_score_adj = +1000  →  Always kill first
```

**What happens when OOM is triggered:**
1. Kernel tries to reclaim memory (page cache, slab caches, etc.).
2. If reclamation fails, OOM killer activates.
3. It iterates all processes, computes OOM scores.
4. Sends SIGKILL to the process with the highest score.
5. If that does not free enough memory, kills the next highest, and so on.

### 2.9.2 /proc/[pid]/oom_score and oom_score_adj

Every process has two OOM-related files in `/proc`:

- **`/proc/[pid]/oom_score`** (read-only): The current OOM score computed by the kernel.
  Higher = more likely to be killed. Range: 0 to ~1000.

- **`/proc/[pid]/oom_score_adj`** (read-write): An adjustment factor. Range: -1000 to +1000.
  Writing -1000 makes the process immune to OOM. Writing +1000 makes it the first target.

### 2.9.3 Hands-On: Setting OOM Score Adjustment from Go

```go
// cmd/oomadj/main.go
//
// oomadj demonstrates how to read and modify the OOM score adjustment
// for the current process. This is critical for Go services running in
// memory-constrained environments (containers, embedded systems).
//
// In a containerized deployment:
//   - Set oom_score_adj = -999 for your most critical service
//     (but NOT -1000, because that makes it immune and could cause
//     the entire cgroup to be OOM-killed instead)
//   - Set oom_score_adj = +500 for less important worker processes
//   - The init/PID 1 process usually has oom_score_adj = -1000
//
// Note: Lowering oom_score_adj below the current value requires
// CAP_SYS_RESOURCE capability (usually root).
//
// Usage:
//
//	go run ./cmd/oomadj
//	sudo go run ./cmd/oomadj   # for lowering oom_score_adj
package main

import (
	"fmt"
	"os"
	"strconv"
	"strings"
)

// getOOMScore reads the current OOM score for the given PID.
// The OOM score is a value computed by the kernel that represents how
// "eligible" a process is to be killed by the OOM killer.
//
// A higher score means the process is more likely to be killed.
// The score is based on the process's memory usage (RSS) relative to
// total system memory, adjusted by oom_score_adj.
func getOOMScore(pid int) (int, error) {
	path := fmt.Sprintf("/proc/%d/oom_score", pid)
	data, err := os.ReadFile(path)
	if err != nil {
		return 0, fmt.Errorf("read %s: %w", path, err)
	}
	score, err := strconv.Atoi(strings.TrimSpace(string(data)))
	if err != nil {
		return 0, fmt.Errorf("parse oom_score: %w", err)
	}
	return score, nil
}

// getOOMScoreAdj reads the current OOM score adjustment for the given PID.
// This is the user-controlled component of the OOM score.
// Range: -1000 (never kill) to +1000 (always kill first).
func getOOMScoreAdj(pid int) (int, error) {
	path := fmt.Sprintf("/proc/%d/oom_score_adj", pid)
	data, err := os.ReadFile(path)
	if err != nil {
		return 0, fmt.Errorf("read %s: %w", path, err)
	}
	adj, err := strconv.Atoi(strings.TrimSpace(string(data)))
	if err != nil {
		return 0, fmt.Errorf("parse oom_score_adj: %w", err)
	}
	return adj, nil
}

// setOOMScoreAdj sets the OOM score adjustment for the given PID.
//
// Increasing the value (making the process more killable) can be done
// by any process for itself. Decreasing the value requires
// CAP_SYS_RESOURCE (typically root).
//
// Common values:
//
//	-1000: Never kill (use sparingly — can cause cgroup-level OOM)
//	 -999: Almost never kill (good for critical services)
//	    0: Default
//	 +500: Prefer to kill (good for batch/worker processes)
//	+1000: Always kill first (good for sacrificial processes)
func setOOMScoreAdj(pid int, adj int) error {
	if adj < -1000 || adj > 1000 {
		return fmt.Errorf("oom_score_adj must be between -1000 and 1000, got %d", adj)
	}
	path := fmt.Sprintf("/proc/%d/oom_score_adj", pid)
	return os.WriteFile(path, []byte(strconv.Itoa(adj)), 0644)
}

func main() {
	pid := os.Getpid()
	fmt.Printf("PID: %d\n\n", pid)

	// Read current OOM score and adjustment.
	score, err := getOOMScore(pid)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error reading OOM score: %v\n", err)
		os.Exit(1)
	}
	adj, err := getOOMScoreAdj(pid)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error reading OOM score adj: %v\n", err)
		os.Exit(1)
	}

	fmt.Printf("Current OOM score:     %d\n", score)
	fmt.Printf("Current OOM score adj: %d\n", adj)

	// Allocate some memory to see the OOM score change.
	fmt.Println("\nAllocating 100 MB to observe OOM score change...")
	data := make([]byte, 100*1024*1024)
	// Touch all pages so they become resident (RSS increases).
	for i := 0; i < len(data); i += 4096 {
		data[i] = 0xFF
	}

	score2, _ := getOOMScore(pid)
	fmt.Printf("OOM score after 100 MB alloc: %d (was %d)\n", score2, score)

	// Try to increase our OOM score adj (always allowed).
	fmt.Println("\nSetting oom_score_adj to +500 (prefer to kill)...")
	err = setOOMScoreAdj(pid, 500)
	if err != nil {
		fmt.Fprintf(os.Stderr, "  Failed: %v\n", err)
	} else {
		score3, _ := getOOMScore(pid)
		adj3, _ := getOOMScoreAdj(pid)
		fmt.Printf("  New OOM score:     %d\n", score3)
		fmt.Printf("  New OOM score adj: %d\n", adj3)
	}

	// Try to decrease (needs root).
	fmt.Println("\nSetting oom_score_adj to -500 (protect from kill)...")
	err = setOOMScoreAdj(pid, -500)
	if err != nil {
		fmt.Fprintf(os.Stderr, "  Failed: %v (need root/CAP_SYS_RESOURCE)\n", err)
	} else {
		score4, _ := getOOMScore(pid)
		adj4, _ := getOOMScoreAdj(pid)
		fmt.Printf("  New OOM score:     %d\n", score4)
		fmt.Printf("  New OOM score adj: %d\n", adj4)
	}

	// Keep data alive to prevent GC from collecting it during measurement.
	_ = data
}
```

---

## 2.10 Putting It All Together — Memory Lifecycle

Let us trace the full lifecycle of a memory allocation in a Go program on Linux:

```
    Complete Memory Lifecycle: p := new(MyStruct)

    1. ESCAPE ANALYSIS (compile time)
       ┌──────────────────────────────────────────────────────┐
       │ Compiler determines: does 'p' escape the function?   │
       │   • If NO:  allocate on stack (free, no GC)          │
       │   • If YES: allocate on heap (GC-managed)            │
       └─────────────────────────┬────────────────────────────┘
                                 │ (escapes to heap)
                                 ▼
    2. SIZE CLASS LOOKUP (runtime)
       ┌──────────────────────────────────────────────────────┐
       │ sizeof(MyStruct) = 96 bytes → size class 10 (128 B) │
       │ (some internal fragmentation: 96/128 = 75% util)    │
       └─────────────────────────┬────────────────────────────┘
                                 ▼
    3. mcache LOOKUP (per-P, no lock)
       ┌──────────────────────────────────────────────────────┐
       │ Check P's mcache for size class 10 free list         │
       │   • If available: pop and return (fast path, ~20 ns) │
       │   • If empty: refill from mcentral                   │
       └─────────────────────────┬────────────────────────────┘
                                 │ (mcache empty)
                                 ▼
    4. mcentral (per-size-class lock)
       ┌──────────────────────────────────────────────────────┐
       │ Lock mcentral for size class 10                      │
       │ Get a span with free objects from partial list       │
       │ Transfer span to mcache, unlock                      │
       │   • If no spans: request from mheap                  │
       └─────────────────────────┬────────────────────────────┘
                                 │ (mcentral empty)
                                 ▼
    5. mheap (global lock)
       ┌──────────────────────────────────────────────────────┐
       │ Lock mheap                                           │
       │ Find a free span of appropriate size in the treap    │
       │   • If found: carve out the pages needed, unlock     │
       │   • If not found: request from OS                    │
       └─────────────────────────┬────────────────────────────┘
                                 │ (need more memory)
                                 ▼
    6. OS: mmap() SYSTEM CALL
       ┌──────────────────────────────────────────────────────┐
       │ syscall: mmap(NULL, 64KB, PROT_RW, MAP_ANON|PRIV)   │
       │ Kernel creates VMA in process address space          │
       │ NO physical memory allocated yet (demand paging)     │
       └─────────────────────────┬────────────────────────────┘
                                 │ (virtual memory reserved)
                                 ▼
    7. FIRST ACCESS → PAGE FAULT
       ┌──────────────────────────────────────────────────────┐
       │ Go runtime writes to the new memory                  │
       │ MMU: no PTE → page fault!                            │
       │ Kernel page fault handler:                           │
       │   a. Look up VMA — valid, anonymous, read-write      │
       │   b. Allocate physical frame from free list           │
       │   c. Zero-fill the frame (security)                   │
       │   d. Create PTE: virtual page → physical frame        │
       │   e. Return to userspace                              │
       │ Instruction retries and succeeds                      │
       └─────────────────────────┬────────────────────────────┘
                                 │ (page now resident)
                                 ▼
    8. PROGRAM USES THE MEMORY
       ┌──────────────────────────────────────────────────────┐
       │ p.Field = value                                      │
       │ Virtual address → TLB hit → physical address → done  │
       │ (subsequent accesses are fast: ~1-4 ns)              │
       └─────────────────────────┬────────────────────────────┘
                                 │ (eventually unreachable)
                                 ▼
    9. GARBAGE COLLECTION
       ┌──────────────────────────────────────────────────────┐
       │ GC mark phase: object not reachable → stays WHITE    │
       │ GC sweep phase: WHITE objects freed                  │
       │   a. Object returned to mspan's free list            │
       │   b. If mspan fully free: return to mcentral         │
       │   c. If mcentral has too many: return to mheap       │
       │   d. mheap may madvise(MADV_DONTNEED) to tell        │
       │      kernel the physical pages can be reclaimed       │
       └─────────────────────────┬────────────────────────────┘
                                 │ (pages reclaimed)
                                 ▼
    10. OS RECLAIMS PHYSICAL MEMORY
       ┌──────────────────────────────────────────────────────┐
       │ After madvise(MADV_DONTNEED):                        │
       │   - Virtual mapping stays (no TLB flush needed)      │
       │   - Physical frames returned to kernel free list     │
       │   - RSS decreases                                    │
       │   - Next access causes minor page fault again         │
       │     (re-allocated as zeroed page)                     │
       └──────────────────────────────────────────────────────┘
```

---

## 2.11 Key Takeaways

1. **Virtual memory is an illusion.** Your Go program sees a flat, private address space.
   Behind the scenes, the MMU translates through 4-level page tables on every access.

2. **Page faults are features, not bugs.** Demand paging, copy-on-write, and memory-mapped
   files all rely on page faults. A minor fault costs ~1-10 μs; a major fault costs ~1-10 ms.

3. **mmap is the Swiss Army knife.** Go's runtime uses it for all memory allocation. You
   can use it directly for zero-copy file I/O, shared memory IPC, and ring buffers.

4. **Go's allocator is three tiers.** mcache (per-P, lock-free) → mcentral (per-size-class
   lock) → mheap (global lock) → OS mmap. Small objects hit the fast path 99% of the time.

5. **Escape analysis determines your GC load.** Use `go build -gcflags="-m"` to understand
   what escapes. Stack allocation is free; heap allocation costs GC cycles.

6. **The GC is concurrent and tunable.** Set `GOMEMLIMIT` to your container's memory
   budget. Consider `GOGC=off` + `GOMEMLIMIT` for the most predictable behavior.

7. **Huge pages reduce TLB misses** for large working sets. Use `perf stat -e dTLB-load-misses`
   to determine if TLB pressure is a bottleneck.

8. **NUMA matters at scale.** On multi-socket servers, memory locality can cause 1.5-3x
   latency differences. Use `numactl` to control placement.

9. **The OOM killer is your last resort.** Set `oom_score_adj` to protect critical services
   and sacrifice less important ones.

## 2.12 Exercises

1. **Memory Map Explorer:** Extend the `/proc/self/maps` parser to also read
   `/proc/self/smaps` and show the RSS, PSS (Proportional Set Size), and swap usage for
   each VMA. Compare RSS vs PSS for shared library mappings.

2. **Page Fault Profiler:** Write a Go program that allocates memory in different patterns
   (sequential, random, strided) and measures page faults per pattern using `getrusage`.
   Plot the results.

3. **mmap vs read() Benchmark:** Write a benchmark that reads a 1 GB file using both
   `mmap` and `os.Read()` in 4 KB chunks. Measure throughput and page faults. Run with
   `perf stat` to compare dTLB miss rates.

4. **Escape Analysis Challenge:** Write 10 functions where you predict (before running
   the compiler) whether each local variable will escape to the heap. Verify with
   `go build -gcflags="-m -m"`. Try to minimize heap escapes.

5. **GC Tuning Lab:** Write a program that simulates a web server workload (short-lived
   request objects + long-lived cache). Run with `GOGC=50`, `GOGC=100`, `GOGC=200`, and
   `GOGC=off GOMEMLIMIT=256MiB`. Compare tail latency (p99) and memory usage.

6. **Memory-Mapped Database:** Extend the mmap KV store to support:
   - Delete operations (with tombstones)
   - Variable-length values
   - Concurrent access from multiple processes using file locks

7. **NUMA Benchmark:** On a NUMA system, write a Go program that allocates memory on one
   node and accesses it from CPUs on another node. Measure the latency difference.

---

## 2.13 Further Reading

- **Understanding the Linux Virtual Memory Manager** — Mel Gorman. The definitive reference
  for Linux memory management internals.
- **Linux Kernel Development** (3rd ed.) — Robert Love. Chapters on memory management and
  the page cache.
- **The Go Memory Model** — https://go.dev/ref/mem — Official specification of Go's memory
  ordering guarantees.
- **Go GC Guide** — https://tip.golang.org/doc/gc-guide — Official guide to tuning Go's GC.
- **TCMalloc Design** — https://google.github.io/tcmalloc/design.html — The allocator that
  inspired Go's memory allocator.
- **What Every Programmer Should Know About Memory** — Ulrich Drepper. Essential reading on
  CPU caches, TLBs, and NUMA.
- **mmap(2) man page** — `man 2 mmap` — Complete reference for the mmap system call.
- **proc(5) man page** — `man 5 proc` — Documentation for /proc filesystem entries.

---

*Next chapter: [Chapter 3: Process Management — fork, exec, namespaces, and cgroups →](./03-linux-processes.md)*
