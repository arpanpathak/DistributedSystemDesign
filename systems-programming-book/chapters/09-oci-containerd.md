# Chapter 9: OCI, containerd & the Container Ecosystem

> *"Standardization is what turns a clever hack into infrastructure."*
> — Solomon Hykes, creator of Docker

In Chapter 8, we built containers from scratch using Linux namespaces, cgroups, and
overlay filesystems. We saw how a container is, at its core, just a specially-configured
Linux process. But the container world doesn't run on one-off hacks — it runs on
*standards* and *layered software*. This chapter is where we zoom out from the kernel
primitives and explore the industrial-strength ecosystem that makes containers
portable, composable, and production-ready.

We will cover the **Open Container Initiative (OCI)** specifications that standardize
images and runtimes, the **runc** reference runtime, the **containerd** daemon that
orchestrates container lifecycle, the **Container Runtime Interface (CRI)** that
connects Kubernetes to runtimes, and the broader ecosystem of sandbox runtimes,
rootless containers, and image distribution. By the end, you will understand every
layer from `docker run` down to the `clone()` system call.

---

## 9.1 The OCI (Open Container Initiative)

### 9.1.1 Why OCI Exists — The Standardization Story

In 2013, Docker exploded onto the scene. It made containers accessible to every
developer by combining Linux namespaces, cgroups, and union filesystems into a
single, easy-to-use tool. But Docker was one company's product. As containers
became critical infrastructure, the industry faced a problem: **vendor lock-in**.

If Docker defined the image format, the runtime behavior, and the distribution
protocol, then every other player — CoreOS, Google, Red Hat, Microsoft — was
building on Docker's proprietary foundation. CoreOS created its own container
runtime (rkt) with its own image format (ACI), and suddenly there were
incompatible container ecosystems.

In June 2015, Docker, CoreOS, Google, Microsoft, AWS, and others founded the
**Open Container Initiative (OCI)** under the Linux Foundation. Docker donated
its container runtime code (which became **runc**) and its image format as the
starting point. The goal was simple: **define open standards so that containers
are portable across any runtime, any platform, any cloud.**

The OCI produces three specifications:

1. **OCI Image Specification** — How container images are built, stored, and
   distributed. Defines the manifest, config, and layer formats.

2. **OCI Runtime Specification** — How a container is configured and executed.
   Defines the `config.json` bundle format and the container lifecycle.

3. **OCI Distribution Specification** — How container images are pushed to and
   pulled from registries. Defines the HTTP API for registry interaction.

```text
    ┌──────────────────────────────────────────────────────────────┐
    │                  OCI Specifications                          │
    │                                                              │
    │  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐  │
    │  │   Image      │  │   Runtime    │  │   Distribution     │  │
    │  │   Spec       │  │   Spec       │  │   Spec             │  │
    │  │             │  │             │  │                    │  │
    │  │  manifest   │  │  config.json │  │  Registry HTTP API │  │
    │  │  config     │  │  lifecycle   │  │  manifests, blobs  │  │
    │  │  layers     │  │  state       │  │  tags, catalogs    │  │
    │  └─────────────┘  └──────────────┘  └────────────────────┘  │
    │                                                              │
    │  Implementations:                                            │
    │  runc, crun, youki, kata-runtime, gVisor runsc              │
    │  containerd, CRI-O, Docker, Podman, Buildah, Skopeo        │
    └──────────────────────────────────────────────────────────────┘
```

The OCI specifications are intentionally minimal. They don't specify orchestration,
networking, storage drivers, or build systems. They specify just enough to ensure
that an image built by Docker can be run by containerd, CRI-O, or Podman — and
that a runtime bundle created by any tool can be executed by runc, crun, or youki.

### 9.1.2 OCI Image Specification

An OCI image is a **content-addressable** archive that describes a filesystem and
its metadata. Let's break down every component.

#### Image Manifest

The manifest is the top-level descriptor of an image. It points to the config and
the filesystem layers:

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:a1b2c3d4e5f6...",
    "size": 7023
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:1a2b3c4d5e6f...",
      "size": 32654947
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:6f5e4d3c2b1a...",
      "size": 16724
    }
  ],
  "annotations": {
    "org.opencontainers.image.authors": "Jane Developer"
  }
}
```

Key points:
- **`schemaVersion`**: Always 2 for current OCI images.
- **`config`**: A descriptor pointing to the image configuration blob.
- **`layers`**: An ordered list of filesystem layer descriptors. The first layer
  is the base; each subsequent layer is applied on top.
- **`annotations`**: Optional key-value metadata.

#### Content-Addressable Storage

Every blob in an OCI image is identified by its **SHA-256 digest**. The digest is
computed over the raw bytes of the blob. This gives us several powerful properties:

- **Integrity**: If a blob's content changes, its digest changes. You can verify
  any blob by re-hashing it.
- **Deduplication**: Two images sharing the same base layer will have the same
  digest for that layer. The layer is stored once.
- **Immutability**: You cannot change a blob without changing its address. This
  makes images inherently cacheable and trustworthy.

```
    Content-Addressable Storage
    ──────────────────────────────────────────

    blob bytes ──► SHA-256 ──► digest (address)

    sha256:a3ed95caeb02... ──► blobs/sha256/a3ed95caeb02...

    If content changes → digest changes → different address
    Same content always → same digest → same address (dedup)
```

#### Image Configuration

The config blob describes the runtime defaults and layer history:

```json
{
  "architecture": "amd64",
  "os": "linux",
  "config": {
    "Env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin"],
    "Cmd": ["/bin/sh"],
    "WorkingDir": "/",
    "ExposedPorts": { "8080/tcp": {} },
    "Labels": { "maintainer": "ops@example.com" }
  },
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:abc123...",
      "sha256:def456..."
    ]
  },
  "history": [
    {
      "created": "2024-01-15T10:30:00Z",
      "created_by": "/bin/sh -c #(nop) ADD file:abc123 in / "
    },
    {
      "created": "2024-01-15T10:30:01Z",
      "created_by": "/bin/sh -c apt-get update && apt-get install -y curl"
    }
  ]
}
```

The `diff_ids` in `rootfs` are the **uncompressed** digests of each layer, while
the manifest references the **compressed** digests. This distinction matters for
layer verification: when you download a compressed layer and decompress it, you
verify against the diff_id.

#### Layer Types

OCI defines several layer media types:

| Media Type | Description |
|---|---|
| `application/vnd.oci.image.layer.v1.tar` | Uncompressed tar archive |
| `application/vnd.oci.image.layer.v1.tar+gzip` | Gzip-compressed tar |
| `application/vnd.oci.image.layer.v1.tar+zstd` | Zstandard-compressed tar |
| `application/vnd.oci.image.layer.nondistributable.v1.tar` | Non-distributable layer |
| `application/vnd.oci.image.layer.nondistributable.v1.tar+gzip` | Non-distributable, gzip |

Non-distributable layers contain content that cannot be freely redistributed (e.g.,
proprietary Windows base layers). Registries must not push these layers to other
registries.

Each layer is a tar archive of filesystem changes:
- New files are simply present in the tar.
- Deleted files are represented by **whiteout files**: a file named
  `.wh.<filename>` signals that `<filename>` from a lower layer should be hidden.
- An **opaque whiteout** `.wh..wh..opq` in a directory means "hide everything
  from lower layers in this directory."

#### Image Index (Multi-Architecture)

A single image tag (like `ubuntu:22.04`) often needs to support multiple
architectures (amd64, arm64, s390x) and operating systems. The **image index**
(also called a manifest list in Docker terminology) solves this:

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.index.v1+json",
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:amd64manifest...",
      "size": 7143,
      "platform": {
        "architecture": "amd64",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "digest": "sha256:arm64manifest...",
      "size": 7143,
      "platform": {
        "architecture": "arm64",
        "os": "linux",
        "variant": "v8"
      }
    }
  ]
}
```

When you `docker pull ubuntu:22.04` on an ARM Mac, the client fetches the index,
finds the manifest matching your platform, and pulls only those layers.

### 9.1.3 Hands-on: Examining an OCI Image Layout with Go

An OCI image can be exported to a directory called an **OCI image layout**. Let's
write a Go program that reads and inspects this layout.

First, create an OCI layout to inspect:

```bash
# Pull an image and export it as an OCI layout
# (requires skopeo or crane)
skopeo copy docker://alpine:3.19 oci:alpine-oci:latest

# Or with crane:
crane pull --format=oci alpine:3.19 alpine-oci
```

The layout directory looks like:

```
alpine-oci/
├── blobs/
│   └── sha256/
│       ├── a3ed95ca...    (layer blob)
│       ├── b7d8f2e1...    (config blob)
│       └── c9f0e3d2...    (manifest blob)
├── index.json
└── oci-layout
```

Now let's inspect it programmatically:

```go
// Package ociinspect provides tools for examining OCI image layouts
// on disk. It demonstrates how content-addressable storage works in
// practice by reading the index, manifest, config, and layer metadata
// from an OCI image layout directory.
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"io"
	"os"
	"path/filepath"
	"strings"
)

// OCILayout represents the oci-layout file at the root of an OCI
// image layout directory. It contains the imageLayoutVersion field
// which must be "1.0.0" for compliant layouts.
type OCILayout struct {
	ImageLayoutVersion string `json:"imageLayoutVersion"`
}

// Descriptor is the OCI content descriptor, a fundamental building
// block of the OCI image specification. Every piece of content
// (manifests, configs, layers) is referenced by a descriptor that
// includes the media type, the content-addressable digest, and
// the byte size for verification.
type Descriptor struct {
	MediaType string `json:"mediaType"`
	Digest    string `json:"digest"`
	Size      int64  `json:"size"`
	// Platform is set in image index entries to indicate which
	// architecture/OS combination this manifest targets.
	Platform *Platform `json:"platform,omitempty"`
	// Annotations provides arbitrary metadata about the content.
	Annotations map[string]string `json:"annotations,omitempty"`
}

// Platform describes the platform (OS + architecture) that an image
// manifest targets. This is used in image indexes to allow clients
// to select the correct manifest for their system.
type Platform struct {
	Architecture string `json:"architecture"`
	OS           string `json:"os"`
	Variant      string `json:"variant,omitempty"`
}

// Index represents an OCI image index (or manifest list). It is the
// top-level entry point for multi-architecture images, listing one
// manifest descriptor per supported platform.
type Index struct {
	SchemaVersion int          `json:"schemaVersion"`
	MediaType     string       `json:"mediaType,omitempty"`
	Manifests     []Descriptor `json:"manifests"`
}

// Manifest represents a single OCI image manifest. It references
// exactly one config descriptor and an ordered list of layer
// descriptors. The layers are applied bottom-up to produce the
// final root filesystem.
type Manifest struct {
	SchemaVersion int          `json:"schemaVersion"`
	MediaType     string       `json:"mediaType,omitempty"`
	Config        Descriptor   `json:"config"`
	Layers        []Descriptor `json:"layers"`
}

// ImageConfig represents the OCI image configuration. It describes
// the default runtime parameters (environment variables, entrypoint,
	// working directory) and the rootfs layer chain (as uncompressed
	// diff_ids) used to reconstruct the container filesystem.
type ImageConfig struct {
	Architecture string       `json:"architecture"`
	OS           string       `json:"os"`
	Config       RunConfig    `json:"config"`
	RootFS       RootFS       `json:"rootfs"`
	History      []HistEntry  `json:"history"`
}

// RunConfig holds the default execution parameters that a container
// runtime should apply when starting a container from this image,
// unless overridden by the user.
type RunConfig struct {
	Env        []string          `json:"Env"`
	Cmd        []string          `json:"Cmd"`
	Entrypoint []string          `json:"Entrypoint"`
	WorkingDir string            `json:"WorkingDir"`
	Labels     map[string]string `json:"Labels"`
}

// RootFS describes the image's root filesystem as an ordered list of
// layer diff IDs. Each diff_id is the SHA-256 digest of the
// *uncompressed* layer tar, which allows verification after
// decompression.
type RootFS struct {
	Type    string   `json:"type"`
	DiffIDs []string `json:"diff_ids"`
}

// HistEntry records a single step in the image build history,
// typically corresponding to one Dockerfile instruction.
type HistEntry struct {
	Created    string `json:"created"`
	CreatedBy  string `json:"created_by"`
	EmptyLayer bool   `json:"empty_layer,omitempty"`
}

// readBlob reads a content-addressable blob from the OCI layout's
// blobs directory. The digest parameter should be in the format
// "sha256:<hex>", which is split to form the path blobs/sha256/<hex>.
func readBlob(layoutDir, digest string) ([]byte, error) {
	// OCI digests are formatted as "algorithm:hex". We split on ":"
	// to construct the filesystem path under the blobs directory.
	parts := strings.SplitN(digest, ":", 2)
	if len(parts) != 2 {
		return nil, fmt.Errorf("invalid digest format: %s", digest)
	}
	blobPath := filepath.Join(layoutDir, "blobs", parts[0], parts[1])
	return os.ReadFile(blobPath)
}

// verifyDigest computes the SHA-256 digest of the given data and
// compares it against the expected digest string. This demonstrates
// the content-addressable integrity verification that is fundamental
// to OCI image storage.
func verifyDigest(data []byte, expected string) bool {
	// Compute the SHA-256 hash of the raw blob bytes.
	h := sha256.Sum256(data)
	computed := "sha256:" + hex.EncodeToString(h[:])
	return computed == expected
}

func main() {
	if len(os.Args) < 2 {
		fmt.Fprintf(os.Stderr, "Usage: %s <oci-layout-dir>\n", os.Args[0])
		os.Exit(1)
	}
	layoutDir := os.Args[1]

	// Step 1: Read and verify the oci-layout file to ensure this
	// directory is a valid OCI image layout.
	layoutData, err := os.ReadFile(filepath.Join(layoutDir, "oci-layout"))
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error reading oci-layout: %v\n", err)
		os.Exit(1)
	}
	var layout OCILayout
	if err := json.Unmarshal(layoutData, &layout); err != nil {
		fmt.Fprintf(os.Stderr, "Error parsing oci-layout: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("OCI Layout Version: %s\n\n", layout.ImageLayoutVersion)

	// Step 2: Read the index.json, which is the entry point for
	// discovering all manifests in this layout.
	indexData, err := os.ReadFile(filepath.Join(layoutDir, "index.json"))
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error reading index.json: %v\n", err)
		os.Exit(1)
	}
	var index Index
	if err := json.Unmarshal(indexData, &index); err != nil {
		fmt.Fprintf(os.Stderr, "Error parsing index.json: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("Index contains %d manifest(s)\n\n", len(index.Manifests))

	// Step 3: For each manifest descriptor in the index, read the
	// manifest blob, verify its digest, and inspect its contents.
	for i, desc := range index.Manifests {
		fmt.Printf("--- Manifest %d ---\n", i)
		fmt.Printf("  MediaType: %s\n", desc.MediaType)
		fmt.Printf("  Digest:    %s\n", desc.Digest)
		fmt.Printf("  Size:      %d bytes\n", desc.Size)
		if desc.Platform != nil {
			fmt.Printf("  Platform:  %s/%s\n", desc.Platform.OS, desc.Platform.Architecture)
		}

		// Read the manifest blob from the content store.
		manifestData, err := readBlob(layoutDir, desc.Digest)
		if err != nil {
			fmt.Fprintf(os.Stderr, "  Error reading manifest: %v\n", err)
			continue
		}

		// Verify content integrity by recomputing the digest.
		if verifyDigest(manifestData, desc.Digest) {
			fmt.Println("  ✓ Digest verified")
		} else {
			fmt.Println("  ✗ DIGEST MISMATCH!")
		}

		var manifest Manifest
		if err := json.Unmarshal(manifestData, &manifest); err != nil {
			fmt.Fprintf(os.Stderr, "  Error parsing manifest: %v\n", err)
			continue
		}

		// Step 4: Read and display the image configuration.
		fmt.Printf("\n  Config: %s (%d bytes)\n", manifest.Config.Digest, manifest.Config.Size)
		configData, err := readBlob(layoutDir, manifest.Config.Digest)
		if err != nil {
			fmt.Fprintf(os.Stderr, "  Error reading config: %v\n", err)
			continue
		}

		var imgConfig ImageConfig
		if err := json.Unmarshal(configData, &imgConfig); err != nil {
			fmt.Fprintf(os.Stderr, "  Error parsing config: %v\n", err)
			continue
		}
		fmt.Printf("  Architecture: %s\n", imgConfig.Architecture)
		fmt.Printf("  OS:           %s\n", imgConfig.OS)
		fmt.Printf("  Cmd:          %v\n", imgConfig.Config.Cmd)
		fmt.Printf("  Env:          %v\n", imgConfig.Config.Env)
		fmt.Printf("  RootFS type:  %s\n", imgConfig.RootFS.Type)
		fmt.Printf("  Layers (diff_ids): %d\n", len(imgConfig.RootFS.DiffIDs))
		for j, diffID := range imgConfig.RootFS.DiffIDs {
			fmt.Printf("    [%d] %s\n", j, diffID)
		}

		// Step 5: List all filesystem layers referenced by the manifest.
		fmt.Printf("\n  Layers (compressed, from manifest): %d\n", len(manifest.Layers))
		for j, layer := range manifest.Layers {
			fmt.Printf("    [%d] %s\n", j, layer.MediaType)
			fmt.Printf("         digest: %s\n", layer.Digest)
			fmt.Printf("         size:   %d bytes (%.2f MB)\n",
				layer.Size, float64(layer.Size)/1024/1024)
		}

		// Step 6: Display build history.
		fmt.Printf("\n  Build History:\n")
		for j, h := range imgConfig.History {
			empty := ""
			if h.EmptyLayer {
				empty = " (empty layer)"
			}
			fmt.Printf("    [%d] %s%s\n", j, h.CreatedBy, empty)
		}
		fmt.Println()
	}

	// Step 7: Enumerate all blobs in the content store and report
	// total storage size.
	blobDir := filepath.Join(layoutDir, "blobs", "sha256")
	entries, err := os.ReadDir(blobDir)
	if err == nil {
		var totalSize int64
		for _, e := range entries {
			info, _ := e.Info()
			if info != nil {
				totalSize += info.Size()
			}
		}
		fmt.Printf("Content Store: %d blobs, %.2f MB total\n",
			len(entries), float64(totalSize)/1024/1024)
	}

	// Suppress unused import warning for io package (used in full version).
	_ = io.Discard
}
```

