# Remediation Summary

> Original image: `ghcr.io/sgrsaga/java-app:v1`
> Final image: `ghcr.io/sgrsaga/java-app:v1-golden-base-app`
> Status: `golden_base_app`

# Container Security Remediation Summary

**Image:** `ghcr.io/sgrsaga/java-app:v1` → `ghcr.io/sgrsaga/java-app:v1-golden-base-app`
**Status:** `golden_base_app`
**Iterations run:** 1
**Base artifact:** `eclipse-temurin:17-jdk-alpine` → published as `ghcr.io/sgrsaga/eclipse-temurin:17-jdk-alpine-golden-base`

---

## 1. Executive Summary

| Metric | Before | After |
|--------|--------|-------|
| **Overall risk rating** | **HIGH** | **NONE / CLEAN** |
| Critical | 0 | 0 |
| High | 3 | 0 |
| Medium | 0 | 0 |
| **Total** | **3** | **0** |

The remediation run achieved a **complete elimination of all detected vulnerabilities**. The pre-remediation image carried an overall risk rating of **HIGH**, driven entirely by three instances of a single OpenSSL CVE (`CVE-2026-14456`) surfacing across the `libcrypto3`, `libssl3`, and `openssl` Alpine packages — all three originating from the same underlying OpenSSL 3.5.7-r0 build.

Following a single-iteration OS package upgrade, the image now reports **zero vulnerabilities at all severity levels**, achieving `golden_base_app` status. There is **no residual risk** in the current scan surface: no unfixed OS packages remain, and no compiled-in or application-level CVEs persist. The resulting base has been promoted and published as a reusable golden base to prevent regression across downstream services.

> **Assessment:** This is a clean, fully-resolved remediation. The only forward-looking concern is **drift over time** — new CVEs will inevitably be disclosed against `eclipse-temurin:17-jdk-alpine` and OpenSSL. Continuous rescanning and scheduled golden-base rebuilds are required to preserve this state.

---

## 2. What Changed

The reduction from 3 HIGH findings to 0 was achieved in **1 iteration** consisting of **1 successful remediation step**.

### Remediation mechanism
The fix was **not** a base image tag bump or a base swap. The base image (`eclipse-temurin:17-jdk-alpine`) was retained. Instead, the resolution came from an **OS-level package upgrade** applied within the base build stage:

- **Step: `os-patch`** — An Alpine blanket package upgrade (`apk upgrade`) was executed in the base stage of the multi-stage build. This bumped the OpenSSL toolchain from `3.5.7-r0` to `3.5.8-r0`, which carries the upstream fix for `CVE-2026-14456`.

### Why one step resolved all three findings
The three findings were **not three distinct defects** — they were the same CVE (`CVE-2026-14456`) reported against three packages that share a single OpenSSL source build:

| Package | Before | After |
|---------|--------|-------|
| `libcrypto3` | 3.5.7-r0 | 3.5.8-r0 (patched) |
| `libssl3` | 3.5.7-r0 | 3.5.8-r0 (patched) |
| `openssl` | 3.5.7-r0 | 3.5.8-r0 (patched) |

Upgrading the OpenSSL family in a single `apk upgrade` pass simultaneously cleared all three.

### Validation
Each step in the remediation trail was validated by:
1. A **full image rebuild**
2. The application's **own test suite**
3. A **Trivy rescan**

The `os-patch` step passed all three gates, transitioning `(CRITICAL, HIGH)` counts from `(0, 3)` → `(0, 0)`. No non-improving steps required rollback in this run.

### Diff summary
```
Resolved (3):          CVE-2026-14456 (libcrypto3, libssl3, openssl)
Still present (0):      (none)
Newly introduced (0):  (none)
```

The upgrade introduced **zero new vulnerabilities** — a clean forward step with no regression trade-off.

---

## 3. Remaining Risk Breakdown

**There is no remaining risk in the current scan surface.**

### 3.1 OS packages with no fix available yet
| Package | CVE | Status |
|---------|-----|--------|
| _(none)_ | _(none)_ | All OS package CVEs resolved via `os-patch` |

No OS packages remain in an unpatched or "no-fix-available" state.

