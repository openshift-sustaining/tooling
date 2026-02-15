# Go Compiler Version Bump: Impact Analysis for OpenShift Builder

## 1. Problem Statement

### Background

The OpenShift Builder (`github.com/openshift/builder`) is the platform component responsible for executing customer-defined BuildConfig operations — Dockerfile builds, Source-to-Image (S2I) builds, and Custom builds. It runs as a pod within the cluster and performs image pulls, builds, and pushes on behalf of end users.

The builder binary is compiled with varying Go versions across OCP releases 4.12 through 4.21. Several CVEs in Go standard library and dependency code require a newer Go compiler to resolve, necessitating a bump of the Go compiler version across all maintained release branches.

**The core concern**: Changing the Go compiler version changes the behavior of the Go standard library that the builder and all its dependencies are compiled against. Since the builder executes customer workloads, any behavioral change could impact end-user build operations across the entire cluster.

### Current State

| OCP Release | BuildConfig Go | OCP Suggestion (ART Gate) | Buildah Version | Buildah Go |
|-------------|---------------|---------------------------|-----------------|------------|
| 4.12 | 1.19 | 1.19 | v1.26.9 | 1.16 |
| 4.13 | 1.19 | 1.19 | v1.29.5 | 1.17 |
| 4.14 | 1.19 | 1.20 | v1.33.12 | 1.20 |
| 4.15 | 1.19 | 1.20 | v1.33.12 | 1.20 |
| 4.16 | 1.21 | 1.21 | v1.33.12 | 1.20 |
| 4.17 | 1.22.0 | 1.22 | v1.37.7 | 1.22.0 |
| 4.18 | 1.22.0 | 1.22 | v1.37.7 | 1.22.0 |
| 4.19 | 1.22.8 | 1.23 | v1.39.7 | 1.22.8 |
| 4.20 | 1.22.8 | 1.24 | v1.39.7 | 1.22.8 |
| 4.21 | 1.22.8 | 1.25 | v1.39.7 | 1.22.8 |

### Future State (After Go Bump)

| OCP Release | BuildConfig Go | OCP Suggestion (ART Gate) | Buildah Version | Buildah Go |
|-------------|---------------|---------------------------|-----------------|------------|
| 4.12 | **1.22.z** | 1.22 | v1.26.z | **1.22** |
| 4.13 | **1.22.z** | 1.22 | v1.29.z | **1.22** |
| 4.14 | **1.22.z** | 1.22 | v1.33.z | **1.22** |
| 4.15 | **1.22.z** | 1.22 | v1.33.z | **1.22** |
| 4.16 | **1.22.z** | 1.22 | v1.33.z | **1.22** |
| 4.17 | 1.22.z | 1.22 | v1.37.z | 1.22 |
| 4.18 | 1.22.z | 1.22 | v1.37.z | 1.22 |
| 4.19 | 1.22.z | 1.23 | v1.39.z | 1.22 |
| 4.20 | 1.22.z | 1.24 | v1.39.z | 1.22 |
| 4.21 | 1.22.z | 1.25 | v1.39.z | 1.22 |

**Key observations:**
- **4.17–4.21 require NO meaningful Go upgrade** — 4.17-4.18 are already at Go 1.22.0 (patch-level z-stream is not a behavioral change), and 4.19-4.21 are already at Go 1.22.8 with ART gates even higher (1.23–1.25).
- **4.12–4.16 are the ONLY branches that require actual Go version upgrades.** These are the focus of this analysis.
- **Buildah is also bumped** — importantly, buildah's own Go version changes too (e.g., v1.26.9 goes from Go 1.16 → 1.22, a 6 minor version jump). Since the builder imports buildah as a Go library, this is part of the same binary compilation.

### Builder's Key Dependencies

The builder imports a significant number of libraries. When the Go compiler changes, ALL of these are recompiled with the new compiler, and their behavior may change based on Go standard library defaults.

| Dependency | Role in Builder | Version (4.12) | Version (4.19) |
|-----------|----------------|----------------|----------------|
| `containers/buildah` | Dockerfile build execution (imported as Go library, not binary) | v1.26.9 | v1.39.7 |
| `containers/image` | ALL registry TLS connections, image pull/push, auth | v5.22.0 | v5.34.3 |
| `containers/storage` | Local image/layer filesystem storage | v1.42.0 | v1.57.2 |
| `containers/common` | Shared container config, pull policies | v0.49.1 | v0.62.3 |
| `openshift/source-to-image` | S2I build strategy execution | v1.3.2 | v1.4.0 |
| `openshift/imagebuilder` | Dockerfile parsing and instruction processing | v1.2.4 | v1.2.15 |
| `openshift/library-go` | Git client, shared OpenShift utilities | v0.0.0-... | v0.0.0-... |
| `fsouza/go-dockerclient` | Docker API client (used for S2I image inspection) | v1.7.11 | v1.12.0 |
| `k8s.io/client-go` | Kubernetes API communication (build status updates) | v0.25.2 | v0.30.2 |
| `k8s.io/kubernetes` | Credential provider for registry auth (`credentialprovider` package) | v1.25.2 | v1.28.2 |
| `golang.org/x/crypto` | Supplemental crypto primitives (SSH, bcrypt, etc.) | **PINNED to March 2020** | v0.29.0 |
| `golang.org/x/net` | HTTP/2, net utilities | v0.17.0 (replace) | v0.33.0 |

### Notable `replace` Directives in 4.12

