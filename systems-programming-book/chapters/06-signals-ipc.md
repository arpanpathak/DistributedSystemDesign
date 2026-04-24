# Chapter 6: Signals, IPC & Synchronization

> *"The art of programming is the art of organizing complexity -- of mastering multitude and avoiding its bastard chaos as effectively as possible."* -- Edsger Dijkstra

Every non-trivial system is composed of multiple cooperating processes. The
operating system provides a rich toolkit for these processes to communicate,
coordinate, and synchronize: **signals** for asynchronous notifications,
**pipes** for streaming data, **shared memory** for zero-copy communication,
**sockets** for structured message passing, and **synchronization primitives**
to keep it all coherent.

This chapter is the bridge between the OS kernel and Go's high-level
concurrency model. We will start with the oldest, crudest IPC mechanism
(signals) and build our way up to Go channels and atomics -- showing at every
step how the elegant Go abstractions map down to raw system calls.

---

## 6.1 Signals -- Asynchronous Notifications

### 6.1.1 What Signals Are

A **signal** is an asynchronous notification sent to a process (or a specific
thread within a process) by the kernel, by another process, or by the process
itself. Signals are the Unix equivalent of hardware interrupts -- but delivered
in software to userspace processes.

When a signal arrives:

1. The kernel sets a **pending bit** for that signal in the target process's
   task_struct.
2. When the process next returns from kernel mode to user mode (e.g., after a
   syscall or timer interrupt), the kernel checks for pending signals.
3. If the signal is not **blocked** (masked), the kernel diverts execution to
   the registered **signal handler** -- or takes the **default action** if no
   handler is installed.

Key properties of signals:

- **Asynchronous**: They arrive at unpredictable times.
- **Limited payload**: Standard signals carry almost no data (just the signal
  number). Real-time signals can carry a small integer or pointer.
- **Non-queuing** (standard signals): If the same signal arrives twice before
  being handled, it may be delivered only once.
- **Per-process or per-thread**: Signals can target a specific thread (via
  `tgkill()`) or the process as a whole (via `kill()`).

### 6.1.2 Signal Types -- The Complete Taxonomy

Every Unix system defines a set of standard signals. Here is the complete
reference for the signals every systems programmer must know:

#### Termination Signals

| Signal    | Number | Default Action | Description |
|-----------|--------|----------------|-------------|
| `SIGTERM` | 15     | Terminate      | Polite termination request. **Can be caught.** |
| `SIGKILL` | 9      | Terminate      | Forceful kill. **Cannot be caught, blocked, or ignored.** |
| `SIGINT`  | 2      | Terminate      | Interrupt from keyboard (Ctrl+C). |
| `SIGQUIT` | 3      | Core dump      | Quit from keyboard (Ctrl+\). Produces core dump. |
| `SIGHUP`  | 1      | Terminate      | Hangup -- terminal closed. Often used to trigger config reload. |

#### Job Control Signals

| Signal     | Number | Default Action | Description |
|------------|--------|----------------|-------------|
| `SIGSTOP`  | 19     | Stop           | Stop process. **Cannot be caught or ignored.** |
| `SIGCONT`  | 18     | Continue       | Resume a stopped process. |
| `SIGTSTP`  | 20     | Stop           | Stop from keyboard (Ctrl+Z). Can be caught. |
| `SIGTTIN`  | 21     | Stop           | Background process tried to read from terminal. |
| `SIGTTOU`  | 22     | Stop           | Background process tried to write to terminal. |

#### Fault Signals

| Signal    | Number | Default Action | Description |
|-----------|--------|----------------|-------------|
| `SIGSEGV` | 11     | Core dump      | Segmentation fault -- invalid memory access. |
| `SIGBUS`  | 7      | Core dump      | Bus error -- misaligned memory access. |
| `SIGFPE`  | 8      | Core dump      | Floating-point exception (includes integer div-by-zero). |
| `SIGILL`  | 4      | Core dump      | Illegal instruction. |
| `SIGABRT` | 6      | Core dump      | Abort -- sent by abort() function. |

#### User-Defined Signals

| Signal    | Number | Default Action | Description |
|-----------|--------|----------------|-------------|
| `SIGUSR1` | 10     | Terminate      | User-defined signal 1. |
| `SIGUSR2` | 12     | Terminate      | User-defined signal 2. |

#### Child and I/O Signals

| Signal     | Number | Default Action | Description |
|------------|--------|----------------|-------------|
| `SIGCHLD`  | 17     | Ignore         | Child process stopped or terminated. |
| `SIGPIPE`  | 13     | Terminate      | Write to pipe with no reader. |
| `SIGURG`   | 23     | Ignore         | Urgent data on socket (out-of-band). |
| `SIGIO`    | 29     | Terminate      | I/O now possible (async I/O notification). |

#### Timer and Alarm Signals

| Signal      | Number | Default Action | Description |
|-------------|--------|----------------|-------------|
| `SIGALRM`  | 14     | Terminate      | Alarm clock -- from alarm() or setitimer(). |
| `SIGVTALRM`| 26     | Terminate      | Virtual timer expired. |
| `SIGPROF`  | 27     | Terminate      | Profiling timer expired. |

### 6.1.3 Signal Delivery Mechanism in the Kernel

Understanding how the kernel actually delivers signals is crucial for writing
correct signal-handling code. Here is the detailed flow:

```
Process A calls kill(pid_B, SIGTERM)
         |
         v
+-------------------------------------+
|  Kernel: sys_kill()                  |
|  1. Find task_struct for pid_B       |
|  2. Permission check (same UID or   |
|     CAP_KILL capability)            |
|  3. Set bit 15 in pending.signal    |
|     bitmask of target task          |
|  4. If target is in                 |
|     TASK_INTERRUPTIBLE sleep,       |
|     wake it up                      |
|  5. If target is on another CPU,    |
|     send IPI to force reschedule    |
+-------------------------------------+
         |
         v
+-------------------------------------+
|  Process B: returning to userspace   |
|  (from syscall, interrupt, etc.)     |
|                                      |
|  Kernel checks: are there pending    |
|  signals not in the blocked mask?    |
|                                      |
|  YES -> dequeue signal               |
|  1. Save current registers to        |
|     signal stack frame on user stack |
|  2. Set instruction pointer to       |
|     signal handler address           |
|  3. Set up return trampoline         |
|     (calls rt_sigreturn)             |
|  4. Return to userspace -- now       |
|     executing signal handler         |
|                                      |
|  Handler returns -> rt_sigreturn()   |
|  -> kernel restores saved registers  |
|  -> process resumes normal execution |
+-------------------------------------+
```

**Critical details:**

- **Pending signals bitmask**: For standard signals (1-31), the kernel uses a
  single bit per signal. This means if SIGTERM is already pending and another
  SIGTERM arrives, it is **lost** -- there is no counter.

- **Signal masks**: Each thread has a `blocked` bitmask. Signals in this mask
  are held pending until unblocked. `SIGKILL` and `SIGSTOP` **cannot** be
  blocked.

- **Interrupted system calls**: When a signal arrives while a process is in a
  blocking syscall (like `read()`), the syscall either returns `EINTR` or is
  automatically restarted, depending on the `SA_RESTART` flag in `sigaction()`.

- **Thread targeting**: For process-directed signals, the kernel picks an
  arbitrary thread that does not have the signal blocked. For thread-directed
  signals (`tgkill()`), only the specified thread receives it.

### 6.1.4 Signal Handlers -- sigaction(), Signal Masks, Pending Signals

The modern way to install a signal handler in C is via `sigaction()`:

```c
/*
 * sigaction structure -- controls how a signal is handled.
 *
 * sa_handler:   Simple handler function, or SIG_IGN/SIG_DFL.
 * sa_sigaction: Extended handler receiving siginfo_t (used with SA_SIGINFO).
 * sa_mask:      Set of signals to block during handler execution.
 * sa_flags:     Behavior flags (SA_RESTART, SA_SIGINFO, SA_NOCLDSTOP, etc.).
 */
struct sigaction {
    void     (*sa_handler)(int);
    void     (*sa_sigaction)(int, siginfo_t *, void *);
    sigset_t   sa_mask;
    int        sa_flags;
};
```

**Key flags:**

- `SA_RESTART`: Automatically restart interrupted system calls.
- `SA_SIGINFO`: Use the three-argument handler form (`sa_sigaction`).
- `SA_NOCLDSTOP`: Don't generate `SIGCHLD` when children stop.
- `SA_RESETHAND`: Reset handler to `SIG_DFL` after one invocation.
- `SA_NODEFER`: Don't automatically block the signal during its own handler.

**Signal mask operations:**

```c
sigset_t set;
sigemptyset(&set);          /* Clear all signals from set */
sigaddset(&set, SIGTERM);   /* Add SIGTERM to set */
sigaddset(&set, SIGINT);    /* Add SIGINT to set */

/*
 * sigprocmask() -- change the thread's signal mask.
 *
 * SIG_BLOCK:   Add signals in 'set' to the blocked mask.
 * SIG_UNBLOCK: Remove signals in 'set' from the blocked mask.
 * SIG_SETMASK: Replace the blocked mask entirely with 'set'.
 *
 * The old mask is saved in 'oldset' (if non-NULL).
 */
sigprocmask(SIG_BLOCK, &set, &oldset);
```

### 6.1.5 Real-Time Signals vs Standard Signals

Linux supports **real-time signals** (SIGRTMIN through SIGRTMAX, typically
signals 34-64) that fix several limitations of standard signals:

| Property         | Standard Signals (1-31) | Real-Time Signals (34-64) |
|------------------|------------------------|---------------------------|
| Queuing          | No -- coalesced         | Yes -- each delivery queued |
| Ordering         | Undefined              | Lowest number first        |
| Payload          | None                   | sigval union (int or ptr)  |
| Default action   | Varies                 | Terminate                  |
| Number available | 31                     | ~30 (SIGRTMAX-SIGRTMIN+1)  |

Real-time signals are sent with `sigqueue()` instead of `kill()`:

```c
/*
 * sigqueue() -- send a real-time signal with a data payload.
 *
 * Unlike kill(), this guarantees queuing: if you send 5 signals,
 * the handler will be invoked 5 times. The sigval union carries
 * either an integer or a pointer as additional data.
 */
union sigval value;
value.sival_int = 42;
sigqueue(target_pid, SIGRTMIN + 3, value);
```

### 6.1.6 Go's Signal Handling: os/signal Package

Go's runtime takes over signal handling at startup. Here is what happens:

1. The Go runtime installs its own handlers for most signals using
   `sigaction()` with `SA_SIGINFO | SA_RESTART | SA_ONSTACK`.
2. `SIGSEGV`, `SIGBUS`, `SIGFPE`, and `SIGILL` are used internally to
   implement goroutine preemption (since Go 1.14) and null-pointer panics.
3. User code interacts with signals through the `os/signal` package, which
   uses **channels** as the delivery mechanism.

The core API:

```go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
)

// demonstrateSignalAPI shows the core os/signal API surface.
//
// Key functions:
//   - signal.Notify(c, sigs...)  -- route specified signals to channel c.
//   - signal.Stop(c)             -- stop routing signals to channel c.
//   - signal.Reset(sigs...)      -- reset specified signals to default behavior.
//   - signal.Ignore(sigs...)     -- ignore specified signals entirely.
//   - signal.Ignored(sig)        -- check if a signal is being ignored.
//
// The channel MUST be buffered. If the channel is full when a signal
// arrives, the signal is dropped (not queued). A buffer size of 1 is
// typical for signals you respond to immediately.
func demonstrateSignalAPI() {
	// Create a buffered channel for signal delivery.
	// Buffer size of 1 ensures we don't miss a signal that arrives
	// while we're processing the previous one.
	sigChan := make(chan os.Signal, 1)

	// Register to receive SIGTERM and SIGINT on this channel.
	// Internally, this tells the Go runtime's signal handler to
	// forward these signals to sigChan instead of taking the
	// default action (process termination).
	signal.Notify(sigChan, syscall.SIGTERM, syscall.SIGINT)

	// Block until a signal arrives.
	sig := <-sigChan
	fmt.Printf("Received signal: %v\n", sig)

	// Clean up: stop forwarding signals to this channel.
	signal.Stop(sigChan)
}
```

**Important Go-specific signal behaviors:**

- `SIGPIPE`: Go ignores SIGPIPE by default. Writing to a closed pipe/socket
  returns an error instead of killing the process.
- `SIGHUP`, `SIGINT`, `SIGTERM`: If no `signal.Notify` is registered, these
  cause the process to exit.
- `SIGQUIT`: By default, causes a goroutine dump (stack trace of all
  goroutines) -- incredibly useful for debugging.
- `SIGURG`: Used internally by the Go runtime for goroutine preemption since
  Go 1.14. Do not catch this signal.

### 6.1.7 SIGTERM vs SIGKILL -- Why SIGKILL Can't Be Caught

This is a fundamental design decision in Unix:

```
SIGTERM (signal 15):
  +-- Can be caught by a signal handler
  +-- Can be blocked (masked)
  +-- Can be ignored
  +-- Allows cleanup: close files, flush buffers, release locks
  +-- The "polite" way to terminate a process

SIGKILL (signal 9):
  +-- CANNOT be caught -- no handler is invoked
  +-- CANNOT be blocked -- the blocked mask is bypassed
  +-- CANNOT be ignored -- SIG_IGN is not honored
  +-- Kernel terminates the process immediately
  +-- Open files are closed by the kernel (not the process)
  +-- Shared memory segments are NOT automatically cleaned up
  +-- The "last resort" for stuck processes
```

**Why can't SIGKILL be caught?**

The kernel needs a guaranteed mechanism to terminate any process. If SIGKILL
could be caught, a malicious or buggy process could install a handler that
ignores it -- making the process unkillable. The same logic applies to SIGSTOP:
the system must be able to freeze any process for debugging or job control.

**The correct shutdown sequence:**

```
1. Send SIGTERM -> give the process time to clean up (5-30 seconds)
2. If still running, send SIGKILL -> force termination
```

This is exactly what `docker stop` does: SIGTERM, wait 10 seconds (configurable
via `--time`), then SIGKILL. Kubernetes follows the same pattern with its
`terminationGracePeriodSeconds`.

### 6.1.8 Hands-On: Graceful Shutdown Handler in Go

This is one of the most important patterns in production Go services. A
graceful shutdown handler ensures that in-flight HTTP requests complete,
database connections close cleanly, and background workers finish their
current task.

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

// GracefulServer wraps an HTTP server with signal-aware shutdown logic.
//
// Architecture:
//   - Listens for SIGTERM and SIGINT via os/signal.
//   - On receiving either signal, initiates a graceful shutdown sequence:
//     1. Stop accepting new connections.
//     2. Wait for in-flight requests to complete (up to shutdownTimeout).
//     3. Run registered cleanup functions.
//     4. Exit cleanly.
//
// This pattern is essential for:
//   - Kubernetes pods (SIGTERM sent during rolling updates).
//   - Docker containers (SIGTERM from docker stop).
//   - systemd services (SIGTERM from systemctl stop).
//   - Any long-running service that must not lose in-flight work.
type GracefulServer struct {
	// httpServer is the underlying HTTP server.
	httpServer *http.Server

	// shutdownTimeout is the maximum time to wait for in-flight
	// requests to complete during shutdown. After this duration,
	// remaining connections are forcibly closed.
	shutdownTimeout time.Duration

	// cleanupFuncs holds functions to run during shutdown, in order.
	// These might close database connections, flush caches, release
	// distributed locks, etc.
	cleanupFuncs []func() error

	// mu protects cleanupFuncs from concurrent modification.
	mu sync.Mutex
}

// NewGracefulServer creates a new server with the given address, handler,
// and shutdown timeout.
//
// The shutdownTimeout controls how long we wait for in-flight requests
// to drain. For most web services, 15-30 seconds is appropriate. For
// services handling long-running uploads or WebSocket connections, you
// may need 60+ seconds.
func NewGracefulServer(addr string, handler http.Handler, timeout time.Duration) *GracefulServer {
	return &GracefulServer{
		httpServer: &http.Server{
			Addr:    addr,
			Handler: handler,
		},
		shutdownTimeout: timeout,
	}
}

// OnShutdown registers a cleanup function to be called during graceful
// shutdown. Functions are called in the order they were registered.
//
// Each function should be idempotent -- it may be called even if the
// resource was never fully initialized (e.g., if startup was interrupted).
func (gs *GracefulServer) OnShutdown(fn func() error) {
	gs.mu.Lock()
	defer gs.mu.Unlock()
	gs.cleanupFuncs = append(gs.cleanupFuncs, fn)
}

// ListenAndServeWithGracefulShutdown starts the HTTP server and blocks
// until a termination signal is received, then performs graceful shutdown.
//
// Signal handling flow:
//  1. We create a buffered channel and register for SIGTERM and SIGINT.
//  2. The HTTP server starts in a goroutine.
//  3. The main goroutine blocks on the signal channel.
//  4. When a signal arrives, we call httpServer.Shutdown() which:
//     - Closes the listening socket (no new connections).
//     - Waits for active requests to complete.
//     - Returns when all connections are idle or context is cancelled.
//  5. We run cleanup functions.
//  6. We return, allowing the process to exit.
func (gs *GracefulServer) ListenAndServeWithGracefulShutdown() error {
	// Step 1: Set up signal handling.
	// Listen for SIGTERM (process managers) and SIGINT (Ctrl+C).
	// Buffer size of 1: only need to know "a signal arrived".
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGTERM, syscall.SIGINT)

	// Step 2: Start the HTTP server in a goroutine.
	serverErr := make(chan error, 1)
	go func() {
		log.Printf("Server starting on %s", gs.httpServer.Addr)
		if err := gs.httpServer.ListenAndServe(); err != http.ErrServerClosed {
			serverErr <- err
		}
		close(serverErr)
	}()

	// Step 3: Wait for a signal or server error.
	select {
	case sig := <-sigChan:
		log.Printf("Received signal %v -- initiating graceful shutdown...", sig)
	case err := <-serverErr:
		return fmt.Errorf("server error: %w", err)
	}

	// Step 4: Graceful HTTP shutdown with timeout.
	ctx, cancel := context.WithTimeout(context.Background(), gs.shutdownTimeout)
	defer cancel()

	log.Printf("Waiting up to %v for in-flight requests to complete...", gs.shutdownTimeout)
	if err := gs.httpServer.Shutdown(ctx); err != nil {
		log.Printf("HTTP shutdown error: %v", err)
	}

	// Step 5: Run cleanup functions.
	log.Println("Running cleanup functions...")
	for i, fn := range gs.cleanupFuncs {
		if err := fn(); err != nil {
			log.Printf("Cleanup function %d failed: %v", i, err)
		}
	}

	log.Println("Graceful shutdown complete.")
	return nil
}

