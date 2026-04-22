# Chapter 22: CNI — Container Network Interface Deep Dive

> *"In Kubernetes, the network is the computer. And CNI is the specification that
> tells the computer how to wire itself together."*

If CSI (Chapter 21) is how Kubernetes talks to storage, then **CNI** — the Container
Network Interface — is how Kubernetes talks to the network. Every time a Pod is
created, the kubelet delegates network setup to a CNI plugin. That plugin creates
network interfaces, assigns IP addresses, sets up routes, and configures firewall
rules. When the Pod dies, the plugin tears it all down.

CNI is deceptively simple: it's a specification for executing a binary with some
environment variables and a JSON config on stdin. But the implementations built on
top of it — Calico, Cilium, Flannel, Weave — are some of the most sophisticated
networking systems in the cloud-native ecosystem.

This chapter takes you from the CNI specification through plugin implementation in
Go to a deep understanding of how the major CNI implementations work.

---

## What is CNI?

### A Specification, Not a Product

CNI is a **specification** maintained by the CNCF (Cloud Native Computing Foundation)
that defines:

1. A **configuration format** (JSON) for describing network configurations
2. A **protocol** (environment variables + stdin/stdout) for invoking plugins
3. An **execution model** (ADD, DEL, CHECK, VERSION operations)
4. A **result format** (JSON) for returning network configuration to the runtime

CNI is intentionally minimal. It doesn't care about overlay networks, BGP routing,
eBPF programs, or encryption. It defines the *interface* — how a container runtime
asks a plugin to set up networking — and leaves the *implementation* entirely to
the plugin.

### Plugin-Based Architecture

A CNI plugin is just an executable binary. The container runtime (CRI-O, containerd)
calls the binary, passes it a JSON configuration on stdin, and reads the result
from stdout. Environment variables tell the plugin what operation to perform and
which container to configure.

```text
┌──────────────────────┐          ┌───────────────────────┐
│   Container Runtime  │          │     CNI Plugin        │
│   (containerd/CRI-O) │          │   (/opt/cni/bin/...)  │
│                      │          │                       │
│  1. Set env vars:    │          │                       │
│     CNI_COMMAND=ADD  │          │                       │
│     CNI_CONTAINERID  │ exec     │                       │
│     CNI_NETNS        │────────►│  4. Read stdin (JSON) │
│     CNI_IFNAME       │          │  5. Parse config      │
│  2. Write JSON config│  stdin   │  6. Create veth pair  │
│     to stdin         │────────►│  7. Assign IP         │
│  3. Execute binary   │          │  8. Set up routes     │
│                      │  stdout  │  9. Write result JSON │
│  10. Read result     │◄────────│                       │
│      from stdout     │          │                       │
└──────────────────────┘          └───────────────────────┘
```

### Not Just Kubernetes

While Kubernetes is the most prominent user of CNI, the specification is used by:

- **Podman**: Uses CNI (or Netavark) for container networking
- **CRI-O**: The Kubernetes-native CRI implementation uses CNI directly
- **rkt** (archived): Was one of the original CNI users
- **Mesos**: Marathon used CNI for container networking
- **Cloud Foundry**: Uses CNI for application networking

Any system that manages Linux network namespaces can use CNI.

---

## The CNI Specification

### Operations

The CNI specification defines four operations, identified by the `CNI_COMMAND`
environment variable:

#### ADD — Attach Container to Network

The ADD operation creates a network interface inside the container's network
namespace and configures it (IP address, routes, DNS).

**When it's called**: When a new Pod sandbox is created (before any containers
in the Pod start).

**What the plugin must do**:
1. Create a network interface in the container's network namespace
2. Assign an IP address to the interface
3. Set up necessary routes
4. Return the configuration (IP, routes, DNS) as JSON on stdout

**Idempotency**: ADD is NOT required to be idempotent. If called twice for the same
`(CNI_CONTAINERID, CNI_IFNAME)` pair, the plugin may return an error.

#### DEL — Detach Container from Network

The DEL operation removes the network interface and cleans up any resources allocated
by ADD.

**When it's called**: When a Pod sandbox is being destroyed (after all containers
have stopped).

**What the plugin must do**:
1. Remove the network interface from the container's network namespace
2. Release the IP address back to the IPAM pool
3. Clean up any routes, firewall rules, or other state

**Idempotency**: DEL MUST be idempotent. It must succeed even if ADD was never
called, or if DEL was already called. This is because the runtime may call DEL
multiple times (e.g., if the first DEL call times out).

#### CHECK — Verify Container Networking

The CHECK operation verifies that the container's networking is still correctly
configured. It's used for health checking.

**When it's called**: Periodically by the runtime, or on demand.

**What the plugin must do**:
1. Verify the network interface exists in the container's namespace
2. Verify the IP address is correctly assigned
3. Verify routes are in place
4. Return an error if anything is wrong

#### VERSION — Report Supported Versions

Returns the CNI specification versions that this plugin supports.

### Environment Variables

The runtime passes the following environment variables to the plugin:

| Variable           | Description                                              |
|--------------------|---------------------------------------------------------|
| `CNI_COMMAND`      | Operation: `ADD`, `DEL`, `CHECK`, or `VERSION`          |
| `CNI_CONTAINERID`  | Unique ID of the container (e.g., Docker container ID)  |
| `CNI_NETNS`        | Path to the container's network namespace               |
|                    | (e.g., `/var/run/netns/cni-12345678-abcd`)              |
| `CNI_IFNAME`       | Name of the interface to create (e.g., `eth0`)          |
| `CNI_ARGS`         | Extra arguments (semicolon-separated key=value pairs)   |
| `CNI_PATH`         | List of paths to search for CNI plugin binaries         |

### Network Configuration Format

The runtime passes a JSON configuration to the plugin on stdin:

```json
{
  "cniVersion": "1.0.0",
  "name": "my-network",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.1.0/24",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  },
  "dns": {
    "nameservers": ["10.96.0.10"],
    "search": ["default.svc.cluster.local", "svc.cluster.local", "cluster.local"]
  }
}
```

**Standard fields:**
| Field        | Description                                                |
|--------------|------------------------------------------------------------|
| `cniVersion` | CNI spec version this config conforms to                   |
| `name`       | Network name — must be unique across the host              |
| `type`       | Plugin binary name (looked up in CNI_PATH)                 |
| `ipam`       | IPAM (IP Address Management) plugin configuration          |
| `dns`        | DNS configuration to return to the runtime                 |

Additional fields (like `bridge`, `isGateway`, `ipMasq`) are plugin-specific.

### Result Format

After ADD, the plugin returns a JSON result on stdout:

```json
{
  "cniVersion": "1.0.0",
  "interfaces": [
    {
      "name": "eth0",
      "mac": "0a:58:0a:f4:01:05",
      "sandbox": "/var/run/netns/cni-12345678-abcd"
    },
    {
      "name": "vethXYZ",
      "mac": "0a:58:0a:f4:01:01"
    }
  ],
  "ips": [
    {
      "address": "10.244.1.5/24",
      "gateway": "10.244.1.1",
      "interface": 0
    }
  ],
  "routes": [
    {
      "dst": "0.0.0.0/0",
      "gw": "10.244.1.1"
    }
  ],
  "dns": {
    "nameservers": ["10.96.0.10"],
    "search": ["default.svc.cluster.local"]
  }
}
```

### Plugin Chaining

CNI supports chaining multiple plugins in sequence. Each plugin in the chain
receives the result of the previous plugin as `prevResult` in the config. This
allows composing simple plugins into complex network configurations.

```json
{
  "cniVersion": "1.0.0",
  "name": "my-chain",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "ipam": {
        "type": "host-local",
        "subnet": "10.244.1.0/24"
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    },
    {
      "type": "bandwidth",
      "ingressRate": 1000000,
      "egressRate": 1000000
    },
    {
      "type": "firewall"
    }
  ]
}
```

In this chain:
1. **bridge**: Creates the bridge, veth pair, and assigns IP (via host-local IPAM)
2. **portmap**: Sets up port mappings (iptables DNAT rules) for `hostPort`
3. **bandwidth**: Applies bandwidth limits using `tc` (traffic control)
4. **firewall**: Sets up iptables rules for network isolation

Each plugin receives the `prevResult` from the previous plugin and can modify it.

---

## How Kubernetes Uses CNI

### The Pod Creation Flow

When a Pod is scheduled to a node, the following sequence occurs:

```
┌─────────┐      ┌─────────────┐      ┌──────────┐      ┌────────────┐
│ kubelet  │─────►│ CRI Runtime │─────►│ CNI      │─────►│ CNI Plugin │
│          │ CRI  │ (containerd)│ CNI  │ Library  │ exec │ (binary)   │
│          │      │             │      │          │      │            │
│ 1. Pod   │      │ 2. Create   │      │ 3. Load  │      │ 4. Create  │
│    spec  │      │    sandbox  │      │    config │      │    veth    │
│    from  │      │    (pause   │      │    from   │      │    pair    │
│    API   │      │    contain- │      │    /etc/  │      │ 5. Assign  │
│    server│      │    er)      │      │    cni/   │      │    IP      │
│          │      │             │      │    net.d/ │      │ 6. Setup   │
│          │      │             │      │          │      │    routes  │
└─────────┘      └─────────────┘      └──────────┘      └────────────┘
```

Step by step:

1. **kubelet** receives a Pod spec from the API server (via the watch loop)
2. **kubelet** calls the CRI (Container Runtime Interface) to create a Pod sandbox
3. **CRI runtime** (containerd/CRI-O) creates the sandbox container (the "pause"
   container) with a new network namespace
4. **CRI runtime** invokes the CNI library to set up networking for the sandbox
5. **CNI library** reads the configuration from `/etc/cni/net.d/` and executes
   the appropriate plugin binary from `/opt/cni/bin/`
6. **CNI plugin** creates the network interface, assigns IP, sets up routes
7. **CNI plugin** returns the result (IP, routes, DNS) to the CRI runtime
8. **CRI runtime** stores the network info and returns to kubelet
9. **kubelet** creates the application containers, which share the sandbox's
   network namespace

### Configuration Location

CNI configuration files are located in `/etc/cni/net.d/` on each node. The CRI
runtime reads the **first file** (alphabetically) in this directory.

```bash
# Typical CNI config directory
ls -la /etc/cni/net.d/
# -rw-r--r-- 1 root root  604 Jan 10 12:00 10-calico.conflist
# -rw-r--r-- 1 root root  292 Jan 10 12:00 calico-kubeconfig

# Or for Flannel:
# -rw-r--r-- 1 root root  438 Jan 10 12:00 10-flannel.conflist

# Or for Cilium:
# -rw-r--r-- 1 root root  145 Jan 10 12:00 05-cilium.conflist
```

The file uses `.conflist` extension for chained configurations and `.conf` for
single-plugin configurations.

