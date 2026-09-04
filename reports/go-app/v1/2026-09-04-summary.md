# Remediation Summary

> Original image: `ghcr.io/sgrsaga/go-app:v1`
> Final image: `ghcr.io/sgrsaga/go-app:v1`
> Status: `no_improvement`

# Container Security Remediation Summary

**Image:** `ghcr.io/sgrsaga/go-app:v1`
**Run status:** `no_improvement`
**Iterations executed:** 1
**Result:** 483 findings before → 483 findings after (Δ 0)

---

## 1. Executive Summary

| State | Risk Rating | CRITICAL | HIGH | Total |
|-------|-------------|----------|------|-------|
| **Before** | 🔴 **Critical** | 22 | 461 | 483 |
| **After**  | 🔴 **Critical** | 22 | 461 | 483 |

**Assessment:** This remediation run achieved **no measurable improvement**. Zero CVEs were resolved, zero were newly introduced, and the final image is byte-for-byte the same tag (`v1`) as the original. The overall risk posture remains **Critical**, driven overwhelmingly by a **compiled-in Go toolchain that is ~4 minor versions out of date** (`stdlib v1.21.13`), plus stale Alpine/musl userland packages (`openssl 3.3.1-r3`, `musl 1.2.5-r0`, `zlib 1.3.1-r1`).

The single dominant issue is that the application binary was built with **Go 1.21.13**, which is end-of-life and carries **1 CRITICAL (`CVE-2025-68121`, ×20 call sites) plus the entire block of `stdlib` HIGH findings**. None of these can be fixed by patching the running container — they require a **rebuild with a current Go toolchain**. The remediation automation had no lever to pull because it did not (or could not) rebuild the application or bump the base image, so the run correctly reported `no_improvement`.

**Bottom line:** The image is **not** safe to promote. The path forward is a rebuild, not a re-scan.

---

## 2. What Changed

**Nothing changed.** The run terminated after a single iteration with the same image digest and tag on both sides.

| Attempted lever | Applied? | Effect |
|-----------------|----------|--------|
| Base image tag bump | ❌ No | — |
| OS package upgrade (`apk upgrade`) | ❌ No | — |
| Base image swap (distroless / newer Alpine) | ❌ No | — |
| Application dependency / Go toolchain upgrade | ❌ No | — |
| Rebuild of Go binary | ❌ No | — |

**Why the run stalled:** The overwhelming majority of findings (≈440 of 483) are attributed to the package `stdlib` at `v1.21.13` — i.e., the Go standard library **statically linked into the application binary**. Automated OS-package remediation (`apk`) cannot touch a compiled Go binary. The remaining ~40 findings are Alpine userland packages that *do* have fixes available, but they were not upgraded — suggesting the pipeline either skipped the `apk upgrade` layer, pinned versions, or ran against a cached/immutable layer.

**Steps/iterations:** 1 iteration, 0 effective remediation actions.

---

## 3. Remaining Risk Breakdown

### 3a. OS packages — fixes ARE available (not applied this run)

These are **directly remediable** by upgrading the base layer. There is **no reason to accept these** — they should be fixed in the next build.

| Package | Installed | Fix Version | CVEs | Max Severity |
|---------|-----------|-------------|------|--------------|
| `libcrypto3` / `libssl3` | `3.3.1-r3` | `3.3.7-r0` | CVE-2026-31789, CVE-2026-28387/28388/28389/28390, CVE-2025-15467, CVE-2025-69421, CVE-2024-6119, CVE-2024-12797 | **CRITICAL** |
| `musl` / `musl-utils` | `1.2.5-r0` | `1.2.5-r3` | CVE-2026-40200, CVE-2025-26519 | HIGH |
| `zlib` | `1.3.1-r1` | `1.3.2-r0` | CVE-2026-22184 | HIGH |