func main() {
	// Create a handler that simulates slow requests.
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		duration := 2 * time.Second
		log.Printf("Handling request %s (will take %v)", r.URL.Path, duration)

		select {
		case <-time.After(duration):
			fmt.Fprintf(w, "Request completed after %v\n", duration)
		case <-r.Context().Done():
			log.Printf("Request %s cancelled: %v", r.URL.Path, r.Context().Err())
			return
		}
	})

	server := NewGracefulServer(":8080", mux, 15*time.Second)

	// Register cleanup handlers.
	server.OnShutdown(func() error {
		log.Println("Closing database connection pool...")
		time.Sleep(100 * time.Millisecond)
		return nil
	})
	server.OnShutdown(func() error {
		log.Println("Flushing metrics buffer...")
		return nil
	})

	if err := server.ListenAndServeWithGracefulShutdown(); err != nil {
		log.Fatalf("Server failed: %v", err)
	}
}
```

**Testing the graceful shutdown:**

```bash
# Terminal 1: Start the server
$ go run main.go
2024/01/15 10:00:00 Server starting on :8080

# Terminal 2: Send a request, then quickly SIGTERM the server
$ curl http://localhost:8080/slow &
$ kill -SIGTERM $(pgrep -f "go run main.go")

# Terminal 1 output:
# Received signal terminated -- initiating graceful shutdown...
# Waiting up to 15s for in-flight requests to complete...
# Handling request /slow (will take 2s)
# Closing database connection pool...
# Flushing metrics buffer...
# Graceful shutdown complete.
```

### 6.1.9 Hands-On: Using SIGUSR1 for Live Config Reload in Go

Many production daemons (nginx, HAProxy, Prometheus) use SIGHUP or SIGUSR1 to
trigger configuration reload without restart. This is a zero-downtime operation.

```go
package main

import (
	"encoding/json"
	"log"
	"os"
	"os/signal"
	"sync"
	"sync/atomic"
	"syscall"
	"time"
)

// Config represents the application configuration that can be
// reloaded at runtime without restarting the process.
//
// We use atomic.Value to store the config so that readers never
// block -- they always get a consistent snapshot of the config.
// This is a classic lock-free pattern for read-heavy workloads.
type Config struct {
	// ListenAddr is the address the server binds to.
	ListenAddr string `json:"listen_addr"`

	// MaxConnections limits the number of concurrent connections.
	MaxConnections int `json:"max_connections"`

	// LogLevel controls verbosity: "debug", "info", "warn", "error".
	LogLevel string `json:"log_level"`

	// RateLimit is the maximum requests per second per client.
	RateLimit float64 `json:"rate_limit"`

	// FeatureFlags controls feature toggles that can be flipped
	// without a restart.
	FeatureFlags map[string]bool `json:"feature_flags"`
}

// ConfigManager handles loading, storing, and hot-reloading configuration.
//
// Design decisions:
//   - Uses atomic.Value for the config pointer so readers never lock.
//   - Validates new config before swapping (don't apply broken config).
//   - Notifies subscribers when config changes (observer pattern).
//   - Thread-safe: multiple goroutines can read config concurrently
//     while a reload happens in the background.
type ConfigManager struct {
	// configPath is the filesystem path to the JSON config file.
	configPath string

	// current holds the current config as an atomic.Value.
	// We store *Config in it. Readers call current.Load().(*Config).
	current atomic.Value

	// subscribers are notified when config changes.
	// The channel receives the new config after a successful reload.
	subscribers []chan<- *Config

	// mu protects subscribers slice during registration.
	mu sync.Mutex

	// reloadCount tracks how many successful reloads have occurred.
	// Useful for metrics and debugging.
	reloadCount atomic.Int64
}

// NewConfigManager creates a new config manager and loads the initial
// configuration from the given path.
//
// Returns an error if the initial config cannot be loaded -- we require
// a valid config at startup. Subsequent reload failures are logged but
// do not crash the process (we keep the last good config).
func NewConfigManager(path string) (*ConfigManager, error) {
	cm := &ConfigManager{configPath: path}

	// Load the initial configuration.
	cfg, err := cm.loadFromDisk()
	if err != nil {
		return nil, err
	}
	cm.current.Store(cfg)
	log.Printf("Initial config loaded: %+v", cfg)

	return cm, nil
}

// Get returns the current configuration. This is safe to call from
// any goroutine at any time -- it never blocks.
//
// The returned pointer should be treated as read-only. Never modify
// the config through this pointer; always reload from disk.
func (cm *ConfigManager) Get() *Config {
	return cm.current.Load().(*Config)
}

// Subscribe registers a channel that will receive the new config
// after each successful reload. The channel should be buffered to
// avoid blocking the reload goroutine.
func (cm *ConfigManager) Subscribe(ch chan<- *Config) {
	cm.mu.Lock()
	defer cm.mu.Unlock()
	cm.subscribers = append(cm.subscribers, ch)
}

// Reload reads the config file from disk, validates it, and atomically
// swaps it into the current config. Notifies all subscribers on success.
//
// If the new config is invalid, the old config is preserved and an
// error is returned. This is critical: a typo in the config file
// should never take down a running service.
func (cm *ConfigManager) Reload() error {
	newCfg, err := cm.loadFromDisk()
	if err != nil {
		return err
	}

	// Validate the new config before applying.
	if err := cm.validate(newCfg); err != nil {
		return err
	}

	// Atomically swap the config. After this line, all new calls to
	// Get() will return newCfg. In-flight operations using the old
	// config pointer continue to work -- they hold a reference to
	// the old Config struct, which won't be garbage collected until
	// they're done with it.
	oldCfg := cm.current.Load().(*Config)
	cm.current.Store(newCfg)
	cm.reloadCount.Add(1)

	log.Printf("Config reloaded (version %d): %s -> %s",
		cm.reloadCount.Load(), oldCfg.LogLevel, newCfg.LogLevel)

	// Notify subscribers (non-blocking send).
	cm.mu.Lock()
	subs := make([]chan<- *Config, len(cm.subscribers))
	copy(subs, cm.subscribers)
	cm.mu.Unlock()

	for _, ch := range subs {
		select {
		case ch <- newCfg:
		default:
			log.Println("Warning: subscriber channel full, skipping notification")
		}
	}

	return nil
}

// loadFromDisk reads and parses the JSON config file.
func (cm *ConfigManager) loadFromDisk() (*Config, error) {
	data, err := os.ReadFile(cm.configPath)
	if err != nil {
		return nil, err
	}

	var cfg Config
	if err := json.Unmarshal(data, &cfg); err != nil {
		return nil, err
	}
	return &cfg, nil
}

// validate checks that the config is sane. Returns an error if any
// field has an invalid value.
func (cm *ConfigManager) validate(cfg *Config) error {
	if cfg.MaxConnections <= 0 {
		return fmt.Errorf("max_connections must be positive, got %d", cfg.MaxConnections)
	}
	if cfg.RateLimit < 0 {
		return fmt.Errorf("rate_limit must be non-negative, got %f", cfg.RateLimit)
	}
	return nil
}

// WatchSignals starts a goroutine that listens for SIGUSR1 and triggers
// a config reload. This is the standard Unix pattern for live reloading.
//
// Usage:
//
//	cm.WatchSignals()
//	// Now: kill -USR1 <pid> triggers a reload
func (cm *ConfigManager) WatchSignals() {
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGUSR1)

	go func() {
		for sig := range sigChan {
			log.Printf("Received %v -- reloading configuration...", sig)
			if err := cm.Reload(); err != nil {
				log.Printf("Config reload failed: %v (keeping current config)", err)
			}
		}
	}()

	log.Printf("Watching for SIGUSR1 (pid=%d) to trigger config reload", os.Getpid())
}

func main() {
	cm, err := NewConfigManager("/etc/myapp/config.json")
	if err != nil {
		log.Fatalf("Failed to load config: %v", err)
	}

	// Start watching for reload signals.
	cm.WatchSignals()

	// Subscribe to config changes.
	configUpdates := make(chan *Config, 1)
	cm.Subscribe(configUpdates)

	// Main application loop.
	go func() {
		for newCfg := range configUpdates {
			log.Printf("Config updated! New log level: %s", newCfg.LogLevel)
			// Reconfigure logging, rate limiters, etc.
		}
	}()

	// Simulate the application running.
	for {
		cfg := cm.Get()
		log.Printf("Tick: log_level=%s, max_conns=%d", cfg.LogLevel, cfg.MaxConnections)
		time.Sleep(5 * time.Second)
	}
}
```

**Note**: The above code references `fmt` in `validate()` -- add `"fmt"` to imports.

**Testing the live reload:**

```bash
# Write initial config
$ cat > /etc/myapp/config.json << 'EOF'
{
  "listen_addr": ":8080",
  "max_connections": 100,
  "log_level": "info",
  "rate_limit": 1000.0,
  "feature_flags": {"new_ui": false}
}
EOF

# Start the application
$ go run main.go &
# Output: Initial config loaded: ...
# Output: Watching for SIGUSR1 (pid=12345) to trigger config reload

# Edit the config
$ sed -i 's/"info"/"debug"/' /etc/myapp/config.json

# Send SIGUSR1 to trigger reload
$ kill -USR1 12345
# Output: Received user defined signal 1 -- reloading configuration...
# Output: Config reloaded (version 1): info -> debug
# Output: Config updated! New log level: debug
```

### 6.1.10 Hands-On: Proper Signal Handling in a Container (PID 1 Behavior)

When a process runs as PID 1 inside a container, signal handling changes
dramatically. This is one of the most common sources of container shutdown
issues.

**The PID 1 problem:**

In a normal Linux system, PID 1 is `init` (or `systemd`). PID 1 has special
signal handling behavior:

1. **Signals without handlers are ignored**: If PID 1 has not installed a
   handler for SIGTERM, the kernel simply **ignores** the signal. This is
   unlike any other process, where unhandled SIGTERM causes termination.

2. **SIGKILL still works**: Even PID 1 can be killed with SIGKILL.

3. **Zombie reaping**: PID 1 is responsible for reaping orphaned zombie
   processes (calling `wait()` on them).

This means if your Go application runs as PID 1 in a Docker container and
you don't call `signal.Notify()`, `docker stop` will hang for 10 seconds
(waiting for graceful shutdown that never happens) and then SIGKILL.

```go
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

// ContainerServer is a PID-1-aware HTTP server that properly handles
// signals inside containers.
//
// PID 1 considerations:
//  1. MUST register signal handlers -- unhandled signals are ignored
//     for PID 1 (the kernel's special treatment of init).
//  2. MUST reap zombie children -- if this process spawns children
//     (directly or via exec), their zombies won't be cleaned up
//     by anyone else.
//  3. SHOULD propagate signals to child processes -- so the entire
//     process tree shuts down gracefully.
//
// Alternative: Use a proper init like tini or dumb-init as PID 1,
// and run your app as PID 2+. But understanding the problem is essential.
type ContainerServer struct {
	server *http.Server
}

// reapZombies starts a goroutine that continuously reaps zombie
// child processes.
//
// When any child process exits, the kernel sends SIGCHLD to the parent.
// If the parent doesn't call wait(), the child becomes a zombie --
// it's dead but its entry in the process table lingers.
//
// For PID 1, this is our responsibility. We call wait4() in a loop
// with WNOHANG to collect all available zombies without blocking.
func (cs *ContainerServer) reapZombies() {
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGCHLD)

	go func() {
		for range sigChan {
			// Reap all available zombie children (non-blocking).
			for {
				var status syscall.WaitStatus
				pid, err := syscall.Wait4(-1, &status, syscall.WNOHANG, nil)
				if pid <= 0 || err != nil {
					break // No more zombies to reap.
				}
				log.Printf("Reaped zombie child pid=%d, status=%d", pid, status.ExitStatus())
			}
		}
	}()
}

// Run starts the server and handles PID-1-appropriate shutdown.
//
// The shutdown flow for a container:
//  1. Container runtime sends SIGTERM (docker stop, kubectl delete pod).
//  2. We catch SIGTERM and initiate graceful shutdown.
//  3. We stop accepting new connections and drain existing ones.
//  4. If shutdown completes within the timeout, we exit 0.
//  5. If not, the container runtime sends SIGKILL after its grace period.
func (cs *ContainerServer) Run() error {
	// Warn if we are PID 1 -- this is important information.
	if os.Getpid() == 1 {
		log.Println("WARNING: Running as PID 1 inside a container.")
		log.Println("Signal handling is active. Zombie reaping is enabled.")
		cs.reapZombies()
	}

	// Register for termination signals. This is CRITICAL for PID 1.
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGTERM, syscall.SIGINT)

	// Start HTTP server.
	go func() {
		log.Printf("Starting server on %s (pid=%d)", cs.server.Addr, os.Getpid())
		if err := cs.server.ListenAndServe(); err != http.ErrServerClosed {
			log.Fatalf("Server error: %v", err)
		}
	}()

	// Wait for shutdown signal.
	sig := <-sigChan
	log.Printf("Received %v -- shutting down (pid=%d)", sig, os.Getpid())

	// Use a shorter timeout than the container's grace period.
	// Docker default is 10s, K8s default is 30s.
	// We use 8s to leave 2s buffer for cleanup.
	ctx, cancel := context.WithTimeout(context.Background(), 8*time.Second)
	defer cancel()

	return cs.server.Shutdown(ctx)
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("OK"))
	})

	cs := &ContainerServer{
		server: &http.Server{Addr: ":8080", Handler: mux},
	}

	if err := cs.Run(); err != nil {
		log.Fatalf("Shutdown error: %v", err)
	}
	log.Println("Clean shutdown complete.")
}
```

**Dockerfile best practice:**

```dockerfile
# BAD: Shell form -- /bin/sh is PID 1, your app is PID 2.
# SIGTERM goes to shell, not your app!
CMD go run main.go

# GOOD: Exec form -- your app is PID 1 directly.
CMD ["./myapp"]

# ALSO GOOD: Use tini as a lightweight init.
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["./myapp"]
```

---

## 6.2 Pipes -- Streaming Data Between Processes

### 6.2.1 Anonymous Pipes -- pipe() Syscall

Pipes are the oldest IPC mechanism in Unix, dating back to 1973. An anonymous
pipe is a unidirectional data channel with two file descriptors: one for
reading and one for writing.

```c
/*
 * pipe() -- create an anonymous pipe.
 *
 * Creates a pair of file descriptors:
 *   pipefd[0] -- read end  (data comes OUT of this end)
 *   pipefd[1] -- write end (data goes INTO this end)
 *
 * Data written to pipefd[1] can be read from pipefd[0].
 * The pipe has a kernel buffer (typically 64KB on Linux since 2.6.11).
 *
 * Key behaviors:
 *   - Writing to a pipe with no readers: SIGPIPE + EPIPE error.
 *   - Reading from a pipe with no writers: returns 0 (EOF).
 *   - Writing more than PIPE_BUF bytes is not guaranteed atomic.
 *   - PIPE_BUF is 4096 bytes on Linux.
 */
int pipefd[2];
pipe(pipefd);  /* or pipe2(pipefd, O_CLOEXEC | O_NONBLOCK) */
```

**Pipe internals in the Linux kernel:**

```
+-------------------+      Kernel Buffer       +-------------------+
| Writer Process    |  =====================>  | Reader Process    |
| write(pipefd[1])  |  [ ring buffer, 64KB ]   | read(pipefd[0])   |
+-------------------+                          +-------------------+

The kernel pipe buffer is implemented as a ring of page-sized segments
(struct pipe_buffer). Each segment holds a reference to a page frame.

Key kernel structures:
  - struct pipe_inode_info: The pipe object itself.
    - bufs: Array of pipe_buffer segments.
    - nrbufs: Number of buffers currently containing data.
    - readers/writers: Reference counts for each end.
    - wait: Wait queue for blocking read/write operations.
  - struct pipe_buffer: One segment of the pipe.
    - page: Pointer to the page frame holding data.
    - offset: Start offset within the page.
    - len: Bytes of valid data in this buffer.
```

### 6.2.2 Named Pipes (FIFOs) -- mkfifo

Named pipes (FIFOs) are like anonymous pipes but exist as filesystem entries.
This allows unrelated processes to communicate.

```bash
# Create a named pipe
$ mkfifo /tmp/myfifo

# ls shows it with a 'p' type
$ ls -la /tmp/myfifo
prw-r--r-- 1 user user 0 Jan 15 10:00 /tmp/myfifo

# Writer (blocks until a reader opens the other end)
$ echo "hello" > /tmp/myfifo

# Reader (in another terminal)
$ cat /tmp/myfifo
hello
```

**Named pipe vs anonymous pipe:**

| Property          | Anonymous Pipe         | Named Pipe (FIFO)     |
|-------------------|------------------------|-----------------------|
| Filesystem entry  | No                     | Yes (visible in ls)   |
| Related processes | Must share via fork()  | Any process can open  |
| Lifetime          | Until all FDs closed   | Until unlinked        |
| Creation          | pipe() syscall         | mkfifo() / mknod()   |
| Directionality    | Unidirectional         | Unidirectional        |

### 6.2.3 Pipe Buffer Size and PIPE_BUF Atomicity

The `PIPE_BUF` constant (4096 bytes on Linux) is critical for correctness:

- **Writes <= PIPE_BUF bytes are atomic**: If two processes write to the same
  pipe simultaneously, and each write is <= PIPE_BUF bytes, the writes will
  not be interleaved. Each write appears as a contiguous chunk.

- **Writes > PIPE_BUF bytes are NOT atomic**: The kernel may interleave data
  from multiple writers. This means if two processes each write 8KB to the
  same pipe, the reader might see data alternating between the two writers.

```c
/*
 * The atomicity guarantee in practice:
 *
 * Process A writes 4000 bytes (< PIPE_BUF = 4096):
 *   -> All 4000 bytes appear contiguously in the pipe.
 *
 * Process B writes 4000 bytes (< PIPE_BUF = 4096):
 *   -> All 4000 bytes appear contiguously in the pipe.
 *
 * Reader sees: [AAAA...4000...AAAA][BBBB...4000...BBBB]
 *          or: [BBBB...4000...BBBB][AAAA...4000...AAAA]
 *   but NEVER: [AABB...interleaved...ABBA]
 *
 * However, if each writes 8000 bytes (> PIPE_BUF):
 * Reader might see: [AAAA...BBBB...AAAA...BBBB]  (interleaved!)
 */
```

You can check and modify the pipe buffer size:

```bash
# Check default pipe size (usually 65536 = 64KB)
$ cat /proc/sys/fs/pipe-max-size
1048576

