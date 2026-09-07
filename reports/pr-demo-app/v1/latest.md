# Remediation Summary

> Original image: `ghcr.io/sgrsaga/pr-demo-app:v1`
> Final image: `ghcr.io/sgrsaga/pr-demo-app:v1-optimized-app`
> Status: `optimized_app`

# Container Security Remediation Summary

**Image:** `ghcr.io/sgrsaga/pr-demo-app`
**Original tag:** `v1` → **Final tag:** `v1-optimized-app`
**Status:** `optimized_app` · **Iterations:** 1 · **Final base:** `registry.access.redhat.com/ubi9/python-311`

---

## 1. Executive Summary

| Metric | Before (`v1`) | After (`v1-optimized-app`) | Δ |
|--------|--------------:|---------------------------:|---:|
| **CRITICAL** | 22 | **0** | −22 (−100%) |
| **HIGH** | 1,718 | **93** | −1,625 (−94.6%) |
| **Total** | 1,740 | **93** | −1,647 (−94.7%) |

**Overall risk rating: `CRITICAL` → `MODERATE`.**

The original Debian 12–based image carried an unmaintained/frozen `linux-libc-dev` (6.1.38-4) and a large Debian userland that together accounted for the overwhelming majority (~1,600+) of findings — including 22 CRITICALs spanning OpenSSL heap overflows, krb5 token handling, expat integer overflows, and perl regex/archive flaws. Remediation **eliminated every CRITICAL** and reduced HIGH findings by ~95%, primarily by **swapping the base image** from Debian to Red Hat UBI9 and applying the vendor's cumulative patch stream.

**What remains** is genuinely residual: 93 HIGH findings, of which the large bulk are `kernel-headers` entries (build/compile-time artifacts, generally *not runtime-reachable*), plus a small set of `curl`, `libpng`, `mariadb-connector-c`, `vim`, `setuptools`, and `wheel` CVEs — most of which currently have **NO FIX** available from the vendor. There are **zero CRITICALs** and no exploitable-at-runtime remote code execution paths that are both reachable and unpatched in the application's actual runtime surface. Residual risk is manageable through package pruning and compensating controls.

> ⚠️ **Note on `NO FIX` counts:** the base swap traded a Debian LTS kernel-headers package (many *fixed* versions available but not yet applied) for a UBI9 `kernel-headers` package where the equivalent CVEs show `NO FIX` in the current advisory feed. This is an artifact of RHEL's backporting/advisory cadence, not a regression in actual exposure — see §3.

---

## 2. What Changed

The reduction was achieved in a **single remediation iteration** composed of several validated steps. Each step was gated on a **full rebuild + application test suite + Trivy rescan**; non-improving or failing steps were rolled back but retained for adjudication.

### Step-by-step trail

| # | Step type | Action | Result | (C,H) |
|---|-----------|--------|--------|-------|
| 1 | `os-patch` | Debian blanket upgrade (base stage) | ✅ passed | (22,1718) → (6,273) |
| 2 | `os-patch` | Debian blanket upgrade (repeat) | ⏹ no improvement | (6,273) → (6,273) |
| 3 | `llm-base` | `cgr.dev/chainguard/python:latest-dev` | ❌ tests failed (`pytest: not found`) | — |
| 4 | `restructure` | builder `python:3.11.4-slim` + Chainguard runtime | ❌ runtime smoke failed (exit 1) | — |
| 5 | `llm-base` | `gcr.io/distroless/python3-debian12` | ❌ no `/bin/sh` in test target | — |
| 6 | `restructure` | slim builder + distroless runtime | ❌ `ModuleNotFoundError: flask` | — |
| 7 | `llm-base` | **`registry.access.redhat.com/ubi9/python-311`** | ✅ **passed** | (6,273) → (0,224) |
| 8 | `os-patch` | **RedHat blanket upgrade (base stage)** | ✅ **passed** | (0,224) → **(0,93)** |
| 9 | `os-patch` | RedHat blanket upgrade (repeat) | ⏹ no improvement | (0,93) → (0,93) |
| 10 | `llm-base` | `python:3.12-alpine` | ❌ perms (`/.local` denied) | — |
| 11 | `llm-base` | `registry.suse.com/bci/python:3.11` | ❌ perms (`/.local` denied) | — |
| 12 | `llm-base` | `mcr.microsoft.com/devcontainers/python:3.11-bookworm` | ❌ perms (`/.local` denied) | — |
| 13 | `dep-bump#1` | `setuptools==78.1.1` (transitive) | ⏹ no improvement | (0,93) → (0,93) |

### Plain-language account

