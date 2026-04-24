# Chapter 4: Linux Networking — Sockets, TCP/IP Stack, Netfilter

> *"The network is the computer."* — John Gage, Sun Microsystems

Every Go program that calls `net.Dial` or `http.ListenAndServe` is sitting atop one of the
most sophisticated software systems ever built: the Linux network stack. This chapter tears
off the abstraction layers. We will build servers from raw syscalls, craft packets by hand,
trace a packet's journey through the kernel, manipulate firewall rules, create isolated
network namespaces, and peek inside Go's runtime to see how the netpoller turns blocking
I/O into goroutine-friendly concurrency.

By the end of this chapter you will understand:

- How the socket API maps to kernel data structures
- Every state in the TCP state machine and why each exists
- How packets traverse Netfilter hooks and iptables chains
- How network namespaces, veth pairs, and bridges form the foundation of container networking
- How eBPF is revolutionizing Linux networking
- What Go's `net` package does under the hood — and when to bypass it

Let's begin at the very bottom: the socket.

---

## 4.1 The Socket API

The Berkeley socket API, introduced in BSD 4.2 (1983), remains the universal interface for
network programming on Unix systems. Every network operation — whether it's a Go HTTP server
or a C database client — ultimately funnels through a handful of syscalls.

### 4.1.1 The Socket Lifecycle

Here is the complete lifecycle for a TCP server:

```text
Server                              Client
──────                              ──────
socket()    → fd                    socket()    → fd
bind()      → assign address
listen()    → mark as passive
                                    connect()   → initiate handshake
accept()    → new connected fd          ↕ (three-way handshake)
read/write  ↔  read/write
close()                             close()
```

Each of these is a syscall that transitions the socket through kernel-level states.

#### socket() — Creating a Communication Endpoint

```c
int socket(int domain, int type, int protocol);
```

This syscall creates a socket and returns a file descriptor. The three parameters define:

- **domain** (address family): Determines the protocol family
- **type**: Determines the communication semantics
- **protocol**: Usually 0 (auto-select) or a specific protocol number

#### Address Families

| Constant    | Value | Description                          |
|-------------|-------|--------------------------------------|
| `AF_INET`   | 2     | IPv4 Internet protocols              |
| `AF_INET6`  | 10    | IPv6 Internet protocols              |
| `AF_UNIX`   | 1     | Local inter-process communication    |
| `AF_PACKET` | 17    | Low-level packet interface           |
| `AF_NETLINK`| 16    | Kernel/user-space interface          |

#### Socket Types

| Constant      | Description                                           |
|---------------|-------------------------------------------------------|
| `SOCK_STREAM` | Reliable, ordered, connection-based byte stream (TCP) |
| `SOCK_DGRAM`  | Connectionless, unreliable datagrams (UDP)            |
| `SOCK_RAW`    | Raw protocol access (ICMP, custom protocols)          |
| `SOCK_SEQPACKET` | Reliable, ordered, connection-based datagrams     |

#### bind() — Assigning an Address

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

Associates a socket with a specific address. For TCP/UDP servers, this is the
IP:port combination. The `sockaddr` structure is the generic container:

```c
// Generic socket address — the "base class"
struct sockaddr {
    sa_family_t sa_family;   // Address family (AF_INET, AF_INET6, etc.)
    char        sa_data[14]; // Protocol-specific address data
};

// IPv4 socket address
struct sockaddr_in {
    sa_family_t    sin_family; // Always AF_INET
    in_port_t      sin_port;   // Port number (network byte order!)
    struct in_addr sin_addr;   // IPv4 address
    char           sin_zero[8]; // Padding to match sockaddr size
};

// IPv6 socket address
struct sockaddr_in6 {
    sa_family_t     sin6_family;   // Always AF_INET6
    in_port_t       sin6_port;     // Port number
    uint32_t        sin6_flowinfo; // Flow information
    struct in6_addr sin6_addr;     // IPv6 address
    uint32_t        sin6_scope_id; // Scope ID
};

// Unix domain socket address
struct sockaddr_un {
    sa_family_t sun_family; // Always AF_UNIX
    char        sun_path[108]; // Pathname
};
```

#### listen() — Marking a Socket as Passive

```c
int listen(int sockfd, int backlog);
```

Transitions a socket from CLOSED to LISTEN state. The `backlog` parameter is
critically important — it controls the size of the **accept queue** (completed
connections waiting for `accept()`). We'll cover SYN queue vs accept queue in
detail in Section 4.2.

#### accept() — Accepting a Connection

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

Blocks until a connection is available in the accept queue, then returns a
**new** file descriptor for the connected socket. The original listening socket
continues to accept new connections.

#### connect() — Initiating a Connection

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

For TCP, this initiates the three-way handshake. For UDP, it merely sets the
default destination address (UDP is connectionless, but `connect()` allows
using `send()` instead of `sendto()`).

---

### 4.1.2 Hands-On: TCP Server from Raw Syscalls in Go

Most Go developers use `net.Listen` and `net.Accept`. Here, we bypass the entire
`net` package and talk directly to the kernel via `syscall`/`golang.org/x/sys/unix`.

```go
// tcp_raw_server.go
//
// A TCP echo server built entirely from raw syscalls.
// This bypasses Go's net package to demonstrate exactly what
// happens at the kernel level when you create a TCP server.
//
// We use golang.org/x/sys/unix for cleaner syscall wrappers,
// but every call maps 1:1 to a Linux syscall.
//
// Usage:
//
//	go run tcp_raw_server.go
//	# In another terminal: echo "hello" | nc localhost 8080
package main

import (
	"fmt"
	"log"
	"os"

	"golang.org/x/sys/unix"
)

func main() {
	// ──────────────────────────────────────────────────────────
	// Step 1: socket() — Create a TCP socket
	//
	// AF_INET    = IPv4
	// SOCK_STREAM = TCP (reliable, ordered byte stream)
	// 0          = let the kernel choose the protocol (IPPROTO_TCP)
	//
	// Under the hood, the kernel allocates a struct socket,
	// associates it with the TCP protocol handler, and returns
	// a file descriptor pointing to this socket.
	// ──────────────────────────────────────────────────────────
	serverFd, err := unix.Socket(unix.AF_INET, unix.SOCK_STREAM, 0)
	if err != nil {
		log.Fatalf("socket() failed: %v", err)
	}
	defer unix.Close(serverFd)

	// ──────────────────────────────────────────────────────────
	// Step 2: Set SO_REUSEADDR
	//
	// Without this, restarting the server immediately after stopping
	// it would fail with "address already in use" because the socket
	// lingers in TIME_WAIT state (explained in Section 4.2).
	//
	// SO_REUSEADDR tells the kernel: "Yes, I know there might be a
	// socket in TIME_WAIT on this address. Bind anyway."
	// ──────────────────────────────────────────────────────────
	err = unix.SetsockoptInt(serverFd, unix.SOL_SOCKET, unix.SO_REUSEADDR, 1)
	if err != nil {
		log.Fatalf("setsockopt(SO_REUSEADDR) failed: %v", err)
	}

	// ──────────────────────────────────────────────────────────
	// Step 3: bind() — Assign address 0.0.0.0:8080
	//
	// We construct a SockaddrInet4 which maps to the kernel's
	// struct sockaddr_in. Port is in host byte order here —
	// Go's unix package handles the conversion to network byte
	// order (big-endian) internally.
	//
	// Addr: [4]byte{0,0,0,0} = INADDR_ANY = bind to all interfaces
	// ──────────────────────────────────────────────────────────
	addr := &unix.SockaddrInet4{
		Port: 8080,
		Addr: [4]byte{0, 0, 0, 0}, // INADDR_ANY
	}
	err = unix.Bind(serverFd, addr)
	if err != nil {
		log.Fatalf("bind() failed: %v", err)
	}

	// ──────────────────────────────────────────────────────────
	// Step 4: listen() — Mark as passive socket
	//
	// The backlog parameter (128 here) controls the maximum length
	// of the accept queue — completed TCP connections waiting for
	// accept(). The kernel may silently cap this to the value in
	// /proc/sys/net/core/somaxconn (typically 4096 on modern kernels).
	//
	// After listen(), the kernel starts responding to incoming SYN
	// packets with SYN-ACK, performing the three-way handshake
	// even before our code calls accept().
	// ──────────────────────────────────────────────────────────
	err = unix.Listen(serverFd, 128)
	if err != nil {
		log.Fatalf("listen() failed: %v", err)
	}

	fmt.Println("Raw TCP server listening on :8080")

	// ──────────────────────────────────────────────────────────
	// Step 5: accept() loop — Accept connections
	//
	// accept() blocks until a completed connection is available
	// in the accept queue. It returns:
	//   - A NEW file descriptor for the connected socket
	//   - The remote address (client IP:port)
	//
	// The original serverFd continues listening for more connections.
	// ──────────────────────────────────────────────────────────
	for {
		clientFd, clientAddr, err := unix.Accept(serverFd)
		if err != nil {
			log.Printf("accept() failed: %v", err)
			continue
		}

		// Type-assert to get the IPv4 address details.
		if sa, ok := clientAddr.(*unix.SockaddrInet4); ok {
			fmt.Printf("Connection from %d.%d.%d.%d:%d\n",
				sa.Addr[0], sa.Addr[1], sa.Addr[2], sa.Addr[3], sa.Port)
		}

		// Handle the connection in a goroutine.
		// Even though we're using raw syscalls for the socket,
		// we can still leverage goroutines for concurrency.
		go handleConnection(clientFd)
	}
}

// handleConnection reads data from the client and echoes it back.
// This is a simple echo server — every byte received is sent back.
//
// Note: We use raw unix.Read/unix.Write instead of os.File or bufio.
// This means we're making direct read(2)/write(2) syscalls.
// In production, you'd want non-blocking I/O or Go's netpoller.
func handleConnection(fd int) {
	defer unix.Close(fd)

	buf := make([]byte, 4096)
	for {
		// ──────────────────────────────────────────────────
		// read() — Read up to 4096 bytes from the socket.
		//
		// This maps directly to the read(2) syscall.
		// Returns:
		//   n > 0  : n bytes were read into buf
		//   n == 0 : EOF — the client closed the connection
		//   err    : some error occurred
		// ──────────────────────────────────────────────────
		n, err := unix.Read(fd, buf)
		if err != nil {
			log.Printf("read() error: %v", err)
			return
		}
		if n == 0 {
			// Client closed the connection (sent FIN).
			return
		}

		// ──────────────────────────────────────────────────
		// write() — Echo the data back.
		//
		// Important: write() may write fewer bytes than
		// requested (short write). In production code, you
		// must loop until all bytes are written. We simplify
		// here for clarity.
		// ──────────────────────────────────────────────────
		_, err = unix.Write(fd, buf[:n])
		if err != nil {
			log.Printf("write() error: %v", err)
			return
		}
	}
}
```

**What happens in the kernel when this code runs:**

1. `socket()` allocates a `struct socket` and a `struct sock` (the protocol-specific
   part). For TCP, this is a `struct tcp_sock`. The kernel creates a file descriptor
   in the process's file descriptor table pointing to this socket.

2. `bind()` associates the socket with port 8080. The kernel adds an entry to the
   TCP port hash table so incoming packets can be routed to this socket.

3. `listen()` transitions the socket state from `TCP_CLOSE` to `TCP_LISTEN`. The
   kernel allocates the SYN queue and accept queue data structures.

4. `accept()` dequeues a completed connection from the accept queue. If the queue
   is empty, the calling thread sleeps until a connection arrives.

---

### 4.1.3 Hands-On: UDP Server from Raw Syscalls in Go

UDP is simpler — no connection setup, no handshake, no state machine.

```go
// udp_raw_server.go
//
// A UDP echo server using raw syscalls.
// UDP is connectionless: there's no accept(), no handshake.
// Each recvfrom() returns one complete datagram with the sender's address.
//
// Usage:
//
//	go run udp_raw_server.go
//	# In another terminal: echo "hello" | nc -u localhost 9090
package main

import (
	"fmt"
	"log"

	"golang.org/x/sys/unix"
)

func main() {
	// ──────────────────────────────────────────────────────────
	// Create a UDP socket.
	//
	// SOCK_DGRAM = datagram socket = UDP
	// Unlike SOCK_STREAM, there is no connection concept.
	// Each send/receive operates on a complete datagram.
	// ──────────────────────────────────────────────────────────
	fd, err := unix.Socket(unix.AF_INET, unix.SOCK_DGRAM, 0)
	if err != nil {
		log.Fatalf("socket() failed: %v", err)
	}
	defer unix.Close(fd)

	// Bind to port 9090 on all interfaces.
	addr := &unix.SockaddrInet4{
		Port: 9090,
		Addr: [4]byte{0, 0, 0, 0},
	}
	if err := unix.Bind(fd, addr); err != nil {
		log.Fatalf("bind() failed: %v", err)
	}

	fmt.Println("Raw UDP server listening on :9090")

	buf := make([]byte, 65535) // Max UDP datagram size
	for {
		// ──────────────────────────────────────────────────
		// Recvfrom() — Receive a datagram.
		//
		// Unlike TCP's read(), recvfrom() returns:
		//   - The data (one complete datagram, never partial)
		//   - The sender's address (so we can reply)
		//   - Flags (e.g., MSG_TRUNC if datagram was truncated)
		//
		// UDP preserves message boundaries: if the sender
		// sends 100 bytes, the receiver gets exactly 100 bytes
		// in one recvfrom() call.
		// ──────────────────────────────────────────────────
		n, from, err := unix.Recvfrom(fd, buf, 0)
		if err != nil {
			log.Printf("recvfrom() error: %v", err)
			continue
		}

		if sa, ok := from.(*unix.SockaddrInet4); ok {
			fmt.Printf("Received %d bytes from %d.%d.%d.%d:%d\n",
				n, sa.Addr[0], sa.Addr[1], sa.Addr[2], sa.Addr[3], sa.Port)
		}

		// ──────────────────────────────────────────────────
		// Sendto() — Send a datagram back to the sender.
		//
		// We must specify the destination address because
		// UDP sockets are connectionless. Each datagram
		// is independently addressed.
		// ──────────────────────────────────────────────────
		err = unix.Sendto(fd, buf[:n], 0, from)
		if err != nil {
			log.Printf("sendto() error: %v", err)
		}
	}
}
```