# Change pipe size programmatically
$ python3 -c "import fcntl; f=open('/dev/stdin'); print(fcntl.fcntl(f, 1032))"
65536  # F_GETPIPE_SZ = 1032
```

### 6.2.4 Hands-On: Building a Pipeline Executor in Go

This example builds a shell-like pipeline executor that can chain multiple
commands together, like `ls -la | grep .go | wc -l`:

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"log"
	"os"
	"os/exec"
	"strings"
)

// PipelineStage represents a single command in a pipeline.
//
// Each stage has:
//   - A command name and arguments.
//   - An stdin (connected to previous stage's stdout, or os.Stdin).
//   - An stdout (connected to next stage's stdin, or os.Stdout).
//   - A stderr (always connected to os.Stderr for error visibility).
type PipelineStage struct {
	// Name is the command to execute (e.g., "grep", "sort", "wc").
	Name string

	// Args are the command arguments (e.g., ["-la"], [".go"], ["-l"]).
	Args []string
}

// Pipeline represents a chain of commands connected by pipes.
//
// Architecture:
//   - Each adjacent pair of stages is connected by an os.Pipe().
//   - The first stage reads from the pipeline's input (default: os.Stdin).
//   - The last stage writes to the pipeline's output (default: os.Stdout).
//   - All stages run concurrently -- this is essential for pipelines
//     to work, because a stage may block on write until the next stage
//     reads from the pipe.
//
// Implementation mirrors how a shell (bash, zsh) implements pipelines:
//  1. Create N-1 pipes for N stages.
//  2. Fork/exec each stage with appropriate stdin/stdout wiring.
//  3. Close unused pipe ends in each process.
//  4. Wait for all processes to complete.
type Pipeline struct {
	// stages are the commands in the pipeline, in order.
	stages []PipelineStage

	// input is where the first stage reads from.
	// Default is os.Stdin if nil.
	input io.Reader

	// output is where the last stage writes to.
	// Default is os.Stdout if nil.
	output io.Writer
}

// NewPipeline creates a pipeline from a shell-like command string.
//
// The input string is split on "|" to separate stages.
// Each stage is split on whitespace to get the command and args.
//
// Example: "ls -la | grep .go | wc -l"
// Produces 3 stages: [ls -la], [grep .go], [wc -l]
func NewPipeline(cmdString string) *Pipeline {
	parts := strings.Split(cmdString, "|")
	stages := make([]PipelineStage, 0, len(parts))

	for _, part := range parts {
		fields := strings.Fields(strings.TrimSpace(part))
		if len(fields) == 0 {
			continue
		}
		stages = append(stages, PipelineStage{
			Name: fields[0],
			Args: fields[1:],
		})
	}

	return &Pipeline{stages: stages}
}

// SetInput sets the input reader for the first stage.
func (p *Pipeline) SetInput(r io.Reader) {
	p.input = r
}

// SetOutput sets the output writer for the last stage.
func (p *Pipeline) SetOutput(w io.Writer) {
	p.output = w
}

// Execute runs the entire pipeline and waits for completion.
//
// Execution flow:
//  1. Create os.Pipe() between each pair of adjacent stages.
//  2. Configure each exec.Cmd with the correct stdin/stdout.
//  3. Start all commands concurrently.
//  4. Close our copies of the pipe file descriptors (critical!
//     if we don't close them, readers will never see EOF).
//  5. Wait for all commands to finish.
//  6. Return the first error encountered, or nil.
//
// This correctly handles backpressure: if a downstream stage is slow,
// the pipe buffer fills up, and upstream stages block on write().
// This is the fundamental flow control mechanism of Unix pipelines.
func (p *Pipeline) Execute() error {
	if len(p.stages) == 0 {
		return fmt.Errorf("empty pipeline")
	}

	// Create all the commands.
	cmds := make([]*exec.Cmd, len(p.stages))
	for i, stage := range p.stages {
		cmds[i] = exec.Command(stage.Name, stage.Args...)
		cmds[i].Stderr = os.Stderr // Always show errors.
	}

	// Wire up pipes between adjacent stages.
	// For N stages, we need N-1 pipes.
	for i := 0; i < len(cmds)-1; i++ {
		// Create a pipe connecting stage i's stdout to stage i+1's stdin.
		reader, writer, err := os.Pipe()
		if err != nil {
			return fmt.Errorf("creating pipe %d: %w", i, err)
		}
		cmds[i].Stdout = writer
		cmds[i+1].Stdin = reader

		// We must close our copies of the pipe ends after starting
		// the commands. Otherwise:
		// - If we keep the write end open, the reader will never see
		//   EOF (because there's still a writer -- us).
		// - If we keep the read end open, the writer might not get
		//   SIGPIPE when it should.
		defer writer.Close()
		defer reader.Close()
	}

	// Wire up the first stage's input.
	if p.input != nil {
		cmds[0].Stdin = p.input
	} else {
		cmds[0].Stdin = os.Stdin
	}

	// Wire up the last stage's output.
	if p.output != nil {
		cmds[len(cmds)-1].Stdout = p.output
	} else {
		cmds[len(cmds)-1].Stdout = os.Stdout
	}

	// Start all commands. They must all be started before we wait
	// on any of them, because the pipeline needs all stages running
	// concurrently for data to flow.
	for i, cmd := range cmds {
		if err := cmd.Start(); err != nil {
			return fmt.Errorf("starting stage %d (%s): %w", i, cmd.Path, err)
		}
	}

	// Close pipe ends that belong to the child processes.
	// This is CRITICAL. Without this, the reader stages will never
	// see EOF because we (the parent) still hold the write end open.
	// (The deferred closes above handle this.)

	// Wait for all commands to complete.
	var firstErr error
	for i, cmd := range cmds {
		if err := cmd.Wait(); err != nil && firstErr == nil {
			firstErr = fmt.Errorf("stage %d (%s) failed: %w", i, p.stages[i].Name, err)
		}
	}

	return firstErr
}

func main() {
	// Example 1: Simple pipeline
	fmt.Println("=== Pipeline: ls -la | grep .go | head -5 ===")
	p1 := NewPipeline("ls -la | grep .go | head -5")
	if err := p1.Execute(); err != nil {
		log.Printf("Pipeline 1 error: %v", err)
	}

	// Example 2: Pipeline with custom input
	fmt.Println("\n=== Pipeline with custom input ===")
	p2 := NewPipeline("sort | uniq -c | sort -rn")
	p2.SetInput(strings.NewReader("banana\napple\nbanana\ncherry\napple\napple\n"))
	var buf bytes.Buffer
	p2.SetOutput(&buf)
	if err := p2.Execute(); err != nil {
		log.Printf("Pipeline 2 error: %v", err)
	}
	fmt.Print(buf.String())

	// Example 3: Process substitution
	fmt.Println("\n=== Pipeline: find . -name '*.go' | xargs wc -l | tail -1 ===")
	p3 := NewPipeline("find . -name *.go | xargs wc -l | tail -1")
	if err := p3.Execute(); err != nil {
		log.Printf("Pipeline 3 error: %v", err)
	}
}
```

### 6.2.5 Hands-On: Named Pipe IPC Between Two Go Processes

```go
// ============================================================
// named_pipe_writer.go -- Writes structured messages to a FIFO.
// ============================================================
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"os"
	"syscall"
	"time"
)

// Message represents a structured IPC message sent over the named pipe.
//
// We use JSON encoding for simplicity and human readability.
// For production use, consider Protocol Buffers or MessagePack
// for better performance and smaller wire size.
type Message struct {
	// Timestamp when the message was created.
	Timestamp time.Time `json:"timestamp"`

	// Type categorizes the message (e.g., "metric", "event", "command").
	Type string `json:"type"`

	// Payload carries the message data.
	Payload string `json:"payload"`

	// Sequence is a monotonically increasing counter for ordering
	// and detecting lost messages.
	Sequence uint64 `json:"sequence"`
}

func main() {
	fifoPath := "/tmp/myapp-fifo"

	// Create the FIFO if it doesn't exist.
	// syscall.Mkfifo creates a named pipe in the filesystem.
	// The permission 0666 allows any user to read/write.
	// In production, use more restrictive permissions.
	if err := syscall.Mkfifo(fifoPath, 0666); err != nil {
		// EEXIST is OK -- the FIFO already exists.
		if !os.IsExist(err) {
			log.Fatalf("Failed to create FIFO: %v", err)
		}
	}
	log.Printf("FIFO created at %s", fifoPath)

	// Open the FIFO for writing. This BLOCKS until a reader opens
	// the other end. This is the default behavior of FIFOs.
	// To avoid blocking, use O_NONBLOCK (returns ENXIO if no reader).
	log.Println("Opening FIFO for writing (waiting for reader)...")
	f, err := os.OpenFile(fifoPath, os.O_WRONLY, 0)
	if err != nil {
		log.Fatalf("Failed to open FIFO: %v", err)
	}
	defer f.Close()
	log.Println("Reader connected!")

	// Send messages with JSON encoding, one per line.
	encoder := json.NewEncoder(f)
	for seq := uint64(0); ; seq++ {
		msg := Message{
			Timestamp: time.Now(),
			Type:      "metric",
			Payload:   fmt.Sprintf("cpu_usage=%.1f%%", 42.0+float64(seq%20)),
			Sequence:  seq,
		}

		if err := encoder.Encode(msg); err != nil {
			log.Printf("Write error (reader disconnected?): %v", err)
			return
		}
		log.Printf("Sent message #%d", seq)
		time.Sleep(time.Second)
	}
}
```

```go
// ============================================================
// named_pipe_reader.go -- Reads structured messages from a FIFO.
// ============================================================
package main

import (
	"encoding/json"
	"log"
	"os"
)

// Message must match the writer's struct definition.
type Message struct {
	Timestamp string `json:"timestamp"`
	Type      string `json:"type"`
	Payload   string `json:"payload"`
	Sequence  uint64 `json:"sequence"`
}

func main() {
	fifoPath := "/tmp/myapp-fifo"

	// Open the FIFO for reading. Blocks until a writer opens the other end.
	log.Printf("Opening FIFO %s for reading (waiting for writer)...", fifoPath)
	f, err := os.Open(fifoPath)
	if err != nil {
		log.Fatalf("Failed to open FIFO: %v", err)
	}
	defer f.Close()
	log.Println("Writer connected!")

	// Decode JSON messages, one per line.
	// json.NewDecoder handles buffering and newline-delimited JSON.
	decoder := json.NewDecoder(f)
	var lastSeq uint64

	for {
		var msg Message
		if err := decoder.Decode(&msg); err != nil {
			log.Printf("Read error (writer disconnected?): %v", err)
			return
		}

		// Detect gaps in sequence numbers (lost messages).
		if msg.Sequence > 0 && msg.Sequence != lastSeq+1 {
			log.Printf("WARNING: Gap detected! Expected seq %d, got %d (%d messages lost)",
				lastSeq+1, msg.Sequence, msg.Sequence-lastSeq-1)
		}
		lastSeq = msg.Sequence

		log.Printf("Received #%d [%s]: %s", msg.Sequence, msg.Type, msg.Payload)
	}
}
```

---

## 6.3 Unix Domain Sockets

### 6.3.1 Stream vs Datagram UDS

Unix Domain Sockets (UDS) are the most versatile IPC mechanism on Unix systems.
Unlike pipes, they support **bidirectional** communication and can transmit
both **stream** and **datagram** data.

```
+------------------+        +------------------+
| Process A        |        | Process B        |
|                  |  UDS   |                  |
| connect() -------|------->| accept()         |
| send()    ------>|------->| recv()           |
| recv()   <-------|<-------| send()           |
+------------------+        +------------------+
      Socket file: /var/run/myapp.sock
```

**Stream (SOCK_STREAM) vs Datagram (SOCK_DGRAM):**

| Property           | Stream (SOCK_STREAM) | Datagram (SOCK_DGRAM) |
|--------------------|----------------------|-----------------------|
| Connection          | Connection-oriented  | Connectionless        |
| Message boundaries  | No (byte stream)     | Yes (per-message)     |
| Ordering           | Guaranteed            | Guaranteed (UDS only) |
| Reliability        | Guaranteed            | Guaranteed (UDS only) |
| Max message size   | Unlimited             | ~208KB (varies)       |
| Backpressure       | TCP-like flow control | Drops if buffer full  |
| Use case           | RPC, HTTP proxying    | Short messages, logs  |

**Note**: Unlike UDP over IP, Unix domain datagram sockets **do** guarantee
ordering and reliability -- they never drop, reorder, or duplicate messages
(unless the receive buffer is full).

### 6.3.2 Abstract Namespace Sockets

Linux supports a special **abstract namespace** for Unix domain sockets that
doesn't require a filesystem path:

```go
// Abstract namespace sockets start with a null byte (\x00).
// They don't appear in the filesystem and are automatically
// cleaned up when all references are closed.
//
// Advantages:
//   - No need to manage socket file cleanup (no stale .sock files).
//   - No filesystem permissions to worry about.
//   - Faster (no filesystem lookup).
//
// Disadvantage:
//   - Linux-only (not available on macOS or BSDs).
//   - No filesystem permissions for access control.
addr := &net.UnixAddr{
	Name: "\x00myapp-abstract-socket",
	Net:  "unix",
}

```

### 6.3.3 Passing File Descriptors Over Unix Sockets (SCM_RIGHTS)

One of the most powerful features of Unix domain sockets is the ability to
**pass open file descriptors** between unrelated processes. This uses the
`SCM_RIGHTS` control message mechanism.

**How it works internally:**

1. Process A has an open file descriptor (e.g., fd 5 pointing to a TCP socket).
2. Process A sends fd 5 over a Unix socket using `sendmsg()` with `SCM_RIGHTS`.
3. The kernel **duplicates** the file descriptor into Process B's fd table.
4. Process B receives a **new fd number** (e.g., fd 8) pointing to the **same**
   underlying kernel file object.
5. Both processes can now independently read/write the same socket/file.

```
Process A (fd 5 -> TCP socket)     Process B (fd 8 -> same TCP socket)
        |                                    ^
        | sendmsg(SCM_RIGHTS, fd=5)          |
        +---> Unix Domain Socket ----->------+
              Kernel duplicates the file
              description (struct file *)
              into Process B's fd table
```

**Use cases:**

- **Socket activation**: systemd opens listening sockets and passes them to
  services (this is how `ListenFDs` / `sd_listen_fds()` works).
- **Privilege separation**: A privileged process opens a raw socket or file,
  then passes the fd to an unprivileged worker.
- **Zero-downtime restarts**: Pass listening sockets from old process to new.
- **Container runtimes**: Pass network sockets between namespaces.

### 6.3.4 Hands-On: Building a UDS-Based RPC Server in Go

```go
package main

import (
	"encoding/gob"
	"fmt"
	"log"
	"net"
	"os"
	"os/signal"
	"syscall"
	"time"
)

// RPCRequest represents a request from client to server.
//
// We use Go's gob encoding for simplicity. In production, you would
// likely use gRPC (which can run over Unix sockets), Protocol Buffers,
// or a custom binary protocol for better performance.
type RPCRequest struct {
	// Method is the RPC method to invoke (e.g., "GetStatus", "SetConfig").
	Method string

	// Args holds the method arguments as key-value pairs.
	Args map[string]string

	// RequestID is a unique identifier for request/response correlation.
	RequestID uint64
}

// RPCResponse is the server's reply to an RPCRequest.
type RPCResponse struct {
	// RequestID echoes back the request's ID for correlation.
	RequestID uint64

	// Result holds the return value on success.
	Result string

	// Error is non-empty if the RPC call failed.
	Error string

	// DurationMs is how long the server took to process the request.
	DurationMs float64
}

// RPCServer listens on a Unix domain socket and processes RPC requests.
//
// Architecture:
//   - Uses SOCK_STREAM (connection-oriented) for reliable delivery.
//   - Each client connection is handled in a separate goroutine.
//   - The socket file is cleaned up on shutdown to prevent stale sockets.
//   - Methods are dispatched via a map for easy extensibility.
type RPCServer struct {
	// socketPath is the filesystem path for the Unix socket.
	socketPath string

	// listener is the Unix socket listener.
	listener net.Listener

	// methods maps method names to handler functions.
	methods map[string]func(map[string]string) (string, error)
}

// NewRPCServer creates a new RPC server listening on the given socket path.
//
// The socket file is created with permissions 0700 (owner only).
// For multi-user access, adjust permissions or use abstract sockets.
func NewRPCServer(socketPath string) (*RPCServer, error) {
	// Remove stale socket file from a previous run.
	// If the file doesn't exist, os.Remove returns an error we ignore.
	os.Remove(socketPath)

	// Listen on the Unix domain socket.
	listener, err := net.Listen("unix", socketPath)
	if err != nil {
		return nil, fmt.Errorf("listen on %s: %w", socketPath, err)
	}

	// Set socket permissions.
	if err := os.Chmod(socketPath, 0700); err != nil {
		listener.Close()
		return nil, fmt.Errorf("chmod %s: %w", socketPath, err)
	}

	server := &RPCServer{
		socketPath: socketPath,
		listener:   listener,
		methods:    make(map[string]func(map[string]string) (string, error)),
	}

	return server, nil
}

// RegisterMethod adds an RPC method handler.
//
// The handler function receives the request args and returns either
// a result string or an error. The server wraps this in an RPCResponse.
func (s *RPCServer) RegisterMethod(name string, handler func(map[string]string) (string, error)) {
	s.methods[name] = handler
}

// Serve starts accepting connections and processing requests.
//
// Each connection is handled in a separate goroutine. The server
// continues running until the listener is closed (e.g., by Shutdown).
func (s *RPCServer) Serve() error {
	log.Printf("RPC server listening on %s", s.socketPath)

	for {
		conn, err := s.listener.Accept()
		if err != nil {
			// Check if this is a "use of closed network connection" error,
			// which means Shutdown() was called.
			if opErr, ok := err.(*net.OpError); ok && opErr.Err.Error() == "use of closed network connection" {
				return nil // Clean shutdown.
			}
			log.Printf("Accept error: %v", err)
			continue
		}

		// Handle each connection in its own goroutine.
		go s.handleConnection(conn)
	}
}

// handleConnection processes all RPC requests on a single connection.
//
// A single connection can carry multiple sequential requests (like
// HTTP keep-alive). The connection is closed when the client disconnects
// or an encoding error occurs.
func (s *RPCServer) handleConnection(conn net.Conn) {
	defer conn.Close()
	log.Printf("Client connected: %v", conn.RemoteAddr())

	decoder := gob.NewDecoder(conn)
	encoder := gob.NewEncoder(conn)

	for {
		var req RPCRequest
		if err := decoder.Decode(&req); err != nil {
			log.Printf("Client disconnected: %v", err)
			return
		}

		// Dispatch the method.
		start := time.Now()
		var resp RPCResponse
		resp.RequestID = req.RequestID

		handler, exists := s.methods[req.Method]
		if !exists {
			resp.Error = fmt.Sprintf("unknown method: %s", req.Method)
		} else {
			result, err := handler(req.Args)
			if err != nil {
				resp.Error = err.Error()
			} else {
				resp.Result = result
			}
		}
		resp.DurationMs = float64(time.Since(start).Microseconds()) / 1000.0

		if err := encoder.Encode(resp); err != nil {
			log.Printf("Encode error: %v", err)
			return
		}
	}
}

// Shutdown gracefully stops the server.
func (s *RPCServer) Shutdown() {
	s.listener.Close()
	os.Remove(s.socketPath)
	log.Println("Server shut down, socket removed.")
}

func main() {
	server, err := NewRPCServer("/tmp/myapp-rpc.sock")
	if err != nil {
		log.Fatalf("Failed to create server: %v", err)
	}

	// Register methods.
	server.RegisterMethod("GetStatus", func(args map[string]string) (string, error) {
		return fmt.Sprintf("uptime=%v, goroutines=running", time.Since(time.Now())), nil
	})

	server.RegisterMethod("Echo", func(args map[string]string) (string, error) {
		msg, ok := args["message"]
		if !ok {
			return "", fmt.Errorf("missing 'message' argument")
		}
		return msg, nil
	})

	// Graceful shutdown on SIGTERM/SIGINT.
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGTERM, syscall.SIGINT)
	go func() {
		<-sigChan
		server.Shutdown()
	}()

	// Start serving.
	if err := server.Serve(); err != nil {
		log.Fatalf("Server error: %v", err)
	}
}
```

