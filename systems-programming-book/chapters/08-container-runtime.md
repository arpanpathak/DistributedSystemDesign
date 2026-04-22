# Chapter 8: Building a Container Runtime

> *"Containers are not a thing. Containers are made up of several different Linux kernel features working together."*
> — Jérôme Petazzoni

---

Containers have fundamentally changed how we build, ship, and run software. Yet to most
developers, a container remains a black box—a magical artifact produced by `docker build`
and launched by `docker run`. In this chapter, we **demystify containers entirely** by
building a fully functional container runtime from scratch in Go.

You will implement every layer of the container stack: **filesystem isolation** with
OverlayFS and `pivot_root`, **process isolation** with Linux namespaces, **resource
limits** with cgroups v2, and **network connectivity** with virtual ethernet pairs and
bridge interfaces. By the end, you will have a working runtime that can pull a rootfs,
configure resource limits, set up networking, and launch isolated processes—all in
roughly 2,000 lines of Go.

**What you will learn:**

- How container runtimes orchestrate namespaces, cgroups, and filesystems
- Building an OverlayFS-based copy-on-write filesystem layer
- Implementing `pivot_root` for secure filesystem isolation
- Configuring cgroups v2 for CPU, memory, and PID limits
- Creating virtual network interfaces (veth pairs) and Linux bridges
- Assembling all components into a working container runtime
- Hardening containers with seccomp, capabilities, and read-only filesystems
- How your runtime compares to production runtimes like runc and crun

**Prerequisites:** Chapter 5 (Linux Namespaces and Cgroups) and Chapter 7 (Virtual
Filesystems and FUSE). Familiarity with Go systems programming (Chapters 1–4).

---


## 8.1 Container Runtime Architecture

Before writing any code, let's understand what a container runtime actually does and
how the pieces fit together. A container runtime is the software responsible for
**creating and managing the lifecycle of containers** on a host system.

### 8.1.1 The Container Stack

The container ecosystem is layered. At the top sit orchestrators like Kubernetes; in
the middle are high-level runtimes like containerd and CRI-O; and at the bottom are
low-level runtimes like runc that directly interact with the kernel.

```text
┌─────────────────────────────────────────────────┐
│                  Orchestrator                    │
│              (Kubernetes, Swarm)                 │
├─────────────────────────────────────────────────┤
│              High-Level Runtime                  │
│           (containerd, CRI-O, dockerd)           │
├─────────────────────────────────────────────────┤
│              Low-Level Runtime                   │
│            (runc, crun, gVisor, kata)            │
├─────────────────────────────────────────────────┤
│                Linux Kernel                      │
│     namespaces │ cgroups │ OverlayFS │ netns     │
└─────────────────────────────────────────────────┘
```

We are building a **low-level runtime**—the component that directly calls Linux system
calls to create isolated environments. Our runtime, which we will call `gobox`, sits at
the same level as runc.

### 8.1.2 What a Low-Level Runtime Does

A low-level container runtime performs a precise sequence of operations to create a
container:

```text
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  1. Prepare  │────▶│  2. Create   │────▶│  3. Start    │
│  Filesystem  │     │  Namespaces  │     │  Process     │
└──────────────┘     └──────────────┘     └──────────────┘
       │                    │                    │
       ▼                    ▼                    ▼
  OverlayFS +          clone(2) with        execve(2) the
  pivot_root          CLONE_NEW*            user command
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  4. Apply    │────▶│  5. Setup    │────▶│  6. Monitor  │
│  Cgroups     │     │  Network     │     │  & Cleanup   │
└──────────────┘     └──────────────┘     └──────────────┘
```

Each step maps directly to kernel primitives we explored in Chapters 5 and 7:

| Step | Kernel Feature | Purpose |
|------|---------------|---------|
| Prepare Filesystem | OverlayFS, `pivot_root(2)` | Isolated, copy-on-write root filesystem |
| Create Namespaces | `clone(2)`, `unshare(2)` | Process, network, mount, UTS, IPC, user isolation |
| Start Process | `execve(2)` | Replace runtime process with container command |
| Apply Cgroups | cgroups v2 filesystem | CPU, memory, PID resource limits |
| Setup Network | `veth`, `bridge`, `netns` | Container network connectivity |
| Monitor & Cleanup | `waitpid(2)`, `rmdir(2)` | Process reaping, resource cleanup |

### 8.1.3 The gobox Runtime Architecture

Our runtime follows a **parent-child process model**. The parent process (the CLI)
orchestrates setup, and the child process (the init process) runs inside the container.

```text
┌─────────────────────────────────────────────────────────┐
│  Host (Parent Process)                                  │
│                                                         │
│  gobox run --cpu 0.5 --mem 256m alpine /bin/sh          │
│       │                                                 │
│       │  1. Prepare OverlayFS                           │
│       │  2. Configure cgroups v2                        │
│       │  3. Create veth pair                            │
│       │  4. fork/exec child with new namespaces         │
│       │                                                 │
│       ▼                                                 │
│  ┌─────────────────────────────────────────────────┐    │
│  │  Container (Child Process)                      │    │
│  │                                                 │    │
│  │  Namespaces: pid, net, mnt, uts, ipc, user      │    │
│  │                                                 │    │
│  │  5. pivot_root to new rootfs                    │    │
│  │  6. Mount /proc, /sys, /dev                     │    │
│  │  7. Set hostname                                │    │
│  │  8. Configure eth0                              │    │
│  │  9. Drop capabilities                           │    │
│  │  10. execve("/bin/sh")                          │    │
│  │                                                 │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

> 🧠 **Mental Model:** Think of the parent process as a construction foreman who
> prepares the building site (filesystem, cgroups, network) and then sends a worker
> (child process) inside to do the actual work. The foreman stays outside to monitor
> progress and clean up when the worker is done.


## 8.2 Project Setup and Structure

Let's set up our Go project with a clean, modular structure that separates concerns
across packages.

### 8.2.1 Project Layout

```text
gobox/
├── go.mod
├── go.sum
├── main.go                 # CLI entry point
├── cmd/
│   ├── run.go              # 'gobox run' command
│   └── init.go             # 'gobox init' (child process entry)
├── container/
│   ├── config.go           # Container configuration
│   ├── container.go        # Container lifecycle management
│   └── state.go            # Container state tracking
├── filesystem/
│   ├── overlay.go          # OverlayFS setup
│   ├── pivot.go            # pivot_root implementation
│   └── mount.go            # /proc, /sys, /dev mounts
├── cgroup/
│   ├── manager.go          # Cgroup v2 manager
│   └── resources.go        # Resource limit configuration
├── network/
│   ├── bridge.go           # Linux bridge setup
│   ├── veth.go             # Veth pair creation
│   └── config.go           # Network configuration
├── security/
│   ├── capabilities.go     # Linux capabilities management
│   └── seccomp.go          # Seccomp BPF profiles
└── rootfs/
    └── alpine/             # Alpine Linux rootfs (downloaded)
```

### 8.2.2 Initializing the Module

```go
// Initialize the Go module
// Run: go mod init github.com/example/gobox

// go.mod
module github.com/example/gobox

go 1.21

require (
golang.org/x/sys v0.15.0
)
```

We use `golang.org/x/sys` for Linux-specific system calls that aren't available in the
standard `syscall` package. This gives us access to `PivotRoot`, `Setns`, and other
low-level primitives.

### 8.2.3 The CLI Entry Point

💻 **Hands-On:** Let's start with the main entry point that dispatches between the
parent (run) and child (init) process roles:

```go
// main.go — CLI entry point for gobox
package main

import (
"fmt"
"os"

"github.com/example/gobox/cmd"
)

func main() {
if len(os.Args) < 2 {
printUsage()
os.Exit(1)
}

switch os.Args[1] {
case "run":
if err := cmd.Run(os.Args[2:]); err != nil {
fmt.Fprintf(os.Stderr, "gobox: %v\n", err)
os.Exit(1)
}
case "init":
// Called internally by the child process—never by the user
if err := cmd.Init(os.Args[2:]); err != nil {
fmt.Fprintf(os.Stderr, "gobox-init: %v\n", err)
os.Exit(1)
}
default:
fmt.Fprintf(os.Stderr, "gobox: unknown command %q\n", os.Args[1])
printUsage()
os.Exit(1)
}
}

func printUsage() {
fmt.Fprintf(os.Stderr, `Usage: gobox <command> [options]

Commands:
  run    Create and start a new container
  init   (internal) Initialize the container environment

Examples:
  gobox run --rootfs ./rootfs/alpine /bin/sh
  gobox run --cpu 0.5 --mem 256m --rootfs ./rootfs/alpine /bin/echo hello
`)
}
```

> ⚡ **Go Code Note:** We use the re-exec pattern here. When `gobox run` creates a child
> process with new namespaces, it calls `gobox init` as the entry point for that child.
> This pattern is used by runc and most production runtimes because Go's goroutine
> scheduler doesn't play well with `fork(2)` — you must `exec` a new process instead.


## 8.3 Container Configuration

A well-structured configuration makes the runtime flexible and testable. We define
the container specification as Go structs.

### 8.3.1 The Configuration Types

```go
// container/config.go — Container configuration types
package container

import (
"encoding/json"
"fmt"
"os"
"time"
)

// Config holds the complete specification for a container.
type Config struct {
// ID is a unique identifier for this container.
ID string `json:"id"`

// RootFS is the path to the root filesystem image.
RootFS string `json:"rootfs"`

// Command is the process to execute inside the container.
Command []string `json:"command"`

// Hostname to set inside the container.
Hostname string `json:"hostname"`

// Environment variables for the container process.
Env []string `json:"env"`

// Resources defines CPU, memory, and PID limits.
Resources ResourceConfig `json:"resources"`

// Network holds network configuration.
Network NetworkConfig `json:"network"`

// Security holds security settings.
Security SecurityConfig `json:"security"`

// WorkDir is the working directory inside the container.
WorkDir string `json:"workdir"`

// Created records when this config was generated.
Created time.Time `json:"created"`
}

// ResourceConfig defines resource limits enforced via cgroups v2.
type ResourceConfig struct {
// CPUQuota is the fraction of a CPU core (e.g., 0.5 = 50%).
CPUQuota float64 `json:"cpu_quota"`

// MemoryLimitBytes is the hard memory limit in bytes.
MemoryLimitBytes int64 `json:"memory_limit_bytes"`

// PidsLimit is the maximum number of processes.
PidsLimit int64 `json:"pids_limit"`
}

// NetworkConfig defines the container network setup.
type NetworkConfig struct {
// BridgeName is the host bridge interface name.
BridgeName string `json:"bridge_name"`

// ContainerIP is the IP address for the container (CIDR notation).
ContainerIP string `json:"container_ip"`

// GatewayIP is the bridge/gateway IP address.
GatewayIP string `json:"gateway_ip"`

// EnableNAT controls whether to set up NAT for outbound traffic.
EnableNAT bool `json:"enable_nat"`
}