**TCP vs UDP at the syscall level:**

```
TCP Server                    UDP Server
──────────                    ──────────
socket(SOCK_STREAM)           socket(SOCK_DGRAM)
bind()                        bind()
listen()                      (no listen — no connections)
accept() → new fd             (no accept — no connections)
read()/write()                recvfrom()/sendto()
close()                       close()
```

---

### 4.1.4 Hands-On: Unix Domain Sockets in Go

Unix domain sockets (`AF_UNIX`) provide IPC on the same host. They're faster
than TCP loopback because they bypass the entire TCP/IP stack — no checksums,
no routing, no packet headers.

```go
// unix_socket_server.go
//
// Demonstrates Unix domain sockets — both stream (SOCK_STREAM)
// and datagram (SOCK_DGRAM) modes.
//
// Unix domain sockets use filesystem paths instead of IP:port.
// They are significantly faster than TCP loopback (127.0.0.1)
// because they bypass the entire network stack.
//
// Common uses:
//   - Docker daemon (/var/run/docker.sock)
//   - MySQL local connections
//   - systemd socket activation
//   - Kubernetes CRI (/var/run/containerd/containerd.sock)
package main

import (
	"fmt"
	"log"
	"os"

	"golang.org/x/sys/unix"
)

const socketPath = "/tmp/echo.sock"

func main() {
	// Clean up any leftover socket file from a previous run.
	// Unlike TCP ports, Unix socket paths persist as filesystem entries.
	os.Remove(socketPath)

	// ──────────────────────────────────────────────────────────
	// Create a Unix stream socket.
	//
	// AF_UNIX       = Unix domain (local IPC)
	// SOCK_STREAM   = Reliable byte stream (like TCP, but local)
	//
	// The kernel creates a unix_sock structure instead of tcp_sock.
	// Data transfer happens via kernel memory copies, not the
	// network stack.
	// ──────────────────────────────────────────────────────────
	fd, err := unix.Socket(unix.AF_UNIX, unix.SOCK_STREAM, 0)
	if err != nil {
		log.Fatalf("socket() failed: %v", err)
	}
	defer unix.Close(fd)

	// Bind to a filesystem path.
	// This creates a socket file visible with `ls -l`.
	addr := &unix.SockaddrUnix{Name: socketPath}
	if err := unix.Bind(fd, addr); err != nil {
		log.Fatalf("bind() failed: %v", err)
	}

	if err := unix.Listen(fd, 128); err != nil {
		log.Fatalf("listen() failed: %v", err)
	}

	fmt.Printf("Unix socket server listening on %s\n", socketPath)

	for {
		clientFd, _, err := unix.Accept(fd)
		if err != nil {
			log.Printf("accept() error: %v", err)
			continue
		}
		go func(cfd int) {
			defer unix.Close(cfd)
			buf := make([]byte, 4096)
			for {
				n, err := unix.Read(cfd, buf)
				if err != nil || n == 0 {
					return
				}
				unix.Write(cfd, buf[:n])
			}
		}(clientFd)
	}
}
```

**Unix domain socket datagram mode:**

```go
// unix_dgram_server.go
//
// Unix domain sockets also support SOCK_DGRAM — connectionless
// message-oriented communication. Unlike UDP, Unix datagrams
// are RELIABLE (no packet loss, since it's all in-kernel memory).
//
// This is used by syslog (journald), some database protocols,
// and inter-process messaging systems.
package main

import (
	"fmt"
	"log"
	"os"

	"golang.org/x/sys/unix"
)

func main() {
	const srvPath = "/tmp/dgram_server.sock"
	os.Remove(srvPath)

	// SOCK_DGRAM with AF_UNIX — connectionless but reliable.
	fd, err := unix.Socket(unix.AF_UNIX, unix.SOCK_DGRAM, 0)
	if err != nil {
		log.Fatalf("socket() failed: %v", err)
	}
	defer unix.Close(fd)

	if err := unix.Bind(fd, &unix.SockaddrUnix{Name: srvPath}); err != nil {
		log.Fatalf("bind() failed: %v", err)
	}

	fmt.Printf("Unix datagram server listening on %s\n", srvPath)

	buf := make([]byte, 65535)
	for {
		// No listen/accept needed — it's connectionless.
		n, from, err := unix.Recvfrom(fd, buf, 0)
		if err != nil {
			log.Printf("recvfrom() error: %v", err)
			continue
		}
		fmt.Printf("Received %d bytes: %s\n", n, string(buf[:n]))

		// Reply to the sender.
		if from != nil {
			unix.Sendto(fd, buf[:n], 0, from)
		}
	}
}
```

**Performance comparison (same host):**

```
Benchmark                    Throughput     Latency
─────────────────────────    ──────────     ───────
TCP loopback (127.0.0.1)    ~6 GB/s        ~10 µs
Unix stream socket           ~12 GB/s       ~3 µs
Unix datagram socket         ~10 GB/s       ~2 µs
```

Unix domain sockets are roughly 2× faster because they skip the entire IP layer:
no routing table lookup, no checksum calculation, no packet fragmentation.

---
## 4.2 TCP Deep Dive

TCP is the backbone of the Internet. Understanding its internals is essential for
debugging production issues, tuning performance, and understanding why certain
design decisions were made.

### 4.2.1 The Three-Way Handshake at the Kernel Level

```
    Client                          Server
      |                               |
      |   ──── SYN (seq=x) ────►     |   Client sends SYN, enters SYN_SENT
      |                               |   Server receives SYN, creates SYN queue entry
      |   ◄── SYN-ACK (seq=y,        |   Server sends SYN-ACK, entry in SYN queue
      |        ack=x+1) ──           |
      |                               |
      |   ──── ACK (ack=y+1) ────►   |   Client enters ESTABLISHED
      |                               |   Server moves connection from SYN queue
      |                               |   to accept queue, enters ESTABLISHED
      |                               |
      |   ◄════ DATA TRANSFER ════►   |
```

**What happens in the kernel during the handshake:**

1. **Client calls `connect()`**: The kernel sends a SYN packet and transitions the
   client socket to `SYN_SENT` state. The kernel starts retransmission timers.

2. **Server receives SYN**: The kernel (not user code!) handles this. A new entry is
   created in the **SYN queue** (also called the incomplete connection queue). The
   kernel sends SYN-ACK. The server application doesn't even know about this yet.

3. **Client receives SYN-ACK**: The kernel sends ACK, transitions to `ESTABLISHED`.
   The `connect()` syscall returns successfully.

4. **Server receives ACK**: The kernel moves the connection from the SYN queue to
   the **accept queue** (completed connections). When the application calls
   `accept()`, it dequeues from here.

### 4.2.2 SYN Queue and Accept Queue

```
                     ┌──────────────────┐
 Incoming SYN ──────►│    SYN Queue     │
                     │ (half-open conns)│
                     └───────┬──────────┘
                             │ SYN-ACK sent,
                             │ ACK received
                             ▼
                     ┌──────────────────┐
                     │  Accept Queue    │
                     │ (completed conns)│──────► accept() returns
                     └──────────────────┘
```

**SYN queue** (half-open connections):
- Entries are connections that received SYN but haven't completed the handshake.
- Size is controlled by: `/proc/sys/net/ipv4/tcp_max_syn_backlog` (default: 128-1024)
- When full: SYN cookies are used (if enabled) or SYNs are silently dropped.

**Accept queue** (completed connections):
- Entries are fully established connections waiting for `accept()`.
- Size is controlled by: `min(backlog, /proc/sys/net/core/somaxconn)`
- When full: Depends on `/proc/sys/net/ipv4/tcp_abort_on_overflow`:
  - 0 (default): Server ignores the final ACK. Client retransmits.
  - 1: Server sends RST. Client sees "connection reset."

**This is a common production issue:** If your application is slow to call `accept()`,
the accept queue fills up and new connections are silently dropped. Monitoring:

```bash
# Show accept queue status (Recv-Q column for LISTEN sockets)
ss -ltn
# Recv-Q = current queue size, Send-Q = max queue size (backlog)
```

### 4.2.3 TCP State Machine — All 11 States

This is the complete TCP state machine. Every TCP connection traverses a subset
of these states:

```
                              ┌───────────────────┐
                              │                   │
                    ┌─────────│      CLOSED        │─────────┐
                    │         │                   │         │
                    │         └───────────────────┘         │
                    │ passive open                  active open
                    │ (listen())                    (connect())
                    │                                       │
                    ▼                                       ▼
           ┌───────────────┐                      ┌───────────────┐
           │               │                      │               │
           │    LISTEN     │                      │   SYN_SENT    │
           │               │                      │               │
           └───────┬───────┘                      └───────┬───────┘
                   │ recv SYN,                            │ recv SYN-ACK,
                   │ send SYN-ACK                         │ send ACK
                   ▼                                       ▼
           ┌───────────────┐                      ┌───────────────┐
           │               │    recv ACK          │               │
           │  SYN_RECEIVED │─────────────────────►│  ESTABLISHED  │
           │               │                      │               │
           └───────────────┘                      └───┬───────┬───┘
                                                      │       │
                                           close()    │       │ recv FIN,
                                           send FIN   │       │ send ACK
                                                      ▼       ▼
                                              ┌──────────┐ ┌──────────┐
                                              │          │ │          │
                                              │ FIN_WAIT │ │CLOSE_WAIT│
                                              │    _1    │ │          │
                                              └────┬─────┘ └────┬─────┘
                                                   │            │
                                        recv FIN+ACK│            │ close(),
                                        /recv ACK   │            │ send FIN
                                                   ▼            ▼
                                 ┌───────────┐ ┌──────────┐ ┌──────────┐
                                 │           │ │          │ │          │
                                 │ FIN_WAIT  │ │ CLOSING  │ │ LAST_ACK │
                                 │    _2     │ │          │ │          │
                                 └─────┬─────┘ └────┬─────┘ └────┬─────┘
                                       │            │            │
                                recv FIN│    recv ACK│    recv ACK│
                                send ACK│            │            │
                                       ▼            ▼            ▼
                                 ┌───────────────────┐    ┌──────────┐
                                 │                   │    │          │
                                 │    TIME_WAIT      │    │  CLOSED  │
                                 │  (2×MSL timeout)  │    │          │
                                 └─────────┬─────────┘    └──────────┘
                                           │
                                    timeout (60s)
                                           │
                                           ▼
                                 ┌───────────────────┐
                                 │      CLOSED        │
                                 └───────────────────┘
```

**All 11 TCP states explained:**

| State         | Description                                                |
|---------------|------------------------------------------------------------|
| `CLOSED`      | No connection exists                                       |
| `LISTEN`      | Server waiting for incoming connections                    |
| `SYN_SENT`    | Client sent SYN, waiting for SYN-ACK                      |
| `SYN_RECEIVED`| Server received SYN, sent SYN-ACK, waiting for ACK        |
| `ESTABLISHED` | Connection is open, data can flow both ways                |
| `FIN_WAIT_1`  | Active closer sent FIN, waiting for ACK                    |
| `FIN_WAIT_2`  | Active closer received ACK of FIN, waiting for peer's FIN  |
| `CLOSE_WAIT`  | Passive closer received FIN, waiting for application close |
| `CLOSING`     | Both sides sent FIN simultaneously (rare)                  |
| `LAST_ACK`    | Passive closer sent FIN, waiting for ACK                   |
| `TIME_WAIT`   | Active closer waiting 2×MSL before fully closing           |

### 4.2.4 TIME_WAIT and Why It Exists

`TIME_WAIT` is the most misunderstood TCP state. The active closer (the side that
sends FIN first) enters `TIME_WAIT` and stays there for 2×MSL (Maximum Segment
Lifetime, typically 60 seconds on Linux).

**Why TIME_WAIT exists:**

1. **Reliable connection termination**: If the final ACK is lost, the peer will
   retransmit its FIN. The TIME_WAIT socket can re-send the ACK.

2. **Prevent old segments from being misinterpreted**: Delayed packets from the old
   connection could arrive at a new connection on the same port. TIME_WAIT ensures
   all old packets have expired.

**TIME_WAIT in production:**

A busy server closing many short-lived connections can accumulate thousands of
TIME_WAIT sockets:

```bash
# Count TIME_WAIT sockets
ss -s | grep timewait
# Or:
cat /proc/net/sockstat

# Reduce TIME_WAIT duration (use cautiously)
echo 1 > /proc/sys/net/ipv4/tcp_tw_reuse
```

**Best practices:**
- Let the client close first (then TIME_WAIT is on the client, not the server)
- Use connection pooling (HTTP keep-alive)
- Enable `tcp_tw_reuse` for outbound connections

### 4.2.5 TCP Socket Options

#### TCP_NODELAY — Disabling Nagle's Algorithm