### 6.3.5 Hands-On: Passing File Descriptors Between Processes in Go

```go
package main

import (
	"fmt"
	"log"
	"net"
	"os"
	"syscall"
)

// sendFD sends an open file descriptor over a Unix domain socket
// using the SCM_RIGHTS control message.
//
// This is the Go equivalent of sendmsg() with cmsg containing SCM_RIGHTS.
// The net package wraps this via UnixConn.WriteMsgUnix().
//
// Parameters:
//   - conn: The Unix domain socket connection to send over.
//   - fd: The file descriptor number to send.
//
// How it works internally:
//  1. We construct an "out-of-band" (OOB) message containing the fd.
//  2. The kernel's sendmsg() implementation sees SCM_RIGHTS in the cmsg.
//  3. The kernel looks up the struct file * for our fd.
//  4. The kernel creates a new fd in the receiving process's table
//     pointing to the same struct file *.
//  5. The receiving process gets a (potentially different) fd number.
func sendFD(conn *net.UnixConn, fd int) error {
	// Build the SCM_RIGHTS control message.
	// syscall.UnixRights() constructs the cmsg header + payload.
	rights := syscall.UnixRights(fd)

	// WriteMsgUnix sends both regular data and out-of-band (control) data.
	// We send a dummy byte as the regular data because some systems
	// require at least 1 byte of regular data with a control message.
	_, _, err := conn.WriteMsgUnix([]byte{0}, rights, nil)
	return err
}

// recvFD receives a file descriptor from a Unix domain socket.
//
// Returns the received file descriptor number in the current process's
// fd table. This fd number will typically be different from the sender's
// fd number, but both point to the same kernel file object.
func recvFD(conn *net.UnixConn) (int, error) {
	// Allocate buffer for the control message.
	// 24 bytes is enough for one fd (cmsg header + one int32).
	buf := make([]byte, 1)
	oob := make([]byte, 24)

	_, oobn, _, _, err := conn.ReadMsgUnix(buf, oob)
	if err != nil {
		return -1, err
	}

	// Parse the control message to extract the fd.
	msgs, err := syscall.ParseSocketControlMessage(oob[:oobn])
	if err != nil {
		return -1, err
	}

	for _, msg := range msgs {
		fds, err := syscall.ParseUnixRights(&msg)
		if err != nil {
			continue
		}
		if len(fds) > 0 {
			return fds[0], nil
		}
	}

	return -1, fmt.Errorf("no file descriptor received")
}

// demonstrateFDPassing shows the complete flow of passing a file
// descriptor between a "parent" and "child" goroutine using a
// Unix domain socket pair.
//
// In real-world usage, the two ends would be in different processes.
// We use goroutines here for demonstration simplicity.
func demonstrateFDPassing() {
	// Create a Unix domain socket pair.
	// socketpair() creates two connected sockets -- no filesystem path needed.
	fds, err := syscall.Socketpair(syscall.AF_UNIX, syscall.SOCK_STREAM, 0)
	if err != nil {
		log.Fatalf("socketpair: %v", err)
	}

	// Convert raw fds to net.UnixConn for easier use.
	parentFile := os.NewFile(uintptr(fds[0]), "parent")
	childFile := os.NewFile(uintptr(fds[1]), "child")

	parentConn, err := net.FileConn(parentFile)
	if err != nil {
		log.Fatalf("FileConn parent: %v", err)
	}
	parentFile.Close()

	childConn, err := net.FileConn(childFile)
	if err != nil {
		log.Fatalf("FileConn child: %v", err)
	}
	childFile.Close()

	parentUDS := parentConn.(*net.UnixConn)
	childUDS := childConn.(*net.UnixConn)

	done := make(chan struct{})

	// Child goroutine: receives the fd and reads from it.
	go func() {
		defer close(done)
		defer childUDS.Close()

		fd, err := recvFD(childUDS)
		if err != nil {
			log.Fatalf("recvFD: %v", err)
		}
		log.Printf("Child: received fd %d", fd)

		// Use the received fd to read from the file.
		file := os.NewFile(uintptr(fd), "received-file")
		defer file.Close()

		buf := make([]byte, 1024)
		n, err := file.Read(buf)
		if err != nil {
			log.Printf("Child: read error: %v", err)
			return
		}
		log.Printf("Child: read %d bytes: %s", n, string(buf[:n]))
	}()

	// Parent: open a file and send its fd to the child.
	file, err := os.Open("/etc/hostname")
	if err != nil {
		log.Fatalf("open: %v", err)
	}
	log.Printf("Parent: opened /etc/hostname as fd %d", file.Fd())

	if err := sendFD(parentUDS, int(file.Fd())); err != nil {
		log.Fatalf("sendFD: %v", err)
	}
	log.Println("Parent: fd sent successfully")
	file.Close() // Parent can close its copy now.
	parentUDS.Close()

	<-done // Wait for child to finish.
}

func main() {
	demonstrateFDPassing()
}
```

---

## 6.4 Shared Memory

### 6.4.1 System V Shared Memory

System V shared memory is the oldest shared memory API. While POSIX shared
memory is preferred for new code, System V is still widely used and you will
encounter it in legacy systems.

```
+------------------+                    +------------------+
| Process A        |                    | Process B        |
|                  |   Shared Memory    |                  |
| shmat() -------->| [kernel page(s)] |<-------- shmat()  |
| ptr = 0x7f...   |  Same physical     | ptr = 0x7f...    |
|                  |  pages mapped into |  (different      |
| *ptr = 42       |  both address      |   virtual addr)  |
|                  |  spaces            | val = *ptr (=42) |
+------------------+                    +------------------+
```

**System V shared memory API:**

```c
/*
 * shmget() -- create or access a shared memory segment.
 *
 * key:    A unique identifier (from ftok() or IPC_PRIVATE).
 * size:   Size in bytes (rounded up to page size).
 * shmflg: IPC_CREAT | IPC_EXCL | permissions (e.g., 0666).
 *
 * Returns a shmid (shared memory identifier), not a file descriptor.
 * This ID is system-wide -- any process with the same key can access it.
 */
int shmid = shmget(key, size, IPC_CREAT | 0666);

/*
 * shmat() -- attach the shared memory segment to the process's address space.
 *
 * shmid:   The ID returned by shmget().
 * shmaddr: Preferred address (usually NULL to let the kernel choose).
 * shmflg:  SHM_RDONLY for read-only access, 0 for read-write.
 *
 * Returns a pointer to the mapped memory. This is a real pointer you
 * can dereference. The kernel has mapped the same physical pages into
 * your address space.
 */
void *ptr = shmat(shmid, NULL, 0);

/*
 * shmdt() -- detach the shared memory segment.
 * Does NOT destroy the segment -- it persists until explicitly removed.
 */
shmdt(ptr);

/*
 * shmctl() -- control operations on the segment.
 * IPC_RMID: Mark the segment for destruction (destroyed when last
 *           process detaches).
 * IPC_STAT: Get segment info (size, permissions, timestamps).
 */
shmctl(shmid, IPC_RMID, NULL);
```

**Danger**: System V shared memory segments persist even after all processes
exit! They must be explicitly removed with `shmctl(IPC_RMID)` or the `ipcrm`
command. Use `ipcs -m` to list all shared memory segments.

### 6.4.2 POSIX Shared Memory

POSIX shared memory is the modern, cleaner API. It uses **file descriptors**
instead of IPC IDs, which means it works with all fd-based tools (poll, epoll,
etc.) and follows Unix's "everything is a file" philosophy.

```c
/*
 * shm_open() -- create/open a POSIX shared memory object.
 *
 * name:  Name of the shared memory object (must start with /).
 *        Lives in /dev/shm/ on Linux.
 * oflag: O_CREAT | O_RDWR, etc.
 * mode:  Permissions (e.g., 0666).
 *
 * Returns a file descriptor. Use ftruncate() to set the size,
 * then mmap() to map it into your address space.
 */
int fd = shm_open("/my-shared-mem", O_CREAT | O_RDWR, 0666);
ftruncate(fd, 4096);  /* Set size to 4KB */
void *ptr = mmap(NULL, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

/* Use the memory... */
((int *)ptr)[0] = 42;

/* Cleanup */
munmap(ptr, 4096);
close(fd);
shm_unlink("/my-shared-mem");  /* Remove the named object */
```

**POSIX vs System V shared memory:**

| Property          | System V (shmget)        | POSIX (shm_open)       |
|-------------------|--------------------------|------------------------|
| Namespace         | IPC key (integer)        | Name (string, /dev/shm)|
| Handle type       | shmid (integer)          | File descriptor        |
| Size setting      | At creation (shmget)     | ftruncate() after open |
| Permissions       | At creation              | Standard file perms    |
| Cleanup           | shmctl(IPC_RMID)         | shm_unlink()           |
| Persistence       | Survives process exit    | Survives until unlink  |
| mmap compatible   | No (uses shmat)          | Yes                    |
| Listing           | ipcs -m                  | ls /dev/shm/           |

### 6.4.3 Hands-On: Shared Memory IPC Between Two Go Processes Using mmap

```go
// ============================================================
// shm_writer.go -- Writes data to a shared memory region using mmap.
// ============================================================
package main

import (
	"encoding/binary"
	"fmt"
	"log"
	"os"
	"syscall"
	"time"
	"unsafe"
)

// SharedHeader is the structure at the beginning of the shared memory
// region. It provides synchronization metadata so the reader knows
// when new data is available.
//
// Memory layout (64 bytes):
//
//	Offset 0:  Version   (uint64) -- schema version for compatibility.
//	Offset 8:  Sequence  (uint64) -- incremented on each write.
//	Offset 16: Timestamp (int64)  -- Unix nanoseconds of last write.
//	Offset 24: DataLen   (uint32) -- length of the data payload.
//	Offset 28: Padding   (36 bytes)
//	Offset 64: Data      (variable length)
//
// The reader polls the Sequence field. When it changes, new data is
// available. This is a simple (but effective) form of lock-free
// communication for single-writer, single-reader scenarios.
type SharedHeader struct {
	Version   uint64
	Sequence  uint64
	Timestamp int64
	DataLen   uint32
	_padding  [36]byte // Pad to 64 bytes for cache line alignment.
}

const (
	shmPath    = "/dev/shm/myapp-shared"
	shmSize    = 4096 // 4KB shared memory region.
	headerSize = 64   // Size of SharedHeader.
	dataOffset = 64   // Offset where data payload begins.
	version    = 1    // Schema version.
)

func main() {
	// Create the shared memory file.
	// On Linux, /dev/shm is a tmpfs -- writes go to RAM, not disk.
	f, err := os.OpenFile(shmPath, os.O_CREATE|os.O_RDWR|os.O_TRUNC, 0666)
	if err != nil {
		log.Fatalf("Failed to create shared memory file: %v", err)
	}

	// Set the file size. mmap requires the file to be at least as
	// large as the mapping size.
	if err := f.Truncate(shmSize); err != nil {
		log.Fatalf("Failed to truncate: %v", err)
	}

	// mmap the file into our address space.
	// MAP_SHARED: Changes are visible to other processes mapping the same file.
	// PROT_READ|PROT_WRITE: We need both read and write access.
	data, err := syscall.Mmap(
		int(f.Fd()),
		0,
		shmSize,
		syscall.PROT_READ|syscall.PROT_WRITE,
		syscall.MAP_SHARED,
	)
	if err != nil {
		log.Fatalf("mmap failed: %v", err)
	}
	defer syscall.Munmap(data)
	f.Close() // File can be closed after mmap -- mapping persists.

	log.Printf("Shared memory writer started. File: %s, Size: %d", shmPath, shmSize)

	// Write data in a loop.
	for seq := uint64(1); ; seq++ {
		// Prepare the payload.
		payload := fmt.Sprintf("Hello from writer! seq=%d time=%s", seq, time.Now().Format(time.RFC3339Nano))
		payloadBytes := []byte(payload)

		// Write the header.
		// We write the data FIRST, then update the sequence number.
		// This ensures the reader sees consistent data when it
		// observes a new sequence number.
		//
		// In a real system, you would use atomic stores and memory
		// barriers. This simplified version works for demonstration
		// because Go's memory model provides sufficient ordering
		// guarantees within a single goroutine.
		copy(data[dataOffset:], payloadBytes)

		// Write header fields.
		binary.LittleEndian.PutUint64(data[0:8], version)
		binary.LittleEndian.PutUint64(data[8:16], seq)
		binary.LittleEndian.PutUint64(data[16:24], uint64(time.Now().UnixNano()))
		binary.LittleEndian.PutUint32(data[24:28], uint32(len(payloadBytes)))

		// Memory barrier: ensure all writes are visible before the
		// reader checks the sequence number.
		// In Go, we can use an atomic store for this purpose.
		// For simplicity, we rely on the sequential consistency of
		// the single-writer pattern here.

		if seq%10 == 0 {
			log.Printf("Written %d messages", seq)
		}
		time.Sleep(100 * time.Millisecond)
	}
}

// Note: For production use, replace the polling-based synchronization
// with proper atomic operations:
//
//	import "sync/atomic"
//	atomic.StoreUint64((*uint64)(unsafe.Pointer(&data[8])), seq)
//
// And on the reader side:
//
//	seq := atomic.LoadUint64((*uint64)(unsafe.Pointer(&data[8])))
//
// The unsafe import above is referenced to show the concept.
var _ = unsafe.Pointer(nil)
```

```go
// ============================================================
// shm_reader.go -- Reads data from a shared memory region using mmap.
// ============================================================
package main

import (
	"encoding/binary"
	"log"
	"os"
	"syscall"
	"time"
)

const (
	shmPath    = "/dev/shm/myapp-shared"
	shmSize    = 4096
	dataOffset = 64
)

func main() {
	// Open the existing shared memory file (read-only).
	f, err := os.Open(shmPath)
	if err != nil {
		log.Fatalf("Failed to open shared memory: %v (is the writer running?)", err)
	}

	// mmap with PROT_READ only -- we're a reader.
	data, err := syscall.Mmap(
		int(f.Fd()),
		0,
		shmSize,
		syscall.PROT_READ,
		syscall.MAP_SHARED,
	)
	if err != nil {
		log.Fatalf("mmap failed: %v", err)
	}
	defer syscall.Munmap(data)
	f.Close()

	log.Printf("Shared memory reader started. Polling %s", shmPath)

	var lastSeq uint64
	for {
		// Read the current sequence number.
		seq := binary.LittleEndian.Uint64(data[8:16])

		if seq != lastSeq && seq > 0 {
			// New data available! Read the payload.
			ts := int64(binary.LittleEndian.Uint64(data[16:24]))
			dataLen := binary.LittleEndian.Uint32(data[24:28])

			payload := make([]byte, dataLen)
			copy(payload, data[dataOffset:dataOffset+int(dataLen)])

			latency := time.Since(time.Unix(0, ts))
			log.Printf("Seq %d (latency: %v): %s", seq, latency, string(payload))

			lastSeq = seq
		}

		// Poll interval. In production, use futex or eventfd for
		// notification instead of polling.
		time.Sleep(50 * time.Millisecond)
	}
}
```

---

## 6.5 Message Queues

### 6.5.1 System V Message Queues

System V message queues provide **typed, queued messages** between processes.
Unlike pipes, each message has a **type** field, allowing receivers to
selectively dequeue messages by type.

```c
/*
 * msgget() -- create or access a message queue.
 *
 * key:    Unique identifier (from ftok() or IPC_PRIVATE).
 * msgflg: IPC_CREAT | permissions.
 *
 * Returns a message queue ID (msqid).
 */
int msqid = msgget(key, IPC_CREAT | 0666);

/*
 * msgsnd() -- send a message to the queue.
 *
 * The message structure must start with a long mtype field.
 * The mtype acts as a "channel" -- receivers can filter by it.
 *
 * msgflg: IPC_NOWAIT to return immediately if queue is full.
 *         0 to block until space is available.
 */
struct msgbuf {
    long mtype;        /* Message type (must be > 0) */
    char mtext[256];   /* Message payload */
};

struct msgbuf msg = {.mtype = 1, .mtext = "Hello!"};
msgsnd(msqid, &msg, strlen(msg.mtext), 0);

/*
 * msgrcv() -- receive a message from the queue.
 *
 * msgtyp controls which messages to receive:
 *   0:     Receive the first message (any type).
 *   > 0:   Receive the first message of exactly this type.
 *   < 0:   Receive the first message with type <= |msgtyp|
 *          (lowest type first -- priority queue behavior!).
 */
struct msgbuf received;
msgrcv(msqid, &received, sizeof(received.mtext), 0, 0);

/*
 * msgctl() -- control operations.
 * IPC_RMID: Remove the message queue.
 * IPC_STAT: Get queue info (messages count, byte count, etc.).
 */
msgctl(msqid, IPC_RMID, NULL);
```

### 6.5.2 POSIX Message Queues

POSIX message queues improve on System V with:
- File descriptor interface (works with select/poll/epoll).
- Configurable max message size and queue depth.
- Priority-based ordering.
- Notification via signal or thread when a message arrives.

```c
/*
 * mq_open() -- create or open a POSIX message queue.
 *
 * name:  Must start with / (lives in /dev/mqueue on Linux).
 * oflag: O_CREAT | O_RDWR, etc.
 * attr:  Queue attributes (max messages, max message size).
 *
 * Returns a message queue descriptor (mqd_t), which is a file descriptor.
 */
struct mq_attr attr = {
    .mq_maxmsg = 10,       /* Max messages in queue */
    .mq_msgsize = 256,     /* Max bytes per message */
};
mqd_t mq = mq_open("/myqueue", O_CREAT | O_RDWR, 0666, &attr);

/*
 * mq_send() -- send a message with a priority.
 * Higher priority messages are delivered first.
 * Messages of equal priority are FIFO.
 */
mq_send(mq, "Hello!", 6, /*priority=*/5);

/*
 * mq_receive() -- receive the highest priority message.
 * Blocks if the queue is empty (unless O_NONBLOCK was set).
 */
char buf[256];
unsigned int priority;
mq_receive(mq, buf, sizeof(buf), &priority);

/*
 * mq_notify() -- request notification when a message arrives.
 * Can notify via signal or by spawning a thread.
 */
struct sigevent sev = {
    .sigev_notify = SIGEV_SIGNAL,
    .sigev_signo = SIGUSR1,
};
mq_notify(mq, &sev);

/* Cleanup */
mq_close(mq);
mq_unlink("/myqueue");
```