// SecurityConfig defines security hardening options.
type SecurityConfig struct {
// DropCapabilities lists capabilities to drop.
DropCapabilities []string `json:"drop_capabilities"`

// ReadonlyRootfs makes the root filesystem read-only.
ReadonlyRootfs bool `json:"readonly_rootfs"`

// NoNewPrivileges prevents gaining new privileges via setuid.
NoNewPrivileges bool `json:"no_new_privileges"`

// SeccompProfile is the path to a seccomp BPF profile.
SeccompProfile string `json:"seccomp_profile"`
}
```

### 8.3.2 Default Configuration

```go
// DefaultConfig returns a sensible default configuration.
func DefaultConfig(rootfs string, command []string) *Config {
return &Config{
ID:       generateID(),
RootFS:   rootfs,
Command:  command,
Hostname: "gobox",
Env: []string{
"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
"TERM=xterm-256color",
"HOME=/root",
},
Resources: ResourceConfig{
CPUQuota:         1.0,    // 100% of one core
MemoryLimitBytes: 256 << 20, // 256 MiB
PidsLimit:        64,     // Max 64 processes
},
Network: NetworkConfig{
BridgeName:  "gobox0",
ContainerIP: "10.10.10.2/24",
GatewayIP:   "10.10.10.1/24",
EnableNAT:   true,
},
Security: SecurityConfig{
DropCapabilities: []string{
"CAP_SYS_ADMIN",
"CAP_NET_RAW",
"CAP_SYS_PTRACE",
"CAP_MKNOD",
},
ReadonlyRootfs:  false,
NoNewPrivileges: true,
},
WorkDir: "/",
Created: time.Now(),
}
}

// generateID produces a short unique container ID.
func generateID() string {
return fmt.Sprintf("gobox-%d", time.Now().UnixNano()%1000000)
}
```

### 8.3.3 Configuration Serialization

We serialize the configuration to JSON so the parent process can pass it to the child
process via a file descriptor or pipe.

```go
// Serialize writes the config as JSON to the given path.
func (c *Config) Serialize(path string) error {
data, err := json.MarshalIndent(c, "", "  ")
if err != nil {
return fmt.Errorf("marshaling config: %w", err)
}
if err := os.WriteFile(path, data, 0644); err != nil {
return fmt.Errorf("writing config to %s: %w", path, err)
}
return nil
}

// LoadConfig reads a Config from a JSON file.
func LoadConfig(path string) (*Config, error) {
data, err := os.ReadFile(path)
if err != nil {
return nil, fmt.Errorf("reading config from %s: %w", path, err)
}
var cfg Config
if err := json.Unmarshal(data, &cfg); err != nil {
return nil, fmt.Errorf("parsing config: %w", err)
}
return &cfg, nil
}
```

### 8.3.4 Parsing CLI Arguments

```go
// cmd/run.go — Parse CLI arguments into a Config
package cmd

import (
"flag"
"fmt"
"log"
"os"

"github.com/example/gobox/container"
)

// Run is the entry point for "gobox run".
func Run(args []string) error {
fs := flag.NewFlagSet("run", flag.ExitOnError)

rootfs := fs.String("rootfs", "", "Path to root filesystem")
cpu := fs.Float64("cpu", 1.0, "CPU quota (fraction of one core)")
mem := fs.String("mem", "256m", "Memory limit (e.g., 128m, 1g)")
pids := fs.Int64("pids", 64, "Maximum number of PIDs")
hostname := fs.String("hostname", "gobox", "Container hostname")

if err := fs.Parse(args); err != nil {
return fmt.Errorf("parsing flags: %w", err)
}

if *rootfs == "" {
return fmt.Errorf("--rootfs is required")
}

remaining := fs.Args()
if len(remaining) == 0 {
return fmt.Errorf("no command specified")
}

memBytes, err := parseMemoryLimit(*mem)
if err != nil {
return fmt.Errorf("invalid memory limit %q: %w", *mem, err)
}

cfg := container.DefaultConfig(*rootfs, remaining)
cfg.Hostname = *hostname
cfg.Resources.CPUQuota = *cpu
cfg.Resources.MemoryLimitBytes = memBytes
cfg.Resources.PidsLimit = *pids

log.Printf("Starting container %s", cfg.ID)
log.Printf("  Rootfs:   %s", cfg.RootFS)
log.Printf("  Command:  %v", cfg.Command)
log.Printf("  CPU:      %.1f cores", cfg.Resources.CPUQuota)
log.Printf("  Memory:   %d MiB", cfg.Resources.MemoryLimitBytes>>20)
log.Printf("  PIDs:     %d", cfg.Resources.PidsLimit)

// Create and start the container (Section 8.8)
ctr := container.New(cfg)
return ctr.Run()
}

// parseMemoryLimit converts human-readable memory strings to bytes.
func parseMemoryLimit(s string) (int64, error) {
if len(s) < 2 {
return 0, fmt.Errorf("too short")
}
suffix := s[len(s)-1]
value := s[:len(s)-1]

var n int64
if _, err := fmt.Sscanf(value, "%d", &n); err != nil {
return 0, err
}

switch suffix {
case 'k', 'K':
return n * 1024, nil
case 'm', 'M':
return n * 1024 * 1024, nil
case 'g', 'G':
return n * 1024 * 1024 * 1024, nil
default:
return 0, fmt.Errorf("unknown suffix %q", string(suffix))
}
}
```

> 🔬 **Deep Dive:** Production runtimes like runc accept OCI runtime spec JSON files
> that conform to the [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec).
> Our Config struct is a simplified version of the OCI spec's `config.json`. We will
> compare the two in Section 8.10.


## 8.4 Filesystem Isolation: OverlayFS and pivot_root

Filesystem isolation is the foundation of container security. Without it, a container
process could read and modify any file on the host. We use two complementary
mechanisms: **OverlayFS** for copy-on-write layering and **pivot_root** to change the
container's view of the root filesystem.

### 8.4.1 Understanding OverlayFS

OverlayFS is a **union filesystem** that merges multiple directory trees into a single
unified view. It supports four directories:

```text
┌─────────────────────────────────────────────────┐
│              Merged View (mountpoint)            │
│  Container sees this as its root filesystem      │
│                                                  │
│  /bin/sh  /etc/hosts  /tmp/myfile               │
└────────────┬──────────────┬─────────────────────┘
             │              │
    ┌────────┴───┐    ┌─────┴──────┐
    │ Upper Layer│    │Lower Layer │
    │  (writable)│    │ (read-only)│
    │            │    │            │
    │ /tmp/myfile│    │ /bin/sh    │
    │            │    │ /etc/hosts │
    └────────────┘    └────────────┘
             │
    ┌────────┴───┐
    │  Work Dir  │
    │ (internal) │
    │            │
    │ OverlayFS  │
    │ bookkeeping│
    └────────────┘
```

| Directory | Purpose | Persistence |
|-----------|---------|-------------|
| **Lower** | Base filesystem image (read-only) | Shared across containers |
| **Upper** | Container modifications (writes, deletes) | Per-container |
| **Work** | Internal OverlayFS bookkeeping | Per-container, opaque |
| **Merged** | Unified view presented to the container | Virtual |

When a container reads a file, OverlayFS first checks the upper layer. If the file
isn't there, it falls through to the lower layer. When a container **writes** to a file
that exists only in the lower layer, OverlayFS performs a **copy-up**: it copies the
file to the upper layer, then applies the modification there. The lower layer is never
modified.

### 8.4.2 Implementing OverlayFS Setup

```go
// filesystem/overlay.go — OverlayFS management
package filesystem

import (
"fmt"
"log"
"os"
"path/filepath"
"syscall"
)

// OverlayFS manages an overlay filesystem for a container.
type OverlayFS struct {
// ContainerID uniquely identifies the container.
ContainerID string

// LowerDir is the read-only base image (e.g., Alpine rootfs).
LowerDir string

// BaseDir is the parent directory for upper, work, and merged dirs.
BaseDir string

// Derived paths
upperDir  string
workDir   string
mergedDir string
}

// NewOverlayFS creates a new OverlayFS instance.
func NewOverlayFS(containerID, lowerDir, baseDir string) *OverlayFS {
return &OverlayFS{
ContainerID: containerID,
LowerDir:    lowerDir,
BaseDir:     baseDir,
upperDir:    filepath.Join(baseDir, containerID, "upper"),
workDir:     filepath.Join(baseDir, containerID, "work"),
mergedDir:   filepath.Join(baseDir, containerID, "merged"),
}
}

// MergedDir returns the path to the merged mount point.
func (o *OverlayFS) MergedDir() string {
return o.mergedDir
}

// Mount creates the directory structure and mounts the overlay.
func (o *OverlayFS) Mount() error {
// Create all required directories
dirs := []string{o.upperDir, o.workDir, o.mergedDir}
for _, dir := range dirs {
if err := os.MkdirAll(dir, 0755); err != nil {
return fmt.Errorf("creating directory %s: %w", dir, err)
}
}

// Construct the mount options string
opts := fmt.Sprintf(
"lowerdir=%s,upperdir=%s,workdir=%s",
o.LowerDir,
o.upperDir,
o.workDir,
)

log.Printf("Mounting OverlayFS: merged=%s", o.mergedDir)
log.Printf("  lowerdir=%s", o.LowerDir)
log.Printf("  upperdir=%s", o.upperDir)

// Mount the overlay filesystem
err := syscall.Mount("overlay", o.mergedDir, "overlay", 0, opts)
if err != nil {
return fmt.Errorf("mounting overlay at %s: %w", o.mergedDir, err)
}

log.Printf("OverlayFS mounted successfully")
return nil
}

// Unmount tears down the overlay and cleans up directories.
func (o *OverlayFS) Unmount() error {
log.Printf("Unmounting OverlayFS at %s", o.mergedDir)

if err := syscall.Unmount(o.mergedDir, 0); err != nil {
return fmt.Errorf("unmounting overlay at %s: %w", o.mergedDir, err)
}

// Clean up the container-specific directory
containerDir := filepath.Join(o.BaseDir, o.ContainerID)
if err := os.RemoveAll(containerDir); err != nil {
return fmt.Errorf("removing container dir %s: %w", containerDir, err)
}

log.Printf("OverlayFS cleanup complete")
return nil
}
```

### 8.4.3 Understanding pivot_root

While OverlayFS provides a merged view, the container process can still access the
host filesystem via `/proc` or by traversing `..` from the root. The `pivot_root(2)`
system call solves this by **swapping the root filesystem** of the calling process.

```text
Before pivot_root:                After pivot_root:

  / (host root)                    / (was merged/)
  ├── bin/                         ├── bin/
  ├── etc/                         ├── etc/
  ├── proc/                        ├── proc/ (new mount)
  ├── var/                         ├── dev/  (new mount)
  │   └── containers/              ├── sys/  (new mount)
  │       └── abc123/              └── .pivot_root/ (old host root)
  │           └── merged/              └── (unmounted)
  │               ├── bin/
  │               └── etc/
  └── ...
