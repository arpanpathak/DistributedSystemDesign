# Chapter 31: Supply Chain Security — Image Signing, SBOM, Policies

## Introduction

The software supply chain is the entire pipeline from source code to running
container: compiling code, pulling dependencies, building images, pushing to
registries, and deploying to clusters. Each step is an attack surface. A
compromised dependency, a tampered image, or a malicious base layer can give
an attacker code execution inside your cluster — with whatever permissions the
Pod's ServiceAccount provides.

Supply chain attacks are not theoretical. SolarWinds (2020) demonstrated that
compromising a build pipeline can breach thousands of organizations. The
Codecov bash uploader attack (2021) showed that a single tampered CI script
can exfiltrate secrets from every CI/CD pipeline that uses it. The Log4Shell
vulnerability (2021) showed that a single library can expose millions of
applications.

This chapter covers the tools and practices that protect the supply chain:
signing images to prove provenance, generating SBOMs to know what's inside,
scanning for vulnerabilities before deployment, enforcing admission policies
that reject unsigned or vulnerable images, and building minimal images that
reduce the attack surface.

**What you will learn:**

- Software supply chain threat model
- Image signing with Cosign (Sigstore)
- SBOM generation with Syft and Trivy
- Vulnerability scanning with Trivy, Grype, and Snyk
- Admission policies with Kyverno, OPA/Gatekeeper, and Sigstore Policy Controller
- Distroless and scratch-based images
- Multi-stage Docker builds for Go applications
- SLSA framework for provenance
- Go programs for image verification

---

## 31.1 Software Supply Chain Threats

### 31.1.1 The Attack Surface

```text
Source Code          Build          Registry         Cluster
┌──────────┐     ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Developer │────▶│ CI/CD    │───▶│ Container│───▶│ Kubelet  │
│ Workstat. │     │ Pipeline │    │ Registry │    │ pulls &  │
│           │     │          │    │          │    │ runs     │
└──────────┘     └──────────┘    └──────────┘    └──────────┘
     ▲                ▲               ▲               ▲
     │                │               │               │
 ┌───┴────┐     ┌────┴─────┐   ┌────┴─────┐   ┌────┴─────┐
 │Threats: │     │Threats:  │   │Threats:  │   │Threats:  │
 │- Compro-│     │- Tampered│   │- Image   │   │- Deploy  │
 │  mised  │     │  build   │   │  replace-│   │  untrust-│
 │  deps   │     │  scripts │   │  ment    │   │  ed      │
 │- Typo-  │     │- Injected│   │- Tag     │   │  images  │
 │  squatt-│     │  malware │   │  mutation│   │- No vuln │
 │  ing    │     │- Secret  │   │- Registry│   │  scanning│
 │- Malici-│     │  leakage │   │  comprom-│   │- Over-   │
 │  ous    │     │- Comprom-│   │  ise     │   │  privile-│
 │  commit │     │  ised CI │   │          │   │  ged pods│
 └─────────┘     └──────────┘   └──────────┘   └──────────┘
```

### 31.1.2 Common Attack Vectors

**Dependency Confusion:**
An attacker publishes a malicious package with the same name as an internal
package to a public registry. If the build system checks the public registry
first (or at all), it pulls the malicious version.

**Typosquatting:**
Packages named `lodsah` instead of `lodash`, `reqeusts` instead of `requests`.
A developer's typo in `go.mod` or `package.json` pulls malicious code.

**Compromised CI/CD:**
If an attacker gains access to the CI/CD pipeline (through a leaked token, a
compromised GitHub Action, or a tampered build script), they can inject arbitrary
code into every build.

**Image Tag Mutation:**
Container image tags are mutable. An attacker who compromises a registry can
replace `myapp:v1.0` with a malicious image. Anyone pulling by tag gets the
attacker's version. This is why **image digests** matter.

**Base Image Poisoning:**
If the base image (`FROM ubuntu:22.04`) is compromised at the registry level,
every image built on top of it includes the malicious code.

### 31.1.3 Defense-in-Depth Strategy

```text
┌─────────────────────────────────────────────────────────┐
│ Layer 1: Secure the Source                               │
│   - Dependency scanning (Dependabot, Renovate)           │
│   - Signed commits                                       │
│   - Branch protection, code review                       │
├─────────────────────────────────────────────────────────┤
│ Layer 2: Secure the Build                                │
│   - Hermetic builds (SLSA Level 3+)                      │
│   - Pinned dependencies (checksums, lockfiles)           │
│   - Build provenance attestation                         │
├─────────────────────────────────────────────────────────┤
│ Layer 3: Secure the Artifacts                            │
│   - Image signing (Cosign)                               │
│   - SBOM generation                                      │
│   - Vulnerability scanning                               │
├─────────────────────────────────────────────────────────┤
│ Layer 4: Secure the Deployment                           │
│   - Admission policies (require signatures)              │
│   - Image digest pinning                                 │
│   - Runtime monitoring                                   │
└─────────────────────────────────────────────────────────┘
```

---

## 31.2 Container Image Signing with Cosign

Cosign is part of the Sigstore project, which provides free, open-source tools
for signing, verifying, and protecting software.

### 31.2.1 How Cosign Works

```text
┌──────────┐     cosign sign     ┌──────────────┐
│  Image   │────────────────────▶│   Registry   │
│  Digest  │                     │              │
│ sha256:  │     ┌──────────┐   │  image:tag   │
│ abc123.. │◀───│ Signing   │   │  + signature │
│          │     │ Key       │   │    (OCI tag) │
└──────────┘     └──────────┘   └──────────────┘

Signing stores the signature as an OCI artifact alongside
the image in the registry (using a predictable tag derived
from the image digest).
```

### 31.2.2 Keyless Signing (Recommended)

Cosign's "keyless" signing uses short-lived certificates from the Sigstore
Certificate Authority (Fulcio), verified by a transparency log (Rekor):

