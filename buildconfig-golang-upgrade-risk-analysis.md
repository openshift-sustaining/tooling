# Go Compiler Version Bump: Impact Analysis for OpenShift Builder

## 1. Problem Statement

### Background

The OpenShift Builder (`github.com/openshift/builder`) is the platform component responsible for executing customer-defined BuildConfig operations — Dockerfile builds, Source-to-Image (S2I) builds, and Custom builds. It runs as a pod within the cluster and performs image pulls, builds, and pushes on behalf of end users.

The builder binary is compiled with varying Go versions across OCP releases 4.12 through 4.21. Several CVEs in Go standard library and dependency code require a newer Go compiler to resolve, necessitating a bump of the Go compiler version across all maintained release branches.

**The core concern**: Changing the Go compiler version changes the behavior of the Go standard library that the builder and all its dependencies are compiled against. Since the builder executes customer workloads, any behavioral change could impact end-user build operations across the entire cluster.

### Current State

- **BuildConfig Go**: The `go` directive in the builder's go.mod (controls GODEBUG behavioral defaults)
- **Golang Builder (ART Gate)**: The actual Go compiler version ART uses to compile the builder binary

| OCP Release | BuildConfig Go (go.mod) | Golang Builder (ART Gate) |
|-------------|------------------------|---------------------------|
| 4.12 | 1.19 | 1.19 |
| 4.13 | 1.19 | 1.19 |
| 4.14 | 1.19 | 1.20 |
| 4.15 | 1.19 | 1.20 |
| 4.16 | 1.21 | 1.21 |
| 4.17 | 1.22.0 | 1.22 |
| 4.18 | 1.22.0 | 1.22 |
| 4.19 | 1.22.8 | 1.23 |
| 4.20 | 1.22.8 | 1.24 |
| 4.21 | 1.22.8 | 1.25 |

### Future State (After Go Bump)

| OCP Release | BuildConfig Go (go.mod) | Golang Builder (ART Gate) |
|-------------|------------------------|---------------------------|
| 4.12 | **1.22.z** | 1.22 |
| 4.13 | **1.22.z** | 1.22 |
| 4.14 | **1.22.z** | 1.22 |
| 4.15 | **1.22.z** | 1.22 |
| 4.16 | **1.22.z** | 1.22 |
| 4.17 | 1.22.z | 1.22 |
| 4.18 | 1.22.z | 1.22 |
| 4.19 | 1.22.z | 1.23 |
| 4.20 | 1.22.z | 1.24 |
| 4.21 | 1.22.z | 1.25 |

**Key observations:**
- **4.17–4.21 require NO meaningful Go upgrade** — 4.17-4.18 are already at Go 1.22.0 (patch-level z-stream is not a behavioral change), and 4.19-4.21 are already at Go 1.22.8 with ART gates even higher (1.23–1.25).
- **4.12–4.16 are the ONLY branches that require actual Go version upgrades.** These are the focus of this analysis.
- All upstream dependencies (buildah, containers/image, source-to-image, client-go, etc.) are compiled into the builder binary. When the builder's Go compiler changes, ALL dependency code is recompiled. Each dependency's own go.mod `go` directive controls per-module semantics like loop variable scoping (Go 1.22) — see the dependency version matrix in Appendix A for per-module `go` directives.

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

The 4.12 pin to a March 2020 `x/crypto` is the highest compilation risk — this is from the Go 1.14 era. The 4.13 pin to September 2022 is less risky but still needs verification. While `x/crypto` provides supplemental cryptographic primitives (not the TLS stack — that's in Go's standard library `crypto/tls`), these old pins may cause compilation compatibility issues with Go 1.22. See section 4.7.

---

## 2. Proposed Plan

### Target State

Bump the Go compiler version to **Go 1.22.z** for the OpenShift Builder (and its upstream dependencies including buildah) across release branches 4.12 through 4.16. Branches 4.17–4.21 are already at Go 1.22.x and require no meaningful changes.

### What Changes and What Does Not

| Aspect | Changes? | Details |
|--------|----------|---------|
| Go compiler version | **YES** (4.12–4.16 only) | Bumped to Go 1.22.z. 4.17+ already at 1.22.x — no change. |
| go.mod `go` directive | **Likely YES** | Should match compiler version per Go conventions |
| Builder source code | NO | No code changes |
| Upstream dependency source code | NO | No code changes to buildah, containers/image, or any other dependency |
| Dependency versions (go.mod `require`) | NO | All dependency versions stay the same |
| `replace` directives | NO | Existing pins remain |

### Branch-by-Branch Go Compiler Jump

**Important distinction**: The `go` directive in go.mod controls GODEBUG behavioral defaults, while the **Golang Builder (ART Gate)** is the actual Go compiler that produces the binary. These can differ — notably on 4.14–4.15 where go.mod says `go 1.19` but ART already compiles with Go 1.20.

The table below shows **compiler jumps** (ART Gate → target), which determine the actual stdlib code compiled into the builder binary:

| OCP Release | Compiler Jump (ART Gate) | go.mod Jump (GODEBUG) | Scope |
|-------------|--------------------------|----------------------|-------|
| **4.12** | 1.19 → 1.22.z (3 minor) | 1.19 → 1.22 | **Largest jump — highest scrutiny** |
| **4.13** | 1.19 → 1.22.z (3 minor) | 1.19 → 1.22 | Large jump |
| **4.14** | **1.20 → 1.22.z** (2 minor) | 1.19 → 1.22 | Moderate (compiler already at 1.20) |
| **4.15** | **1.20 → 1.22.z** (2 minor) | 1.19 → 1.22 | Moderate (compiler already at 1.20) |
| **4.16** | 1.21 → 1.22.z (1 minor) | 1.21 → 1.22 | Small jump |
| **4.17–4.18** | Already 1.22 | Already 1.22.0 | **No change needed** |
| **4.19** | Already 1.23 | Already 1.22.8 | **No change needed** |
| **4.20** | Already 1.24 | Already 1.22.8 | **No change needed** |
| **4.21** | Already 1.25 | Already 1.22.8 | **No change needed** |