### Plugin Binary Location

CNI plugin binaries are stored in `/opt/cni/bin/`:

```bash
# Standard CNI plugin binaries
ls /opt/cni/bin/
# bandwidth    calico       dhcp         flannel      host-device
# bridge       calico-ipam  firewall     host-local   install
# ipvlan       loopback     macvlan      portmap      ptp
# sbr          static       tuning       vlan         vrf
```

### Hands-On: Examining CNI Configuration on a Node

```bash
# SSH into a Kubernetes node (or use kubectl debug)
kubectl debug node/worker-1 -it --image=busybox:1.36

# Inside the debug container, the host filesystem is at /host
cat /host/etc/cni/net.d/10-calico.conflist
```

Example Calico conflist:

```json
{
  "name": "k8s-pod-network",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "datastore_type": "kubernetes",
      "mtu": 0,
      "nodename_file_optional": false,
      "log_level": "Info",
      "log_file_path": "/var/log/calico/cni/cni.log",
      "ipam": {
        "type": "calico-ipam",
        "assign_ipv4": "true",
        "assign_ipv6": "false"
      },
      "container_settings": {
        "allow_ip_forwarding": false
      },
      "policy": {
        "type": "k8s"
      },
      "kubernetes": {
        "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
      }
    },
    {
      "type": "portmap",
      "snat": true,
      "capabilities": {
        "portMappings": true
      }
    },
    {
      "type": "bandwidth",
      "capabilities": {
        "bandwidth": true
      }
    }
  ]
}
```

---

## Built-in CNI Plugins

The CNI project maintains a set of reference plugins. These are the building blocks
that more complex CNI implementations use.

### Main Plugins

These plugins create network interfaces:

| Plugin       | Description                                                |
|--------------|------------------------------------------------------------|
| `bridge`     | Creates a Linux bridge and adds the container to it        |
| `ptp`        | Creates a point-to-point link (veth pair, no bridge)       |
| `host-local` | (Actually IPAM, not main) Allocates IPs from local ranges  |
| `macvlan`    | Creates a new MAC address on a host interface              |
| `ipvlan`     | Like macvlan, but shares the host MAC address              |
| `vlan`       | Creates a VLAN-tagged sub-interface                        |
| `host-device`| Moves an existing host device into the container namespace |

### IPAM Plugins

IPAM (IP Address Management) plugins handle IP address allocation:

| Plugin       | Description                                                |
|--------------|------------------------------------------------------------|
| `host-local` | Allocates IPs from configured ranges, stores in local files|
| `dhcp`       | Requests an IP from an external DHCP server                |
| `static`     | Assigns a fixed IP address (useful for testing)            |

### Meta Plugins

Meta plugins modify the behavior of other plugins in a chain:

| Plugin       | Description                                                |
|--------------|------------------------------------------------------------|
| `flannel`    | Reads Flannel subnet config, delegates to bridge           |
| `portmap`    | Creates port mappings (iptables DNAT) for hostPort         |
| `bandwidth`  | Rate-limits traffic using Linux tc (traffic control)       |
| `tuning`     | Sets sysctl parameters and interface attributes            |
| `firewall`   | Adds iptables rules to allow/deny traffic                  |
| `sbr`        | Sets up source-based routing for multi-interface Pods      |

### Hands-On: Manual CNI Plugin Invocation

You can invoke CNI plugins manually to understand how they work. This is invaluable
for debugging:

```bash
# Set up environment variables that the CNI plugin expects
export CNI_COMMAND=ADD
export CNI_CONTAINERID=test-container-123
export CNI_NETNS=/var/run/netns/test-ns
export CNI_IFNAME=eth0
export CNI_PATH=/opt/cni/bin

# Create a test network namespace
ip netns add test-ns

# Invoke the bridge plugin with a config on stdin
echo '{
  "cniVersion": "1.0.0",
  "name": "test-net",
  "type": "bridge",
  "bridge": "cni-test0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.99.0.0/24",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  }
}' | /opt/cni/bin/bridge

# Expected output (JSON result):
# {
#   "cniVersion": "1.0.0",
#   "interfaces": [
#     { "name": "cni-test0", "mac": "3e:4b:5c:6d:7e:8f" },
#     { "name": "veth12345678", "mac": "0a:58:0a:63:00:01" },
#     { "name": "eth0", "mac": "0a:58:0a:63:00:02", "sandbox": "/var/run/netns/test-ns" }
#   ],
#   "ips": [
#     { "address": "10.99.0.2/24", "gateway": "10.99.0.1", "interface": 2 }
#   ],
#   "routes": [
#     { "dst": "0.0.0.0/0", "gw": "10.99.0.1" }
#   ]
# }

# Verify the setup
ip netns exec test-ns ip addr show eth0
# eth0: <BROADCAST,MULTICAST,UP,LOWER_UP>
#     inet 10.99.0.2/24 brd 10.99.0.255 scope global eth0

ip netns exec test-ns ip route
# default via 10.99.0.1 dev eth0
# 10.99.0.0/24 dev eth0 proto kernel scope link src 10.99.0.2

# Verify the bridge on the host
ip link show cni-test0
# cni-test0: <BROADCAST,MULTICAST,UP,LOWER_UP> state UP

# Clean up — invoke DEL
export CNI_COMMAND=DEL
echo '{
  "cniVersion": "1.0.0",
  "name": "test-net",
  "type": "bridge",
  "bridge": "cni-test0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "10.99.0.0/24"
  }
}' | /opt/cni/bin/bridge

# Delete the test namespace
ip netns del test-ns
```

---

## Bridge Plugin Deep Dive

The bridge plugin is the most commonly used CNI main plugin. It's the foundation
for Flannel and many simple CNI setups. Understanding it deeply means understanding
how most Kubernetes Pod networking works.

### How the Bridge Plugin Works

When the bridge plugin's ADD is called:

1. **Create the bridge** (if it doesn't exist): A Linux bridge device (`cni0` by
   default) is created on the host.

2. **Create a veth pair**: A virtual ethernet pair is created. Think of it as a
   virtual cable — a packet sent in one end comes out the other.

3. **Move one end into the container**: The container-side veth is moved into the
   container's network namespace and renamed to the requested interface name
   (usually `eth0`).

4. **Attach the other end to the bridge**: The host-side veth is attached to the
   bridge.

5. **Call IPAM**: The IPAM plugin assigns an IP address to the container-side
   interface.

6. **Set up the gateway**: If `isGateway` is true, the bridge itself gets an IP
   address (the gateway address for the subnet).

7. **Set up routes**: A default route via the gateway is added inside the container.

8. **IP masquerade**: If `ipMasq` is true, iptables MASQUERADE rules are added so
   the container can reach external networks using the host's IP.

### ASCII Diagram: Bridge Topology

```
┌──────────────────────────────────────────────────────────────────────┐
│                            Host (Node)                                │
│                                                                      │
│                        ┌──────────────┐                              │
│                        │  Linux Bridge │                              │
│                        │   cni0       │                              │
│                        │  10.244.1.1  │ ← Gateway for subnet         │
│                        └──┬───┬───┬──┘                              │
│                           │   │   │                                  │
│              ┌────────────┘   │   └────────────┐                    │
│              │                │                │                    │
│         ┌────┴────┐     ┌────┴────┐     ┌────┴────┐               │
│         │vethAAAA │     │vethBBBB │     │vethCCCC │  Host-side     │
│         └────┬────┘     └────┬────┘     └────┬────┘  veth ends     │
│ ─ ─ ─ ─ ─ ─ │ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ │ ─ ─ Namespace ─ ─ │
│         ┌────┴────┐     ┌────┴────┐     ┌────┴────┐  boundary     │
│         │  eth0   │     │  eth0   │     │  eth0   │               │
│         │10.244.  │     │10.244.  │     │10.244.  │  Container-   │
│         │  1.2    │     │  1.3    │     │  1.4    │  side veth    │
│         └─────────┘     └─────────┘     └─────────┘  ends          │
│                                                                      │
│         ┌─────────┐     ┌─────────┐     ┌─────────┐               │
│         │  Pod A  │     │  Pod B  │     │  Pod C  │               │
│         │ (nginx) │     │ (redis) │     │ (app)   │               │
│         └─────────┘     └─────────┘     └─────────┘               │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │  Host Network Stack                                       │       │
│  │  eth0: 192.168.1.10  ← Host's real network interface     │       │
│  │  iptables: MASQUERADE for 10.244.1.0/24                   │       │
│  └──────────────────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────────┘
```

### Packet Flow: Pod A → Pod B (Same Node)

When Pod A (10.244.1.2) sends a packet to Pod B (10.244.1.3):

```
1. Pod A's eth0 → vethAAAA (through veth pair)
2. vethAAAA → cni0 bridge (frame enters bridge)
3. cni0 bridge → vethBBBB (bridge forwards based on MAC learning)
4. vethBBBB → Pod B's eth0 (through veth pair)
```

This is pure Layer 2 switching — the Linux bridge acts as a virtual switch.

### Packet Flow: Pod A → External (Different Node)

When Pod A (10.244.1.2) sends a packet to 8.8.8.8:

```
1. Pod A's eth0 → vethAAAA (through veth pair)
2. vethAAAA → cni0 bridge
3. Bridge → host network stack (packet is for non-local destination)
4. Host routing table → eth0 (host's physical interface)
5. iptables MASQUERADE → source IP changed from 10.244.1.2 to 192.168.1.10
6. eth0 → physical network → internet
```

### Packet Flow: Pod A (Node 1) → Pod D (Node 2)

This is where things get interesting. The bridge plugin alone can't route between
nodes — it only handles local Pod-to-Pod traffic. For cross-node traffic, you need
an **overlay network** (VXLAN, IPIP) or **BGP routing**, which is what Flannel,
Calico, and Cilium provide.

---

## Writing a CNI Plugin in Go — From Scratch

Now let's write a complete, working CNI plugin in Go. This plugin creates a bridge,
assigns IPs using a built-in simple allocator, and connects containers to the bridge.

### Project Structure

```
my-cni-plugin/
├── go.mod
├── go.sum
├── main.go          # Entry point — dispatches to cmdAdd, cmdDel, cmdCheck
├── bridge.go        # Bridge creation and veth pair management
├── ipam.go          # Simple IP address allocation
└── config.go        # Configuration parsing
```

### config.go — Configuration Parsing