### 3.2 Compiled-in / application-level CVEs
| Component | CVE | Remediation guidance |
|-----------|-----|----------------------|
| _(none)_ | _(none)_ | No application-level or compiled-in CVEs detected post-remediation |

No JVM-level, application dependency, or statically-linked CVEs were present in the final scan.

> **Note on scope:** A "0 findings" result reflects the coverage of the scanner (Trivy) against known, published advisories as of the scan date. It does **not** guarantee the absence of zero-day, unpublished, or non-scannable (e.g., business-logic) vulnerabilities. Treat this as *clean against known CVEs*, not *provably invulnerable*.

---

## 4. Risk Acceptance Template

No CVEs remain that require acceptance for this image. The template below is retained for **future use** — if a subsequent scan surfaces an unfixable finding that the team elects to accept, copy the block below into the risk register.

```
CVE: <ID>
Status: Risk Accepted
Reason: <why this is acceptable in this deployment — e.g., component not reachable
         from the app's execution path, vulnerable function never invoked,
         no fix available upstream, mitigated by compensating control X>
Reviewed by: <name>
Review date: <YYYY-MM-DD>
Next review: <YYYY-MM-DD (review date + 90 days)>
```

**Usage rules:**
- Every accepted CVE **must** name a reviewer and a concrete reason (reachability analysis preferred over "low likelihood").
- `Next review` must not exceed **90 days** from `Review date`. Accepted risks expire and must be re-adjudicated.
- Risk acceptance is **per-deployment-context**; an acceptance valid for an internal batch job is not automatically valid for an internet-facing service.

---

## 5. Residual Risk Guidance

Although the image is currently clean, defense-in-depth controls should remain enforced to contain the impact of **future** CVEs (particularly in OpenSSL/TLS-facing code paths, which were the source of the resolved finding). The following compensating controls are recommended baseline for any deployment of this image.

### 5.1 Network Policies
Restrict pod ingress/egress to only required peers. A DoS-class OpenSSL CVE (like the one just resolved) is far less exploitable when the attack surface is not broadly reachable.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: java-app-restrict
spec:
  podSelector:
    matchLabels:
      app: java-app
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector:
            matchLabels: { role: api-gateway }
      ports:
        - protocol: TCP
          port: 8443
  egress:
    - to:
        - namespaceSelector:
            matchLabels: { name: data-tier }
      ports:
        - protocol: TCP
          port: 5432
```

### 5.2 mTLS Enforcement
Terminate and re-originate TLS at a service mesh sidecar (Istio/Linkerd) so the application's OpenSSL stack is not the sole/first TLS parser exposed to untrusted peers. Enforce STRICT mutual TLS.

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: java-app-mtls
spec:
  selector:
    matchLabels:
      app: java-app
  mtls:
    mode: STRICT
```

### 5.3 Read-Only Root Filesystem
Prevent an attacker who gains code execution from tampering with binaries or the (now-patched) OpenSSL libraries on disk.

```yaml
securityContext:
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  runAsNonRoot: true
  runAsUser: 10001
  capabilities:
    drop: ["ALL"]
volumeMounts:
  - name: tmp
    mountPath: /tmp        # writable scratch only where required
```

### 5.4 seccomp / AppArmor Profiles
Constrain the syscall surface to limit exploitation of memory-corruption or resource-exhaustion class bugs (the resolved CVE was DoS via unbounded memory growth — combine with pod memory limits).

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/java-app: runtime/default
resources:
  limits:
    memory: "1Gi"      # hard cap blunts memory-growth DoS classes
    cpu: "1000m"
```

### 5.5 Continuous Assurance
The golden state degrades over time as new advisories are published.

| Control | Recommendation |
|---------|----------------|
| Scheduled rescan | Trivy scan the golden base **daily** in CI |
| Golden-base rebuild | Rebuild + re-promote `eclipse-temurin:17-jdk-alpine-golden-base` on a **weekly** cadence to absorb new `apk upgrade` fixes |
| Admission control | Enforce that only images derived from the published golden base are admitted to production |
| Drift alerting | Alert if any downstream image regresses to an OpenSSL version `< 3.5.8-r0` |

---

*Report generated for run `golden_base_app` — 1 iteration, 1 successful step (`os-patch`), 3 CVE instances resolved, 0 residual.*