**The real work is concentrated on 4.12–4.16.** Branches 4.17–4.21 are already at Go 1.22.x and require no action.

**Key observation**: For **4.14–4.15**, the ART Gate is already Go 1.20, so Go 1.20 stdlib changes (constant-time crypto, etc.) are already compiled into the currently-shipping binary. The actual compiler jump is only 2 minor versions.

**Note on upstream dependencies**: Each dependency compiled into the builder binary has its own go.mod `go` directive, which controls per-module semantics (e.g., Go 1.22 loop variable scoping). The dependency `go` directives are listed in Appendix A. Regardless of each dependency's `go` directive, ALL dependency code is recompiled with the builder's new Go compiler and uses the new Go stdlib.

---

## 3. Go Standard Library Changes by Version

This section catalogs the **behavioral changes** in each Go release that will be newly introduced into the builder binary by the Go 1.22 bump. Only Go 1.20, 1.21, and 1.22 are covered — changes from Go 1.17 through 1.19 are already compiled into the currently-shipping builder binary (ART Gate ≥ 1.19 for all branches) and are not impacted.

**Which changes are newly introduced into the builder binary by this bump:**

| Go Version Changes | 4.12–4.13 (ART 1.19→1.22) | 4.14–4.15 (ART 1.20→1.22) | 4.16 (ART 1.21→1.22) |
|-------------------|:-:|:-:|:-:|
| Go 1.20 changes | **YES** | — | — |
| Go 1.21 changes | **YES** | **YES** | — |
| Go 1.22 changes | **YES** | **YES** | **YES** |

**Note**: "—" means the change is already compiled into the currently-shipping builder binary. "YES" means it will be newly introduced. All upstream dependencies (buildah, containers/image, etc.) are compiled as part of the builder binary, so these changes apply to the builder AND all its dependency code equally.

**Per-module `go` directive caveat**: While the builder's compiler determines the stdlib code, some Go 1.22 behavioral changes (specifically loop variable scoping) are controlled by each module's own go.mod `go` directive. Upstream dependencies with a `go` directive below 1.22 retain old loop variable semantics even when compiled with Go 1.22. See Section 4.8.4 and Appendix A for details.

---

### 3.1 Go 1.20 Changes (applies to: builder 4.12–4.13)

These are the changes introduced when the compiler moves from Go 1.19 to Go 1.20. For builder 4.14–4.15, ART already compiles with Go 1.20, so these are already active.

| Change | Package | Type | GODEBUG | Builder Relevance |
|--------|---------|------|---------|-------------------|
| **`math/rand` auto-seeded** — Global random source no longer starts at `Seed(1)`. Top-level functions produce different sequences on each run. | `math/rand` | Breaking | `randautoseed=0` | **NONE** — builder's `randomBuildTag()` benefits from auto-seeding. Already active via GODEBUG mapping. |
| **`archive/tar` insecure path detection** — `Reader.Next` can return `ErrInsecurePath` for dangerous filenames (absolute paths, `../` traversal). Default: allowed. | `archive/tar` | GODEBUG | `tarinsecurepath=0` to enable | **NONE** — builder uses `bsdtar` binary, not Go's `archive/tar`. Upstream deps use `tar.NewWriter` (writing only, not reading). |
| **`archive/zip` insecure path detection** — Similar to tar. Default: allowed. | `archive/zip` | GODEBUG | `zipinsecurepath=0` to enable | **NONE** — builder/buildah don't use Go's `archive/zip` |
| **Constant-time RSA and ECDSA implementations** — 15–45% slower RSA, 5–30% slower ECDSA operations. | `crypto` | Compiled code | None | **LOW** — affects TLS handshake performance only. Minor impact on registry pull/push latency. |
| **`tls.CertificateVerificationError` new type** — TLS handshake failures due to cert verification return new error type. | `crypto/tls` | Behavioral | None | **LOW** — may affect error type assertions in retry logic |
| **HTTP HEAD requests with body now accepted** (server-side). | `net/http` | Behavioral | None | **NONE** — builder is a client |
| **Cookie name whitespace trimmed** instead of rejected. | `net/http` | Behavioral | None | **NONE** — builder doesn't handle HTTP cookies for builds |
| **`ReverseProxy` no longer injects User-Agent** header. | `net/http` | Behavioral | None | **NONE** — builder doesn't use ReverseProxy |
| **`archive/zip` rejects data in directory entries**. | `archive/zip` | Behavioral | None | **NONE** — builder/buildah don't use Go's `archive/zip` |

### 3.2 Go 1.21 Changes (applies to: builder 4.12–4.15)

Go 1.21 formalizes the GODEBUG compatibility mechanism. For builder 4.16 (ART Gate 1.21), these changes are already compiled in.