```go
// config.go
//
// Package main provides a minimal CNI plugin implementation.
//
// This file handles parsing the CNI network configuration that the container
// runtime passes to the plugin on stdin. The configuration is a JSON document
// that conforms to the CNI specification, with additional plugin-specific
// fields for our bridge-based networking.
//
// Example configuration:
//
//	{
	//	    "cniVersion": "1.0.0",
	//	    "name": "my-network",
	//	    "type": "my-cni-plugin",
	//	    "bridge": "cni0",
	//	    "subnet": "10.244.0.0/24",
	//	    "gateway": "10.244.0.1",
	//	    "mtu": 1500
	//	}
package main

import (
	"encoding/json"
	"fmt"
	"net"
)

// PluginConf represents the CNI network configuration for our plugin.
//
// It embeds the standard CNI configuration fields (cniVersion, name, type)
// and adds our plugin-specific fields for bridge configuration and IP
// address management.
//
// The CNI runtime deserializes this from the JSON passed on stdin.
// Plugin-specific fields that are unknown to the CNI libraries are
// preserved in the raw JSON and parsed here.
type PluginConf struct {
	// CNIVersion is the version of the CNI specification that this
	// configuration conforms to. Must be "1.0.0" or compatible.
	CNIVersion string `json:"cniVersion"`

	// Name is the network name. Must be unique across all CNI networks
	// on a single host. Used as a key for IPAM state.
	Name string `json:"name"`

	// Type is the name of the CNI plugin binary to execute.
	// The runtime looks for this binary in the directories listed
	// in the CNI_PATH environment variable.
	Type string `json:"type"`

	// BridgeName is the name of the Linux bridge device to create
	// or use. Defaults to "cni0" if not specified.
	// All containers on the same bridge share a Layer 2 domain.
	BridgeName string `json:"bridge"`

	// Subnet is the CIDR notation subnet from which to allocate
	// container IP addresses. For example, "10.244.1.0/24" provides
	// 254 usable addresses (10.244.1.1 through 10.244.1.254).
	Subnet string `json:"subnet"`

	// Gateway is the IP address to assign to the bridge interface.
	// This becomes the default gateway for all containers on this bridge.
	// If empty, defaults to the first usable IP in the subnet.
	Gateway string `json:"gateway"`

	// MTU (Maximum Transmission Unit) is the maximum packet size for
	// the container's network interface. Defaults to 1500 if not set.
	// For overlay networks (VXLAN), this should be reduced by the
	// encapsulation overhead (typically 1450 for VXLAN).
	MTU int `json:"mtu"`
}

// parseConfig deserializes the JSON configuration from stdin into a
// PluginConf struct.
//
// It performs validation to ensure all required fields are present
// and that the subnet is a valid CIDR notation. This function is
// called at the beginning of every CNI operation (ADD, DEL, CHECK).
//
// Parameters:
//   - data: raw JSON bytes read from stdin
//
// Returns:
//   - *PluginConf: the parsed and validated configuration
//   - error: if the JSON is malformed or required fields are missing
func parseConfig(data []byte) (*PluginConf, error) {
	conf := &PluginConf{}

	// Unmarshal the JSON configuration.
	// encoding/json will populate matching struct fields and ignore
	// unknown fields (which may be present for plugin chaining).
	if err := json.Unmarshal(data, conf); err != nil {
		return nil, fmt.Errorf("failed to parse CNI config: %w", err)
	}

	// Validate required fields
	if conf.Name == "" {
		return nil, fmt.Errorf("missing required field: name")
	}
	if conf.Subnet == "" {
		return nil, fmt.Errorf("missing required field: subnet")
	}

	// Validate the subnet is valid CIDR notation
	_, ipNet, err := net.ParseCIDR(conf.Subnet)
	if err != nil {
		return nil, fmt.Errorf("invalid subnet %q: %w", conf.Subnet, err)
	}

	// Apply defaults
	if conf.BridgeName == "" {
		conf.BridgeName = "cni0"
	}
	if conf.MTU == 0 {
		conf.MTU = 1500
	}
	if conf.Gateway == "" {
		// Default gateway: first usable IP in the subnet
		// For 10.244.1.0/24, the gateway would be 10.244.1.1
		gw := nextIP(ipNet.IP)
		conf.Gateway = gw.String()
	}

	return conf, nil
}

// nextIP returns the next IP address after the given one.
//
// For example, nextIP(10.244.1.0) returns 10.244.1.1.
// This is used to compute the default gateway address (first usable
	// IP in the subnet).
//
// Note: This function does not handle overflow — it assumes the input
// is not the broadcast address of a subnet.
func nextIP(ip net.IP) net.IP {
	// Work with the 4-byte representation for IPv4
	ip = ip.To4()
	if ip == nil {
		return nil
	}

	// Copy to avoid mutating the input
	next := make(net.IP, len(ip))
	copy(next, ip)

	// Increment the last octet, carrying over as needed
	for i := len(next) - 1; i >= 0; i-- {
		next[i]++
		if next[i] != 0 {
			break // No carry needed
		}
	}
	return next
}
```

### ipam.go — Simple IP Address Allocation

```go
// ipam.go
//
// This file implements a minimal IP Address Management (IPAM) system
// for our CNI plugin. In production, you would use a proper IPAM plugin
// like host-local, DHCP, or Whereabouts. This implementation stores
// allocated IPs in a JSON file on disk, using file locking to handle
// concurrent allocations from multiple CNI invocations.
//
// The allocation strategy is simple: iterate through all IPs in the
// subnet, skip the network address, gateway, and broadcast, and return
// the first unallocated IP.
//
// State file location: /var/lib/cni/my-plugin/<network-name>.json
//
// This IPAM implementation is intentionally simplistic for educational
// purposes. Production IPAM must handle:
//   - Cluster-wide coordination (not just per-node)
//   - IPv6 address allocation
//   - Dual-stack (IPv4 + IPv6 simultaneously)
//   - Subnet exhaustion and rebalancing
//   - Lease expiry and garbage collection
package main

import (
	"encoding/json"
	"fmt"
	"net"
	"os"
	"path/filepath"
	"syscall"
)

const (
	// ipamStateDir is the directory where we store IPAM allocation state.
	// Each network gets its own JSON file in this directory.
	ipamStateDir = "/var/lib/cni/my-plugin"
)

// IPAMState represents the persisted state of IP address allocations
// for a single network. It is serialized to/from a JSON file.
type IPAMState struct {
	// Allocations maps container IDs to their allocated IP addresses.
	// Using container ID as the key ensures we can release the correct
	// IP when DEL is called, even if multiple containers share a subnet.
	Allocations map[string]string `json:"allocations"`
}

// allocateIP assigns an unused IP address from the configured subnet
// to the specified container.
//
// The allocation algorithm:
//  1. Load existing allocations from the state file
//  2. Iterate through all IPs in the subnet
//  3. Skip the network address, gateway, broadcast, and already-allocated IPs
//  4. Assign the first available IP
//  5. Save the updated state file
//
// File locking (flock) is used to prevent race conditions when multiple
// CNI ADD operations run concurrently (which happens during Pod startup
	// storms).
//
// Parameters:
//   - conf: the parsed plugin configuration (contains subnet and gateway)
//   - containerID: unique identifier for the container requesting an IP
//
// Returns:
//   - net.IP: the allocated IP address
//   - *net.IPNet: the subnet (with the allocated IP set)
//   - error: if the subnet is exhausted or state file operations fail
func allocateIP(conf *PluginConf, containerID string) (net.IP, *net.IPNet, error) {
	// Parse the subnet from the configuration
	_, ipNet, err := net.ParseCIDR(conf.Subnet)
	if err != nil {
		return nil, nil, fmt.Errorf("invalid subnet: %w", err)
	}

	// Ensure the state directory exists
	if err := os.MkdirAll(ipamStateDir, 0o755); err != nil {
		return nil, nil, fmt.Errorf("failed to create IPAM state dir: %w", err)
	}

	// Open (or create) the state file with exclusive locking
	statePath := filepath.Join(ipamStateDir, conf.Name+".json")
	f, err := os.OpenFile(statePath, os.O_RDWR|os.O_CREATE, 0o644)
	if err != nil {
		return nil, nil, fmt.Errorf("failed to open IPAM state: %w", err)
	}
	defer f.Close()

	// Acquire an exclusive file lock to prevent concurrent allocations
	// from corrupting the state file. This is a blocking lock — if another
	// CNI invocation holds the lock, we wait until it's released.
	if err := syscall.Flock(int(f.Fd()), syscall.LOCK_EX); err != nil {
		return nil, nil, fmt.Errorf("failed to lock IPAM state: %w", err)
	}
	defer syscall.Flock(int(f.Fd()), syscall.LOCK_UN)

	// Load existing state (or initialize empty state)
	state := &IPAMState{Allocations: make(map[string]string)}
	decoder := json.NewDecoder(f)
	if err := decoder.Decode(state); err != nil {
		// If the file is empty or corrupt, start fresh
		state = &IPAMState{Allocations: make(map[string]string)}
	}

	// Check if this container already has an allocation (idempotency)
	if existingIP, ok := state.Allocations[containerID]; ok {
		ip := net.ParseIP(existingIP)
		allocatedNet := &net.IPNet{
			IP:   ip,
			Mask: ipNet.Mask,
		}
		return ip, allocatedNet, nil
	}

	// Build a set of already-allocated IPs for O(1) lookup
	allocated := make(map[string]bool)
	for _, ip := range state.Allocations {
		allocated[ip] = true
	}

	// Parse the gateway IP so we can skip it during allocation
	gatewayIP := net.ParseIP(conf.Gateway)

	// Iterate through all IPs in the subnet and find the first available one.
	//
	// We start from the network address and increment, skipping:
	//   - The network address itself (e.g., 10.244.1.0)
	//   - The gateway address (e.g., 10.244.1.1)
	//   - Any already-allocated addresses
	//   - The broadcast address (e.g., 10.244.1.255)
	var selectedIP net.IP
	for ip := nextIP(ipNet.IP); ipNet.Contains(ip); ip = nextIP(ip) {
		// Skip the gateway
		if ip.Equal(gatewayIP) {
			continue
		}

		// Skip the broadcast address (all host bits set to 1)
		if isBroadcast(ip, ipNet) {
			continue
		}

		// Skip already-allocated IPs
		if allocated[ip.String()] {
			continue
		}

		// Found an available IP
		selectedIP = make(net.IP, len(ip))
		copy(selectedIP, ip)
		break
	}

	if selectedIP == nil {
		return nil, nil, fmt.Errorf("no available IPs in subnet %s", conf.Subnet)
	}

	// Record the allocation
	state.Allocations[containerID] = selectedIP.String()

	// Write the updated state back to the file.
	// Truncate first to handle the case where the new state is shorter
	// than the old state.
	if err := f.Truncate(0); err != nil {
		return nil, nil, fmt.Errorf("failed to truncate state file: %w", err)
	}
	if _, err := f.Seek(0, 0); err != nil {
		return nil, nil, fmt.Errorf("failed to seek state file: %w", err)
	}
	encoder := json.NewEncoder(f)
	encoder.SetIndent("", "  ")
	if err := encoder.Encode(state); err != nil {
		return nil, nil, fmt.Errorf("failed to write IPAM state: %w", err)
	}

	allocatedNet := &net.IPNet{
		IP:   selectedIP,
		Mask: ipNet.Mask,
	}
	return selectedIP, allocatedNet, nil
}

// releaseIP deallocates an IP address previously assigned to a container.
//
// This is called during the DEL operation. It removes the container's
// allocation from the state file, making the IP available for future
// allocations.
//
// Idempotency: If the container has no allocation (because DEL was
	// already called, or ADD was never called), this function succeeds
// silently. This satisfies the CNI spec requirement that DEL must be
// idempotent.
//
// Parameters:
//   - conf: the parsed plugin configuration
//   - containerID: the container whose IP should be released
//
// Returns:
//   - error: if state file operations fail
func releaseIP(conf *PluginConf, containerID string) error {
	statePath := filepath.Join(ipamStateDir, conf.Name+".json")

	f, err := os.OpenFile(statePath, os.O_RDWR, 0o644)
	if err != nil {
		if os.IsNotExist(err) {
			// No state file means no allocations — nothing to release.
			// This is not an error (idempotency).
			return nil
		}
		return fmt.Errorf("failed to open IPAM state: %w", err)
	}
	defer f.Close()

	// Acquire exclusive lock
	if err := syscall.Flock(int(f.Fd()), syscall.LOCK_EX); err != nil {
		return fmt.Errorf("failed to lock IPAM state: %w", err)
	}
	defer syscall.Flock(int(f.Fd()), syscall.LOCK_UN)

	// Load state
	state := &IPAMState{Allocations: make(map[string]string)}
	decoder := json.NewDecoder(f)
	if err := decoder.Decode(state); err != nil {
		return nil // Corrupt or empty state — nothing to release
	}

	// Remove the allocation for this container
	delete(state.Allocations, containerID)

	// Write updated state
	if err := f.Truncate(0); err != nil {
		return fmt.Errorf("failed to truncate state file: %w", err)
	}
	if _, err := f.Seek(0, 0); err != nil {
		return fmt.Errorf("failed to seek state file: %w", err)
	}
	encoder := json.NewEncoder(f)
	encoder.SetIndent("", "  ")
	return encoder.Encode(state)
}

// isBroadcast returns true if the given IP is the broadcast address
// for the specified subnet.
//
// The broadcast address has all host bits set to 1. For example,
// in the 10.244.1.0/24 subnet, the broadcast is 10.244.1.255.
//
// Parameters:
//   - ip: the IP address to check
//   - network: the subnet
//
// Returns:
//   - true if ip is the broadcast address of network
func isBroadcast(ip net.IP, network *net.IPNet) bool {
	ip = ip.To4()
	if ip == nil {
		return false
	}
	// Broadcast address = network address OR (NOT subnet mask)
	// For each byte, check if the host bits are all 1s.
	for i := range ip {
		// ~mask[i] gives the host bits. If ip[i] has all those bits set,
		// and matches the network address in the network bits, it's broadcast.
		if ip[i] != (network.IP[i] | ^network.Mask[i]) {
			return false
		}
	}
	return true
}
```