Nagle's algorithm buffers small writes and combines them into larger segments to
reduce the number of small packets. This is efficient for bulk transfers but
adds latency for interactive protocols.

```go
// disable_nagle.go
//
// Demonstrates setting TCP_NODELAY to disable Nagle's algorithm.
//
// Nagle's algorithm (RFC 896) works like this:
//   - If there's unacknowledged data in flight AND the new data is small
//     (less than MSS), buffer it until either:
//     a) The outstanding data is ACKed, OR
//     b) Enough data accumulates to fill a segment (MSS)
//
// This adds up to ~40ms latency (the delayed ACK timer) for small writes.
// For interactive protocols (SSH, game servers, trading systems),
// this is unacceptable.
//
// TCP_NODELAY disables Nagle: every write() immediately sends a packet.
package main

import (
	"log"

	"golang.org/x/sys/unix"
)

// setTCPNoDelay disables Nagle's algorithm on the given socket.
// After this call, every Write() will immediately result in a TCP segment,
// regardless of size. This reduces latency at the cost of more packets
// on the wire (and potentially lower throughput for bulk transfers).
func setTCPNoDelay(fd int) error {
	return unix.SetsockoptInt(
		fd,
		unix.IPPROTO_TCP, // Level: TCP protocol options
		unix.TCP_NODELAY, // Option: disable Nagle
		1,                // Value: 1 = enabled (yes, disable Nagle)
	)
}

// setTCPCork enables TCP_CORK on the given socket.
// When corked, the kernel accumulates small writes and sends them as
// a single segment when:
//   - The cork is removed (setsockopt TCP_CORK = 0)
//   - A full MSS worth of data accumulates
//   - 200ms passes
//
// This is useful for building a response in multiple write() calls
// (e.g., HTTP headers + body) and sending them as one segment.
//
// Pattern: Cork → write headers → write body → Uncork
func setTCPCork(fd int, enable bool) error {
	val := 0
	if enable {
		val = 1
	}
	return unix.SetsockoptInt(fd, unix.IPPROTO_TCP, unix.TCP_CORK, val)
}

func main() {
	fd, err := unix.Socket(unix.AF_INET, unix.SOCK_STREAM, 0)
	if err != nil {
		log.Fatal(err)
	}
	defer unix.Close(fd)

	// Disable Nagle — every write sends immediately.
	if err := setTCPNoDelay(fd); err != nil {
		log.Fatalf("TCP_NODELAY failed: %v", err)
	}
	log.Println("TCP_NODELAY set successfully")
}
```

#### SO_REUSEADDR and SO_REUSEPORT

```go
// reuseport_server.go
//
// Demonstrates SO_REUSEADDR and SO_REUSEPORT socket options.
//
// SO_REUSEADDR (common):
//   - Allows binding to an address in TIME_WAIT state
//   - Allows binding to 0.0.0.0:80 when another socket is on 192.168.1.1:80
//   - Almost always set on server sockets
//
// SO_REUSEPORT (advanced):
//   - Allows MULTIPLE sockets to bind to the SAME address:port
//   - The kernel load-balances incoming connections across all sockets
//   - Each socket typically runs in a separate thread/process
//   - Used by NGINX, Envoy, and other high-performance servers
//   - Eliminates the accept() thundering herd problem
package main

import (
	"fmt"
	"log"
	"os"
	"runtime"

	"golang.org/x/sys/unix"
)

func main() {
	numCPU := runtime.NumCPU()
	fmt.Printf("Starting %d listeners with SO_REUSEPORT\n", numCPU)

	// Create one listener per CPU core.
	// Each goroutine has its own socket, all bound to the same port.
	// The kernel distributes incoming connections across all sockets.
	for i := 0; i < numCPU; i++ {
		go func(id int) {
			fd, err := unix.Socket(unix.AF_INET, unix.SOCK_STREAM, 0)
			if err != nil {
				log.Fatalf("[%d] socket() failed: %v", id, err)
			}
			defer unix.Close(fd)

			// SO_REUSEADDR — allow bind even if TIME_WAIT sockets exist.
			unix.SetsockoptInt(fd, unix.SOL_SOCKET, unix.SO_REUSEADDR, 1)

			// SO_REUSEPORT — allow multiple sockets on the same port.
			// The kernel uses a hash of the incoming connection's
			// source IP:port to consistently route to the same socket.
			unix.SetsockoptInt(fd, unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)

			addr := &unix.SockaddrInet4{Port: 8080}
			if err := unix.Bind(fd, addr); err != nil {
				log.Fatalf("[%d] bind() failed: %v", id, err)
			}

			if err := unix.Listen(fd, 128); err != nil {
				log.Fatalf("[%d] listen() failed: %v", id, err)
			}

			fmt.Printf("[Worker %d] Listening on :8080\n", id)

			for {
				clientFd, _, err := unix.Accept(fd)
				if err != nil {
					continue
				}
				fmt.Printf("[Worker %d] Accepted connection\n", id)
				unix.Close(clientFd)
			}
		}(i)
	}

	// Block forever.
	select {}
}
```

#### TCP Keepalive

```go
// tcp_keepalive.go
//
// TCP keepalive sends periodic probes on idle connections to detect:
//  1. Dead peers (crashed without sending FIN)
//  2. Network path failures
//  3. Stateful firewall/NAT timeout (firewalls drop idle connections)
//
// Without keepalive, a TCP connection to a crashed peer will appear
// "established" forever — reads will block indefinitely.
//
// Keepalive parameters:
//
//	TCP_KEEPIDLE  — seconds of idle time before first probe (default: 7200 = 2 hours!)
//	TCP_KEEPINTVL — seconds between probes (default: 75)
//	TCP_KEEPCNT   — number of failed probes before giving up (default: 9)
//
// With defaults: 2 hours + 9 × 75s = ~2h11m before detecting a dead peer.
// In production, you want much shorter values.
package main

import (
	"log"

	"golang.org/x/sys/unix"
)

// enableKeepalive configures TCP keepalive on the given socket.
// This sends a probe every 'interval' seconds after 'idle' seconds
// of inactivity. After 'count' failed probes, the connection is
// considered dead and read()/write() will return an error.
func enableKeepalive(fd, idle, interval, count int) error {
	// Enable keepalive.
	if err := unix.SetsockoptInt(fd, unix.SOL_SOCKET, unix.SO_KEEPALIVE, 1); err != nil {
		return err
	}

	// Seconds of idle time before the first keepalive probe.
	if err := unix.SetsockoptInt(fd, unix.IPPROTO_TCP, unix.TCP_KEEPIDLE, idle); err != nil {
		return err
	}

	// Seconds between subsequent keepalive probes.
	if err := unix.SetsockoptInt(fd, unix.IPPROTO_TCP, unix.TCP_KEEPINTVL, interval); err != nil {
		return err
	}

	// Number of failed probes before declaring the connection dead.
	if err := unix.SetsockoptInt(fd, unix.IPPROTO_TCP, unix.TCP_KEEPCNT, count); err != nil {
		return err
	}

	return nil
}

func main() {
	fd, err := unix.Socket(unix.AF_INET, unix.SOCK_STREAM, 0)
	if err != nil {
		log.Fatal(err)
	}
	defer unix.Close(fd)

	// Send first probe after 10s idle, then every 5s, give up after 3 failures.
	// Total detection time: 10 + 3×5 = 25 seconds.
	if err := enableKeepalive(fd, 10, 5, 3); err != nil {
		log.Fatalf("keepalive setup failed: %v", err)
	}
	log.Println("TCP keepalive configured: idle=10s, interval=5s, count=3")
}
```

### 4.2.6 TCP Window Scaling and Congestion Control

**Window Scaling:**

The TCP receive window tells the sender how much data the receiver can buffer.
The original TCP header allows a 16-bit window (max 65535 bytes). Window scaling
(RFC 1323) adds a scale factor negotiated during the handshake:

```
Actual window = Window field × 2^(scale factor)
```

With a scale factor of 7: 65535 × 128 = ~8 MB. This is essential for
high-bandwidth, high-latency links (the bandwidth-delay product).

```bash
# Check window scaling
cat /proc/sys/net/ipv4/tcp_window_scaling  # 1 = enabled (default)

# See actual window sizes
ss -ti | grep -A2 wscale
```

**Congestion Control:**

TCP congestion control prevents the sender from overwhelming the network. Linux
supports multiple algorithms:

| Algorithm | Description                                          | When to Use          |
|-----------|------------------------------------------------------|----------------------|
| Cubic     | Default since Linux 2.6.19. Loss-based.              | General purpose      |
| BBR       | Google's algorithm. Model-based (bandwidth × RTT).   | High-latency links   |
| Reno      | Classic algorithm. Simple AIMD.                       | Legacy               |

```bash
# List available congestion control algorithms
cat /proc/sys/net/ipv4/tcp_available_congestion_control

# Check current algorithm
cat /proc/sys/net/ipv4/tcp_congestion_control

# Switch to BBR (requires kernel 4.9+)
echo bbr > /proc/sys/net/ipv4/tcp_congestion_control
```

**BBR vs Cubic:**
- Cubic reacts to packet loss (reduces window). On lossy links (WiFi), it
  under-utilizes bandwidth.
- BBR probes for available bandwidth and RTT, maintaining a model of the path.
  It achieves higher throughput on lossy or high-latency links.

### 4.2.7 Hands-On: Examining TCP States from Go

```go
// tcp_state_monitor.go
//
// Reads /proc/net/tcp to show all TCP connections and their states.
// This is the same data that `ss` and `netstat` display.
//
// /proc/net/tcp format (each line is a socket):
//
//	sl  local_address rem_address   st tx_queue rx_queue ...
//
// The "st" field is the TCP state as a hex number:
//
//	01 = ESTABLISHED    06 = TIME_WAIT
//	02 = SYN_SENT       07 = CLOSE
//	03 = SYN_RECV       08 = CLOSE_WAIT
//	04 = FIN_WAIT1      09 = LAST_ACK
//	05 = FIN_WAIT2      0A = LISTEN
//	                     0B = CLOSING
package main

import (
	"bufio"
	"encoding/hex"
	"fmt"
	"log"
	"net"
	"os"
	"strconv"
	"strings"
)

// tcpStateNames maps the hex state values from /proc/net/tcp
// to human-readable TCP state names. These correspond directly
// to the kernel's enum tcp_state values.
var tcpStateNames = map[string]string{
	"01": "ESTABLISHED",
	"02": "SYN_SENT",
	"03": "SYN_RECV",
	"04": "FIN_WAIT1",
	"05": "FIN_WAIT2",
	"06": "TIME_WAIT",
	"07": "CLOSE",
	"08": "CLOSE_WAIT",
	"09": "LAST_ACK",
	"0A": "LISTEN",
	"0B": "CLOSING",
}

// parseHexAddr converts the hex-encoded address from /proc/net/tcp
// into a human-readable "IP:port" string.
//
// /proc/net/tcp stores addresses as AABBCCDD:PORT where:
//   - AABBCCDD is the IPv4 address in little-endian hex
//   - PORT is the port number in hex
//
// Example: "0100007F:1F90" → "127.0.0.1:8080"
func parseHexAddr(s string) string {
	parts := strings.Split(s, ":")
	if len(parts) != 2 {
		return s
	}

	// Parse the hex IP address (little-endian on x86).
	hexIP := parts[0]
	if len(hexIP) != 8 {
		return s
	}
	ipBytes, err := hex.DecodeString(hexIP)
	if err != nil {
		return s
	}
	// Reverse bytes for little-endian → big-endian conversion.
	ip := net.IPv4(ipBytes[3], ipBytes[2], ipBytes[1], ipBytes[0])

	// Parse the hex port.
	port, err := strconv.ParseUint(parts[1], 16, 16)
	if err != nil {
		return s
	}

	return fmt.Sprintf("%s:%d", ip.String(), port)
}

func main() {
	// Open /proc/net/tcp — the kernel's view of all TCP sockets.
	// This is a pseudo-file generated on-the-fly by the kernel.
	file, err := os.Open("/proc/net/tcp")
	if err != nil {
		log.Fatalf("Failed to open /proc/net/tcp: %v\n"+
			"(This program must run on Linux)", err)
	}
	defer file.Close()

	scanner := bufio.NewScanner(file)

	fmt.Printf("%-6s %-25s %-25s %-15s\n",
		"Slot", "Local Address", "Remote Address", "State")
	fmt.Println(strings.Repeat("-", 75))

	lineNum := 0
	for scanner.Scan() {
		lineNum++
		if lineNum == 1 {
			continue // Skip header line
		}

		fields := strings.Fields(scanner.Text())
		if len(fields) < 4 {
			continue
		}

		slot := fields[0]
		localAddr := parseHexAddr(fields[1])
		remoteAddr := parseHexAddr(fields[2])
		state := tcpStateNames[fields[3]]

		fmt.Printf("%-6s %-25s %-25s %-15s\n",
			slot, localAddr, remoteAddr, state)
	}
}
```

### 4.2.8 Hands-On: Building a Concurrent TCP Proxy in Go