| Change | Package | Type | GODEBUG | Builder Relevance |
|--------|---------|------|---------|-------------------|
| **GODEBUG compatibility mechanism formalized** — `go` directive in go.mod now controls GODEBUG defaults. Versions < 1.20 map to `go 1.20`. | `runtime` | Framework | N/A | **KEY MECHANISM** — this is why GODEBUG jump for all branches is go1.20→go1.22 (not go1.19→go1.22) |
| **`panic(nil)` now produces `*runtime.PanicNilError`** — `recover()` no longer returns `nil` for `panic(nil)`. Code using `recover() == nil` to detect "no panic" will break. | `runtime` | Breaking | `panicnil=1` | **LOW** — no `panic(nil)` patterns found in builder or buildah source code audit |
| **GC tail latency reduced** — Up to 40% reduction in GC tail latency. Possible small throughput cost. | `runtime` | Compiled code | None | **POSITIVE** — reduced latency for build operations |
| **Transparent huge pages managed by runtime** — May reduce memory overhead by up to 50% in pathological cases. | `runtime` | Compiled code | None | **POSITIVE** — better memory efficiency |
| **TLS session tickets issued on every resumption** — Reduces cross-connection tracking. May increase cost with many session ticket keys. | `crypto/tls` | Compiled code | None | **NONE** — transparent to registry connections |
| **Extended Master Secret (RFC 7627) support** — Client and server now support EMS for improved TLS security. | `crypto/tls` | Compiled code | None | **POSITIVE** — improved TLS security |
| **More specific TLS alert codes for client auth failures** — Returns "certificate required", "unknown CA", "expired", or "bad certificate" instead of generic alerts. | `crypto/tls` | Compiled code | None | **LOW** — may change error messages in build logs for cert failures |
| **`crypto/x509` name constraint enforcement corrected** — Applied to issued certs, not the cert expressing the constraint. | `crypto/x509` | Bug fix | None | **LOW** — may change validation outcomes for unusual cert chains |
| **`crypto/sha256` hardware acceleration** — SHA-224/SHA-256 use native CPU instructions on amd64 (3–4x speedup). | `crypto/sha256` | Compiled code | None | **POSITIVE** — faster TLS handshakes and content verification |
| **`math/rand` global source uses `runtime.fastrand64`** — Per-thread scaling, no global mutex. | `math/rand` | Compiled code | None | **NONE** — transparent improvement |
| **`flag` panics on duplicate flag definitions** — Programs registering the same flag name twice will panic at startup. | `flag` | Breaking | None | **CHECK NEEDED** — builder should be verified for duplicate flag registrations (unlikely) |
| **`reflect.SliceHeader` / `StringHeader` deprecated** — Should use `unsafe.Slice`/`unsafe.String` instead. | `reflect` | Deprecation | None | **NONE** — no usage found in builder/buildah |

### 3.3 Go 1.22 Changes (applies to: ALL branches 4.12–4.16)

| Change | Package | Type | GODEBUG | Builder Relevance |
|--------|---------|------|---------|-------------------|
| **For-loop variable per-iteration scoping** — Each iteration creates new variables. Controlled by each module's `go` directive, NOT the main module. | Language | Breaking | Per-module `go` directive | **NONE** — no affected patterns found in builder. Upstream deps retain old semantics (go.mod < 1.22). See Section 4.8.4. |
| **RSA key exchange cipher suites removed from defaults** — CipherSuites without ECDHE no longer offered by default. | `crypto/tls` | Breaking | `tlsrsakex=1` | **NONE** — c/image explicitly sets `CipherSuites` in all versions |
| **TLS 1.2 minimum for servers** — Default `MinVersion` for servers raised to TLS 1.2. | `crypto/tls` | Breaking | `tls10server=1` | **NONE** — builder is a client, not a TLS server |
| **Empty Content-Length header rejected** — HTTP requests/responses with empty `Content-Length: ` header value are now errors. | `net/http` | Breaking | `httplaxcontentlength=1` | **LOW** — builder is an HTTP client; malformed Content-Length from registries is very rare |
| **Enhanced `ServeMux` routing patterns** — `{` and `}` in patterns now interpreted as wildcards. | `net/http` | Breaking | `httpmuxgo121=1` | **NONE** — builder does not serve HTTP or register ServeMux patterns |
| **`encoding/json` `\b`/`\f` escaping changed** — Backspace/form-feed serialized as `\b`/`\f` instead of `\u0008`/`\u000c`. | `encoding/json` | Behavioral | None | **NONE** — semantically equivalent JSON. Builder doesn't do byte-level JSON comparison |
| **Heap metadata alignment change** — Objects previously aligned to 16+ bytes may now be 8-byte aligned. | `runtime` | Compiled code | `noallocheaders=1` | **NONE** — no assembly or cgo code in builder depends on alignment |
| **`math/rand` uses ChaCha8 PRNG** — Cryptographic-quality random generator for global source. | `math/rand` | Compiled code | None | **NONE** — transparent change; builder's `randomBuildTag()` gets better randomness |
| **`ExportKeyingMaterial` restricted** — Requires TLS 1.3 or Extended Master Secret. | `crypto/tls` | Breaking | `tlsunsafeekm=1` | **NONE** — builder doesn't use EKM |
| **TLS max RSA key size** — Default limit of 8192 bits. | `crypto/tls` | New limit | `tlsmaxrsasize=N` | **NONE** — registry certs don't use >8192-bit RSA keys |
| **`slices.Delete`/`Compact`/`Replace` zero removed elements** — Prevents memory leaks from dangling pointers in slice tails. | `slices` | Behavioral | None | **NONE** — builder/buildah on these branches don't use `slices` package |
| **`reflect.Value.IsZero()` returns `true` for `-0.0`** and structs with non-zero blank fields. | `reflect` | Behavioral | None | **NONE** — no affected patterns |

---

## 4. Impact Analysis

### 4.1 Builder Architecture and Network Communication Paths

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

### 4.2 Impact Path 1: Registry TLS Connections (containers/image)

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

### 4.3 Impact Path 2: Kubernetes API Communication (k8s.io/client-go)

The builder communicates with the Kubernetes API server to update build status via `HandleBuildStatusUpdate()` in `common.go`. This uses `k8s.io/client-go`, which has its own TLS configuration derived from the service account token and kubeconfig.

**Impact**: `k8s.io/client-go` configures its own `tls.Config` based on the cluster CA bundle and service account credentials. It explicitly sets TLS parameters. The Go version bump does not change how client-go establishes TLS connections to the API server.

### 4.4 Impact Path 3: Source-to-Image (openshift/source-to-image)

For S2I builds, the builder uses `openshift/source-to-image` as a Go library. Key operations:

- **Builder image pull**: Handled via `fsouza/go-dockerclient`, which connects to a local Docker/CRI socket (unix socket, not TLS).
- **S2I script download**: If scripts are fetched via HTTP(S), this uses Go's `net/http` directly. The S2I library's go.mod says `go 1.18`, so its own module behavior stays at Go 1.18 semantics.
- **Image inspection**: Uses `fsouza/go-dockerclient` to inspect images via the local container runtime socket.

**Impact**: S2I script download over HTTPS could theoretically be affected by TLS default changes. However, Go's TLS 1.2 client default was already in effect under Go 1.19 (changed in Go 1.18). The S2I library's own go.mod says `go 1.18`, but GODEBUG defaults are controlled by the main module (builder), not the dependency. Since the effective GODEBUG level is already `go 1.20`, no additional changes apply.