### bridge.go — Bridge and Veth Pair Management

```go
// bridge.go
//
// This file handles the Linux networking operations for our CNI plugin:
//
//   - Creating and configuring a Linux bridge device
//   - Creating veth (virtual ethernet) pairs
//   - Moving network interfaces between network namespaces
//   - Configuring IP addresses and routes inside containers
//
// It uses the "netlink" library (vishvananda/netlink) for programmatic
// access to the Linux networking stack. Under the hood, netlink is the
// same interface that the `ip` command uses — it communicates with the
// kernel via Netlink sockets (AF_NETLINK, NETLINK_ROUTE).
//
// Key networking concepts:
//
//   - Bridge: A Layer 2 virtual switch. Frames entering one port are
//     forwarded to the appropriate port based on MAC address learning.
//
//   - Veth pair: A pair of virtual network interfaces connected like a
//     pipe. Whatever goes in one end comes out the other. One end is
//     placed in the container's network namespace, the other stays on
//     the host and is attached to the bridge.
//
//   - Network namespace: A Linux kernel feature that gives a process its
//     own view of the network stack (interfaces, routes, iptables, etc.).
//     Containers use network namespaces for isolation.
package main

import (
	"crypto/rand"
	"fmt"
	"net"
	"os"
	"runtime"

	"github.com/containernetworking/plugins/pkg/ns"
	"github.com/vishvananda/netlink"
)

// ensureBridge creates a Linux bridge device if it doesn't already exist,
// and configures it with the specified gateway IP address and MTU.
//
// If the bridge already exists (e.g., from a previous CNI ADD on the
	// same node), this function verifies it's up and returns it.
//
// The bridge serves as a virtual switch connecting all containers on
// this network. It gets the gateway IP address, making it the default
// gateway for all containers.
//
// Parameters:
//   - name: the bridge device name (e.g., "cni0")
//   - gatewayIP: the IP address to assign to the bridge (e.g., "10.244.1.1/24")
//   - mtu: maximum transmission unit for the bridge interface
//
// Returns:
//   - *netlink.Bridge: the bridge device (for attaching veth pairs)
//   - error: if bridge creation or configuration fails
func ensureBridge(name string, gatewayIP *net.IPNet, mtu int) (*netlink.Bridge, error) {
	// Check if the bridge already exists
	link, err := netlink.LinkByName(name)
	if err == nil {
		// Bridge exists — verify it's actually a bridge type
		br, ok := link.(*netlink.Bridge)
		if !ok {
			return nil, fmt.Errorf("device %s exists but is not a bridge", name)
		}

		// Ensure the bridge is up
		if err := netlink.LinkSetUp(br); err != nil {
			return nil, fmt.Errorf("failed to set bridge %s up: %w", name, err)
		}
		return br, nil
	}

	// Bridge doesn't exist — create it.
	// netlink.Bridge embeds netlink.LinkAttrs which holds common
	// link attributes like name, MTU, and hardware address.
	br := &netlink.Bridge{
		LinkAttrs: netlink.LinkAttrs{
			Name: name,
			MTU:  mtu,
			// TxQLen sets the transmit queue length. A value of -1
			// means "use the kernel default" (usually 1000).
			TxQLen: -1,
		},
	}

	// Create the bridge device in the kernel
	if err := netlink.LinkAdd(br); err != nil {
		return nil, fmt.Errorf("failed to create bridge %s: %w", name, err)
	}

	// Assign the gateway IP to the bridge.
	// This makes the bridge reachable as a gateway from containers.
	addr := &netlink.Addr{IPNet: gatewayIP}
	if err := netlink.AddrAdd(br, addr); err != nil {
		return nil, fmt.Errorf("failed to assign IP %s to bridge %s: %w",
			gatewayIP.String(), name, err)
	}

	// Bring the bridge up (equivalent to `ip link set cni0 up`)
	if err := netlink.LinkSetUp(br); err != nil {
		return nil, fmt.Errorf("failed to set bridge %s up: %w", name, err)
	}

	return br, nil
}

// createVethPair creates a veth pair, attaches the host-side to the bridge,
// and moves the container-side into the specified network namespace.
//
// A veth pair is like a virtual ethernet cable: packets sent into one end
// come out the other. We place one end in the container (becomes "eth0")
// and attach the other end to the bridge (becomes "vethXXXX" on the host).
//
// The function performs these steps:
//  1. Generate a random name for the host-side veth
//  2. Create the veth pair in the host namespace
//  3. Attach the host-side veth to the bridge
//  4. Move the container-side veth into the container's network namespace
//  5. Rename it to the requested interface name (usually "eth0")
//
// Parameters:
//   - bridge: the bridge to attach the host-side veth to
//   - containerNS: path to the container's network namespace
//   - ifName: the interface name inside the container (usually "eth0")
//   - mtu: maximum transmission unit for the veth interfaces
//
// Returns:
//   - string: the name of the host-side veth interface
//   - error: if any networking operation fails
func createVethPair(bridge *netlink.Bridge, containerNS string, ifName string, mtu int) (string, error) {
	// Generate a random name for the host-side veth.
	// We use random bytes to avoid collisions when multiple containers
	// are being created simultaneously.
	hostVethName, err := randomVethName()
	if err != nil {
		return "", fmt.Errorf("failed to generate veth name: %w", err)
	}

	// Create the veth pair.
	// The "veth" type automatically creates two linked interfaces.
	// LinkAttrs configures the host-side ("parent"), and PeerName
	// is the container-side interface.
	veth := &netlink.Veth{
		LinkAttrs: netlink.LinkAttrs{
			Name: hostVethName,
			MTU:  mtu,
		},
		PeerName: "cni-tmp-" + hostVethName, // Temporary name; renamed later
	}

	if err := netlink.LinkAdd(veth); err != nil {
		return "", fmt.Errorf("failed to create veth pair: %w", err)
	}

	// Get the host-side veth link object (we need it to attach to bridge)
	hostVeth, err := netlink.LinkByName(hostVethName)
	if err != nil {
		return "", fmt.Errorf("failed to find host veth %s: %w", hostVethName, err)
	}

	// Attach the host-side veth to the bridge.
	// This is equivalent to `ip link set vethXXXX master cni0`.
	if err := netlink.LinkSetMaster(hostVeth, bridge); err != nil {
		return "", fmt.Errorf("failed to attach %s to bridge: %w",
			hostVethName, err)
	}

	// Bring the host-side veth up
	if err := netlink.LinkSetUp(hostVeth); err != nil {
		return "", fmt.Errorf("failed to set %s up: %w", hostVethName, err)
	}

	// Get the container-side veth
	containerVeth, err := netlink.LinkByName(veth.PeerName)
	if err != nil {
		return "", fmt.Errorf("failed to find container veth: %w", err)
	}

	// Open the container's network namespace.
	// Network namespaces are represented as file descriptors to entries
	// in /var/run/netns/ or /proc/<pid>/ns/net.
	netNS, err := ns.GetNS(containerNS)
	if err != nil {
		return "", fmt.Errorf("failed to open netns %s: %w", containerNS, err)
	}
	defer netNS.Close()

	// Move the container-side veth into the container's network namespace.
	// After this, the interface is only visible inside the container.
	if err := netlink.LinkSetNsFd(containerVeth, int(netNS.Fd())); err != nil {
		return "", fmt.Errorf("failed to move veth to container netns: %w", err)
	}

	// Enter the container's network namespace to rename and configure
	// the interface.
	//
	// ns.Do() locks the current goroutine to its OS thread (required
		// because network namespaces are per-thread in Linux), switches to
	// the target namespace, executes the function, and switches back.
	err = netNS.Do(func(_ ns.NetNS) error {
			// Rename the temporary interface to the requested name (usually "eth0")
			containerLink, err := netlink.LinkByName(veth.PeerName)
			if err != nil {
				return fmt.Errorf("failed to find veth in container: %w", err)
			}

			if err := netlink.LinkSetName(containerLink, ifName); err != nil {
				return fmt.Errorf("failed to rename interface to %s: %w", ifName, err)
			}

			return nil
		})
	if err != nil {
		return "", err
	}

	return hostVethName, nil
}

// configureContainerNetworking enters the container's network namespace
// and configures the interface with an IP address, brings it up, and
// adds a default route via the gateway.
//
// This function is called after the veth pair is created and the
// container-side interface has been moved into the container's namespace.
//
// Parameters:
//   - containerNS: path to the container's network namespace
//   - ifName: the interface name inside the container (e.g., "eth0")
//   - ipAddr: the IP address with subnet mask to assign (e.g., "10.244.1.5/24")
//   - gateway: the default gateway IP address (e.g., 10.244.1.1)
//
// Returns:
//   - string: the MAC address of the configured interface
//   - error: if any configuration step fails
func configureContainerNetworking(containerNS string, ifName string, ipAddr *net.IPNet, gateway net.IP) (string, error) {
	var macAddr string

	netNS, err := ns.GetNS(containerNS)
	if err != nil {
		return "", fmt.Errorf("failed to open netns: %w", err)
	}
	defer netNS.Close()

	// Enter the container's network namespace to configure networking.
	//
	// We must do all interface configuration (IP assignment, routes)
	// from within the namespace. If we did it from the host namespace,
	// the operations would apply to the host's network stack instead.
	err = netNS.Do(func(_ ns.NetNS) error {
			// Lock the goroutine to its OS thread. This is critical because
			// network namespace switches are per-thread, and Go's goroutine
			// scheduler can migrate goroutines between OS threads.
			runtime.LockOSThread()

			// Get the interface by name
			link, err := netlink.LinkByName(ifName)
			if err != nil {
				return fmt.Errorf("failed to find %s in container: %w", ifName, err)
			}

			// Assign the IP address to the interface.
			// This is equivalent to: ip addr add 10.244.1.5/24 dev eth0
			addr := &netlink.Addr{IPNet: ipAddr}
			if err := netlink.AddrAdd(link, addr); err != nil {
				return fmt.Errorf("failed to assign IP %s to %s: %w",
					ipAddr.String(), ifName, err)
			}

			// Bring the interface up.
			// This is equivalent to: ip link set eth0 up
			if err := netlink.LinkSetUp(link); err != nil {
				return fmt.Errorf("failed to bring %s up: %w", ifName, err)
			}

			// Also bring up the loopback interface.
			// Without this, localhost (127.0.0.1) won't work in the container.
			lo, err := netlink.LinkByName("lo")
			if err == nil {
				netlink.LinkSetUp(lo)
			}

			// Add a default route via the gateway.
			// This is equivalent to: ip route add default via 10.244.1.1
			defaultRoute := &netlink.Route{
				Dst: &net.IPNet{
					IP:   net.IPv4zero,           // 0.0.0.0
					Mask: net.CIDRMask(0, 32),    // /0 — matches everything
				},
				Gw:        gateway,               // Next hop: the bridge IP
				LinkIndex: link.Attrs().Index,
			}
			if err := netlink.RouteAdd(defaultRoute); err != nil {
				return fmt.Errorf("failed to add default route: %w", err)
			}

			// Capture the MAC address for the result
			macAddr = link.Attrs().HardwareAddr.String()

			return nil
		})

	return macAddr, err
}

// deleteContainerVeth enters the container's network namespace and deletes
// the specified interface. Deleting one end of a veth pair automatically
// deletes the other end (the host-side veth), which also detaches it
// from the bridge.
//
// This function is called during the DEL operation.
//
// Parameters:
//   - containerNS: path to the container's network namespace
//   - ifName: the interface name to delete (e.g., "eth0")
//
// Returns:
//   - error: if the namespace cannot be entered or the interface deletion fails
func deleteContainerVeth(containerNS string, ifName string) error {
	netNS, err := ns.GetNS(containerNS)
	if err != nil {
		// If the namespace doesn't exist, the container is already gone.
		// This is not an error — DEL must be idempotent.
		if os.IsNotExist(err) {
			return nil
		}
		return fmt.Errorf("failed to open netns: %w", err)
	}
	defer netNS.Close()

	return netNS.Do(func(_ ns.NetNS) error {
			link, err := netlink.LinkByName(ifName)
			if err != nil {
				// Interface doesn't exist — already cleaned up. Not an error.
				return nil
			}

			// Deleting the container-side veth automatically deletes the
			// host-side veth too (kernel handles this for veth pairs).
			if err := netlink.LinkDel(link); err != nil {
				return fmt.Errorf("failed to delete %s: %w", ifName, err)
			}

			return nil
		})
}

// randomVethName generates a random name for the host-side veth interface.
//
// Linux interface names are limited to 15 characters (IFNAMSIZ - 1).
// We use "veth" + 8 hex characters = 12 characters, well within the limit.
//
// The random suffix prevents collisions when multiple CNI ADD operations
// run concurrently on the same node.
//
// Returns:
//   - string: a name like "veth1a2b3c4d"
//   - error: if reading from the random source fails
func randomVethName() (string, error) {
	bytes := make([]byte, 4)
	if _, err := rand.Read(bytes); err != nil {
		return "", fmt.Errorf("failed to generate random bytes: %w", err)
	}
	return fmt.Sprintf("veth%x", bytes), nil
}
```