```text
┌──────────┐  1. OIDC Login   ┌──────────┐
│ Developer │────────────────▶│  Identity │
│           │                  │  Provider │
│           │◀── 2. ID Token ─│  (GitHub, │
└─────┬─────┘                  │   Google) │
      │                        └──────────┘
      │ 3. Request cert
      ▼
┌──────────┐  4. Short-lived  ┌──────────┐
│  Fulcio  │     certificate  │  Rekor   │
│  (CA)    │─────────────────▶│  (Trans- │
│          │                   │  parency │
└──────────┘                   │  Log)    │
                               └──────────┘
```

1. Developer authenticates with an OIDC provider (GitHub, Google, Microsoft)
2. Fulcio issues a short-lived certificate (valid ~10 minutes)
3. The signature is recorded in Rekor (transparency log)
4. The certificate expires, but the Rekor entry proves the signature was made
   while the certificate was valid

**Advantage:** No key management. No secrets to store or rotate. The signing
identity is tied to the developer's or CI system's OIDC identity.

### 31.2.3 Installation

```bash
# Install Cosign
# macOS
brew install cosign

# Linux
COSIGN_VERSION=2.2.4
curl -fsSL https://github.com/sigstore/cosign/releases/download/v${COSIGN_VERSION}/cosign-linux-amd64 -o /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign

# Verify the installation
cosign version
```

### 31.2.4 Hands-On: Signing an Image with Cosign

#### Key-based Signing

```bash
# Step 1: Generate a key pair
cosign generate-key-pair
# Creates cosign.key (private, password-protected) and cosign.pub (public)

# Step 2: Build and push an image
docker build -t ghcr.io/myorg/myapp:v1.0 .
docker push ghcr.io/myorg/myapp:v1.0

# Step 3: Sign the image
# The signature is pushed to the registry alongside the image
cosign sign --key cosign.key ghcr.io/myorg/myapp:v1.0
# Enter password for cosign.key

# Step 4: Verify the signature
cosign verify --key cosign.pub ghcr.io/myorg/myapp:v1.0
# Output:
# Verification for ghcr.io/myorg/myapp:v1.0 --
# The following checks were performed on each of these signatures:
#   - The cosign claims were validated
#   - The signatures were verified against the specified public key

# Step 5: Sign with annotations (metadata)
cosign sign --key cosign.key \
  -a "git-sha=$(git rev-parse HEAD)" \
  -a "build-id=${BUILD_ID}" \
  -a "built-by=ci-pipeline" \
  ghcr.io/myorg/myapp:v1.0
```

#### Keyless Signing (CI/CD)

```bash
# In GitHub Actions, keyless signing is automatic
# COSIGN_EXPERIMENTAL=1 enables keyless mode (now default in v2)
cosign sign ghcr.io/myorg/myapp@sha256:abc123...

# In CI, the OIDC token from the CI provider is used automatically
# GitHub Actions provides ACTIONS_ID_TOKEN_REQUEST_URL and
# ACTIONS_ID_TOKEN_REQUEST_TOKEN environment variables

# Verify keyless signatures
cosign verify \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  --certificate-identity-regexp="^https://github.com/myorg/myapp/" \
  ghcr.io/myorg/myapp:v1.0
```

#### GitHub Actions Workflow

```yaml
name: Build, Sign, and Push
on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write  # Required for keyless signing

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: sigstore/cosign-installer@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push
        id: build
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Sign the image (keyless)
        run: |
          cosign sign --yes \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}

      - name: Attest SBOM
        run: |
          syft ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }} \
            -o spdx-json > sbom.spdx.json
          cosign attest --yes --predicate sbom.spdx.json \
            --type spdxjson \
            ghcr.io/${{ github.repository }}@${{ steps.build.outputs.digest }}
```

---

## 31.3 SBOM — Software Bill of Materials

An SBOM lists every component in your software: libraries, frameworks, OS packages,
and their versions. It's essential for:

- **Vulnerability response:** When a new CVE is published, you can instantly check
  if any of your images contain the affected package
- **License compliance:** Know which licenses your dependencies use
- **Supply chain transparency:** Understand exactly what's in your images

### 31.3.1 SBOM Formats

| Format | Description |
|--------|-------------|
| **SPDX** | Linux Foundation standard, ISO/IEC 5962:2021 |
| **CycloneDX** | OWASP standard, focused on security use cases |
| **Syft JSON** | Anchore's native format (rich detail) |

### 31.3.2 Generating SBOMs with Syft

```bash
# Install Syft
brew install syft
# Or
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin

# Generate SBOM from a container image
syft ghcr.io/myorg/myapp:v1.0 -o spdx-json > sbom.spdx.json
syft ghcr.io/myorg/myapp:v1.0 -o cyclonedx-json > sbom.cdx.json
syft ghcr.io/myorg/myapp:v1.0 -o syft-json > sbom.syft.json

# Generate SBOM from a directory (source code)
syft dir:. -o spdx-json > source-sbom.spdx.json

# Generate SBOM from a Dockerfile
syft docker:Dockerfile -o spdx-json > build-sbom.spdx.json

# Human-readable table output
syft ghcr.io/myorg/myapp:v1.0 -o table
```

Example Syft table output:

```
NAME                    VERSION        TYPE
alpine-baselayout       3.4.3-r1       apk
alpine-baselayout-data  3.4.3-r1       apk
alpine-keys             2.4-r1         apk
apk-tools               2.14.0-r2      apk
busybox                 1.36.1-r2      apk
busybox-binsh           1.36.1-r2      apk
ca-certificates-bundle  20230506-r0    apk
libc-utils              0.7.2-r5       apk
libcrypto3              3.1.4-r0       apk
libssl3                 3.1.4-r0       apk
musl                    1.2.4_git20... apk
musl-utils              1.2.4_git20... apk
scanelf                 1.3.7-r1       apk
ssl_client              1.36.1-r2      apk
zlib                    1.3-r2         apk
```

### 31.3.3 Hands-On: Generating SBOM with Syft