Both 4.12 and 4.13 go.mod files contain `replace` directives that pin `golang.org/x/crypto` to old versions:

**Builder 4.12:**
```
golang.org/x/crypto => golang.org/x/crypto v0.0.0-20200323165209-0ec3e9974c59  (March 2020, Go 1.14 era)
golang.org/x/net => golang.org/x/net v0.17.0
github.com/docker/docker => github.com/docker/docker v0.0.0-20200911110540-7ca355652fe0
github.com/containerd/containerd => github.com/containerd/containerd v1.6.6
```

**Builder 4.13:**
```
golang.org/x/crypto => golang.org/x/crypto v0.0.0-20220919173607-35f4265a4bc0  (September 2022, Go 1.19 era)
```

The 4.12 pin to a March 2020 `x/crypto` is the highest compilation risk — this is from the Go 1.14 era. The 4.13 pin to September 2022 is less risky but still needs verification. While `x/crypto` provides supplemental cryptographic primitives (not the TLS stack — that's in Go's standard library `crypto/tls`), these old pins may cause compilation compatibility issues with Go 1.22. See section 3.7.

---

## 2. Proposed Plan

### Target State

Bump the Go compiler version to **Go 1.22.z** for both the OpenShift Builder and Buildah across release branches 4.12 through 4.16. Branches 4.17–4.21 are already at Go 1.22.x and require no meaningful changes.

### What Changes and What Does Not

| Aspect | Changes? | Details |
|--------|----------|---------|
| Go compiler version (Builder) | **YES** (4.12–4.16 only) | Bumped to Go 1.22.z. 4.17+ already at 1.22.x — no change. |
| Go compiler version (Buildah) | **YES** (4.12–4.16 only) | Bumped to Go 1.22.z (buildah currently lags behind builder on some branches). 4.17+ already at 1.22.x. |
| go.mod `go` directive | **Likely YES** | Should match compiler version per Go conventions |
| Builder source code | NO | No code changes |
| Buildah source code | NO | No code changes |
| Dependency versions (go.mod `require`) | NO | All dependency versions stay the same |
| `replace` directives | NO | Existing pins remain |

### Branch-by-Branch Go Version Jump

| OCP Release | Builder Go Jump | Buildah Go Jump | Scope |
|-------------|----------------|-----------------|-------|
| **4.12** | 1.19 → 1.22.z | **1.16 → 1.22** (6 minor) | **Largest jump — highest scrutiny** |
| **4.13** | 1.19 → 1.22.z | **1.17 → 1.22** (5 minor) | Large jump |
| **4.14** | 1.19 → 1.22.z | 1.20 → 1.22 (2 minor) | Moderate jump |
| **4.15** | 1.19 → 1.22.z | 1.20 → 1.22 (2 minor) | Moderate jump |
| **4.16** | 1.21 → 1.22.z | 1.20 → 1.22 (2 minor) | Small jump |
| **4.17** | Already 1.22.0 | Already 1.22.0 | **No change needed** |
| **4.18** | Already 1.22.0 | Already 1.22.0 | **No change needed** |
| **4.19** | Already 1.22.8 | Already 1.22.8 | **No change needed** |
| **4.20** | Already 1.22.8 | Already 1.22.8 | **No change needed** |
| **4.21** | Already 1.22.8 | Already 1.22.8 | **No change needed** |

**The real work is concentrated on 4.12–4.16.** Branches 4.17–4.21 are already at Go 1.22.x and require no action.

**The buildah Go version jump for 4.12 (Go 1.16 → 1.22) is the single largest jump** in this plan. Since the builder imports buildah as a Go library, buildah's code is compiled as part of the builder binary, making this relevant to the builder's behavior.

---

## 3. Impact Analysis

### 3.1 Builder Architecture and Network Communication Paths

To assess the Go bump's impact, we need to understand every point where the builder communicates over the network, as these are the paths most affected by Go standard library changes (TLS, HTTP, x509).

```
OpenShift Builder Binary
│
├─ BUILD EXECUTION ─────────────────────────────────────────────────
│  ├── daemonless.go ──→ buildah (Go library)
│  │     ├── Pull image  ──→ containers/image ──→ Registry (TLS)
│  │     ├── Build image ──→ imagebuildah.BuildDockerfiles()
│  │     └── Push image  ──→ containers/image ──→ Registry (TLS)
│  │
│  ├── sti.go ──→ openshift/source-to-image (Go library)
│  │     ├── Pull builder image ──→ fsouza/go-dockerclient
│  │     └── Script download    ──→ net/http (proxy-aware)
│  │
│  └── docker.go ──→ DockerClient interface (routes to daemonless)
│
├─ SOURCE HANDLING ─────────────────────────────────────────────────
│  ├── source.go ──→ git clone (shells out to `git` binary, NOT Go net/http)
│  ├── source.go ──→ bsdtar extraction (shells out to `bsdtar` binary)
│  └── source.go ──→ extractSourceFromImage ──→ containers/image (TLS)
│
├─ AUTHENTICATION ──────────────────────────────────────────────────
│  ├── cmd/dockercfg/ ──→ reads .dockercfg/.dockerconfigjson files
│  │     └── Uses k8s.io/kubernetes/pkg/credentialprovider
│  └── daemonless.go  ──→ containers/image/pkg/docker/config
│
├─ CERTIFICATE TRUST ───────────────────────────────────────────────
│  ├── transient_mounts.go ──→ Mounts /etc/pki/ca-trust into build container
│  ├── transient_mounts.go ──→ Mounts RHSM certs and entitlements
│  └── containers/image/pkg/tlsclientconfig ──→ Loads certs from
│        /etc/containers/certs.d/ and /etc/docker/certs.d/
│
└─ KUBERNETES API ──────────────────────────────────────────────────
   └── HandleBuildStatusUpdate() ──→ k8s.io/client-go ──→ API Server (TLS)
```