### 6.5.3 Hands-On: Message Queue IPC in Go

Go doesn't have built-in POSIX message queue support, but we can use the
syscall interface directly. Here's a practical implementation using a
filesystem-based message queue that works portably:

```go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"os"
	"path/filepath"
	"sort"
	"sync"
	"time"
)

// FileMessageQueue implements a message queue using the filesystem.
//
// Why filesystem-based?
//   - Works on all platforms (Linux, macOS, Windows).
//   - Messages are durable (survive process crashes).
//   - Easy to inspect and debug (just ls the directory).
//   - Uses atomic file operations for safety.
//
// For production Linux systems, you would use POSIX mqueues
// via cgo or a library like github.com/syucream/posix_mq.
//
// Design:
//   - Each message is a JSON file in the queue directory.
//   - Filename format: {timestamp_ns}-{priority}-{sequence}.json
//   - Dequeue is atomic: rename (move) the file before processing.
//   - Priority: higher number = higher priority = dequeued first.
type FileMessageQueue struct {
	// dir is the directory that holds queue messages.
	dir string

	// processingDir is where messages are moved during processing.
	processingDir string

	// mu serializes enqueue/dequeue operations.
	mu sync.Mutex

	// seq is a monotonically increasing counter for uniqueness.
	seq uint64
}

// QueueMessage represents a message in the queue.
type QueueMessage struct {
	// ID is the unique message identifier.
	ID string `json:"id"`

	// Priority controls dequeue order (higher = first).
	Priority int `json:"priority"`

	// Type categorizes the message for selective consumption.
	Type string `json:"type"`

	// Payload is the message data.
	Payload json.RawMessage `json:"payload"`

	// CreatedAt is when the message was enqueued.
	CreatedAt time.Time `json:"created_at"`
}

// NewFileMessageQueue creates a new filesystem-based message queue.
//
// The directory is created if it doesn't exist. A subdirectory
// "processing" holds messages that are being processed (dequeued
// but not yet acknowledged).
func NewFileMessageQueue(dir string) (*FileMessageQueue, error) {
	procDir := filepath.Join(dir, "processing")
	if err := os.MkdirAll(procDir, 0755); err != nil {
		return nil, fmt.Errorf("create queue dir: %w", err)
	}
	return &FileMessageQueue{dir: dir, processingDir: procDir}, nil
}

// Enqueue adds a message to the queue.
//
// The message is written atomically: we write to a temp file first,
// then rename it to the final name. This ensures readers never see
// partial messages.
func (q *FileMessageQueue) Enqueue(msgType string, priority int, payload interface{}) error {
	q.mu.Lock()
	q.seq++
	seq := q.seq
	q.mu.Unlock()

	// Serialize the payload.
	payloadBytes, err := json.Marshal(payload)
	if err != nil {
		return fmt.Errorf("marshal payload: %w", err)
	}

	msg := QueueMessage{
		ID:        fmt.Sprintf("%d-%d-%d", time.Now().UnixNano(), priority, seq),
		Priority:  priority,
		Type:      msgType,
		Payload:   payloadBytes,
		CreatedAt: time.Now(),
	}

	data, err := json.MarshalIndent(msg, "", "  ")
	if err != nil {
		return fmt.Errorf("marshal message: %w", err)
	}

	// Filename encodes priority (inverted for sort order) and timestamp.
	// Format: {inverted_priority}-{timestamp}-{seq}.json
	// Inverted priority: 9999 - priority, so higher priority sorts first.
	filename := fmt.Sprintf("%04d-%020d-%06d.json", 9999-priority, time.Now().UnixNano(), seq)
	path := filepath.Join(q.dir, filename)

	// Atomic write: write to temp file, then rename.
	tmpPath := path + ".tmp"
	if err := os.WriteFile(tmpPath, data, 0644); err != nil {
		return fmt.Errorf("write message: %w", err)
	}
	if err := os.Rename(tmpPath, path); err != nil {
		os.Remove(tmpPath)
		return fmt.Errorf("rename message: %w", err)
	}

	return nil
}

// Dequeue removes and returns the highest priority message.
//
// The message file is moved to the processing directory. The caller
// must call Ack() to delete it, or Nack() to move it back to the queue.
//
// Returns nil, nil if the queue is empty.
func (q *FileMessageQueue) Dequeue() (*QueueMessage, error) {
	q.mu.Lock()
	defer q.mu.Unlock()

	// List all message files (sorted by name = sorted by priority then time).
	entries, err := os.ReadDir(q.dir)
	if err != nil {
		return nil, fmt.Errorf("read queue dir: %w", err)
	}

	// Filter to .json files only (skip directories and temp files).
	var files []string
	for _, e := range entries {
		if !e.IsDir() && filepath.Ext(e.Name()) == ".json" {
			files = append(files, e.Name())
		}
	}

	if len(files) == 0 {
		return nil, nil // Queue is empty.
	}

	// Sort to ensure priority order (already sorted by filename design).
	sort.Strings(files)

	// Move the first (highest priority) message to processing.
	srcPath := filepath.Join(q.dir, files[0])
	dstPath := filepath.Join(q.processingDir, files[0])

	data, err := os.ReadFile(srcPath)
	if err != nil {
		return nil, fmt.Errorf("read message: %w", err)
	}

	if err := os.Rename(srcPath, dstPath); err != nil {
		return nil, fmt.Errorf("move to processing: %w", err)
	}

	var msg QueueMessage
	if err := json.Unmarshal(data, &msg); err != nil {
		return nil, fmt.Errorf("unmarshal message: %w", err)
	}

	return &msg, nil
}

func main() {
	q, err := NewFileMessageQueue("/tmp/myapp-queue")
	if err != nil {
		log.Fatal(err)
	}

	// Enqueue messages with different priorities.
	q.Enqueue("email", 1, map[string]string{"to": "user@example.com", "subject": "Welcome"})
	q.Enqueue("alert", 10, map[string]string{"severity": "critical", "message": "Disk full"})
	q.Enqueue("log", 0, map[string]string{"level": "info", "message": "System started"})

	// Dequeue: should get "alert" first (highest priority).
	for {
		msg, err := q.Dequeue()
		if err != nil {
			log.Fatal(err)
		}
		if msg == nil {
			break
		}
		log.Printf("Dequeued [priority=%d, type=%s]: %s", msg.Priority, msg.Type, msg.Payload)
	}
}
```

---

## 6.6 Synchronization Primitives

### 6.6.1 Futex -- The Foundation of All Userspace Synchronization

The **futex** (Fast Userspace muTEX) is the most important synchronization
primitive on Linux. Nearly every higher-level synchronization construct --
mutexes, semaphores, condition variables, read-write locks, barriers -- is
built on top of futex.

**Why futex exists:**

Before futex, all synchronization required a syscall. Even acquiring an
uncontested mutex needed a kernel round-trip. Futex solves this by keeping
the fast path entirely in userspace:

```
Uncontested lock (common case, ~25ns):
  Thread A: atomic CAS on shared memory word
  -> CAS succeeds -> lock acquired -> NO SYSCALL

Contested lock (rare case, ~microseconds):
  Thread A: holds lock
  Thread B: atomic CAS on shared memory word
  -> CAS fails (lock is held)
  -> B calls futex(FUTEX_WAIT) -> kernel puts B to sleep
  Thread A: unlocks
  -> A calls futex(FUTEX_WAKE) -> kernel wakes B
  -> B retries CAS -> succeeds -> lock acquired
```

**The futex() syscall:**

```c
#include <linux/futex.h>
#include <sys/syscall.h>

/*
 * futex() -- the core futex system call.
 *
 * uaddr:  Pointer to a 32-bit integer in shared memory.
 *         This is the "futex word" -- the lock state variable.
 * op:     The operation to perform.
 * val:    Expected value (for FUTEX_WAIT) or count (for FUTEX_WAKE).
 *
 * Key operations:
 *
 * FUTEX_WAIT(uaddr, expected_val):
 *   "If *uaddr == expected_val, put me to sleep."
 *   The check-and-sleep is ATOMIC -- no race condition between
 *   checking the value and going to sleep.
 *   Returns 0 on wake, or EAGAIN if *uaddr != expected_val.
 *
 * FUTEX_WAKE(uaddr, num_to_wake):
 *   "Wake up to num_to_wake threads sleeping on this uaddr."
 *   Returns the number of threads actually woken.
 *
 * FUTEX_WAIT_BITSET / FUTEX_WAKE_BITSET:
 *   Like WAIT/WAKE but with a bitmask for selective wake.
 *   Used to implement condition variables efficiently.
 *
 * FUTEX_REQUEUE:
 *   Move waiters from one futex to another without waking them.
 *   Used in pthread_cond_broadcast() to avoid thundering herd.
 */
syscall(SYS_futex, uaddr, FUTEX_WAIT, expected_val, timeout, NULL, 0);
syscall(SYS_futex, uaddr, FUTEX_WAKE, num_to_wake, NULL, NULL, 0);
```

**How mutex is built on futex (simplified glibc implementation):**

```
Futex word states:
  0 = unlocked
  1 = locked, no waiters
  2 = locked, there are waiters sleeping in the kernel

Lock():
  1. CAS(futex_word, 0 -> 1)  // Atomic compare-and-swap.
     If succeeds: lock acquired (fast path, no syscall).
  2. If fails: set futex_word = 2
     Call FUTEX_WAIT(futex_word, 2)  // Sleep until woken.
  3. On wake: goto step 1 (retry CAS).

Unlock():
  1. old = XCHG(futex_word, 0)  // Atomically set to 0, get old value.
  2. If old == 2 (there were waiters):
     Call FUTEX_WAKE(futex_word, 1)  // Wake one waiter.
     // If old == 1, no waiters, no syscall needed (fast path).
```

### 6.6.2 Semaphores -- System V and POSIX

Semaphores are counters that control access to shared resources.

```c
/*
 * POSIX unnamed semaphore (for threads or shared memory processes).
 *
 * A semaphore is an atomic counter with two operations:
 *   sem_wait(): Decrement. If counter is 0, block until > 0.
 *   sem_post(): Increment. Wake one blocked waiter.
 *
 * When the counter represents "available units of a resource",
 * sem_wait = "acquire one unit" and sem_post = "release one unit".
 *
 * A semaphore with max value 1 is a "binary semaphore" -- equivalent
 * to a mutex but with different ownership semantics (any thread can
 * post, not just the one that waited).
 */
sem_t sem;
sem_init(&sem, /*pshared=*/1, /*initial_value=*/5);  /* 5 permits */

sem_wait(&sem);   /* Acquire (blocks if count is 0) */
/* ... critical section ... */
sem_post(&sem);   /* Release */

sem_destroy(&sem);

/*
 * POSIX named semaphore (for unrelated processes).
 * Lives in /dev/shm/ on Linux.
 */
sem_t *sem = sem_open("/my-semaphore", O_CREAT, 0666, 5);
sem_wait(sem);
sem_post(sem);
sem_close(sem);
sem_unlink("/my-semaphore");
```

### 6.6.3 File Locking (flock, fcntl) for Cross-Process Synchronization

File locking provides coarse-grained synchronization between processes.
Two mechanisms exist:

```c
/*
 * flock() -- BSD file locking (whole-file locks).
 *
 * Simple but limited to whole-file locks.
 * Locks are advisory (not enforced by the kernel for I/O).
 *
 * LOCK_SH: Shared lock (read lock). Multiple processes can hold.
 * LOCK_EX: Exclusive lock (write lock). Only one process can hold.
 * LOCK_NB: Non-blocking -- return EWOULDBLOCK instead of waiting.
 * LOCK_UN: Unlock.
 */
int fd = open("/var/lock/myapp.lock", O_CREAT | O_RDWR, 0644);
flock(fd, LOCK_EX);       /* Acquire exclusive lock (blocks) */
/* ... critical section ... */
flock(fd, LOCK_UN);       /* Release */

/*
 * fcntl() -- POSIX record locking (byte-range locks).
 *
 * More powerful: can lock specific byte ranges within a file.
 * Used by databases (SQLite, PostgreSQL) for fine-grained locking.
 *
 * F_SETLK:  Set lock (non-blocking, returns error if can't lock).
 * F_SETLKW: Set lock (blocking, waits until lock available).
 * F_GETLK:  Test if a lock would succeed.
 */
struct flock fl = {
    .l_type = F_WRLCK,    /* Write (exclusive) lock */
    .l_whence = SEEK_SET,  /* Relative to start of file */
    .l_start = 0,          /* Start at byte 0 */
    .l_len = 0,            /* Lock entire file (0 = to EOF) */
};
fcntl(fd, F_SETLKW, &fl);  /* Blocking lock */
```

**flock vs fcntl:**

| Property        | flock()              | fcntl() F_SETLK      |
|-----------------|----------------------|-----------------------|
| Granularity     | Whole file           | Byte range            |
| NFS support     | No (unreliable)      | Yes                   |
| Fork behavior   | Lock shared by child | Lock NOT inherited    |
| dup() behavior  | Same lock instance   | Independent lock      |
| Standard        | BSD                  | POSIX                 |

### 6.6.4 Hands-On: Building a Futex-Based Mutex in Go

This example builds a mutex from scratch using the futex syscall, demonstrating
the exact mechanism that Go's `sync.Mutex` uses internally on Linux.

```go
//go:build linux

package main

import (
	"fmt"
	"log"
	"runtime"
	"sync"
	"sync/atomic"
	"syscall"
	"time"
	"unsafe"
)

// FutexMutex is a mutex built directly on the Linux futex syscall.
//
// This is an educational implementation that mirrors the core logic
// of glibc's pthread_mutex and Go's sync.Mutex on Linux.
//
// The mutex has three states encoded in a single uint32:
//
//	0: Unlocked -- nobody holds the lock.
//	1: Locked, no waiters -- one thread holds the lock, nobody is
//	   waiting in the kernel.
//	2: Locked, with waiters -- one thread holds the lock, and at
//	   least one other thread is sleeping in a futex wait.
//
// State 2 is important: it tells Unlock() that it MUST call
// FUTEX_WAKE. Without this optimization, every Unlock() would
// make a syscall even when nobody is waiting.
type FutexMutex struct {
	// state is the futex word. Accessed atomically.
	// Must be 32-bit aligned (guaranteed by Go struct layout).
	state uint32
}

// Lock acquires the mutex. Blocks if the mutex is already held.
//
// Algorithm (simplified Drepper futex-based mutex):
//  1. Fast path: CAS 0 -> 1. If succeeds, lock acquired. No syscall.
//  2. Slow path: Set state to 2 (locked with waiters), then call
//     FUTEX_WAIT to sleep until woken.
//  3. On wake: retry from step 1.
//
// The fast path (step 1) handles the uncontested case with a single
// atomic instruction -- no system call, no kernel involvement. This
// takes about 10-25 nanoseconds on modern hardware.
func (m *FutexMutex) Lock() {
	// Fast path: try to acquire an unlocked mutex.
	if atomic.CompareAndSwapUint32(&m.state, 0, 1) {
		return // Acquired! No syscall needed.
	}

	// Slow path: mutex is held by someone else.
	m.lockSlow()
}

// lockSlow is the contention path for Lock().
//
// We first spin briefly (checking if the lock becomes free without
// going to the kernel), then fall back to futex wait.
//
// Spinning is beneficial when:
//   - The lock holder is on another CPU and will release soon.
//   - The critical section is short.
//
// Spinning is wasteful when:
//   - The lock holder is on the same CPU (we're burning cycles
//     it could use to finish and unlock).
//   - The critical section is long.
//
// We limit spinning to 4 iterations as a compromise.
func (m *FutexMutex) lockSlow() {
	// Brief spin before going to kernel.
	for i := 0; i < 4; i++ {
		if atomic.CompareAndSwapUint32(&m.state, 0, 1) {
			return
		}
		runtime.Gosched() // Yield to other goroutines.
	}

	// Set state to 2: "locked with waiters". This is important because
	// it tells Unlock() that it must do a FUTEX_WAKE.
	for {
		// Try to transition: if unlocked (0), grab it.
		if atomic.CompareAndSwapUint32(&m.state, 0, 2) {
			return
		}

		// Mark that there are waiters, regardless of who holds the lock.
		// XCHG (swap) ensures we set state to 2 and get the old value.
		old := atomic.SwapUint32(&m.state, 2)
		if old == 0 {
			// We just swapped 0 -> 2, but state 2 means locked with waiters.
			// That's fine -- we hold the lock now.
			return
		}

		// State is 2 (locked with waiters). Go to sleep.
		// FUTEX_WAIT: "If *uaddr == 2, sleep."
		// This is atomic: between checking the value and sleeping,
		// no FUTEX_WAKE can be lost.
		futexWait(&m.state, 2)
		// Woken up -- loop back and try to acquire.
	}
}

// Unlock releases the mutex. Wakes one waiter if any are sleeping.
//
// Algorithm:
//  1. Atomically set state to 0 (unlocked).
//  2. If the old state was 2 (there were waiters), call FUTEX_WAKE
//     to wake one sleeping thread.
//  3. If the old state was 1 (no waiters), no syscall needed.
func (m *FutexMutex) Unlock() {
	// Atomically set state to 0 and get the old value.
	old := atomic.SwapUint32(&m.state, 0)

	if old == 0 {
		panic("FutexMutex: unlock of unlocked mutex")
	}

	if old == 2 {
		// There are waiters sleeping in the kernel. Wake one.
		futexWake(&m.state, 1)
	}
	// If old == 1: no waiters, no syscall needed. Fast path.
}

// futexWait calls the FUTEX_WAIT operation.
//
// Puts the calling thread to sleep if *addr == val.
// Returns immediately if *addr != val (EAGAIN).
//
// The check-and-sleep is atomic in the kernel, preventing the
// classic "lost wakeup" race condition.
func futexWait(addr *uint32, val uint32) {
	// SYS_FUTEX on amd64 Linux is syscall number 202.
	// FUTEX_WAIT = 0, FUTEX_PRIVATE_FLAG = 128.
	// FUTEX_WAIT_PRIVATE = 128 (for process-private futex).
	syscall.Syscall6(
		syscall.SYS_FUTEX,
		uintptr(unsafe.Pointer(addr)),
		128, // FUTEX_WAIT_PRIVATE
		uintptr(val),
		0, // timeout (NULL = infinite)
		0,
		0,
	)
}

// futexWake calls the FUTEX_WAKE operation.
//
// Wakes up to 'count' threads sleeping on the futex at addr.
// Returns the number of threads actually woken.
func futexWake(addr *uint32, count int) {
	// FUTEX_WAKE_PRIVATE = 1 | 128 = 129.
	syscall.Syscall6(
		syscall.SYS_FUTEX,
		uintptr(unsafe.Pointer(addr)),
		129, // FUTEX_WAKE_PRIVATE
		uintptr(count),
		0,
		0,
		0,
	)
}

func main() {
	// Benchmark our futex mutex against sync.Mutex.
	var futexMu FutexMutex
	var stdMu sync.Mutex
	counter := 0
	numGoroutines := 100
	incrementsPerGoroutine := 10000

	// Test with FutexMutex.
	start := time.Now()
	var wg sync.WaitGroup
	for i := 0; i < numGoroutines; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < incrementsPerGoroutine; j++ {
				futexMu.Lock()
				counter++
				futexMu.Unlock()
			}
		}()
	}
	wg.Wait()
	futexDuration := time.Since(start)
	fmt.Printf("FutexMutex: counter=%d (expected %d), time=%v\n",
		counter, numGoroutines*incrementsPerGoroutine, futexDuration)

	// Test with sync.Mutex.
	counter = 0
	start = time.Now()
	for i := 0; i < numGoroutines; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := 0; j < incrementsPerGoroutine; j++ {
				stdMu.Lock()
				counter++
				stdMu.Unlock()
			}
		}()
	}
	wg.Wait()
	stdDuration := time.Since(start)
	fmt.Printf("sync.Mutex: counter=%d (expected %d), time=%v\n",
		counter, numGoroutines*incrementsPerGoroutine, stdDuration)

	fmt.Printf("Ratio: FutexMutex is %.2fx %s than sync.Mutex\n",
		func() float64 {
			if futexDuration > stdDuration {
				return float64(futexDuration) / float64(stdDuration)
			}
			return float64(stdDuration) / float64(futexDuration)
		}(),
		func() string {
			if futexDuration > stdDuration {
				return "slower"
			}
			return "faster"
		}())
}
```