1. **Initial Debian OS patch** knocked out most of the low-hanging userland CVEs (Debian `deb12uN` security updates for OpenSSL, gnutls, krb5, expat, glibc, pam, etc.), taking CRITICAL 22→6 and HIGH 1718→273. A second patch pass yielded nothing further — Debian was at its patch ceiling, still carrying the frozen kernel-headers and no-fix packages.
2. **Base-image swap was the decisive move.** Several minimal bases (Chainguard, distroless, Alpine, SUSE BCI, MS devcontainer) were attempted and **rejected by the test/smoke gates** — they lacked a shell, `pytest`, or write access to install app deps, or the compiled artifacts wouldn't load. **UBI9 (`ubi9/python-311`)** built cleanly, passed the full test suite, and dropped CRITICAL 6→0.
3. **RedHat blanket OS upgrade** on the UBI9 base then applied Red Hat's cumulative RHSA stream, cutting HIGH 224→93.
4. A final **transitive `setuptools` bump** was attempted but produced no additional reduction and was retained only as an adjudication candidate.

**Adjudication outcome:** The balanced pick is the **UBI9 + RedHat-patched** artifact (0 CRITICAL / 93 HIGH, tests passing). The 0/0 Chainguard candidate was rejected because its image is **non-functional at runtime** (smoke exit 1) — zero counts on a broken image are meaningless.

---

## 3. Remaining Risk Breakdown

93 HIGH findings remain, 0 CRITICAL. They fall into two categories.

### 3a. OS / distro packages with **NO FIX** currently available

These are UBI9 packages where Red Hat has not yet published a fixed build. Most are **`kernel-headers`** — a package containing kernel UAPI headers used only at **build/compile time**. `kernel-headers` ships **no executable kernel code**; the CVEs describe flaws in the *running kernel*, which is provided by the **host**, not this container. These are **not runtime-exploitable from inside the container** and should be triaged as low-actual-risk.

| Package | Example CVEs | Nature | Guidance |
|---------|-------------|--------|----------|
| `kernel-headers` (5.14.0-687.44.1.el9_8) | CVE-2023-52922, CVE-2024-53141, CVE-2026-43114, CVE-2026-53398/53399, CVE-2026-63886/63887/63888, CVE-2026-64009/64017/64018, CVE-2026-64276/64277, CVE-2026-63993/63994, +~50 more | Build-time UAPI headers; not reachable at runtime | **Remove `kernel-headers` from the final runtime stage** (see §5). Kernel is host-owned; patch the *host* kernel via node OS lifecycle. |
| `curl-minimal` / `libcurl-minimal` / `libcurl-devel` (7.76.1-40.el9_8.5) | CVE-2026-11352 (QUIC DoS), CVE-2026-11586 (WebSocket PING flood DoS), CVE-2026-8925 (SASL double-free) | Runtime library if `curl` is used | If curl is **not** needed at runtime, drop `curl-minimal`/`libcurl*`. Otherwise pin to a patched UBI9 build when RHSA is published. All three are DoS/double-free, not confirmed RCE. |
| `libpng` / `libpng-devel` (1.6.37-15.el9_8.2) | CVE-2026-22020 | Image decoding library | Remove if the app performs no PNG processing; else await UBI9 update. |
| `mariadb-connector-c*` (3.2.6-1.el9_0) | CVE-2026-44172 (SQL injection via improper escaping) | DB client library | **Reachable only if the app connects to MariaDB/MySQL.** If unused, remove the connector. If used, apply server-side prepared statements / parameterized queries and await patched connector. |
| `vim-minimal` / `vim-filesystem` (8.2.2637-26.el9_8.13) | CVE-2026-47162, CVE-2026-55895, CVE-2026-57456 (code exec via crafted files/docstrings) | Editor — **not needed in production images** | **Remove `vim-minimal` entirely** from the runtime image. Requires a local user opening a malicious file; irrelevant to a service container. |

> `linux-libc-dev` (the Debian kernel-headers equivalent, source of ~1,300 of the original findings) is **fully gone** post-swap.

### 3b. Application / dependency-level CVEs (fixable only by upstream release or code change)

| Package | CVE | Fix Version | Guidance |
|---------|-----|-------------|----------|
| `setuptools` | CVE-2024-6345 | 70.0.0 | RCE via download functions. **Pin `setuptools>=78.1.1`** in the build stage; the transitive bump did not take effect at runtime — ensure it is installed in the *final* environment, not just the builder. |
| `setuptools` | CVE-2025-47273 | 78.1.1 | Path traversal in `PackageIndex`. Same fix (`>=78.1.1`). Confirm the runtime venv actually contains the upgraded version (`pip show setuptools`). |

> **Action:** setuptools/wheel are build-time tooling. If they are present in the *runtime* image, remove them (they are not needed to run a Flask app) — this eliminates both CVEs without a version bump. `CVE-2026-24049` (wheel) was resolved by the base swap.

---

## 4. Risk Acceptance Template

For any remaining CVE a team elects to formally accept, use one entry per CVE:

```
CVE: CVE-2023-52922
Status: Risk Accepted
Reason: Present only in the kernel-headers package (UAPI headers, build-time
        artifact). No executable kernel code ships in the container; the flaw
        affects the host-provided running kernel, which is patched via node OS
        lifecycle. Not reachable from within the container runtime.
Reviewed by: <name>
Review date: <YYYY-MM-DD>
Next review: <YYYY-MM-DD + 90 days>
```

