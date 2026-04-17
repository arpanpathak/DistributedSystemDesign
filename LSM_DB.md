# Building a High‑Write‑Throughput Database: An LSM‑Engine Deep Dive

If you’re aiming for millions of writes per second on commodity hardware, you quickly learn that **in‑place updates are the enemy**. Every random disk I/O kills your throughput. The solution? Turn all writes into sequential appends. That’s the core insight behind **Log‑Structured Merge‑Trees (LSM Trees)** – the engine powering RocksDB, Cassandra, and ScyllaDB.

In this post, I’ll walk through the architecture, the trade‑offs, and a clean, production‑inspired Go implementation of a write‑optimised key‑value store.

---

## 📖 Table of Contents

1. [The Core Philosophy: Sequential > Random](#1-the-core-philosophy-sequential--random)  
2. [Real‑World Use Cases](#2-real‑world-use-cases)  
3. [Data Structures Deep Dive](#3-data-structures-deep-dive)  
   - [Write‑Ahead Log (WAL) and `fsync`](#write-ahead-log-wal-and-fsync)  
   - [MemTable](#memtable)  
   - [Skip List](#skip-list)  
   - [SSTable (Sorted String Table)](#sstable-sorted-string-table)  
   - [Bloom Filter](#bloom-filter)  
   - [Compaction](#compaction)  
4. [A Production‑Minded Go Implementation](#4-a-production‑minded-go-implementation)  
   - [Project Structure](#project-structure)  
   - [Package `wal`](#package-wal)  
   - [Package `skiplist`](#package-skiplist)  
   - [Package `memtable`](#package-memtable)  
   - [Package `bloom`](#package-bloom)  
   - [Package `sstable`](#package-sstable)  
   - [Package `compaction`](#package-compaction)  
   - [Package `database`](#package-database)  
   - [CLI Example `cmd/lsmcli/main.go`](#cli-example-cmdlsmclimaingo)  
5. [Scaling to a Distributed World](#5-scaling-to-a-distributed-world)  
6. [The Gotchas: Retry Storms and Cascading Failures](#6-the-gotchas-retry-storms-and-cascading-failures)  
7. [Fault Tolerance and the Operator Pattern](#7-fault-tolerance-and-the-operator-pattern)  
8. [The Final Reasoning](#8-the-final-reasoning)  

---

## 1. The Core Philosophy: Sequential > Random

A spinning disk does ~100 random IOs per second, but **hundreds of MB/s** of sequential writes. Even NVMe SSDs favour sequential patterns under high queue depths. An LSM database:

- **Appends every write** to a Write‑Ahead Log (WAL) – durability without seeks.
- **Buffers writes in memory** (MemTable) – usually a balanced tree or skip list.
- **Flushes sorted immutable files** (SSTables) to disk when the MemTable fills up.
- **Compacts files in the background** to control read amplification and reclaim space.

Reads may need to check multiple files, but writes stay lightning‑fast.

---

## 2. Real‑World Use Cases

| Domain | Why LSM? |
|--------|-----------|
| **High‑frequency trading** | Microsecond‑level write latency, replay‑safe WAL. |
| **IoT telemetry** | Millions of sensors, each appending timestamped values. |
| **Observability (Prometheus, Loki)** | High‑cardinality labels, time‑series append‑only workload. |
| **Message queue offsets** | Sequential commit log with periodic snapshots. |
| **Edge caching** | Crash‑recoverable write buffer with minimal disk chatter. |

Any system that can trade some read speed and eventual consistency for **ingest at hardware speed** is a prime candidate.

---

## 3. Data Structures Deep Dive

### Write‑Ahead Log (WAL) and `fsync`

A WAL is an append‑only file on disk. Every write is written to the WAL before it updates the in‑memory MemTable. If the process crashes, the WAL is replayed to rebuild the MemTable.

`fsync` is a system call that forces data from the operating system’s page cache to the physical disk. Without `fsync`, a power loss could lose writes that the OS claimed were written. Calling `fsync` on every write makes it durable but slower. Production systems batch writes or use `fdatasync` (skips metadata updates) to balance safety and speed.

**Go struct:**
```go
type WAL struct {
    file   *os.File      // underlying file on disk
    writer *bufio.Writer // buffered writer to reduce system calls
    mu     sync.Mutex    // serialise appends for thread safety
}
```

### MemTable

The MemTable is an in‑memory write buffer. All writes go here first. When the MemTable reaches a size limit (e.g., 32 MB), it is flushed to disk as an SSTable. This batching converts many random writes into one sequential write. A skip list is often used to keep keys sorted for efficient range scans during flush and compaction.

**Go struct (using skip list):**
```go
type MemTable struct {
    mu    sync.RWMutex
    data  *skiplist.SkipList // sorted in‑memory store
    size  int                // approximate entry count
    limit int                // max entries before flush
}
```

### Skip List

A skip list is a probabilistic data structure that maintains a sorted sequence. It consists of multiple layers of linked lists. The bottom layer contains all elements. Each higher layer acts as an “express lane” that skips many nodes. Nodes are given a random height (geometric distribution). Search, insert, and delete run in O(log N) expected time.

**Why use a skip list instead of a hash map?**  
- Maintains keys in sorted order for efficient range scans (critical for compactions).  
- Simpler to implement lock‑free versions for high concurrency compared to balanced trees.

**Go structs:**
```go
// Node represents one element in the skip list.
type Node struct {
    key   string
    value string
    next  []*Node // next[i] points to the next node at level i (0 = base level)
}

// SkipList is a concurrent‑safe sorted map.
type SkipList struct {
    head   *Node          // dummy head node with empty key
    height int            // current maximum level
    rand   *rand.Rand     // random source for node heights
    mu     sync.RWMutex   // protects all fields
}
```

### SSTable (Sorted String Table)

An SSTable is an immutable, sorted file on disk. Once written, it is never modified. New writes create new SSTable files. The file format is simple: each line is `key:value\n`, and keys are sorted. An in‑memory index (key → file offset) allows binary search on the sorted keys plus a single disk seek to read the value. This gives O(log N) read performance.

**Go struct:**
```go
// SSTable represents an immutable sorted file on disk.
type SSTable struct {
    path   string            // absolute file path
    keys   []string          // sorted list of all keys (for binary search)
    offset map[string]int64  // key → byte offset in the file
}
```

### Bloom Filter

A Bloom filter is a probabilistic data structure that tests whether an element is **definitely not** in a set or **maybe** in the set. It consists of a bit array of size `m` and `k` independent hash functions. To add a key, set bits at `h1(key), h2(key), ..., hk(key)`. To test, if any of those bits is 0, the key is definitely absent. If all are 1, it may be present (false positives possible, no false negatives).

**Parameters:**  
- `m = -n * ln(p) / (ln(2)^2)` where `n` = expected number of keys, `p` = desired false positive probability.  
- `k = (m/n) * ln(2)`.

Bloom filters drastically reduce disk reads for missing keys by avoiding SSTable lookups when the filter says “definitely not present”.

**Go struct:**
```go
// BloomFilter implements a classic Bloom filter.
type BloomFilter struct {
    bits []bool  // bit array
    m    uint    // number of bits
    k    uint    // number of hash functions
}
```

### Compaction

Compaction merges multiple SSTables into one. This limits the total number of files (read amplification) and removes overwritten or deleted keys (space amplification). Two common strategies:

- **Size‑tiered compaction:** Merge N same‑size files into one larger file. Simple but can cause write amplification spikes.
- **Leveled compaction:** Data is organised into levels. Level 0 comes directly from MemTable flushes (keys may overlap). Higher levels are sorted and non‑overlapping. Compaction moves data from level L to L+1, merging and sorting. Better read performance, higher write amplification.

**Go struct:**
```go
// Compaction holds configuration for a merge operation.
type Compaction struct {
    strategy string                // "size_tiered" or "leveled"
    inputs   []*sstable.SSTable    // tables to merge
    output   string                // path for the new SSTable
}
```

---

## 4. A Production‑Minded Go Implementation

Let’s build a minimal but complete LSM store. It includes:

- **WAL** – `fsync`’ed on every write (optional batching for higher throughput).
- **MemTable** – a skip list protected by a mutex.
- **Flush** – writes a sorted segment file (`.sst`).
- **Basic compaction** – merges two segments into one (single‑threaded, for clarity).
- **Read path** – checks MemTable first, then scans SSTables from newest to oldest.
- **Bloom filter** – optional, integrated into SSTable for fast negative lookups.

We’ll organise the code into separate packages for modularity, with **generous documentation comments** on every exported type, function, and critical internal logic.

### Project Structure

```
lsmdb/
├── go.mod
├── cmd/
│   └── lsmcli/
│       └── main.go
├── wal/
│   └── wal.go
├── skiplist/
│   └── skiplist.go
├── memtable/
│   └── memtable.go
├── bloom/
│   └── bloom.go
├── sstable/
│   ├── sstable.go
│   └── bloom_filter.go   # optional integration
├── compaction/
│   └── compaction.go
└── database/
    └── db.go
```

### Package `wal`

```go
// Package wal provides a write‑ahead log for durability.
// All writes are appended to a file with fsync to guarantee persistence.
package wal

import (
	"bufio"
	"os"
	"sync"
)

// WAL represents a write‑ahead log file.
type WAL struct {
	file   *os.File      // underlying file on disk
	writer *bufio.Writer // buffered writer to reduce syscalls
	mu     sync.Mutex    // ensures atomic appends
}

// Open creates or opens a WAL file at the given path.
// If the file does not exist, it is created with 0644 permissions.
// The file is opened in append‑only mode.
func Open(path string) (*WAL, error) {
	// O_APPEND: all writes go to the end of the file
	// O_CREAT: create if missing
	// O_RDWR: we need read for recovery
	f, err := os.OpenFile(path, os.O_APPEND|os.O_CREATE|os.O_RDWR, 0644)
	if err != nil {
		return nil, err
	}
	return &WAL{
		file:   f,
		writer: bufio.NewWriter(f),
	}, nil
}

// Append writes a key‑value pair to the end of the WAL.
// The format is "key:value\n". It flushes the buffer and calls fsync
// to ensure the data is on stable storage before returning.
func (w *WAL) Append(key, value string) error {
	w.mu.Lock()
	defer w.mu.Unlock()

	// Write to buffered writer first (reduces system calls)
	if _, err := w.writer.WriteString(key + ":" + value + "\n"); err != nil {
		return err
	}
	// Flush the buffer to the OS page cache
	if err := w.writer.Flush(); err != nil {
		return err
	}
	// Force the OS cache to disk (fsync) – critical for durability
	return w.file.Sync()
}

// Replay reads the entire WAL from the beginning and calls the callback
// for every key‑value pair found. Used during startup to rebuild the MemTable.
func (w *WAL) Replay(callback func(key, value string)) error {
	w.mu.Lock()
	defer w.mu.Unlock()

	// Seek to the start of the file
	if _, err := w.file.Seek(0, 0); err != nil {
		return err
	}
	scanner := bufio.NewScanner(w.file)
	for scanner.Scan() {
		line := scanner.Text()
		// Find the colon separator (naive; production code handles escaped colons)
		for i := 0; i < len(line); i++ {
			if line[i] == ':' {
				callback(line[:i], line[i+1:])
				break
			}
		}
	}
	return scanner.Err()
}

// Close releases the WAL file handle.
func (w *WAL) Close() error {
	return w.file.Close()
}
```

### Package `skiplist`

```go
// Package skiplist implements a concurrent‑safe skip list.
// It maintains keys in sorted order and provides O(log N) average lookups.
package skiplist

import (
	"math/rand"
	"sync"
	"time"
)

const maxHeight = 16 // enough for 2^16 elements with high probability

// Node represents a single element in the skip list.
type Node struct {
	key   string
	value string
	next  []*Node // next[i] is the next node at level i (0 = base level)
}

// SkipList is a probabilistic sorted map.
type SkipList struct {
	head   *Node          // dummy head node (key is empty string)
	height int            // current maximum level of the list
	rand   *rand.Rand     // random source for node heights
	mu     sync.RWMutex   // protects all fields
}

// New creates a new empty skip list.
func New() *SkipList {
	// Initialise head node with maxHeight empty pointers
	head := &Node{
		next: make([]*Node, maxHeight),
	}
	return &SkipList{
		head:   head,
		height: 1,
		rand:   rand.New(rand.NewSource(time.Now().UnixNano())),
	}
}

// randomHeight returns a random level for a new node.
// The probability of level i is 2^(-i). This ensures O(log N) expected height.
func (s *SkipList) randomHeight() int {
	h := 1
	for h < maxHeight && s.rand.Intn(2) == 0 {
		h++
	}
	return h
}

// Put inserts or updates a key‑value pair.
func (s *SkipList) Put(key, value string) {
	s.mu.Lock()
	defer s.mu.Unlock()

	// Find predecessors at each level
	prev := make([]*Node, maxHeight)
	x := s.head
	for i := s.height - 1; i >= 0; i-- {
		// Move forward at level i while the next node's key is less than target
		for x.next[i] != nil && x.next[i].key < key {
			x = x.next[i]
		}
		prev[i] = x
	}
	// After loop, x.next[0] is the first node with key >= target
	if x.next[0] != nil && x.next[0].key == key {
		// Key exists: update value
		x.next[0].value = value
		return
	}
	// Key does not exist: insert new node
	newHeight := s.randomHeight()
	if newHeight > s.height {
		// If new node is taller, set predecessors for the extra levels to head
		for i := s.height; i < newHeight; i++ {
			prev[i] = s.head
		}
		s.height = newHeight
	}
	newNode := &Node{
		key:   key,
		value: value,
		next:  make([]*Node, newHeight),
	}
	// Insert at all levels
	for i := 0; i < newHeight; i++ {
		newNode.next[i] = prev[i].next[i]
		prev[i].next[i] = newNode
	}
}

// Get retrieves the value for a key. Returns (value, true) if found, otherwise ("", false).
func (s *SkipList) Get(key string) (string, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()

	x := s.head
	// Traverse from highest level down
	for i := s.height - 1; i >= 0; i-- {
		for x.next[i] != nil && x.next[i].key < key {
			x = x.next[i]
		}
	}
	// x.next[0] is the first node with key >= target
	if x.next[0] != nil && x.next[0].key == key {
		return x.next[0].value, true
	}
	return "", false
}

// Snapshot returns a map of all key‑value pairs in sorted order.
// This is used when flushing the MemTable to disk.
func (s *SkipList) Snapshot() map[string]string {
	s.mu.RLock()
	defer s.mu.RUnlock()

	result := make(map[string]string)
	x := s.head.next[0] // start at the first real node
	for x != nil {
		result[x.key] = x.value
		x = x.next[0]
	}
	return result
}
```

### Package `memtable`

```go
// Package memtable provides an in‑memory write buffer using a skip list.
package memtable

import (
	"lsmdb/skiplist"
	"sync"
)

// MemTable is a thread‑safe write buffer with a size limit.
type MemTable struct {
	mu    sync.RWMutex
	data  *skiplist.SkipList // sorted in‑memory store
	size  int                // approximate entry count
	limit int                // max entries before a flush is triggered
}

// New creates a MemTable with the given size limit.
// Once the number of entries reaches 'limit', IsFull() returns true.
func New(limit int) *MemTable {
	return &MemTable{
		data:  skiplist.New(),
		limit: limit,
	}
}

// Put inserts or updates a key‑value pair.
// It increments the size only if the key is new (not an update).
func (m *MemTable) Put(key, value string) {
	m.mu.Lock()
	defer m.mu.Unlock()
	// Check if key already exists to avoid overcounting size
	if _, ok := m.data.Get(key); !ok {
		m.size++
	}
	m.data.Put(key, value)
}

// Get retrieves a value by key. The boolean indicates whether the key exists.
func (m *MemTable) Get(key string) (string, bool) {
	m.mu.RLock()
	defer m.mu.RUnlock()
	return m.data.Get(key)
}

// IsFull returns true if the MemTable has reached its size limit.
func (m *MemTable) IsFull() bool {
	m.mu.RLock()
	defer m.mu.RUnlock()
	return m.size >= m.limit
}

// Snapshot returns a copy of all data and resets the MemTable.
// This is used when flushing to disk: we take a snapshot, then the caller
// writes it to an SSTable, while new writes go to a fresh MemTable.
func (m *MemTable) Snapshot() map[string]string {
	m.mu.Lock()
	defer m.mu.Unlock()
	snap := m.data.Snapshot()          // grab all data
	m.data = skiplist.New()            // allocate a new empty skip list
	m.size = 0                         // reset size
	return snap
}
```

### Package `bloom`

```go
// Package bloom implements a Bloom filter for probabilistic membership tests.
package bloom

import (
	"hash/fnv"
	"math"
)

// BloomFilter is a probabilistic set membership data structure.
type BloomFilter struct {
	bits []bool // bit array
	m    uint   // number of bits
	k    uint   // number of hash functions
}

// New creates a Bloom filter with expected number of items and false positive rate.
// Formula: m = -n * ln(p) / (ln(2)^2)
//          k = (m/n) * ln(2)
func New(expectedItems int, falsePositiveRate float64) *BloomFilter {
	n := float64(expectedItems)
	p := falsePositiveRate
	ln2 := math.Ln2
	// Calculate optimal number of bits
	m := uint(-n * math.Log(p) / (ln2 * ln2))
	// Calculate optimal number of hash functions
	k := uint(float64(m) / n * ln2)
	return &BloomFilter{
		bits: make([]bool, m),
		m:    m,
		k:    k,
	}
}

// hash returns a 64‑bit hash of the key with a seed.
// Using FNV‑1a for simplicity.
func (bf *BloomFilter) hash(key string, seed uint32) uint64 {
	h := fnv.New64a()
	h.Write([]byte(key))
	// Mix the seed into the hash
	return h.Sum64() ^ uint64(seed)
}

// Add inserts a key into the Bloom filter.
func (bf *BloomFilter) Add(key string) {
	// Generate k independent hashes using double hashing technique
	// For simplicity, we use the same hash function with different seeds
	for i := uint(0); i < bf.k; i++ {
		hashVal := bf.hash(key, uint32(i))
		bitPos := hashVal % uint64(bf.m)
		bf.bits[bitPos] = true
	}
}

// MightContain returns true if the key might be in the set.
// If it returns false, the key is definitely not in the set.
func (bf *BloomFilter) MightContain(key string) bool {
	for i := uint(0); i < bf.k; i++ {
		hashVal := bf.hash(key, uint32(i))
		bitPos := hashVal % uint64(bf.m)
		if !bf.bits[bitPos] {
			return false // definitely not present
		}
	}
	return true // maybe present
}
```

### Package `sstable`

```go
// Package sstable manages immutable, sorted files on disk.
package sstable

import (
	"bufio"
	"lsmdb/bloom"
	"os"
	"sort"
)

// SSTable represents an immutable sorted file on disk.
type SSTable struct {
	path   string            // absolute file path
	keys   []string          // sorted list of all keys
	offset map[string]int64  // key → byte offset in the file
	filter *bloom.BloomFilter // optional Bloom filter for fast negative lookups
}

// Create writes a new SSTable file from a snapshot map.
// The map can be in any order; this function sorts the keys and writes
// a line‑oriented format "key:value\n". It also builds an optional Bloom filter.
func Create(path string, data map[string]string, useBloom bool, bloomExpected int) error {
	f, err := os.Create(path)
	if err != nil {
		return err
	}
	defer f.Close()

	// 1. Sort all keys so the file is ordered
	keys := make([]string, 0, len(data))
	for k := range data {
		keys = append(keys, k)
	}
	sort.Strings(keys)

	// 2. Write each key‑value pair and record offsets
	writer := bufio.NewWriter(f)
	offsets := make(map[string]int64)
	for _, k := range keys {
		pos, _ := f.Seek(0, 1) // current offset from start
		offsets[k] = pos
		if _, err := writer.WriteString(k + ":" + data[k] + "\n"); err != nil {
			return err
		}
	}
	if err := writer.Flush(); err != nil {
		return err
	}

	// 3. Optionally build a Bloom filter for this SSTable
	var filter *bloom.BloomFilter
	if useBloom {
		filter = bloom.New(bloomExpected, 0.01) // 1% false positive rate
		for k := range data {
			filter.Add(k)
		}
	}
	// For simplicity, we don't persist the Bloom filter to disk in this example.
	// In production, you would write it to a separate file or embed it.
	_ = filter
	return nil
}

// Open loads an existing SSTable file and rebuilds its in‑memory index.
// It reads the entire file once to record offsets for binary search later.
func Open(path string) (*SSTable, error) {
	f, err := os.Open(path)
	if err != nil {
		return nil, err
	}
	defer f.Close()

	sst := &SSTable{
		path:   path,
		keys:   []string{},
		offset: make(map[string]int64),
	}
	scanner := bufio.NewScanner(f)
	var pos int64 = 0
	for scanner.Scan() {
		line := scanner.Text()
		// Find the colon separator
		for i := 0; i < len(line); i++ {
			if line[i] == ':' {
				key := line[:i]
				sst.keys = append(sst.keys, key)
				sst.offset[key] = pos
				break
			}
		}
		// Advance position: length of line + 1 for the newline
		pos += int64(len(line)) + 1
	}
	return sst, scanner.Err()
}

// Get looks up a key in the SSTable using binary search on the keys slice,
// then reads the value directly from the file at the recorded offset.
// If a Bloom filter is present and says "definitely not present", it returns empty quickly.
func (s *SSTable) Get(key string) (string, error) {
	// Optional Bloom filter check
	if s.filter != nil && !s.filter.MightContain(key) {
		return "", nil // definitely not present
	}
	// Binary search on sorted keys
	idx := sort.SearchStrings(s.keys, key)
	if idx >= len(s.keys) || s.keys[idx] != key {
		return "", nil // not found
	}
	// Open the file, seek to the offset, read the line
	f, err := os.Open(s.path)
	if err != nil {
		return "", err
	}
	defer f.Close()
	if _, err := f.Seek(s.offset[key], 0); err != nil {
		return "", err
	}
	reader := bufio.NewReader(f)
	line, err := reader.ReadString('\n')
	if err != nil {
		return "", err
	}
	// Extract value after colon
	for i := 0; i < len(line); i++ {
		if line[i] == ':' {
			return line[i+1 : len(line)-1], nil // trim newline
		}
	}
	return "", nil
}

// Path returns the file path of this SSTable.
func (s *SSTable) Path() string {
	return s.path
}

// Keys returns the sorted list of keys (for compaction).
func (s *SSTable) Keys() []string {
	return s.keys
}
```

### Package `compaction`

```go
// Package compaction merges multiple SSTables into one to reduce read amplification.
package compaction

import (
	"lsmdb/sstable"
)

// Merge takes a list of SSTables and produces a single new SSTable that
// contains the latest value for each key (last file in the list wins).
// This is a simplified "size‑tiered" compaction strategy.
func Merge(outputPath string, tables []*sstable.SSTable, useBloom bool) error {
	// 1. Collect all keys from all tables (may contain duplicates)
	allKeys := make(map[string]bool)
	for _, t := range tables {
		for _, k := range t.Keys() {
			allKeys[k] = true
		}
	}
	// 2. For each key, find the newest value (by scanning tables from newest to oldest)
	merged := make(map[string]string)
	for k := range allKeys {
		// Tables are provided in order from oldest to newest.
		// We scan backwards so the last table (newest) wins.
		for i := len(tables) - 1; i >= 0; i-- {
			val, err := tables[i].Get(k)
			if err == nil && val != "" {
				merged[k] = val
				break
			}
		}
	}
	// 3. Write the merged data to a new SSTable
	return sstable.Create(outputPath, merged, useBloom, len(merged))
}
```

### Package `database`

```go
// Package database glues together WAL, MemTable, and SSTables.
// It handles writes (append to WAL, insert into MemTable, trigger flush),
// reads (check MemTable first, then SSTables from newest to oldest),
// and recovery (replay WAL on startup).
package database

import (
	"fmt"
	"lsmdb/compaction"
	"lsmdb/memtable"
	"lsmdb/sstable"
	"lsmdb/wal"
	"os"
	"path/filepath"
	"sort"
)

// DB is the main LSM database handle.
type DB struct {
	dir       string
	wal       *wal.WAL
	memtable  *memtable.MemTable
	sstables  []*sstable.SSTable // sorted from oldest to newest
	flushSize int
}

// Open creates or loads a database from a directory.
func Open(dir string) (*DB, error) {
	if err := os.MkdirAll(dir, 0700); err != nil {
		return nil, err
	}
	walPath := filepath.Join(dir, "wal.log")
	w, err := wal.Open(walPath)
	if err != nil {
		return nil, err
	}
	db := &DB{
		dir:       dir,
		wal:       w,
		memtable:  memtable.New(100), // 100 entries for demo; real: 32MB worth
		flushSize: 100,
	}
	// Recover from WAL: replay all entries into MemTable
	if err := db.recover(); err != nil {
		return nil, err
	}
	// Load existing SSTables from disk
	if err := db.loadSSTables(); err != nil {
		return nil, err
	}
	return db, nil
}

// recover replays the WAL to rebuild the MemTable.
func (db *DB) recover() error {
	return db.wal.Replay(func(key, value string) {
		db.memtable.Put(key, value)
	})
}

// loadSSTables reads all segment_*.sst files and opens them as SSTables.
func (db *DB) loadSSTables() error {
	files, err := filepath.Glob(filepath.Join(db.dir, "segment_*.sst"))
	if err != nil {
		return err
	}
	sort.Strings(files) // older first by naming convention
	for _, f := range files {
		sst, err := sstable.Open(f)
		if err != nil {
			return err
		}
		db.sstables = append(db.sstables, sst)
	}
	return nil
}

// Put inserts or updates a key‑value pair.
func (db *DB) Put(key, value string) error {
	// 1. Write to WAL for durability
	if err := db.wal.Append(key, value); err != nil {
		return err
	}
	// 2. Insert into MemTable
	db.memtable.Put(key, value)
	// 3. If MemTable is full, flush it to disk
	if db.memtable.IsFull() {
		if err := db.flush(); err != nil {
			return err
		}
	}
	return nil
}

// flush takes a snapshot of the MemTable, writes it as a new SSTable,
// and then optionally triggers compaction.
func (db *DB) flush() error {
	// Snapshot current MemTable
	data := db.memtable.Snapshot()
	if len(data) == 0 {
		return nil
	}
	// Create a new SSTable file
	segID := len(db.sstables) + 1
	path := filepath.Join(db.dir, fmt.Sprintf("segment_%05d.sst", segID))
	if err := sstable.Create(path, data, true, len(data)); err != nil {
		return err
	}
	// Open the new SSTable and add to the list
	sst, err := sstable.Open(path)
	if err != nil {
		return err
	}
	db.sstables = append(db.sstables, sst)
	// Optionally run compaction if there are too many files
	if len(db.sstables) >= 4 {
		if err := db.runCompaction(); err != nil {
			return err
		}
	}
	return nil
}

// runCompaction merges the two oldest SSTables into one and removes the old ones.
func (db *DB) runCompaction() error {
	if len(db.sstables) < 2 {
		return nil
	}
	// Take the two oldest SSTables
	toCompact := db.sstables[:2]
	outputPath := filepath.Join(db.dir, fmt.Sprintf("compacted_%d.sst", len(db.sstables)))
	if err := compaction.Merge(outputPath, toCompact, true); err != nil {
		return err
	}
	// Open the new compacted SSTable
	newSST, err := sstable.Open(outputPath)
	if err != nil {
		return err
	}
	// Remove the old files from disk and from the slice
	for _, old := range toCompact {
		os.Remove(old.Path())
	}
	// Replace the two old ones with the new one
	db.sstables = append([]*sstable.SSTable{newSST}, db.sstables[2:]...)
	return nil
}

// Get retrieves the value for a key.
// It checks the MemTable first, then scans SSTables from newest to oldest.
func (db *DB) Get(key string) (string, bool) {
	// Check MemTable first
	if val, ok := db.memtable.Get(key); ok {
		return val, true
	}
	// Check SSTables from newest to oldest
	for i := len(db.sstables) - 1; i >= 0; i-- {
		val, err := db.sstables[i].Get(key)
		if err == nil && val != "" {
			return val, true
		}
	}
	return "", false
}

// Close flushes any remaining data and closes the WAL.
func (db *DB) Close() error {
	// Force a final flush if there is data in MemTable
	if db.memtable.IsFull() {
		db.flush()
	}
	return db.wal.Close()
}
```

### CLI Example `cmd/lsmcli/main.go`

```go
// Command lsmcli is a simple command‑line interface for the LSM database.
package main

import (
	"flag"
	"fmt"
	"lsmdb/database"
	"os"
)

func main() {
	dir := flag.String("dir", "./data", "database directory")
	flag.Parse()
	db, err := database.Open(*dir)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to open database: %v\n", err)
		os.Exit(1)
	}
	defer db.Close()

	if flag.NArg() < 1 {
		fmt.Println("Usage: lsmcli <put|get> [key] [value]")
		os.Exit(1)
	}
	switch flag.Arg(0) {
	case "put":
		if flag.NArg() != 3 {
			fmt.Println("put requires key and value")
			os.Exit(1)
		}
		key := flag.Arg(1)
		value := flag.Arg(2)
		if err := db.Put(key, value); err != nil {
			fmt.Fprintf(os.Stderr, "Put error: %v\n", err)
			os.Exit(1)
		}
		fmt.Printf("Put %q = %q\n", key, value)
	case "get":
		if flag.NArg() != 2 {
			fmt.Println("get requires key")
			os.Exit(1)
		}
		key := flag.Arg(1)
		val, ok := db.Get(key)
		if !ok {
			fmt.Printf("Key %q not found\n", key)
			os.Exit(1)
		}
		fmt.Printf("%q\n", val)
	default:
		fmt.Printf("Unknown command: %s\n", flag.Arg(0))
		os.Exit(1)
	}
}
```

---

## 5. Scaling to a Distributed World

Once you move past a single machine, things get spicy. To get high availability, you need replication. But if you wait for every replica to confirm a write, your throughput tanks. This is the classic CAP theorem trade off. You usually pick **Eventual Consistency** where you write to a leader and it syncs to followers in the background.

---

## 6. The Gotchas: Retry Storms and Cascading Failures

In a distributed system, network blips are a fact of life. If a node goes down and every client starts retrying immediately, you get a **Retry Storm** that acts like a self inflicted DDoS attack. The fix is using **Exponential Backoff with Jitter**. You also need to watch out for **Write Amplification** where background compaction starts fighting with your active writes for disk bandwidth.

---

## 7. Fault Tolerance and the Operator Pattern

To make this truly resilient, you use the **Operator pattern**. This is like having a robot SRE watching your database. If a pod dies, the Operator detects the missing state, attaches the persistent volume to a new node, and replays the WAL to bring the memory state back to life. It handles the healing so you can sleep at night.

---

## 8. Conclusion

Building for high writes is about accepting that you can't be perfect everywhere. You trade some read speed (because you might have to check multiple files) and some immediate consistency for the ability to ingest data at the speed of your hardware. It is pure engineering craftsmanship where you optimize for the bottleneck and let the system breathe.