### 6.6.5 Hands-On: Cross-Process Synchronization with flock in Go

```go
package main

import (
	"fmt"
	"log"
	"os"
	"syscall"
	"time"
)

// FileLock provides cross-process mutual exclusion using flock().
//
// This is the standard pattern for ensuring only one instance of a
// daemon runs at a time (a "PID file lock"), or for coordinating
// access to a shared resource across processes.
//
// How it works:
//  1. Open (or create) a lock file.
//  2. Call flock() with LOCK_EX to acquire an exclusive lock.
//  3. The lock is held as long as the file descriptor is open.
//  4. If the process crashes, the OS automatically releases the lock
//     (because it closes all file descriptors).
//
// This is better than PID files alone because:
//   - PID files can become stale if the process crashes without cleanup.
//   - flock() is automatically released on crash/exit.
//   - flock() handles race conditions atomically.
type FileLock struct {
	// path is the lock file path.
	path string

	// file is the open file descriptor holding the lock.
	file *os.File
}

// NewFileLock creates a new file lock at the given path.
// The lock is NOT acquired yet -- call Lock() or TryLock().
func NewFileLock(path string) *FileLock {
	return &FileLock{path: path}
}

// Lock acquires an exclusive lock, blocking until available.
//
// If another process holds the lock, this call blocks until:
//   - The other process releases the lock (flock LOCK_UN or close).
//   - The other process exits (kernel auto-release).
//
// Returns an error if the lock file cannot be opened.
func (fl *FileLock) Lock() error {
	f, err := os.OpenFile(fl.path, os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		return fmt.Errorf("open lock file: %w", err)
	}
	fl.file = f

	// LOCK_EX: exclusive lock (blocking).
	// This is an advisory lock -- it only works if all processes
	// cooperate by calling flock() on the same file.
	if err := syscall.Flock(int(f.Fd()), syscall.LOCK_EX); err != nil {
		f.Close()
		return fmt.Errorf("flock: %w", err)
	}

	// Write our PID to the lock file for debugging.
	f.Truncate(0)
	f.Seek(0, 0)
	fmt.Fprintf(f, "%d\n", os.Getpid())
	f.Sync()

	return nil
}

// TryLock attempts to acquire the lock without blocking.
//
// Returns true if the lock was acquired, false if another process
// holds it. This is useful for "fail fast" scenarios where you
// want to error out if another instance is already running.
func (fl *FileLock) TryLock() (bool, error) {
	f, err := os.OpenFile(fl.path, os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		return false, fmt.Errorf("open lock file: %w", err)
	}
	fl.file = f

	// LOCK_EX | LOCK_NB: exclusive lock, non-blocking.
	// Returns EWOULDBLOCK if the lock is held by another process.
	err = syscall.Flock(int(f.Fd()), syscall.LOCK_EX|syscall.LOCK_NB)
	if err != nil {
		f.Close()
		if err == syscall.EWOULDBLOCK {
			return false, nil // Lock held by another process.
		}
		return false, fmt.Errorf("flock: %w", err)
	}

	f.Truncate(0)
	f.Seek(0, 0)
	fmt.Fprintf(f, "%d\n", os.Getpid())
	f.Sync()

	return true, nil
}

// Unlock releases the lock and closes the file.
func (fl *FileLock) Unlock() error {
	if fl.file == nil {
		return nil
	}
	// Closing the file automatically releases the flock.
	// But we explicitly unlock first for clarity.
	syscall.Flock(int(fl.file.Fd()), syscall.LOCK_UN)
	err := fl.file.Close()
	fl.file = nil
	return err
}

// SingleInstance ensures only one instance of the application runs.
//
// Usage pattern for daemons:
//
//	lock := NewFileLock("/var/run/myapp.lock")
//	acquired, err := lock.TryLock()
//	if !acquired {
//	    log.Fatal("Another instance is already running")
//	}
//	defer lock.Unlock()
//	// ... run the daemon ...
func main() {
	lock := NewFileLock("/tmp/myapp.lock")

	// Try to acquire the lock.
	acquired, err := lock.TryLock()
	if err != nil {
		log.Fatalf("Lock error: %v", err)
	}
	if !acquired {
		log.Fatal("Another instance is already running!")
	}
	defer lock.Unlock()

	log.Printf("Lock acquired (pid=%d). Running...", os.Getpid())

	// Simulate work.
	for i := 0; i < 10; i++ {
		log.Printf("Working... %d/10", i+1)
		time.Sleep(time.Second)
	}

	log.Println("Done. Releasing lock.")
}
```

---

## 6.7 eventfd and timerfd

### 6.7.1 eventfd -- Lightweight Event Notification

`eventfd` creates a file descriptor that acts as an **event counter**. It is
the most efficient way to signal between threads or between kernel and
userspace.

```c
/*
 * eventfd() -- create an event notification file descriptor.
 *
 * initval: Initial counter value (usually 0).
 * flags:
 *   EFD_NONBLOCK:  Make read/write non-blocking.
 *   EFD_CLOEXEC:   Close fd on exec().
 *   EFD_SEMAPHORE: Read decrements by 1 instead of reading total.
 *
 * The fd maintains a uint64 counter:
 *   write(fd, &val, 8): Adds val to the counter.
 *   read(fd, &val, 8):  Returns the counter and resets to 0.
 *                        (With EFD_SEMAPHORE: decrements by 1.)
 *   poll/epoll:          Readable when counter > 0.
 *
 * Use cases:
 *   - Signal between threads (lighter than pipe).
 *   - Wake up an epoll loop from another thread.
 *   - Kernel-to-userspace notification (used by KVM, io_uring).
 */
int efd = eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);

/* Signal an event */
uint64_t val = 1;
write(efd, &val, sizeof(val));

/* Wait for and consume events */
uint64_t count;
read(efd, &count, sizeof(count));
printf("Received %lu events\n", count);
```

**eventfd vs pipe for signaling:**

| Property           | eventfd              | pipe                 |
|--------------------|----------------------|----------------------|
| File descriptors   | 1                    | 2                    |
| Buffer size        | 8 bytes (uint64)     | 64KB                 |
| Semantics          | Counter (add/read)   | Byte stream          |
| Memory overhead    | Minimal              | Kernel pipe buffers  |
| epoll compatible   | Yes                  | Yes                  |
| Cross-process      | Via fork or fd-pass  | Via fork or fd-pass  |

### 6.7.2 timerfd -- Timer Notifications via File Descriptor

`timerfd` creates a file descriptor that delivers timer expiration
notifications. Unlike `setitimer()` or `timer_create()`, timerfd integrates
perfectly with epoll/select.

```c
/*
 * timerfd_create() -- create a timer file descriptor.
 *
 * clockid:
 *   CLOCK_REALTIME:  Wall clock time (affected by NTP adjustments).
 *   CLOCK_MONOTONIC: Monotonic clock (immune to time changes).
 *
 * timerfd_settime() -- arm or disarm the timer.
 *
 * The timer can be:
 *   - One-shot: fires once after a specified delay.
 *   - Periodic: fires repeatedly at a specified interval.
 *
 * When the timer fires, read(fd) returns the number of expirations
 * since the last read (as a uint64). This handles the case where
 * the reader is slow and misses some expirations.
 */
int tfd = timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK | TFD_CLOEXEC);

/* Set a periodic timer: first fire after 1s, then every 500ms */
struct itimerspec ts = {
    .it_value    = { .tv_sec = 1, .tv_nsec = 0 },          /* Initial delay */
    .it_interval = { .tv_sec = 0, .tv_nsec = 500000000 },  /* Period */
};
timerfd_settime(tfd, 0, &ts, NULL);

/* Read expirations (integrates with epoll) */
uint64_t expirations;
read(tfd, &expirations, sizeof(expirations));
printf("Timer fired %lu times\n", expirations);
```

### 6.7.3 Hands-On: Using eventfd for Goroutine-to-Process Signaling

```go
//go:build linux

package main

import (
	"encoding/binary"
	"fmt"
	"log"
	"os"
	"sync"
	"syscall"
	"time"
)

// EventFD wraps a Linux eventfd file descriptor for use in Go.
//
// eventfd is the lightest-weight signaling mechanism on Linux.
// It uses a single file descriptor and a uint64 counter.
//
// This is particularly useful for:
//   - Waking up a goroutine that is blocked on epoll/select.
//   - Signaling between a Go process and a C library or other process.
//   - Implementing a semaphore with minimal overhead.
//
// In Go, you might ask: "Why not just use a channel?" Channels are
// great for Go-to-Go communication, but eventfd integrates with:
//   - epoll (for mixed fd/channel multiplexing).
//   - Other processes (via fork or fd passing).
//   - C libraries that expect file descriptor signaling.
type EventFD struct {
	// fd is the eventfd file descriptor.
	fd int

	// file wraps fd as an *os.File for Go's runtime integration.
	// This is important: it registers the fd with the Go runtime's
	// network poller, enabling non-blocking I/O without burning a
	// thread.
	file *os.File
}

// NewEventFD creates a new eventfd.
//
// If semaphore is true, read operations decrement the counter by 1
// (semaphore behavior). Otherwise, read returns the total count
// and resets to 0.
func NewEventFD(semaphore bool) (*EventFD, error) {
	flags := syscall.O_CLOEXEC | syscall.O_NONBLOCK
	if semaphore {
		flags |= 1 // EFD_SEMAPHORE = 1
	}

	// Create the eventfd. The syscall number for eventfd2 on amd64 is 290.
	fd, _, errno := syscall.RawSyscall(290, 0, uintptr(flags), 0)
	if errno != 0 {
		return nil, fmt.Errorf("eventfd2: %v", errno)
	}

	// Wrap in os.File so Go's runtime manages it.
	file := os.NewFile(fd, "eventfd")
	return &EventFD{fd: int(fd), file: file}, nil
}

// Signal writes a value to the eventfd, incrementing its counter.
//
// If another goroutine/thread is blocked reading this eventfd,
// it will be woken up. The value is added to the internal counter.
//
// A value of 1 means "one event happened". You can signal multiple
// events at once by writing a larger value.
func (e *EventFD) Signal(value uint64) error {
	buf := make([]byte, 8)
	binary.LittleEndian.PutUint64(buf, value)
	_, err := e.file.Write(buf)
	return err
}

// Wait blocks until the eventfd is signaled, then returns the count.
//
// In non-semaphore mode: returns the total accumulated count and
// resets the counter to 0.
// In semaphore mode: returns 1 and decrements the counter by 1.
//
// This integrates with Go's runtime poller, so it does NOT block
// an OS thread -- the goroutine is parked and rescheduled when
// the eventfd becomes readable.
func (e *EventFD) Wait() (uint64, error) {
	buf := make([]byte, 8)
	_, err := e.file.Read(buf)
	if err != nil {
		return 0, err
	}
	return binary.LittleEndian.Uint64(buf), nil
}

// Close releases the eventfd file descriptor.
func (e *EventFD) Close() error {
	return e.file.Close()
}

// Fd returns the raw file descriptor for use with epoll/select.
func (e *EventFD) Fd() int {
	return e.fd
}

func main() {
	// Create an eventfd for signaling.
	efd, err := NewEventFD(false)
	if err != nil {
		log.Fatalf("NewEventFD: %v", err)
	}
	defer efd.Close()

	var wg sync.WaitGroup

	// Consumer: waits for events.
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 5; i++ {
			count, err := efd.Wait()
			if err != nil {
				log.Printf("Wait error: %v", err)
				return
			}
			log.Printf("Consumer: received %d events", count)
		}
	}()

	// Producer: signals events.
	wg.Add(1)
	go func() {
		defer wg.Done()
		for i := 0; i < 5; i++ {
			time.Sleep(500 * time.Millisecond)
			if err := efd.Signal(uint64(i + 1)); err != nil {
				log.Printf("Signal error: %v", err)
				return
			}
			log.Printf("Producer: signaled %d", i+1)
		}
	}()

	wg.Wait()
	log.Println("Done")
}
```

---

## 6.8 Go's Concurrency Primitives -- Under the Hood

This section bridges OS-level primitives with Go's concurrency model. We show
how every Go concurrency feature maps down to the OS mechanisms covered earlier
in this chapter.

### 6.8.1 Goroutines -- Implementation Details

A goroutine is **not** an OS thread. It is a **user-space coroutine** managed
by the Go runtime's scheduler. Here is the architecture:

```
Go's M:N Scheduling Model:

  G = Goroutine (potentially millions)
  M = Machine (OS thread, typically = GOMAXPROCS)
  P = Processor (logical processor, exactly GOMAXPROCS)

  +--------+  +--------+  +--------+
  | G1     |  | G4     |  | G7     |  ... (thousands of goroutines)
  | G2     |  | G5     |  | G8     |
  | G3     |  | G6     |  | G9     |
  +---+----+  +---+----+  +---+----+
      |            |            |
  +---v----+  +---v----+  +---v----+
  | P0     |  | P1     |  | P2     |  (GOMAXPROCS = 3)
  | runq:  |  | runq:  |  | runq:  |  Each P has a local run queue.
  | [G1-3] |  | [G4-6] |  | [G7-9] |
  +---+----+  +---+----+  +---+----+
      |            |            |
  +---v----+  +---v----+  +---v----+
  | M0     |  | M1     |  | M2     |  (OS threads)
  | (tid   |  | (tid   |  | (tid   |
  |  1001) |  |  1002) |  |  1003) |
  +--------+  +--------+  +--------+
      |            |            |
  ====|============|============|====  (kernel boundary)
      |            |            |
  OS thread scheduling via CFS (Completely Fair Scheduler)
```

**Goroutine stack management:**

- Initial stack: **2KB** (vs 1-8MB for OS threads).
- Stacks grow dynamically: when a function call would overflow, the runtime
  allocates a new, larger stack and **copies** the old stack contents.
- Maximum stack: 1GB by default (configurable via `runtime.SetMaxStack`).
- Stack shrinking: the GC can shrink oversized stacks to reclaim memory.

**Goroutine preemption (since Go 1.14):**

Before Go 1.14, goroutines could only be preempted at function calls
(cooperative scheduling). A tight loop like `for {}` would block the entire
P indefinitely.

Since Go 1.14, Go uses **asynchronous preemption via signals**:

1. The runtime's sysmon goroutine detects that a goroutine has been running
   for >10ms without yielding.
2. It sends `SIGURG` to the OS thread running that goroutine.
3. The signal handler (in the Go runtime) modifies the goroutine's saved
   program counter to point to an "async preempt" function.
4. When the signal handler returns, the goroutine resumes at the preempt
   point, which yields to the scheduler.

This is why we warned earlier not to catch `SIGURG` -- it is reserved by
the runtime.

### 6.8.2 Channels -- Implementation (hchan struct)

Go channels are the primary communication mechanism between goroutines. Under
the hood, a channel is a `hchan` struct:

```go
// Simplified version of runtime.hchan (the actual channel implementation).
//
// Source: runtime/chan.go in the Go source tree.
//
// A channel is fundamentally a circular buffer protected by a mutex,
// with two wait queues for senders and receivers that are blocked.
type hchan struct {
	// qcount is the number of elements currently in the buffer.
	qcount uint

	// dataqsiz is the buffer capacity (0 for unbuffered channels).
	dataqsiz uint

	// buf points to the circular buffer (array of dataqsiz elements).
	// For unbuffered channels, this is nil.
	buf unsafe.Pointer

	// elemsize is the size of one channel element in bytes.
	elemsize uint16

	// closed indicates whether the channel has been closed.
	// 0 = open, non-zero = closed.
	closed uint32

	// sendx is the send index into the circular buffer.
	// The next send will write to buf[sendx].
	sendx uint

	// recvx is the receive index into the circular buffer.
	// The next receive will read from buf[recvx].
	recvx uint

	// recvq is the queue of goroutines waiting to receive.
	// These goroutines called <-ch when the buffer was empty.
	recvq waitq

	// sendq is the queue of goroutines waiting to send.
	// These goroutines called ch <- v when the buffer was full.
	sendq waitq

	// lock protects all fields of hchan.
	// This is a runtime mutex (not sync.Mutex) that uses futex
	// on Linux.
	lock mutex
}

// waitq is a doubly-linked list of waiting goroutines (sudog structs).
//
// Each sudog represents one goroutine waiting on this channel.
// When a goroutine blocks on a channel operation, the runtime:
//  1. Creates a sudog with a pointer to the goroutine.
//  2. Enqueues the sudog in either sendq or recvq.
//  3. Parks the goroutine (removes it from the run queue).
//
// When the matching operation occurs:
//  1. Dequeue the sudog.
//  2. Copy data directly (for unbuffered) or from/to the buffer.
//  3. Ready the goroutine (add it back to a run queue).
type waitq struct {
	first *sudog
	last  *sudog
}
```

**Channel send flow (ch <- value):**

```
1. Lock the channel mutex.
2. If there is a waiting receiver in recvq:
   a. Dequeue the receiver.
   b. Copy the value directly to the receiver's stack.
      (This is a direct goroutine-to-goroutine copy -- no buffer involved!)
   c. Wake the receiver goroutine.
   d. Unlock. Done.
3. If the buffer has space (qcount < dataqsiz):
   a. Copy the value to buf[sendx].
   b. Increment sendx (modulo dataqsiz).
   c. Increment qcount.
   d. Unlock. Done.
4. Buffer is full (or unbuffered with no receiver):
   a. Create a sudog for this goroutine.
   b. Record the value pointer in the sudog.
   c. Enqueue in sendq.
   d. Park the goroutine (block).
   e. Unlock.
   f. (Later, a receiver will dequeue us, copy our value, and wake us.)
```