```go
// tcp_proxy.go
//
// A concurrent TCP proxy (Layer 4 forwarder) built from raw syscalls.
// This proxy sits between clients and a backend server, forwarding
// TCP streams bidirectionally.
//
// Architecture:
//
//	Client ←──TCP──→ Proxy ←──TCP──→ Backend
//
// For each client connection, the proxy:
//  1. Accepts the client connection
//  2. Opens a new connection to the backend
//  3. Spawns two goroutines to copy data in both directions
//  4. When either side closes, both connections are torn down
//
// This is essentially what HAProxy, Envoy, and kube-proxy do at the
// TCP level (though they add health checking, load balancing, etc.)
//
// Usage:
//
//	go run tcp_proxy.go -listen :8080 -backend 127.0.0.1:3000
package main

import (
	"flag"
	"fmt"
	"io"
	"log"
	"net"
	"sync"
	"sync/atomic"
)

var (
	listenAddr  = flag.String("listen", ":8080", "Address to listen on")
	backendAddr = flag.String("backend", "127.0.0.1:3000", "Backend address to proxy to")
)

// activeConns tracks the number of active proxy connections.
// We use atomic operations for lock-free concurrent access.
var activeConns int64

// proxyConnection handles a single proxied connection.
// It opens a connection to the backend and bidirectionally copies
// data between client and backend.
//
// The io.Copy calls block until one side closes. We use a WaitGroup
// to ensure both directions are fully copied before closing.
func proxyConnection(clientConn net.Conn) {
	atomic.AddInt64(&activeConns, 1)
	defer func() {
		atomic.AddInt64(&activeConns, -1)
		clientConn.Close()
	}()

	// Connect to the backend.
	backendConn, err := net.Dial("tcp", *backendAddr)
	if err != nil {
		log.Printf("Backend connection failed: %v", err)
		return
	}
	defer backendConn.Close()

	log.Printf("Proxying %s → %s (active: %d)",
		clientConn.RemoteAddr(), *backendAddr, atomic.LoadInt64(&activeConns))

	// Bidirectional copy using two goroutines.
	// Each goroutine copies data in one direction until EOF or error.
	var wg sync.WaitGroup
	wg.Add(2)

	// Client → Backend
	go func() {
		defer wg.Done()
		n, _ := io.Copy(backendConn, clientConn)
		// Half-close: signal to backend that client is done sending.
		// This sends a FIN to the backend.
		if tc, ok := backendConn.(*net.TCPConn); ok {
			tc.CloseWrite()
		}
		log.Printf("Client → Backend: %d bytes", n)
	}()

	// Backend → Client
	go func() {
		defer wg.Done()
		n, _ := io.Copy(clientConn, backendConn)
		if tc, ok := clientConn.(*net.TCPConn); ok {
			tc.CloseWrite()
		}
		log.Printf("Backend → Client: %d bytes", n)
	}()

	wg.Wait()
}

func main() {
	flag.Parse()

	listener, err := net.Listen("tcp", *listenAddr)
	if err != nil {
		log.Fatalf("Failed to listen on %s: %v", *listenAddr, err)
	}
	defer listener.Close()

	fmt.Printf("TCP proxy listening on %s → %s\n", *listenAddr, *backendAddr)

	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Printf("Accept error: %v", err)
			continue
		}
		go proxyConnection(conn)
	}
}
```

---

## 4.3 Raw Sockets & Packet Crafting

Raw sockets (`SOCK_RAW`) give you direct access to the IP layer. You can
construct entire packets — headers and all — and inject them into the network.

### 4.3.1 Understanding SOCK_RAW

```
Normal socket (SOCK_STREAM/SOCK_DGRAM):
  Application data → TCP/UDP header added → IP header added → sent

Raw socket (SOCK_RAW):
  You provide: [IP header (optional)] [Protocol header] [Payload]
  Kernel adds: [Ethernet header]
  
  With IP_HDRINCL: You build the entire IP packet
  Without IP_HDRINCL: Kernel builds the IP header, you build the rest
```

Raw sockets require `CAP_NET_RAW` capability (usually root access).

### 4.3.2 Hands-On: Building a Ping Utility in Go

```go
// raw_ping.go
//
// A ping utility built from scratch using raw sockets.
// This demonstrates SOCK_RAW and manual ICMP packet construction.
//
// ICMP (Internet Control Message Protocol) operates at the IP layer.
// Ping sends ICMP Echo Request packets and listens for Echo Reply.
//
// ICMP Echo Request packet format:
//
//	┌──────┬──────┬──────────────────┐
//	│ Type │ Code │    Checksum      │  (4 bytes)
//	│  8   │  0   │                  │
//	├──────┴──────┼──────────────────┤
//	│ Identifier  │ Sequence Number  │  (4 bytes)
//	├─────────────┴──────────────────┤
//	│          Payload               │  (variable)
//	└────────────────────────────────┘
//
// ICMP Echo Reply has Type=0, same Identifier and Sequence.
//
// Usage:
//
//	sudo go run raw_ping.go 8.8.8.8
//
// Note: Requires root or CAP_NET_RAW capability.
package main

import (
	"encoding/binary"
	"fmt"
	"log"
	"net"
	"os"
	"time"

	"golang.org/x/sys/unix"
)

// icmpHeader represents the ICMP header fields.
// We manually construct this instead of using a library
// to show exactly what goes on the wire.
type icmpHeader struct {
	Type     uint8  // 8 for Echo Request, 0 for Echo Reply
	Code     uint8  // 0 for Echo
	Checksum uint16 // Internet checksum of the ICMP message
	ID       uint16 // Identifier (to match requests with replies)
	Seq      uint16 // Sequence number
}

// marshalICMP serializes an ICMP echo request packet.
// The checksum is calculated over the entire ICMP message
// (header + payload) using the standard Internet checksum algorithm.
func marshalICMP(typ uint8, code uint8, id uint16, seq uint16, payload []byte) []byte {
	// Build the packet: 8-byte header + payload.
	pkt := make([]byte, 8+len(payload))
	pkt[0] = typ
	pkt[1] = code
	// Checksum field starts at 0 for calculation.
	pkt[2] = 0
	pkt[3] = 0
	binary.BigEndian.PutUint16(pkt[4:6], id)
	binary.BigEndian.PutUint16(pkt[6:8], seq)
	copy(pkt[8:], payload)

	// Calculate the Internet checksum.
	// This is the one's complement of the one's complement sum
	// of all 16-bit words in the message.
	cs := internetChecksum(pkt)
	binary.BigEndian.PutUint16(pkt[2:4], cs)

	return pkt
}

// internetChecksum computes the RFC 1071 Internet checksum.
// This algorithm is used by IP, TCP, UDP, and ICMP headers.
// It adds all 16-bit words, folds carries, and returns the complement.
func internetChecksum(data []byte) uint16 {
	var sum uint32
	length := len(data)

	// Sum all 16-bit words.
	for i := 0; i+1 < length; i += 2 {
		sum += uint32(data[i])<<8 | uint32(data[i+1])
	}
	// If odd length, add the last byte padded with zero.
	if length%2 == 1 {
		sum += uint32(data[length-1]) << 8
	}

	// Fold 32-bit sum to 16 bits: add the carry bits.
	for sum>>16 != 0 {
		sum = (sum & 0xFFFF) + (sum >> 16)
	}

	return uint16(^sum)
}

func main() {
	if len(os.Args) < 2 {
		fmt.Fprintf(os.Stderr, "Usage: %s <host>\n", os.Args[0])
		os.Exit(1)
	}

	target := os.Args[1]

	// Resolve the target to an IP address.
	addrs, err := net.LookupHost(target)
	if err != nil {
		log.Fatalf("DNS lookup failed: %v", err)
	}
	destIP := net.ParseIP(addrs[0]).To4()
	if destIP == nil {
		log.Fatalf("Not an IPv4 address: %s", addrs[0])
	}

	// ──────────────────────────────────────────────────────
	// Create a raw socket for ICMP.
	//
	// AF_INET    = IPv4
	// SOCK_RAW   = Raw socket (access to IP layer)
	// IPPROTO_ICMP = We want to send/receive ICMP packets
	//
	// The kernel handles the IP header for us (unless we set
	// IP_HDRINCL). We only need to construct the ICMP portion.
	// ──────────────────────────────────────────────────────
	fd, err := unix.Socket(unix.AF_INET, unix.SOCK_RAW, unix.IPPROTO_ICMP)
	if err != nil {
		log.Fatalf("socket(SOCK_RAW) failed: %v\n"+
			"Hint: Run with sudo or set CAP_NET_RAW", err)
	}
	defer unix.Close(fd)

	// Set a receive timeout so we don't block forever.
	tv := unix.Timeval{Sec: 2, Usec: 0}
	unix.SetsockoptTimeval(fd, unix.SOL_SOCKET, unix.SO_RCVTIMEO, &tv)

	dest := &unix.SockaddrInet4{
		Addr: [4]byte{destIP[0], destIP[1], destIP[2], destIP[3]},
	}

	fmt.Printf("PING %s (%s)\n", target, destIP.String())

	// Send 4 pings.
	for seq := uint16(1); seq <= 4; seq++ {
		// Construct the ICMP Echo Request.
		// Type=8 (Echo Request), Code=0, ID=process PID, Seq=sequence number.
		payload := []byte("HELLO-FROM-GO-RAW-PING")
		pkt := marshalICMP(8, 0, uint16(os.Getpid()&0xFFFF), seq, payload)

		// Record the send time for RTT calculation.
		start := time.Now()

		// Send the ICMP packet.
		if err := unix.Sendto(fd, pkt, 0, dest); err != nil {
			log.Printf("sendto() failed: %v", err)
			continue
		}

		// Receive the reply.
		// The kernel delivers the entire IP packet (IP header + ICMP message).
		// We need to skip the IP header (typically 20 bytes) to get to ICMP.
		recvBuf := make([]byte, 1500)
		n, _, err := unix.Recvfrom(fd, recvBuf, 0)
		if err != nil {
			log.Printf("Request timeout for seq %d", seq)
			continue
		}

		rtt := time.Since(start)

		// Parse the IP header to find its length.
		// The first nibble of byte 0 is the IP version (4).
		// The second nibble is the header length in 32-bit words.
		if n < 20 {
			continue
		}
		ipHdrLen := int(recvBuf[0]&0x0F) * 4 // IHL field × 4 bytes

		// Parse the ICMP reply (after the IP header).
		if n < ipHdrLen+8 {
			continue
		}
		icmpType := recvBuf[ipHdrLen]
		if icmpType == 0 { // Echo Reply
			replySeq := binary.BigEndian.Uint16(recvBuf[ipHdrLen+6 : ipHdrLen+8])
			fmt.Printf("%d bytes from %s: seq=%d time=%v\n",
				n-ipHdrLen, destIP, replySeq, rtt.Round(time.Microsecond))
		}

		time.Sleep(time.Second)
	}
}
```

---

## 4.4 The Linux Network Stack

Understanding the journey of a packet through the kernel is essential for
performance tuning, debugging, and understanding tools like eBPF and XDP.

### 4.4.1 Packet Journey: NIC to Application

```
  ┌─────────────────────────────────────────────────────────────────────┐
  │                        HARDWARE / NIC                               │
  │  Packet arrives on wire → DMA to ring buffer → Interrupt (IRQ)     │
  └─────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                        DEVICE DRIVER                                │
  │  IRQ handler → schedule NAPI poll                                  │
  │  NAPI poll → read packets from ring buffer → alloc sk_buff         │
  │  → netif_receive_skb()                                             │
  └─────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                     GENERIC RECEIVE OFFLOAD (GRO)                   │
  │  Coalesce multiple small packets into fewer large ones              │
  │  (Reduces per-packet processing overhead)                           │
  └─────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                     NETFILTER / IPTABLES                            │
  │  PREROUTING → routing decision → INPUT / FORWARD                   │
  │  (Packet filtering, NAT, connection tracking)                       │
  └─────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                     PROTOCOL HANDLER                                │
  │  IP layer: validate header, reassemble fragments                    │
  │  TCP layer: find matching socket, process segment                   │
  │           → update state machine, manage window, ACK               │
  │           → copy payload to socket receive buffer                   │
  └─────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                     SOCKET BUFFER                                    │
  │  Data sits in sk_buff chain until application calls read()          │
  │  (Wake up any sleeping reader)                                      │
  └─────────────────────────────────────┬───────────────────────────────┘
                                        │
                                        ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │                     APPLICATION (User Space)                        │
  │  read() / recv() → copy from kernel buffer to user buffer          │
  └─────────────────────────────────────────────────────────────────────┘
```

### 4.4.2 sk_buff — The Core Networking Data Structure

The `sk_buff` (socket buffer) is THE fundamental data structure in Linux networking.
Every packet is represented as an `sk_buff`:

```c
// Simplified struct sk_buff (actual struct has ~200 fields)
struct sk_buff {
    // Linked list pointers (packets are chained).
    struct sk_buff *next, *prev;
    
    // The socket this packet belongs to.
    struct sock *sk;
    
    // Network device this packet arrived on / will be sent from.
    struct net_device *dev;
    
    // Pointers into the packet data:
    //   head → the start of the allocated buffer
    //   data → the start of the current protocol header
    //   tail → the end of the payload
    //   end  → the end of the allocated buffer
    //
    //   ┌───────────────────────────────────────────┐
    //   │ head    │ data       │ tail   │ end       │
    //   │ (alloc  │ (current   │ (data  │ (alloc    │
    //   │  start) │  header)   │  end)  │  end)     │
    //   └─────────┴────────────┴────────┴───────────┘
    //   │← headroom →│← packet data →│← tailroom →│
    unsigned char *head, *data, *tail, *end;
    
    // Protocol-specific header pointers.
    // These are set as the packet is parsed layer by layer.
    union {
        struct tcphdr  *th;   // TCP header
        struct udphdr  *uh;   // UDP header
        struct icmphdr *icmph; // ICMP header
        struct iphdr   *iph;  // IP header
    } h;
    
    unsigned int len;      // Total packet length
    __u8 protocol;         // IP protocol (TCP=6, UDP=17, ICMP=1)
};
```