Running this program on an Alpine OCI layout produces:

```
$ go run ociinspect.go alpine-oci/
OCI Layout Version: 1.0.0

Index contains 1 manifest(s)

--- Manifest 0 ---
  MediaType: application/vnd.oci.image.manifest.v1+json
  Digest:    sha256:c9f0e3d2...
  Size:      482 bytes
  ✓ Digest verified

  Config: sha256:b7d8f2e1... (1472 bytes)
  Architecture: amd64
  OS:           linux
  Cmd:          [/bin/sh]
  Env:          [PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin]
  RootFS type:  layers
  Layers (diff_ids): 1
    [0] sha256:4abcdef12345...

  Layers (compressed, from manifest): 1
    [0] application/vnd.oci.image.layer.v1.tar+gzip
         digest: sha256:a3ed95caeb02...
         size:   3401613 bytes (3.24 MB)

  Build History:
    [0] /bin/sh -c #(nop) ADD file:abc123 in /

Content Store: 3 blobs, 3.25 MB total
```

### 9.1.4 Hands-on: Pulling and Unpacking an Image Manually with Go

Let's go deeper and pull an image from a registry, then unpack its layers into a
root filesystem — all without Docker or containerd:

```go
// Package imagepull demonstrates how to pull an OCI/Docker image from
// a container registry using only the HTTP Distribution API. It fetches
// the manifest, downloads each layer blob, and unpacks them into a
// local root filesystem directory. This is the "from scratch" approach
// to understanding what `docker pull` does under the hood.
package main

import (
	"archive/tar"
	"compress/gzip"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"path/filepath"
	"strings"
)

// registryManifest represents a simplified Docker/OCI image manifest
// as returned by a v2 registry. We use Docker media types here for
// compatibility with Docker Hub, but the structure is very similar
// to the OCI manifest format.
type registryManifest struct {
	SchemaVersion int             `json:"schemaVersion"`
	MediaType     string          `json:"mediaType"`
	Config        layerDescriptor `json:"config"`
	Layers        []layerDescriptor `json:"layers"`
}

// layerDescriptor identifies a single blob (config or layer) in the
// registry by its media type, content-addressable digest, and byte
// size. The client uses the digest to construct the blob download URL.
type layerDescriptor struct {
	MediaType string `json:"mediaType"`
	Digest    string `json:"digest"`
	Size      int64  `json:"size"`
}

// getAuthToken obtains a short-lived Bearer token from the Docker Hub
// authentication service. Public images require a token even for
// anonymous pulls; the token is scoped to pull access for a specific
// repository.
func getAuthToken(repo string) (string, error) {
	// Docker Hub's token endpoint. For other registries, the client
	// would parse the Www-Authenticate header from a 401 response
	// to discover the token endpoint dynamically.
	url := fmt.Sprintf(
		"https://auth.docker.io/token?service=registry.docker.io&scope=repository:%s:pull",
		repo,
	)
	resp, err := http.Get(url)
	if err != nil {
		return "", fmt.Errorf("auth request failed: %w", err)
	}
	defer resp.Body.Close()

	var result struct {
		Token string `json:"token"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
		return "", fmt.Errorf("parsing auth response: %w", err)
	}
	return result.Token, nil
}

// fetchManifest retrieves the image manifest for the given repository
// and tag from the Docker Hub registry. It sends the appropriate
// Accept header to request a Docker v2 manifest (schema 2), which
// is wire-compatible with OCI manifests for our purposes.
func fetchManifest(repo, tag, token string) (*registryManifest, error) {
	url := fmt.Sprintf("https://registry-1.docker.io/v2/%s/manifests/%s", repo, tag)

	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		return nil, err
	}
	// Request a Docker v2 manifest. We could also accept the OCI
	// media type, but Docker Hub serves Docker-format manifests
	// for most official images.
	req.Header.Set("Accept", "application/vnd.docker.distribution.manifest.v2+json")
	req.Header.Set("Authorization", "Bearer "+token)

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, fmt.Errorf("manifest request failed: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		body, _ := io.ReadAll(resp.Body)
		return nil, fmt.Errorf("manifest fetch returned %d: %s", resp.StatusCode, body)
	}

	var manifest registryManifest
	if err := json.NewDecoder(resp.Body).Decode(&manifest); err != nil {
		return nil, fmt.Errorf("parsing manifest: %w", err)
	}
	return &manifest, nil
}

// downloadBlob streams a layer blob from the registry to a local file.
// Blobs are downloaded by their content-addressable digest, which the
// registry uses to locate the blob in its storage backend. The blob
// may be a compressed tar archive (for layers) or a JSON document
// (for configs).
func downloadBlob(repo, digest, token, destPath string) error {
	url := fmt.Sprintf("https://registry-1.docker.io/v2/%s/blobs/%s", repo, digest)

	req, err := http.NewRequest("GET", url, nil)
	if err != nil {
		return err
	}
	req.Header.Set("Authorization", "Bearer "+token)

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return fmt.Errorf("blob download failed: %w", err)
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("blob download returned %d", resp.StatusCode)
	}

	// Create the destination file and stream the blob content to disk.
	// In production, you would also verify the digest after download.
	f, err := os.Create(destPath)
	if err != nil {
		return err
	}
	defer f.Close()

	written, err := io.Copy(f, resp.Body)
	if err != nil {
		return fmt.Errorf("writing blob: %w", err)
	}
	fmt.Printf("  Downloaded %s (%.2f MB)\n", digest[:24], float64(written)/1024/1024)
	return nil
}

// unpackLayer extracts a gzip-compressed tar layer into the target
// rootfs directory. This is the fundamental operation that builds up
// the container's filesystem: each layer is unpacked in order, with
// later layers overwriting or deleting files from earlier layers.
//
// OCI whiteout files (.wh.*) signal file deletions across layers.
// We handle them here for correctness.
func unpackLayer(layerPath, rootfs string) error {
	f, err := os.Open(layerPath)
	if err != nil {
		return err
	}
	defer f.Close()

	// Layers are typically gzip-compressed. Wrap the file reader
	// in a gzip decompressor before feeding it to the tar reader.
	gz, err := gzip.NewReader(f)
	if err != nil {
		return fmt.Errorf("gzip open: %w", err)
	}
	defer gz.Close()

	tr := tar.NewReader(gz)
	for {
		hdr, err := tr.Next()
		if err == io.EOF {
			break
		}
		if err != nil {
			return fmt.Errorf("tar read: %w", err)
		}

		// Construct the full path within the rootfs for this entry.
		target := filepath.Join(rootfs, hdr.Name)

		// Handle OCI whiteout files: a file named .wh.<name> means
		// we should delete <name> from the filesystem.
		base := filepath.Base(hdr.Name)
		if strings.HasPrefix(base, ".wh.") {
			// This is a whiteout marker. Remove the corresponding
			// file or directory from the rootfs.
			deleteName := strings.TrimPrefix(base, ".wh.")
			deletePath := filepath.Join(filepath.Dir(target), deleteName)
			os.RemoveAll(deletePath)
			continue
		}

		// Create the filesystem entry based on the tar header type.
		switch hdr.Typeflag {
		case tar.TypeDir:
			// Create directory with the permissions from the tar header.
			if err := os.MkdirAll(target, os.FileMode(hdr.Mode)); err != nil {
				return err
			}
		case tar.TypeReg:
			// Ensure the parent directory exists before creating the file.
			if err := os.MkdirAll(filepath.Dir(target), 0755); err != nil {
				return err
			}
			outFile, err := os.OpenFile(target, os.O_CREATE|os.O_WRONLY|os.O_TRUNC, os.FileMode(hdr.Mode))
			if err != nil {
				return err
			}
			if _, err := io.Copy(outFile, tr); err != nil {
				outFile.Close()
				return err
			}
			outFile.Close()
		case tar.TypeSymlink:
			// Create symbolic links as specified in the layer.
			os.Remove(target)
			if err := os.Symlink(hdr.Linkname, target); err != nil {
				return err
			}
		case tar.TypeLink:
			// Hard links reference another file in the same layer.
			linkTarget := filepath.Join(rootfs, hdr.Linkname)
			os.Remove(target)
			if err := os.Link(linkTarget, target); err != nil {
				return err
			}
		}
	}
	return nil
}