```bash
# Step 1: Build a sample Go application image
cat > Dockerfile <<EOF
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /app/server ./cmd/server

FROM alpine:3.19
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/server /usr/local/bin/server
ENTRYPOINT ["/usr/local/bin/server"]
EOF

docker build -t myapp:latest .

# Step 2: Generate SBOM in multiple formats
syft myapp:latest -o spdx-json > sbom-spdx.json
syft myapp:latest -o cyclonedx-json > sbom-cdx.json

# Step 3: Inspect the SBOM
cat sbom-spdx.json | jq '.packages | length'
# Shows the total number of packages

cat sbom-spdx.json | jq '.packages[] | select(.name == "openssl") | {name, version: .versionInfo}'
# Check specific package versions

# Step 4: Attach the SBOM to the image as an attestation
cosign attest --yes --predicate sbom-spdx.json \
  --type spdxjson \
  ghcr.io/myorg/myapp:v1.0

# Step 5: Verify and retrieve the SBOM attestation
cosign verify-attestation \
  --type spdxjson \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  --certificate-identity-regexp="^https://github.com/myorg/" \
  ghcr.io/myorg/myapp:v1.0
```

### 31.3.4 Generating SBOMs with Trivy

```bash
# Trivy can also generate SBOMs
trivy image --format spdx-json --output sbom.spdx.json myapp:latest
trivy image --format cyclonedx --output sbom.cdx.json myapp:latest

# Generate SBOM from filesystem
trivy fs --format spdx-json --output source-sbom.json .
```

---

## 31.4 Vulnerability Scanning

### 31.4.1 Trivy — Comprehensive Scanner

Trivy is the most popular open-source scanner. It detects vulnerabilities in:
OS packages, language-specific packages (Go, npm, pip, etc.), IaC misconfigurations,
secrets, and licenses.

```bash
# Install Trivy
brew install trivy

# Scan a container image
trivy image myapp:latest

# Scan with severity filter
trivy image --severity HIGH,CRITICAL myapp:latest

# Scan and fail CI if critical vulnerabilities found
trivy image --exit-code 1 --severity CRITICAL myapp:latest

# Scan with SBOM as input
trivy sbom sbom.spdx.json

# Scan a filesystem (source code dependencies)
trivy fs --scanners vuln .

# Scan Kubernetes manifests for misconfigurations
trivy config ./k8s-manifests/

# Scan a running cluster
trivy k8s --report summary cluster
```

Example Trivy output:

```
myapp:latest (alpine 3.19.0)
=============================
Total: 3 (HIGH: 2, CRITICAL: 1)

┌──────────────┬────────────────┬──────────┬────────┬───────────────┬──────────────────────┐
│   Library    │ Vulnerability  │ Severity │ Status │   Installed   │    Fixed Version     │
├──────────────┼────────────────┼──────────┼────────┼───────────────┼──────────────────────┤
│ libcrypto3   │ CVE-2024-0727  │ CRITICAL │ fixed  │ 3.1.4-r0      │ 3.1.4-r3             │
│ libssl3      │ CVE-2024-0727  │ CRITICAL │ fixed  │ 3.1.4-r0      │ 3.1.4-r3             │
│ zlib         │ CVE-2023-45853 │ HIGH     │ fixed  │ 1.3-r2        │ 1.3.1-r0             │
└──────────────┴────────────────┴──────────┴────────┴───────────────┴──────────────────────┘
```

### 31.4.2 Grype — Anchore's Vulnerability Scanner

```bash
# Install Grype
brew install grype

# Scan a container image
grype myapp:latest

# Scan with SBOM input (faster — no need to re-catalog)
syft myapp:latest -o syft-json > sbom.json
grype sbom:sbom.json

# Fail on severity threshold
grype myapp:latest --fail-on critical

# Output in JSON for CI integration
grype myapp:latest -o json > vulnerabilities.json
```

### 31.4.3 Scanning in CI/CD

```yaml
# GitHub Actions vulnerability scanning
name: Security Scan
on:
  pull_request:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Trivy vulnerability scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: myapp:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Upload Trivy scan results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

---

## 31.5 Image Admission Policies

Admission policies ensure that only trusted, verified images are deployed to
the cluster. They run as admission webhooks that intercept Pod creation.

### 31.5.1 Kyverno Image Verification

Kyverno is a Kubernetes-native policy engine that uses CRDs for policies:

```bash
# Install Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno -n kyverno --create-namespace
```

```yaml
# Policy: Require all images to be signed with Cosign
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce  # Block non-compliant Pods
  background: true
  rules:
    - name: verify-cosign-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"
          attestors:
            - entries:
                - keyless:
                    issuer: "https://token.actions.githubusercontent.com"
                    subject: "https://github.com/myorg/*"
                    rekor:
                      url: "https://rekor.sigstore.dev"
---
# Policy: Require specific key-based signature
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-key-signed-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-key-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "registry.example.com/*"
          attestors:
            - entries:
                - keys:
                    publicKeys: |-
                      -----BEGIN PUBLIC KEY-----
                      MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
                      -----END PUBLIC KEY-----
---
# Policy: Require SBOM attestation
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-sbom-attestation
spec:
  validationFailureAction: Enforce
  rules:
    - name: verify-sbom-exists
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"
          attestations:
            - type: "https://spdx.dev/Document"
              attestors:
                - entries:
                    - keyless:
                        issuer: "https://token.actions.githubusercontent.com"
                        subject: "https://github.com/myorg/*"
              conditions:
                - all:
                    # Verify the SBOM is not empty
                    - key: "{{ packages }}"
                      operator: GreaterThan
                      value: 0
---
# Policy: Require image digests (no mutable tags)
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-digest
spec:
  validationFailureAction: Enforce
  rules:
    - name: require-digest
      match:
        any:
          - resources:
              kinds:
                - Pod
      validate:
        message: "Images must use digests, not tags"
        pattern:
          spec:
            containers:
              - image: "*@sha256:*"
            initContainers:
              - image: "*@sha256:*"
```

### 31.5.2 OPA/Gatekeeper Image Policies

```bash
# Install Gatekeeper
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper gatekeeper/gatekeeper -n gatekeeper-system --create-namespace
```

```yaml
# ConstraintTemplate for allowed registries
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedregistries
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRegistries
      validation:
        openAPIV3Schema:
          type: object
          properties:
            registries:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedregistries

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not registry_allowed(container.image)
          msg := sprintf(
            "Image '%v' is from an untrusted registry. Allowed: %v",
            [container.image, input.parameters.registries]
          )
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          not registry_allowed(container.image)
          msg := sprintf(
            "Init container image '%v' is from an untrusted registry",
            [container.image]
          )
        }

        registry_allowed(image) {
          registry := input.parameters.registries[_]
          startswith(image, registry)
        }