As a packet moves through the stack, the `data` pointer is adjusted:

```
  Ethernet frame:  ┌──────────┬──────────┬──────────┬─────────┐
                   │ Eth hdr  │ IP hdr   │ TCP hdr  │ Payload │
                   └──────────┴──────────┴──────────┴─────────┘
                   ↑ data (at NIC driver)
  
  After eth_type_trans():
                   ┌──────────┬──────────┬──────────┬─────────┐
                   │ Eth hdr  │ IP hdr   │ TCP hdr  │ Payload │
                   └──────────┴──────────┴──────────┴─────────┘
                              ↑ data (at IP layer)
  
  After ip_rcv():
                   ┌──────────┬──────────┬──────────┬─────────┐
                   │ Eth hdr  │ IP hdr   │ TCP hdr  │ Payload │
                   └──────────┴──────────┴──────────┴─────────┘
                                         ↑ data (at TCP layer)
```

### 4.4.3 NAPI — Interrupt Mitigation

Traditional packet processing: one interrupt per packet. At 10 Gbps (~15 million
packets/second), this would overwhelm any CPU.

**NAPI (New API)** solves this with an interrupt-poll hybrid:

1. First packet: NIC raises a hardware interrupt.
2. The IRQ handler **disables further interrupts** and schedules a NAPI poll.
3. The poll function processes packets from the ring buffer **without interrupts**.
4. When the ring buffer is empty (or budget is exhausted), re-enable interrupts.

```
High traffic:  IRQ → [poll poll poll poll poll ... ] → IRQ → [poll poll ...]
                     (no interrupts, just polling)

Low traffic:   IRQ → [poll] → IRQ → [poll] → ...
                     (reverts to interrupt-driven)
```

This is adaptive: under high load, it approaches pure polling (like DPDK).
Under low load, it's interrupt-driven (low latency, no CPU waste).

### 4.4.4 GRO and GSO — Offloading

**GRO (Generic Receive Offload):**
Coalesces multiple received packets into fewer larger packets before passing
them up the stack. This reduces per-packet overhead (fewer sk_buff allocations,
fewer protocol header parsings).

**GSO (Generic Segmentation Offload):**
Delays segmentation of large outbound packets as long as possible. The
application can write 64KB; the kernel keeps it as one large buffer and only
segments it at the last moment (or lets the NIC hardware do it via TSO).

```
Without GRO:  [pkt1] [pkt2] [pkt3] [pkt4] → 4 × sk_buff → 4 × protocol parse
With GRO:     [pkt1] [pkt2] [pkt3] [pkt4] → 1 × sk_buff → 1 × protocol parse
```

### 4.4.5 Hands-On: Tracing Packet Flow

While we can't use ftrace directly from Go, understanding the tracing points
is invaluable for debugging:

```bash
# Trace packet receive path using ftrace
echo 1 > /sys/kernel/debug/tracing/events/net/netif_receive_skb/enable
cat /sys/kernel/debug/tracing/trace_pipe

# Trace TCP events
echo 1 > /sys/kernel/debug/tracing/events/tcp/tcp_probe/enable

# Using bpftrace (if available)
# Count packets per second by protocol:
bpftrace -e 'tracepoint:net:netif_receive_skb {
    @proto[args->protocol] = count();
}'

# Trace TCP retransmissions:
bpftrace -e 'tracepoint:tcp:tcp_retransmit_skb {
    printf("retransmit: %s:%d -> %s:%d\n",
        ntop(args->saddr), args->sport,
        ntop(args->daddr), args->dport);
}'
```

---

## 4.5 Netfilter & iptables

Netfilter is the Linux kernel framework for packet filtering, NAT, and mangling.
`iptables` is the traditional userspace tool to configure Netfilter rules. Understanding
Netfilter is essential because it's the foundation of Linux firewalls, Docker networking,
and Kubernetes service routing.

### 4.5.1 Netfilter Hooks

Netfilter provides five hooks where kernel modules can register callback functions
to process packets:

```
                                    ┌────────────┐
                                    │  Routing   │
                                    │  Decision  │
                                    └──────┬─────┘
                                           │
    ┌──────────┐       ┌──────────┐        │         ┌──────────┐
    │          │       │          │   ┌────┴────┐    │          │
───►│PREROUTING├──────►│  ROUTE   ├──►│  INPUT  ├───►│ Local    │
    │          │       │          │   └─────────┘    │ Process  │
    └──────────┘       └────┬─────┘                  └────┬─────┘
                            │                              │
                       ┌────┴─────┐                  ┌─────┴────┐
                       │ FORWARD  │                  │  OUTPUT  │
                       └────┬─────┘                  └────┬─────┘
                            │                              │
                            ▼                              ▼
                       ┌──────────┐                  ┌──────────┐
                       │POSTROUTING│◄─────────────────┤POSTROUTING│
                       └────┬─────┘                  └──────────┘
                            │
                            ▼
                       Out to network
```

**Simplified packet flow:**

```
Incoming packet destined for this host:
  PREROUTING → INPUT → Local Process

Incoming packet to be forwarded:
  PREROUTING → FORWARD → POSTROUTING → Out

Locally generated packet:
  Local Process → OUTPUT → POSTROUTING → Out
```

### 4.5.2 iptables Tables and Chains

iptables organizes rules into **tables**, each containing **chains**:

| Table    | Purpose                          | Chains                                |
|----------|----------------------------------|---------------------------------------|
| `raw`    | Bypass connection tracking       | PREROUTING, OUTPUT                    |
| `mangle` | Packet header modification       | All 5 hooks                           |
| `nat`    | Network Address Translation      | PREROUTING, INPUT, OUTPUT, POSTROUTING|
| `filter` | Packet filtering (accept/drop)   | INPUT, FORWARD, OUTPUT                |

**Chain traversal order for incoming packets:**

```
Packet arrives at NIC
         │
         ▼
┌─────────────────────────────────────────────────────┐
│                   PREROUTING                         │
│  raw → conntrack → mangle → nat (DNAT)              │
└─────────────────────┬───────────────────────────────┘
                      │
                ┌─────┴──────┐
                │  Routing   │
                │  Decision  │
                └──┬──────┬──┘
                   │      │
          For us   │      │  For another host
                   ▼      ▼
┌──────────────────┐  ┌──────────────────┐
│     INPUT        │  │    FORWARD       │
│ mangle → filter  │  │ mangle → filter  │
└────────┬─────────┘  └────────┬─────────┘
         │                     │
         ▼                     ▼
   Local Process        ┌──────────────────────────────┐
                        │         POSTROUTING           │
                        │  mangle → nat (SNAT/MASQ)     │
                        └───────────────┬──────────────┘
                                        │
                                        ▼
                                   Out to network
```

### 4.5.3 Connection Tracking (conntrack)

Connection tracking is the stateful firewall engine. It tracks every connection
(even UDP "connections") and allows rules to match on connection state:

```bash
# View the connection tracking table
conntrack -L

# Example output:
# tcp  6 431967 ESTABLISHED src=10.0.0.1 dst=93.184.216.34 sport=54321 dport=80
#   packets=12 bytes=1500  src=93.184.216.34 dst=10.0.0.1 sport=80 dport=54321
#   packets=10 bytes=8000 [ASSURED] mark=0 use=1

# Show conntrack statistics
conntrack -S
```

**Connection states:**

| State       | Description                                            |
|-------------|--------------------------------------------------------|
| `NEW`       | First packet of a connection (SYN for TCP)             |
| `ESTABLISHED`| Packets in both directions have been seen              |
| `RELATED`   | Related to an existing connection (e.g., FTP data)     |
| `INVALID`   | Doesn't match any known connection                     |

**Typical stateful firewall rules:**

```bash
# Allow established and related connections (this is the performance shortcut —
# once a connection is tracked, subsequent packets skip most rule evaluation)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow new SSH connections
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j ACCEPT

# Drop everything else
iptables -A INPUT -j DROP
```

### 4.5.4 NAT — Network Address Translation

#### DNAT (Destination NAT) — Port Forwarding

```bash
# Forward port 80 on the host to port 8080 on 192.168.1.100
# This rule modifies the DESTINATION address of incoming packets.
iptables -t nat -A PREROUTING -p tcp --dport 80 \
    -j DNAT --to-destination 192.168.1.100:8080

# You also need to allow the forwarded traffic
iptables -A FORWARD -p tcp -d 192.168.1.100 --dport 8080 -j ACCEPT
```

#### SNAT (Source NAT) — Outbound NAT

```bash
# Change the source address of outgoing packets from 192.168.1.0/24
# to the host's public IP (10.0.0.1).
# This is what your home router does.
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 \
    -j SNAT --to-source 10.0.0.1

# MASQUERADE — Like SNAT but automatically uses the outgoing interface's IP.
# Use this when the public IP is dynamic (DHCP).
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE
```

### 4.5.5 Hands-On: NAT Setup Example

```bash
#!/bin/bash
# nat_gateway.sh
#
# Set up a Linux machine as a NAT gateway.
# This allows machines on the private network (192.168.1.0/24)
# to access the internet through this machine.
#
# This is exactly what Docker does for container networking!

# Enable IP forwarding — allows the kernel to forward packets
# between interfaces (required for routing/NAT).
echo 1 > /proc/sys/net/ipv4/ip_forward

# Flush existing rules.
iptables -t nat -F
iptables -F FORWARD

# MASQUERADE outgoing traffic from the private network.
# Packets leaving eth0 will have their source IP rewritten
# to eth0's IP address.
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE

# Allow forwarded traffic for established connections.
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow new outbound connections from the private network.
iptables -A FORWARD -s 192.168.1.0/24 -o eth0 -j ACCEPT

# Drop all other forwarded traffic.
iptables -A FORWARD -j DROP

echo "NAT gateway configured"
echo "Private network: 192.168.1.0/24 → Internet via eth0"
```

### 4.5.6 Examining the Conntrack Table

```bash
# List all tracked connections
conntrack -L

# Watch connections in real-time
conntrack -E

# Show only TCP connections in ESTABLISHED state
conntrack -L -p tcp --state ESTABLISHED

# Count connections by state
conntrack -L -p tcp 2>/dev/null | awk '{print $4}' | sort | uniq -c | sort -rn

# Check conntrack table size and usage
cat /proc/sys/net/netfilter/nf_conntrack_max    # Max entries
cat /proc/sys/net/netfilter/nf_conntrack_count  # Current entries

# If the table fills up, NEW packets are dropped!
# Increase the limit for busy servers:
echo 262144 > /proc/sys/net/netfilter/nf_conntrack_max
```

### 4.5.7 Why This Matters for Kubernetes

Kubernetes Service routing depends heavily on iptables (or IPVS):

```
When a pod sends a packet to a ClusterIP Service (e.g., 10.96.0.1:80):

1. The packet hits the OUTPUT chain.
2. kube-proxy has installed DNAT rules in the nat table:
   -A KUBE-SERVICES -d 10.96.0.1/32 -p tcp --dport 80 -j KUBE-SVC-XXXX
   -A KUBE-SVC-XXXX -m statistic --mode random --probability 0.333 -j KUBE-SEP-AAA
   -A KUBE-SVC-XXXX -m statistic --mode random --probability 0.500 -j KUBE-SEP-BBB
   -A KUBE-SVC-XXXX -j KUBE-SEP-CCC
   -A KUBE-SEP-AAA -j DNAT --to-destination 10.244.1.5:8080
   -A KUBE-SEP-BBB -j DNAT --to-destination 10.244.2.7:8080
   -A KUBE-SEP-CCC -j DNAT --to-destination 10.244.3.9:8080
   
3. The destination is rewritten to a random pod IP.
4. Conntrack remembers the mapping so replies are correctly un-NATed.
```

**Problems with iptables at scale:**
- Rules are evaluated linearly. 10,000 services = 10,000+ rules.
- Updating rules requires rewriting the entire table (O(n) updates).
- This is why large clusters use **IPVS mode** (hash-based, O(1) lookup) or
  **eBPF** (Cilium — bypasses iptables entirely).

---

## 4.6 nftables — The iptables Successor

`nftables` replaces `iptables` (and `ip6tables`, `arptables`, `ebtables`) with a
single, cleaner framework. It's been in the kernel since 3.13 (2014).

### 4.6.1 Why nftables?

| Feature           | iptables          | nftables              |
|-------------------|-------------------|-----------------------|
| Syntax            | Complex, verbose  | Cleaner, consistent   |
| Atomic updates    | No (full replace) | Yes (transactions)    |
| IPv4/IPv6         | Separate tools    | Unified               |
| Performance       | Linear matching   | Set-based matching    |
| Rule format       | Fixed             | Programmable (maps)   |

### 4.6.2 nft Syntax Basics

```bash
# Create a table (replaces -t filter, -t nat, etc.)
nft add table inet my_filter

# Create a chain (replaces -A INPUT, etc.)
# 'type filter hook input priority 0' = attach to the input hook
nft add chain inet my_filter input \
    '{ type filter hook input priority 0; policy drop; }'

# Add rules
nft add rule inet my_filter input \
    ct state established,related accept

nft add rule inet my_filter input \
    tcp dport 22 accept

nft add rule inet my_filter input \
    tcp dport { 80, 443 } accept

# Use sets for efficient matching (hash-based, not linear)
nft add set inet my_filter allowed_ports '{ type inet_service; }'
nft add element inet my_filter allowed_ports '{ 22, 80, 443, 8080 }'
nft add rule inet my_filter input \
    tcp dport @allowed_ports accept

# List all rules
nft list ruleset

# Atomic replacement (no window of vulnerability)
nft -f /etc/nftables.conf
```