**Note**: The `fsouza/go-dockerclient` connects to the local container runtime via Unix domain socket — no TLS involved. The CRI-O client in the builder (`crioclient/crioclient.go`) similarly uses a Unix socket with `http.Transport.Dial` (deprecated field — should use `DialContext`). This deprecated field still functions in Go 1.22.

### 4.5 Impact Path 4: Git Clone and Source Extraction

- **Git clone**: The builder shells out to the `git` binary via `exec.Command`. The git binary uses its own TLS stack (OpenSSL/GnuTLS on RHEL), completely independent of Go's `crypto/tls`. **No Go version impact.**
- **Archive extraction**: Uses `exec.Command("bsdtar", ...)` — external binary, not Go's `archive/tar`. **No Go version impact.**
- **File copying**: Uses `exec.Command("cp", ...)` — external binary. **No Go version impact.**
- **Image source extraction**: Uses `buildah.NewBuilder()` with `types.SystemContext` → goes through `containers/image` → covered in section 4.2.

### 4.6 Impact Path 5: Docker Auth and Credential Handling

The builder reads Docker credentials from the filesystem via `cmd/dockercfg/cfg.go`:
- Reads `.dockercfg`, `.dockerconfigjson`, `config.json` files
- Uses `k8s.io/kubernetes/pkg/credentialprovider` for credential keyring lookup
- Passes credentials to `containers/image` via `config.SetAuthentication()`

This is pure file I/O and JSON parsing — no network or TLS operations. **No Go version impact.**

### 4.7 Compilation Compatibility: x/crypto Pin in 4.12 and 4.13

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

### 4.8 Go Standard Library Behavioral Changes

These changes apply to the builder binary as a whole — all code in the builder and its dependencies compiled together.

**Two distinct mechanisms control stdlib behavior:**

1. **Go compiler (ART Gate)** — Determines the actual stdlib code compiled into the binary. Pure implementation changes (e.g., constant-time crypto in Go 1.20) take effect based on the compiler version. For 4.14–4.15, ART already compiles with Go 1.20, so Go 1.20 implementation changes are already active in the shipping binary.

2. **go.mod `go` directive** — Controls **GODEBUG behavioral defaults** via Go 1.21+'s compatibility mechanism. These are runtime toggles for behavioral changes that have backward-compatibility switches (e.g., TLS defaults, HTTP parsing strictness).

#### 4.8.1 GODEBUG Compatibility Mechanism

Go 1.21+ uses the `go` directive in the **main module's** go.mod to set GODEBUG defaults. Go versions below 1.20 in go.mod are mapped to `go 1.20` for GODEBUG purposes.

| Builder `go` directive | Effective GODEBUG | Notes |
|------------------------|-------------------|-------|
| Current: `go 1.19` (4.12–4.15) | Maps to **`go 1.20`** | Most Go 1.20 defaults already active |
| Current: `go 1.21` (4.16) | **`go 1.21`** | Go 1.21 defaults active |
| Future: `go 1.22` | **`go 1.22`** | Adds go1.21 + go1.22 defaults (4.12–4.15) or go1.22 only (4.16) |

This means for 4.12–4.15, the **effective GODEBUG jump is go1.20 → go1.22**. For 4.16, it is **go1.21 → go1.22**. Many impactful changes are already active.

#### 4.8.2 Changes Already Active Under Current Configuration

These changes are already in effect because the current `go 1.19` directive maps to GODEBUG `go 1.20`:

| Change | Introduced In | Status | Builder Relevance |
|--------|--------------|--------|-------------------|
| SHA-1 x509 certificate rejection | Go 1.18 | **Already active** (`x509sha1=0`) | Customers using SHA-1 signed certs for private registries would already be failing |
| x509 Common Name fallback disabled | Go 1.15–1.17 | **Already active** | Certs must use SAN, not CN — already enforced |
| `math/rand` auto-seeding | Go 1.20 | **Already active** (`randautoseed=1`) | Builder's `randomBuildTag()` already auto-seeded |
| `os/exec` PATH security | Go 1.19 | **Already active** | No current-dir binary resolution |
| TLS 1.0/1.1 client default | Go 1.18 | **Overridden** by c/image explicit config | See section 4.2 |
| `;` rejected in URL query params | Go 1.17 | **Already active** | Already in effect under Go 1.19 |
| `archive/tar` insecure paths | Go 1.20 | **Not applicable** | Builder uses `bsdtar` binary, not Go's `archive/tar` |

#### 4.8.3 New GODEBUG Defaults Activated by Bumping to `go 1.22`

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

#### 4.8.4 For-Loop Variable Semantics (Go 1.22)

Go 1.22 changes loop variable capture to per-iteration scope. This is controlled by each module's `go` directive, NOT the main module's:

| Module | go.mod `go` directive | Gets New Semantics? |
|--------|----------------------|---------------------|
| **openshift/builder** (main module) | 1.22 (after bump) | **YES** — builder's own code. Audited: no affected patterns found. |
| containers/buildah v1.26.9 | 1.16 | No (below 1.22) |
| containers/buildah v1.33.12 | 1.20 | No (below 1.22) |
| containers/image v5.22.0 | 1.17 | No (below 1.22) |
| containers/image v5.29.4 | 1.19 | No (below 1.22) |
| openshift/source-to-image v1.3.2 | 1.18 | No (below 1.22) |
| k8s.io/client-go v0.25.2 | ~1.19 | No (below 1.22) |

Since none of the upstream dependencies have a go.mod `go` directive ≥ 1.22, their code retains the old loop variable semantics. Only the builder's own code gets per-iteration scoping — and the audit found zero affected patterns.

**Source code audit**: Manual review of all builder source files (`daemonless.go`, `docker.go`, `source.go`, `sti.go`, `common.go`, `dockerutil.go`, `util.go`, `transient_mounts.go`) found **no closure-captured loop variables** that would change behavior. Key upstream dependencies (buildah `add.go`, containers/image) were also spot-checked and found safe.