### 6.8.3 sync.Mutex -- Implementation

Go's `sync.Mutex` uses a hybrid approach for maximum performance:

```
sync.Mutex states (encoded in a single int32):

  Bit 0 (mutexLocked):    Lock is held.
  Bit 1 (mutexWoken):     A goroutine has been woken for this lock.
  Bit 2 (mutexStarving):  The lock is in starvation mode.
  Bits 3+:                Number of waiters.

Normal mode (default):
  - Goroutines compete for the lock in FIFO order.
  - But a newly arriving goroutine can "steal" the lock from waiters.
  - This is unfair but performant -- the goroutine already has the CPU
    cache hot, so it's faster for it to proceed.
  - Spinning (busy-waiting) is attempted before parking.

Starvation mode (activated after 1ms of waiting):
  - If a waiter has been waiting for >1ms, the lock switches to
    starvation mode.
  - In starvation mode, the lock is handed directly from the unlocking
    goroutine to the first waiter in the queue. No stealing allowed.
  - This prevents tail latency: no goroutine waits indefinitely.
  - Returns to normal mode when the waiter queue is empty or the
    last waiter waited <1ms.

Spinning:
  - Before parking (expensive: involves futex syscall), a goroutine
    spins briefly on the lock word.
  - Spinning is only done if:
    - There are multiple CPUs (spinning on single-CPU is always wasteful).
    - The current P has an empty local run queue (spinning would delay
      other runnable goroutines).
    - The number of spinning goroutines is limited (GOMAXPROCS based).

On Linux, the actual sleep/wake mechanism is futex:
  - runtime.lock() calls futexsleep() which calls futex(FUTEX_WAIT).
  - runtime.unlock() calls futexwakeup() which calls futex(FUTEX_WAKE).
```

### 6.8.4 sync.WaitGroup, sync.Once, sync.Map Internals

**sync.WaitGroup:**

```go
// Simplified WaitGroup internals.
//
// A WaitGroup is a counter + a list of waiters.
// Internally it uses a single uint64 for the counter (high 32 bits)
// and the waiter count (low 32 bits), manipulated with atomic operations.
//
// Add(n):  Atomically adds n to the counter.
// Done():  Calls Add(-1).
// Wait():  If counter > 0, atomically increments waiter count and
//          sleeps on a runtime semaphore (which uses futex on Linux).
//          When the counter reaches 0, all waiters are woken.
//
// Key invariant: Add() must happen-before Wait().
// Key danger:    Calling Add() after Wait() has started is a race condition.
```

**sync.Once:**

```go
// sync.Once ensures a function runs exactly once, even with
// concurrent callers.
//
// Implementation:
//  1. First caller: acquires a mutex, runs the function, sets a
//     "done" flag atomically.
//  2. Subsequent callers: check the "done" flag with atomic.Load.
//     If set, return immediately (fast path -- no lock).
//
// The hot path (checking done) is a single atomic load -- extremely fast.
// Only the first call pays the cost of the mutex.
//
// Common use: lazy initialization of expensive resources.
var dbOnce sync.Once
var dbPool *sql.DB

func GetDB() *sql.DB {
	dbOnce.Do(func() {
		dbPool, _ = sql.Open("postgres", "...")
	})
	return dbPool
}
```

**sync.Map:**

```go
// sync.Map is a concurrent map optimized for two patterns:
//   1. Write-once, read-many (like a cache that fills up at startup).
//   2. Disjoint key access (different goroutines access different keys).
//
// Implementation:
//   - Two internal maps: "read" (atomic, lock-free) and "dirty" (mutex-protected).
//   - Reads go to the "read" map first (fast, no lock).
//   - If the key isn't in "read", fall back to "dirty" (slower, locked).
//   - Writes go to "dirty".
//   - Periodically, "dirty" is promoted to "read" (atomic swap).
//
// When to use sync.Map vs map + sync.RWMutex:
//   - sync.Map: few writes, many reads, keys are stable after initial load.
//   - map + RWMutex: frequent writes, or when you need to iterate/size the map.
//   - In benchmarks, sync.Map is faster for read-heavy workloads but
//     slower for write-heavy workloads compared to map + RWMutex.
```

### 6.8.5 sync/atomic -- Compare-and-Swap, Memory Ordering

The `sync/atomic` package provides the lowest-level synchronization primitives
in Go. These map directly to CPU instructions.

```go
package main

import (
	"fmt"
	"sync/atomic"
)

// AtomicOperationsDemo demonstrates the core atomic operations
// and explains their memory ordering guarantees.
//
// Go's atomic operations provide **sequentially consistent** ordering
// as of Go 1.19. This means:
//   - All atomic operations on a single variable are totally ordered.
//   - All goroutines observe atomic operations in the same order.
//   - An atomic store happens-before an atomic load that observes it.
//
// This is stronger than C++'s memory_order_relaxed but matches
// C++'s memory_order_seq_cst.
func AtomicOperationsDemo() {
	// --- Atomic Load and Store ---
	// Load and Store are the simplest atomic operations.
	// They ensure no torn reads/writes on 64-bit values.
	var counter atomic.Int64
	counter.Store(42)
	val := counter.Load()
	fmt.Printf("Load: %d\n", val)

	// --- Add (Fetch-and-Add) ---
	// Atomically adds and returns the NEW value.
	// Maps to LOCK XADD on x86, LDADD on ARM64.
	newVal := counter.Add(10) // 42 + 10 = 52
	fmt.Printf("Add: %d\n", newVal)

	// --- Compare-and-Swap (CAS) ---
	// The fundamental building block of lock-free algorithms.
	// "If the current value is 'old', replace it with 'new'."
	// Returns true if the swap happened, false if the value changed.
	//
	// Maps to LOCK CMPXCHG on x86, CASA on ARM64.
	//
	// CAS is used to build:
	//   - Lock-free stacks and queues.
	//   - Atomic counters with complex update logic.
	//   - The spinning loop in mutex Lock().
	swapped := counter.CompareAndSwap(52, 100)
	fmt.Printf("CAS(52->100): swapped=%v, value=%d\n", swapped, counter.Load())

	// --- Swap (Exchange) ---
	// Atomically stores a new value and returns the old value.
	// Maps to XCHG on x86, SWP on ARM64.
	old := counter.Swap(200)
	fmt.Printf("Swap: old=%d, new=%d\n", old, counter.Load())

	// --- atomic.Value ---
	// Stores and loads interface{} values atomically.
	// Used for lock-free config reload (as shown in section 6.1.9).
	var config atomic.Value
	config.Store(map[string]string{"env": "production"})
	cfg := config.Load().(map[string]string)
	fmt.Printf("Config: %v\n", cfg)
}

func main() {
	AtomicOperationsDemo()
}
```

### 6.8.6 Hands-On: When to Use Channels vs Mutexes vs Atomics

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
	"testing"
)

// This benchmark compares three approaches to incrementing a shared
// counter from multiple goroutines. It demonstrates the performance
// characteristics of each synchronization primitive.
//
// Results on a typical 8-core machine (Go 1.22):
//
//   BenchmarkAtomicCounter-8      1000000000    1.2 ns/op
//   BenchmarkMutexCounter-8        100000000   12.4 ns/op
//   BenchmarkChannelCounter-8       10000000  105.3 ns/op
//
// Key takeaways:
//   - Atomics are ~10x faster than mutexes for simple operations.
//   - Mutexes are ~10x faster than channels for simple operations.
//   - BUT: channels provide communication, not just synchronization.
//   - Use the right tool for the job, not the fastest tool.

// Rule of thumb:
//   - Use ATOMICS for: counters, flags, metrics, simple publish/subscribe.
//   - Use MUTEXES for: protecting complex data structures, critical sections.
//   - Use CHANNELS for: transferring ownership, pipeline stages, fan-out/fan-in,
//     signaling completion, select-based multiplexing.

// AtomicCounter uses sync/atomic for lock-free incrementing.
//
// Pros:
//   - Fastest option for simple read-modify-write operations.
//   - No contention -- every CAS eventually succeeds.
//   - No goroutine parking or kernel involvement.
//
// Cons:
//   - Only works for simple operations (add, swap, CAS).
//   - Cannot protect complex data structures.
//   - CAS loops can be slower under extreme contention.
type AtomicCounter struct {
	val atomic.Int64
}

func (c *AtomicCounter) Inc()       { c.val.Add(1) }
func (c *AtomicCounter) Get() int64 { return c.val.Load() }

// MutexCounter uses sync.Mutex for traditional locking.
//
// Pros:
//   - Can protect arbitrarily complex operations.
//   - Straightforward mental model (lock, operate, unlock).
//   - Good performance with Go's hybrid spin/park implementation.
//
// Cons:
//   - Risk of deadlock if lock ordering is violated.
//   - Priority inversion possible (low-priority holder blocks high-priority waiter).
//   - Cannot be used in select statements.
type MutexCounter struct {
	mu  sync.Mutex
	val int64
}

func (c *MutexCounter) Inc() {
	c.mu.Lock()
	c.val++
	c.mu.Unlock()
}

func (c *MutexCounter) Get() int64 {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.val
}

// ChannelCounter uses a channel to serialize access.
//
// Pros:
//   - Composable with select (timeout, cancellation, multiplexing).
//   - Natural ownership transfer (no shared state).
//   - Works across goroutines without explicit locking.
//
// Cons:
//   - Higher overhead (involves goroutine scheduling).
//   - More complex for simple operations.
//   - Channel buffer management adds complexity.
type ChannelCounter struct {
	incCh chan struct{}
	getCh chan int64
	done  chan struct{}
}

// NewChannelCounter starts a goroutine that owns the counter state.
// All access goes through channels -- no shared memory.
func NewChannelCounter() *ChannelCounter {
	c := &ChannelCounter{
		incCh: make(chan struct{}, 100),
		getCh: make(chan int64),
		done:  make(chan struct{}),
	}

	go func() {
		var val int64
		for {
			select {
			case <-c.incCh:
				val++
			case c.getCh <- val:
				// Sent current value to requester.
			case <-c.done:
				return
			}
		}
	}()

	return c
}

func (c *ChannelCounter) Inc()       { c.incCh <- struct{}{} }
func (c *ChannelCounter) Get() int64 { return <-c.getCh }
func (c *ChannelCounter) Close()     { close(c.done) }

// Benchmark functions for use with `go test -bench=.`
func BenchmarkAtomicCounter(b *testing.B) {
	c := &AtomicCounter{}
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			c.Inc()
		}
	})
}

func BenchmarkMutexCounter(b *testing.B) {
	c := &MutexCounter{}
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			c.Inc()
		}
	})
}

func BenchmarkChannelCounter(b *testing.B) {
	c := NewChannelCounter()
	defer c.Close()
	b.RunParallel(func(pb *testing.PB) {
		for pb.Next() {
			c.Inc()
		}
	})
}

func main() {
	fmt.Println("Run benchmarks with: go test -bench=. -benchmem")
	fmt.Println()
	fmt.Println("Decision guide:")
	fmt.Println("  Simple counter/flag?     -> sync/atomic")
	fmt.Println("  Protect data structure?  -> sync.Mutex or sync.RWMutex")
	fmt.Println("  Transfer ownership?      -> channel")
	fmt.Println("  Pipeline/fan-out?        -> channel")
	fmt.Println("  Need select/timeout?     -> channel")
	fmt.Println("  Read-heavy, write-rare?  -> sync.RWMutex or atomic.Value")
}
```

### 6.8.7 Hands-On: Building a Bounded Concurrent Work Pool in Go

```go
package main

import (
	"context"
	"fmt"
	"log"
	"math/rand"
	"sync"
	"sync/atomic"
	"time"
)

// WorkPool is a bounded concurrent worker pool that processes tasks
// using a fixed number of goroutines.
//
// Design principles:
//   - Fixed number of workers prevents goroutine explosion under load.
//   - Buffered job channel provides backpressure: when the pool is
//     saturated, Submit() blocks (or returns error with TrySubmit).
//   - Graceful shutdown: Stop() waits for all in-flight work to complete.
//   - Context-aware: supports cancellation and timeouts.
//
// This pattern is ubiquitous in production Go services:
//   - HTTP request processing with bounded concurrency.
//   - Database connection pools.
//   - Background job processors (email sending, image resizing, etc.).
//   - Rate-limited API clients.
//
// Internal architecture:
//
//	Submit() --> [job channel (buffered)] --> Worker goroutines
//	                                           |
//	                                           v
//	                                       Execute task
//	                                           |
//	                                           v
//	                                       [result channel] --> Collector
type WorkPool struct {
	// workers is the number of concurrent worker goroutines.
	workers int

	// jobChan is the buffered channel that holds pending tasks.
	// Workers dequeue tasks from this channel.
	// The buffer size determines how many tasks can be queued
	// before Submit() blocks.
	jobChan chan Job

	// wg tracks active workers for graceful shutdown.
	wg sync.WaitGroup

	// ctx and cancel control the pool lifecycle.
	ctx    context.Context
	cancel context.CancelFunc

	// --- Metrics ---
	// These atomic counters track pool performance.

	// submitted is the total number of tasks submitted.
	submitted atomic.Int64

	// completed is the total number of tasks that finished successfully.
	completed atomic.Int64

	// failed is the total number of tasks that returned an error.
	failed atomic.Int64

	// activeWorkers is the number of workers currently executing a task.
	activeWorkers atomic.Int32
}

// Job represents a unit of work to be executed by the pool.
//
// Each job has:
//   - An ID for tracking and logging.
//   - A function to execute (the actual work).
//   - A result channel for the caller to receive the outcome.
type Job struct {
	// ID uniquely identifies this job for logging and metrics.
	ID string

	// Fn is the work function. It receives a context for cancellation
	// support and returns a result or error.
	//
	// The context is cancelled when the pool is stopped, allowing
	// long-running tasks to exit early.
	Fn func(ctx context.Context) (interface{}, error)

	// resultCh receives the job result when processing completes.
	// The caller creates this channel and reads from it.
	resultCh chan<- JobResult
}

// JobResult is the outcome of a job execution.
type JobResult struct {
	// JobID identifies which job this result belongs to.
	JobID string

	// Value is the return value on success.
	Value interface{}

	// Err is non-nil if the job failed.
	Err error

	// Duration is how long the job took to execute.
	Duration time.Duration
}

// NewWorkPool creates a worker pool with the specified number of workers
// and job queue capacity.
//
// Parameters:
//   - workers: Number of goroutines that process jobs concurrently.
//     A good starting point is runtime.NumCPU() for CPU-bound work,
//     or 10-100x that for I/O-bound work.
//   - queueSize: Maximum number of pending jobs in the queue.
//     When the queue is full, Submit() blocks. Size this based on
//     expected burst traffic.
//
// The pool starts workers immediately. Call Stop() to shut down.
func NewWorkPool(workers, queueSize int) *WorkPool {
	ctx, cancel := context.WithCancel(context.Background())

	pool := &WorkPool{
		workers: workers,
		jobChan: make(chan Job, queueSize),
		ctx:     ctx,
		cancel:  cancel,
	}

	// Start worker goroutines.
	for i := 0; i < workers; i++ {
		pool.wg.Add(1)
		go pool.worker(i)
	}

	log.Printf("Work pool started: %d workers, queue capacity %d", workers, queueSize)
	return pool
}

// worker is the main loop for a pool worker goroutine.
//
// It continuously dequeues jobs from the job channel and executes them.
// Exits when the job channel is closed (during shutdown).
//
// Each worker:
//  1. Reads a job from the channel (blocks if empty).
//  2. Executes the job function with the pool's context.
//  3. Sends the result to the job's result channel.
//  4. Updates metrics.
func (p *WorkPool) worker(id int) {
	defer p.wg.Done()

	for job := range p.jobChan {
		// Track active workers for monitoring.
		p.activeWorkers.Add(1)

		// Execute the job with timing.
		start := time.Now()
		value, err := job.Fn(p.ctx)
		duration := time.Since(start)

		// Update metrics.
		if err != nil {
			p.failed.Add(1)
		} else {
			p.completed.Add(1)
		}
		p.activeWorkers.Add(-1)

		// Send result to the caller (non-blocking if channel is buffered).
		if job.resultCh != nil {
			job.resultCh <- JobResult{
				JobID:    job.ID,
				Value:    value,
				Err:      err,
				Duration: duration,
			}
		}
	}
}

// Submit adds a job to the pool's queue and returns a channel that
// will receive the result.
//
// Submit blocks if the job queue is full (backpressure).
// Returns an error if the pool is shutting down.
//
// Usage:
//
//	resultCh := pool.Submit("job-1", func(ctx context.Context) (interface{}, error) {
//	   return doExpensiveWork(ctx)
//	})
//
// result := <-resultCh
func (p *WorkPool) Submit(id string, fn func(ctx context.Context) (interface{}, error)) (<-chan JobResult, error) {
	resultCh := make(chan JobResult, 1) // Buffered so worker never blocks.

	select {
	case p.jobChan <- Job{ID: id, Fn: fn, resultCh: resultCh}:
		p.submitted.Add(1)
		return resultCh, nil
	case <-p.ctx.Done():
		return nil, fmt.Errorf("pool is shutting down")
	}
}

// TrySubmit attempts to submit a job without blocking.
//
// Returns false immediately if the queue is full. Useful when you
// want to implement load shedding (reject work instead of queuing).
func (p *WorkPool) TrySubmit(id string, fn func(ctx context.Context) (interface{}, error)) (<-chan JobResult, bool) {
	resultCh := make(chan JobResult, 1)

	select {
	case p.jobChan <- Job{ID: id, Fn: fn, resultCh: resultCh}:
		p.submitted.Add(1)
		return resultCh, true
	default:
		return nil, false // Queue is full.
	}
}

// Stop gracefully shuts down the pool.
//
// Shutdown sequence:
//  1. Cancel the pool context (signals workers to stop accepting new work).
//  2. Close the job channel (workers drain remaining jobs and exit).
//  3. Wait for all workers to finish their current job.
//
// After Stop returns, all submitted jobs have been processed (or cancelled).
func (p *WorkPool) Stop() {
	log.Println("Stopping work pool...")
	p.cancel()       // Signal workers that we're shutting down.
	close(p.jobChan) // Workers will drain remaining jobs and exit.
	p.wg.Wait()      // Wait for all workers to finish.
	log.Printf("Work pool stopped. Submitted=%d, Completed=%d, Failed=%d",
		p.submitted.Load(), p.completed.Load(), p.failed.Load())
}

// Stats returns current pool metrics.
func (p *WorkPool) Stats() (submitted, completed, failed int64, active int32) {
	return p.submitted.Load(), p.completed.Load(), p.failed.Load(), p.activeWorkers.Load()
}