### main.go — Entry Point and CNI Command Dispatch

```go
// main.go
//
// This is the entry point for our CNI plugin. It uses the CNI plugin SDK
// (github.com/containernetworking/cni/pkg/skel) to handle the CNI protocol:
//
//   - Parsing environment variables (CNI_COMMAND, CNI_CONTAINERID, etc.)
//   - Reading the JSON configuration from stdin
//   - Dispatching to the appropriate handler (ADD, DEL, CHECK)
//   - Formatting and writing the result to stdout
//
// The skel.PluginMain function handles all the boilerplate, so our code
// only needs to implement the three handler functions: cmdAdd, cmdDel,
// and cmdCheck.
//
// Build:
//
//	CGO_ENABLED=0 go build -o my-cni-plugin .
//
// Install:
//
//	sudo cp my-cni-plugin /opt/cni/bin/
//
// Test manually:
//
//	export CNI_COMMAND=ADD
//	export CNI_CONTAINERID=test-123
//	export CNI_NETNS=/var/run/netns/test
//	export CNI_IFNAME=eth0
//	export CNI_PATH=/opt/cni/bin
//	echo '{"cniVersion":"1.0.0","name":"test","type":"my-cni-plugin","subnet":"10.99.0.0/24"}' | ./my-cni-plugin
package main

import (
	"encoding/json"
	"fmt"
	"net"
	"os"
	"runtime"

	"github.com/containernetworking/cni/pkg/skel"
	"github.com/containernetworking/cni/pkg/types"
	current "github.com/containernetworking/cni/pkg/types/100"
	"github.com/containernetworking/cni/pkg/version"
)

// cmdAdd is called when CNI_COMMAND=ADD. It performs the full network
// setup for a container:
//
//  1. Parse the network configuration from stdin
//  2. Create (or reuse) a Linux bridge
//  3. Allocate an IP address for the container
//  4. Create a veth pair (host-side attached to bridge, container-side in netns)
//  5. Configure the container's interface with the allocated IP
//  6. Set up the default route via the bridge gateway
//  7. Return the result (IP, routes, DNS) as JSON on stdout
//
// The container runtime (containerd/CRI-O) calls this function for every
// new Pod sandbox. The CNI_NETNS environment variable points to the
// sandbox's network namespace, and CNI_IFNAME is the desired interface
// name (usually "eth0").
//
// Parameters:
//   - args: the CNI arguments (container ID, netns path, interface name, config)
//
// Returns:
//   - error: if any step fails. The error is returned to the runtime,
//     which will mark the Pod as failed.
func cmdAdd(args *skel.CmdArgs) error {
	// Parse the JSON configuration from stdin
	conf, err := parseConfig(args.StdinData)
	if err != nil {
		return fmt.Errorf("failed to parse config: %w", err)
	}

	// Parse the gateway IP and subnet
	gatewayIP := net.ParseIP(conf.Gateway)
	if gatewayIP == nil {
		return fmt.Errorf("invalid gateway IP: %s", conf.Gateway)
	}

	_, subnet, err := net.ParseCIDR(conf.Subnet)
	if err != nil {
		return fmt.Errorf("invalid subnet: %w", err)
	}

	// The gateway needs the subnet mask for the bridge interface
	gatewayIPNet := &net.IPNet{
		IP:   gatewayIP,
		Mask: subnet.Mask,
	}

	// Step 1: Create or reuse the bridge
	bridge, err := ensureBridge(conf.BridgeName, gatewayIPNet, conf.MTU)
	if err != nil {
		return fmt.Errorf("failed to ensure bridge: %w", err)
	}

	// Step 2: Allocate an IP address for this container
	containerIP, containerIPNet, err := allocateIP(conf, args.ContainerID)
	if err != nil {
		return fmt.Errorf("IPAM allocation failed: %w", err)
	}

	// Step 3: Create a veth pair and move one end into the container's namespace
	hostVethName, err := createVethPair(bridge, args.Netns, args.IfName, conf.MTU)
	if err != nil {
		// Clean up: release the allocated IP if veth creation fails
		releaseIP(conf, args.ContainerID)
		return fmt.Errorf("failed to create veth pair: %w", err)
	}

	// Step 4: Configure the container's interface with the allocated IP
	macAddr, err := configureContainerNetworking(
		args.Netns, args.IfName, containerIPNet, gatewayIP,
	)
	if err != nil {
		releaseIP(conf, args.ContainerID)
		return fmt.Errorf("failed to configure container networking: %w", err)
	}

	// Step 5: Build and return the CNI result.
	//
	// The result tells the runtime what IP was assigned, what the gateway
	// is, and what DNS servers to configure in the container's
	// /etc/resolv.conf.
	result := &current.Result{
		CNIVersion: conf.CNIVersion,
		Interfaces: []*current.Interface{
			{
				// The bridge interface on the host
				Name: conf.BridgeName,
				Mac:  bridge.Attrs().HardwareAddr.String(),
			},
			{
				// The host-side veth
				Name: hostVethName,
			},
			{
				// The container-side interface
				Name:    args.IfName,
				Mac:     macAddr,
				Sandbox: args.Netns,
			},
		},
		IPs: []*current.IPConfig{
			{
				// The IP assigned to the container
				Address:   net.IPNet{IP: containerIP, Mask: subnet.Mask},
				Gateway:   gatewayIP,
				Interface: intPtr(2), // Index into Interfaces — the container iface
			},
		},
		Routes: []*types.Route{
			{
				// Default route via the bridge gateway
				Dst: net.IPNet{
					IP:   net.IPv4zero,
					Mask: net.CIDRMask(0, 32),
				},
				GW: gatewayIP,
			},
		},
	}

	// Serialize the result to stdout.
	// The CNI library handles the actual writing.
	return types.PrintResult(result, conf.CNIVersion)
}

// cmdDel is called when CNI_COMMAND=DEL. It tears down the container's
// networking:
//
//  1. Delete the veth pair (deleting one end automatically deletes the other)
//  2. Release the container's IP address back to the IPAM pool
//
// DEL MUST be idempotent — it must succeed even if ADD was never called
// or if DEL was already called. This is because the runtime may call
// DEL multiple times (e.g., on timeout and retry).
//
// Parameters:
//   - args: the CNI arguments (container ID, netns path, interface name, config)
//
// Returns:
//   - error: if cleanup fails in a non-recoverable way
func cmdDel(args *skel.CmdArgs) error {
	conf, err := parseConfig(args.StdinData)
	if err != nil {
		return fmt.Errorf("failed to parse config: %w", err)
	}

	// Delete the container's veth interface.
	// This automatically deletes the host-side veth and detaches from bridge.
	// If the netns or interface doesn't exist, this is a no-op (idempotent).
	if args.Netns != "" {
		if err := deleteContainerVeth(args.Netns, args.IfName); err != nil {
			return fmt.Errorf("failed to delete veth: %w", err)
		}
	}

	// Release the IP address back to the pool
	if err := releaseIP(conf, args.ContainerID); err != nil {
		return fmt.Errorf("failed to release IP: %w", err)
	}

	return nil
}

// cmdCheck is called when CNI_COMMAND=CHECK. It verifies that the
// container's networking is correctly configured.
//
// This function validates:
//   - The container's network interface exists
//   - The interface has the expected IP address
//   - The default route is in place
//
// Parameters:
//   - args: the CNI arguments
//
// Returns:
//   - error: if the networking is misconfigured
func cmdCheck(args *skel.CmdArgs) error {
	// For this minimal implementation, we just verify the interface exists
	// in the container's namespace and has an IP address.
	//
	// A production plugin would also verify:
	//   - The IP matches the IPAM allocation
	//   - The default route is correct
	//   - The bridge exists and the host-side veth is attached
	//   - ARP entries are valid
	//   - iptables rules are in place

	_, err := parseConfig(args.StdinData)
	if err != nil {
		return fmt.Errorf("failed to parse config: %w", err)
	}

	// In a full implementation, we would enter the container's netns
	// and verify the interface configuration here.
	// For brevity, we return nil (network is "OK").

	return nil
}

// intPtr is a helper that returns a pointer to an int.
// Used for the Interface field in current.IPConfig, which takes *int.
func intPtr(i int) *int {
	return &i
}

// main is the entry point. It delegates to the CNI plugin SDK which handles:
//   - Parsing CNI_COMMAND, CNI_CONTAINERID, CNI_NETNS, CNI_IFNAME
//   - Reading stdin
//   - Calling the appropriate handler
//   - Writing the result to stdout or stderr
//
// We support CNI spec versions 0.3.0 through 1.0.0.
func main() {
	// Lock the main goroutine to its OS thread.
	// This is required because network namespace operations (setns)
	// are per-thread in Linux, and Go's goroutine scheduler can move
	// goroutines between OS threads.
	runtime.LockOSThread()

	skel.PluginMain(
		cmdAdd,
		cmdCheck,
		cmdDel,
		version.All, // Support all CNI spec versions
		"my-cni-plugin v0.1.0",
	)
}
```