**Remediation:**
```dockerfile
# In the runtime stage of your Dockerfile:
FROM alpine:3.21           # or newer patch tag that ships openssl>=3.3.7-r0
RUN apk upgrade --no-cache \
 && apk add --no-cache --upgrade \
      libcrypto3 libssl3 musl musl-utils zlib
```
> ⚠️ `openssl 3.3.7-r0` clears the **only remaining CRITICAL that is trivially fixable at the OS layer** (`CVE-2026-31789`, heap overflow) plus a use-after-free RCE (`CVE-2026-28387`). Prioritize this.

### 3b. Compiled-in / application-level CVEs — require rebuild or code change

All `stdlib` findings are baked into the binary at compile time. **No container patch can remove them.** The fix is to **rebuild with a patched Go toolchain**.

| CVE (representative) | Component | Fixed in Go | Class |
|----------------------|-----------|-------------|-------|
| **CVE-2025-68121** (CRITICAL) | `crypto/tls` cert validation | 1.24.13 / 1.25.7 | Auth bypass |
| CVE-2026-56858 | `html/template` | 1.25.13 / 1.26.6 | XSS |
| CVE-2026-56862 | `crypto/tls` | 1.25.13 / 1.26.6 | DoS |
| CVE-2026-39822 | `os.Root` symlink | 1.25.12 / 1.26.5 | Path escape |
| CVE-2026-56853 | `net/http` (h2c) | 1.25.13 / 1.26.6 | Protocol downgrade |
| CVE-2026-33814 | `net/http/http2` | 1.25.10 / 1.26.3 | HTTP/2 DoS |
| CVE-2026-27145 / -32280 / -32281 | `crypto/x509` | 1.25.9–1.25.11 | DoS |
| CVE-2026-33818, -56859 | `encoding/asn1`, `encoding/xml` | 1.25.13 | DoS |
| CVE-2025-61726 / -61729 / -25679 / -56860 | `net/url`, `crypto/x509` | 1.24.11+ | DoS |
| CVE-2026-39820 / -42499 | `net/mail` | 1.25.10 | DoS |
| CVE-2026-33811 / -39836 | `net` | 1.25.10 | DoS |
| CVE-2026-42504 | `mime` | 1.25.11 | DoS |
| CVE-2024-34156 | `encoding/gob` | 1.22.7 / 1.23.1 | DoS |

**Single remediation clears all of the above.** The highest fix floor across the set is **Go 1.25.13 / 1.26.6 / 1.27.0-rc.3**. Building with **Go 1.25.13** (latest stable line satisfying every listed fix version) eliminates the CRITICAL and **every** `stdlib` HIGH.

```dockerfile
# Build stage — bump the toolchain
FROM golang:1.25.13-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# Ensure the compiler version, not just the module go directive, is used:
RUN go build -trimpath -ldflags="-s -w" -o /app ./...
```
Also update the module directive to pull in patched `golang.org/x/net` (covers the vendored `idna`/`http2` findings surfaced as `CVE-2026-39821`, `CVE-2026-33814`):
```bash
go get golang.org/x/net@latest
go mod tidy
```

**Verification after rebuild:**
```bash
go version -m ./app | grep -E 'go1\.|golang.org/x/net'   # confirm toolchain + deps
trivy image --severity CRITICAL,HIGH ghcr.io/sgrsaga/go-app:v2
```

---

## 4. Risk Acceptance Template

> Use **only** for CVEs a team consciously chooses not to fix this cycle. Given that **all** remaining CVEs here are fixable via rebuild/base-bump, risk acceptance should be the **exception**, time-boxed, and justified per-CVE.

```
CVE: <ID>
Status: Risk Accepted
Reason: <e.g., affected code path (net/mail parsing) is not reachable in this
         service; ingress is mTLS-only and untrusted input cannot reach the
         vulnerable parser; DoS-only with no data exposure>
Reviewed by: <name / role>
Review date: <YYYY-MM-DD>
Next review: <YYYY-MM-DD + 90 days>
Compensating controls: <ref to section 5 items applied>
Tracking ticket: <JIRA/GH issue link>
```