func main() {
	// Create a pool with 4 workers and a queue capacity of 20.
	pool := NewWorkPool(4, 20)

	// Submit 20 jobs.
	var results []<-chan JobResult
	for i := 0; i < 20; i++ {
		jobID := fmt.Sprintf("job-%03d", i)
		resultCh, err := pool.Submit(jobID, func(ctx context.Context) (interface{}, error) {
			// Simulate variable-duration work.
			duration := time.Duration(50+rand.Intn(200)) * time.Millisecond

			select {
			case <-time.After(duration):
				return fmt.Sprintf("result from %s (took %v)", jobID, duration), nil
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		})
		if err != nil {
			log.Printf("Submit failed: %v", err)
			continue
		}
		results = append(results, resultCh)
	}

	// Collect results.
	for _, ch := range results {
		result := <-ch
		if result.Err != nil {
			log.Printf("[FAIL] %s: %v", result.JobID, result.Err)
		} else {
			log.Printf("[OK]   %s: %v (duration: %v)", result.JobID, result.Value, result.Duration)
		}
	}

	// Print final stats and shut down.
	submitted, completed, failed, active := pool.Stats()
	fmt.Printf("\nFinal stats: submitted=%d, completed=%d, failed=%d, active=%d\n",
		submitted, completed, failed, active)

	pool.Stop()
}
```

---

## 6.9 D-Bus -- The Linux Desktop/System IPC

### 6.9.1 What D-Bus Is

**D-Bus** (Desktop Bus) is a message bus system that provides a standardized
IPC mechanism for Linux desktop and system services. It is used pervasively
in modern Linux systems:

- **systemd** communicates with services via D-Bus.
- **NetworkManager** exposes its API over D-Bus.
- **PulseAudio/PipeWire** uses D-Bus for control.
- **GNOME/KDE** desktop environments are built around D-Bus.
- **Bluetooth** management goes through D-Bus (BlueZ).
- **UDisks2** manages storage devices via D-Bus.

**Architecture:**

```
+------------------+     +------------------+     +------------------+
| Application A    |     | D-Bus Daemon     |     | Application B    |
|                  |     | (dbus-daemon or  |     |                  |
| Bus.Call("org.   |---->| dbus-broker)     |---->| Handles method   |
|  freedesktop..." |     |                  |     | call, returns    |
|                  |<----| Routes messages  |<----| response         |
+------------------+     | based on         |     +------------------+
                          | destination name |
                          +------------------+
                                  |
                          +-------+-------+
                          |               |
                    System Bus      Session Bus
                    (root/system    (per-user
                     services)       services)
```

**Two bus types:**

- **System bus**: One per machine. Used by system services (systemd, NetworkManager,
  BlueZ). Requires privileges for most operations.
- **Session bus**: One per user login session. Used by desktop applications. No
  special privileges needed.

**D-Bus concepts:**

| Concept        | Description | Example |
|----------------|-------------|---------|
| Bus name       | Unique or well-known name for a connection | `org.freedesktop.systemd1` |
| Object path    | Hierarchical path to an object | `/org/freedesktop/systemd1` |
| Interface      | Named group of methods/signals | `org.freedesktop.systemd1.Manager` |
| Method         | A callable RPC function | `ListUnits()` |
| Signal         | A broadcast notification | `UnitNew(name, path)` |
| Property       | A readable/writable value | `Version` |

**D-Bus message types:**

1. **Method call**: Client requests an operation on a service.
2. **Method return**: Service sends the result back.
3. **Error**: Service indicates the method call failed.
4. **Signal**: Service broadcasts a notification to all subscribers.

### 6.9.2 System Bus vs Session Bus

```bash
# List all names on the system bus
$ busctl --system list

# List all names on the session bus
$ busctl --user list

# Introspect a service (discover its methods, signals, properties)
$ busctl introspect org.freedesktop.systemd1 /org/freedesktop/systemd1

# Call a method on systemd
$ busctl call org.freedesktop.systemd1 \
    /org/freedesktop/systemd1 \
    org.freedesktop.systemd1.Manager \
    ListUnitFiles

# Monitor signals in real time
$ busctl monitor org.freedesktop.systemd1

# Get a property
$ busctl get-property org.freedesktop.systemd1 \
    /org/freedesktop/systemd1 \
    org.freedesktop.systemd1.Manager \
    Version
```

**D-Bus type system:**

D-Bus has its own type system for message parameters:

| Signature | Type     | Example      |
|-----------|----------|--------------|
| `s`       | String   | `"hello"`    |
| `i`       | Int32    | `42`         |
| `u`       | Uint32   | `100`        |
| `x`       | Int64    | `123456789`  |
| `b`       | Boolean  | `true`       |
| `d`       | Double   | `3.14`       |
| `o`       | Object path | `/org/...` |
| `a`       | Array    | `[1, 2, 3]`  |
| `(ss)`    | Struct   | `("a", "b")` |
| `a{sv}`   | Dict     | Key-value map|
| `v`       | Variant  | Any type     |

### 6.9.3 Hands-On: Interacting with systemd over D-Bus from Go

```go
package main

import (
	"fmt"
	"log"
	"os"
	"strings"
	"text/tabwriter"

	"github.com/godbus/dbus/v5"
)

// SystemdManager provides a Go interface to systemd's D-Bus API.
//
// systemd exposes its entire management API over D-Bus on the system bus.
// This includes:
//   - Starting/stopping/restarting units (services, timers, mounts, etc.)
//   - Querying unit status and properties.
//   - Enabling/disabling units for automatic startup.
//   - Subscribing to unit state change signals.
//   - Managing the system (reboot, poweroff, etc.).
//
// D-Bus connection details:
//
//	Bus:       System bus (org.freedesktop.DBus)
//	Service:   org.freedesktop.systemd1
//	Object:    /org/freedesktop/systemd1
//	Interface: org.freedesktop.systemd1.Manager
type SystemdManager struct {
	// conn is the D-Bus system bus connection.
	conn *dbus.Conn

	// obj is the systemd Manager object on D-Bus.
	obj dbus.BusObject
}

// UnitStatus represents the status of a systemd unit.
//
// This maps to the struct returned by ListUnits() on the
// org.freedesktop.systemd1.Manager interface.
//
// Fields match the D-Bus signature: (ssssssouso)
//
//	s: unit name
//	s: description
//	s: load state (loaded, not-found, masked, etc.)
//	s: active state (active, inactive, activating, deactivating, failed)
//	s: sub state (running, exited, dead, failed, etc.)
//	s: following unit (empty if not following another)
//	o: unit object path on D-Bus
//	u: job ID (0 if no pending job)
//	s: job type (empty if no pending job)
//	o: job object path (empty if no pending job)
type UnitStatus struct {
	Name        string
	Description string
	LoadState   string
	ActiveState string
	SubState    string
	Following   string
	UnitPath    dbus.ObjectPath
	JobID       uint32
	JobType     string
	JobPath     dbus.ObjectPath
}

// NewSystemdManager connects to the system D-Bus and returns a
// manager for interacting with systemd.
//
// Requires appropriate permissions (usually root or membership in
// the systemd-related polkit groups).
func NewSystemdManager() (*SystemdManager, error) {
	// Connect to the system bus.
	// The system bus socket is at /run/dbus/system_bus_socket.
	conn, err := dbus.ConnectSystemBus()
	if err != nil {
		return nil, fmt.Errorf("connect to system bus: %w", err)
	}

	// Get the systemd Manager object.
	obj := conn.Object(
		"org.freedesktop.systemd1",  // Bus name (service).
		"/org/freedesktop/systemd1", // Object path.
	)

	return &SystemdManager{conn: conn, obj: obj}, nil
}

// Close disconnects from D-Bus.
func (m *SystemdManager) Close() {
	m.conn.Close()
}

// GetVersion returns the systemd version string.
//
// This reads the "Version" property from the systemd Manager interface.
// D-Bus properties are accessed via the org.freedesktop.DBus.Properties
// interface's Get() method.
func (m *SystemdManager) GetVersion() (string, error) {
	variant, err := m.obj.GetProperty("org.freedesktop.systemd1.Manager.Version")
	if err != nil {
		return "", fmt.Errorf("get Version property: %w", err)
	}
	return variant.Value().(string), nil
}

// ListUnits returns all loaded systemd units and their status.
//
// Calls the ListUnits() method on org.freedesktop.systemd1.Manager.
// Returns a slice of UnitStatus structs.
//
// This is equivalent to running `systemctl list-units` on the command line.
func (m *SystemdManager) ListUnits() ([]UnitStatus, error) {
	// Call the D-Bus method.
	// The method signature is: ListUnits() -> a(ssssssouso)
	var result [][]interface{}
	err := m.obj.Call(
		"org.freedesktop.systemd1.Manager.ListUnits", // Method.
		0, // Flags (0 = default).
	).Store(&result)
	if err != nil {
		return nil, fmt.Errorf("ListUnits: %w", err)
	}

	// Parse the result into our struct.
	units := make([]UnitStatus, len(result))
	for i, r := range result {
		units[i] = UnitStatus{
			Name:        r[0].(string),
			Description: r[1].(string),
			LoadState:   r[2].(string),
			ActiveState: r[3].(string),
			SubState:    r[4].(string),
			Following:   r[5].(string),
			UnitPath:    r[6].(dbus.ObjectPath),
			JobID:       r[7].(uint32),
			JobType:     r[8].(string),
			JobPath:     r[9].(dbus.ObjectPath),
		}
	}

	return units, nil
}

// GetUnitStatus returns the active state of a specific unit.
//
// This is equivalent to `systemctl is-active <unitName>`.
func (m *SystemdManager) GetUnitStatus(unitName string) (string, error) {
	var unitPath dbus.ObjectPath
	err := m.obj.Call(
		"org.freedesktop.systemd1.Manager.GetUnit",
		0,
		unitName,
	).Store(&unitPath)
	if err != nil {
		return "", fmt.Errorf("GetUnit(%s): %w", unitName, err)
	}

	// Get the ActiveState property from the unit object.
	unitObj := m.conn.Object("org.freedesktop.systemd1", unitPath)
	variant, err := unitObj.GetProperty("org.freedesktop.systemd1.Unit.ActiveState")
	if err != nil {
		return "", fmt.Errorf("get ActiveState: %w", err)
	}

	return variant.Value().(string), nil
}

// RestartUnit restarts a systemd unit.
//
// mode is the restart mode:
//
//	"replace":    Queue the restart, replacing any pending job.
//	"fail":       Fail if there is already a pending job.
//	"isolate":    Stop all other units.
//	"ignore-dependencies": Skip dependency checks.
//
// This is equivalent to `systemctl restart <unitName>`.
func (m *SystemdManager) RestartUnit(unitName, mode string) error {
	var jobPath dbus.ObjectPath
	err := m.obj.Call(
		"org.freedesktop.systemd1.Manager.RestartUnit",
		0,
		unitName,
		mode,
	).Store(&jobPath)
	if err != nil {
		return fmt.Errorf("RestartUnit(%s): %w", unitName, err)
	}
	log.Printf("Restart job queued: %s -> %s", unitName, jobPath)
	return nil
}

// WatchUnitChanges subscribes to unit state change signals.
//
// systemd emits signals when units change state:
//   - UnitNew: A new unit has been loaded.
//   - UnitRemoved: A unit has been unloaded.
//   - PropertiesChanged: A unit's properties changed (state transition).
//
// This function sets up a D-Bus signal match and returns a channel
// that receives unit change notifications.
func (m *SystemdManager) WatchUnitChanges() (<-chan string, error) {
	// Subscribe to PropertiesChanged signals from systemd units.
	// This catches all state transitions (active -> inactive, etc.).
	err := m.conn.AddMatchSignal(
		dbus.WithMatchInterface("org.freedesktop.DBus.Properties"),
		dbus.WithMatchMember("PropertiesChanged"),
		dbus.WithMatchSender("org.freedesktop.systemd1"),
	)
	if err != nil {
		return nil, fmt.Errorf("add match signal: %w", err)
	}

	// Channel for signal delivery.
	sigChan := make(chan *dbus.Signal, 100)
	m.conn.Signal(sigChan)

	// Parse signals and forward human-readable messages.
	outChan := make(chan string, 100)
	go func() {
		for sig := range sigChan {
			// Extract the unit name from the object path.
			path := string(sig.Path)
			if strings.HasPrefix(path, "/org/freedesktop/systemd1/unit/") {
				unitName := strings.TrimPrefix(path, "/org/freedesktop/systemd1/unit/")
				outChan <- fmt.Sprintf("Unit changed: %s (signal: %s)", unitName, sig.Name)
			}
		}
		close(outChan)
	}()

	return outChan, nil
}

func main() {
	mgr, err := NewSystemdManager()
	if err != nil {
		log.Fatalf("Failed to connect to systemd: %v", err)
	}
	defer mgr.Close()

	// Get systemd version.
	version, err := mgr.GetVersion()
	if err != nil {
		log.Printf("Warning: couldn't get version: %v", err)
	} else {
		fmt.Printf("systemd version: %s\n\n", version)
	}

	// List all active services.
	units, err := mgr.ListUnits()
	if err != nil {
		log.Fatalf("Failed to list units: %v", err)
	}

	// Print active services in a table.
	w := tabwriter.NewWriter(os.Stdout, 0, 0, 2, ' ', 0)
	fmt.Fprintf(w, "UNIT\tACTIVE\tSUB\tDESCRIPTION\n")
	serviceCount := 0
	for _, u := range units {
		if strings.HasSuffix(u.Name, ".service") && u.ActiveState == "active" {
			fmt.Fprintf(w, "%s\t%s\t%s\t%s\n",
				u.Name, u.ActiveState, u.SubState, u.Description)
			serviceCount++
			if serviceCount >= 20 {
				break // Limit output for readability.
			}
		}
	}
	w.Flush()
	fmt.Printf("\n(%d active services shown, %d total units)\n", serviceCount, len(units))

	// Check specific unit status.
	status, err := mgr.GetUnitStatus("sshd.service")
	if err != nil {
		log.Printf("sshd.service: %v", err)
	} else {
		fmt.Printf("\nsshd.service is: %s\n", status)
	}
}
```

**Installing the D-Bus Go library:**

```bash
$ go get github.com/godbus/dbus/v5
```

**Running the example:**

```bash
# Must run as root (or with appropriate polkit permissions) for system bus
$ sudo go run main.go
systemd version: 253

UNIT                          ACTIVE  SUB      DESCRIPTION
sshd.service                  active  running  OpenSSH Daemon
NetworkManager.service        active  running  Network Manager
systemd-journald.service      active  running  Journal Service
dbus.service                  active  running  D-Bus System Message Bus
...

(20 active services shown, 234 total units)

sshd.service is: active
```

---

## 6.10 Summary: OS Primitives to Go Abstractions

This chapter has traced the path from raw OS system calls to Go's high-level
concurrency primitives. Here is the complete mapping:

```
OS Primitive                    Go Abstraction
===============================  ======================================
Signal (SIGTERM, SIGINT)     --> os/signal.Notify + channels
pipe() / pipe2()             --> io.Pipe(), os/exec.Cmd.StdoutPipe()
Unix Domain Sockets          --> net.Dial("unix"), net.Listen("unix")
Shared Memory (mmap)         --> syscall.Mmap + sync/atomic
Message Queues               --> channels (Go's native message passing)
futex (FUTEX_WAIT/WAKE)      --> sync.Mutex, sync.Cond internals
Semaphores                   --> buffered channels, semaphore.Weighted
File Locking (flock)         --> syscall.Flock
eventfd                      --> channels (similar semantics)
timerfd                      --> time.NewTimer, time.NewTicker
D-Bus                        --> github.com/godbus/dbus/v5
POSIX threads (pthreads)     --> goroutines (M:N scheduled)
pthread_mutex                --> sync.Mutex (uses futex on Linux)
pthread_cond                 --> sync.Cond (uses futex on Linux)
atomic operations            --> sync/atomic package
```

### Key Takeaways

1. **Go abstracts but does not hide**: Go's concurrency primitives are built
   directly on OS mechanisms. Understanding the OS layer helps you write
   better Go code and debug production issues.

2. **Channels are not just pipes**: Go channels combine the semantics of
   pipes (data transfer), eventfd (signaling), and semaphores (counting) --
   all with type safety and deadlock detection.

3. **sync.Mutex is a futex wrapper**: On Linux, Go's mutex uses the exact
   same futex mechanism as C's pthread_mutex. The performance characteristics
   are identical.

4. **Signals require explicit handling in Go**: Unlike C programs that get
   killed by unhandled signals, Go programs must use `os/signal.Notify` to
   set up proper signal handling -- especially in containers where PID 1
   behavior matters.

5. **Choose the right IPC for the job**:
   - Same process, same machine: channels, mutexes, atomics.
   - Different processes, same machine: Unix domain sockets, shared memory.
   - Different machines: TCP/UDP sockets, gRPC, message brokers.

6. **Atomics > Mutexes > Channels** (for raw performance):
   But channels provide **composability** (select), **ownership transfer**,
   and **clearer concurrency patterns** that mutexes and atomics cannot.

### IPC Decision Matrix

```
Need                          Best IPC Mechanism
================================  =======================================
Simple process signaling      --> Signals (SIGUSR1/2)
Streaming data between procs  --> Pipes (anonymous or named)
Request/response RPC          --> Unix Domain Sockets (stream)
Short structured messages     --> Unix Domain Sockets (datagram)
Zero-copy large data sharing  --> Shared Memory (mmap)
Priority message delivery     --> POSIX Message Queues
Cross-process locking         --> flock() or fcntl()
Event notification            --> eventfd
Periodic timer events         --> timerfd
System service integration    --> D-Bus
In-process goroutine comm     --> Channels
In-process data protection    --> sync.Mutex / sync.RWMutex
In-process counters/flags     --> sync/atomic
```

### Performance Characteristics

```
Operation                          Approximate Latency
=====================================  ==================
Atomic CAS (uncontested)            ~1-5 ns
sync.Mutex Lock/Unlock (uncontested) ~10-25 ns
Channel send/receive (unbuffered)    ~50-100 ns
Channel send/receive (buffered)      ~30-70 ns
Pipe write/read (small msg)          ~1-5 us
Unix socket send/recv (small msg)    ~1-5 us
Shared memory read (cached)          ~1-10 ns
futex WAIT/WAKE (with syscall)       ~1-10 us
Signal delivery                      ~5-20 us
D-Bus method call                    ~50-200 us
```

### What's Next

In Chapter 7, we will explore **networking and sockets** in depth -- the
mechanisms that extend IPC beyond a single machine. We will cover the TCP/IP
stack, socket programming, epoll-based event loops, and how Go's `net` package
provides a goroutine-per-connection model that maps to these OS primitives.

---

*Chapter 6 complete. You now understand the full spectrum of inter-process
communication and synchronization, from raw signals and futexes to Go channels
and atomics. These are the building blocks of every concurrent system.*