#### 4.8.5 Deprecated APIs Still Used

| API | Deprecated Since | Used In | Impact |
|-----|-----------------|---------|--------|
| `ioutil.ReadFile` | Go 1.16 | `common.go`, `source.go` | **NONE** — still functional, wraps `os.ReadFile` |
| `ioutil.TempFile` | Go 1.16 | `daemonless.go` | **NONE** — still functional, wraps `os.CreateTemp` |
| `ioutil.TempDir` | Go 1.16 | `daemonless.go` | **NONE** — still functional, wraps `os.MkdirTemp` |
| `ioutil.WriteFile` | Go 1.16 | `source.go` | **NONE** — still functional, wraps `os.WriteFile` |
| `net.Dialer.DualStack` | Go 1.12 | c/image v5.22.0 `tlsclientconfig.go` | **NONE** — field ignored since Go 1.12 |

All deprecated APIs remain fully functional in Go 1.22. No behavioral changes.

#### 4.8.6 Crypto Performance

Go 1.20 introduced constant-time implementations for RSA and ECDSA, which are slightly slower:
- RSA operations: 15–45% slower
- ECDSA operations: 5–30% slower

This is a compiled code change (not GODEBUG-controlled), so it depends on the **actual compiler version** (ART Gate), not the go.mod directive:

| Branch | Current ART Gate | Constant-time crypto already active? | Impact of Go 1.22 bump |
|--------|-----------------|--------------------------------------|------------------------|
| 4.12–4.13 | Go 1.19 | **No** — compiled with Go 1.19 | **Will activate** — minor crypto performance reduction |
| 4.14–4.15 | Go 1.20 | **Yes** — already compiled with Go 1.20 | No additional change |
| 4.16 | Go 1.21 | **Yes** | No additional change |

For 4.12–4.13, the Go 1.22 bump will introduce the Go 1.20 constant-time crypto into the binary. The performance impact is minor and only affects TLS handshakes (not data transfer), so registry pull/push throughput is unaffected.

---

## 5. Risk Assessment Summary

### Per-Release Risk Matrix

| OCP | Compiler Jump (ART Gate) | GODEBUG Jump (go.mod) | TLS Impact | x/crypto Risk | Overall |
|-----|--------------------------|----------------------|------------|---------------|---------|
| **4.12** | 1.19→1.22 (3 minor) | go1.20→go1.22 | NONE (explicit MinVersion + CipherSuites) | **Compile check needed** (2020 pin) | **LOW** |
| **4.13** | 1.19→1.22 (3 minor) | go1.20→go1.22 | NONE (explicit MinVersion + CipherSuites) | **Compile check needed** (Sep 2022 pin) | **LOW** |
| **4.14** | **1.20→1.22** (2 minor) | go1.20→go1.22 | NONE (explicit CipherSuites, TLS 1.2 default) | None (v0.19.0) | **LOW** |
| **4.15** | **1.20→1.22** (2 minor) | go1.20→go1.22 | NONE (same as 4.14) | None (v0.19.0) | **LOW** |
| **4.16** | 1.21→1.22 (1 minor) | go1.21→go1.22 | NONE (same as 4.14) | None (v0.19.0) | **MINIMAL** |
| **4.17–18** | **no change** (already 1.22) | None | NONE | None | **NONE** |
| **4.19** | **no change** (already 1.23) | None | NONE | None | **NONE** |
| **4.20** | **no change** (already 1.24) | None | NONE | None | **NONE** |
| **4.21** | **no change** (already 1.25) | None | NONE | None | **NONE** |

**Note**: 4.17–4.21 are already at Go 1.22.x. No meaningful Go upgrade is needed for these branches. The analysis below focuses on 4.12–4.16.

### Known Test Failures Requiring Code Changes

The Go bump will cause **3 unit test failures** in `pkg/build/builder/common_test.go` due to Go 1.20's `math/rand` auto-seeding change. These tests use `rand.Seed(0)` to produce deterministic random output and assert exact string matches — the seeded sequence changes under Go 1.20+. This is a test-only issue; the production code (`randomBuildTag()`, `containerName()`) is unaffected and actually benefits from auto-seeding. See Appendix B for the full analysis and recommended fix.

### Why the Risk is Lower Than Initially Expected

1. **TLS is explicitly configured in the dependency that handles ALL registry connections.** The `containers/image` library always explicitly sets `CipherSuites` and (in v5.22.0) `MinVersion`. Go's default TLS changes do not propagate through explicit configurations.

2. **GODEBUG is already at the `go 1.20` level.** The current `go 1.19` directive maps to `go 1.20` for GODEBUG. Most impactful stdlib changes (SHA-1 cert rejection, rand seeding, PATH security, TLS 1.0 client deprecation) are already active in the currently-shipping builder. Additionally, for **4.14–4.15**, ART already compiles with Go 1.20, so Go 1.20 implementation changes (constant-time crypto, etc.) are already present in the shipping binary — the actual compiler jump for these branches is only 2 minor versions.

3. **The builder delegates sensitive operations to external binaries.** Git clone → `git` binary (uses OS TLS, not Go). Archive extraction → `bsdtar` binary. File copy → `cp` binary. These are not affected by Go version changes.

4. **No loop variable capture bugs found.** Manual audit of all builder source files found zero closure-captured loop variables affected by Go 1.22's per-iteration change.

5. **The builder itself has no TLS code.** Zero `tls.Config`, zero `MinVersion`, zero `GODEBUG` settings. Everything flows through dependencies which explicitly configure their own TLS.

6. **Kubernetes API communication is independently configured.** `k8s.io/client-go` sets up its own TLS based on kubeconfig/service-account — not affected by Go default changes.

---

## 6. CI Test Coverage

This section documents the existing CI test workflows that run for each builder release branch via `ci-operator`. These tests will serve as the validation gate after the Go compiler bump — if they pass, the builder binary behavior is confirmed unchanged.

Source: `openshift/release` repo, `ci-operator/config/openshift/builder/openshift-builder-release-4.*.yaml`