---
# Constraint: Only allow images from our registries
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRegistries
metadata:
  name: allowed-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
  parameters:
    registries:
      - "ghcr.io/myorg/"
      - "registry.example.com/"
      - "gcr.io/distroless/"
```

### 31.5.3 Sigstore Policy Controller

The Sigstore Policy Controller directly integrates Cosign verification into
Kubernetes admission:

```bash
# Install Policy Controller
helm repo add sigstore https://sigstore.github.io/helm-charts
helm install policy-controller sigstore/policy-controller \
  -n cosign-system --create-namespace
```

```yaml
# ClusterImagePolicy — verify all images from our org
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: verify-myorg-images
spec:
  images:
    - glob: "ghcr.io/myorg/**"
  authorities:
    - keyless:
        url: https://fulcio.sigstore.dev
        identities:
          - issuer: https://token.actions.githubusercontent.com
            subjectRegExp: "^https://github.com/myorg/.*$"
      ctlog:
        url: https://rekor.sigstore.dev
    - key:
        data: |
          -----BEGIN PUBLIC KEY-----
          MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE...
          -----END PUBLIC KEY-----
```

### 31.5.4 Hands-On: Admission Policy to Require Signed Images

```bash
# Step 1: Install Kyverno
helm install kyverno kyverno/kyverno -n kyverno --create-namespace

# Step 2: Create the verification policy
cat <<EOF | kubectl apply -f -
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-images
  annotations:
    policies.kyverno.io/title: Verify Image Signatures
    policies.kyverno.io/description: >-
      Requires all container images from ghcr.io/myorg to be
      signed with Cosign using keyless signing from GitHub Actions.
spec:
  validationFailureAction: Enforce
  webhookTimeoutSeconds: 30
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"
          attestors:
            - entries:
                - keyless:
                    issuer: "https://token.actions.githubusercontent.com"
                    subject: "https://github.com/myorg/*"
EOF

# Step 3: Test with an unsigned image (should be REJECTED)
kubectl run unsigned-test --image=ghcr.io/myorg/myapp:unsigned
# Error: admission webhook denied the request:
# resource Pod/default/unsigned-test was blocked due to the
# following policies: verify-images

# Step 4: Test with a signed image (should be ALLOWED)
kubectl run signed-test --image=ghcr.io/myorg/myapp:v1.0
# pod/signed-test created

# Step 5: View policy reports
kubectl get policyreport -A
kubectl get clusterpolicyreport
```

---

## 31.6 Distroless Images — Minimal Attack Surface

Distroless images contain only the application and its runtime dependencies.
No package manager, no shell, no utilities — nothing an attacker could use.

### 31.6.1 Why Distroless?

```
Traditional Image (Ubuntu-based):
┌─────────────────────────────────────┐
│ Shell (bash, sh)                    │ ← Attacker gets a shell
│ Package Manager (apt, apk)          │ ← Attacker installs tools
│ System Utilities (curl, wget, etc.) │ ← Attacker exfiltrates data
│ Many OS Libraries                   │ ← Large vulnerability surface
│ Your Application                    │
│ Total: 200-500 MB, 100+ CVEs       │
└─────────────────────────────────────┘

Distroless Image:
┌─────────────────────────────────────┐
│ Minimal C Library (glibc or musl)   │
│ CA Certificates                     │
│ Timezone Data                       │
│ Your Application                    │
│ Total: 20-50 MB, 0-5 CVEs          │
└─────────────────────────────────────┘

Scratch-based (Go/Rust static binary):
┌─────────────────────────────────────┐
│ Your Application (static binary)    │
│ Total: 5-20 MB, 0 CVEs             │
└─────────────────────────────────────┘
```

### 31.6.2 Available Distroless Images

| Image | Use Case |
|-------|----------|
| `gcr.io/distroless/static-debian12` | Static binaries (Go, Rust) |
| `gcr.io/distroless/base-debian12` | Dynamically-linked binaries |
| `gcr.io/distroless/cc-debian12` | C/C++ with libstdc++ |
| `gcr.io/distroless/java21-debian12` | Java 21 applications |
| `gcr.io/distroless/python3-debian12` | Python 3 applications |
| `gcr.io/distroless/nodejs22-debian12` | Node.js 22 applications |

### 31.6.3 Debugging Distroless Containers

Since distroless images have no shell, debugging requires special techniques:

```bash
# Use kubectl debug to attach an ephemeral debug container
kubectl debug -it pod/myapp --image=busybox --target=myapp

# Or use the "debug" variant of distroless
# gcr.io/distroless/static-debian12:debug includes a busybox shell
# NEVER use debug variants in production
```

---

## 31.7 Multi-Stage Docker Builds

### 31.7.1 Why Multi-Stage Builds?

Single-stage builds include build tools, compilers, and source code in the
final image. This is wasteful and insecure:

```dockerfile
# BAD: Single-stage build
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o /app/server
CMD ["/app/server"]
# Image size: ~1.2 GB (includes Go compiler, build tools, source code)
# Contains: build tools (gcc, make), Go source, test files
```

Multi-stage builds separate the build environment from the runtime environment:

```dockerfile
# GOOD: Multi-stage build
# Stage 1: Build
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /app/server

# Stage 2: Runtime (minimal)
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]
# Image size: ~10 MB (just the static binary + minimal OS)
```

### 31.7.2 Hands-On: Multi-Stage Dockerfile for a Go Application

This is a production-grade Dockerfile with every security best practice:

```dockerfile
# ==============================================================================
# Stage 1: Build the Go binary
# ==============================================================================
# Use a specific digest, not a mutable tag, for reproducible builds.
# The alpine variant is smaller than the standard golang image.
FROM golang:1.22-alpine@sha256:abc123... AS builder

# Install certificates and timezone data at build time.
# These will be copied to the final image.
RUN apk add --no-cache ca-certificates tzdata

# Create a non-root user for the final image.
# We create it here because the final image has no user management tools.
RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid 65534 \
    appuser