```

The key difference from `chroot` is that `pivot_root` actually **moves** the mount
point, while `chroot` only changes the path resolution root. A process in a `chroot`
jail can escape with a few system calls; `pivot_root` combined with unmounting the old
root is much more secure.

### 8.4.4 Implementing pivot_root

```go
// filesystem/pivot.go — pivot_root implementation
package filesystem

import (
"fmt"
"log"
"os"
"path/filepath"
"syscall"
)

// PivotRoot changes the root filesystem of the current process to newRoot.
// This must be called from inside the new mount namespace.
func PivotRoot(newRoot string) error {
log.Printf("Performing pivot_root to %s", newRoot)

// pivot_root requires that newRoot is a mount point.
// Bind-mount it onto itself to satisfy this requirement.
if err := syscall.Mount(newRoot, newRoot, "", syscall.MS_BIND|syscall.MS_REC, ""); err != nil {
return fmt.Errorf("bind mount newRoot: %w", err)
}

// Create directory for the old root
pivotDir := filepath.Join(newRoot, ".pivot_root")
if err := os.MkdirAll(pivotDir, 0700); err != nil {
return fmt.Errorf("creating pivot dir: %w", err)
}

// Perform the pivot
// After this call:
//   - newRoot becomes the new /
//   - the old / is moved to .pivot_root
if err := syscall.PivotRoot(newRoot, pivotDir); err != nil {
return fmt.Errorf("pivot_root(%s, %s): %w", newRoot, pivotDir, err)
}

// Change to the new root directory
if err := os.Chdir("/"); err != nil {
return fmt.Errorf("chdir to new root: %w", err)
}

// Unmount the old root filesystem
// MNT_DETACH performs a lazy unmount—the filesystem is unmounted
// immediately from the namespace but actual cleanup happens when
// all references are dropped.
pivotDir = "/.pivot_root"
if err := syscall.Unmount(pivotDir, syscall.MNT_DETACH); err != nil {
return fmt.Errorf("unmounting old root: %w", err)
}

// Remove the pivot directory
if err := os.Remove(pivotDir); err != nil {
return fmt.Errorf("removing pivot dir: %w", err)
}

log.Printf("pivot_root complete — new root is /")
return nil
}
```

### 8.4.5 Setting Up Essential Mounts

After `pivot_root`, the container needs several pseudo-filesystems mounted to function
properly:

```go
// filesystem/mount.go — Essential container mounts
package filesystem

import (
"fmt"
"log"
"os"
"syscall"
)

// MountSpec describes a filesystem mount to set up inside the container.
type MountSpec struct {
Source string
Target string
FSType string
Flags  uintptr
Data   string
}

// SetupContainerMounts creates essential mounts inside the container.
// Must be called after pivot_root.
func SetupContainerMounts() error {
mounts := []MountSpec{
{
// /proc — process information pseudo-filesystem
Source: "proc",
Target: "/proc",
FSType: "proc",
Flags:  syscall.MS_NOSUID | syscall.MS_NODEV | syscall.MS_NOEXEC,
},
{
// /sys — kernel and device information
Source: "sysfs",
Target: "/sys",
FSType: "sysfs",
Flags:  syscall.MS_NOSUID | syscall.MS_NODEV | syscall.MS_NOEXEC | syscall.MS_RDONLY,
},
{
// /dev — device nodes (minimal tmpfs)
Source: "tmpfs",
Target: "/dev",
FSType: "tmpfs",
Flags:  syscall.MS_NOSUID | syscall.MS_STRICTATIME,
Data:   "mode=755,size=65536k",
},
{
// /dev/pts — pseudo-terminal devices
Source: "devpts",
Target: "/dev/pts",
FSType: "devpts",
Flags:  syscall.MS_NOSUID | syscall.MS_NOEXEC,
Data:   "newinstance,ptmxmode=0666,mode=0620",
},
{
// /dev/shm — shared memory
Source: "shm",
Target: "/dev/shm",
FSType: "tmpfs",
Flags:  syscall.MS_NOSUID | syscall.MS_NODEV | syscall.MS_NOEXEC,
Data:   "mode=1777,size=65536k",
},
}

for _, m := range mounts {
log.Printf("Mounting %s at %s (type=%s)", m.Source, m.Target, m.FSType)

// Ensure the mount target directory exists
if err := os.MkdirAll(m.Target, 0755); err != nil {
return fmt.Errorf("creating mount target %s: %w", m.Target, err)
}

err := syscall.Mount(m.Source, m.Target, m.FSType, m.Flags, m.Data)
if err != nil {
return fmt.Errorf("mounting %s at %s: %w", m.Source, m.Target, err)
}
}

// Create essential device nodes
if err := createDeviceNodes(); err != nil {
return fmt.Errorf("creating device nodes: %w", err)
}

return nil
}

// createDeviceNodes creates minimal /dev entries needed by most programs.
func createDeviceNodes() error {
// Symbolic links for standard I/O
links := map[string]string{
"/dev/stdin":  "/proc/self/fd/0",
"/dev/stdout": "/proc/self/fd/1",
"/dev/stderr": "/proc/self/fd/2",
}

for dst, src := range links {
if err := os.Symlink(src, dst); err != nil {
return fmt.Errorf("creating symlink %s -> %s: %w", dst, src, err)
}
}

// Create /dev/null, /dev/zero, /dev/random, /dev/urandom
devices := []struct {
path  string
major uint32
minor uint32
}{
{"/dev/null", 1, 3},
{"/dev/zero", 1, 5},
{"/dev/random", 1, 8},
{"/dev/urandom", 1, 9},
{"/dev/tty", 5, 0},
}

for _, dev := range devices {
devNum := int(dev.major*256 + dev.minor)
if err := syscall.Mknod(dev.path, syscall.S_IFCHR|0666, devNum); err != nil {
return fmt.Errorf("mknod %s: %w", dev.path, err)
}
}

log.Printf("Device nodes created")
return nil
}
```

> 🔬 **Deep Dive:** The mount flags we use are important for security:
> - `MS_NOSUID` — Ignore set-user-ID and set-group-ID bits on executables
> - `MS_NODEV` — Do not interpret device special files
> - `MS_NOEXEC` — Do not allow program execution from this filesystem
> - `MS_RDONLY` — Mount read-only
>
> Production runtimes apply these flags extensively to minimize the container's attack
> surface. We mount `/sys` as read-only because containers should not modify kernel
> parameters.


## 8.5 Namespace Isolation

In Chapter 5, we explored Linux namespaces in detail. Now we apply that knowledge to
our container runtime. We create a new set of namespaces for each container, giving it
an isolated view of PIDs, network interfaces, mount points, hostnames, and IPC
resources.

### 8.5.1 Namespace Overview for Containers

```text
┌─────────────────────────────────────────────────────────────┐
│                    Host Namespaces                          │
│                                                             │
│  PID NS (host)  │  NET NS (host)   │  MNT NS (host)       │
│  pid 1: systemd │  eth0: 10.0.2.15 │  /: ext4             │
│  pid 42: gobox  │  gobox0: bridge   │  /proc, /sys, ...    │
│                 │                   │                       │
├─────────────────┼───────────────────┼───────────────────────┤
│                    Container Namespaces                     │
│                                                             │
│  PID NS (new)   │  NET NS (new)    │  MNT NS (new)        │
│  pid 1: /bin/sh │  eth0: 10.10.10.2│  /: overlayfs        │
│                 │  lo:  127.0.0.1  │  /proc (new mount)   │
│                 │                   │                       │
│  UTS NS (new)   │  IPC NS (new)    │  USER NS (optional)  │
│  hostname:gobox │  isolated shm    │  uid mapping         │
└─────────────────┴───────────────────┴───────────────────────┘
```

Each namespace type provides specific isolation:

| Namespace | Clone Flag | What It Isolates |
|-----------|-----------|------------------|
| PID | `CLONE_NEWPID` | Process ID number space |
| Network | `CLONE_NEWNET` | Network interfaces, routing, iptables |
| Mount | `CLONE_NEWNS` | Mount points and filesystem tree |
| UTS | `CLONE_NEWUTS` | Hostname and domain name |
| IPC | `CLONE_NEWIPC` | System V IPC, POSIX message queues |
| User | `CLONE_NEWUSER` | User/group ID mappings |
| Cgroup | `CLONE_NEWCGROUP` | Cgroup root directory |

### 8.5.2 Creating the Child Process with Namespaces

In Go, we cannot use `fork(2)` directly because the Go runtime's goroutine scheduler
uses multiple OS threads. Instead, we use `os/exec` to re-execute ourselves with new
namespaces via `SysProcAttr.Cloneflags`.

```go
// container/container.go — Container lifecycle management
package container

import (
"encoding/json"
"fmt"
"log"
"os"
"os/exec"
"path/filepath"
"syscall"

"github.com/example/gobox/cgroup"
"github.com/example/gobox/filesystem"
"github.com/example/gobox/network"
)

// Container manages the lifecycle of a single container.
type Container struct {
Config  *Config
overlay *filesystem.OverlayFS
cgroup  *cgroup.Manager
net     *network.Veth
}

// New creates a new Container from the given config.
func New(cfg *Config) *Container {
return &Container{
Config: cfg,
}
}

// Run creates and starts the container. This is the parent process entry point.
func (c *Container) Run() error {
log.Printf("=== Starting container %s ===", c.Config.ID)

// Step 1: Set up the overlay filesystem
if err := c.setupFilesystem(); err != nil {
return fmt.Errorf("filesystem setup: %w", err)
}
defer c.cleanup()

// Step 2: Set up cgroups
if err := c.setupCgroups(); err != nil {
return fmt.Errorf("cgroup setup: %w", err)
}

// Step 3: Write config for the child process
configPath := filepath.Join(c.overlay.MergedDir(), ".gobox-config.json")
if err := c.Config.Serialize(configPath); err != nil {
return fmt.Errorf("serializing config: %w", err)
}

// Step 4: Create the child process with new namespaces
child, err := c.createChild(configPath)
if err != nil {
return fmt.Errorf("creating child process: %w", err)
}

// Step 5: Set up networking (requires child's PID for netns)
if err := c.setupNetworking(child.Process.Pid); err != nil {
log.Printf("WARNING: network setup failed: %v", err)
// Continue without networking—container still works for local tasks
}

// Step 6: Wait for the child process to exit
log.Printf("Waiting for container process (PID %d)", child.Process.Pid)
if err := child.Wait(); err != nil {
return fmt.Errorf("container exited with error: %w", err)
}

log.Printf("=== Container %s exited ===", c.Config.ID)
return nil
}