### 6.1 Container Tests (Unit / Verification)

These run in a container from the `src` image without provisioning a cluster.

| Test | Command | Behavior | 4.12 | 4.13 | 4.14 | 4.15 | 4.16 | 4.17 | 4.18 | 4.19 | 4.20 | 4.21 |
|------|---------|----------|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
| `unit` | `make test` | skip_if_only_changed (docs) | YES | YES | YES | YES | YES | YES | YES | YES | YES | YES |
| `verify` | `make verify` | skip_if_only_changed (docs) | YES | YES | YES | YES | YES | YES | YES | YES | YES | YES |

### 6.2 E2E Workflow Tests (Cluster-Provisioned)

These provision an AWS cluster and run end-to-end tests against a real OpenShift installation with the builder image under test.

| Test | Workflow | Behavior | 4.12–4.21 |
|------|----------|----------|:---------:|
| `e2e-aws-ovn` | `openshift-e2e-aws` | skip_if_only_changed | ALL |
| `e2e-aws-ovn-builds` | `openshift-e2e-aws-builds` | skip_if_only_changed | ALL |
| `e2e-aws-ovn-proxy` | `openshift-e2e-aws-proxy` | optional, always_run=false | ALL |
| `e2e-aws-ovn-image-ecosystem` | `openshift-e2e-aws-image-ecosystem` | skip_if_only_changed | ALL |
| `e2e-aws-ovn-cgroupsv2` | `openshift-e2e-aws-cgroupsv2` | optional, always_run=false | ALL |
| `e2e-aws-ovn-builds-techpreview` | `openshift-e2e-aws-builds` | optional, always_run=false (TechPreview) | ALL |
| `security` | `openshift-ci-security` | optional | ALL |

### 6.3 Dependency Verification (4.19+ only)

| Test | Step Ref | 4.12–4.18 | 4.19–4.21 |
|------|----------|:---------:|:---------:|
| `verify-deps` | `go-verify-deps` | — | YES |

### 6.4 Test Coverage Assessment

**Total tests per branch:**
- **4.12–4.18**: 9 tests (2 container + 7 e2e)
- **4.19–4.21**: 10 tests (2 container + 7 e2e + 1 dep verification)

**Coverage is uniform across all branches.** The same test workflows run on every release from 4.12 through 4.21. Key validation points:

| BuildConfig Operation | Covered By | What It Validates |
|----------------------|------------|-------------------|
| Dockerfile build + image push | `e2e-aws-ovn-builds` | Full build lifecycle: pull base image → build → push to internal registry (TLS, auth, buildah execution) |
| S2I build | `e2e-aws-ovn-builds` | Source-to-image strategy: pull builder image, run assemble/run scripts, push result |
| Image ecosystem (various languages) | `e2e-aws-ovn-image-ecosystem` | Builds across multiple language stacks (Ruby, Python, Node.js, etc.) |
| Build behind proxy | `e2e-aws-ovn-proxy` | HTTP/HTTPS proxy handling, `PROXY_URL` parsing |
| Build with TechPreview features | `e2e-aws-ovn-builds-techpreview` | BuildConfig operations with `TechPreviewNoUpgrade` feature gate |
| cgroups v2 compatibility | `e2e-aws-ovn-cgroupsv2` | Build execution under cgroups v2 (runtime behavior) |
| General platform e2e | `e2e-aws-ovn` | Broad OpenShift conformance including builds |
| Unit tests | `unit` | Builder source code logic, `make test` |
| Code verification | `verify` | Lint, formatting, `make verify` |

**Conclusion**: The existing CI coverage is sufficient to detect behavioral regressions from the Go compiler bump. The `e2e-aws-ovn-builds` and `e2e-aws-ovn-image-ecosystem` workflows exercise the full BuildConfig lifecycle — image pull (TLS), build execution (buildah), and image push (TLS + auth) — which are the primary paths where Go stdlib changes could surface.

---

## 7. Suggestions and Guidelines

### 7.1 Pre-Implementation Checklist

Before submitting the Go bump, verify the following:

**For ALL branches (4.12–4.16):**
- [ ] `go build` succeeds with Go 1.22.z without source code changes
- [ ] `go vet` passes without new issues
- [ ] **Fix `rand.Seed(0)` in `common_test.go`** — `TestRandomBuildTag` and `TestContainerName` use `rand.Seed(0)` for deterministic output and assert exact strings. Go 1.20+ makes `rand.Seed()` a no-op on the global source, so these tests WILL FAIL. Fix by using `rand.New(rand.NewSource(0))` or by changing assertions to validate format/length instead of exact values. See Appendix B for details.
- [ ] Existing unit tests pass (after `rand.Seed` fix above)
- [ ] CI pipeline produces a valid builder image

**For 4.12 specifically:**
- [ ] Verify compilation with the pinned `golang.org/x/crypto v0.0.0-20200323165209` (March 2020) — highest compilation risk; if it fails, the `replace` directive needs updating
- [ ] Verify compilation with the pinned `github.com/docker/docker v0.0.0-20200911110540` — very old version, potential compatibility issues
- [ ] Verify compilation with the pinned `github.com/containerd/containerd v1.6.6`

**For 4.13 specifically:**
- [ ] Verify compilation with the pinned `golang.org/x/crypto v0.0.0-20220919173607` (September 2022) — lower risk than 4.12 but still needs verification

### 7.2 Implementation Order

Start with the lowest-risk branches and work backward. Branches 4.17–4.21 require no action (already at Go 1.22.x).

1. **Phase 1**: 4.16 — ART Gate 1.21 → 1.22 (1 minor version jump). Smallest change. Low risk.
2. **Phase 2**: 4.14–4.15 — ART Gate 1.20 → 1.22 (2 minor). Modern dependencies (c/image v5.29.4, x/crypto v0.19.0). Low risk.
3. **Phase 3**: 4.13 — ART Gate 1.19 → 1.22 (3 minor). Older dependencies, pinned x/crypto from Sep 2022. Needs compilation verification.
4. **Phase 4**: 4.12 — ART Gate 1.19 → 1.22 (3 minor). Oldest dependencies, pinned x/crypto from March 2020. Requires thorough compilation and runtime verification.