func main() {
	// Pull the Alpine Linux image (small and fast for demonstration).
	repo := "library/alpine"
	tag := "3.19"
	outputDir := "pulled-image"

	fmt.Printf("Pulling %s:%s from Docker Hub...\n\n", repo, tag)

	// Step 1: Authenticate with the registry.
	token, err := getAuthToken(repo)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Authentication failed: %v\n", err)
		os.Exit(1)
	}
	fmt.Println("✓ Authenticated with Docker Hub")

	// Step 2: Fetch the image manifest.
	manifest, err := fetchManifest(repo, tag, token)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Manifest fetch failed: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("✓ Got manifest: %d layers\n\n", len(manifest.Layers))

	// Step 3: Create directories for blobs and rootfs.
	blobDir := filepath.Join(outputDir, "blobs")
	rootfsDir := filepath.Join(outputDir, "rootfs")
	os.MkdirAll(blobDir, 0755)
	os.MkdirAll(rootfsDir, 0755)

	// Step 4: Download and unpack each layer in order.
	for i, layer := range manifest.Layers {
		fmt.Printf("Layer %d/%d:\n", i+1, len(manifest.Layers))

		// Use the digest hex as the filename for the downloaded blob.
		blobFile := filepath.Join(blobDir, strings.Replace(layer.Digest, ":", "-", 1))
		if err := downloadBlob(repo, layer.Digest, token, blobFile); err != nil {
			fmt.Fprintf(os.Stderr, "  Download failed: %v\n", err)
			os.Exit(1)
		}

		// Unpack the layer into the rootfs directory. Layers are
		// applied in order: the first layer forms the base, and
		// subsequent layers overlay their changes.
		if err := unpackLayer(blobFile, rootfsDir); err != nil {
			fmt.Fprintf(os.Stderr, "  Unpack failed: %v\n", err)
			os.Exit(1)
		}
		fmt.Printf("  ✓ Unpacked into rootfs\n\n")
	}

	// Step 5: Download the image config for reference.
	configFile := filepath.Join(outputDir, "config.json")
	if err := downloadBlob(repo, manifest.Config.Digest, token, configFile); err != nil {
		fmt.Fprintf(os.Stderr, "Config download failed: %v\n", err)
	}

	fmt.Println("Done! Root filesystem is at:", rootfsDir)
	fmt.Println("You can explore it with: ls", rootfsDir)
	fmt.Println("Or chroot into it with:  sudo chroot", rootfsDir, "/bin/sh")
}
```

---

## 9.2 OCI Runtime Specification

The OCI Runtime Specification defines how to **run** a container. While the Image
Spec tells us how to store and distribute filesystem images, the Runtime Spec tells
us how to configure and execute a process inside an isolated environment.

### 9.2.1 config.json — The Runtime Bundle

An **OCI runtime bundle** is a directory containing:
1. A `config.json` file describing the container configuration.
2. A root filesystem directory (typically called `rootfs`).

The runtime (e.g., runc) consumes this bundle to create and start a container.

Here's a complete `config.json` annotated with explanations:

```json
{
    "ociVersion": "1.0.2",
    "process": {
        "terminal": true,
        "user": {
            "uid": 0,
            "gid": 0
        },
        "args": ["/bin/sh"],
        "env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "TERM=xterm"
        ],
        "cwd": "/",
        "capabilities": {
            "bounding": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"],
            "effective": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"],
            "permitted": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"],
            "ambient": ["CAP_AUDIT_WRITE", "CAP_KILL", "CAP_NET_BIND_SERVICE"]
        },
        "rlimits": [
            {
                "type": "RLIMIT_NOFILE",
                "hard": 1024,
                "soft": 1024
            }
        ],
        "noNewPrivileges": true
    },
    "root": {
        "path": "rootfs",
        "readonly": false
    },
    "hostname": "my-container",
    "mounts": [
        {
            "destination": "/proc",
            "type": "proc",
            "source": "proc"
        },
        {
            "destination": "/dev",
            "type": "tmpfs",
            "source": "tmpfs",
            "options": ["nosuid", "strictatime", "mode=755", "size=65536k"]
        },
        {
            "destination": "/dev/pts",
            "type": "devpts",
            "source": "devpts",
            "options": ["nosuid", "noexec", "newinstance", "ptmxmode=0666", "mode=0620"]
        },
        {
            "destination": "/dev/shm",
            "type": "tmpfs",
            "source": "shm",
            "options": ["nosuid", "noexec", "nodev", "mode=1777", "size=65536k"]
        },
        {
            "destination": "/dev/mqueue",
            "type": "mqueue",
            "source": "mqueue",
            "options": ["nosuid", "noexec", "nodev"]
        },
        {
            "destination": "/sys",
            "type": "sysfs",
            "source": "sysfs",
            "options": ["nosuid", "noexec", "nodev", "ro"]
        }
    ],
    "linux": {
        "namespaces": [
            {"type": "pid"},
            {"type": "network"},
            {"type": "ipc"},
            {"type": "uts"},
            {"type": "mount"},
            {"type": "cgroup"}
        ],
        "resources": {
            "memory": {
                "limit": 536870912
            },
            "cpu": {
                "shares": 1024,
                "quota": 100000,
                "period": 100000
            },
            "pids": {
                "limit": 512
            }
        },
        "seccomp": {
            "defaultAction": "SCMP_ACT_ERRNO",
            "architectures": ["SCMP_ARCH_X86_64"],
            "syscalls": [
                {
                    "names": [
                        "accept", "bind", "clone", "close", "connect",
                        "execve", "exit", "exit_group", "fcntl", "fork",
                        "getcwd", "getpid", "ioctl", "listen", "mmap",
                        "mprotect", "munmap", "open", "openat", "read",
                        "recvmsg", "sendmsg", "socket", "stat", "write"
                    ],
                    "action": "SCMP_ACT_ALLOW"
                }
            ]
        },
        "maskedPaths": [
            "/proc/acpi",
            "/proc/kcore",
            "/proc/keys",
            "/proc/latency_stats",
            "/proc/timer_list",
            "/proc/timer_stats",
            "/proc/sched_debug",
            "/sys/firmware"
        ],
        "readonlyPaths": [
            "/proc/asound",
            "/proc/bus",
            "/proc/fs",
            "/proc/irq",
            "/proc/sys",
            "/proc/sysrq-trigger"
        ]
    }
}
```

Let's break down the major sections:

**Process**: Defines what runs inside the container:
- `args`: The command to execute (equivalent to Docker's `CMD`/`ENTRYPOINT`).
- `env`: Environment variables.
- `capabilities`: Linux capabilities granted to the process. The four sets
  (bounding, effective, permitted, ambient) control privilege escalation.
- `noNewPrivileges`: Prevents `execve()` from gaining privileges via setuid bits.
- `rlimits`: Resource limits (file descriptors, process count, etc.).

**Root**: The container's root filesystem. `readonly: true` would make it immutable.

**Mounts**: Filesystems mounted inside the container. The standard set includes:
- `/proc` — process information pseudo-filesystem.
- `/dev` — device nodes (tmpfs so the container gets its own).
- `/dev/pts` — pseudo-terminal devices.
- `/sys` — sysfs (read-only to prevent kernel parameter modification).

**Linux-specific**:
- `namespaces`: Which Linux namespaces to create (PID, network, IPC, UTS, mount, cgroup).
- `resources`: Cgroup limits for memory, CPU, and PIDs.
- `seccomp`: System call filter. Default-deny with an allowlist of safe syscalls.
- `maskedPaths`: Paths bind-mounted to `/dev/null` to hide sensitive kernel info.
- `readonlyPaths`: Paths made read-only inside the container.

### 9.2.2 Container Lifecycle

The OCI Runtime Spec defines a strict lifecycle for containers:

```
                    ┌──────────────────────────────┐
                    │        OCI Container          │
                    │         Lifecycle              │
                    └──────────────────────────────┘

    ┌─────────┐    create    ┌─────────┐    start    ┌─────────┐
    │         │─────────────►│         │────────────►│         │
    │  (none) │              │ Created │             │ Running │
    │         │              │         │             │         │
    └─────────┘              └─────────┘             └────┬────┘
                                                          │
                                                    exit/signal
                                                          │
                                                     ┌────▼────┐
                                                     │         │
                                                     │ Stopped │
                                                     │         │
                                                     └────┬────┘
                                                          │
                                                       delete
                                                          │
                                                     ┌────▼────┐
                                                     │         │
                                                     │ (none)  │
                                                     │         │
                                                     └─────────┘
```

**create**: Sets up the container environment (namespaces, cgroups, mounts, rootfs)
but does NOT start the user process. The container init process is paused.

**start**: Signals the init process to execute the user-specified `args`. The
container transitions to `running`.

**kill**: Sends a signal (default: SIGTERM) to the container's init process.

**delete**: Removes the container's resources (cgroups, state files). Only valid
when the container is stopped.

This two-phase create/start design allows pre-start hooks to run between creation
and execution — useful for setting up networking (e.g., creating veth pairs).

### 9.2.3 Container State

A runtime must be able to report container state:

```json
{
    "ociVersion": "1.0.2",
    "id": "my-container-01",
    "status": "running",
    "pid": 12345,
    "bundle": "/containers/my-container-01",
    "annotations": {
        "org.example.owner": "ops-team"
    }
}
```

### 9.2.4 Hands-on: Creating an OCI Runtime Bundle Manually

```bash
#!/bin/bash
# create-bundle.sh — Creates a minimal OCI runtime bundle from an
# Alpine rootfs. This script demonstrates the manual steps that
# container runtimes automate: creating a rootfs directory and a
# config.json that conforms to the OCI Runtime Spec.

set -euo pipefail

BUNDLE_DIR="./my-bundle"
ROOTFS="$BUNDLE_DIR/rootfs"

echo "=== Creating OCI Runtime Bundle ==="

# Step 1: Create the bundle directory structure.
# An OCI bundle is just a directory with rootfs/ and config.json.
mkdir -p "$ROOTFS"

# Step 2: Populate the rootfs from the Alpine Docker image.
# We use Docker to export a filesystem, but you could use any method
# to populate rootfs (debootstrap, downloading a tarball, etc.).
docker export $(docker create alpine:3.19) | tar -C "$ROOTFS" -xf -

# Step 3: Generate a default config.json using runc.
# runc spec produces a valid OCI config.json with sensible defaults.
cd "$BUNDLE_DIR"
runc spec
cd -

echo "Bundle created at $BUNDLE_DIR"
echo "Contents:"
echo "  $BUNDLE_DIR/config.json  (OCI runtime config)"
echo "  $BUNDLE_DIR/rootfs/      (container root filesystem)"
echo ""
echo "Inspect the config:"
echo "  cat $BUNDLE_DIR/config.json | python3 -m json.tool"
```

### 9.2.5 Hands-on: Running a Container with runc Directly

```bash
#!/bin/bash
# run-with-runc.sh — Demonstrates running a container directly with
# runc, bypassing Docker and containerd entirely. This shows the
# lowest-level interface to OCI container execution.

set -euo pipefail

BUNDLE_DIR="./my-bundle"

echo "=== Running container with runc ==="

# Create the container. This sets up namespaces, cgroups, and mounts
# but doesn't start the user process yet.
sudo runc create --bundle "$BUNDLE_DIR" my-container-01

# Check the state — should be "created"
echo "State after create:"
sudo runc state my-container-01

# Start the container — this triggers the init process to exec the
# command specified in config.json (default: /bin/sh).
sudo runc start my-container-01

# List running containers managed by this runc instance.
echo "Running containers:"
sudo runc list

# Execute an additional process inside the running container.
# This is equivalent to `docker exec`.
sudo runc exec my-container-01 /bin/ls /

# Send SIGTERM to gracefully stop the container.
sudo runc kill my-container-01 SIGTERM

# Wait a moment, then delete the container and clean up resources.
sleep 2
sudo runc delete my-container-01

echo "Container deleted."
```

---

## 9.3 OCI Distribution Specification

The OCI Distribution Spec standardizes how container images are pushed to and pulled
from registries. It defines an HTTP API that all compliant registries must implement.

### 9.3.1 Registry API Overview

```
    Registry API Endpoints
    ─────────────────────────────────────────────────────────

    GET  /v2/                                    # API version check
    GET  /v2/<name>/manifests/<reference>         # Pull manifest
    PUT  /v2/<name>/manifests/<reference>         # Push manifest
    GET  /v2/<name>/blobs/<digest>                # Pull blob
    HEAD /v2/<name>/blobs/<digest>                # Check blob exists
    POST /v2/<name>/blobs/uploads/                # Initiate blob upload
    PATCH /v2/<name>/blobs/uploads/<uuid>         # Upload blob chunk
    PUT  /v2/<name>/blobs/uploads/<uuid>?digest=  # Complete upload
    GET  /v2/<name>/tags/list                     # List tags
    GET  /v2/_catalog                             # List repositories
```

**Content Negotiation**: The client sends an `Accept` header to indicate which
manifest formats it supports:

```
Accept: application/vnd.oci.image.manifest.v1+json,
        application/vnd.oci.image.index.v1+json,
        application/vnd.docker.distribution.manifest.v2+json,
        application/vnd.docker.distribution.manifest.list.v2+json
```

The registry responds with the best matching format. This allows registries to
serve both OCI and Docker-format manifests from the same storage.

### 9.3.2 Hands-on: Pulling an Image Using the HTTP API in Go

```go
// Package registryclient demonstrates direct interaction with an OCI
// Distribution-compliant container registry using raw HTTP requests.
// This program implements the pull workflow: authenticate, fetch the
// manifest, then download each blob (layer). It shows exactly what
// happens at the network level when you run "docker pull".
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strings"
)

// tokenResponse represents the JSON response from a Docker Hub (or
	// compatible) token authentication endpoint. The token field contains
// a short-lived JWT used to authorize subsequent registry API calls.
type tokenResponse struct {
	Token string `json:"token"`
}

// manifestResponse represents the essential fields of an OCI/Docker
// image manifest. We only decode the fields we need for the pull
// operation: the config descriptor and the layer descriptors.
type manifestResponse struct {
	SchemaVersion int              `json:"schemaVersion"`
	MediaType     string           `json:"mediaType"`
	Config        blobDescriptor   `json:"config"`
	Layers        []blobDescriptor `json:"layers"`
}

// blobDescriptor identifies a content-addressable blob in the registry.
// The digest serves as both an address (to fetch the blob) and a
// checksum (to verify the blob after download).
type blobDescriptor struct {
	MediaType string `json:"mediaType"`
	Digest    string `json:"digest"`
	Size      int64  `json:"size"`
}

// tagsResponse represents the response from the /v2/<name>/tags/list
// endpoint. It lists all available tags for a given repository.
type tagsResponse struct {
	Name string   `json:"name"`
	Tags []string `json:"tags"`
}

// registryClient encapsulates the HTTP client and authentication state
// needed to interact with a container registry. It handles token-based
// authentication transparently.
type registryClient struct {
	// httpClient is the underlying HTTP client used for all requests.
	httpClient *http.Client
	// registryURL is the base URL of the registry (e.g.,
		// "https://registry-1.docker.io").
	registryURL string
	// token is the Bearer token obtained during authentication.
	// It is included in the Authorization header of all API requests.
	token string
}

// newRegistryClient creates a new registryClient configured for Docker
// Hub. It obtains an authentication token scoped to pull access for
// the specified repository.
func newRegistryClient(repo string) (*registryClient, error) {
	client := &registryClient{
		httpClient:  &http.Client{},
		registryURL: "https://registry-1.docker.io",
	}

	// Obtain a Bearer token from Docker Hub's auth service.
	// Even anonymous pulls require a token.
	authURL := fmt.Sprintf(
		"https://auth.docker.io/token?service=registry.docker.io&scope=repository:%s:pull",
		repo,
	)
	resp, err := client.httpClient.Get(authURL)
	if err != nil {
		return nil, fmt.Errorf("auth failed: %w", err)
	}
	defer resp.Body.Close()

	var tok tokenResponse
	if err := json.NewDecoder(resp.Body).Decode(&tok); err != nil {
		return nil, fmt.Errorf("parsing token: %w", err)
	}
	client.token = tok.Token
	return client, nil
}

// doRequest performs an authenticated HTTP request to the registry.
// It adds the Bearer token and any specified Accept headers. This
// centralizes the authentication logic so callers don't need to
// manage tokens directly.
func (c *registryClient) doRequest(method, path string, acceptTypes []string) (*http.Response, error) {
	url := c.registryURL + path
	req, err := http.NewRequest(method, url, nil)
	if err != nil {
		return nil, err
	}
	req.Header.Set("Authorization", "Bearer "+c.token)

	// Set Accept headers for content negotiation. The registry uses
	// these to determine which manifest format to return.
	for _, at := range acceptTypes {
		req.Header.Add("Accept", at)
	}

	return c.httpClient.Do(req)
}