### go.mod

```
module github.com/example/my-cni-plugin

go 1.21

require (
    github.com/containernetworking/cni v1.1.2
    github.com/containernetworking/plugins v1.4.0
    github.com/vishvananda/netlink v1.2.1-beta.2
)
```

### Building and Testing

```bash
# Build the plugin (static binary for portability)
CGO_ENABLED=0 GOOS=linux go build -o my-cni-plugin .

# Install the plugin
sudo cp my-cni-plugin /opt/cni/bin/

# Create a CNI configuration file
sudo tee /etc/cni/net.d/99-my-plugin.conflist <<EOF
{
  "cniVersion": "1.0.0",
  "name": "my-test-network",
  "plugins": [
    {
      "type": "my-cni-plugin",
      "bridge": "cni-test0",
      "subnet": "10.99.0.0/24",
      "gateway": "10.99.0.1",
      "mtu": 1500
    }
  ]
}
EOF

# Test manually with a network namespace
sudo ip netns add test-container

export CNI_COMMAND=ADD
export CNI_CONTAINERID=test-manual-001
export CNI_NETNS=/var/run/netns/test-container
export CNI_IFNAME=eth0
export CNI_PATH=/opt/cni/bin

echo '{
  "cniVersion": "1.0.0",
  "name": "my-test-network",
  "type": "my-cni-plugin",
  "bridge": "cni-test0",
  "subnet": "10.99.0.0/24",
  "gateway": "10.99.0.1",
  "mtu": 1500
}' | sudo -E /opt/cni/bin/my-cni-plugin

# Verify the container has networking
sudo ip netns exec test-container ip addr show eth0
# eth0: ... inet 10.99.0.2/24 ...

sudo ip netns exec test-container ip route
# default via 10.99.0.1 dev eth0

# Test connectivity from container to gateway
sudo ip netns exec test-container ping -c 1 10.99.0.1

# Clean up
export CNI_COMMAND=DEL
echo '{
  "cniVersion": "1.0.0",
  "name": "my-test-network",
  "type": "my-cni-plugin",
  "bridge": "cni-test0",
  "subnet": "10.99.0.0/24"
}' | sudo -E /opt/cni/bin/my-cni-plugin

sudo ip netns del test-container
```

---

## Popular CNI Implementations

### Flannel

Flannel is the simplest and most widely deployed CNI plugin. It was created by
CoreOS (now part of Red Hat) and provides basic overlay networking.

#### How Flannel Works

```
┌─────────────────────────────┐    ┌─────────────────────────────┐
│         Node 1               │    │         Node 2               │
│  Subnet: 10.244.1.0/24      │    │  Subnet: 10.244.2.0/24      │
│                              │    │                              │
│  ┌────────┐  ┌────────┐     │    │     ┌────────┐  ┌────────┐  │
│  │ Pod A  │  │ Pod B  │     │    │     │ Pod C  │  │ Pod D  │  │
│  │.1.2    │  │.1.3    │     │    │     │.2.2    │  │.2.3    │  │
│  └───┬────┘  └───┬────┘     │    │     └───┬────┘  └───┬────┘  │
│      │           │          │    │         │           │       │
│  ┌───┴───────────┴───┐      │    │     ┌───┴───────────┴───┐   │
│  │    cni0 bridge    │      │    │     │    cni0 bridge    │   │
│  │   10.244.1.1      │      │    │     │   10.244.2.1      │   │
│  └────────┬──────────┘      │    │     └────────┬──────────┘   │
│           │                  │    │              │              │
│  ┌────────┴──────────┐      │    │     ┌────────┴──────────┐   │
│  │   flannel.1       │      │    │     │   flannel.1       │   │
│  │   VXLAN Interface │      │    │     │   VXLAN Interface │   │
│  │   VNI: 1          │      │    │     │   VNI: 1          │   │
│  └────────┬──────────┘      │    │     └────────┬──────────┘   │
│           │ UDP:8472         │    │              │              │
│  ┌────────┴──────────┐      │    │     ┌────────┴──────────┐   │
│  │   eth0            │      │    │     │   eth0            │   │
│  │   192.168.1.10    │◄─────┼────┼────►│   192.168.1.11    │   │
│  └───────────────────┘      │    │     └───────────────────┘   │
└─────────────────────────────┘    └─────────────────────────────┘
                    Physical Network (192.168.1.0/24)
```

**Flannel's approach:**

1. **Subnet allocation**: Each node gets a unique /24 subnet from a larger CIDR
   (e.g., 10.244.0.0/16). Flannel stores subnet assignments in etcd or the
   Kubernetes API (via a ConfigMap or annotation on Node objects).

2. **VXLAN encapsulation**: When Pod A (10.244.1.2 on Node 1) sends a packet to
   Pod C (10.244.2.2 on Node 2), the packet hits the `flannel.1` VXLAN interface,
   which encapsulates the original Ethernet frame inside a UDP packet (port 8472)
   with the outer source IP as 192.168.1.10 and outer destination as 192.168.1.11.

3. **flanneld daemon**: A `flanneld` process runs on each node as a DaemonSet. It:
   - Watches Node objects for subnet assignments
   - Configures the VXLAN interface with the correct FDB (forwarding database)
     entries so the kernel knows which VTEP (VXLAN Tunnel Endpoint) to use for
     each remote subnet
   - Writes the subnet configuration to a file that the Flannel CNI plugin reads

**Advantages**: Simple, works everywhere, minimal configuration.
**Disadvantages**: No network policies, overlay adds latency and CPU overhead,
limited to VXLAN or host-gw backends.

### Calico

Calico is a networking and network policy engine. It's the most popular choice for
production Kubernetes clusters that need network policies.

#### BGP-Based Routing (Default)

Calico's default mode uses BGP (Border Gateway Protocol) to distribute routes
between nodes. No overlay — packets are routed natively at Layer 3.

```
┌─────────────────────────────┐    ┌─────────────────────────────┐
│         Node 1               │    │         Node 2               │
│  Subnet: 10.244.1.0/24      │    │  Subnet: 10.244.2.0/24      │
│                              │    │                              │
│  ┌────────┐  ┌────────┐     │    │     ┌────────┐  ┌────────┐  │
│  │ Pod A  │  │ Pod B  │     │    │     │ Pod C  │  │ Pod D  │  │
│  │.1.2    │  │.1.3    │     │    │     │.2.2    │  │.2.3    │  │
│  └───┬────┘  └───┬────┘     │    │     └───┬────┘  └───┬────┘  │
│      │           │          │    │         │           │       │
│  Routing table:              │    │  Routing table:             │
│  10.244.2.0/24 via 192...11  │    │  10.244.1.0/24 via 192...10│
│                              │    │                              │
│  ┌───────────────────┐      │    │     ┌───────────────────┐   │
│  │   eth0            │      │    │     │   eth0            │   │
│  │   192.168.1.10    │◄─────┼────┼────►│   192.168.1.11    │   │
│  └───────────────────┘      │    │     └───────────────────┘   │
│                              │    │                              │
│  BIRD BGP daemon ◄───────────┼────┼───► BIRD BGP daemon        │
│  (peering with Node 2)      │    │     (peering with Node 1)   │
└─────────────────────────────┘    └─────────────────────────────┘
```

**How it works:**

1. **Per-node routing**: Instead of a bridge, Calico uses proxy ARP and routing.
   Each Pod gets a point-to-point veth pair, and the host has a route to the Pod's
   IP via the veth.