// createChild spawns the container init process with new namespaces.
func (c *Container) createChild(configPath string) (*exec.Cmd, error) {
// Re-execute ourselves as "gobox init" with the config path
child := exec.Command("/proc/self/exe", "init", configPath)

// Set up new namespaces for the child
child.SysProcAttr = &syscall.SysProcAttr{
Cloneflags: syscall.CLONE_NEWPID |    // New PID namespace
syscall.CLONE_NEWNS |             // New mount namespace
syscall.CLONE_NEWUTS |            // New UTS namespace
syscall.CLONE_NEWIPC |            // New IPC namespace
syscall.CLONE_NEWNET,             // New network namespace
// Unshare the filesystem attributes so mount changes
// don't propagate to the host
Unshareflags: syscall.CLONE_NEWNS,
}

// Connect stdio
child.Stdin = os.Stdin
child.Stdout = os.Stdout
child.Stderr = os.Stderr

log.Printf("Starting child process with new namespaces")
if err := child.Start(); err != nil {
return nil, fmt.Errorf("starting child: %w", err)
}

log.Printf("Child process started: PID %d", child.Process.Pid)
return child, nil
}
```

### 8.5.3 The Init Process (Child Side)

The init process runs inside the new namespaces. It performs the container-side setup:
`pivot_root`, mount essential filesystems, set hostname, and finally `exec` the user's
command.

```go
// cmd/init.go — Container init process (runs inside namespaces)
package cmd

import (
"fmt"
"log"
"os"
"syscall"

"github.com/example/gobox/container"
"github.com/example/gobox/filesystem"
)

// Init is the entry point for the container init process.
// It is called as "gobox init <config-path>".
func Init(args []string) error {
if len(args) < 1 {
return fmt.Errorf("init requires config path argument")
}

configPath := args[0]

// Load the config written by the parent
cfg, err := container.LoadConfig(configPath)
if err != nil {
return fmt.Errorf("loading config: %w", err)
}

log.Printf("[init] Container %s starting", cfg.ID)

// Step 1: Set the hostname in the new UTS namespace
if err := syscall.Sethostname([]byte(cfg.Hostname)); err != nil {
return fmt.Errorf("setting hostname: %w", err)
}
log.Printf("[init] Hostname set to %s", cfg.Hostname)

// Step 2: Perform pivot_root to the overlay merged directory
if err := filesystem.PivotRoot(cfg.RootFS); err != nil {
return fmt.Errorf("pivot_root: %w", err)
}
log.Printf("[init] pivot_root complete")

// Step 3: Mount /proc, /sys, /dev inside the container
if err := filesystem.SetupContainerMounts(); err != nil {
return fmt.Errorf("setting up mounts: %w", err)
}
log.Printf("[init] Mounts configured")

// Step 4: Change to the working directory
if err := os.Chdir(cfg.WorkDir); err != nil {
return fmt.Errorf("chdir to %s: %w", cfg.WorkDir, err)
}

// Step 5: Set environment variables
for _, env := range cfg.Env {
parts := splitEnvVar(env)
if len(parts) == 2 {
os.Setenv(parts[0], parts[1])
}
}

// Step 6: Clean up the config file
os.Remove("/.gobox-config.json")

// Step 7: Execute the requested command
// syscall.Exec replaces the current process image—this never returns
// on success.
log.Printf("[init] Executing: %v", cfg.Command)
binary, err := findExecutable(cfg.Command[0])
if err != nil {
return fmt.Errorf("finding executable %q: %w", cfg.Command[0], err)
}

return syscall.Exec(binary, cfg.Command, cfg.Env)
}

// splitEnvVar splits "KEY=VALUE" into ["KEY", "VALUE"].
func splitEnvVar(env string) []string {
for i, c := range env {
if c == '=' {
return []string{env[:i], env[i+1:]}
}
}
return []string{env}
}

// findExecutable searches PATH for the given command.
func findExecutable(name string) (string, error) {
// If it's already an absolute path, use it directly
if name[0] == '/' {
if _, err := os.Stat(name); err != nil {
return "", fmt.Errorf("stat %s: %w", name, err)
}
return name, nil
}

// Search PATH
paths := []string{"/usr/local/bin", "/usr/bin", "/bin", "/sbin"}
for _, dir := range paths {
full := dir + "/" + name
if _, err := os.Stat(full); err == nil {
return full, nil
}
}

return "", fmt.Errorf("command not found: %s", name)
}
```

> 🧠 **Mental Model:** The init process is like a astronaut entering a space station
> through an airlock. The airlock sequence is: seal the door (pivot_root), pressurize
> (mount filesystems), set controls (hostname, env), then float free (exec the command).
> Once `syscall.Exec` succeeds, the init process ceases to exist—it's replaced entirely
> by the user's command.


## 8.6 Resource Limits with Cgroups v2

Namespaces provide isolation of **what a process can see**; cgroups control **how much
it can use**. We implement a cgroups v2 manager that limits CPU, memory, and the number
of processes a container can create.

### 8.6.1 Cgroups v2 Filesystem Interface

Cgroups v2 uses a **unified hierarchy** mounted at `/sys/fs/cgroup`. Each cgroup is a
directory, and resource limits are set by writing to files within that directory.

```text
/sys/fs/cgroup/
├── cgroup.controllers         # Available controllers
├── cgroup.subtree_control     # Enabled controllers for children
├── gobox/                     # Our runtime's cgroup
│   ├── gobox-abc123/          # Per-container cgroup
│   │   ├── cgroup.procs       # PIDs in this cgroup
│   │   ├── cpu.max            # CPU limit
│   │   ├── memory.max         # Memory limit
│   │   ├── memory.current     # Current memory usage
│   │   ├── pids.max           # Max PIDs
│   │   └── pids.current       # Current PID count
│   └── cgroup.subtree_control
└── ...
```

### 8.6.2 The Cgroup Manager

```go
// cgroup/manager.go — Cgroups v2 resource manager
package cgroup

import (
"fmt"
"log"
"os"
"path/filepath"
"strconv"
"strings"
)

const (
// cgroupRoot is the cgroups v2 mount point.
cgroupRoot = "/sys/fs/cgroup"

// cgroupPrefix is our runtime's cgroup subtree.
cgroupPrefix = "gobox"
)

// Manager controls cgroup resources for a single container.
type Manager struct {
// containerID is the unique container identifier.
containerID string

// path is the full path to this container's cgroup directory.
path string

// resources holds the configured limits.
resources Resources
}

// Resources defines the resource limits to apply.
type Resources struct {
// CPUQuota is the fraction of a CPU core (e.g., 0.5 = 50%).
CPUQuota float64

// MemoryLimitBytes is the hard memory limit.
MemoryLimitBytes int64

// PidsLimit is the maximum number of processes.
PidsLimit int64
}

// NewManager creates a cgroup manager for the given container.
func NewManager(containerID string, res Resources) *Manager {
return &Manager{
containerID: containerID,
path:        filepath.Join(cgroupRoot, cgroupPrefix, containerID),
resources:   res,
}
}

// Setup creates the cgroup directory and applies resource limits.
func (m *Manager) Setup() error {
log.Printf("Setting up cgroup at %s", m.path)

// Ensure the parent cgroup exists and has controllers enabled
parentPath := filepath.Join(cgroupRoot, cgroupPrefix)
if err := os.MkdirAll(parentPath, 0755); err != nil {
return fmt.Errorf("creating parent cgroup: %w", err)
}

// Enable controllers in the parent
if err := enableControllers(parentPath); err != nil {
return fmt.Errorf("enabling controllers: %w", err)
}

// Create the container's cgroup directory
if err := os.MkdirAll(m.path, 0755); err != nil {
return fmt.Errorf("creating cgroup dir: %w", err)
}

// Apply resource limits
if err := m.applyCPULimit(); err != nil {
return fmt.Errorf("setting CPU limit: %w", err)
}

if err := m.applyMemoryLimit(); err != nil {
return fmt.Errorf("setting memory limit: %w", err)
}

if err := m.applyPidsLimit(); err != nil {
return fmt.Errorf("setting PIDs limit: %w", err)
}

log.Printf("Cgroup configured: cpu=%.1f, mem=%d MiB, pids=%d",
m.resources.CPUQuota,
m.resources.MemoryLimitBytes>>20,
m.resources.PidsLimit)

return nil
}

// enableControllers enables cpu, memory, and pids controllers on a cgroup.
func enableControllers(path string) error {
controllers := "+cpu +memory +pids"
controlFile := filepath.Join(path, "cgroup.subtree_control")

if err := os.WriteFile(controlFile, []byte(controllers), 0644); err != nil {
return fmt.Errorf("writing subtree_control at %s: %w", controlFile, err)
}

return nil
}
```

### 8.6.3 Applying CPU Limits

The CPU controller in cgroups v2 uses a **quota/period** model. The `cpu.max` file
takes two values: the quota (microseconds of CPU time) and the period (microseconds).

```text
cpu.max format:  QUOTA PERIOD

Examples:
  "50000 100000"   → 50% of one CPU core (50ms every 100ms)
  "100000 100000"  → 100% of one CPU core
  "200000 100000"  → 200% = 2 CPU cores
  "max 100000"     → Unlimited CPU
```

```go
// applyCPULimit sets the CPU quota based on the configured fraction.
func (m *Manager) applyCPULimit() error {
const period = 100000 // 100ms in microseconds

var quotaStr string
if m.resources.CPUQuota <= 0 {
quotaStr = "max" // Unlimited
} else {
quota := int(m.resources.CPUQuota * float64(period))
quotaStr = strconv.Itoa(quota)
}

value := fmt.Sprintf("%s %d", quotaStr, period)
return m.writeFile("cpu.max", value)
}
```

### 8.6.4 Applying Memory Limits

```go
// applyMemoryLimit sets the hard memory limit.
func (m *Manager) applyMemoryLimit() error {
if m.resources.MemoryLimitBytes <= 0 {
return m.writeFile("memory.max", "max")
}
return m.writeFile("memory.max", strconv.FormatInt(m.resources.MemoryLimitBytes, 10))
}
```

### 8.6.5 Applying PID Limits

```go
// applyPidsLimit sets the maximum number of processes.
func (m *Manager) applyPidsLimit() error {
if m.resources.PidsLimit <= 0 {
return m.writeFile("pids.max", "max")
}
return m.writeFile("pids.max", strconv.FormatInt(m.resources.PidsLimit, 10))
}
```

### 8.6.6 Adding a Process to the Cgroup

```go
// AddProcess adds a process (by PID) to this cgroup.
func (m *Manager) AddProcess(pid int) error {
log.Printf("Adding PID %d to cgroup %s", pid, m.path)
return m.writeFile("cgroup.procs", strconv.Itoa(pid))
}

// writeFile is a helper that writes a value to a cgroup control file.
func (m *Manager) writeFile(filename, value string) error {
path := filepath.Join(m.path, filename)
if err := os.WriteFile(path, []byte(value), 0644); err != nil {
return fmt.Errorf("writing %s to %s: %w", value, path, err)
}
log.Printf("  cgroup: %s = %s", filename, strings.TrimSpace(value))
return nil
}