// getManifest fetches and parses the image manifest for the given
// repository and reference (tag or digest). It requests the Docker v2
// manifest format for broad compatibility.
func (c *registryClient) getManifest(repo, ref string) (*manifestResponse, error) {
	path := fmt.Sprintf("/v2/%s/manifests/%s", repo, ref)
	resp, err := c.doRequest("GET", path, []string{
			"application/vnd.docker.distribution.manifest.v2+json",
			"application/vnd.oci.image.manifest.v1+json",
		})
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		body, _ := io.ReadAll(resp.Body)
		return nil, fmt.Errorf("manifest GET %d: %s", resp.StatusCode, body)
	}

	// Log the actual content type returned by the registry so we can
	// see which format was negotiated.
	fmt.Printf("  Content-Type: %s\n", resp.Header.Get("Content-Type"))
	fmt.Printf("  Docker-Content-Digest: %s\n", resp.Header.Get("Docker-Content-Digest"))

	var m manifestResponse
	if err := json.NewDecoder(resp.Body).Decode(&m); err != nil {
		return nil, err
	}
	return &m, nil
}

// listTags retrieves all tags for a repository. This demonstrates the
// /v2/<name>/tags/list endpoint defined by the Distribution Spec.
func (c *registryClient) listTags(repo string) (*tagsResponse, error) {
	path := fmt.Sprintf("/v2/%s/tags/list", repo)
	resp, err := c.doRequest("GET", path, nil)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	var tags tagsResponse
	if err := json.NewDecoder(resp.Body).Decode(&tags); err != nil {
		return nil, err
	}
	return &tags, nil
}

// headBlob checks whether a blob exists in the registry without
// downloading it. This is useful for checking if a layer is already
// present before uploading (content-addressable deduplication).
func (c *registryClient) headBlob(repo, digest string) (int64, error) {
	path := fmt.Sprintf("/v2/%s/blobs/%s", repo, digest)
	resp, err := c.doRequest("HEAD", path, nil)
	if err != nil {
		return 0, err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return 0, fmt.Errorf("blob HEAD returned %d", resp.StatusCode)
	}
	return resp.ContentLength, nil
}

// downloadBlob downloads a blob from the registry and writes it to the
// specified destination path. This is used to fetch both config blobs
// and layer blobs during the pull process.
func (c *registryClient) downloadBlob(repo, digest, destPath string) error {
	path := fmt.Sprintf("/v2/%s/blobs/%s", repo, digest)
	resp, err := c.doRequest("GET", path, nil)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("blob GET returned %d", resp.StatusCode)
	}

	f, err := os.Create(destPath)
	if err != nil {
		return err
	}
	defer f.Close()

	n, err := io.Copy(f, resp.Body)
	if err != nil {
		return err
	}
	fmt.Printf("  Wrote %d bytes to %s\n", n, destPath)
	return nil
}

func main() {
	repo := "library/alpine"
	tag := "3.19"

	fmt.Printf("=== OCI Distribution API Demo ===\n")
	fmt.Printf("Repository: %s\n", repo)
	fmt.Printf("Tag: %s\n\n", tag)

	// Step 1: Create an authenticated registry client.
	client, err := newRegistryClient(repo)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to create client: %v\n", err)
		os.Exit(1)
	}
	fmt.Println("✓ Authenticated\n")

	// Step 2: List available tags.
	tags, err := client.listTags(repo)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to list tags: %v\n", err)
	} else {
		fmt.Printf("Available tags (%d total): %s...\n\n",
			len(tags.Tags),
			strings.Join(tags.Tags[:min(5, len(tags.Tags))], ", "))
	}

	// Step 3: Fetch the manifest.
	fmt.Println("Fetching manifest:")
	manifest, err := client.getManifest(repo, tag)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to get manifest: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("  Schema: v%d\n", manifest.SchemaVersion)
	fmt.Printf("  Config: %s (%d bytes)\n", manifest.Config.Digest[:24], manifest.Config.Size)
	fmt.Printf("  Layers: %d\n\n", len(manifest.Layers))

	// Step 4: Check each layer's existence and size.
	for i, layer := range manifest.Layers {
		size, err := client.headBlob(repo, layer.Digest)
		if err != nil {
			fmt.Printf("  Layer %d: HEAD failed: %v\n", i, err)
			continue
		}
		fmt.Printf("  Layer %d: %s (%.2f MB, type: %s)\n",
			i, layer.Digest[:24], float64(size)/1024/1024, layer.MediaType)
	}

	fmt.Println("\n✓ All layers verified in registry")
}

---

## 9.4 runc — The Reference OCI Runtime

### 9.4.1 Architecture and Code Structure

**runc** is the reference implementation of the OCI Runtime Specification. Originally
extracted from Docker's `libcontainer`, runc is a standalone CLI tool that creates
and runs containers given an OCI bundle.

```text
    runc Architecture
    ─────────────────────────────────────────────

    runc CLI
    ├── create     → sets up container (namespaces, cgroups, mounts)
    ├── start      → signals init process to exec user command
    ├── exec       → runs additional process in existing container
    ├── kill       → sends signal to container process
    ├── delete     → tears down container resources
    ├── list       → lists containers
    ├── state      → reports container state
    └── spec       → generates default config.json

    Internal Flow (runc create):
    ┌──────────┐     fork      ┌──────────────┐
    │  runc    │──────────────►│  runc init   │
    │  parent  │               │  (child)     │
    │          │  socketpair   │              │
    │          │◄─────────────►│  - unshare() │
    │          │               │  - pivot_root│
    │          │               │  - mount     │
    │          │   "ready"     │  - setns     │
    │          │◄──────────────│  - exec      │
    │          │               │              │
    │  setup   │   "start"    │  → user      │
    │  cgroups │──────────────►│    process   │
    └──────────┘               └──────────────┘
```

runc uses a **double-fork** strategy for container creation:

1. **Parent runc process**: Parses config.json, sets up the container state
   directory, creates cgroups, and prepares the environment.

2. **runc init process**: A child process that enters the new namespaces using
   `clone()` or `unshare()`. It performs the container setup *from inside* the
   namespaces: mounting filesystems, calling `pivot_root()`, configuring network
   interfaces, setting capabilities, and applying seccomp filters. Once ready,
   it blocks waiting for the parent's "start" signal.

3. **User process**: After receiving the start signal, the init process calls
   `execve()` to replace itself with the user's command.

The parent and init processes communicate via a Unix socket pair. This allows the
parent to relay configuration and control messages while the init process reports
its PID and readiness.

### 9.4.2 How runc Creates Containers

Let's trace the creation of a container step by step:

```text
    Container Creation Sequence
    ─────────────────────────────────────────────

    1. runc reads config.json
    2. Creates socketpair for parent ↔ init communication
    3. Forks child process
    4. Child calls clone() with CLONE_NEW{PID,NET,MNT,UTS,IPC,USER}
       (or unshare() + fork)
    5. Child (now in new namespaces):
       a. Sets hostname (UTS namespace)
       b. Mounts rootfs, proc, dev, sys
       c. Calls pivot_root() to change root
       d. Sets up /dev devices (null, zero, random, etc.)
       e. Applies seccomp filter
       f. Drops capabilities
       g. Signals parent: "I'm ready"
       h. Waits for parent's "start" signal
    6. Parent:
       a. Receives child PID
       b. Moves child into cgroup
       c. Runs pre-start hooks (e.g., network setup)
       d. Writes container state file
       e. Signals child: "start"
    7. Child calls execve(args[0], args, env)
       → Now running the user's process
```

### 9.4.3 runc vs crun vs youki — Alternative Runtimes

While runc is the reference implementation, several alternative OCI runtimes exist:

| Runtime | Language | Key Differentiator |
|---------|----------|--------------------|
| **runc** | Go | Reference implementation, most widely used |
| **crun** | C | Faster startup (~50%), lower memory, systemd-native |
| **youki** | Rust | Memory-safe, modern, growing community |
| **kata-runtime** | Go | Runs containers in lightweight VMs |
| **runsc** (gVisor) | Go | User-space kernel for syscall interception |