### 7.3 go.mod `go` Directive Decision

**Recommended: Bump the `go` directive to `1.22`.**

- Matches the compiler, which is the Go convention
- Activates Go 1.22 GODEBUG defaults (all analyzed as safe for the builder)
- Enables per-iteration loop variable semantics (no affected patterns found in builder code)
- Future issues can be mitigated with explicit `//go:debug` directives without reverting the compiler

**Alternative: Keep the `go` directive at the current value (e.g., `1.19`).**

- Creates a compiler/directive mismatch (unusual, may cause tooling warnings)
- Preserves GODEBUG at the `go 1.20` mapping level
- Only gains the compiler's security fixes without adopting any new stdlib defaults

### 7.4 GODEBUG Safety Net (Optional)

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

### 7.5 Testing Strategy

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

### 7.6 Monitoring After Rollout

Watch for these specific error patterns in build logs:

| Error Pattern | Possible Cause | Mitigation |
|---------------|---------------|------------|
| `tls: protocol version not supported` | TLS version negotiation failure (unlikely given explicit config) | Set `GODEBUG=tls10default=1` |
| `x509: certificate signed by unknown authority` | CA trust chain issue | Verify `/etc/pki/ca-trust` mount |
| `x509: certificate relies on legacy Common Name field` | SAN-less cert (already failing pre-bump) | Customer must update cert |
| `x509: certificate has expired or is not yet valid` | Clock or cert issue (not Go version related) | N/A |
| `unauthorized` or `authentication required` | HTTP header/auth handling change | Set `GODEBUG=httplaxcontentlength=1` |
| Build pod panic/crash | `panic(nil)` behavior change or other runtime issue | Check pod logs for stack trace |

### 7.7 Rollback Plan

| Severity | Action |
|----------|--------|
| **Specific behavior issue** | Set `GODEBUG` env var on builder pod to revert individual behaviors |
| **Multiple issues** | Revert the `go` directive in go.mod while keeping the new compiler |
| **Compilation failure** | Revert Go compiler version in build pipeline |
| **Widespread build failures** | Revert the entire change (Go compiler + go.mod directive) |

### 7.8 Release Notes Guidance

Suggested release note text:

> **Builder Go compiler update**: The OpenShift Builder has been recompiled with Go 1.22 to address security vulnerabilities in the Go standard library. This is a compiler-only change — no builder functionality has been modified. Build operations (Dockerfile, S2I, Custom) continue to function as before.
>
> **Note for OCP 4.14+**: Private registries must support TLS 1.2 or higher. TLS 1.0/1.1 connections are not supported. This was already the effective behavior in previous releases.
>
> **Note for OCP 4.12–4.13**: TLS 1.0 connections to private registries remain supported due to explicit configuration in the container image library.

---

## 8. Appendix

### A. Complete Builder Dependency Version Matrix

| Dependency | 4.12 | 4.13 | 4.14 | 4.15 | 4.16 | 4.17 | 4.18 | 4.19 | 4.20 | 4.21 |
|-----------|------|------|------|------|------|------|------|------|------|------|
| **go directive** | 1.19 | 1.19 | 1.19 | 1.19 | 1.21 | 1.22.0 | 1.22.0 | 1.22.8 | 1.22.8 | 1.22.8 |
| **ART Gate** | 1.19 | 1.19 | 1.20 | 1.20 | 1.21 | 1.22 | 1.22 | 1.23 | 1.24 | 1.25 |
| containers/buildah | v1.26.9 | v1.29.5 | v1.33.12 | v1.33.12 | v1.33.12 | v1.37.7 | v1.37.7 | v1.39.7 | v1.39.7 | v1.39.7 |
| containers/image | v5.22.0 | v5.24.1 | v5.29.4 | v5.29.4 | v5.29.4 | v5.32.2 | v5.32.2 | v5.34.3 | v5.34.3 | v5.34.3 |
| containers/storage | v1.42.0 | v1.45.3 | v1.51.2 | v1.51.2 | v1.51.2 | v1.55.1 | v1.55.1 | v1.57.2 | v1.57.2 | v1.57.2 |
| containers/common | v0.49.1 | v0.51.2 | v0.57.7 | v0.57.7 | v0.57.7 | v0.60.4 | v0.60.4 | v0.62.3 | v0.62.3 | v0.62.3 |
| openshift/source-to-image | v1.3.2 | v1.3.2 | v1.3.2 | v1.3.9 | v1.3.9 | v1.4.0 | v1.4.0 | v1.4.0 | v1.4.0 | v1.4.0 |
| openshift/imagebuilder | v1.2.4 | v1.2.4 | v1.2.15 | v1.2.6 | v1.2.15 | v1.2.15 | v1.2.14 | v1.2.15 | v1.2.15 | v1.2.15 |
| fsouza/go-dockerclient | v1.7.11 | v1.9.3 | v1.10.0 | v1.10.0 | v1.10.0 | v1.12.0 | v1.11.1 | v1.12.0 | v1.12.0 | v1.12.0 |
| k8s.io/client-go | v0.25.2 | v0.26.1 | v0.28.2 | v0.28.2 | v0.28.2 | v0.30.2 | v0.30.2 | v0.30.2 | v0.30.2 | v0.30.2 |
| golang.org/x/crypto | **PINNED Mar 2020** | **PINNED Sep 2022** | v0.19.0 | v0.19.0 | v0.19.0 | v0.28.0 | v0.31.0 | v0.29.0 | v0.32.0 | v0.32.0 |
| golang.org/x/net | v0.17.0 (replace) | v0.17.0 (replace) | v0.17.0 (replace) | v0.18.0 | v0.18.0 | v0.30.0 | v0.33.0 | v0.33.0 | v0.34.0 | v0.34.0 |

### B. Local Code Analysis Findings (from cloned repos)

The following findings come from direct analysis of the locally cloned `builder` repository and spot-checks of key upstream dependencies.