// GetMemoryUsage reads the current memory consumption.
func (m *Manager) GetMemoryUsage() (int64, error) {
data, err := os.ReadFile(filepath.Join(m.path, "memory.current"))
if err != nil {
return 0, fmt.Errorf("reading memory.current: %w", err)
}
return strconv.ParseInt(strings.TrimSpace(string(data)), 10, 64)
}

// GetPidsCount reads the current number of processes.
func (m *Manager) GetPidsCount() (int64, error) {
data, err := os.ReadFile(filepath.Join(m.path, "pids.current"))
if err != nil {
return 0, fmt.Errorf("reading pids.current: %w", err)
}
return strconv.ParseInt(strings.TrimSpace(string(data)), 10, 64)
}

// Cleanup removes the cgroup directory.
func (m *Manager) Cleanup() error {
log.Printf("Removing cgroup %s", m.path)

// We can only remove a cgroup directory when it has no processes
if err := os.Remove(m.path); err != nil {
return fmt.Errorf("removing cgroup %s: %w", m.path, err)
}
return nil
}
```

> 🔬 **Deep Dive: OOM Killer Integration**
>
> When a container exceeds its memory limit, the kernel's OOM (Out-Of-Memory) killer
> terminates processes within the cgroup. You can monitor this via `memory.events`:
>
> ```text
> $ cat /sys/fs/cgroup/gobox/abc123/memory.events
> low 0
> high 0
> max 3        ← Container hit the memory limit 3 times
> oom 1        ← OOM killer was invoked once
> oom_kill 1   ← One process was killed
> ```
>
> Production runtimes often set `memory.oom.group = 1` to kill all processes in the
> cgroup when OOM occurs, preventing partial container states.


## 8.7 Container Networking

A container without networking is useful for batch jobs but insufficient for most real
workloads. In this section, we build a basic container network using **veth pairs** and
a **Linux bridge**.

### 8.7.1 Container Networking Model

```text
┌──────────────────────────────────────────────────────────────┐
│  Host                                                        │
│                                                              │
│  ┌──────────┐       ┌───────────────┐       ┌──────────┐    │
│  │   eth0   │       │   gobox0      │       │ iptables │    │
│  │10.0.2.15 │       │  (bridge)     │       │   NAT    │    │
│  │          │◄─────▶│ 10.10.10.1/24 │◄─────▶│ MASQUERADE│   │
│  └──────────┘       └───────┬───────┘       └──────────┘    │
│                             │                                │
│                     ┌───────┴───────┐                        │
│                     │  veth-abc123  │                        │
│                     │  (host side)  │                        │
│                     └───────┬───────┘                        │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │
│                             │  network namespace boundary    │
│                     ┌───────┴───────┐                        │
│                     │     eth0      │                        │
│                     │  10.10.10.2   │                        │
│                     │  (container)  │                        │
│                     └───────────────┘                        │
│                                                              │
│  Container Network Namespace                                 │
│  ┌─────────────────────────────────────────────────────┐     │
│  │  eth0: 10.10.10.2/24                                │     │
│  │  lo:   127.0.0.1                                    │     │
│  │  default route → 10.10.10.1 (bridge)                │     │
│  │                                                     │     │
│  │  Process: /bin/sh (PID 1 inside namespace)          │     │
│  └─────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

The networking setup involves three key steps:
1. Create a **bridge interface** on the host (acts as a virtual switch)
2. Create a **veth pair** — one end goes in the container, one on the host
3. Set up **NAT/masquerade** rules so the container can reach the internet

### 8.7.2 Creating the Bridge Interface

```go
// network/bridge.go — Linux bridge management
package network

import (
"fmt"
"log"
"net"
"os/exec"
"strings"
)

// Bridge represents a Linux bridge interface.
type Bridge struct {
Name string
IP   string // CIDR notation, e.g. "10.10.10.1/24"
}

// SetupBridge creates and configures a Linux bridge.
// If the bridge already exists, it just ensures it is up.
func SetupBridge(name, ipCIDR string) (*Bridge, error) {
b := &Bridge{Name: name, IP: ipCIDR}

// Check if bridge already exists
if interfaceExists(name) {
log.Printf("Bridge %s already exists, ensuring it is up", name)
return b, run("ip", "link", "set", name, "up")
}

log.Printf("Creating bridge %s with IP %s", name, ipCIDR)

// Create the bridge interface
if err := run("ip", "link", "add", name, "type", "bridge"); err != nil {
return nil, fmt.Errorf("creating bridge: %w", err)
}

// Assign IP address
if err := run("ip", "addr", "add", ipCIDR, "dev", name); err != nil {
return nil, fmt.Errorf("assigning IP to bridge: %w", err)
}

// Bring the bridge up
if err := run("ip", "link", "set", name, "up"); err != nil {
return nil, fmt.Errorf("bringing bridge up: %w", err)
}

log.Printf("Bridge %s created and up", name)
return b, nil
}

// EnableNAT sets up iptables rules for container internet access.
func EnableNAT(bridgeName, subnet string) error {
log.Printf("Enabling NAT for subnet %s via bridge %s", subnet, bridgeName)

// Enable IP forwarding
if err := run("sysctl", "-w", "net.ipv4.ip_forward=1"); err != nil {
return fmt.Errorf("enabling IP forwarding: %w", err)
}

// Add MASQUERADE rule for the container subnet
// This rewrites the source IP of outgoing packets from the container
// to the host's IP address.
err := run("iptables", "-t", "nat", "-A", "POSTROUTING",
"-s", subnet, "-j", "MASQUERADE")
if err != nil {
return fmt.Errorf("adding NAT rule: %w", err)
}

// Allow forwarding for the bridge
err = run("iptables", "-A", "FORWARD",
"-i", bridgeName, "-j", "ACCEPT")
if err != nil {
return fmt.Errorf("adding forward rule: %w", err)
}

err = run("iptables", "-A", "FORWARD",
"-o", bridgeName, "-j", "ACCEPT")
if err != nil {
return fmt.Errorf("adding return forward rule: %w", err)
}

return nil
}

// interfaceExists checks if a network interface already exists.
func interfaceExists(name string) bool {
ifaces, err := net.Interfaces()
if err != nil {
return false
}
for _, iface := range ifaces {
if iface.Name == name {
return true
}
}
return false
}

// run executes a command and returns any error.
func run(name string, args ...string) error {
cmd := exec.Command(name, args...)
output, err := cmd.CombinedOutput()
if err != nil {
return fmt.Errorf("%s %s: %s: %w",
name, strings.Join(args, " "),
strings.TrimSpace(string(output)), err)
}
return nil
}
```

### 8.7.3 Creating Veth Pairs

A **veth pair** is a virtual ethernet cable with two ends. One end stays in the host
namespace and connects to the bridge; the other end moves into the container's network
namespace.

```go
// network/veth.go — Virtual ethernet pair management
package network

import (
"fmt"
"log"
)

// Veth represents a virtual ethernet pair.
type Veth struct {
// HostName is the interface name on the host side.
HostName string

// ContainerName is the interface name inside the container.
ContainerName string

// ContainerIP is the IP address assigned to the container end.
ContainerIP string

// GatewayIP is the default gateway inside the container.
GatewayIP string

// BridgeName is the bridge to attach the host end to.
BridgeName string

// ContainerPID is the PID of the container init process.
ContainerPID int
}

// NewVeth creates a new veth pair configuration.
func NewVeth(containerID string, containerPID int, cfg NetworkConfig) *Veth {
// Use container ID to make unique interface names
hostName := fmt.Sprintf("veth-%s", containerID[:8])

return &Veth{
HostName:      hostName,
ContainerName: "eth0",
ContainerIP:   cfg.ContainerIP,
GatewayIP:     cfg.GatewayIP,
BridgeName:    cfg.BridgeName,
ContainerPID:  containerPID,
}
}

// Setup creates the veth pair and configures both ends.
func (v *Veth) Setup() error {
log.Printf("Setting up veth pair: %s <-> %s (PID %d)",
v.HostName, v.ContainerName, v.ContainerPID)

// Step 1: Create the veth pair
err := run("ip", "link", "add", v.HostName,
"type", "veth", "peer", "name", v.ContainerName)
if err != nil {
return fmt.Errorf("creating veth pair: %w", err)
}

// Step 2: Attach host end to the bridge
if err := run("ip", "link", "set", v.HostName, "master", v.BridgeName); err != nil {
return fmt.Errorf("attaching %s to bridge: %w", v.HostName, err)
}

// Step 3: Bring up the host end
if err := run("ip", "link", "set", v.HostName, "up"); err != nil {
return fmt.Errorf("bringing up %s: %w", v.HostName, err)
}

// Step 4: Move the container end into the container's network namespace
// We reference the namespace by PID
pidStr := fmt.Sprintf("%d", v.ContainerPID)
err = run("ip", "link", "set", v.ContainerName, "netns", pidStr)
if err != nil {
return fmt.Errorf("moving %s to netns: %w", v.ContainerName, err)
}

// Step 5: Configure the container end (run commands in the container's netns)
nsPrefix := []string{"nsenter", "-t", pidStr, "-n", "--"}

// Assign IP address
if err := runWithPrefix(nsPrefix, "ip", "addr", "add", v.ContainerIP, "dev", v.ContainerName); err != nil {
return fmt.Errorf("assigning container IP: %w", err)
}

// Bring up the container interface
if err := runWithPrefix(nsPrefix, "ip", "link", "set", v.ContainerName, "up"); err != nil {
return fmt.Errorf("bringing up container interface: %w", err)
}

// Bring up loopback
if err := runWithPrefix(nsPrefix, "ip", "link", "set", "lo", "up"); err != nil {
return fmt.Errorf("bringing up loopback: %w", err)
}

// Extract gateway IP without CIDR prefix for the route
gwIP := stripCIDR(v.GatewayIP)

// Set default route
if err := runWithPrefix(nsPrefix, "ip", "route", "add", "default", "via", gwIP); err != nil {
return fmt.Errorf("setting default route: %w", err)
}

log.Printf("Veth pair configured: %s (host) <-> %s@%s (container)",
v.HostName, v.ContainerName, v.ContainerIP)

return nil
}

// Cleanup removes the veth pair (removing one end removes both).
func (v *Veth) Cleanup() error {
log.Printf("Cleaning up veth pair %s", v.HostName)
return run("ip", "link", "del", v.HostName)
}

// runWithPrefix executes a command with a prefix (e.g., nsenter).
func runWithPrefix(prefix []string, name string, args ...string) error {
fullArgs := append(prefix, name)
fullArgs = append(fullArgs, args...)
return run(fullArgs[0], fullArgs[1:]...)
}

// stripCIDR removes the /prefix from a CIDR address.
func stripCIDR(cidr string) string {
for i, c := range cidr {
if c == '/' {
return cidr[:i]
}
}
return cidr
}
```