2. **BGP peering**: The BIRD BGP daemon on each node peers with BIRD on other nodes
   (full mesh by default, or via route reflectors at scale). Each node advertises
   its local Pod subnets to all other nodes.

3. **No encapsulation**: Packets between Pods on different nodes are routed natively.
   The source is 10.244.1.2, the destination is 10.244.2.2, and the packet travels
   across the physical network with these IPs. **No overhead.**

4. **Network policies**: Calico programs iptables rules (or eBPF programs) on each
   node to enforce Kubernetes NetworkPolicy resources. This is the primary reason
   teams choose Calico.

**VXLAN mode**: When BGP isn't possible (e.g., in cloud environments where the
network doesn't support BGP or doesn't allow IP forwarding for unknown IPs), Calico
can fall back to VXLAN overlay, similar to Flannel.

**Data planes**: Calico supports two data planes:
- **iptables** (default): Proven, stable, works everywhere
- **eBPF**: Higher performance, replaces kube-proxy, supports direct server return

### Cilium

Cilium is the newest and most technically advanced CNI plugin. It's built entirely
on eBPF (extended Berkeley Packet Filter), a Linux kernel technology that allows
running sandboxed programs inside the kernel without modifying kernel source code.

#### Why eBPF Changes Everything

Traditional CNI plugins (Flannel, Calico with iptables) use iptables for packet
filtering and NAT. iptables uses a linear rule chain — for every packet, the kernel
walks through every rule until it finds a match. At scale (thousands of Services,
millions of connections), this becomes a bottleneck.

eBPF replaces iptables with hash-map lookups in the kernel. Instead of a linear
scan through thousands of rules, a single hash lookup finds the matching rule in
O(1) time.

```
iptables (traditional):                    eBPF (Cilium):

Packet arrives                             Packet arrives
    │                                          │
    ├─ Rule 1: No match                        ├─ Hash lookup:
    ├─ Rule 2: No match                        │   key = (srcIP, dstIP, dstPort)
    ├─ Rule 3: No match                        │   → Service backend found
    ├─ ...                                     │   → DNAT to Pod IP
    ├─ Rule 4,827: Match!                      │
    │   → DNAT to Pod IP                       └─ Done (O(1))
    │
    └─ O(n) where n = number of rules
```

#### Cilium Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          Node                                    │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │
│  │  Pod A   │  │  Pod B   │  │  Pod C   │                      │
│  │  eth0    │  │  eth0    │  │  eth0    │                      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                      │
│       │              │              │                            │
│  ┌────┴──────────────┴──────────────┴──────┐                    │
│  │                                         │                    │
│  │           eBPF Programs                 │                    │
│  │                                         │                    │
│  │  ┌──────────┐ ┌────────┐ ┌──────────┐  │                    │
│  │  │ Ingress  │ │  NAT   │ │ Egress   │  │                    │
│  │  │ Policy   │ │  (svc  │ │ Policy   │  │                    │
│  │  │ Enforce  │ │  LB)   │ │ Enforce  │  │                    │
│  │  └──────────┘ └────────┘ └──────────┘  │                    │
│  │                                         │                    │
│  │  ┌──────────┐ ┌────────┐ ┌──────────┐  │                    │
│  │  │ L7 Proxy │ │Conntrk │ │ Hubble   │  │                    │
│  │  │ (HTTP,   │ │(BPF    │ │(observ-  │  │                    │
│  │  │  gRPC)   │ │ maps)  │ │ ability) │  │                    │
│  │  └──────────┘ └────────┘ └──────────┘  │                    │
│  │                                         │                    │
│  └─────────────────────────────────────────┘                    │
│                                                                 │
│  ┌─────────────────────────────────┐                            │
│  │  cilium-agent (DaemonSet)       │                            │
│  │  - Watches K8s API              │                            │
│  │  - Compiles and loads eBPF      │                            │
│  │  - Manages maps and policies    │                            │
│  └─────────────────────────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
```

**Cilium's key features:**

1. **Replaces kube-proxy**: Cilium handles Service load balancing in eBPF, eliminating
   the need for kube-proxy entirely. This removes thousands of iptables rules.

2. **Identity-based policies**: Instead of IP-based policies (which break with
   dynamic IPs), Cilium assigns security identities to Pods based on labels. Policies
   reference identities, not IPs.

3. **L7 policies**: Cilium can enforce HTTP and gRPC-level policies:
   ```yaml
   # Allow only GET requests to /api/public/*
   apiVersion: cilium.io/v2
   kind: CiliumNetworkPolicy
   metadata:
     name: l7-rule
   spec:
     endpointSelector:
       matchLabels:
         app: api-server
     ingress:
     - fromEndpoints:
       - matchLabels:
           app: frontend
       toPorts:
       - ports:
         - port: "80"
           protocol: TCP
         rules:
           http:
           - method: "GET"
             path: "/api/public/.*"
   ```

4. **Hubble**: Built-in observability platform that captures network flows at the
   eBPF level. Provides a UI similar to Wireshark but for Kubernetes:
   ```bash
   # Watch all HTTP traffic in real-time
   hubble observe --protocol http --follow
   
   # See dropped packets (policy violations)
   hubble observe --verdict DROPPED
   ```

5. **Service mesh**: Cilium can act as a sidecar-free service mesh, providing mTLS,
   L7 load balancing, and traffic management without injecting sidecar proxies.

**Why Cilium is the future**: eBPF is to networking what containers were to compute
— a fundamental shift in how we think about the problem. By moving networking logic
into the kernel with eBPF, Cilium achieves performance that iptables-based solutions
cannot match, while providing visibility that was previously impossible without
sidecar proxies.

### Weave Net

Weave Net creates a mesh overlay network using VXLAN or a userspace fallback
("sleeve mode") when kernel features aren't available.

**Key features:**
- **Encryption**: Built-in IPsec encryption for overlay traffic
- **Multicast**: Supports multicast and broadcast over the overlay
- **Automatic mesh**: Nodes discover each other and form a full mesh
- **DNS**: Built-in DNS server for service discovery

**Weave's niche**: Environments where you need encryption without a service mesh,
or where you need multicast support. Otherwise, Calico or Cilium are generally
preferred.

### Comparison Table

| Feature              | Flannel     | Calico        | Cilium           | Weave Net    |
|----------------------|-------------|---------------|------------------|--------------|
| **Overlay**          | VXLAN       | VXLAN, IPIP   | VXLAN, Geneve    | VXLAN, Sleeve|
| **Native routing**   | host-gw     | BGP           | Direct routing   | No           |
| **Network Policy**   | No          | Yes (iptables/eBPF)| Yes (eBPF)  | Yes (limited)|
| **L7 Policy**        | No          | No            | Yes (HTTP, gRPC) | No           |
| **Encryption**       | No (use WireGuard)| WireGuard  | WireGuard, IPsec | IPsec       |
| **Replaces kube-proxy**| No       | Yes (eBPF mode)| Yes              | No           |
| **Observability**    | Minimal     | Flow logs     | Hubble           | Weave Scope  |
| **Performance**      | Good        | Very Good     | Best             | Good         |
| **Complexity**       | Low         | Medium        | High             | Low          |
| **Best for**         | Dev/Simple  | Production    | Advanced/Scale   | Simple+Encrypt|

---

## IPAM (IP Address Management)

IPAM is the subsystem responsible for allocating and tracking IP addresses for
containers. In CNI, IPAM is implemented as a separate plugin that the main plugin
delegates to.

### host-local

The most common IPAM plugin. It allocates IPs from configured ranges and stores
the allocation state in local files on each node.

```json
{
  "ipam": {
    "type": "host-local",
    "ranges": [
      [
        {
          "subnet": "10.244.1.0/24",
          "rangeStart": "10.244.1.10",
          "rangeEnd": "10.244.1.200",
          "gateway": "10.244.1.1"
        }
      ]
    ],
    "routes": [
      { "dst": "0.0.0.0/0" }
    ],
    "dataDir": "/var/lib/cni/networks"
  }
}
```

**State storage**: host-local stores allocations as files in `/var/lib/cni/networks/<network-name>/`.
Each file is named with the IP address and contains the container ID:

```bash
ls /var/lib/cni/networks/my-network/
# 10.244.1.2    10.244.1.3    10.244.1.4    last_reserved_ip.0    lock

cat /var/lib/cni/networks/my-network/10.244.1.2
# abc123def456    # Container ID
```

**Limitation**: host-local is node-scoped. It has no cluster-wide coordination. Each
node must be assigned a non-overlapping subnet. The orchestrator (Kubernetes) handles
subnet assignment via the node's `podCIDR`.

### DHCP

The DHCP IPAM plugin requests an IP from an external DHCP server. This is useful when
integrating with existing network infrastructure that uses DHCP for IP management.

```json
{
  "ipam": {
    "type": "dhcp"
  }
}
```

The DHCP plugin runs a daemon (`dhcp daemon`) that maintains DHCP leases on behalf of
containers.

### Whereabouts

Whereabouts is a cluster-wide IPAM plugin that coordinates IP allocation across
nodes using the Kubernetes API (CRDs) as the backing store.

```json
{
  "ipam": {
    "type": "whereabouts",
    "range": "10.244.0.0/16",
    "range_start": "10.244.0.10",
    "range_end": "10.244.255.250"
  }
}
```

**Advantages over host-local:**
- Cluster-wide IP coordination (no pre-assigned per-node subnets)
- IP range overlap detection
- Garbage collection of stale allocations
- Works with Multus for secondary network interfaces

### Hands-On: IPAM Configuration Examples

```json
// host-local with multiple ranges (dual-stack IPv4 + IPv6)
{
  "ipam": {
    "type": "host-local",
    "ranges": [
      [
        {
          "subnet": "10.244.1.0/24",
          "gateway": "10.244.1.1"
        }
      ],
      [
        {
          "subnet": "fd00:244:1::/64",
          "gateway": "fd00:244:1::1"
        }
      ]
    ],
    "routes": [
      { "dst": "0.0.0.0/0" },
      { "dst": "::/0" }
    ]
  }
}
```

```json
// Static IPAM — assigns a fixed IP (useful for testing or special Pods)
{
  "ipam": {
    "type": "static",
    "addresses": [
      {
        "address": "10.244.1.100/24",
        "gateway": "10.244.1.1"
      }
    ]
  }
}
```

---

## Advanced: Multi-Network (Multus)

### The Problem

By default, Kubernetes gives each Pod exactly one network interface (plus loopback).
But some workloads need multiple network interfaces:

- **Telco/NFV**: Network functions need separate data plane and control plane interfaces
- **SR-IOV**: Direct hardware NIC access for high-performance networking
- **DPDK**: Userspace networking for packet processing
- **Management networks**: Separate interface for monitoring/management traffic

### Multus CNI

Multus is a "meta-plugin" — a CNI plugin that calls other CNI plugins. It delegates
the primary interface to the cluster's main CNI plugin (Calico, Cilium, etc.) and
creates additional interfaces using other CNI plugins.

```
┌──────────────────────────────────────────────────┐
│                     Pod                           │
│                                                  │
│  ┌──────┐    ┌──────────┐    ┌───────────────┐   │
│  │ eth0 │    │   net1    │    │    net2       │   │
│  │Calico│    │  macvlan  │    │   SR-IOV      │   │
│  │10.244│    │  192.168  │    │   (hardware)  │   │
│  │.1.5  │    │  .100.5   │    │   10.0.0.5    │   │
│  └──┬───┘    └────┬──────┘    └──────┬────────┘   │
└─────┼─────────────┼─────────────────┼─────────────┘
      │             │                 │
      │ Primary     │ Secondary       │ Secondary
      │ (Calico)    │ (macvlan)       │ (sriov)
      ▼             ▼                 ▼
   Cluster       Physical          VF (Virtual
   Network       Network           Function)