**Key advantage: Atomic updates.** With iptables, updating rules requires
flushing and rebuilding the entire chain — there's a brief window where no
rules apply. nftables supports transactional updates.

---

## 4.7 Network Namespaces

Network namespaces are the foundation of container networking. Each namespace
has its own network stack: interfaces, routing table, iptables rules, and
sockets. A process in one namespace cannot see or communicate with interfaces
in another namespace (unless explicitly connected).

### 4.7.1 Creating Network Namespaces

```bash
# Create two network namespaces
ip netns add ns1
ip netns add ns2

# List namespaces
ip netns list

# Run a command inside a namespace
ip netns exec ns1 ip addr    # Shows only the loopback interface

# Each namespace starts with only a loopback interface (down by default)
ip netns exec ns1 ip link set lo up
```

### 4.7.2 veth Pairs — Virtual Ethernet

A veth (virtual Ethernet) pair is like a virtual cable with two ends. Whatever
enters one end comes out the other. They're used to connect network namespaces.

```bash
# Create a veth pair: veth0 ←→ veth1
ip link add veth0 type veth peer name veth1

# Move each end into a different namespace
ip link set veth0 netns ns1
ip link set veth1 netns ns2

# Assign IP addresses
ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth0
ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth1

# Bring interfaces up
ip netns exec ns1 ip link set veth0 up
ip netns exec ns2 ip link set veth1 up

# Now they can communicate!
ip netns exec ns1 ping 10.0.0.2
```

### 4.7.3 Bridges — Connecting Multiple Namespaces

A bridge is a virtual Layer 2 switch. It connects multiple veth pairs so
namespaces can communicate in a hub-like topology.

```
┌─────────────┐          ┌─────────┐          ┌─────────────┐
│     ns1     │          │  Host   │          │     ns2     │
│  10.0.0.1   │          │         │          │  10.0.0.2   │
│   veth0     │──────────│  br0    │──────────│   veth2     │
│             │  veth1   │ (bridge)│  veth3   │             │
└─────────────┘          └─────────┘          └─────────────┘
```

### 4.7.4 Hands-On: Complete Namespace Setup with Bridge

```bash
#!/bin/bash
# netns_bridge.sh
#
# Create two network namespaces connected via a bridge.
# This is essentially what Docker does for bridge networking.

set -e

# Clean up any previous state.
ip netns del ns1 2>/dev/null || true
ip netns del ns2 2>/dev/null || true
ip link del br0 2>/dev/null || true

# ─── Step 1: Create the bridge ───────────────────────────
# A bridge is a virtual Layer 2 switch.
ip link add br0 type bridge
ip addr add 10.0.0.254/24 dev br0
ip link set br0 up

# ─── Step 2: Create namespaces ───────────────────────────
ip netns add ns1
ip netns add ns2

# ─── Step 3: Create veth pairs ───────────────────────────
# veth0 ←→ veth0-br (ns1's connection to the bridge)
ip link add veth0 type veth peer name veth0-br
# veth1 ←→ veth1-br (ns2's connection to the bridge)
ip link add veth1 type veth peer name veth1-br

# ─── Step 4: Move veth ends into namespaces ──────────────
ip link set veth0 netns ns1
ip link set veth1 netns ns2

# ─── Step 5: Attach bridge ends to the bridge ────────────
ip link set veth0-br master br0
ip link set veth1-br master br0
ip link set veth0-br up
ip link set veth1-br up

# ─── Step 6: Configure addresses inside namespaces ───────
ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth0
ip netns exec ns1 ip link set veth0 up
ip netns exec ns1 ip link set lo up
ip netns exec ns1 ip route add default via 10.0.0.254

ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth1
ip netns exec ns2 ip link set veth1 up
ip netns exec ns2 ip link set lo up
ip netns exec ns2 ip route add default via 10.0.0.254

# ─── Step 7: Test connectivity ───────────────────────────
echo "=== ns1 → ns2 ==="
ip netns exec ns1 ping -c 2 10.0.0.2

echo "=== ns2 → ns1 ==="
ip netns exec ns2 ping -c 2 10.0.0.1

echo "=== ns1 → bridge (host) ==="
ip netns exec ns1 ping -c 2 10.0.0.254

echo "Network namespaces connected successfully!"
echo "  ns1: 10.0.0.1"
echo "  ns2: 10.0.0.2"
echo "  bridge: 10.0.0.254"
```

### 4.7.5 Hands-On: Network Namespaces from Go Using Netlink

```go
// netns_setup.go
//
// Programmatically creates network namespaces, veth pairs, and bridges
// using the vishvananda/netlink library — the Go equivalent of the `ip` command.
//
// This is what container runtimes (containerd, CRI-O) and CNI plugins
// (Calico, Flannel, Cilium) do to set up container networking.
//
// Dependencies:
//
//	go get github.com/vishvananda/netlink
//	go get github.com/vishvananda/netns
//
// Must run as root.
package main

import (
	"fmt"
	"log"
	"net"
	"runtime"

	"github.com/vishvananda/netlink"
	"github.com/vishvananda/netns"
)

// createVethPair creates a veth pair with the given names.
// A veth pair is like a virtual cable — packets sent to one end
// appear at the other end. This is the fundamental building block
// for connecting network namespaces.
func createVethPair(name1, name2 string) error {
	veth := &netlink.Veth{
		LinkAttrs: netlink.LinkAttrs{Name: name1},
		PeerName:  name2,
	}
	return netlink.LinkAdd(veth)
}

// moveToNamespace moves a network interface into a network namespace.
// After this call, the interface is no longer visible in the current namespace.
// This is equivalent to: ip link set <ifname> netns <ns>
func moveToNamespace(ifName string, nsHandle netns.NsHandle) error {
	link, err := netlink.LinkByName(ifName)
	if err != nil {
		return fmt.Errorf("interface %s not found: %w", ifName, err)
	}
	return netlink.LinkSetNsFd(link, int(nsHandle))
}

// configureInterface assigns an IP address and brings up an interface.
// This must be called from within the target namespace.
func configureInterface(ifName string, cidr string) error {
	link, err := netlink.LinkByName(ifName)
	if err != nil {
		return fmt.Errorf("interface %s not found: %w", ifName, err)
	}

	addr, err := netlink.ParseAddr(cidr)
	if err != nil {
		return fmt.Errorf("invalid CIDR %s: %w", cidr, err)
	}

	if err := netlink.AddrAdd(link, addr); err != nil {
		return fmt.Errorf("failed to add address: %w", err)
	}

	return netlink.LinkSetUp(link)
}

func main() {
	// Lock OS thread — network namespace operations are per-thread.
	// Go's goroutine scheduler can move goroutines between OS threads,
	// which would cause namespace confusion. LockOSThread prevents this.
	runtime.LockOSThread()
	defer runtime.UnlockOSThread()

	// Save the current (host) namespace so we can return to it.
	hostNS, err := netns.Get()
	if err != nil {
		log.Fatalf("Failed to get host namespace: %v", err)
	}
	defer hostNS.Close()

	// ─── Create a new network namespace ──────────────────
	// This is equivalent to: ip netns add my_container
	newNS, err := netns.New()
	if err != nil {
		log.Fatalf("Failed to create namespace: %v", err)
	}
	defer newNS.Close()

	// Switch back to the host namespace to create the veth pair.
	netns.Set(hostNS)

	// ─── Create veth pair ────────────────────────────────
	fmt.Println("Creating veth pair: veth-host ←→ veth-container")
	if err := createVethPair("veth-host", "veth-container"); err != nil {
		log.Fatalf("Failed to create veth pair: %v", err)
	}

	// Move one end into the new namespace.
	fmt.Println("Moving veth-container into new namespace")
	if err := moveToNamespace("veth-container", newNS); err != nil {
		log.Fatalf("Failed to move interface: %v", err)
	}

	// Configure the host end.
	fmt.Println("Configuring veth-host: 10.200.0.1/24")
	if err := configureInterface("veth-host", "10.200.0.1/24"); err != nil {
		log.Fatalf("Failed to configure host interface: %v", err)
	}

	// Switch to the new namespace to configure the container end.
	netns.Set(newNS)

	fmt.Println("Configuring veth-container: 10.200.0.2/24")
	if err := configureInterface("veth-container", "10.200.0.2/24"); err != nil {
		log.Fatalf("Failed to configure container interface: %v", err)
	}

	// Bring up loopback.
	lo, _ := netlink.LinkByName("lo")
	netlink.LinkSetUp(lo)

	// Add default route.
	gw := net.ParseIP("10.200.0.1")
	route := &netlink.Route{
		Gw: gw,
	}
	if err := netlink.RouteAdd(route); err != nil {
		log.Printf("Warning: failed to add default route: %v", err)
	}

	// List interfaces in the new namespace.
	links, _ := netlink.LinkList()
	fmt.Println("\nInterfaces in new namespace:")
	for _, link := range links {
		addrs, _ := netlink.AddrList(link, netlink.FAMILY_V4)
		for _, addr := range addrs {
			fmt.Printf("  %s: %s\n", link.Attrs().Name, addr.IPNet)
		}
	}

	// Switch back to host namespace.
	netns.Set(hostNS)
	fmt.Println("\nNamespace setup complete!")
	fmt.Println("From host: ping 10.200.0.2")
}
```

---

## 4.8 Advanced: eBPF for Networking

eBPF (extended Berkeley Packet Filter) is arguably the most important Linux kernel
technology of the past decade. It allows running sandboxed programs inside the kernel
without modifying kernel source code or loading kernel modules.

### 4.8.1 What is eBPF?

```
Traditional approach:
  Want new kernel feature? → Modify kernel source → Recompile → Reboot
  (Slow, risky, requires kernel developer skills)

eBPF approach:
  Write eBPF program → Compile to eBPF bytecode → Load into kernel → Attach to hook
  (Fast, safe — the kernel's verifier ensures safety)
```

**eBPF programs are:**
- Written in restricted C (or generated from Go/Rust/Python)
- Compiled to eBPF bytecode
- Verified by the kernel (guaranteed to terminate, bounded loops, no null derefs)
- JIT-compiled to native machine code for performance
- Attached to specific kernel hooks (tracepoints, kprobes, network hooks)

### 4.8.2 eBPF for Networking — The Key Hook Points

```
┌─────────────────────────────────────────────────────────────────────┐
│                        NETWORKING eBPF HOOKS                        │
│                                                                     │
│  ┌──────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐         │
│  │ XDP  │───►│  TC      │───►│ Netfilter│───►│ Socket   │         │
│  │      │    │ (ingress/│    │  hooks   │    │  hooks   │         │
│  │Driver│    │  egress) │    │          │    │(sockops, │         │
│  │level │    │          │    │          │    │ sk_msg)  │         │
│  └──────┘    └──────────┘    └──────────┘    └──────────┘         │
│     ↑              ↑              ↑              ↑                 │
│   Earliest      After          In the         Per-socket          │
│   (fastest)    sk_buff        Netfilter       level               │
│               allocation      framework                           │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.8.3 XDP — eXpress Data Path

XDP runs eBPF programs at the **earliest possible point** in the receive path —
before the kernel even allocates an `sk_buff`. This makes it incredibly fast:

```
Normal packet path:
  NIC → IRQ → NAPI → sk_buff alloc → protocol parsing → iptables → socket

XDP packet path:
  NIC → IRQ → NAPI → XDP program → (drop/redirect/pass)
                       ↑
                    No sk_buff allocation yet!
                    Can process 10-20 million pps per core
```

**XDP actions:**

| Action       | Description                                        |
|--------------|----------------------------------------------------|
| `XDP_DROP`   | Drop the packet (never enters the network stack)   |
| `XDP_PASS`   | Continue normal processing (like no XDP)           |
| `XDP_TX`     | Send the packet back out the same NIC              |
| `XDP_REDIRECT`| Send to another NIC or CPU                        |
| `XDP_ABORTED`| Error — drop and generate trace event              |

**Use cases:**
- DDoS mitigation: Drop malicious packets at line rate
- Load balancing: Facebook's Katran uses XDP for L4 load balancing
- Packet forwarding: Redirect between interfaces without entering the stack

### 4.8.4 TC (Traffic Control) eBPF Programs

TC eBPF programs run after `sk_buff` allocation, giving access to the full
packet metadata. They can be attached to both ingress and egress paths:

```bash
# Attach a TC BPF program to an interface
tc qdisc add dev eth0 clsact
tc filter add dev eth0 ingress bpf da obj my_prog.o sec classifier
tc filter add dev eth0 egress bpf da obj my_prog.o sec classifier

# TC programs have access to:
#   - Full sk_buff metadata
#   - Can modify packets (rewrite headers)
#   - Can redirect to other interfaces
#   - Can classify packets for QoS
```

### 4.8.5 Why Cilium Uses eBPF Instead of iptables

Cilium (a Kubernetes CNI plugin) replaces iptables with eBPF programs:

```
Traditional kube-proxy (iptables):
  Packet → PREROUTING → conntrack → nat rules (O(n)) → routing → FORWARD → POSTROUTING

Cilium (eBPF):
  Packet → TC/XDP eBPF program → hash lookup (O(1)) → redirect to pod

Benefits:
  - O(1) service lookup vs O(n) iptables rules
  - No conntrack overhead for pod-to-pod traffic
  - Rich Layer 7 visibility (HTTP, gRPC, Kafka)
  - Pod identity-based policies (not IP-based)
  - 10× faster than iptables at scale