### 8.7.4 Network Configuration Type

```go
// network/config.go — Network configuration
package network

// NetworkConfig holds networking parameters for a container.
type NetworkConfig struct {
BridgeName  string
ContainerIP string
GatewayIP   string
EnableNAT   bool
}
```

> 🧠 **Mental Model:** Think of a veth pair as a network cable with a plug on each
> end. One plug goes into the bridge (a virtual network switch) on the host side. The
> other plug goes into the container's network namespace. The bridge connects all
> containers to each other and, via NAT, to the outside world.

### 8.7.5 How Container DNS Works

For DNS resolution, the container needs a valid `/etc/resolv.conf`. Our runtime writes
one during filesystem setup:

```go
// filesystem/dns.go — DNS configuration for containers
package filesystem

import (
"fmt"
"os"
)

// SetupDNS creates /etc/resolv.conf inside the container rootfs.
func SetupDNS(rootfs string) error {
resolvConf := rootfs + "/etc/resolv.conf"

content := "# Generated by gobox\nnameserver 8.8.8.8\nnameserver 8.8.4.4\n"

if err := os.WriteFile(resolvConf, []byte(content), 0644); err != nil {
return fmt.Errorf("writing resolv.conf: %w", err)
}

return nil
}
```


## 8.8 Assembling the Complete Runtime

Now we connect all the pieces. The parent process orchestrates filesystem, cgroups, and
networking setup, then spawns the child process.

### 8.8.1 Parent Process: Filesystem Setup

```go
// container/container.go — continued

// setupFilesystem creates the OverlayFS and prepares the rootfs.
func (c *Container) setupFilesystem() error {
baseDir := "/var/lib/gobox/containers"

c.overlay = filesystem.NewOverlayFS(
c.Config.ID,
c.Config.RootFS,
baseDir,
)

if err := c.overlay.Mount(); err != nil {
return fmt.Errorf("mounting overlay: %w", err)
}

// Set up DNS in the merged rootfs
if err := filesystem.SetupDNS(c.overlay.MergedDir()); err != nil {
return fmt.Errorf("setting up DNS: %w", err)
}

// Write /etc/hostname
hostnameFile := c.overlay.MergedDir() + "/etc/hostname"
if err := os.WriteFile(hostnameFile, []byte(c.Config.Hostname+"\n"), 0644); err != nil {
log.Printf("WARNING: could not write hostname file: %v", err)
}

return nil
}
```

### 8.8.2 Parent Process: Cgroup Setup

```go
// setupCgroups creates the cgroup and applies resource limits.
func (c *Container) setupCgroups() error {
c.cgroup = cgroup.NewManager(c.Config.ID, cgroup.Resources{
CPUQuota:         c.Config.Resources.CPUQuota,
MemoryLimitBytes: c.Config.Resources.MemoryLimitBytes,
PidsLimit:        c.Config.Resources.PidsLimit,
})

return c.cgroup.Setup()
}
```

### 8.8.3 Parent Process: Network Setup

```go
// setupNetworking creates the bridge and veth pair.
func (c *Container) setupNetworking(childPID int) error {
netCfg := network.NetworkConfig{
BridgeName:  c.Config.Network.BridgeName,
ContainerIP: c.Config.Network.ContainerIP,
GatewayIP:   c.Config.Network.GatewayIP,
EnableNAT:   c.Config.Network.EnableNAT,
}

// Create or ensure bridge exists
_, err := network.SetupBridge(netCfg.BridgeName, netCfg.GatewayIP)
if err != nil {
return fmt.Errorf("setting up bridge: %w", err)
}

// Enable NAT if requested
if netCfg.EnableNAT {
subnet := "10.10.10.0/24"
if err := network.EnableNAT(netCfg.BridgeName, subnet); err != nil {
return fmt.Errorf("enabling NAT: %w", err)
}
}

// Create veth pair and configure container networking
c.net = network.NewVeth(c.Config.ID, childPID, netCfg)
if err := c.net.Setup(); err != nil {
return fmt.Errorf("setting up veth: %w", err)
}

// Add child to the cgroup
if err := c.cgroup.AddProcess(childPID); err != nil {
return fmt.Errorf("adding process to cgroup: %w", err)
}

return nil
}
```

### 8.8.4 Cleanup

```go
// cleanup tears down all container resources.
func (c *Container) cleanup() {
log.Printf("Cleaning up container %s", c.Config.ID)

if c.net != nil {
if err := c.net.Cleanup(); err != nil {
log.Printf("WARNING: veth cleanup failed: %v", err)
}
}

if c.cgroup != nil {
if err := c.cgroup.Cleanup(); err != nil {
log.Printf("WARNING: cgroup cleanup failed: %v", err)
}
}

if c.overlay != nil {
if err := c.overlay.Unmount(); err != nil {
log.Printf("WARNING: overlay cleanup failed: %v", err)
}
}

log.Printf("Cleanup complete")
}
```

### 8.8.5 Container State Machine

A container follows a well-defined lifecycle:

```text
                    ┌─────────────┐
                    │   Created   │
                    └──────┬──────┘
                           │ Run()
                           ▼
                    ┌─────────────┐
           ┌───────│   Starting  │───────┐
           │       └──────┬──────┘       │
           │              │              │
           ▼              ▼              ▼
   ┌──────────────┐ ┌──────────┐ ┌──────────────┐
   │  FS Setup    │ │  Cgroup  │ │   Network    │
   │  (overlay +  │ │  Setup   │ │   Setup      │
   │  pivot_root) │ │          │ │ (bridge+veth)│
   └──────┬───────┘ └────┬─────┘ └──────┬───────┘
          │               │              │
          └───────────────┼──────────────┘
                          │
                          ▼
                   ┌─────────────┐
                   │   Running   │
                   │  (child is  │
                   │  executing) │
                   └──────┬──────┘
                          │ child exits
                          ▼
                   ┌─────────────┐
                   │   Stopped   │
                   │  (cleanup)  │
                   └──────┬──────┘
                          │
                          ▼
                   ┌─────────────┐
                   │  Destroyed  │
                   └─────────────┘
```

```go
// container/state.go — Container state tracking
package container

import (
"fmt"
"sync"
"time"
)

// State represents the container lifecycle state.
type State int

const (
StateCreated State = iota
StateStarting
StateRunning
StateStopped
StateDestroyed
)

// String returns a human-readable state name.
func (s State) String() string {
switch s {
case StateCreated:
return "created"
case StateStarting:
return "starting"
case StateRunning:
return "running"
case StateStopped:
return "stopped"
case StateDestroyed:
return "destroyed"
default:
return "unknown"
}
}

// ContainerState tracks the current state and transitions.
type ContainerState struct {
mu        sync.Mutex
current   State
pid       int
exitCode  int
startedAt time.Time
stoppedAt time.Time
}

// NewContainerState creates a state tracker in the Created state.
func NewContainerState() *ContainerState {
return &ContainerState{
current: StateCreated,
}
}

// Transition moves to a new state, validating the transition is legal.
func (cs *ContainerState) Transition(newState State) error {
cs.mu.Lock()
defer cs.mu.Unlock()

// Define valid transitions
valid := map[State][]State{
StateCreated:  {StateStarting},
StateStarting: {StateRunning, StateStopped},
StateRunning:  {StateStopped},
StateStopped:  {StateDestroyed},
}

allowed, ok := valid[cs.current]
if !ok {
return fmt.Errorf("no transitions from state %s", cs.current)
}

for _, s := range allowed {
if s == newState {
cs.current = newState
if newState == StateRunning {
cs.startedAt = time.Now()
}
if newState == StateStopped {
cs.stoppedAt = time.Now()
}
return nil
}
}

return fmt.Errorf("invalid transition: %s → %s", cs.current, newState)
}

// Current returns the current state.
func (cs *ContainerState) Current() State {
cs.mu.Lock()
defer cs.mu.Unlock()
return cs.current
}
```

> ⚡ **Go Code Note:** The state machine uses a mutex to ensure thread safety. This
> matters when multiple goroutines may check or modify container state concurrently—for
> example, a monitoring goroutine reading state while the main goroutine processes the
> child exit.


## 8.9 Security Hardening

A basic container with namespaces and cgroups provides isolation, but a determined
attacker can still escape. In this section, we add defense-in-depth measures:
**Linux capabilities**, **seccomp BPF filters**, and additional mount hardening.

### 8.9.1 Linux Capabilities

Linux divides the traditional superuser privilege into discrete units called
**capabilities**. Instead of granting a container full root power, we give it only the
capabilities it needs.

```text
┌────────────────────────────────────────────────────────────────┐
│                    Full Root Privileges                        │
│                                                                │
│  CAP_SYS_ADMIN   CAP_NET_ADMIN   CAP_SYS_PTRACE              │
│  CAP_NET_RAW     CAP_MKNOD       CAP_SYS_MODULE              │
│  CAP_DAC_OVERRIDE CAP_CHOWN      CAP_SETUID                  │
│  CAP_SETGID      CAP_KILL        CAP_NET_BIND_SERVICE        │
│  ... (41 capabilities total)                                  │
└────────────────────────────────────────────────────────────────┘
                              │
                     Drop dangerous ones
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                Container Capabilities                          │
│                                                                │
│  ✅ CAP_CHOWN       ✅ CAP_SETUID     ✅ CAP_SETGID           │
│  ✅ CAP_KILL        ✅ CAP_NET_BIND_SERVICE                   │
│  ❌ CAP_SYS_ADMIN   ❌ CAP_NET_RAW    ❌ CAP_SYS_PTRACE      │
│  ❌ CAP_MKNOD       ❌ CAP_SYS_MODULE ❌ CAP_DAC_OVERRIDE    │
└────────────────────────────────────────────────────────────────┘
```

| Capability | Risk if Granted | Why Drop It |
|-----------|----------------|-------------|
| `CAP_SYS_ADMIN` | Mount filesystems, load BPF, trace processes | Essentially root; enables container escape |
| `CAP_NET_RAW` | Create raw sockets, craft arbitrary packets | ARP spoofing, network attacks |
| `CAP_SYS_PTRACE` | Trace and inspect other processes | Read secrets from other containers |
| `CAP_SYS_MODULE` | Load kernel modules | Complete kernel compromise |
| `CAP_MKNOD` | Create device nodes | Access to host devices |

### 8.9.2 Implementing Capability Dropping