```

### Network Attachment Definitions

Multus uses a CRD called `NetworkAttachmentDefinition` to describe additional
networks:

```yaml
# Create a macvlan network attachment
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: macvlan-conf
  namespace: default
spec:
  config: |
    {
      "cniVersion": "1.0.0",
      "type": "macvlan",
      "master": "eth0",
      "mode": "bridge",
      "ipam": {
        "type": "host-local",
        "subnet": "192.168.100.0/24",
        "rangeStart": "192.168.100.100",
        "rangeEnd": "192.168.100.200"
      }
    }
```

```yaml
# Create an SR-IOV network attachment for high-performance networking
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: sriov-net
  namespace: default
spec:
  config: |
    {
      "cniVersion": "1.0.0",
      "type": "sriov",
      "vlan": 100,
      "ipam": {
        "type": "whereabouts",
        "range": "10.0.0.0/24"
      }
    }
```

```yaml
# Pod requesting multiple interfaces
apiVersion: v1
kind: Pod
metadata:
  name: multi-net-pod
  annotations:
    # Comma-separated list of NetworkAttachmentDefinitions
    k8s.v1.cni.cncf.io/networks: macvlan-conf, sriov-net
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "ip addr show && sleep 3600"]
    resources:
      requests:
        intel.com/sriov_netdevice: "1"   # SR-IOV requires resource request
      limits:
        intel.com/sriov_netdevice: "1"
```

```bash
# Check the Pod's interfaces
kubectl exec multi-net-pod -- ip addr
# 1: lo: <LOOPBACK,UP,LOWER_UP>
#     inet 127.0.0.1/8
# 2: eth0@...: <BROADCAST,MULTICAST,UP,LOWER_UP>          ← Primary (Calico)
#     inet 10.244.1.5/24
# 3: net1@...: <BROADCAST,MULTICAST,UP,LOWER_UP>          ← macvlan-conf
#     inet 192.168.100.105/24
# 4: net2@...: <BROADCAST,MULTICAST,UP,LOWER_UP>          ← sriov-net
#     inet 10.0.0.5/24
```

---

## Debugging CNI Issues

CNI failures are one of the most common reasons Pods get stuck in
`ContainerCreating` state. Here's a systematic approach to debugging.

### Common Failures

#### 1. CNI Binary Not Found

```
Warning  FailedCreatePodSandBox  kubelet  Failed to create pod sandbox:
  rpc error: ... failed to find plugin "calico" in path [/opt/cni/bin]
```

**Cause**: The CNI plugin binary is missing from `/opt/cni/bin/`.
**Fix**: Verify the CNI plugin DaemonSet is running and has copied the binaries.

```bash
# Check if the binary exists
ls -la /opt/cni/bin/calico

# Check if the CNI DaemonSet Pods are running
kubectl get pods -n kube-system -l k8s-app=calico-node
```

#### 2. CNI Config Not Found

```
Warning  FailedCreatePodSandBox  kubelet  Failed to create pod sandbox:
  rpc error: ... no networks found in /etc/cni/net.d
```

**Cause**: No CNI configuration file in `/etc/cni/net.d/`.
**Fix**: The CNI DaemonSet hasn't written its config yet, or it failed.

```bash
# Check for config files
ls -la /etc/cni/net.d/

# Check CNI DaemonSet logs
kubectl logs -n kube-system -l k8s-app=calico-node --tail=50
```

#### 3. IP Exhaustion

```
Warning  FailedCreatePodSandBox  kubelet  Failed to create pod sandbox:
  rpc error: ... no IP addresses available in range set
```

**Cause**: All IPs in the node's Pod CIDR have been allocated.
**Fix**: Increase the cluster CIDR, check for leaked IPs, or reduce Pod density.

```bash
# Check how many IPs are allocated on this node
ls /var/lib/cni/networks/<network-name>/ | wc -l

# Check node's podCIDR
kubectl get node <node-name> -o jsonpath='{.spec.podCIDR}'

# Look for orphaned IPs (IPs allocated but no corresponding Pod)
# Compare allocated IPs with running Pods
kubectl get pods --field-selector spec.nodeName=<node-name> -o wide
```

#### 4. Invalid CNI Config

```
Warning  FailedCreatePodSandBox  kubelet  Failed to create pod sandbox:
  rpc error: ... error parsing configuration: invalid character '}' ...
```

**Cause**: Malformed JSON in the CNI config file.
**Fix**: Validate the JSON.

```bash
# Validate JSON syntax
python3 -m json.tool /etc/cni/net.d/10-calico.conflist

# Or use jq
cat /etc/cni/net.d/10-calico.conflist | jq .
```

### Diagnostic Commands

```bash
# 1. Check Pod status and events
kubectl describe pod <pod-name>

# 2. Check kubelet logs for CNI errors
journalctl -u kubelet --since "5 minutes ago" | grep -i cni

# 3. Check CNI plugin logs (varies by plugin)
# Calico:
kubectl logs -n kube-system -l k8s-app=calico-node -c calico-node --tail=100

# Cilium:
kubectl logs -n kube-system -l k8s-app=cilium --tail=100

# 4. Check the node's network configuration
kubectl debug node/<node-name> -it --image=nicolaka/netshoot -- bash

# Inside the debug container:
# Check CNI config
cat /host/etc/cni/net.d/*.conflist | python3 -m json.tool

# Check CNI binaries
ls -la /host/opt/cni/bin/

# Check IP allocations
ls /host/var/lib/cni/networks/

# Check interfaces and routes
ip addr show
ip route show
iptables -t nat -L -n | head -50

# Check bridge
brctl show
# or
ip link show type bridge

# 5. Test connectivity from a Pod
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- bash

# Inside the Pod:
ip addr show
ip route show
nslookup kubernetes.default.svc.cluster.local
ping -c 3 <other-pod-ip>
traceroute <other-pod-ip>
curl -v http://<service-ip>:<port>

# 6. Check Cilium status (if using Cilium)
kubectl exec -n kube-system -l k8s-app=cilium -- cilium status
kubectl exec -n kube-system -l k8s-app=cilium -- cilium bpf endpoint list
kubectl exec -n kube-system -l k8s-app=cilium -- cilium bpf ct list global | head

# 7. Check Calico status (if using Calico)
kubectl exec -n kube-system -l k8s-app=calico-node -- calicoctl node status
kubectl exec -n kube-system -l k8s-app=calico-node -- calicoctl get nodes -o wide
```

### Hands-On: Debugging CNI Step-by-Step

Scenario: Pods are stuck in `ContainerCreating` on a newly provisioned node.

```bash
# Step 1: Check Pod events
kubectl describe pod stuck-pod
# Events:
#   Warning  FailedCreatePodSandBox  kubelet
#   Failed to create pod sandbox: rpc error: ...
#   plugin type="calico" failed: error getting ClusterInformation: ...

# Step 2: Check the CNI DaemonSet on this node
kubectl get pods -n kube-system -l k8s-app=calico-node \
  --field-selector spec.nodeName=worker-3
# NAME                READY   STATUS            RESTARTS
# calico-node-xyz     0/1     CrashLoopBackOff  5

# Step 3: Check the DaemonSet Pod logs
kubectl logs -n kube-system calico-node-xyz -c calico-node --previous
# ERROR: Failed to reach apiserver: connection refused

# Step 4: Root cause — the node can't reach the API server
# Check network connectivity from the node
kubectl debug node/worker-3 -it --image=busybox -- wget -T 5 -q -O- \
  https://kubernetes.default.svc:443/healthz
# wget: can't connect to remote host

# Step 5: Fix — the node's routing table is missing a route to the
# API server's service CIDR. Add it:
# (This would be done on the node itself)
ip route add 10.96.0.0/12 via 192.168.1.1 dev eth0

# Step 6: Verify the CNI DaemonSet Pod recovers
kubectl get pods -n kube-system -l k8s-app=calico-node \
  --field-selector spec.nodeName=worker-3 -w
# calico-node-xyz     1/1     Running     0

# Step 7: Verify stuck Pods start
kubectl get pods -w
# stuck-pod    0/1     ContainerCreating → Running
```

---

## Summary

CNI is the foundation of Kubernetes networking. Understanding it means understanding:

1. **The specification**: A simple protocol (env vars + JSON stdin/stdout) that
   decouples container runtimes from network implementations.

2. **The plugin ecosystem**: From simple bridges to eBPF-powered service meshes,
   all built on the same interface.

3. **The networking primitives**: Bridges, veth pairs, network namespaces, routes,
   and iptables/eBPF — the Linux kernel features that make container networking
   possible.

| Concept              | Key Takeaway                                              |
|----------------------|-----------------------------------------------------------|
| CNI Spec             | Binary + env vars + JSON stdin → JSON stdout              |
| ADD/DEL/CHECK        | Create, destroy, verify container networking              |
| Plugin chaining      | Compose simple plugins into complex configurations        |
| Bridge plugin        | Linux bridge + veth pairs + IPAM                          |
| Flannel              | Simple VXLAN overlay, no network policies                 |
| Calico               | BGP routing + iptables/eBPF network policies              |
| Cilium               | Full eBPF — replaces kube-proxy, L7 policies, Hubble      |
| IPAM                 | host-local (per-node), Whereabouts (cluster-wide)         |
| Multus               | Multiple network interfaces per Pod                       |
| Debugging            | Check binary, config, logs, IPs, routes                   |

The CNI plugin you choose has a massive impact on your cluster's performance,
security, and operational complexity. For development: Flannel. For production with
network policies: Calico. For the future — high performance, observability, service
mesh: Cilium.

In the next chapter, we'll look at the Kubernetes scheduler — how it decides which
node each Pod runs on, and how storage topology (from Chapter 21) and network
topology (from this chapter) influence that decision.