```

### 4.8.6 Hands-On: eBPF Concepts and Tools

While writing eBPF programs in Go requires the `cilium/ebpf` library (a full
chapter in itself), here's how to explore eBPF capabilities:

```bash
# List all loaded eBPF programs
bpftool prog list

# Show details of a specific program
bpftool prog show id 42

# Dump the bytecode of an eBPF program
bpftool prog dump xlated id 42

# List eBPF maps (shared data structures between eBPF programs and userspace)
bpftool map list

# Dump a map's contents
bpftool map dump id 7

# Attach an XDP program to an interface
ip link set dev eth0 xdp obj xdp_drop.o sec xdp

# Detach XDP
ip link set dev eth0 xdp off

# List all tracepoints available for eBPF
bpftool feature probe

# Using bpftrace for one-liner eBPF programs:

# Count syscalls by process:
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'

# Track TCP connections:
bpftrace -e 'kprobe:tcp_connect { printf("connect: %s\n", comm); }'

# Measure TCP retransmissions:
bpftrace -e 'tracepoint:tcp:tcp_retransmit_skb {
    @retrans[ntop(args->saddr)] = count();
}'
```

**Go and eBPF (cilium/ebpf library):**

```go
// ebpf_example.go
//
// Conceptual example showing how to load and attach an eBPF program
// using the cilium/ebpf library.
//
// In practice, you'd write the eBPF program in C, compile it with
// clang/LLVM to BPF bytecode, and load it from Go.
//
// Dependencies:
//
//	go get github.com/cilium/ebpf
//
// The eBPF C program (compiled separately):
//
//	SEC("xdp")
//	int xdp_drop_icmp(struct xdp_md *ctx) {
//	    void *data = (void *)(long)ctx->data;
//	    void *data_end = (void *)(long)ctx->data_end;
//	    struct ethhdr *eth = data;
//	    if (eth + 1 > data_end) return XDP_PASS;
//	    if (eth->h_proto != htons(ETH_P_IP)) return XDP_PASS;
//	    struct iphdr *ip = (void *)(eth + 1);
//	    if (ip + 1 > data_end) return XDP_PASS;
//	    if (ip->protocol == IPPROTO_ICMP) return XDP_DROP;
//	    return XDP_PASS;
//	}
package main

import (
	"fmt"
	"log"

	"github.com/cilium/ebpf"
	"github.com/cilium/ebpf/link"
)

func main() {
	// Load the compiled eBPF object file.
	// This contains the eBPF bytecode and map definitions.
	spec, err := ebpf.LoadCollectionSpec("xdp_drop_icmp.o")
	if err != nil {
		log.Fatalf("Failed to load eBPF spec: %v", err)
	}

	// Load the program into the kernel.
	// The kernel's eBPF verifier will check the program for safety:
	//   - No unbounded loops
	//   - No null pointer dereferences
	//   - All memory accesses are bounds-checked
	//   - Program is guaranteed to terminate
	coll, err := ebpf.NewCollection(spec)
	if err != nil {
		log.Fatalf("Failed to create eBPF collection: %v", err)
	}
	defer coll.Close()

	prog := coll.Programs["xdp_drop_icmp"]
	if prog == nil {
		log.Fatal("Program 'xdp_drop_icmp' not found in eBPF object")
	}

	// Attach the XDP program to a network interface.
	// Every packet received on this interface will now pass through
	// our eBPF program BEFORE entering the normal network stack.
	xdpLink, err := link.AttachXDP(link.XDPOptions{
		Program:   prog,
		Interface: 2, // Interface index (use net.InterfaceByName to look up)
	})
	if err != nil {
		log.Fatalf("Failed to attach XDP program: %v", err)
	}
	defer xdpLink.Close()

	fmt.Println("XDP program attached! ICMP packets will be dropped.")
	fmt.Println("Press Ctrl+C to detach and exit.")

	// Block forever (the eBPF program runs in the kernel independently).
	select {}
}
```

---

## 4.9 Go's net Package Internals

Go's `net` package provides a clean, portable networking API. But understanding
what happens underneath helps you make better decisions about performance,
timeouts, and connection management.

### 4.9.1 How net.Dial Works Under the Hood

When you call `net.Dial("tcp", "example.com:80")`, here's the full call chain:

```
net.Dial("tcp", "example.com:80")
  │
  ├─► DNS Resolution
  │     ├─► Pure Go resolver (default on most platforms)
  │     │     └─► Read /etc/resolv.conf → send UDP DNS query → parse response
  │     └─► cgo resolver (when CGO_ENABLED=1 and certain conditions)
  │           └─► Call libc's getaddrinfo() via cgo
  │
  ├─► For each resolved IP address (may try multiple):
  │     ├─► socket(AF_INET, SOCK_STREAM, 0)
  │     ├─► Set non-blocking mode: fcntl(fd, F_SETFL, O_NONBLOCK)
  │     ├─► connect() → returns EINPROGRESS (non-blocking)
  │     ├─► Register fd with the netpoller (epoll_ctl)
  │     ├─► Park the goroutine (gopark) — goroutine sleeps, OS thread is free
  │     ├─► epoll_wait reports the fd is writable → connection complete
  │     ├─► Wake the goroutine (goready)
  │     └─► Return the connected socket wrapped in net.TCPConn
  │
  └─► Return (*net.TCPConn, nil)
```

**Key insight:** Go NEVER blocks an OS thread on network I/O. Every socket is
set to non-blocking mode, and the goroutine is parked until the I/O completes.
This is why you can have 100,000 goroutines each with a network connection
without needing 100,000 OS threads.

### 4.9.2 The Netpoller — Goroutine-Per-Connection Model

The netpoller is the bridge between Go's goroutine scheduler and the OS's
I/O multiplexing (epoll on Linux):

```
┌─────────────────────────────────────────────────────────────────┐
│                     Go Runtime                                   │
│                                                                 │
│  Goroutine 1        Goroutine 2        Goroutine 3              │
│  (reading from      (writing to        (accepting on            │
│   conn A)            conn B)            listener)               │
│     │                  │                  │                     │
│     ▼                  ▼                  ▼                     │
│  ┌──────────────────────────────────────────────┐               │
│  │              NETPOLLER                        │               │
│  │                                              │               │
│  │  Parks goroutines waiting for I/O            │               │
│  │  Registers fds with epoll                    │               │
│  │  Wakes goroutines when I/O is ready          │               │
│  │                                              │               │
│  │  netpollblock(fd, 'r') → park goroutine      │               │
│  │  netpollunblock(fd, 'r') → wake goroutine    │               │
│  └──────────────────────┬───────────────────────┘               │
│                         │                                       │
└─────────────────────────┼───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Linux Kernel                                 │
│                                                                 │
│  epoll instance:                                                │
│    fd_A → EPOLLIN  (readable)                                   │
│    fd_B → EPOLLOUT (writable)                                   │
│    fd_C → EPOLLIN  (new connection)                             │
│                                                                 │
│  epoll_wait() → returns ready fds → netpoller wakes goroutines │
└─────────────────────────────────────────────────────────────────┘
```

**How the netpoller integrates with the scheduler:**

1. When a goroutine does `conn.Read()`, the runtime calls `read()` on the
   non-blocking fd.
2. If `read()` returns `EAGAIN` (no data yet), the runtime calls
   `netpollblock()` which parks the goroutine and registers the fd with epoll.
3. The goroutine is removed from the run queue — it consumes NO CPU.
4. A background thread periodically calls `epoll_wait()` (or the scheduler
   checks between scheduling decisions).
5. When data arrives, epoll reports the fd. The runtime calls `netpollunblock()`
   to wake the goroutine and put it back on the run queue.

This is why Go achieves the simplicity of blocking I/O with the performance
of non-blocking I/O. Each goroutine writes simple sequential code, but the
runtime multiplexes thousands of connections onto a few OS threads.

### 4.9.3 DNS Resolution in Go

Go has two DNS resolvers:

**Pure Go resolver** (default):
- Reads `/etc/resolv.conf` for nameserver configuration
- Sends UDP DNS queries directly
- Non-blocking, goroutine-friendly
- Used by default when possible

**cgo resolver:**
- Calls libc's `getaddrinfo()` via cgo
- Blocks an OS thread during resolution
- Required when:
  - `/etc/nsswitch.conf` has complex configuration
  - LDAP or other non-DNS name resolution is needed
  - `CGO_ENABLED=1` and the system needs cgo (macOS)

```go
// Force the pure Go resolver (even if cgo is available):
import "net"

func init() {
	// This environment variable forces the pure Go resolver.
	// Alternatively: GODEBUG=netdns=go
	net.DefaultResolver.PreferGo = true
}
```

### 4.9.4 Hands-On: Connection Pooling in Go

Connection pooling reuses TCP connections instead of creating new ones for each
request. This avoids the overhead of TCP handshake + TLS handshake for every
request.

```go
// connpool.go
//
// A production-quality TCP connection pool.
//
// Connection pooling is essential for any client that makes frequent
// connections to the same server (databases, APIs, microservices).
//
// Without pooling:
//
//	Each request: DNS lookup + TCP handshake + TLS handshake + request + close
//	Cost: ~1-50ms per connection setup
//
// With pooling:
//
//	First request: Full setup (same as above)
//	Subsequent requests: Reuse existing connection (~0ms setup)
//
// This pool implements:
//   - Configurable max connections per host
//   - Idle timeout (close connections unused for too long)
//   - Health checking (verify connections are still alive)
//   - Thread-safe (safe for concurrent goroutine use)
package main

import (
	"errors"
	"fmt"
	"log"
	"net"
	"sync"
	"time"
)

// ConnPool manages a pool of reusable TCP connections to a single host.
// It is safe for concurrent use by multiple goroutines.
type ConnPool struct {
	// addr is the target address (host:port) for all connections.
	addr string

	// maxConns is the maximum number of connections in the pool.
	maxConns int

	// idleTimeout is how long an idle connection can sit in the pool
	// before being closed. Prevents holding stale connections.
	idleTimeout time.Duration

	// mu protects the pool's internal state.
	mu sync.Mutex

	// idleConns holds connections available for reuse.
	// We use a slice as a LIFO stack — most recently used connections
	// are reused first (they're most likely to still be alive).
	idleConns []*poolConn

	// numOpen tracks the total number of open connections
	// (both idle and in-use).
	numOpen int

	// closed indicates the pool has been shut down.
	closed bool
}

// poolConn wraps a net.Conn with metadata for pool management.
type poolConn struct {
	conn      net.Conn  // The underlying TCP connection.
	idleSince time.Time // When this connection became idle.
}

// NewConnPool creates a new connection pool for the given address.
// maxConns limits the total number of connections (idle + in-use).
// idleTimeout controls how long unused connections are kept.
func NewConnPool(addr string, maxConns int, idleTimeout time.Duration) *ConnPool {
	pool := &ConnPool{
		addr:        addr,
		maxConns:    maxConns,
		idleTimeout: idleTimeout,
		idleConns:   make([]*poolConn, 0, maxConns),
	}

	// Start a background goroutine to clean up expired idle connections.
	go pool.cleanupLoop()

	return pool
}

// Get retrieves a connection from the pool, or creates a new one if
// the pool is empty. Returns an error if the pool is closed or the
// maximum connection limit is reached.
func (p *ConnPool) Get() (net.Conn, error) {
	p.mu.Lock()

	if p.closed {
		p.mu.Unlock()
		return nil, errors.New("pool is closed")
	}

	// Try to reuse an idle connection (LIFO order).
	for len(p.idleConns) > 0 {
		// Pop from the end (most recently returned = most likely alive).
		n := len(p.idleConns)
		pc := p.idleConns[n-1]
		p.idleConns = p.idleConns[:n-1]

		// Check if the connection has exceeded the idle timeout.
		if time.Since(pc.idleSince) > p.idleTimeout {
			pc.conn.Close()
			p.numOpen--
			continue // Try the next one
		}

		p.mu.Unlock()

		// Quick health check: set a short deadline and try to read.
		// If the connection is dead, read() returns an error immediately.
		pc.conn.SetReadDeadline(time.Now().Add(1 * time.Millisecond))
		one := make([]byte, 1)
		_, err := pc.conn.Read(one)
		pc.conn.SetReadDeadline(time.Time{}) // Clear deadline.

		if err != nil {
			// Connection is dead — close it and try again.
			if netErr, ok := err.(net.Error); ok && netErr.Timeout() {
				// Timeout means no data available — connection is alive!
				return pc.conn, nil
			}
			pc.conn.Close()
			p.mu.Lock()
			p.numOpen--
			p.mu.Unlock()
			return p.Get() // Recursive retry
		}

		return pc.conn, nil
	}

	// No idle connections available. Create a new one if under the limit.
	if p.numOpen >= p.maxConns {
		p.mu.Unlock()
		return nil, errors.New("connection pool exhausted")
	}

	p.numOpen++
	p.mu.Unlock()

	// Dial a new connection (outside the lock to avoid blocking).
	conn, err := net.DialTimeout("tcp", p.addr, 5*time.Second)
	if err != nil {
		p.mu.Lock()
		p.numOpen--
		p.mu.Unlock()
		return nil, fmt.Errorf("dial failed: %w", err)
	}

	return conn, nil
}