```go
// security/capabilities.go — Linux capabilities management
package security

import (
"fmt"
"log"
"strings"
"unsafe"

"golang.org/x/sys/unix"
)

// capabilityMap maps capability names to their numeric values.
var capabilityMap = map[string]uintptr{
"CAP_CHOWN":            0,
"CAP_DAC_OVERRIDE":     1,
"CAP_DAC_READ_SEARCH":  2,
"CAP_FOWNER":           3,
"CAP_FSETID":           4,
"CAP_KILL":             5,
"CAP_SETGID":           6,
"CAP_SETUID":           7,
"CAP_SETPCAP":          8,
"CAP_NET_BIND_SERVICE": 10,
"CAP_NET_RAW":          13,
"CAP_SYS_CHROOT":       18,
"CAP_SYS_PTRACE":       19,
"CAP_SYS_ADMIN":        21,
"CAP_SYS_MODULE":       16,
"CAP_MKNOD":            27,
"CAP_AUDIT_WRITE":      29,
"CAP_SETFCAP":          31,
}

// DropCapabilities removes the specified capabilities from the current process.
func DropCapabilities(caps []string) error {
log.Printf("Dropping capabilities: %s", strings.Join(caps, ", "))

for _, name := range caps {
capNum, ok := capabilityMap[name]
if !ok {
return fmt.Errorf("unknown capability: %s", name)
}

// Drop from the bounding set — prevents regaining via exec
if err := unix.Prctl(unix.PR_CAPBSET_DROP, capNum, 0, 0, 0); err != nil {
return fmt.Errorf("dropping capability %s from bounding set: %w", name, err)
}
}

log.Printf("Capabilities dropped successfully")
return nil
}

// SetNoNewPrivileges prevents the process (and children) from gaining
// new privileges via setuid/setgid executables.
func SetNoNewPrivileges() error {
log.Printf("Setting no_new_privs")
if err := unix.Prctl(unix.PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0); err != nil {
return fmt.Errorf("setting no_new_privs: %w", err)
}
return nil
}

// capHeader is the Linux capability header structure.
type capHeader struct {
version uint32
pid     int32
}

// capData is the Linux capability data structure (per-set).
type capData struct {
effective   uint32
permitted   uint32
inheritable uint32
}

// GetCurrentCaps returns a human-readable list of current capabilities.
func GetCurrentCaps() ([]string, error) {
var hdr capHeader
hdr.version = 0x20080522 // _LINUX_CAPABILITY_VERSION_3
hdr.pid = 0              // current process

var data [2]capData

_, _, errno := unix.Syscall(
unix.SYS_CAPGET,
uintptr(unsafe.Pointer(&hdr)),
uintptr(unsafe.Pointer(&data[0])),
0,
)
if errno != 0 {
return nil, fmt.Errorf("capget: %w", errno)
}

var result []string
for name, num := range capabilityMap {
idx := num / 32
bit := uint32(1) << (num % 32)
if idx < 2 && data[idx].effective&bit != 0 {
result = append(result, name)
}
}

return result, nil
}
```

### 8.9.3 Seccomp BPF Filtering

**Seccomp** (Secure Computing) restricts which system calls a process can make. We use
seccomp-BPF (Berkeley Packet Filter) to define a whitelist of allowed syscalls.

```text
┌─────────────────────────────────────────────────┐
│          Process makes a system call             │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────┐
│           Seccomp BPF Filter                     │
│                                                  │
│   Is syscall in whitelist?                       │
│                                                  │
│   ┌─────────────────────────────────────┐       │
│   │ read, write, open, close, mmap,     │  YES  │
│   │ mprotect, brk, ioctl, access,       │──────▶│ ALLOW
│   │ execve, getpid, clone, wait4,       │       │
│   │ socket, connect, bind, listen, ...  │       │
│   └─────────────────────────────────────┘       │
│                                                  │
│   ┌─────────────────────────────────────┐       │
│   │ mount, umount2, pivot_root,         │  NO   │
│   │ reboot, kexec_load, init_module,    │──────▶│ KILL or ERRNO
│   │ delete_module, ptrace, ...          │       │
│   └─────────────────────────────────────┘       │
└─────────────────────────────────────────────────┘
```

```go
// security/seccomp.go — Seccomp BPF profile for containers
package security

import (
"log"

"golang.org/x/sys/unix"
)

// DefaultSeccompDenyList returns syscalls that should be blocked in containers.
// This is a simplified version—production runtimes use more comprehensive lists.
func DefaultSeccompDenyList() []string {
return []string{
"kexec_load",       // Load a new kernel
"kexec_file_load",  // Load a new kernel from file
"reboot",           // Reboot the system
"init_module",      // Load a kernel module
"finit_module",     // Load a kernel module from fd
"delete_module",    // Unload a kernel module
"acct",             // Process accounting
"swapon",           // Enable swap
"swapoff",          // Disable swap
"mount",            // Mount filesystem (after init)
"umount2",          // Unmount filesystem (after init)
"pivot_root",       // Change root filesystem (after init)
"keyctl",           // Kernel keyring manipulation
"add_key",          // Add key to keyring
"request_key",      // Request key from keyring
"open_by_handle_at", // Open file by handle (CVE-2015-1322)
}
}

// ApplySeccompProfile applies a basic seccomp profile using prctl.
// In production, you would use the SECCOMP_SET_MODE_FILTER with BPF.
func ApplySeccompProfile() error {
log.Printf("Applying seccomp profile")

// Set seccomp to strict mode as a baseline
// Note: A full implementation would use SECCOMP_SET_MODE_FILTER
// with a BPF program for fine-grained control.
//
// For a production-grade implementation, you would:
// 1. Build a BPF filter program
// 2. Load it via prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog)
//
// Here we demonstrate the concept with no_new_privs which is a
// prerequisite for seccomp filters:
if err := unix.Prctl(unix.PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0); err != nil {
return err
}

log.Printf("Seccomp baseline applied (no_new_privs set)")
return nil
}
```

### 8.9.4 Read-Only Root Filesystem

Making the root filesystem read-only prevents containers from modifying system files:

```go
// filesystem/readonly.go — Read-only root filesystem support
package filesystem

import (
"fmt"
"log"
"syscall"
)

// RemountReadOnly remounts the root filesystem as read-only.
// Writable directories (/tmp, /var/run) must be tmpfs mounts.
func RemountReadOnly() error {
log.Printf("Remounting root filesystem as read-only")

// First, set up writable tmpfs mounts for directories that need writes
writableDirs := []string{"/tmp", "/var/tmp", "/run"}
for _, dir := range writableDirs {
err := syscall.Mount("tmpfs", dir, "tmpfs",
syscall.MS_NOSUID|syscall.MS_NODEV, "size=65536k")
if err != nil {
log.Printf("WARNING: could not mount tmpfs on %s: %v", dir, err)
}
}

// Remount root as read-only
err := syscall.Mount("", "/", "", syscall.MS_REMOUNT|syscall.MS_RDONLY, "")
if err != nil {
return fmt.Errorf("remounting / as read-only: %w", err)
}

log.Printf("Root filesystem is now read-only")
return nil
}
```

### 8.9.5 Security Layers Summary