WORKDIR /build

# Copy dependency files first for better layer caching.
# go.mod and go.sum change less frequently than source code,
# so this layer is cached across most builds.
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Copy the source code
COPY . .

# Build the binary with all security and size optimizations:
#   CGO_ENABLED=0  — Static binary, no C library dependency
#   -trimpath      — Remove file system paths from binary (security)
#   -ldflags:
#     -s           — Strip symbol table (smaller binary)
#     -w           — Strip DWARF debug info (smaller binary)
#     -extldflags "-static" — Force static linking
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -trimpath \
    -ldflags="-s -w -extldflags '-static' \
      -X main.version=$(git describe --tags --always 2>/dev/null || echo dev) \
      -X main.buildTime=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    -o /build/server \
    ./cmd/server

# ==============================================================================
# Stage 2: Create the minimal runtime image
# ==============================================================================
# scratch is a completely empty image — the absolute minimum.
# For Go static binaries, this is ideal.
FROM scratch

# Import CA certificates for HTTPS connections
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Import timezone data for time.LoadLocation()
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# Import the non-root user and group files
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /etc/group /etc/group

# Copy the compiled binary
COPY --from=builder /build/server /server

# Run as non-root user
USER appuser:appuser

# Expose the application port
EXPOSE 8080

# Set health check path (used by Kubernetes probes)
# Note: scratch images can't run shell commands for HEALTHCHECK,
# so we rely on Kubernetes liveness/readiness probes instead.

# Use the binary directly — no shell needed
ENTRYPOINT ["/server"]
```

Build and verify:

```bash
# Build the image
docker build -t myapp:scratch .

# Check the image size
docker images myapp:scratch
# REPOSITORY   TAG       IMAGE ID       SIZE
# myapp        scratch   abc123...      8.2MB

# Scan for vulnerabilities
trivy image myapp:scratch
# Total: 0 (no OS packages = no OS vulnerabilities!)

# Try to shell into it (will fail — no shell)
docker run -it myapp:scratch /bin/sh
# exec: "/bin/sh": stat /bin/sh: no such file or directory

# Generate SBOM
syft myapp:scratch -o table
# Only shows the Go binary and its compiled-in dependencies
```

---

## 31.8 Image Provenance — SLSA Framework

SLSA (Supply-chain Levels for Software Artifacts, pronounced "salsa") is a
framework for ensuring the integrity of software artifacts throughout the
supply chain.

### 31.8.1 SLSA Levels

| Level | Requirements | Protects Against |
|-------|-------------|------------------|
| **Level 0** | No guarantees | Nothing |
| **Level 1** | Build process documented, provenance generated | Mistakes |
| **Level 2** | Hosted build service, authenticated provenance | Tampering after build |
| **Level 3** | Hardened builds, non-falsifiable provenance | Tampering during build |
| **Level 4** | Hermetic, reproducible builds, two-person review | Insider threats |

### 31.8.2 SLSA Provenance with GitHub Actions

GitHub Actions has native SLSA provenance generation:

```yaml
name: SLSA Build
on:
  push:
    tags: ['v*']

permissions:
  contents: read
  packages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.ref_name }}

  provenance:
    needs: build
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v2.0.0
    with:
      image: ghcr.io/${{ github.repository }}
      digest: ${{ needs.build.outputs.digest }}
    secrets:
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

### 31.8.3 Verifying Provenance

```bash
# Verify SLSA provenance
slsa-verifier verify-image \
  ghcr.io/myorg/myapp:v1.0@sha256:abc123... \
  --source-uri github.com/myorg/myapp \
  --source-tag v1.0

# Cosign can also verify in-toto attestations
cosign verify-attestation \
  --type slsaprovenance \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  --certificate-identity-regexp="^https://github.com/slsa-framework/slsa-github-generator/" \
  ghcr.io/myorg/myapp:v1.0
```

---

## 31.9 Hands-On: Go Program That Verifies Image Signatures