// Put returns a connection to the pool for reuse.
// If the pool is full or closed, the connection is closed instead.
func (p *ConnPool) Put(conn net.Conn) {
	p.mu.Lock()
	defer p.mu.Unlock()

	if p.closed || len(p.idleConns) >= p.maxConns {
		conn.Close()
		p.numOpen--
		return
	}

	p.idleConns = append(p.idleConns, &poolConn{
		conn:      conn,
		idleSince: time.Now(),
	})
}

// Close shuts down the pool and closes all idle connections.
func (p *ConnPool) Close() {
	p.mu.Lock()
	defer p.mu.Unlock()

	p.closed = true
	for _, pc := range p.idleConns {
		pc.conn.Close()
	}
	p.idleConns = nil
}

// cleanupLoop periodically removes expired idle connections.
// This runs in a background goroutine.
func (p *ConnPool) cleanupLoop() {
	ticker := time.NewTicker(30 * time.Second)
	defer ticker.Stop()

	for range ticker.C {
		p.mu.Lock()
		if p.closed {
			p.mu.Unlock()
			return
		}

		// Remove expired connections.
		alive := p.idleConns[:0]
		for _, pc := range p.idleConns {
			if time.Since(pc.idleSince) > p.idleTimeout {
				pc.conn.Close()
				p.numOpen--
			} else {
				alive = append(alive, pc)
			}
		}
		p.idleConns = alive
		p.mu.Unlock()
	}
}

// Stats returns current pool statistics.
func (p *ConnPool) Stats() (idle, open int) {
	p.mu.Lock()
	defer p.mu.Unlock()
	return len(p.idleConns), p.numOpen
}

func main() {
	pool := NewConnPool("example.com:80", 10, 30*time.Second)
	defer pool.Close()

	// Simulate concurrent requests.
	var wg sync.WaitGroup
	for i := 0; i < 20; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()

			conn, err := pool.Get()
			if err != nil {
				log.Printf("[%d] Get failed: %v", id, err)
				return
			}

			// Use the connection (send an HTTP request).
			fmt.Fprintf(conn, "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: keep-alive\r\n\r\n")

			buf := make([]byte, 1024)
			n, _ := conn.Read(buf)
			fmt.Printf("[%d] Received %d bytes\n", id, n)

			// Return the connection to the pool for reuse.
			pool.Put(conn)
		}(i)
	}

	wg.Wait()
	idle, open := pool.Stats()
	fmt.Printf("Pool stats: %d idle, %d open\n", idle, open)
}
```

### 4.9.5 Hands-On: High-Performance HTTP Reverse Proxy

```go
// reverse_proxy.go
//
// A high-performance HTTP reverse proxy demonstrating Go's networking
// strengths. This is a simplified version of what Caddy, Traefik,
// and nginx (with Go modules) do.
//
// Key performance features:
//   - Connection pooling to backends (via http.Transport)
//   - Streaming (doesn't buffer entire response in memory)
//   - Proper hop-by-hop header handling
//   - Graceful shutdown with connection draining
//   - Health checking of backends
//
// This shows how Go's goroutine-per-connection model naturally handles
// thousands of concurrent proxy connections without callbacks or
// explicit event loops.
//
// Usage:
//
//	go run reverse_proxy.go -listen :8080 -backends "http://localhost:3001,http://localhost:3002"
package main

import (
	"context"
	"flag"
	"fmt"
	"io"
	"log"
	"net"
	"net/http"
	"net/url"
	"os"
	"os/signal"
	"strings"
	"sync"
	"sync/atomic"
	"syscall"
	"time"
)

var (
	listenAddr  = flag.String("listen", ":8080", "Address to listen on")
	backendList = flag.String("backends", "http://localhost:3000",
		"Comma-separated list of backend URLs")
)

// Backend represents a single backend server.
// It tracks health status so we can avoid sending traffic to dead backends.
type Backend struct {
	URL   *url.URL    // The backend's base URL.
	Alive atomic.Bool // Whether the backend is currently healthy.
}

// LoadBalancer distributes requests across multiple backends.
// It uses simple round-robin selection, skipping unhealthy backends.
type LoadBalancer struct {
	backends []*Backend    // List of all backends.
	current  atomic.Uint64 // Round-robin counter.
	mu       sync.RWMutex
}

// NewLoadBalancer creates a load balancer from a comma-separated list of URLs.
func NewLoadBalancer(urls string) *LoadBalancer {
	lb := &LoadBalancer{}

	for _, rawURL := range strings.Split(urls, ",") {
		rawURL = strings.TrimSpace(rawURL)
		u, err := url.Parse(rawURL)
		if err != nil {
			log.Fatalf("Invalid backend URL %q: %v", rawURL, err)
		}
		b := &Backend{URL: u}
		b.Alive.Store(true)
		lb.backends = append(lb.backends, b)
	}

	if len(lb.backends) == 0 {
		log.Fatal("No backends configured")
	}

	return lb
}

// Next returns the next healthy backend using round-robin selection.
// If no healthy backends are available, returns nil.
func (lb *LoadBalancer) Next() *Backend {
	n := len(lb.backends)
	start := int(lb.current.Add(1) - 1)

	// Try each backend once, starting from the round-robin position.
	for i := 0; i < n; i++ {
		idx := (start + i) % n
		if lb.backends[idx].Alive.Load() {
			return lb.backends[idx]
		}
	}

	return nil // All backends are down.
}

// HealthCheck periodically checks if backends are reachable.
// This runs in a background goroutine and updates the Alive flag.
func (lb *LoadBalancer) HealthCheck(interval time.Duration) {
	client := &http.Client{Timeout: 2 * time.Second}

	for {
		for _, b := range lb.backends {
			// Try to reach the backend's root path.
			resp, err := client.Get(b.URL.String() + "/health")
			if err != nil || resp.StatusCode >= 500 {
				if b.Alive.Swap(false) {
					log.Printf("Backend %s is DOWN", b.URL)
				}
			} else {
				if !b.Alive.Swap(true) {
					log.Printf("Backend %s is UP", b.URL)
				}
			}
			if resp != nil {
				resp.Body.Close()
			}
		}
		time.Sleep(interval)
	}
}

// hopByHopHeaders are headers that apply to a single transport-level
// connection and must NOT be forwarded by proxies (RFC 2616 §13.5.1).
var hopByHopHeaders = map[string]bool{
	"Connection":          true,
	"Keep-Alive":          true,
	"Proxy-Authenticate":  true,
	"Proxy-Authorization": true,
	"Te":                  true,
	"Trailer":             true,
	"Transfer-Encoding":   true,
	"Upgrade":             true,
}

// ProxyHandler is the HTTP handler that forwards requests to backends.
type ProxyHandler struct {
	lb     *LoadBalancer
	client *http.Client
}

// ServeHTTP handles each incoming request by:
//  1. Selecting a healthy backend
//  2. Creating a new request to that backend
//  3. Forwarding the request (including body)
//  4. Streaming the response back to the client
func (ph *ProxyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// Select a backend.
	backend := ph.lb.Next()
	if backend == nil {
		http.Error(w, "Service Unavailable — all backends are down", http.StatusServiceUnavailable)
		return
	}

	// Construct the backend URL.
	targetURL := *backend.URL
	targetURL.Path = r.URL.Path
	targetURL.RawQuery = r.URL.RawQuery

	// Create the backend request.
	backendReq, err := http.NewRequestWithContext(r.Context(), r.Method, targetURL.String(), r.Body)
	if err != nil {
		http.Error(w, "Bad Gateway", http.StatusBadGateway)
		return
	}

	// Copy headers, removing hop-by-hop headers.
	for key, values := range r.Header {
		if hopByHopHeaders[key] {
			continue
		}
		for _, v := range values {
			backendReq.Header.Add(key, v)
		}
	}

	// Set X-Forwarded-For header.
	clientIP, _, _ := net.SplitHostPort(r.RemoteAddr)
	if prior := r.Header.Get("X-Forwarded-For"); prior != "" {
		clientIP = prior + ", " + clientIP
	}
	backendReq.Header.Set("X-Forwarded-For", clientIP)
	backendReq.Header.Set("X-Forwarded-Host", r.Host)
	backendReq.Header.Set("X-Forwarded-Proto", "http")

	// Send the request to the backend.
	resp, err := ph.client.Do(backendReq)
	if err != nil {
		log.Printf("Backend %s error: %v", backend.URL, err)
		backend.Alive.Store(false) // Mark backend as down.
		http.Error(w, "Bad Gateway", http.StatusBadGateway)
		return
	}
	defer resp.Body.Close()

	// Copy response headers.
	for key, values := range resp.Header {
		if hopByHopHeaders[key] {
			continue
		}
		for _, v := range values {
			w.Header().Add(key, v)
		}
	}

	// Write the status code.
	w.WriteHeader(resp.StatusCode)

	// Stream the response body to the client.
	// io.Copy uses a 32KB buffer and streams without buffering
	// the entire response in memory. This handles large responses
	// (file downloads, video streaming) efficiently.
	io.Copy(w, resp.Body)
}

func main() {
	flag.Parse()

	lb := NewLoadBalancer(*backendList)

	// Start background health checking.
	go lb.HealthCheck(10 * time.Second)

	// Create an HTTP client with connection pooling.
	// This is critical for performance — reuses TCP connections to backends.
	transport := &http.Transport{
		// MaxIdleConns controls the total number of idle connections
		// across ALL backends.
		MaxIdleConns: 100,

		// MaxIdleConnsPerHost controls idle connections per backend.
		// Set this high enough to handle your expected concurrency.
		MaxIdleConnsPerHost: 50,

		// IdleConnTimeout closes idle connections after this duration.
		// Prevents holding stale connections to dead backends.
		IdleConnTimeout: 90 * time.Second,

		// DialContext configures the TCP connection to backends.
		DialContext: (&net.Dialer{
			Timeout:   5 * time.Second,  // TCP handshake timeout
			KeepAlive: 30 * time.Second, // TCP keepalive interval
		}).DialContext,

		// ForceAttemptHTTP2 enables HTTP/2 if the backend supports it.
		// HTTP/2 multiplexes requests over a single TCP connection.
		ForceAttemptHTTP2: true,
	}

	handler := &ProxyHandler{
		lb:     lb,
		client: &http.Client{Transport: transport},
	}

	server := &http.Server{
		Addr:         *listenAddr,
		Handler:      handler,
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 30 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Graceful shutdown on SIGTERM/SIGINT.
	go func() {
		sigCh := make(chan os.Signal, 1)
		signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)
		sig := <-sigCh
		log.Printf("Received signal %v, shutting down gracefully...", sig)

		// Give active connections 30 seconds to complete.
		ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
		defer cancel()
		server.Shutdown(ctx)
	}()

	fmt.Printf("Reverse proxy listening on %s\n", *listenAddr)
	fmt.Printf("Backends: %s\n", *backendList)

	if err := server.ListenAndServe(); err != http.ErrServerClosed {
		log.Fatalf("Server error: %v", err)
	}

	log.Println("Server stopped gracefully")
}
```

---

## 4.10 Summary

This chapter took you deep into the Linux network stack — from raw syscalls to
kernel data structures to the tools that make modern infrastructure possible.

### Key Takeaways

1. **The socket API is the universal interface.** `socket()`, `bind()`, `listen()`,
   `accept()`, `connect()` — every network program in every language uses these
   syscalls. Understanding them means understanding networking.

2. **TCP is complex for good reasons.** The 11-state state machine, TIME_WAIT,
   congestion control — each exists to solve a real problem. When you tune
   `TCP_NODELAY`, `SO_REUSEPORT`, or keepalive, you're talking directly to this
   state machine.

3. **Packets have a long journey through the kernel.** NIC → interrupt → NAPI →
   sk_buff → GRO → Netfilter → protocol handler → socket buffer → application.
   Each step is a potential bottleneck and a potential observation point.

4. **Netfilter/iptables is the backbone of Linux firewalling and NAT.** Docker
   and Kubernetes depend on it heavily. Understanding chain traversal order and
   connection tracking demystifies container networking.

5. **Network namespaces + veth pairs + bridges = container networking.** This is
   exactly what Docker and Kubernetes CNI plugins do. There's no magic — just
   these three primitives combined in different ways.

6. **eBPF is the future.** XDP for ultra-fast packet processing, TC for flexible
   packet manipulation, socket-level hooks for application transparency. Cilium
   and other projects are replacing iptables with eBPF for better performance
   and richer features.

7. **Go's net package is elegant but not magic.** The netpoller translates
   blocking-style goroutine code into non-blocking epoll-based I/O. Understanding
   this helps you write better networked Go programs and debug performance issues.

### What's Next

In Chapter 5, we'll explore **Linux Storage and File Systems** — how data persists
to disk, the VFS layer, ext4 and XFS internals, the page cache, direct I/O,
and how Go interacts with the storage stack. The journey from `open()` to spinning
platters (or flash cells) is just as fascinating as the journey from `socket()`
to the wire.

---

## 4.11 Further Reading

- **"TCP/IP Illustrated, Volume 1"** by W. Richard Stevens — The definitive TCP reference
- **"Linux Kernel Networking"** by Rami Rosen — Deep dive into kernel networking code
- **"BPF Performance Tools"** by Brendan Gregg — eBPF for observability and performance
- **Linux kernel source:** `net/ipv4/tcp.c`, `net/core/sock.c`, `net/netfilter/`
- **RFC 793** — TCP specification
- **RFC 9293** — TCP specification (updated, 2022)
- **Cilium documentation:** https://docs.cilium.io — eBPF-based networking in practice
- **Linux networking stack diagram:** https://www.thomas-krenn.com/en/wiki/Linux_Networking_Stack
- **Go netpoller source:** `runtime/netpoll_epoll.go` in the Go repository