```text
┌─────────────────────────────────────────────────────────┐
│                  Defense in Depth                        │
│                                                         │
│  Layer 1: Namespaces                                    │
│  ├── PID: Container only sees its own processes         │
│  ├── NET: Isolated network stack                        │
│  ├── MNT: Separate filesystem tree                      │
│  ├── UTS: Own hostname                                  │
│  └── IPC: Isolated shared memory                        │
│                                                         │
│  Layer 2: Filesystem                                    │
│  ├── pivot_root: Cannot access host root                │
│  ├── OverlayFS: Base image is read-only                 │
│  └── MS_RDONLY: Root remounted read-only                │
│                                                         │
│  Layer 3: Cgroups                                       │
│  ├── CPU limits: Cannot monopolize CPU                  │
│  ├── Memory limits: OOM killed if exceeded              │
│  └── PID limits: Cannot fork-bomb                       │
│                                                         │
│  Layer 4: Capabilities                                  │
│  ├── Drop CAP_SYS_ADMIN: No mount, no BPF              │
│  ├── Drop CAP_NET_RAW: No raw sockets                  │
│  └── no_new_privs: Cannot escalate                     │
│                                                         │
│  Layer 5: Seccomp                                       │
│  ├── Block dangerous syscalls (reboot, modules)         │
│  └── Whitelist approach in production                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

> 🔬 **Deep Dive: Container Escapes**
>
> Known container escape techniques and our mitigations:
>
> | Attack Vector | Description | Mitigation |
> |--------------|-------------|------------|
> | `chroot` escape | Nested chroot + fd tricks | `pivot_root` + unmount old root |
> | `/proc` abuse | Access host info via `/proc` | New PID namespace, masked paths |
> | Raw socket | ARP spoofing on host network | Drop `CAP_NET_RAW` |
> | Kernel module | Load malicious module | Drop `CAP_SYS_MODULE`, seccomp |
> | Setuid binary | Escalate to root | `no_new_privs` flag |
> | Device access | Read host disks via `/dev` | Minimal `/dev` with tmpfs |
> | Fork bomb | Exhaust host PIDs | cgroups `pids.max` |
> | Memory bomb | Exhaust host memory | cgroups `memory.max` |


## 8.10 Comparison with Production Runtimes

Our gobox runtime implements the core mechanics of containerization. Let's see how it
compares to production runtimes and what additional features they provide.

### 8.10.1 Feature Comparison

| Feature | gobox (ours) | runc | crun | gVisor | Kata Containers |
|---------|:---:|:---:|:---:|:---:|:---:|
| PID namespace | ✅ | ✅ | ✅ | ✅ | ✅ |
| Network namespace | ✅ | ✅ | ✅ | ✅ | ✅ |
| Mount namespace | ✅ | ✅ | ✅ | ✅ | ✅ |
| OverlayFS | ✅ | ✅ | ✅ | ✅ | ✅ |
| pivot_root | ✅ | ✅ | ✅ | ✅ | N/A |
| Cgroups v2 | ✅ | ✅ | ✅ | ✅ | ✅ |
| Seccomp BPF | Basic | ✅ | ✅ | ✅ | N/A |
| User namespaces | ❌ | ✅ | ✅ | ✅ | N/A |
| OCI compliance | ❌ | ✅ | ✅ | ✅ | ✅ |
| Rootless mode | ❌ | ✅ | ✅ | ✅ | ❌ |
| Checkpoint/restore | ❌ | ✅ | ✅ | ❌ | ❌ |
| Kernel isolation | ❌ | ❌ | ❌ | ✅ | ✅ |
| Language | Go | Go | C | Go | Go |
| Binary size | ~5MB | ~10MB | ~300KB | ~150MB | Varies |
| Startup time | ~50ms | ~50ms | ~15ms | ~200ms | ~500ms |

### 8.10.2 Runtime Architecture Comparison

```text
┌───────────────────────────────────────────────────────────────┐
│  Traditional Container Runtime (runc)                         │
│                                                               │
│  Container Process ──────────────▶ Host Kernel                │
│  (syscalls go directly to host kernel)                        │
│                                                               │
│  Isolation: namespaces + cgroups + seccomp                    │
│  Attack surface: Full kernel syscall interface                │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│  gVisor (Sandbox Runtime)                                     │
│                                                               │
│  Container Process ──▶ Sentry (user-space kernel) ──▶ Host    │
│  (syscalls intercepted by gVisor's Sentry)                    │
│                                                               │
│  Isolation: Application kernel in user space                  │
│  Attack surface: Limited kernel interface (~70 syscalls)       │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│  Kata Containers (VM-based)                                   │
│                                                               │
│  Container Process ──▶ Guest Kernel ──▶ Hypervisor ──▶ Host   │
│  (runs inside a lightweight VM)                               │
│                                                               │
│  Isolation: Hardware virtualization (VT-x)                    │
│  Attack surface: Hypervisor interface only                    │
└───────────────────────────────────────────────────────────────┘
```

### 8.10.3 OCI Runtime Specification

Our gobox runtime uses a custom configuration format. Production runtimes conform to
the **OCI Runtime Specification**, which defines a standard container configuration
(`config.json`) and lifecycle API.

```text
OCI Runtime Spec config.json (simplified):

{
  "ociVersion": "1.0.2",
  "process": {
    "terminal": true,
    "user": { "uid": 0, "gid": 0 },
    "args": ["/bin/sh"],
    "env": ["PATH=/usr/bin:/bin", "TERM=xterm"],
    "cwd": "/",
    "capabilities": {
      "bounding": ["CAP_AUDIT_WRITE", "CAP_KILL", ...],
      "effective": ["CAP_AUDIT_WRITE", "CAP_KILL", ...],
      ...
    }
  },
  "root": {
    "path": "rootfs",
    "readonly": false
  },
  "mounts": [
    { "destination": "/proc", "type": "proc", "source": "proc" },
    { "destination": "/dev", "type": "tmpfs", "source": "tmpfs" },
    ...
  ],
  "linux": {
    "namespaces": [
      { "type": "pid" },
      { "type": "network" },
      { "type": "mount" },
      { "type": "uts" },
      { "type": "ipc" }
    ],
    "resources": {
      "memory": { "limit": 268435456 },
      "cpu": { "quota": 50000, "period": 100000 },
      "pids": { "limit": 64 }
    },
    "seccomp": { ... }
  }
}
```

Mapping our Config to OCI fields:

| Our Config Field | OCI Spec Field | Notes |
|-----------------|---------------|-------|
| `Config.Command` | `process.args` | Identical |
| `Config.Env` | `process.env` | Identical |
| `Config.RootFS` | `root.path` | OCI uses relative path |
| `Config.Hostname` | `hostname` | Identical |
| `Config.Resources.CPUQuota` | `linux.resources.cpu.quota/period` | OCI uses raw µs values |
| `Config.Resources.MemoryLimitBytes` | `linux.resources.memory.limit` | Identical |
| `Config.Security.DropCapabilities` | `process.capabilities` | OCI specifies per-set |

### 8.10.4 What We Did Not Implement

Our runtime demonstrates the core concepts but omits several production features:

1. **User namespaces** — Maps container UID 0 to an unprivileged host UID,
   enabling rootless containers. This is complex due to filesystem permission mapping.

2. **Image pulling** — Production runtimes pull OCI images from registries,
   unpack layers, and manage image storage. We use a pre-existing rootfs directory.

3. **Checkpoint/restore** — CRIU (Checkpoint/Restore In Userspace) can freeze a
   running container and resume it later, even on a different host.

4. **Hooks** — OCI spec defines prestart, createRuntime, createContainer,
   startContainer, poststart, and poststop hooks for extensibility.

5. **Terminal handling** — Proper PTY allocation and management for interactive
   containers with signal forwarding.

6. **Logging** — Structured logging of container stdout/stderr to files with
   log rotation.

> 💻 **Hands-On: Running gobox**
>
> To test the runtime end to end:
>
> ```text
> # Download an Alpine rootfs
> $ mkdir -p rootfs/alpine
> $ curl -O https://dl-cdn.alpinelinux.org/alpine/v3.19/releases/x86_64/\
>     alpine-minirootfs-3.19.0-x86_64.tar.gz
> $ tar xzf alpine-minirootfs-*.tar.gz -C rootfs/alpine
>
> # Build and run
> $ go build -o gobox .
> $ sudo ./gobox run --rootfs ./rootfs/alpine --cpu 0.5 --mem 128m /bin/sh
>
> # Inside the container:
> / # hostname
> gobox
> / # ps aux
> PID   USER     TIME  COMMAND
>     1 root      0:00 /bin/sh
>     2 root      0:00 ps aux
> / # cat /proc/1/cgroup
> 0::/gobox/gobox-123456
> / # ip addr show eth0
> 3: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
>     inet 10.10.10.2/24 scope global eth0
> ```


## Summary

In this chapter, we built a container runtime from scratch, implementing every layer
of the container stack:

| Component | Implementation | Key Syscalls/APIs |
|-----------|---------------|-------------------|
| **Filesystem** | OverlayFS + `pivot_root` | `mount(2)`, `pivot_root(2)` |
| **Process isolation** | Linux namespaces via `clone` flags | `clone(2)`, `unshare(2)` |
| **Resource limits** | Cgroups v2 filesystem interface | `write(2)` to cgroupfs |
| **Networking** | Veth pairs + Linux bridge + NAT | `ip(8)`, `iptables(8)` |
| **Security** | Capabilities + seccomp + read-only root | `prctl(2)`, `capset(2)` |
| **Lifecycle** | State machine with cleanup | `waitpid(2)`, `execve(2)` |

Key takeaways:

- **Containers are processes** — they are not virtual machines. Every container is a
  regular Linux process with additional kernel-enforced isolation boundaries.

- **Namespaces isolate visibility** — what a process can see (PIDs, network, mounts).

- **Cgroups limit resources** — how much a process can consume (CPU, memory, PIDs).

- **OverlayFS enables layers** — the copy-on-write filesystem model that makes
  container images shareable and efficient.

- **pivot_root > chroot** — `pivot_root` provides stronger isolation because it
  actually moves the mount point, while `chroot` only changes path resolution.

- **Defense in depth** — production containers use multiple overlapping security
  mechanisms: namespaces, cgroups, capabilities, seccomp, and filesystem hardening.

- **The re-exec pattern** — Go runtimes use `/proc/self/exe` to re-execute
  themselves as the container init process because Go's goroutine scheduler is
  incompatible with `fork(2)`.

In Chapter 9, we will see how production container ecosystems (OCI, containerd,
Kubernetes CRI) build on these primitives to create the robust, standardized
infrastructure used in production environments worldwide.

---

## Exercises

### Exercise 8.1: Resource Monitoring Dashboard

Build a monitoring tool that reads cgroup files to display real-time resource usage:

```text
┌─ Container gobox-123456 ─────────────────────────┐
│                                                   │
│  CPU:    ████████░░░░░░░░░░░░  42%  (0.5 cores)  │
│  Memory: ██████████████░░░░░░  68%  (174/256 MiB) │
│  PIDs:   ████░░░░░░░░░░░░░░░░  12/64              │
│                                                   │
│  Uptime: 3m 42s                                   │
│  State:  running                                  │
└───────────────────────────────────────────────────┘
```

Read from these cgroup files:
- `cpu.stat` for CPU usage
- `memory.current` / `memory.max` for memory
- `pids.current` / `pids.max` for PIDs

### Exercise 8.2: Container Exec

Implement a `gobox exec` command that runs a new process inside an already-running
container. You will need to:

1. Find the container's namespace files in `/proc/<pid>/ns/`
2. Use `setns(2)` to join each namespace
3. Use `execve(2)` to run the new command

This is how `docker exec` works.

### Exercise 8.3: Port Forwarding

Add a `--port` flag that sets up port forwarding from the host to the container:

```text
gobox run --rootfs ./rootfs/alpine --port 8080:80 nginx
```

Use `iptables` DNAT rules to forward `host:8080` → `container:80`.

### Exercise 8.4: Multi-Layer OverlayFS

Extend the filesystem module to support multiple lower layers, similar to how Docker
images have multiple layers:

```text
gobox run --layer ./layers/base --layer ./layers/python --layer ./layers/app /app/start
```

OverlayFS supports multiple lower directories separated by colons:
`lowerdir=layer3:layer2:layer1`

### Exercise 8.5: User Namespace Support

Add `--userns` flag to enable user namespace mapping:

1. Set `CLONE_NEWUSER` in clone flags
2. Write `/proc/<pid>/uid_map` to map container UID 0 to host UID 100000
3. Write `/proc/<pid>/gid_map` similarly
4. Write `deny` to `/proc/<pid>/setgroups`

This enables rootless containers where the container thinks it runs as root but
actually maps to an unprivileged host user.

### Exercise 8.6: Container Checkpoint

Implement basic checkpoint/restore using CRIU:

1. Install CRIU on the host
2. Use `criu dump` to checkpoint a running container
3. Use `criu restore` to resume it
4. Measure the checkpoint/restore time

### Exercise 8.7: Seccomp BPF Filter

Implement a proper seccomp BPF filter using `SECCOMP_SET_MODE_FILTER`:

1. Build a BPF program using `golang.org/x/net/bpf`
2. Whitelist approximately 300 safe syscalls
3. Return `SECCOMP_RET_ERRNO` for blocked calls
4. Test with a program that tries to call `reboot(2)`

### Exercise 8.8: Container Networking — Multiple Containers

Extend the networking to support multiple containers on the same bridge:

1. Launch three containers with IPs 10.10.10.2, 10.10.10.3, 10.10.10.4
2. Verify containers can ping each other
3. Verify all containers can reach the internet
4. Implement a simple round-robin load balancer using iptables

---

## Further Reading

- **"Containers from Scratch"** by Liz Rice — Conference talk and article that inspired
  many DIY container projects. Available on YouTube.

- **OCI Runtime Specification** — The official standard for container runtimes:
  https://github.com/opencontainers/runtime-spec

- **runc source code** — The reference OCI runtime implementation:
  https://github.com/opencontainers/runc

- **crun source code** — A fast OCI runtime written in C:
  https://github.com/containers/crun

- **"Container Security" by Liz Rice** (O'Reilly, 2020) — Comprehensive coverage of
  container security mechanisms including namespaces, cgroups, seccomp, and AppArmor.

- **Linux man pages** — Essential references:
  - `man 2 clone` — Process creation with namespace flags
  - `man 2 pivot_root` — Change the root filesystem
  - `man 2 mount` — Mount filesystems
  - `man 7 cgroups` — Control groups overview
  - `man 7 namespaces` — Namespace types and semantics
  - `man 2 seccomp` — Secure computing with BPF

- **cgroups v2 documentation** — Kernel documentation for the unified cgroup hierarchy:
  https://docs.kernel.org/admin-guide/cgroup-v2.html

- **gVisor documentation** — Google's application kernel for containers:
  https://gvisor.dev/docs/

- **Kata Containers architecture** — VM-based container isolation:
  https://katacontainers.io/docs/