#### Builder Test Files (release-4.12)

All 22 `*_test.go` files in the builder repo were audited for patterns affected by the Go 1.19→1.22 bump. The audit checked for: `rand.Seed()` usage, `ioutil` deprecations, `panic(nil)` recovery, TLS configuration, `unsafe` usage, `sort.Slice` stability assumptions, JSON byte-level comparisons, `t.Parallel()` with loop variable capture, `archive/tar` reader usage, HTTP server patterns, `reflect.SliceHeader`/`StringHeader`, `errors.As`/`Is` patterns, and `exec.Command` PATH sensitivity.

**CRITICAL: `rand.Seed(0)` Test Failures**

`pkg/build/builder/common_test.go` uses `rand.Seed(0)` to produce deterministic random output and then asserts exact string matches. Go 1.20 changed `math/rand` so that the global source is auto-seeded and `rand.Seed()` on the global source becomes effectively a no-op when the global source has already been used. This means the seeded sequences will differ from what the tests expect.

| Test | Line | Expected Value | Status |
|------|------|---------------|--------|
| `TestRandomBuildTag` | 96 | `"temp.builder.openshift.io/test/build-1:f1f85ff5"` | **WILL FAIL** |
| `TestRandomBuildTag` (long name) | 96 | `"8a0f9d66cde28a0ebb1e3ee8ef9a484ce687afe0:f1f85ff5"` | **WILL FAIL** |
| `TestContainerName` | 118 | `"openshift_test-strategy-build_my-build_ns_hook_f1f85ff5"` | **WILL FAIL** |
| `TestRandomBuildTagNoDupes` | 105 | (uniqueness check, not exact match) | Likely OK |

**Fix required**: Replace `rand.Seed(0)` with a local `rand.New(rand.NewSource(0))` and pass it to the functions under test, OR change the test assertions to validate format/length rather than exact output values.

**Note**: These are test-only failures — the production `randomBuildTag()` and `containerName()` functions actually benefit from Go 1.20's auto-seeding (better randomness without explicit seeding). The fix is purely in the test expectations.

**LOW: `ioutil` Deprecation (13 of 22 test files)**

| File | `ioutil` Calls | Impact |
|------|---------------|--------|
| `pkg/build/builder/source_test.go` | 11 (`ReadFile`, `TempDir`, `WriteFile`) | None — functional |
| `pkg/build/builder/docker_test.go` | 3 (`TempDir`, `WriteFile`) | None — functional |
| `pkg/build/builder/common_test.go` | 1 (`TempDir`) | None — functional |
| `pkg/build/builder/cmd/dockercfg/cfg_test.go` | 3 (`TempDir`, `WriteFile`) | None — functional |
| + 9 other test files | Various | None — functional |

All `ioutil` usage is deprecated since Go 1.16 but remains fully functional through Go 1.22. These are wrappers that delegate to `os` package equivalents. No test failures, but represents maintenance debt.

**All Other Patterns: Clean**

| Pattern Checked | Files with Hits | Impact |
|----------------|----------------|--------|
| TLS configuration in tests | 0 | N/A |
| `panic(nil)` / recovery | 0 | N/A |
| `unsafe.Pointer` / `reflect.SliceHeader` | 0 | N/A |
| `sort.Slice` stability assumptions | 0 | N/A |
| JSON byte-level comparison | 0 | N/A |
| `t.Parallel()` with loop variable capture | 0 | N/A |
| `archive/tar` reader | 0 | N/A |
| HTTP server (`httptest.NewServer`) | 1 (`source_test.go`) | None — trivial test server |
| `exec.Command` | 1 (`source_test.go` — `git`) | None — explicit binary path |
| Error string comparison | 1 (`docker_test.go`) | None — app-level errors, not stdlib |

#### Builder Source (release-4.12)

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
| `context.TODO/Background` usage | Standard usage for dependency calls and K8s API — no version sensitivity |

#### Key Upstream Dependencies (spot-checked)

| Dependency | Check | Result |
|-----------|-------|--------|
| containers/buildah (release-1.26) | Goroutines inside for-loops | Found in `add.go` (lines 392, 398, 492, 527) — all safe: `wg.Wait()` synchronizes before next iteration. go.mod says `go 1.16` so new loop semantics don't apply regardless. |
| containers/buildah (release-1.26) | `archive/tar` usage | Used for WRITING only (`tar.NewWriter`), not reading. `tarinsecurepath` only affects `tar.Reader`. No impact. |
| containers/buildah (release-1.26) | TLS configuration | **NONE** — all TLS delegated to containers/image |
| containers/buildah (release-1.26) | `exec.Command` usage | Container runtimes (`runc`/`crun`), `slirp4netns`, overlay mount — all explicit binary paths |
| containers/image (v5.22.0) | TLS configuration | Explicit `MinVersion` and `CipherSuites` — see Section 4.2 |
| containers/image (v5.29.4) | TLS configuration | Explicit `CipherSuites`, Go default `MinVersion` (TLS 1.2) — see Section 4.2 |

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
| `pkg/build/builder/common_test.go` | `rand.Seed(0)` usage, deterministic output assertions, `ioutil` usage |
| `pkg/build/builder/source_test.go` | `ioutil` usage (11 calls), `exec.Command("git")`, `httptest.NewServer` |
| `pkg/build/builder/docker_test.go` | `ioutil` usage, error string comparisons, mixed `os.MkdirTemp`/`ioutil.TempDir` |
| All 22 `*_test.go` files | Full audit for Go bump impact patterns (see Appendix B) |
| `containers/image/.../docker_client.go` | TLS config creation (`newDockerClient`), transport setup (`detectProperties`) |
| `containers/image/.../tlsclientconfig.go` | Certificate loading, system cert pool, HTTP transport defaults |
| `containers/buildah/pull.go` | Image pull delegation to `containers/common/libimage` |
| Builder `go.mod` (all branches) | Dependency versions, `go` directive, `replace` directives |
| Dependency `go.mod` files | Per-module `go` directives (controls loopvar and per-module GODEBUG) |

### D. Key Source Code Excerpts

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