```go
// Package main demonstrates programmatic verification of container
// image signatures using the Cosign Go library. This is the building
// block for creating custom admission controllers, CI/CD gates, or
// audit tools that verify image integrity before deployment.
//
// The program verifies:
//   1. Key-based signatures — using a known public key
//   2. Keyless signatures — using OIDC identity and Sigstore
//   3. Attestations — verifying SBOM or provenance attestations
//
// In production, this logic would run inside a Kubernetes admission
// webhook, a CI/CD pipeline step, or a periodic audit job.
package main

import (
	"context"
	"crypto"
	"encoding/json"
	"fmt"
	"log"
	"os"

	"github.com/google/go-containerregistry/pkg/name"
	"github.com/sigstore/cosign/v2/cmd/cosign/cli/fulcio"
	"github.com/sigstore/cosign/v2/cmd/cosign/cli/rekor"
	"github.com/sigstore/cosign/v2/pkg/cosign"
	"github.com/sigstore/cosign/v2/pkg/oci"
	sigs "github.com/sigstore/cosign/v2/pkg/signature"
	"github.com/sigstore/sigstore/pkg/cryptoutils"
)

// ImageVerifier provides methods for verifying container image
// signatures and attestations. It encapsulates the Cosign
// verification logic and provides a clean interface for
// different verification patterns.
type ImageVerifier struct {
	// rekorClient connects to the Rekor transparency log.
	// Rekor records all signing events, enabling after-the-fact
	// audit of who signed what and when.
	rekorClient *rekor.Client

	// fulcioRoots contains the trusted root certificates for
	// the Fulcio Certificate Authority. These are used to verify
	// the signing certificates in keyless flows.
	fulcioRoots *fulcio.TrustedRoot
}

// NewImageVerifier creates a verifier configured to use the
// public Sigstore infrastructure (Fulcio, Rekor, TUF).
// For air-gapped environments, you would configure custom
// roots and a private Rekor instance.
func NewImageVerifier() (*ImageVerifier, error) {
	return &ImageVerifier{}, nil
}

// VerificationResult contains the outcome of an image
// verification check, including the signer identity,
// verification method, and any attestations found.
type VerificationResult struct {
	// Verified indicates whether the image passed verification.
	Verified bool `json:"verified"`

	// ImageRef is the full image reference that was verified.
	// Includes the digest for unambiguous identification.
	ImageRef string `json:"imageRef"`

	// SignerIdentity contains information about who signed
	// the image. For keyless signing, this includes the OIDC
	// issuer and subject (e.g., GitHub Actions workflow URL).
	SignerIdentity string `json:"signerIdentity,omitempty"`

	// SigningMethod describes how the image was signed
	// (key-based, keyless, etc.).
	SigningMethod string `json:"signingMethod"`

	// Signatures is the number of valid signatures found.
	Signatures int `json:"signatures"`

	// Error contains the verification error message, if any.
	Error string `json:"error,omitempty"`
}

// VerifyWithKey verifies an image signature using a known
// public key. This is the simplest verification mode — you
// distribute your public key to all verification points and
// they check that the image was signed with the corresponding
// private key.
//
// The publicKeyPath should point to a PEM-encoded public key
// file (e.g., cosign.pub generated by "cosign generate-key-pair").
func (v *ImageVerifier) VerifyWithKey(
	ctx context.Context,
	imageRef string,
	publicKeyPath string,
) (*VerificationResult, error) {
	result := &VerificationResult{
		ImageRef:      imageRef,
		SigningMethod: "key-based",
	}

	// Parse the image reference. This resolves tags to digests
	// and validates the reference format.
	ref, err := name.ParseReference(imageRef)
	if err != nil {
		result.Error = fmt.Sprintf("invalid image reference: %v", err)
		return result, err
	}

	// Load the public key from the file. The key must be a
	// PEM-encoded ECDSA, RSA, or ED25519 public key.
	pubKeyBytes, err := os.ReadFile(publicKeyPath)
	if err != nil {
		result.Error = fmt.Sprintf("failed to read public key: %v", err)
		return result, err
	}

	// Parse the PEM-encoded public key into a crypto.PublicKey.
	pubKey, err := cryptoutils.UnmarshalPEMToPublicKey(pubKeyBytes)
	if err != nil {
		result.Error = fmt.Sprintf("failed to parse public key: %v", err)
		return result, err
	}

	// Create a Cosign verifier from the public key. The verifier
	// knows how to check ECDSA/RSA/ED25519 signatures.
	verifier, err := sigs.LoadPublicKeyVerifier(pubKey, crypto.SHA256)
	if err != nil {
		result.Error = fmt.Sprintf("failed to create verifier: %v", err)
		return result, err
	}

	// Configure the verification options. These control what
	// checks are performed beyond basic signature verification.
	checkOpts := &cosign.CheckOpts{
		// SigVerifier uses the provided public key to verify
		// the signature bytes against the image digest.
		SigVerifier: verifier,

		// IgnoreTlog skips Rekor transparency log verification.
		// For key-based signing without Rekor, set this to true.
		// For maximum security, leave it false and ensure
		// signatures were recorded in Rekor.
		IgnoreTlog: true,

		// IgnoreSCT skips Signed Certificate Timestamp verification.
		// Only relevant for keyless signing.
		IgnoreSCT: true,
	}

	// Perform the verification. This:
	// 1. Fetches the signature OCI artifact from the registry
	// 2. Verifies the signature against the image digest
	// 3. Optionally checks the Rekor transparency log
	signatures, bundleVerified, err := cosign.VerifyImageSignatures(
		ctx, ref, checkOpts,
	)
	if err != nil {
		result.Error = fmt.Sprintf("verification failed: %v", err)
		return result, nil // Return result with error, not Go error
	}

	result.Verified = true
	result.Signatures = len(signatures)

	if bundleVerified {
		log.Printf("Bundle verification: PASSED (Rekor entry confirmed)")
	}

	// Extract signer information from the first signature.
	if len(signatures) > 0 {
		payload, _ := signatures[0].Payload()
		result.SignerIdentity = fmt.Sprintf("key-based (%d bytes payload)",
			len(payload))
	}

	return result, nil
}

// VerifyKeyless verifies an image using Sigstore's keyless
// signing infrastructure. Instead of a static public key, it
// verifies that:
//   1. The image was signed with a certificate from Fulcio
//   2. The certificate was issued to the expected OIDC identity
//   3. The signing event is recorded in Rekor
//
// This is the recommended verification method for images signed
// in CI/CD pipelines (e.g., GitHub Actions).
//
// Parameters:
//   - imageRef: Full image reference (e.g., "ghcr.io/org/app@sha256:...")
//   - issuer: Expected OIDC issuer (e.g., "https://token.actions.githubusercontent.com")
//   - subjectRegexp: Regexp matching the expected certificate subject
//     (e.g., "^https://github.com/myorg/myapp/.*$")
func (v *ImageVerifier) VerifyKeyless(
	ctx context.Context,
	imageRef string,
	issuer string,
	subjectRegexp string,
) (*VerificationResult, error) {
	result := &VerificationResult{
		ImageRef:      imageRef,
		SigningMethod: "keyless (Sigstore)",
	}

	ref, err := name.ParseReference(imageRef)
	if err != nil {
		result.Error = fmt.Sprintf("invalid image reference: %v", err)
		return result, err
	}

	// For keyless verification, we need the Fulcio root
	// certificates and Rekor client. These are fetched from
	// the Sigstore TUF repository, which provides a secure
	// distribution channel for the trust root.
	rootCerts, err := fulcio.GetRoots()
	if err != nil {
		result.Error = fmt.Sprintf("failed to get Fulcio roots: %v", err)
		return result, err
	}

	// Configure keyless verification options.
	checkOpts := &cosign.CheckOpts{
		// RootCerts are the trusted Fulcio CA certificates.
		// The signing certificate must chain to one of these roots.
		RootCerts: rootCerts,

		// Identities specifies the expected OIDC identity of the
		// signer. Both issuer and subject must match for the
		// verification to succeed.
		Identities: []cosign.Identity{
			{
				Issuer:        issuer,
				SubjectRegExp: subjectRegexp,
			},
		},

		// RekorClient verifies that the signing event was recorded
		// in the Rekor transparency log. This provides:
		//   - Non-repudiation: the signer cannot deny signing
		//   - Discoverability: all signing events are public
		//   - Consistency: the log is append-only
		// Leave nil for testing; set for production.
	}

	signatures, _, err := cosign.VerifyImageSignatures(ctx, ref, checkOpts)
	if err != nil {
		result.Error = fmt.Sprintf("keyless verification failed: %v", err)
		return result, nil
	}

	result.Verified = true
	result.Signatures = len(signatures)

	// Extract the signer identity from the certificate.
	if len(signatures) > 0 {
		cert, _ := signatures[0].Cert()
		if cert != nil {
			result.SignerIdentity = fmt.Sprintf(
				"issuer=%s, subject=%s",
				cert.Issuer.CommonName,
				cert.Subject.CommonName,
			)
		}
	}

	return result, nil
}

// VerifyAttestation verifies that an image has a specific type
// of attestation (e.g., SBOM, SLSA provenance, vulnerability scan).
// Attestations are signed statements about an image that provide
// additional metadata beyond the image itself.
//
// The predicateType specifies what kind of attestation to look for:
//   - "https://spdx.dev/Document" — SPDX SBOM
//   - "https://cyclonedx.org/bom" — CycloneDX SBOM
//   - "https://slsa.dev/provenance/v1" — SLSA provenance
func (v *ImageVerifier) VerifyAttestation(
	ctx context.Context,
	imageRef string,
	publicKeyPath string,
	predicateType string,
) (bool, []oci.Signature, error) {
	ref, err := name.ParseReference(imageRef)
	if err != nil {
		return false, nil, fmt.Errorf("invalid image reference: %w", err)
	}

	pubKeyBytes, err := os.ReadFile(publicKeyPath)
	if err != nil {
		return false, nil, fmt.Errorf("failed to read public key: %w", err)
	}

	pubKey, err := cryptoutils.UnmarshalPEMToPublicKey(pubKeyBytes)
	if err != nil {
		return false, nil, fmt.Errorf("failed to parse public key: %w", err)
	}

	verifier, err := sigs.LoadPublicKeyVerifier(pubKey, crypto.SHA256)
	if err != nil {
		return false, nil, fmt.Errorf("failed to create verifier: %w", err)
	}

	checkOpts := &cosign.CheckOpts{
		SigVerifier: verifier,
		IgnoreTlog:  true,
		IgnoreSCT:   true,
	}

	// VerifyImageAttestations checks for in-toto attestations
	// associated with the image. These are DSSE-signed envelopes
	// containing a predicate (the attestation content).
	attestations, _, err := cosign.VerifyImageAttestations(
		ctx, ref, checkOpts,
	)
	if err != nil {
		return false, nil, fmt.Errorf("attestation verification failed: %w", err)
	}

	// Filter attestations by predicate type.
	var matching []oci.Signature
	for _, att := range attestations {
		payload, err := att.Payload()
		if err != nil {
			continue
		}

		// Parse the DSSE envelope to extract the predicate type.
		var envelope struct {
			PayloadType string `json:"payloadType"`
			Payload     string `json:"payload"`
		}
		if err := json.Unmarshal(payload, &envelope); err != nil {
			continue
		}

		if envelope.PayloadType == predicateType {
			matching = append(matching, att)
		}
	}

	return len(matching) > 0, matching, nil
}

func main() {
	if len(os.Args) < 2 {
		fmt.Fprintf(os.Stderr, "Usage: %s <image-ref> [public-key-path]\n", os.Args[0])
		fmt.Fprintf(os.Stderr, "\nExamples:\n")
		fmt.Fprintf(os.Stderr, "  %s ghcr.io/myorg/myapp:v1.0 cosign.pub\n", os.Args[0])
		fmt.Fprintf(os.Stderr, "  %s ghcr.io/myorg/myapp@sha256:abc123...\n", os.Args[0])
		os.Exit(1)
	}

	imageRef := os.Args[1]
	ctx := context.Background()

	verifier, err := NewImageVerifier()
	if err != nil {
		log.Fatalf("Failed to create verifier: %v", err)
	}

	// If a public key is provided, verify with key-based signing.
	if len(os.Args) >= 3 {
		publicKeyPath := os.Args[2]
		fmt.Printf("Verifying %s with key %s...\n\n", imageRef, publicKeyPath)

		result, err := verifier.VerifyWithKey(ctx, imageRef, publicKeyPath)
		if err != nil {
			log.Fatalf("Verification error: %v", err)
		}

		printResult(result)

		// Also check for SBOM attestation if key is provided.
		fmt.Println("\nChecking for SBOM attestation...")
		hasSBOM, _, err := verifier.VerifyAttestation(
			ctx, imageRef, publicKeyPath, "https://spdx.dev/Document",
		)
		if err != nil {
			fmt.Printf("  SBOM attestation check failed: %v\n", err)
		} else if hasSBOM {
			fmt.Println("  ✓ SBOM attestation found and verified")
		} else {
			fmt.Println("  ✗ No SBOM attestation found")
		}
	} else {
		// No key provided — try keyless verification.
		// This requires the image to be signed with Sigstore keyless
		// signing (e.g., from GitHub Actions).
		fmt.Printf("Verifying %s with keyless (Sigstore)...\n\n", imageRef)

		result, err := verifier.VerifyKeyless(
			ctx,
			imageRef,
			"https://token.actions.githubusercontent.com",
			".*", // Accept any subject for demo
		)
		if err != nil {
			log.Fatalf("Verification error: %v", err)
		}

		printResult(result)
	}
}

// printResult formats and displays the verification result
// in a human-readable format with clear pass/fail indicators.
func printResult(result *VerificationResult) {
	resultJSON, _ := json.MarshalIndent(result, "", "  ")
	fmt.Println(string(resultJSON))

	fmt.Println()
	if result.Verified {
		fmt.Printf("✓ VERIFIED — %d valid signature(s) found\n",
			result.Signatures)
		if result.SignerIdentity != "" {
			fmt.Printf("  Signer: %s\n", result.SignerIdentity)
		}
	} else {
		fmt.Printf("✗ VERIFICATION FAILED\n")
		if result.Error != "" {
			fmt.Printf("  Error: %s\n", result.Error)
		}
	}
}
```