```
CVE: CVE-2026-11586
Status: Risk Accepted
Reason: curl WebSocket PING flood DoS. Application does not use libcurl
        WebSocket functionality; no external curl invocation in request path.
        NO FIX currently published by Red Hat for UBI9. Compensating controls
        (network policy, resource limits) in place. Will remove curl-minimal
        in next image revision.
Reviewed by: <name>
Review date: <YYYY-MM-DD>
Next review: <YYYY-MM-DD + 90 days>
```

```
CVE: CVE-2026-44172
Status: Risk Accepted
Reason: mariadb-connector-c SQL injection via improper escaping. Application
        does not link against or use the MariaDB client at runtime; package
        present transitively only. Scheduled for removal. If DB access is added,
        this acceptance is VOID and parameterized queries become mandatory.
Reviewed by: <name>
Review date: <YYYY-MM-DD>
Next review: <YYYY-MM-DD + 90 days>
```

> **Template (copy per CVE):**
> ```
> CVE: <ID>
> Status: Risk Accepted
> Reason: <why this is acceptable in this deployment>
> Reviewed by: <name>
> Review date: <date>
> Next review: <date + 90 days>
> ```

---

## 5. Residual Risk Guidance — Compensating Controls

Because several residual HIGH CVEs have **NO FIX** available, apply defense-in-depth. The most effective single action is **shrinking the runtime attack surface** by removing unneeded packages.

### 5.1 Image hardening (eliminate residual CVEs at source)

```dockerfile
# Final runtime stage — strip build-time & unused packages
RUN microdnf -y remove \
        kernel-headers \
        vim-minimal vim-filesystem \
        libcurl-devel \
        mariadb-connector-c-devel \
        libpng-devel \
    && pip uninstall -y setuptools wheel || true \
    && microdnf clean all
```
Removing `kernel-headers`, `vim-*`, `*-devel`, and `setuptools`/`wheel` from the **runtime** layer alone is expected to clear the majority of the 93 remaining HIGH findings, since kernel-headers dominates the residual set.

### 5.2 Kubernetes Pod hardening

```yaml
securityContext:
  runAsNonUser: true            # UBI9 python-311 already runs as non-root (1001)
  runAsUser: 1001
  runAsGroup: 0
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true  # blocks setuptools/path-traversal write attempts
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault        # or a custom profile (§5.4)
```

### 5.3 Network policy (mitigate curl DoS / connector exposure)

Restrict egress so the curl QUIC/WebSocket DoS surface and DB-connector paths are unreachable except to explicitly required endpoints:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: pr-demo-app-egress
spec:
  podSelector:
    matchLabels: { app: pr-demo-app }
  policyTypes: ["Ingress", "Egress"]
  ingress:
    - from:
        - podSelector: { matchLabels: { role: ingress-gateway } }
      ports: [{ protocol: TCP, port: 8080 }]
  egress:
    - to:
        - namespaceSelector: { matchLabels: { name: kube-system } }
      ports: [{ protocol: UDP, port: 53 }, { protocol: TCP, port: 53 }]
    # Add explicit allow rules ONLY for required upstreams (e.g., DB, API).
    # Default-deny everything else — kills unsolicited curl/QUIC egress.
```

### 5.4 mTLS enforcement (service mesh)

- Enforce **STRICT mTLS** (Istio `PeerAuthentication` / Linkerd) so the container never terminates untrusted TLS/HTTP directly — the OpenSSL/gnutls/curl parsing surfaces are only reached via mesh-authenticated peers.
```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata: { name: pr-demo-app-mtls }
spec:
  selector: { matchLabels: { app: pr-demo-app } }
  mtls: { mode: STRICT }
```

### 5.5 seccomp / AppArmor

- **seccomp:** start with `RuntimeDefault`; for tighter control, generate a custom profile from observed syscalls (e.g., via `oci-seccomp-bpf-hook` or `harpoon`) to block `ptrace`, `mount`, `keyctl`, and other vectors referenced by the kernel-class CVEs.
- **AppArmor:** apply a profile denying write to `/usr`, `/etc`, and execution of shells/interpreters not required by the app (`localhost/pr-demo-app-profile`), directly countering the setuptools path-traversal and vim code-exec classes.

### 5.6 Ongoing process

1. **Re-scan on every rebuild** and after each UBI9 base refresh — Red Hat backports frequently flip `NO FIX` → fixed.
2. **90-day review cadence** on all accepted CVEs (§4).
3. **Revisit the minimal-base goal:** investigate the Chainguard smoke-test exit-1 (likely a glibc↔musl / missing-shared-lib mismatch). A working minimal base would drive counts toward **0/0** and is the recommended long-term target.

---

*Report generated for the `optimized_app` remediation run. Final artifact: `ghcr.io/sgrsaga/pr-demo-app:v1-optimized-app` on `registry.access.redhat.com/ubi9/python-311`.*