**Do NOT risk-accept the following without executive sign-off** (network-reachable, exploit ≠ DoS-only):

```
CVE: CVE-2025-68121
Status: Risk Accepted
Reason: MUST NOT accept — crypto/tls certificate validation bypass (CRITICAL).
        Directly undermines TLS trust. Rebuild required.
Reviewed by: <security lead — mandatory>
Review date: <YYYY-MM-DD>
Next review: <30 days max>
```
```
CVE: CVE-2026-31789
Status: Risk Accepted
Reason: MUST NOT accept — OpenSSL heap overflow, fix trivially available
        (3.3.7-r0). Bump base image.
Reviewed by: <security lead — mandatory>
Review date: <YYYY-MM-DD>
Next review: <30 days max>
```

---

## 5. Residual Risk Guidance (Compensating Controls)

These controls **reduce blast radius while the rebuild is pending** — they do **not** substitute for fixing `CVE-2025-68121` or `CVE-2026-31789`.

### Network isolation
Limit reachability of the DoS-heavy `net/*`, `mime`, `encoding/*` parsers to trusted sources only.
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: go-app-restrict }
spec:
  podSelector: { matchLabels: { app: go-app } }
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - namespaceSelector: { matchLabels: { trust: internal } }
      ports: [{ protocol: TCP, port: 8443 }]
  egress:
    - to:
        - namespaceSelector: { matchLabels: { trust: internal } }
```

### mTLS enforcement (mitigates TLS-validation and h2c CVEs)
Terminate/validate TLS at a mesh sidecar so the vulnerable in-process `crypto/tls` path (`CVE-2025-68121`) is not the sole trust boundary, and disable cleartext h2c (`CVE-2026-56853`).
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata: { name: go-app-mtls }
spec:
  selector: { matchLabels: { app: go-app } }
  mtls: { mode: STRICT }
```

### Read-only root filesystem (mitigates `os.Root` symlink escape, `CVE-2026-39822`)
```yaml
securityContext:
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  runAsUser: 65532
  capabilities: { drop: ["ALL"] }
volumeMounts:
  - { name: tmp, mountPath: /tmp }
volumes:
  - { name: tmp, emptyDir: {} }
```

### seccomp / AppArmor (limits post-exploit syscalls for the OpenSSL UAF/RCE)
```yaml
securityContext:
  seccompProfile: { type: RuntimeDefault }
# AppArmor (annotation form for older clusters):
# container.apparmor.security.beta.kubernetes.io/go-app: runtime/default
```

### Resource limits (blunts the many DoS-class CVEs)
Nearly all remaining HIGHs are memory/CPU-exhaustion DoS. Bound them:
```yaml
resources:
  limits:   { cpu: "1", memory: "512Mi" }
  requests: { cpu: "250m", memory: "128Mi" }
```
Also enforce request timeouts and body-size limits at the ingress/proxy layer to starve `net/url`, `net/mail`, `mime`, and `encoding/xml` amplification vectors.

---

### Recommended Next Action (priority order)
1. **Rebuild with Go 1.25.13** → eliminates the CRITICAL `CVE-2025-68121` and every `stdlib` HIGH (~440 findings).
2. **Bump base image / `apk upgrade`** to get `openssl 3.3.7-r0`, `musl 1.2.5-r3`, `zlib 1.3.2-r0` → clears the remaining CRITICAL (`CVE-2026-31789`) and all OS HIGHs.
3. **Re-tag as `v2` and re-scan** — do not overwrite `v1`; the immutable-tag reuse is what made this run a no-op.
4. Apply Section 5 controls immediately as interim mitigation while (1) and (2) ship.

> After steps 1–2, expected residual: **0 CRITICAL / near-0 HIGH**. This entire finding set is remediable in a single rebuild — no genuine "no-fix-available" risk exists in the current inventory.