**crun** is increasingly popular in production because it starts containers faster
(C is cheaper to initialize than Go's runtime) and integrates natively with systemd
for cgroup management.

**youki** is a Rust rewrite that aims to combine runc's correctness with Rust's
memory safety guarantees. It's fully OCI-compliant and passes the runtime-tools
test suite.

### 9.4.4 Hands-on: Tracing runc Execution with strace

```bash
#!/bin/bash
# trace-runc.sh — Uses strace to observe the system calls runc makes
# when creating a container. This reveals the namespace, cgroup, and
# mount operations that implement container isolation.

set -euo pipefail

BUNDLE_DIR="./my-bundle"

# Trace runc with strace, following child processes (-f), filtering
# for the system calls most relevant to container creation.
echo "=== Tracing runc create ==="
sudo strace -f \
    -e trace=clone,clone3,unshare,mount,pivot_root,execve,setns,\
seccomp,prctl,sethostname,setgroups,setuid,setgid \
    -o runc-trace.log \
    runc run --bundle "$BUNDLE_DIR" traced-container &

# Give it a moment to start, then inspect the trace
sleep 3

echo "=== Key system calls from the trace ==="

# Show clone/unshare calls that create namespaces
echo ""
echo "--- Namespace creation ---"
grep -E "clone|unshare" runc-trace.log | head -20

# Show pivot_root call that changes the root filesystem
echo ""
echo "--- Root filesystem change ---"
grep "pivot_root" runc-trace.log

# Show mount calls for /proc, /dev, /sys
echo ""
echo "--- Mount operations ---"
grep "mount" runc-trace.log | head -30

# Show seccomp filter installation
echo ""
echo "--- Seccomp ---"
grep "seccomp" runc-trace.log

# Show the final execve that starts the user process
echo ""
echo "--- execve (user process) ---"
grep "execve" runc-trace.log | tail -5

# Clean up
sudo runc kill traced-container SIGKILL 2>/dev/null || true
sudo runc delete traced-container 2>/dev/null || true
```

Expected output shows the sequence of system calls:

```
--- Namespace creation ---
clone3({flags=CLONE_NEWNS|CLONE_NEWUTS|CLONE_NEWIPC|CLONE_NEWPID|CLONE_NEWNET, ...})

--- Root filesystem change ---
pivot_root(".", ".")

--- Mount operations ---
mount("proc", "/proc", "proc", MS_NOSUID|MS_NOEXEC|MS_NODEV, NULL)
mount("tmpfs", "/dev", "tmpfs", MS_NOSUID|MS_STRICTATIME, "mode=755,size=65536k")
mount("sysfs", "/sys", "sysfs", MS_NOSUID|MS_NOEXEC|MS_NODEV|MS_RDONLY, NULL)

--- Seccomp ---
seccomp(SECCOMP_SET_MODE_FILTER, 0, {len=123, filter=0x...})

--- execve (user process) ---
execve("/bin/sh", ["/bin/sh"], ["PATH=/usr/local/sbin:...", "TERM=xterm"])
```

---

## 9.5 containerd — The Container Daemon

### 9.5.1 Architecture Overview

**containerd** is an industry-standard container runtime daemon. It manages the
complete container lifecycle: image pull, storage, container execution, and
supervision. It is the runtime used by Docker, Kubernetes (via CRI), and many
cloud providers.

containerd is designed as a **plugin-based, gRPC-driven daemon**:

```
    containerd Architecture
    ══════════════════════════════════════════════════════════════════

    ┌─────────────────────────────────────────────────────────────┐
    │                        Clients                              │
    │  Docker Engine │ ctr │ crictl │ nerdctl │ Custom Go Client  │
    └───────┬─────────┬──────┬────────┬──────────┬────────────────┘
            │         │      │        │          │
            ▼         ▼      ▼        ▼          ▼
    ┌─────────────────────────────────────────────────────────────┐
    │                     gRPC API (ttrpc)                         │
    │  containerd.services.images.v1                              │
    │  containerd.services.containers.v1                          │
    │  containerd.services.tasks.v1                               │
    │  containerd.services.content.v1                             │
    │  containerd.services.snapshots.v1                           │
    │  containerd.services.namespaces.v1                          │
    │  containerd.services.leases.v1                              │
    │  runtime.v1.CRI (Kubernetes CRI plugin)                    │
    └───────┬─────────┬──────┬────────┬──────────┬────────────────┘
            │         │      │        │          │
    ┌───────▼─────────▼──────▼────────▼──────────▼────────────────┐
    │                    Core Services                             │
    │                                                              │
    │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────┐  │
    │  │  Image   │ │Container │ │  Task    │ │   Content     │  │
    │  │  Service │ │ Service  │ │  Service │ │   Store       │  │
    │  │          │ │          │ │          │ │               │  │
    │  │ pull     │ │ create   │ │ create   │ │ ingest blobs  │  │
    │  │ push     │ │ get      │ │ start    │ │ fetch blobs   │  │
    │  │ list     │ │ update   │ │ kill     │ │ GC blobs      │  │
    │  │ delete   │ │ delete   │ │ delete   │ │               │  │
    │  └──────────┘ └──────────┘ └────┬─────┘ └───────────────┘  │
    │                                  │                           │
    │  ┌──────────┐ ┌──────────┐      │       ┌───────────────┐  │
    │  │Snapshot  │ │  Lease   │      │       │  Namespace    │  │
    │  │ Service  │ │  Service │      │       │  Service      │  │
    │  │          │ │          │      │       │               │  │
    │  │ overlayfs│ │ prevents │      │       │ multi-tenant  │  │
    │  │ native   │ │ GC of    │      │       │ isolation     │  │
    │  │ devmapper│ │ in-use   │      │       │               │  │
    │  │ stargz   │ │ content  │      │       │               │  │
    │  └──────────┘ └──────────┘      │       └───────────────┘  │
    │                                  │                           │
    └──────────────────────────────────┼───────────────────────────┘
                                       │
                              ┌────────▼────────┐
                              │   Shim v2 API   │
                              │                 │
                              │  containerd-    │
                              │  shim-runc-v2   │
                              └────────┬────────┘
                                       │
                              ┌────────▼────────┐
                              │    runc/crun    │
                              │                 │
                              │  OCI Runtime    │
                              └────────┬────────┘
                                       │
                              ┌────────▼────────┐
                              │   Container     │
                              │   Process       │
                              │                 │
                              │  (isolated via  │
                              │   namespaces,   │
                              │   cgroups)      │
                              └─────────────────┘
```

### 9.5.2 Key Services

**Image Service**: Manages container images. Handles pulling from registries,
storing image metadata, and resolving image references to content digests.

**Container Service**: Manages container metadata (what image a container uses,
its config, labels, etc.). Note: a "container" in containerd is just metadata —
it doesn't run anything until you create a Task.

**Task Service**: The actual running instance of a container. Tasks have processes
with PIDs. Creating a task from a container starts the isolated environment.

**Content Store**: Stores immutable blobs (layers, configs, manifests) in a
content-addressable store. Every blob is identified by its SHA-256 digest.

**Snapshotter**: Manages filesystem snapshots for container layers. The default
snapshotter uses overlayfs, but alternatives include devicemapper, native
(copy-based), and stargz (lazy-pulling).

**Lease Service**: Prevents garbage collection from removing content that is still
in use. When pulling an image, a lease is taken on the content to ensure the GC
doesn't collect it mid-pull.

**Namespace Service**: containerd supports multiple namespaces for multi-tenancy.
Docker uses the "moby" namespace. Kubernetes uses "k8s.io". These are **not**
Linux kernel namespaces — they're logical partitions within containerd.

### 9.5.3 Namespaces in containerd

containerd namespaces are a multi-tenancy mechanism:

```
    containerd Namespaces
    ─────────────────────────────────────────────

    ┌─────────────────────────────────────────┐
    │              containerd                  │
    │                                          │
    │  ┌──────────────┐  ┌──────────────────┐ │
    │  │  "moby"      │  │  "k8s.io"        │ │
    │  │  (Docker)    │  │  (Kubernetes)     │ │
    │  │              │  │                   │ │
    │  │  images:     │  │  images:          │ │
    │  │  - nginx     │  │  - pause:3.9     │ │
    │  │  - redis     │  │  - coredns       │ │
    │  │              │  │  - nginx         │ │
    │  │  containers: │  │                   │ │
    │  │  - web-01    │  │  containers:      │ │
    │  │  - cache-01  │  │  - coredns-abc   │ │
    │  └──────────────┘  └──────────────────┘ │
    │                                          │
    │  ┌──────────────┐                        │
    │  │  "default"   │                        │
    │  │  (ctr/nerdctl│                        │
    │  │   CLI tools) │                        │
    │  └──────────────┘                        │
    └─────────────────────────────────────────┘
```

Docker and Kubernetes can share the same containerd instance without conflicts.
Images in the "moby" namespace are invisible to Kubernetes and vice versa, even
if they reference the same underlying content in the content store (the blobs
themselves are shared for deduplication).

### 9.5.4 Shim v2 API — Decoupling containerd from Container Processes

The **shim** is one of containerd's most important architectural decisions. Instead
of containerd directly managing container processes, it delegates to a **shim
process** that acts as the parent of each container:

```
    Shim Architecture
    ═════════════════════════════════════════════════════════════

    ┌────────────┐          ┌──────────────┐         ┌─────────┐
    │ containerd │  ttrpc   │   shim v2    │  fork   │  runc   │
    │            │─────────►│              │────────►│ create  │
    │  manages   │          │  per-container│         │         │
    │  lifecycle │◄─────────│  supervisor   │         │  → init │
    │  metadata  │  events  │              │         │  → exec │
    │            │          │  holds stdio │         └─────────┘
    └──────┬─────┘          │  reaps zombie│
           │                │  reports exit│
           │                └──────────────┘
           │
    containerd can restart
    without killing containers!

    Why Shims Exist:
    ─────────────────────────────────────────────
    1. containerd can restart/upgrade without stopping containers
    2. Each container's stdio is managed by its own shim
    3. Zombie process reaping happens in the shim, not containerd
    4. Different shims can use different runtimes:
       - containerd-shim-runc-v2   → runc/crun
       - containerd-shim-kata-v2   → Kata Containers (VM)
       - containerd-shim-runsc-v1  → gVisor (user-space kernel)
       - containerd-shim-spin-v2   → Spin (WebAssembly)
```

The shim binary is named following the convention `containerd-shim-<runtime>-<version>`.
containerd locates the shim binary on `$PATH` and launches it when creating a task.

The shim communicates with containerd via **ttrpc** (a lightweight gRPC variant
optimized for local communication). Key operations:
- `Create`: Start a new container via the OCI runtime.
- `Start`: Signal the container to begin executing.
- `Delete`: Clean up container resources.
- `Kill`: Send a signal to the container.
- `Exec`: Run an additional process in the container.
- `ResizePty`: Resize the container's pseudo-terminal.
- `CloseIO`: Close stdin.

### 9.5.5 Hands-on: Using containerd Client Library in Go

```go
// Package main demonstrates using the containerd Go client library to
// perform the full container lifecycle: pull an image, create a
// container, start it as a task, wait for it to complete, and collect
// its output. This is the programmatic equivalent of:
//   ctr image pull docker.io/library/alpine:3.19
//   ctr run --rm docker.io/library/alpine:3.19 demo /bin/echo "hello"
package main

import (
	"context"
	"fmt"
	"os"
	"syscall"
	"time"

	containerd "github.com/containerd/containerd/v2/client"
	"github.com/containerd/containerd/v2/pkg/cio"
	"github.com/containerd/containerd/v2/pkg/namespaces"
	"github.com/containerd/containerd/v2/pkg/oci"
)

func main() {
	if err := run(); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
}

// run performs the complete container lifecycle using the containerd
// client library. Each step is annotated to show the mapping between
// containerd API concepts and container operations.
func run() error {
	// Step 1: Connect to the containerd daemon via its Unix socket.
	// The default socket path is /run/containerd/containerd.sock.
	// containerd must be running for this connection to succeed.
	client, err := containerd.New("/run/containerd/containerd.sock")
	if err != nil {
		return fmt.Errorf("connecting to containerd: %w", err)
	}
	defer client.Close()

	// Step 2: Set the containerd namespace. Namespaces are containerd's
	// multi-tenancy mechanism (not Linux namespaces). Docker uses "moby",
	// Kubernetes uses "k8s.io", and CLI tools typically use "default".
	ctx := namespaces.WithNamespace(context.Background(), "demo")

	// Step 3: Pull an image from a registry. This downloads the manifest,
	// config, and all layer blobs into containerd's content store, then
	// unpacks the layers using the default snapshotter (overlayfs).
	fmt.Println("Pulling image...")
	image, err := client.Pull(ctx, "docker.io/library/alpine:3.19",
		containerd.WithPullUnpack,
	)
	if err != nil {
		return fmt.Errorf("pulling image: %w", err)
	}
	fmt.Printf("✓ Pulled: %s\n", image.Name())
	fmt.Printf("  Size: %d bytes\n", image.Target().Size)
	fmt.Printf("  Digest: %s\n\n", image.Target().Digest)

	// Step 4: Create a container. In containerd, a "container" is
	// metadata that describes the container configuration: which image
	// to use, what OCI spec to apply, what snapshotter to use, etc.
	// Creating a container does NOT start any processes.
	fmt.Println("Creating container...")
	container, err := client.NewContainer(ctx, "demo-container",
		// Use the pulled image as the container's root filesystem.
		containerd.WithImage(image),
		// Allocate a new read-write snapshot for this container's
		// filesystem, layered on top of the image's read-only layers.
		containerd.WithNewSnapshot("demo-snapshot", image),
		// Generate an OCI runtime spec with default settings and
		// override the process args to run our command.
		containerd.WithNewSpec(
			oci.WithImageConfig(image),
			oci.WithProcessArgs("/bin/echo", "Hello from containerd!"),
		),
	)
	if err != nil {
		return fmt.Errorf("creating container: %w", err)
	}
	defer container.Delete(ctx, containerd.WithSnapshotCleanup)
	fmt.Printf("✓ Created container: %s\n\n", container.ID())

	// Step 5: Create a task from the container. A "task" is the actual
	// running instance — it has a PID, consumes resources, and produces
	// I/O. Creating a task sets up the namespaces and cgroups (via the
		// shim → runc chain) but doesn't start the user process yet.
	fmt.Println("Creating task...")
	task, err := container.NewTask(ctx, cio.NewCreator(cio.WithStdio))
	if err != nil {
		return fmt.Errorf("creating task: %w", err)
	}
	defer task.Delete(ctx)
	fmt.Printf("✓ Task created (PID: %d)\n\n", task.Pid())

	// Step 6: Set up a channel to wait for the task to exit. We must
	// call Wait() before Start() to avoid a race where the task exits
	// before we begin waiting.
	exitStatusC, err := task.Wait(ctx)
	if err != nil {
		return fmt.Errorf("waiting for task: %w", err)
	}

	// Step 7: Start the task. This signals the container's init process
	// to call execve() with the specified process args. The container
	// transitions from "created" to "running".
	fmt.Println("Starting task...")
	if err := task.Start(ctx); err != nil {
		return fmt.Errorf("starting task: %w", err)
	}
	fmt.Println("✓ Task started\n")

	// Step 8: Wait for the task to exit and collect the exit status.
	// The channel receives an ExitStatus when the container's main
	// process terminates.
	select {
	case exitStatus := <-exitStatusC:
		code, _, err := exitStatus.Result()
		if err != nil {
			return fmt.Errorf("getting exit status: %w", err)
		}
		fmt.Printf("✓ Task exited with code: %d\n", code)
	case <-time.After(30 * time.Second):
		// Safety timeout — kill the task if it runs too long.
		fmt.Println("Timeout! Killing task...")
		task.Kill(ctx, syscall.SIGKILL)
		<-exitStatusC
	}

	return nil
}
```

### 9.5.6 Running a Long-Lived Container with Log Streaming

```go
// Package main demonstrates running a long-lived container with
// containerd and streaming its stdout/stderr output to the host.
// This is the programmatic equivalent of "docker run -d" followed
// by "docker logs -f".
package main

import (
	"bytes"
	"context"
	"fmt"
	"os"
	"syscall"
	"time"

	containerd "github.com/containerd/containerd/v2/client"
	"github.com/containerd/containerd/v2/pkg/cio"
	"github.com/containerd/containerd/v2/pkg/namespaces"
	"github.com/containerd/containerd/v2/pkg/oci"
)

func main() {
	if err := run(); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
}

// run creates a container that prints a message every second for 5
// seconds, captures its output, and demonstrates graceful shutdown.
func run() error {
	client, err := containerd.New("/run/containerd/containerd.sock")
	if err != nil {
		return fmt.Errorf("connecting: %w", err)
	}
	defer client.Close()

	ctx := namespaces.WithNamespace(context.Background(), "demo")

	// Pull the image if not already present.
	image, err := client.Pull(ctx, "docker.io/library/alpine:3.19",
		containerd.WithPullUnpack,
	)
	if err != nil {
		return fmt.Errorf("pulling: %w", err)
	}

	// Create a container that runs a shell loop printing timestamps.
	container, err := client.NewContainer(ctx, "long-running-demo",
		containerd.WithImage(image),
		containerd.WithNewSnapshot("long-running-snapshot", image),
		containerd.WithNewSpec(
			oci.WithImageConfig(image),
			oci.WithProcessArgs("/bin/sh", "-c",
				`for i in 1 2 3 4 5; do echo "[$(date)] Iteration $i"; sleep 1; done`),
		),
	)
	if err != nil {
		return fmt.Errorf("creating container: %w", err)
	}
	defer container.Delete(ctx, containerd.WithSnapshotCleanup)

	// Capture stdout and stderr into buffers instead of forwarding
	// directly to the host's stdio. This is useful for programmatic
	// log collection and processing.
	var stdout, stderr bytes.Buffer
	task, err := container.NewTask(ctx,
		cio.NewCreator(cio.WithStreams(nil, &stdout, &stderr)),
	)
	if err != nil {
		return fmt.Errorf("creating task: %w", err)
	}
	defer task.Delete(ctx)

	exitCh, err := task.Wait(ctx)
	if err != nil {
		return fmt.Errorf("waiting: %w", err)
	}

	if err := task.Start(ctx); err != nil {
		return fmt.Errorf("starting: %w", err)
	}
	fmt.Printf("Container started (PID: %d)\n\n", task.Pid())

	// Poll for output while the container runs. In production, you
	// would use streaming I/O instead of polling.
	ticker := time.NewTicker(2 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case exitStatus := <-exitCh:
			code, _, _ := exitStatus.Result()
			fmt.Printf("\n--- Container stdout ---\n%s", stdout.String())
			if stderr.Len() > 0 {
				fmt.Printf("\n--- Container stderr ---\n%s", stderr.String())
			}
			fmt.Printf("\nExited with code: %d\n", code)
			return nil
		case <-ticker.C:
			fmt.Printf("  [output so far: %d bytes stdout, %d bytes stderr]\n",
				stdout.Len(), stderr.Len())
		case <-time.After(30 * time.Second):
			fmt.Println("Timeout, killing container...")
			task.Kill(ctx, syscall.SIGKILL)
		}
	}
}
```

---

## 9.6 Container Image Management

### 9.6.1 Image Pulling — Registry to Local Storage

When you pull an image, several layers of software collaborate:

```
    Image Pull Flow
    ═══════════════════════════════════════════════════════════

    Client (docker/ctr/crictl)
      │
      ▼
    containerd Image Service
      │
      │  1. Resolve reference → registry
      │  2. Fetch manifest (GET /v2/<name>/manifests/<ref>)
      │  3. Parse manifest → list of layer digests
      │  4. For each layer:
      │     a. Check content store (already have it?)
      │     b. If not, fetch blob (GET /v2/<name>/blobs/<digest>)
      │     c. Verify digest
      │     d. Store in content store
      │  5. Unpack layers → snapshotter
      │
      ▼
    Content Store                    Snapshotter
    ┌────────────────────┐          ┌─────────────────────┐
    │ blobs/             │          │ Active snapshots:    │
    │   sha256/          │  unpack  │   base-layer (ro)    │
    │     a3ed95ca... ───┼────────►│   app-layer  (ro)    │
    │     b7d8f2e1...    │          │   container  (rw)    │
    │     c9f0e3d2...    │          │                      │
    │                    │          │ Uses overlayfs:      │
    │ Immutable, content │          │ lower=base:app       │
    │ addressable store  │          │ upper=container-rw   │
    └────────────────────┘          └─────────────────────┘
```

### 9.6.2 Snapshotter — Filesystem Layer Management

The **snapshotter** is containerd's abstraction for managing filesystem layers.
It replaces Docker's "storage driver" concept with a cleaner interface:

| Snapshotter | Backend | Use Case |
|-------------|---------|----------|
| **overlayfs** | Linux overlay filesystem | Default, best performance |
| **native** | Simple copy | Fallback when overlayfs unavailable |
| **devmapper** | Device mapper thin provisioning | Block-level, good for write-heavy |
| **stargz** | eStargz lazy-loading | Pull only needed files on demand |
| **zfs** | ZFS filesystem | ZFS-based hosts |

The snapshotter interface is beautifully simple:

```go
// Snapshotter defines the interface for managing filesystem snapshots.
// Each snapshot is a layer in the container's filesystem. Snapshots
// can be read-only ("committed") or read-write ("active"). The
// snapshotter handles the details of how layers are composed — whether
// via overlayfs mount options, device mapper thin volumes, or simple
// file copies.
type Snapshotter interface {
	// Stat returns information about a snapshot: its name, parent,
	// kind (active vs committed), creation time, and update time.
	Stat(ctx context.Context, key string) (Info, error)

	// Prepare creates a new active (read-write) snapshot with the
	// given key, based on the specified parent snapshot. The returned
	// Mounts can be used to mount the snapshot's filesystem. For
	// overlayfs, this returns an overlay mount combining all parent
	// layers as lowerdirs with a new upperdir for writes.
	Prepare(ctx context.Context, key, parent string, opts ...Opt) ([]Mount, error)

	// View creates a read-only view of a snapshot. Useful for
	// inspecting a layer's contents without modification.
	View(ctx context.Context, key, parent string, opts ...Opt) ([]Mount, error)

	// Commit converts an active (read-write) snapshot into a
	// committed (read-only) snapshot. This is called after unpacking
	// a layer to make it available as a parent for other snapshots.
	Commit(ctx context.Context, name, key string, opts ...Opt) error

	// Mounts returns the mount operations needed to access the
	// snapshot's filesystem.
	Mounts(ctx context.Context, key string) ([]Mount, error)

	// Remove deletes a snapshot and its associated resources.
	Remove(ctx context.Context, key string) error
}
```

### 9.6.3 Content Store — How Blobs Are Stored

containerd's content store is a straightforward content-addressable blob store:

```
    /var/lib/containerd/io.containerd.content.v1.content/
    ├── blobs/
    │   └── sha256/
    │       ├── a3ed95caeb02ffe68cdd9fd84406680ae93d633cb16422d00e8a7c22955b46d4
    │       ├── b7d8f2e1a9c0... (image config)
    │       └── c9f0e3d2a1b4... (manifest)
    └── ingest/
        └── <temporary files during download>
```

The `ingest` directory holds temporary files during blob downloads. Once a blob
is fully downloaded and its digest verified, it is atomically renamed into the
`blobs/sha256/` directory. This ensures that partially downloaded blobs never
appear in the content store.

### 9.6.4 Garbage Collection

containerd's garbage collector removes unreferenced content and snapshots:

```
    Garbage Collection
    ─────────────────────────────────────────────

    Step 1: Mark phase
      - Start from image references (roots)
      - Follow manifest → config, layers
      - Follow snapshot parents
      - Mark all reachable content

    Step 2: Sweep phase
      - Delete unmarked blobs from content store
      - Delete unreferenced snapshots

    Leases prevent GC during operations:
      - Image pull acquires a lease on content
      - Container creation acquires a lease on snapshots
      - Leases are released when operations complete
      - Expired leases are cleaned up automatically
```

### 9.6.5 Hands-on: Working with containerd's Snapshotter in Go

```go
// Package main demonstrates direct interaction with containerd's
// snapshotter service. It shows how to create, mount, and inspect
// filesystem snapshots that form the layers of a container's rootfs.
// This is the layer management that happens behind the scenes when
// you pull an image or create a container.
package main

import (
	"context"
	"fmt"
	"os"
	"strings"

	containerd "github.com/containerd/containerd/v2/client"
	"github.com/containerd/containerd/v2/pkg/namespaces"
)

func main() {
	if err := run(); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
}

// run demonstrates the snapshot lifecycle: prepare, mount, commit,
// and inspect. Each snapshot represents a filesystem layer.
func run() error {
	// Connect to containerd.
	client, err := containerd.New("/run/containerd/containerd.sock")
	if err != nil {
		return fmt.Errorf("connecting: %w", err)
	}
	defer client.Close()

	ctx := namespaces.WithNamespace(context.Background(), "demo")

	// Get the default snapshotter (typically overlayfs).
	snapshotter := client.SnapshotService("overlayfs")

	// Step 1: Prepare an active (read-write) snapshot with no parent.
	// This represents the base layer of a container image.
	fmt.Println("=== Preparing base snapshot ===")
	mounts, err := snapshotter.Prepare(ctx, "base-active", "")
	if err != nil {
		return fmt.Errorf("prepare base: %w", err)
	}
	fmt.Printf("Base mounts: %+v\n", mounts)

	// In a real scenario, we would mount these and populate the
	// filesystem with the base image layer contents.
	// mount(mounts, "/mnt/base-layer")
	// unpackTar(layerBlob, "/mnt/base-layer")

	// Step 2: Commit the active snapshot to make it read-only.
	// Committed snapshots can serve as parents for new snapshots.
	fmt.Println("\n=== Committing base snapshot ===")
	if err := snapshotter.Commit(ctx, "base-committed", "base-active"); err != nil {
		return fmt.Errorf("commit base: %w", err)
	}
	fmt.Println("Base layer committed (now read-only)")

	// Step 3: Prepare a new active snapshot on top of the base.
	// This represents an application layer in the image.
	fmt.Println("\n=== Preparing app layer snapshot ===")
	appMounts, err := snapshotter.Prepare(ctx, "app-active", "base-committed")
	if err != nil {
		return fmt.Errorf("prepare app: %w", err)
	}

	// For overlayfs, the mounts will show the layer composition:
	// lower=base-committed, upper=app-active-rw
	for _, m := range appMounts {
		fmt.Printf("  Mount type: %s\n", m.Type)
		fmt.Printf("  Source: %s\n", m.Source)
		for _, opt := range m.Options {
			if strings.HasPrefix(opt, "lowerdir=") || strings.HasPrefix(opt, "upperdir=") {
				fmt.Printf("  Option: %s\n", opt)
			}
		}
	}

	// Step 4: Commit the app layer.
	if err := snapshotter.Commit(ctx, "app-committed", "app-active"); err != nil {
		return fmt.Errorf("commit app: %w", err)
	}

	// Step 5: Prepare a read-write container snapshot on top of
	// the committed layers. This is what a running container uses —
	// all writes go to this snapshot's upperdir.
	fmt.Println("\n=== Preparing container snapshot ===")
	containerMounts, err := snapshotter.Prepare(ctx, "container-rw", "app-committed")
	if err != nil {
		return fmt.Errorf("prepare container: %w", err)
	}
	fmt.Printf("Container filesystem ready with %d mount(s)\n", len(containerMounts))
	for _, m := range containerMounts {
		fmt.Printf("  %s mount: %s\n", m.Type, m.Source)
	}

	// Step 6: Inspect snapshot metadata.
	fmt.Println("\n=== Snapshot Info ===")
	for _, name := range []string{"base-committed", "app-committed"} {
		info, err := snapshotter.Stat(ctx, name)
		if err != nil {
			fmt.Printf("  %s: error: %v\n", name, err)
			continue
		}
		fmt.Printf("  %s:\n", info.Name)
		fmt.Printf("    Parent: %s\n", info.Parent)
		fmt.Printf("    Kind:   %v\n", info.Kind)
		fmt.Printf("    Created: %s\n", info.Created)
	}

	// Cleanup: remove snapshots in reverse order (children first).
	fmt.Println("\n=== Cleanup ===")
	for _, name := range []string{"container-rw", "app-committed", "base-committed"} {
		if err := snapshotter.Remove(ctx, name); err != nil {
			fmt.Printf("  Remove %s: %v\n", name, err)
		} else {
			fmt.Printf("  Removed %s\n", name)
		}
	}

	return nil
}
```

### 9.6.6 Hands-on: Building a Simple Image Puller in Go

```go
// Package main implements a minimal but complete image puller using
// the containerd client library. It demonstrates how containerd
// resolves image references, downloads content, and unpacks layers
// into the snapshotter. This is the equivalent of `ctr image pull`
// but written explicitly to show each step.
package main

import (
	"context"
	"fmt"
	"os"

	containerd "github.com/containerd/containerd/v2/client"
	"github.com/containerd/containerd/v2/core/images"
	"github.com/containerd/containerd/v2/pkg/namespaces"
	ocispec "github.com/opencontainers/image-spec/specs-go/v1"
)

func main() {
	if err := pullImage(); err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}
}

// pullImage demonstrates a step-by-step image pull with detailed
// logging at each stage. It shows the progression from image
// reference resolution through manifest parsing to layer unpacking.
func pullImage() error {
	client, err := containerd.New("/run/containerd/containerd.sock")
	if err != nil {
		return fmt.Errorf("connecting: %w", err)
	}
	defer client.Close()

	ctx := namespaces.WithNamespace(context.Background(), "demo")
	ref := "docker.io/library/nginx:alpine"

	fmt.Printf("=== Pulling %s ===\n\n", ref)

	// Step 1: Pull with progress reporting. The WithPullUnpack option
	// tells containerd to unpack the image layers into the snapshotter
	// immediately after downloading.
	image, err := client.Pull(ctx, ref,
		containerd.WithPullUnpack,
	)
	if err != nil {
		return fmt.Errorf("pull failed: %w", err)
	}

	// Step 2: Inspect the pulled image metadata.
	fmt.Printf("Image: %s\n", image.Name())
	fmt.Printf("Target digest: %s\n", image.Target().Digest)
	fmt.Printf("Target media type: %s\n", image.Target().MediaType)
	fmt.Printf("Target size: %d bytes\n\n", image.Target().Size)

	// Step 3: Read the image manifest to inspect its layers.
	// We use the images.Manifest helper to resolve and parse the
	// manifest from the content store.
	cs := client.ContentStore()
	manifest, err := images.Manifest(ctx, cs, image.Target(), nil)
	if err != nil {
		return fmt.Errorf("reading manifest: %w", err)
	}

	fmt.Printf("Manifest layers: %d\n", len(manifest.Layers))
	for i, layer := range manifest.Layers {
		fmt.Printf("  [%d] %s\n", i, layer.MediaType)
		fmt.Printf("       digest: %s\n", layer.Digest)
		fmt.Printf("       size:   %d bytes (%.2f MB)\n\n",
			layer.Size, float64(layer.Size)/1024/1024)
	}

	// Step 4: Read the image config to see runtime defaults.
	configDesc := manifest.Config
	fmt.Printf("Config: %s (%d bytes)\n", configDesc.Digest, configDesc.Size)

	// Read the config blob from the content store.
	configReader, err := cs.ReaderAt(ctx, ocispec.Descriptor{
			Digest: configDesc.Digest,
			Size:   configDesc.Size,
		})
	if err != nil {
		return fmt.Errorf("reading config: %w", err)
	}
	defer configReader.Close()

	// Step 5: List all images in this namespace.
	fmt.Println("\n=== All images in namespace ===")
	imageList, err := client.ImageService().List(ctx)
	if err != nil {
		return fmt.Errorf("listing images: %w", err)
	}
	for _, img := range imageList {
		fmt.Printf("  %s (%s)\n", img.Name, img.Target.Digest.String()[:24])
	}

	return nil
}
```

---

## 9.7 CRI (Container Runtime Interface)

### 9.7.1 What CRI Is

The **Container Runtime Interface (CRI)** is Kubernetes' plugin API for container
runtimes. Before CRI, Kubernetes had hardcoded Docker support (the "dockershim").
CRI abstracts the runtime, allowing Kubernetes to work with any compliant runtime.

```
    CRI in the Kubernetes Stack
    ═══════════════════════════════════════════════════

    ┌──────────────┐
    │   kubelet    │  (Kubernetes node agent)
    │              │
    │  CRI client  │
    └──────┬───────┘
           │  gRPC
           │
    ┌──────▼───────┐     ┌──────────────┐     ┌──────────┐
    │  containerd  │────►│  shim-runc   │────►│  runc    │
    │  (CRI plugin)│     │              │     │          │
    └──────────────┘     └──────────────┘     └──────────┘
           OR
    ┌──────▼───────┐     ┌──────────────┐     ┌──────────┐
    │   CRI-O     │────►│  conmon       │────►│  runc    │
    │              │     │              │     │          │
    └──────────────┘     └──────────────┘     └──────────┘
```

### 9.7.2 CRI API — RuntimeService and ImageService

CRI defines two gRPC services:

**RuntimeService** — Container and pod lifecycle:
```protobuf
// RuntimeService defines the gRPC interface that Kubernetes' kubelet
// uses to manage containers and pods. A CRI implementation must
// provide all of these RPCs for the kubelet to function correctly.
service RuntimeService {
    // RunPodSandbox creates and starts a pod-level sandbox. In Linux,
    // this creates the network namespace and other pod-level resources.
    rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse);

    // StopPodSandbox stops a running pod sandbox.
    rpc StopPodSandbox(StopPodSandboxRequest) returns (StopPodSandboxResponse);

    // RemovePodSandbox removes a pod sandbox and all associated data.
    rpc RemovePodSandbox(RemovePodSandboxRequest) returns (RemovePodSandboxResponse);

    // CreateContainer creates a new container within a pod sandbox.
    rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse);

    // StartContainer starts a previously created container.
    rpc StartContainer(StartContainerRequest) returns (StartContainerResponse);

    // StopContainer stops a running container with a specified timeout.
    rpc StopContainer(StopContainerRequest) returns (StopContainerResponse);

    // RemoveContainer removes a container and its associated resources.
    rpc RemoveContainer(RemoveContainerRequest) returns (RemoveContainerResponse);

    // ExecSync executes a command in a container synchronously.
    rpc ExecSync(ExecSyncRequest) returns (ExecSyncResponse);

    // ListContainers lists all containers matching the given filter.
    rpc ListContainers(ListContainersRequest) returns (ListContainersResponse);

    // ContainerStatus returns detailed status of a container.
    rpc ContainerStatus(ContainerStatusRequest) returns (ContainerStatusResponse);

    // ListPodSandbox lists all pod sandboxes matching the given filter.
    rpc ListPodSandbox(ListPodSandboxRequest) returns (ListPodSandboxResponse);
}
```

**ImageService** — Image management:
```protobuf
// ImageService defines the gRPC interface for managing container
// images. The kubelet uses this to pull images needed by pods and
// to manage the node's image cache.
service ImageService {
    // PullImage pulls an image from a registry to the node.
    rpc PullImage(PullImageRequest) returns (PullImageResponse);

    // RemoveImage removes an image from the node.
    rpc RemoveImage(RemoveImageRequest) returns (RemoveImageResponse);

    // ImageStatus returns the status (size, digest, tags) of an image.
    rpc ImageStatus(ImageStatusRequest) returns (ImageStatusResponse);

    // ListImages lists all images on the node.
    rpc ListImages(ListImagesRequest) returns (ListImagesResponse);

    // ImageFsInfo returns information about the image filesystem usage.
    rpc ImageFsInfo(ImageFsInfoRequest) returns (ImageFsInfoResponse);
}
```

### 9.7.3 How containerd Implements CRI

containerd implements CRI as a built-in plugin (the `cri` plugin). When the
kubelet sends a CRI request, containerd translates it into its internal API:

```
    CRI Request Mapping
    ─────────────────────────────────────────────

    CRI: RunPodSandbox
      → containerd: create container (pause)
      → containerd: create task
      → containerd: start task
      → setup CNI networking

    CRI: CreateContainer
      → containerd: pull image (if needed)
      → containerd: create container
      → containerd: create snapshot

    CRI: StartContainer
      → containerd: create task
      → containerd: start task

    CRI: StopContainer
      → containerd: kill task (SIGTERM, then SIGKILL)
      → containerd: delete task

    CRI: RemoveContainer
      → containerd: delete container
      → containerd: remove snapshot
```

The **pod sandbox** is a special container running the `pause` image. It holds
the network namespace and other pod-level resources. All containers in the pod
share the sandbox's network namespace, which is why containers in the same pod
can communicate over `localhost`.

### 9.7.4 CRI-O — The Alternative CRI Implementation

**CRI-O** is a lightweight CRI implementation designed specifically for Kubernetes.
Unlike containerd (which is a general-purpose container daemon), CRI-O does exactly
one thing: implement CRI.

| Feature | containerd | CRI-O |
|---------|-----------|-------|
| Purpose | General container daemon | Kubernetes-only CRI |
| Used by | Docker, K8s, nerdctl, cloud providers | K8s only (Red Hat, SUSE) |
| OCI Runtime | runc, kata, gVisor (via shims) | runc, crun, kata (via conmon) |
| Image format | OCI + Docker | OCI + Docker |
| Scope | Images, containers, tasks, content, snapshots | CRI only |

### 9.7.5 Hands-on: Making CRI Calls with crictl

```bash
#!/bin/bash
# cri-demo.sh — Demonstrates using crictl to interact with a CRI
# runtime (containerd or CRI-O). crictl is the official CLI tool
# for debugging CRI implementations.

set -euo pipefail

# Configure crictl to talk to containerd's CRI socket.
# For CRI-O, use: unix:///var/run/crio/crio.sock
export CONTAINER_RUNTIME_ENDPOINT=unix:///run/containerd/containerd.sock

echo "=== CRI Runtime Info ==="
crictl info | head -20

echo ""
echo "=== Pull an image via CRI ==="
crictl pull docker.io/library/alpine:3.19

echo ""
echo "=== List images ==="
crictl images

echo ""
echo "=== Create a pod sandbox ==="
# CRI requires a pod sandbox config in JSON format.
cat > pod-config.json <<EOF
{
    "metadata": {
        "name": "demo-pod",
        "namespace": "default",
        "uid": "demo-uid-12345"
    },
    "log_directory": "/tmp/demo-pod-logs"
}
EOF

POD_ID=$(crictl runp pod-config.json)
echo "Pod sandbox created: $POD_ID"

echo ""
echo "=== Create a container in the pod ==="
cat > container-config.json <<EOF
{
    "metadata": {
        "name": "demo-container"
    },
    "image": {
        "image": "docker.io/library/alpine:3.19"
    },
    "command": ["/bin/sh", "-c", "echo 'Hello from CRI!' && sleep 5"],
    "log_path": "demo-container.log"
}
EOF

CONTAINER_ID=$(crictl create "$POD_ID" container-config.json pod-config.json)
echo "Container created: $CONTAINER_ID"

echo ""
echo "=== Start the container ==="
crictl start "$CONTAINER_ID"

echo ""
echo "=== Container status ==="
crictl inspect "$CONTAINER_ID" | head -30

echo ""
echo "=== Container logs ==="
sleep 2
crictl logs "$CONTAINER_ID"

echo ""
echo "=== List running containers ==="
crictl ps

echo ""
echo "=== Cleanup ==="
crictl stop "$CONTAINER_ID" 2>/dev/null || true
crictl rm "$CONTAINER_ID"
crictl stopp "$POD_ID"
crictl rmp "$POD_ID"
rm -f pod-config.json container-config.json

echo "Done!"
```

---

## 9.8 Docker's Place in the Ecosystem

### 9.8.1 Docker Engine → containerd → runc

Modern Docker is a layered system:

```
    Docker Architecture (2024)
    ═══════════════════════════════════════════════════

    ┌──────────────────────────────┐
    │        docker CLI            │  User-facing tool
    │        (docker run, build)   │
    └─────────────┬────────────────┘
                  │ REST API
    ┌─────────────▼────────────────┐
    │        dockerd               │  Docker daemon
    │        (Docker Engine)       │
    │                              │
    │  - Build images (BuildKit)   │
    │  - Manage networks           │
    │  - Manage volumes            │
    │  - Swarm orchestration       │
    │  - Docker Compose            │
    └─────────────┬────────────────┘
                  │ gRPC
    ┌─────────────▼────────────────┐
    │        containerd            │  Container runtime daemon
    │                              │
    │  - Image management          │
    │  - Container lifecycle       │
    │  - Snapshotter               │
    │  - Content store             │
    └─────────────┬────────────────┘
                  │ ttrpc (shim API)
    ┌─────────────▼────────────────┐
    │  containerd-shim-runc-v2     │  Per-container supervisor
    └─────────────┬────────────────┘
                  │ fork+exec
    ┌─────────────▼────────────────┐
    │          runc                │  OCI runtime
    │                              │
    │  - Create namespaces         │
    │  - Set up cgroups            │
    │  - Mount rootfs              │
    │  - Apply seccomp             │
    │  - exec user process         │
    └──────────────────────────────┘
```

When you run `docker run alpine echo hello`, this happens:

1. **docker CLI** sends a request to **dockerd** via REST API.
2. **dockerd** translates the request, pulls the image (via containerd), creates
   a container (via containerd), and starts it.
3. **containerd** launches **containerd-shim-runc-v2** for this container.
4. The shim invokes **runc** to create the actual Linux container.
5. runc calls `clone()`, `pivot_root()`, `execve()` — and `echo hello` runs.

### 9.8.2 Why Kubernetes Removed dockershim

Kubernetes 1.24 (May 2022) removed **dockershim**, the built-in Docker support.
The reason was architectural:

```
    Before (with dockershim):
    ─────────────────────────────────────────────
    kubelet → dockershim → dockerd → containerd → runc
              ^^^^^^^^^^^
              Extra layer maintained by Kubernetes

    After (direct CRI):
    ─────────────────────────────────────────────
    kubelet → containerd (CRI plugin) → runc
              No extra layers!
```

dockershim was a translation layer maintained within the Kubernetes codebase. It
translated CRI calls into Docker API calls. But Docker's API is much broader than
CRI needs — it includes build, Swarm, volumes, and networks. Maintaining dockershim
meant Kubernetes was carrying a translation layer for features it didn't use.

By removing dockershim, Kubernetes talks directly to containerd (or CRI-O) via CRI.
This is simpler, more efficient, and removes a maintenance burden. Importantly,
Docker-built images still work perfectly — they're OCI-compliant and any CRI runtime
can run them.

### 9.8.3 Docker vs containerd vs CRI-O Comparison

| Feature | Docker | containerd | CRI-O |
|---------|--------|-----------|-------|
| Image build | ✓ (BuildKit) | ✗ (use buildctl) | ✗ (use Buildah) |
| Image push/pull | ✓ | ✓ | ✓ |
| Docker Compose | ✓ | ✗ (use nerdctl) | ✗ |
| Swarm mode | ✓ | ✗ | ✗ |
| CRI support | ✗ (removed) | ✓ (built-in plugin) | ✓ (native) |
| CLI tool | docker | ctr / nerdctl | crictl |
| Developer experience | ★★★★★ | ★★★☆☆ | ★★☆☆☆ |
| Production K8s | Via containerd | ✓ | ✓ |
| Footprint | ~200MB | ~50MB | ~40MB |

---

## 9.9 Rootless Containers

### 9.9.1 Why Rootless Matters

Traditional containers require root privileges to create namespaces, configure
cgroups, and mount filesystems. This is a security concern: if a container escape
occurs, the attacker has root on the host.

**Rootless containers** run the entire container stack — runtime, daemon, and
container processes — without root privileges. This dramatically reduces the
blast radius of container escapes.

```
    Rootless vs Traditional Containers
    ═══════════════════════════════════════════════════

    Traditional (rootful):
    ┌──────────────────────────────────────────┐
    │  Host (root)                             │
    │  ┌────────────────────────────────────┐  │
    │  │ containerd (root)                  │  │
    │  │  ┌─────────────────────────────┐   │  │
    │  │  │ runc (root)                 │   │  │
    │  │  │  ┌──────────────────────┐   │   │  │
    │  │  │  │ Container (uid 0→0)  │   │   │  │
    │  │  │  │ root in container =  │   │   │  │
    │  │  │  │ root on host!       │   │   │  │
    │  │  │  └──────────────────────┘   │   │  │
    │  │  └─────────────────────────────┘   │  │
    │  └────────────────────────────────────┘  │
    └──────────────────────────────────────────┘

    Rootless:
    ┌──────────────────────────────────────────┐
    │  Host (unprivileged user uid=1000)       │
    │  ┌────────────────────────────────────┐  │
    │  │ containerd (uid=1000)              │  │
    │  │  ┌─────────────────────────────┐   │  │
    │  │  │ runc (uid=1000)             │   │  │
    │  │  │  ┌──────────────────────┐   │   │  │
    │  │  │  │ Container (uid 0→   │   │   │  │
    │  │  │  │   mapped to 1000)   │   │   │  │
    │  │  │  │ root in container = │   │   │  │
    │  │  │  │ unprivileged on host│   │   │  │
    │  │  │  └──────────────────────┘   │   │  │
    │  │  └─────────────────────────────┘   │  │
    │  └────────────────────────────────────┘  │
    └──────────────────────────────────────────┘
```

### 9.9.2 User Namespaces + Rootless Runtimes

The key technology enabling rootless containers is **user namespaces**. A user
namespace remaps UIDs: the container sees uid 0 (root), but on the host this
maps to an unprivileged uid (e.g., 100000).

```
    User Namespace UID Mapping
    ─────────────────────────────────────────────

    Container UID    Host UID
    ─────────────    ────────
    0 (root)    →    100000 (unprivileged)
    1           →    100001
    2           →    100002
    ...              ...
    65535       →    165535

    /etc/subuid:
    myuser:100000:65536
    (user "myuser" can map 65536 UIDs starting from 100000)
```

Rootless containers have some limitations:
- Cannot bind to ports below 1024 (unless using `net.ipv4.ip_unprivileged_port_start`).
- Networking requires `slirp4netns` or `pasta` (user-space networking).
- Some storage drivers don't work (overlayfs requires kernel 5.11+ for rootless).
- Cannot use certain cgroup features without systemd delegation.

### 9.9.3 Rootless containerd, Docker, and Podman

All major container tools support rootless mode:

```bash
#!/bin/bash
# rootless-demo.sh — Demonstrates running containers without root.

# === Rootless Docker ===
# Install rootless Docker (runs dockerd as your user)
dockerd-rootless-setuptool.sh install

# Now docker commands work without sudo
docker run --rm alpine echo "I am rootless Docker!"
docker info | grep "Root Dir"
# Shows: ~/.local/share/docker (not /var/lib/docker)

# === Rootless containerd ===
# Install rootless containerd
containerd-rootless-setuptool.sh install

# Use nerdctl (rootless-aware docker-compatible CLI)
nerdctl run --rm alpine echo "I am rootless containerd!"

# === Podman (rootless by default) ===
# Podman is designed rootless-first — no daemon needed
podman run --rm alpine echo "I am rootless Podman!"
podman info | grep rootless
# Shows: rootless: true
```

### 9.9.4 Hands-on: Running Rootless Containers

```go
// Package main demonstrates verifying that a container is running in
// rootless mode by checking user namespace mappings and effective UIDs.
// This program is meant to be run inside a rootless container to
// confirm the UID remapping is working correctly.
package main

import (
	"fmt"
	"os"
	"os/user"
	"strings"
)

func main() {
	fmt.Println("=== Rootless Container Verification ===\n")

	// Check the effective UID inside the container.
	// In a rootless container, we appear to be root (uid 0) but
	// the host sees us as an unprivileged user.
	fmt.Printf("UID:  %d\n", os.Getuid())
	fmt.Printf("GID:  %d\n", os.Getgid())
	fmt.Printf("EUID: %d\n", os.Geteuid())

	// Look up the current user.
	u, err := user.Current()
	if err != nil {
		fmt.Printf("User lookup error: %v\n", err)
	} else {
		fmt.Printf("User: %s (uid=%s)\n", u.Username, u.Uid)
	}

	// Read the UID map to see the user namespace remapping.
	// In a rootless container, /proc/self/uid_map shows:
	//   0  100000  65536
	// meaning container uid 0 maps to host uid 100000.
	fmt.Println("\n--- UID Mapping (/proc/self/uid_map) ---")
	uidMap, err := os.ReadFile("/proc/self/uid_map")
	if err != nil {
		fmt.Printf("Error reading uid_map: %v\n", err)
	} else {
		fmt.Print(string(uidMap))
		// Parse the mapping to determine if we're rootless.
		lines := strings.TrimSpace(string(uidMap))
		fields := strings.Fields(lines)
		if len(fields) >= 2 {
			if fields[0] == "0" && fields[1] != "0" {
				fmt.Println("✓ Running in rootless mode (uid 0 → non-zero host uid)")
			} else if fields[0] == "0" && fields[1] == "0" {
				fmt.Println("⚠ Running as real root (not rootless)")
			}
		}
	}

	// Read the GID map similarly.
	fmt.Println("\n--- GID Mapping (/proc/self/gid_map) ---")
	gidMap, err := os.ReadFile("/proc/self/gid_map")
	if err != nil {
		fmt.Printf("Error reading gid_map: %v\n", err)
	} else {
		fmt.Print(string(gidMap))
	}

	// Check if we can perform privileged operations.
	fmt.Println("\n--- Privilege Check ---")
	// Try to read a file that only real root can access.
	if _, err := os.ReadFile("/proc/kcore"); err != nil {
		fmt.Println("✓ Cannot read /proc/kcore (properly restricted)")
	} else {
		fmt.Println("⚠ Can read /proc/kcore (running with real root!)")
	}
}
```

---

## 9.10 Kata Containers & gVisor — Sandbox Runtimes

### 9.10.1 Why Sandbox Runtimes Exist

Standard containers share the host kernel. A kernel vulnerability can allow a
container to escape and compromise the host or other containers. In **multi-tenant
environments** (like public clouds running untrusted workloads), this is
unacceptable.

**Sandbox runtimes** add an additional isolation boundary:

```
    Isolation Spectrum
    ═══════════════════════════════════════════════════

    Least Isolated                           Most Isolated
    ─────────────────────────────────────────────────────

    Processes    Containers    gVisor     Kata VMs    Full VMs
    (no isolation)(namespaces) (user-space (lightweight (hardware
                  + cgroups)   kernel)    VM per pod)  virtualization)

    ←──── Performance ────────────────── Security ────►
```

### 9.10.2 Kata Containers — Lightweight VMs

Kata Containers runs each container (or pod) in its own **lightweight virtual
machine**. The VM provides hardware-enforced isolation through the hypervisor,
while remaining fast enough for container workloads.

```
    Kata Containers Architecture
    ═══════════════════════════════════════════════════

    ┌───────────────────────────────────────────────┐
    │                    Host                        │
    │                                                │
    │  ┌──────────────────────────────────────────┐ │
    │  │  containerd + containerd-shim-kata-v2    │ │
    │  └────────────────────┬─────────────────────┘ │
    │                       │                        │
    │  ┌────────────────────▼─────────────────────┐ │
    │  │            Hypervisor (QEMU/Cloud         │ │
    │  │            Hypervisor/Firecracker)        │ │
    │  │                                           │ │
    │  │  ┌────────────────────────────────────┐  │ │
    │  │  │         Lightweight VM              │  │ │
    │  │  │                                     │  │ │
    │  │  │  ┌──────────────────────────────┐  │  │ │
    │  │  │  │  Guest Kernel (minimal Linux)│  │  │ │
    │  │  │  └──────────────────────────────┘  │  │ │
    │  │  │                                     │  │ │
    │  │  │  ┌──────────┐  ┌──────────┐        │  │ │
    │  │  │  │  kata-   │  │ Container│        │  │ │
    │  │  │  │  agent   │  │ Process  │        │  │ │
    │  │  │  │ (manages │  │          │        │  │ │
    │  │  │  │  container│  │          │        │  │ │
    │  │  │  │  inside  │  │          │        │  │ │
    │  │  │  │  the VM) │  │          │        │  │ │
    │  │  │  └──────────┘  └──────────┘        │  │ │
    │  │  └────────────────────────────────────┘  │ │
    │  └───────────────────────────────────────────┘ │
    └───────────────────────────────────────────────┘
```

**How Kata works**:
1. containerd creates a task using the Kata shim (`containerd-shim-kata-v2`).
2. The Kata shim launches a lightweight VM using QEMU, Cloud Hypervisor, or
   Firecracker.
3. Inside the VM, a **kata-agent** process manages the container.
4. The container runs inside the VM with traditional namespace isolation.
5. The VM boundary provides hardware-enforced isolation from the host.

Kata is OCI-compatible: the same OCI images work with both runc and Kata. The only
difference is the isolation boundary.

**Hypervisor options**:
- **QEMU**: Full-featured, supports many architectures, higher overhead.
- **Cloud Hypervisor**: Rust-based, minimal, optimized for cloud workloads.
- **Firecracker**: Created by AWS for Lambda/Fargate, extremely fast boot (<125ms).

### 9.10.3 gVisor — User-Space Kernel

gVisor takes a different approach: instead of running a VM, it interposes a
**user-space kernel** (called **Sentry**) between the container and the host
kernel. The Sentry intercepts and re-implements Linux system calls, dramatically
reducing the host kernel's attack surface.

```
    gVisor Architecture
    ═══════════════════════════════════════════════════

    ┌───────────────────────────────────────────────┐
    │                  Host Kernel                   │
    │  (only ~70 syscalls exposed to gVisor)         │
    └────────────────────┬──────────────────────────┘
                         │
    ┌────────────────────▼──────────────────────────┐
    │              gVisor Sentry                      │
    │  (user-space kernel, implements ~300 syscalls)  │
    │                                                 │
    │  ┌──────────────────────────────────────────┐  │
    │  │  Virtual filesystem (VFS2)                │  │
    │  │  Virtual networking (netstack)            │  │
    │  │  Signal handling, memory management       │  │
    │  │  /proc, /sys emulation                   │  │
    │  └──────────────────────────────────────────┘  │
    │                                                 │
    │  Platform: ptrace or KVM                        │
    │  (how syscalls are intercepted)                 │
    └────────────────────┬──────────────────────────┘
                         │
    ┌────────────────────▼──────────────────────────┐
    │           gVisor Gofer                          │
    │  (file system proxy, runs on host)              │
    │                                                 │
    │  Sentry → Gofer: "open /etc/hosts"              │
    │  Gofer → Host FS: actual file open              │
    │  Gofer → Sentry: file descriptor                │
    │                                                 │
    │  The Sentry never directly accesses host files. │
    │  The Gofer mediates all filesystem access.       │
    └─────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────┐
    │               Container Process                   │
    │  (thinks it's talking to a Linux kernel,          │
    │   but actually talking to the Sentry)             │
    │                                                   │
    │  syscall(write, 1, "hello\n", 6)                  │
    │       │                                           │
    │       └──► intercepted by Sentry                  │
    │            └──► Sentry implements write()          │
    │                 └──► outputs to container stdout   │
    └─────────────────────────────────────────────────┘
```

**Key gVisor components**:
- **Sentry**: The user-space kernel. Implements most Linux system calls (about 300)
  in Go. The container process makes syscalls that are intercepted and handled by
  the Sentry without reaching the host kernel.
- **Gofer**: A file system proxy that mediates container access to host files.
  The Sentry communicates with the Gofer over a 9P protocol. This prevents the
  container from directly accessing the host filesystem.
- **runsc**: The OCI-compatible runtime binary (equivalent to runc). It sets up
  the Sentry and Gofer for each container.

**Platform modes**:
- **ptrace**: Uses `ptrace()` to intercept syscalls. Works everywhere but slower.
- **KVM**: Uses KVM virtualization for syscall interception. Much faster but
  requires `/dev/kvm` access.

### 9.10.4 RuntimeClass in Kubernetes

Kubernetes uses **RuntimeClass** to allow different pods to use different runtimes:

```yaml
# Define runtime classes for different isolation levels.
# RuntimeClass tells the kubelet which runtime handler (containerd
# shim) to use when creating pods that reference this class.

# Standard container runtime (runc) — fast, shared kernel.
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: standard
handler: runc

---
# gVisor runtime — user-space kernel for untrusted workloads.
# Uses the runsc shim (containerd-shim-runsc-v1).
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc

---
# Kata Containers — VM-based isolation for maximum security.
# Uses the Kata shim (containerd-shim-kata-v2).
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata
```

Pods reference a RuntimeClass to select their isolation level:

```yaml
# This pod runs with gVisor's user-space kernel.
# The runtimeClassName field tells the kubelet to use the "gvisor"
# RuntimeClass, which maps to the runsc containerd shim.
apiVersion: v1
kind: Pod
metadata:
  name: untrusted-workload
spec:
  runtimeClassName: gvisor
  containers:
  - name: app
    image: nginx:alpine
    ports:
    - containerPort: 80

---
# This pod runs inside a lightweight VM via Kata Containers.
apiVersion: v1
kind: Pod
metadata:
  name: highly-isolated-workload
spec:
  runtimeClassName: kata
  containers:
  - name: app
    image: python:3.12-slim
    command: ["python3", "-m", "http.server", "8080"]
```

**Comparison summary**:

| Aspect | runc | gVisor | Kata |
|--------|------|--------|------|
| Isolation | Namespaces + cgroups | User-space kernel | Hardware VM |
| Host kernel exposure | Full (~350 syscalls) | Minimal (~70 syscalls) | None (own kernel) |
| Performance overhead | None | ~5-30% CPU, 2x memory | ~10-20% CPU, VM boot |
| Boot time | ~100ms | ~150ms | ~500ms-2s |
| Compatibility | Full Linux | Most apps (~98%) | Full Linux |
| Use case | Trusted workloads | Untrusted code, FaaS | Multi-tenant, regulated |

---

## 9.11 Putting It All Together

Let's trace the complete journey from `kubectl run` to a running container:

```
    Complete Container Lifecycle
    ═══════════════════════════════════════════════════════════════

    1. kubectl run nginx --image=nginx:alpine
       │
    2. │  API server creates Pod object
       │
    3. ▼  Scheduler assigns Pod to a node
       │
    4. ▼  kubelet watches for assigned Pods
       │
    5. ▼  kubelet calls CRI: RunPodSandbox()
       │   → containerd creates pause container
       │   → CNI plugin configures networking
       │
    6. ▼  kubelet calls CRI: PullImage("nginx:alpine")
       │   → containerd resolves "nginx:alpine"
       │   → Fetches manifest from registry
       │   → Downloads layer blobs
       │   → Stores in content store
       │   → Unpacks via snapshotter (overlayfs)
       │
    7. ▼  kubelet calls CRI: CreateContainer()
       │   → containerd creates container metadata
       │   → Creates read-write snapshot for container
       │
    8. ▼  kubelet calls CRI: StartContainer()
       │   → containerd creates a Task
       │   → Launches containerd-shim-runc-v2
       │   → Shim invokes runc create + start
       │   → runc: clone() with CLONE_NEW*
       │   → runc: pivot_root() to rootfs
       │   → runc: mount /proc, /dev, /sys
       │   → runc: apply seccomp filter
       │   → runc: drop capabilities
       │   → runc: execve("nginx", ...)
       │
    9. ▼  nginx is running in an isolated container!
       │
   10. ▼  kubelet monitors via CRI: ContainerStatus()
       │   → containerd queries task state
       │   → Shim reports PID, resource usage
```

This is the stack we've been building toward throughout this book:

- **Chapters 1-3**: System calls, processes, `fork()`, `execve()` — the
  foundation of `runc`.
- **Chapter 4**: File systems, mounts, `pivot_root()` — how containers get
  their rootfs.
- **Chapter 5**: Networking, sockets — how containers communicate.
- **Chapter 6-7**: Namespaces, cgroups — the kernel primitives for isolation.
- **Chapter 8**: Building a container from scratch — putting it all together.
- **Chapter 9 (this chapter)**: The production ecosystem — OCI, runc, containerd,
  CRI, Kubernetes.
- **Chapter 10 (next)**: Kubernetes itself — the orchestrator that manages it all.

---

## 9.12 Summary

In this chapter, we explored the container ecosystem that transforms kernel
primitives into production infrastructure:

1. **OCI Specifications** standardize images (manifest, config, layers), runtimes
   (config.json, lifecycle), and distribution (registry HTTP API). Content-
   addressable storage with SHA-256 digests provides integrity, deduplication,
   and immutability.

2. **runc** is the reference OCI runtime that creates containers using clone(),
   pivot_root(), and execve(). Alternative runtimes (crun, youki) provide
   different trade-offs in performance and safety.

3. **containerd** is the industry-standard container daemon with a plugin-based
   architecture. Its key innovation is the **shim model** — each container gets
   its own supervisor process, allowing containerd to restart without killing
   containers.

4. **CRI** is Kubernetes' interface to container runtimes. containerd and CRI-O
   both implement CRI, allowing Kubernetes to run containers without Docker.

5. **Rootless containers** use user namespaces to eliminate the need for root
   privileges, dramatically improving security.

6. **Sandbox runtimes** (Kata Containers, gVisor) add additional isolation
   boundaries for multi-tenant environments — hardware VMs and user-space
   kernels, respectively.

The container ecosystem is a remarkable example of **layered standardization**.
Each layer has a clear responsibility and a well-defined interface. Images built
by Docker run on containerd. Runtimes that implement OCI can be used by any
orchestrator. Kubernetes uses CRI to work with any compliant runtime.

In the next chapter, we'll climb the final rung of this ladder: **Kubernetes**
itself — how it orchestrates thousands of containers across hundreds of nodes
to build reliable distributed systems.

---

## 9.13 Exercises

1. **OCI Image Inspector**: Extend the OCI image inspector from Section 9.1.3 to
   also verify layer digests by downloading and re-hashing each blob.

2. **Minimal Image Builder**: Write a Go program that creates a minimal OCI image
   from a single statically-compiled binary (no base image).

3. **Registry Explorer**: Using the Distribution API code from Section 9.3.2,
   build a CLI tool that lists repositories, tags, and manifest details from
   any OCI-compliant registry.

4. **containerd Plugin**: Write a simple containerd snapshotter plugin that logs
   all snapshot operations (prepare, commit, remove) to a file.

5. **Runtime Comparison**: Run the same workload with runc, crun, and youki.
   Measure cold-start time, memory usage, and throughput. Document your findings.

6. **CRI Debugger**: Write a Go program using the CRI gRPC API to list all pods,
   containers, and images on a Kubernetes node.

7. **Rootless Analysis**: Set up rootless containerd, run a container, and verify
   using `/proc/self/uid_map` that UID remapping is active. Document the UID
   mappings.

8. **Sandbox Runtime Comparison**: Deploy gVisor and Kata Containers on a test
   cluster. Run a syscall-heavy benchmark (e.g., a web server) on runc, gVisor,
   and Kata. Compare latency and throughput.

---

## 9.14 Further Reading

- **OCI Image Specification**: https://github.com/opencontainers/image-spec
- **OCI Runtime Specification**: https://github.com/opencontainers/runtime-spec
- **OCI Distribution Specification**: https://github.com/opencontainers/distribution-spec
- **runc**: https://github.com/opencontainers/runc
- **containerd**: https://github.com/containerd/containerd
- **containerd Client Documentation**: https://pkg.go.dev/github.com/containerd/containerd
- **CRI API**: https://github.com/kubernetes/cri-api
- **CRI-O**: https://github.com/cri-o/cri-o
- **Kata Containers**: https://github.com/kata-containers/kata-containers
- **gVisor**: https://github.com/google/gvisor
- **Rootless Containers**: https://rootlesscontaine.rs
- **youki**: https://github.com/containers/youki
- **crun**: https://github.com/containers/crun
- **Podman**: https://github.com/containers/podman
- **BuildKit**: https://github.com/moby/buildkit