---

## 31.10 Putting It All Together — A Complete Supply Chain Pipeline

Here's a comprehensive CI/CD pipeline that implements all the supply chain
security practices covered in this chapter:

```yaml
name: Secure Supply Chain Pipeline
on:
  push:
    branches: [main]
    tags: ['v*']

permissions:
  contents: read
  packages: write
  id-token: write
  security-events: write

env:
  IMAGE: ghcr.io/${{ github.repository }}

jobs:
  # Step 1: Lint and test
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: '1.22'
      - run: go test ./... -race -coverprofile=coverage.out
      - run: go vet ./...

  # Step 2: Build minimal image
  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/build-push-action@v5
        id: build
        with:
          push: true
          tags: ${{ env.IMAGE }}:${{ github.sha }}
          # Use the multi-stage Dockerfile from section 31.7.2

  # Step 3: Vulnerability scan
  scan:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE }}@${{ needs.build.outputs.digest }}
          format: sarif
          output: trivy.sarif
          severity: CRITICAL,HIGH
          exit-code: '1'

      - uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy.sarif

  # Step 4: Generate SBOM
  sbom:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: anchore/sbom-action@v0
        with:
          image: ${{ env.IMAGE }}@${{ needs.build.outputs.digest }}
          format: spdx-json
          output-file: sbom.spdx.json

      - uses: actions/upload-artifact@v4
        with:
          name: sbom
          path: sbom.spdx.json

  # Step 5: Sign the image and attest SBOM
  sign:
    needs: [build, scan, sbom]
    runs-on: ubuntu-latest
    steps:
      - uses: sigstore/cosign-installer@v3

      - uses: actions/download-artifact@v4
        with:
          name: sbom

      - name: Sign the image (keyless)
        run: |
          cosign sign --yes \
            ${{ env.IMAGE }}@${{ needs.build.outputs.digest }}

      - name: Attest SBOM
        run: |
          cosign attest --yes \
            --predicate sbom.spdx.json \
            --type spdxjson \
            ${{ env.IMAGE }}@${{ needs.build.outputs.digest }}

  # Step 6: Generate SLSA provenance
  provenance:
    needs: [build, sign]
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_container_slsa3.yml@v2.0.0
    with:
      image: ghcr.io/${{ github.repository }}
      digest: ${{ needs.build.outputs.digest }}
    secrets:
      registry-username: ${{ github.actor }}
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

---

## 31.11 Security Best Practices Summary

### 31.11.1 Image Security Checklist

| Practice | Priority | Tool |
|----------|----------|------|
| Use image digests, not tags | **Critical** | Kyverno policy |
| Sign all images | **Critical** | Cosign |
| Scan for vulnerabilities | **Critical** | Trivy, Grype |
| Generate SBOMs | High | Syft, Trivy |
| Use distroless/scratch base images | High | Dockerfile |
| Multi-stage builds | High | Dockerfile |
| Enforce admission policies | High | Kyverno, Gatekeeper |
| Pin base image digests | High | Dockerfile |
| Generate provenance | Medium | SLSA, Cosign |
| Run as non-root | **Critical** | Dockerfile, SecurityContext |

### 31.11.2 What NOT to Do

1. **Never** use `latest` tag in production — it's mutable and unpredictable
2. **Never** run containers as root unless absolutely necessary
3. **Never** use `privileged: true` without extreme justification
4. **Never** skip vulnerability scanning "because we're behind schedule"
5. **Never** trust a public image without verification
6. **Never** include build tools in production images
7. **Never** store secrets in image layers (they persist even if "deleted")
8. **Never** disable admission policies "temporarily" in production

---

## 31.12 Summary

Supply chain security is not a single tool or technique — it's a comprehensive
approach that secures every step from source code to running container:

1. **Image signing** (Cosign) proves that an image was built by a trusted entity
   and hasn't been tampered with
2. **SBOMs** (Syft, Trivy) provide visibility into what's inside your images,
   enabling rapid vulnerability response
3. **Vulnerability scanning** (Trivy, Grype) catches known vulnerabilities before
   they reach production
4. **Admission policies** (Kyverno, Gatekeeper, Sigstore Policy Controller) enforce
   that only trusted images run in the cluster
5. **Minimal images** (distroless, scratch) reduce the attack surface by removing
   unnecessary tools and packages
6. **Provenance** (SLSA) provides cryptographic proof of how an artifact was built

The Go program in this chapter demonstrates that image verification is not magic —
it's standard cryptographic signature verification that can be integrated into any
tool or service. Whether you build it into an admission webhook, a CI/CD gate, or
a periodic audit job, the fundamental operation is the same: verify the signature,
check the identity, trust the artifact.

Supply chain security is an ongoing process, not a one-time setup. New
vulnerabilities are discovered daily, signing keys can be compromised, and
attack techniques evolve. The tools and practices in this chapter give you the
foundation to build a resilient supply chain that adapts to these challenges.

---

## Exercises

1. **End-to-End Signing:** Build a Go application, create a multi-stage Dockerfile
   with a scratch base, push to a registry, sign with Cosign, and verify the
   signature programmatically using the Go library.

2. **SBOM Analysis:** Generate SBOMs for the same application using both Syft and
   Trivy. Compare the results — do they find the same packages? Use the SBOM to
   check for a specific CVE.

3. **Admission Policy:** Deploy Kyverno and create a policy that requires all images
   in the `production` namespace to be signed. Test with both signed and unsigned
   images. Add an exception for system namespaces.

4. **Image Size Comparison:** Build the same Go application with four different
   base images: Ubuntu, Alpine, distroless, and scratch. Compare image sizes,
   vulnerability counts, and startup times.

5. **Supply Chain Attack Simulation:** In a test environment, create a "malicious"
   image with the same tag as a legitimate one. Deploy admission policies that
   would catch this attack (digest pinning, signature verification). Document
   which policies catch the attack and which don't.

6. **Custom Admission Webhook:** Using the Go verification code from section 31.9,
   build a Kubernetes admission webhook that verifies Cosign signatures on all
   Pod creation requests. Deploy it and test with signed/unsigned images.