### 3.2 Impact Path 1: Registry TLS Connections (containers/image)

**This is the highest-impact path.** Every image pull, push, and inspection goes through `containers/image`, which creates the `tls.Config` used for registry connections. The builder itself contains **zero** explicit TLS configuration — no `tls.Config`, no `MinVersion`, no `GODEBUG` settings.

#### How TLS is Configured (Source-Code Verified)

We inspected `containers/image/docker/docker_client.go` across all relevant versions. The TLS configuration is created in the `newDockerClient()` function.

**containers/image v5.22.0** (used in builder 4.12–4.13):

```go
// docker/docker_client.go
func serverDefault() *tls.Config {
    return &tls.Config{
        MinVersion:   tls.VersionTLS10,  // EXPLICIT: allows TLS 1.0
        CipherSuites: tlsconfig.DefaultServerAcceptedCiphers,  // EXPLICIT
    }
}

func newDockerClient(...) {
    tlsClientConfig := serverDefault()  // ← uses explicit config
    tlsclientconfig.SetupCertificates(certDir, tlsClientConfig)
    ...
}
```

**containers/image v5.29.4** (used in builder 4.14–4.16):

```go
// docker/docker_client.go
func newDockerClient(...) {
    tlsClientConfig := &tls.Config{
        CipherSuites: tlsconfig.DefaultServerAcceptedCiphers,  // EXPLICIT
        // MinVersion NOT SET — relies on Go default (TLS 1.2 since Go 1.18)
    }
    ...
}
```

**containers/image v5.32.2** (used in builder 4.17–4.18): Same as v5.29.4.

**containers/image v5.34.3** (used in builder 4.19–4.21):

```go
// docker/docker_client.go — already compiled with Go 1.22
func newDockerClient(...) {
    tlsClientConfig := &tls.Config{
        CipherSuites: tlsconfig.ClientDefault().CipherSuites,  // EXPLICIT
    }
    ...
}
```

#### Why the Go Bump Has NO Impact on Registry TLS

1. **CipherSuites is explicitly set in ALL versions.** Go 1.22 deprecated RSA key exchange cipher suites by default, but this only affects the default cipher list when `CipherSuites` is `nil`. Since `containers/image` always explicitly populates `CipherSuites`, Go's deprecation does not apply.

2. **MinVersion is explicitly set in v5.22.0 (4.12–4.13).** Go 1.18 changed the default MinVersion from TLS 1.0 to TLS 1.2, but `v5.22.0` explicitly sets `MinVersion: tls.VersionTLS10`, overriding Go's default. Even after bumping to Go 1.22, TLS 1.0 connections remain supported on 4.12–4.13.

3. **MinVersion already relies on Go's TLS 1.2 default in v5.29.4+ (4.14+).** Since these branches already compile with Go 1.19 (where the default was already TLS 1.2), bumping to Go 1.22 doesn't change the effective minimum.

| c/image Version | Builder Releases | MinVersion | CipherSuites | Go Bump TLS Impact |
|----------------|-----------------|------------|--------------|-------------------|
| v5.22.0 | 4.12–4.13 | **Explicit TLS 1.0** | Explicit | **NONE** |
| v5.29.4 | 4.14–4.16 | Go default (TLS 1.2) | Explicit | **NONE** |
| v5.32.2 | 4.17–4.18 | Go default (TLS 1.2) | Explicit | **NONE** |
| v5.34.3 | 4.19–4.21 | Go default (TLS 1.2) | Explicit | **NONE** |

#### Certificate Loading (containers/image/pkg/tlsclientconfig)

The `SetupCertificates()` function loads CA certificates (`.crt`), client certificates (`.cert`), and private keys (`.key`) from per-registry directories. This code path uses `x509.SystemCertPool()` (or `tlsconfig.SystemCertPool()` in older versions) and `tls.LoadX509KeyPair()` — both are stable standard library APIs unaffected by Go version changes.

The builder also mounts host CA trust via `appendCATrustMount()` (binds `/etc/pki/ca-trust` into the build container) and RHSM subscription certs via `appendRHSMMount()`. These are filesystem mounts — no Go version sensitivity.

### 3.3 Impact Path 2: Kubernetes API Communication (k8s.io/client-go)

The builder communicates with the Kubernetes API server to update build status via `HandleBuildStatusUpdate()` in `common.go`. This uses `k8s.io/client-go`, which has its own TLS configuration derived from the service account token and kubeconfig.

**Impact**: `k8s.io/client-go` configures its own `tls.Config` based on the cluster CA bundle and service account credentials. It explicitly sets TLS parameters. The Go version bump does not change how client-go establishes TLS connections to the API server.

### 3.4 Impact Path 3: Source-to-Image (openshift/source-to-image)

For S2I builds, the builder uses `openshift/source-to-image` as a Go library. Key operations:

- **Builder image pull**: Handled via `fsouza/go-dockerclient`, which connects to a local Docker/CRI socket (unix socket, not TLS).
- **S2I script download**: If scripts are fetched via HTTP(S), this uses Go's `net/http` directly. The S2I library's go.mod says `go 1.18`, so its own module behavior stays at Go 1.18 semantics.
- **Image inspection**: Uses `fsouza/go-dockerclient` to inspect images via the local container runtime socket.

**Impact**: S2I script download over HTTPS could theoretically be affected by TLS default changes. However, Go's TLS 1.2 client default was already in effect under Go 1.19 (changed in Go 1.18). The S2I library's own go.mod says `go 1.18`, but GODEBUG defaults are controlled by the main module (builder), not the dependency. Since the effective GODEBUG level is already `go 1.20`, no additional changes apply.

**Note**: The `fsouza/go-dockerclient` connects to the local container runtime via Unix domain socket — no TLS involved. The CRI-O client in the builder (`crioclient/crioclient.go`) similarly uses a Unix socket with `http.Transport.Dial` (deprecated field — should use `DialContext`). This deprecated field still functions in Go 1.22.

### 3.5 Impact Path 4: Git Clone and Source Extraction

- **Git clone**: The builder shells out to the `git` binary via `exec.Command`. The git binary uses its own TLS stack (OpenSSL/GnuTLS on RHEL), completely independent of Go's `crypto/tls`. **No Go version impact.**
- **Archive extraction**: Uses `exec.Command("bsdtar", ...)` — external binary, not Go's `archive/tar`. **No Go version impact.**
- **File copying**: Uses `exec.Command("cp", ...)` — external binary. **No Go version impact.**
- **Image source extraction**: Uses `buildah.NewBuilder()` with `types.SystemContext` → goes through `containers/image` → covered in section 3.2.

### 3.6 Impact Path 5: Docker Auth and Credential Handling

The builder reads Docker credentials from the filesystem via `cmd/dockercfg/cfg.go`:
- Reads `.dockercfg`, `.dockerconfigjson`, `config.json` files
- Uses `k8s.io/kubernetes/pkg/credentialprovider` for credential keyring lookup
- Passes credentials to `containers/image` via `config.SetAuthentication()`

This is pure file I/O and JSON parsing — no network or TLS operations. **No Go version impact.**

### 3.7 Compilation Compatibility: x/crypto Pin in 4.12 and 4.13

Both 4.12 and 4.13 pin `golang.org/x/crypto` to old versions via `replace` directives:

| Branch | `replace` Pin | Pin Date | Era |
|--------|--------------|----------|-----|
| **4.12** | `v0.0.0-20200323165209-0ec3e9974c59` | **March 2020** | Go 1.14 |
| **4.13** | `v0.0.0-20220919173607-35f4265a4bc0` | **September 2022** | Go 1.19 |
| 4.14+ | No replace — uses `v0.19.0` directly | February 2024 | Go 1.21+ |

