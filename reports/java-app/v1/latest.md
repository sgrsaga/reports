# Remediation Summary

> Original image: `ghcr.io/sgrsaga/java-app:v1`
> Final image: `ghcr.io/sgrsaga/java-app:v1-golden-base-app`
> Status: `golden_base_app`

# Container Security Remediation Summary

**Image under remediation:** `ghcr.io/sgrsaga/java-app:v1`
**Final remediated image:** `ghcr.io/sgrsaga/java-app:v1-golden-base-app`
**Final status:** `golden_base_app` ✅
**Iterations run:** 1
**Outcome:** All 5 findings resolved — image reached a clean, golden-base state.

---

## 1. Executive Summary

| Metric | Before | After |
|--------|--------|-------|
| Overall Risk Rating | **HIGH** | **NONE / CLEAN** |
| Critical | 0 | 0 |
| High | 5 | 0 |
| Medium | 0 | 0 |
| Total findings | 5 | 0 |

**Assessment:** The original image `v1` carried a **HIGH** aggregate risk rating driven by five HIGH-severity CVEs, all of which were denial-of-service (DoS) class vulnerabilities in core OS libraries (`openssl`/`libcrypto3`/`libssl3` and `libexpat`). Every one of these findings had an upstream Alpine package fix already available, meaning the risk was fully remediable without any application code change.

A single, self-contained remediation iteration — a blanket Alpine package upgrade in the base stage — resolved **100% (5/5)** of the findings with **zero regressions** and **zero newly introduced CVEs**. The remediated image passed a full rebuild, the application's own test suite, and a clean Trivy rescan. **No residual vulnerability risk remains** at the time of this scan; the sections below on remaining risk and acceptance are provided as forward-looking governance guidance rather than for currently open findings.

---

## 2. What Changed

The reduction was achieved through a **single OS-level package upgrade step**, not a base-tag bump, base swap, or application dependency change.

- **Iterations run:** 1
- **Remediation steps applied:** 1 (all successful, none rolled back)

### Remediation trail

| Step | Action | Stage | Result | (Crit, High) transition |
|------|--------|-------|--------|--------------------------|
| `os-patch` | Alpine blanket package upgrade (`apk upgrade`) | base stage | ✅ passed | (0, 5) → (0, 0) |

**Plain-language account:**

All five HIGH findings were rooted in outdated Alpine system packages shipped in the `eclipse-temurin:17-jdk-alpine` base layer:

- `openssl`, `libcrypto3`, `libssl3` at `3.5.7-r0` → upgraded to `3.5.8-r0` (fixes `CVE-2026-14456`)
- `libexpat` at `2.8.3-r0` → upgraded to `2.8.4-r0` (fixes `CVE-2026-66046` and `CVE-2026-76641`)

Because every affected package had an available fixed version in the Alpine repositories, a blanket in-stage upgrade pulled all three OpenSSL components and the Expat library up to patched releases in one pass. Note that `CVE-2026-14456` appears three times in the "Resolved" diff — this reflects the **same CVE resolved across three distinct packages** (`libcrypto3`, `libssl3`, `openssl`), all remediated by the single OpenSSL bump.

The upgraded base was captured as a reusable **golden base artifact** for downstream reuse:

- **Final base:** `eclipse-temurin:17-jdk-alpine`
- **Published golden base:** `ghcr.io/sgrsaga/eclipse-temurin:17-jdk-alpine-golden-base`

Every step was validated by a full image rebuild, execution of the application's test suite, and a Trivy rescan before acceptance.

---

## 3. Remaining Risk Breakdown

**Current residual vulnerability count: 0.**

- **Still present (0):** none
- **Newly introduced (0):** none

### 3.1 OS packages with no fix available yet

None. Every OS package finding had an available upstream fix, and all were applied.

### 3.2 Compiled-in / application-level CVEs

None. No application-level, JAR-embedded, or compiled-in CVEs remained after remediation. The application layer was unaffected by this run — all findings originated in the base OS layer.

> ⚠️ **Note on `apk upgrade` durability:** A blanket OS upgrade patches against the package state at *build time*. New CVEs will surface in these same packages over time. The golden base should be rebuilt and rescanned on a recurring cadence (recommended: weekly, or on any new HIGH/CRITICAL advisory affecting OpenSSL or Expat) to prevent drift back into a vulnerable state.

---

## 4. Risk Acceptance Template

No open CVEs require acceptance at this time. Retain the template below for any **future** finding that a team chooses to accept rather than remediate:

```
CVE: <ID>
Status: Risk Accepted
Reason: <why this is acceptable in this deployment — e.g., vulnerable code path
         not reachable, feature not enabled, no network exposure, mitigating control X>
Reviewed by: <name>
Review date: <date>
Next review: <date + 90 days>
```

Example (illustrative only — not an active finding):

```
CVE: CVE-2026-14456
Status: Risk Accepted
Reason: DoS-class OpenSSL vuln; TLS termination handled at ingress/service mesh,
        pod-internal TLS uses restricted cipher paths not affected. Deployed only
        in an internal-only namespace with NetworkPolicy egress lockdown.
Reviewed by: <security-lead>
Review date: 2026-06-01
Next review: 2026-08-30
```

---

## 5. Residual Risk Guidance — Compensating Controls

Even with a clean scan, the following controls harden the runtime against **newly disclosed** DoS-class vulnerabilities (the exact profile of the CVEs remediated here) before the next base rebuild lands. Apply defense-in-depth:

### 5.1 Network Policies (limit blast radius of DoS/reachability)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: java-app-default-deny
spec:
  podSelector:
    matchLabels:
      app: java-app
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: ingress-gateway
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: platform-services
```

Restricting ingress to trusted gateways limits exposure of the OpenSSL/Expat parsing paths that the resolved CVEs targeted.

### 5.2 mTLS Enforcement

Enforce strict mutual TLS at the mesh (Istio/Linkerd) so all inter-service traffic is authenticated and encrypted, preventing untrusted peers from reaching TLS/XML parsing surfaces:

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: java-app-mtls-strict
  namespace: java-app
spec:
  mtls:
    mode: STRICT
```

### 5.3 Read-Only Root Filesystem

Prevents write-based exploitation and tampering with patched libraries:

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
    mountPath: /tmp
volumes:
  - name: tmp
    emptyDir: {}
```

### 5.4 seccomp / AppArmor Profiles

Constrain the syscall surface to blunt exploitation of memory-growth / DoS vulnerabilities:

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/java-app: runtime/default
```

### 5.5 Resource Limits (direct DoS mitigation)

Because all remediated CVEs were **DoS / unbounded-memory-growth** class, enforce hard memory/CPU ceilings so a malicious payload cannot exhaust the node:

```yaml
resources:
  limits:
    memory: "1Gi"
    cpu: "1000m"
  requests:
    memory: "512Mi"
    cpu: "250m"
```

---

## Appendix — Diff Summary

| Category | Count | CVEs |
|----------|-------|------|
| Resolved | 5 | `CVE-2026-14456` (×3: `openssl`, `libcrypto3`, `libssl3`), `CVE-2026-66046`, `CVE-2026-76641` |
| Still present | 0 | — |
| Newly introduced | 0 | — |

**Final verdict:** Image `ghcr.io/sgrsaga/java-app:v1-golden-base-app` is clean and promoted to **golden base app** status. Maintain via scheduled golden-base rebuilds to prevent CVE drift.