**Risk assessment**:
- `golang.org/x/crypto` provides supplemental crypto primitives (SSH, bcrypt, chacha20poly1305, curve25519, etc.) — NOT the TLS stack (that's `crypto/tls` in the standard library).
- Go maintains backward compatibility for `x/` packages, but x/crypto versions interact with internal Go compiler/runtime APIs. The March 2020 pin (4.12) is the highest risk — spanning 4 years and 8 Go minor versions. The September 2022 pin (4.13) is lower risk but still 2+ years old.
- **Recommendation**: Verify that `go build` succeeds with Go 1.22 on BOTH the 4.12 and 4.13 branches WITHOUT any code changes. If compilation fails due to the `x/crypto` pin, the `replace` directive will need to be updated to a newer `x/crypto` version.

### 3.8 Go Standard Library Behavioral Changes

These changes apply to the builder binary as a whole — all code in the builder and its dependencies compiled together.

#### 3.8.1 GODEBUG Compatibility Mechanism

Go 1.21+ uses the `go` directive in the **main module's** go.mod to set GODEBUG defaults. Go versions below 1.20 in go.mod are mapped to `go 1.20` for GODEBUG purposes.

| Builder `go` directive | Effective GODEBUG | Notes |
|------------------------|-------------------|-------|
| Current: `go 1.19` | Maps to **`go 1.20`** | Most Go 1.20 defaults already active |
| Future: `go 1.22` | **`go 1.22`** | Adds go1.21 + go1.22 defaults |

This means the **effective GODEBUG jump is go1.20 → go1.22**, not go1.19 → go1.22. Many impactful changes are already active.

#### 3.8.2 Changes Already Active Under Current Configuration

These changes are already in effect because the current `go 1.19` directive maps to GODEBUG `go 1.20`:

| Change | Introduced In | Status | Builder Relevance |
|--------|--------------|--------|-------------------|
| SHA-1 x509 certificate rejection | Go 1.18 | **Already active** (`x509sha1=0`) | Customers using SHA-1 signed certs for private registries would already be failing |
| x509 Common Name fallback disabled | Go 1.15–1.17 | **Already active** | Certs must use SAN, not CN — already enforced |
| `math/rand` auto-seeding | Go 1.20 | **Already active** (`randautoseed=1`) | Builder's `randomBuildTag()` already auto-seeded |
| `os/exec` PATH security | Go 1.19 | **Already active** | No current-dir binary resolution |
| TLS 1.0/1.1 client default | Go 1.18 | **Overridden** by c/image explicit config | See section 3.2 |
| `;` rejected in URL query params | Go 1.17 | **Already active** | Already in effect under Go 1.19 |
| `archive/tar` insecure paths | Go 1.20 | **Not applicable** | Builder uses `bsdtar` binary, not Go's `archive/tar` |

#### 3.8.3 New GODEBUG Defaults Activated by Bumping to `go 1.22`

Only these GODEBUG settings change when moving from effective `go 1.20` to `go 1.22`:

| GODEBUG Setting | New Default | What It Does | Builder Impact |
|-----------------|-------------|-------------|----------------|
| `httplaxcontentlength` | `0` (strict) | Rejects invalid Content-Length headers in HTTP responses | **LOW** — builder is an HTTP client; malformed Content-Length from registries is very rare |
| `httpmuxgo121` | `0` (new) | New ServeMux routing patterns | **NONE** — builder does not serve HTTP |
| `tlsrsakex` | `0` (disabled) | Disables RSA key exchange by default | **NONE** — c/image explicitly sets CipherSuites |
| `tls10server` | `0` (TLS 1.2) | TLS 1.2 minimum for servers | **NONE** — builder is a client, not a server |
| `panicnil` | `0` (error) | `panic(nil)` now produces `runtime.PanicNilError` | **LOW** — unlikely pattern in builder or deps |
| `multipathtcp` | `0` | MPTCP support | **NONE** |
| `winsymlink` | `0` | Windows symlink handling | **NONE** — Linux only |
| `zipinsecurepath` | `0` (reject) | Rejects insecure paths in ZIP files | **NONE** — builder doesn't use Go's `archive/zip` |

#### 3.8.4 For-Loop Variable Semantics (Go 1.22)

Go 1.22 changes loop variable capture to per-iteration scope. This is controlled by each module's `go` directive, NOT the main module's:

| Module | go.mod `go` directive | Gets New Semantics? | Notes |
|--------|----------------------|---------------------|-------|
| **openshift/builder** (main module) | 1.22 (after bump) | **YES** — builder's own code | Audited: no affected patterns found |
| containers/buildah v1.26.9 | 1.16 | No — unless go.mod is also bumped | If buildah's go.mod `go` directive is bumped to 1.22, YES |
| containers/buildah v1.33.12 | 1.20 | No — unless go.mod is also bumped | Same caveat |
| containers/image v5.22.0 | 1.17 | No | Dependency module, not main |
| containers/image v5.29.4 | 1.19 | No | Dependency module, not main |
| openshift/source-to-image v1.3.2 | 1.18 | No | Dependency module, not main |
| k8s.io/client-go v0.25.2 | ~1.19 | No | Dependency module, not main |

**Important caveat for buildah**: If the buildah repo's go.mod `go` directive is also bumped to `1.22` (as implied by the "Buildah Go" column in the plan), then buildah's own code DOES get per-iteration loop semantics. Buildah has goroutines inside for-loops in `add.go` (lines 392, 398, 492, 527), but all are synchronized with `wg.Wait()` before the next iteration, so behavior is identical under old and new semantics.

**Source code audit**: Manual review of all builder source files (`daemonless.go`, `docker.go`, `source.go`, `sti.go`, `common.go`, `dockerutil.go`, `util.go`, `transient_mounts.go`) found **no closure-captured loop variables** that would change behavior. Buildah's goroutine-in-loop patterns in `add.go` were also verified safe.

#### 3.8.5 Deprecated APIs Still Used

| API | Deprecated Since | Used In | Impact |
|-----|-----------------|---------|--------|
| `ioutil.ReadFile` | Go 1.16 | `common.go`, `source.go` | **NONE** — still functional, wraps `os.ReadFile` |
| `ioutil.TempFile` | Go 1.16 | `daemonless.go` | **NONE** — still functional, wraps `os.CreateTemp` |
| `ioutil.TempDir` | Go 1.16 | `daemonless.go` | **NONE** — still functional, wraps `os.MkdirTemp` |
| `ioutil.WriteFile` | Go 1.16 | `source.go` | **NONE** — still functional, wraps `os.WriteFile` |
| `net.Dialer.DualStack` | Go 1.12 | c/image v5.22.0 `tlsclientconfig.go` | **NONE** — field ignored since Go 1.12 |

All deprecated APIs remain fully functional in Go 1.22. No behavioral changes.

#### 3.8.6 Crypto Performance

Go 1.20 introduced constant-time implementations for RSA and ECDSA, which are slightly slower:
- RSA operations: 15–45% slower
- ECDSA operations: 5–30% slower

These would already be active under the current GODEBUG mapping (`go 1.20`). **No additional change from bumping to Go 1.22.**

---

## 4. Risk Assessment Summary

### Per-Release Risk Matrix

| OCP | Builder Go Jump | Buildah Go Jump | GODEBUG Jump | TLS Impact | x/crypto Risk | Overall |
|-----|----------------|-----------------|-------------|------------|---------------|---------|
| **4.12** | 1.19→1.22 | **1.16→1.22** | go1.20→go1.22 | NONE (explicit MinVersion + CipherSuites) | **Compile check needed** (2020 pin) | **LOW** |
| **4.13** | 1.19→1.22 | **1.17→1.22** | go1.20→go1.22 | NONE (explicit MinVersion + CipherSuites) | **Compile check needed** (Sep 2022 pin) | **LOW** |
| **4.14** | 1.19→1.22 | 1.20→1.22 | go1.20→go1.22 | NONE (explicit CipherSuites, TLS 1.2 default) | None (v0.19.0) | **LOW** |
| **4.15** | 1.19→1.22 | 1.20→1.22 | go1.20→go1.22 | NONE (same as 4.14) | None (v0.19.0) | **LOW** |
| **4.16** | 1.21→1.22 | 1.20→1.22 | go1.21→go1.22 | NONE (same as 4.14) | None (v0.19.0) | **MINIMAL** |
| **4.17–18** | **no change** (already 1.22.0) | **no change** | None | NONE | None | **NONE** |
| **4.19–21** | **no change** (already 1.22.8) | **no change** | None | NONE | None | **NONE** |

**Note**: 4.17–4.21 are already at Go 1.22.x. No meaningful Go upgrade is needed for these branches. The analysis below focuses on 4.12–4.16.

### Why the Risk is Lower Than Initially Expected

1. **TLS is explicitly configured in the dependency that handles ALL registry connections.** The `containers/image` library always explicitly sets `CipherSuites` and (in v5.22.0) `MinVersion`. Go's default TLS changes do not propagate through explicit configurations.

2. **GODEBUG is already at the `go 1.20` level.** The current `go 1.19` directive maps to `go 1.20` for GODEBUG. Most impactful stdlib changes (SHA-1 cert rejection, rand seeding, PATH security, TLS 1.0 client deprecation) are already active in the currently-shipping builder.

3. **The builder delegates sensitive operations to external binaries.** Git clone → `git` binary (uses OS TLS, not Go). Archive extraction → `bsdtar` binary. File copy → `cp` binary. These are not affected by Go version changes.

4. **No loop variable capture bugs found.** Manual audit of all builder source files found zero closure-captured loop variables affected by Go 1.22's per-iteration change.

5. **The builder itself has no TLS code.** Zero `tls.Config`, zero `MinVersion`, zero `GODEBUG` settings. Everything flows through dependencies which explicitly configure their own TLS.

6. **Kubernetes API communication is independently configured.** `k8s.io/client-go` sets up its own TLS based on kubeconfig/service-account — not affected by Go default changes.

---

## 5. Suggestions and Guidelines

### 5.1 Pre-Implementation Checklist

Before submitting the Go bump, verify the following:

**For ALL branches (4.12–4.16):**
- [ ] `go build` succeeds with Go 1.22.z without source code changes
- [ ] `go vet` passes without new issues
- [ ] Existing unit tests pass
- [ ] CI pipeline produces a valid builder image

**For 4.12 specifically:**
- [ ] Verify compilation with the pinned `golang.org/x/crypto v0.0.0-20200323165209` (March 2020) — highest compilation risk; if it fails, the `replace` directive needs updating
- [ ] Verify compilation with the pinned `github.com/docker/docker v0.0.0-20200911110540` — very old version, potential compatibility issues
- [ ] Verify compilation with the pinned `github.com/containerd/containerd v1.6.6`

**For 4.13 specifically:**
- [ ] Verify compilation with the pinned `golang.org/x/crypto v0.0.0-20220919173607` (September 2022) — lower risk than 4.12 but still needs verification

### 5.2 Implementation Order

Start with the lowest-risk branches and work backward. Branches 4.17–4.21 require no action (already at Go 1.22.x).

1. **Phase 1**: 4.16 — Builder: Go 1.21 → 1.22 (1 minor). Buildah: Go 1.20 → 1.22 (2 minor). Smallest jump. Low risk.
2. **Phase 2**: 4.14–4.15 — Builder: Go 1.19 → 1.22 (3 minor). Buildah: Go 1.20 → 1.22 (2 minor). Modern dependencies (c/image v5.29.4, x/crypto v0.19.0). Low risk.
3. **Phase 3**: 4.13 — Builder: Go 1.19 → 1.22 (3 minor). **Buildah: Go 1.17 → 1.22 (5 minor)**. Older dependencies. Needs compilation verification.
4. **Phase 4**: 4.12 — Builder: Go 1.19 → 1.22 (3 minor). **Buildah: Go 1.16 → 1.22 (6 minor — the largest jump in the entire plan)**. Oldest dependencies, pinned x/crypto from March 2020. Requires thorough compilation and runtime verification.

### 5.3 go.mod `go` Directive Decision

**Recommended: Bump the `go` directive to `1.22`.**

- Matches the compiler, which is the Go convention
- Activates Go 1.22 GODEBUG defaults (all analyzed as safe for the builder)
- Enables per-iteration loop variable semantics (no affected patterns found in builder code)
- Future issues can be mitigated with explicit `//go:debug` directives without reverting the compiler

**Alternative: Keep the `go` directive at the current value (e.g., `1.19`).**

- Creates a compiler/directive mismatch (unusual, may cause tooling warnings)
- Preserves GODEBUG at the `go 1.20` mapping level
- Only gains the compiler's security fixes without adopting any new stdlib defaults

### 5.4 GODEBUG Safety Net (Optional)

If extra caution is desired, set GODEBUG overrides at deployment time via environment variable on the builder pod:

```
GODEBUG=x509sha1=1,httplaxcontentlength=1
```

Or via Go source directive in the builder's main package:

```go
//go:debug x509sha1=1
//go:debug httplaxcontentlength=1
```

**Based on the analysis, these are likely unnecessary** — SHA-1 cert rejection was already active under Go 1.19, and HTTP content-length validation affects servers more than clients.

### 5.5 Testing Strategy

#### Tier 1: Core BuildConfig Operations (All branches)

| Test | What It Validates |
|------|------------------|
| Dockerfile build with `FROM` from internal registry | Registry pull over TLS, buildah execution |
| Dockerfile build with `FROM` from external registry (quay.io) | External TLS, DNS resolution |
| S2I build with builder image pull | S2I strategy, source-to-image library, image inspection |
| BuildConfig push to internal OpenShift registry | Registry push over TLS, auth credential handling |
| BuildConfig with git source over HTTPS | Git clone (external binary, not Go TLS — validates git still works) |
| BuildConfig with binary input | `bsdtar` extraction |

#### Tier 2: Auth and Certificate Paths (4.12–4.16)

| Test | What It Validates |
|------|------------------|
| Build with `.dockercfg` auth | Credential provider, JSON parsing |
| Build with `.dockerconfigjson` auth | Alternate auth format |
| Build against registry with self-signed CA | Cert loading from `/etc/containers/certs.d/` |
| Build against registry with client cert auth | Client cert TLS via `tlsclientconfig.SetupCertificates()` |
| Build with custom CA bundle via `/etc/pki/ca-trust` | CA trust mount chain |
| Build with ICSP/IDMS mirror registries | Registry mirror resolution + TLS |

#### Tier 3: Edge Cases (4.12–4.13 only)

| Test | What It Validates |
|------|------------------|
| Build against TLS 1.0-only registry | c/image v5.22.0's explicit `MinVersion: TLS10` still works with Go 1.22 |
| S2I incremental build | Image pull with push credentials (different auth path) |
| Build with HTTP_PROXY/HTTPS_PROXY | Proxy connection through `http.ProxyFromEnvironment` |
| Build with very large context | Memory and I/O behavior under new runtime |

### 5.6 Monitoring After Rollout

Watch for these specific error patterns in build logs:

| Error Pattern | Possible Cause | Mitigation |
|---------------|---------------|------------|
| `tls: protocol version not supported` | TLS version negotiation failure (unlikely given explicit config) | Set `GODEBUG=tls10default=1` |
| `x509: certificate signed by unknown authority` | CA trust chain issue | Verify `/etc/pki/ca-trust` mount |
| `x509: certificate relies on legacy Common Name field` | SAN-less cert (already failing pre-bump) | Customer must update cert |
| `x509: certificate has expired or is not yet valid` | Clock or cert issue (not Go version related) | N/A |
| `unauthorized` or `authentication required` | HTTP header/auth handling change | Set `GODEBUG=httplaxcontentlength=1` |
| Build pod panic/crash | `panic(nil)` behavior change or other runtime issue | Check pod logs for stack trace |

### 5.7 Rollback Plan

| Severity | Action |
|----------|--------|
| **Specific behavior issue** | Set `GODEBUG` env var on builder pod to revert individual behaviors |
| **Multiple issues** | Revert the `go` directive in go.mod while keeping the new compiler |
| **Compilation failure** | Revert Go compiler version in build pipeline |
| **Widespread build failures** | Revert the entire change (Go compiler + go.mod directive) |

### 5.8 Release Notes Guidance

Suggested release note text:

> **Builder Go compiler update**: The OpenShift Builder has been recompiled with Go 1.22 to address security vulnerabilities in the Go standard library. This is a compiler-only change — no builder functionality has been modified. Build operations (Dockerfile, S2I, Custom) continue to function as before.
>
> **Note for OCP 4.14+**: Private registries must support TLS 1.2 or higher. TLS 1.0/1.1 connections are not supported. This was already the effective behavior in previous releases.
>
> **Note for OCP 4.12–4.13**: TLS 1.0 connections to private registries remain supported due to explicit configuration in the container image library.

---

## 6. Appendix

### A. Complete Builder Dependency Version Matrix

| Dependency | 4.12 | 4.13 | 4.14 | 4.16 | 4.17 | 4.19 |
|-----------|------|------|------|------|------|------|
| **go directive** | 1.19 | 1.19 | 1.19 | 1.21 | 1.22.0 | 1.22.8 |
| containers/buildah | v1.26.9 (go 1.16) | v1.29.5 (go 1.17) | v1.33.12 (go 1.20) | v1.33.12 (go 1.20) | v1.37.7 | v1.39.7 (go 1.22.8) |
| containers/image | v5.22.0 (go 1.17) | v5.24.1 | v5.29.4 (go 1.19) | v5.29.4 (go 1.19) | v5.32.2 | v5.34.3 (go 1.22.8) |
| containers/storage | v1.42.0 | v1.45.3 | v1.51.2 | v1.51.2 | v1.55.1 | v1.57.2 |
| containers/common | v0.49.1 | v0.51.2 | v0.57.7 | v0.57.7 | v0.60.4 | v0.62.3 |
| openshift/source-to-image | v1.3.2 (go 1.18) | v1.3.2 | v1.3.2 | v1.3.2 | v1.4.0 | v1.4.0 |
| openshift/imagebuilder | v1.2.4 | v1.2.4 | v1.2.15 | v1.2.15 | v1.2.15 | v1.2.15 |
| fsouza/go-dockerclient | v1.7.11 | v1.9.3 | v1.10.x | v1.10.x | v1.12.0 | v1.12.0 |
| k8s.io/client-go | v0.25.2 | v0.26.1 | v0.28.2 | v0.28.2 | v0.30.2 | v0.30.2 |
| golang.org/x/crypto | **PINNED Mar 2020** | **PINNED Sep 2022** | v0.19.0 | v0.19.0 | v0.28.0 | v0.29.0 |
| golang.org/x/net | v0.17.0 (replace) | v0.17.0 (replace) | v0.17.0 (replace) | v0.18.0 | v0.30.0 | v0.33.0 |

### B. Local Code Analysis Findings (from cloned repos)

The following findings come from direct analysis of the locally cloned `builder` and `buildah` repositories.

#### Builder (release-4.12)

| Check | Result |
|-------|--------|
| Closures inside for-loops | **NONE found** — zero `go func()` or `func()` closures that capture loop variables |
| `crypto/tls`, `tls.Config`, `MinVersion`, `GODEBUG` references | **NONE found** — zero TLS configuration code in entire `pkg/build/` |
| Direct HTTP client usage | **Only crioclient** — `crioclient.go` creates `http.Client` for Unix socket (not TLS). Uses deprecated `tr.Dial` (works in Go 1.22). |
| `exec.Command` usage | **3 calls only**: `cp` (2x), `bsdtar` (1x) — all standard binaries, not affected by `os/exec` PATH changes |
| `panic(nil)` usage | **NONE found** |
| `unsafe.Pointer`, `reflect.SliceHeader` | **NONE found** |
| `ioutil` usage | **12 files** — all using deprecated but functional wrappers (`ioutil.ReadFile`, `TempFile`, `TempDir`, `WriteFile`) |
| URL parsing (`url.Parse`) | `util.go:ParseProxyURL()` — already handles `url.Parse` behavior changes with explicit fallback logic |
| SCM auth cert handling | `scmauth/cacert.go` — creates gitconfig pointing to CA cert file. TLS handled by `git` binary, not Go |
| `context.TODO/Background` usage | Standard usage for buildah and K8s API calls — no version sensitivity |

#### Buildah (release-1.26, used in builder 4.12)

| Check | Result |
|-------|--------|
| Goroutines inside for-loops | **Found in `add.go`** — lines 392, 398, 492, 527. BUT: (1) `wg.Wait()` synchronizes before next iteration, so behavior is identical under old and new semantics. (2) buildah's go.mod says `go 1.16`, so per-iteration loop variable semantics do NOT apply to buildah's code regardless. |
| `archive/tar` usage | **Used in `add.go`, `digester.go`, `image.go`** — but for WRITING tar (via `tar.NewWriter`), NOT reading. The `tarinsecurepath` GODEBUG change only affects `tar.Reader`. No impact. |
| TLS configuration | **NONE found** — all TLS is delegated to `containers/image` |
| `exec.Command` usage | Container runtime commands (`runc`/`crun`), `slirp4netns` for networking, overlay mount programs — all explicit binary paths |

### C. Source Code Files Analyzed

| File | What Was Checked |
|------|-----------------|
| `pkg/build/builder/daemonless.go` | Buildah integration, `types.SystemContext` usage, TLS config, cert mounts |
| `pkg/build/builder/docker.go` | Docker build strategy, `DockerClient` interface routing |
| `pkg/build/builder/dockerutil.go` | `DockerClient` interface definition, retry logic, auth handling |
| `pkg/build/builder/source.go` | Git clone (`exec.Command("git")`), binary extraction (`exec.Command("bsdtar")`), image source extraction |
| `pkg/build/builder/sti.go` | S2I build strategy, source-to-image library integration |
| `pkg/build/builder/common.go` | `crypto/sha1` (build tags), `math/rand` (random tags), build status updates |
| `pkg/build/builder/util.go` | URL parsing, registry normalization, cert mount paths |
| `pkg/build/builder/transient_mounts.go` | CA trust mounts, RHSM cert mounts, build volume mounts |
| `pkg/build/builder/cmd/dockercfg/cfg.go` | Docker auth file reading, credential provider integration |
| `containers/image/.../docker_client.go` | TLS config creation (`newDockerClient`), transport setup (`detectProperties`) |
| `containers/image/.../tlsclientconfig.go` | Certificate loading, system cert pool, HTTP transport defaults |
| `containers/buildah/pull.go` | Image pull delegation to `containers/common/libimage` |
| Builder `go.mod` (all branches) | Dependency versions, `go` directive, `replace` directives |
| Dependency `go.mod` files | Per-module `go` directives (controls loopvar and per-module GODEBUG) |

### C. Key Source Code Excerpts

**containers/image v5.22.0 — TLS configuration (4.12–4.13)**:
```go
// docker/docker_client.go:168-173
func serverDefault() *tls.Config {
    return &tls.Config{
        MinVersion:   tls.VersionTLS10,
        CipherSuites: tlsconfig.DefaultServerAcceptedCiphers,
    }
}
```

**containers/image v5.29.4+ — TLS configuration (4.14+)**:
```go
// docker/docker_client.go — MinVersion removed, CipherSuites still explicit
tlsClientConfig := &tls.Config{
    CipherSuites: tlsconfig.DefaultServerAcceptedCiphers,
}
```

**Builder — no TLS code anywhere**:
```
$ grep -r "tls\.Config\|MinVersion\|GODEBUG" pkg/build/builder/
(no results)
```

**Builder — external binary usage (not Go stdlib)**:
```go
// source.go — archive extraction
exec.Command("bsdtar", "-x", "-o", "-m", "-f", "-", "-C", dir)

// source.go — file copy
exec.Command("cp", "-vLRf", src, dst)

// source.go — git clone handled by git binary via GitClient interface
```
