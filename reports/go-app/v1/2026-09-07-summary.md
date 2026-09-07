# Remediation Summary

> Original image: `ghcr.io/sgrsaga/go-app:v1`
> Final image: `ghcr.io/sgrsaga/go-app:v1-golden-base-app`
> Status: `golden_base_app`

# Container Security Remediation Report

**Image:** `ghcr.io/sgrsaga/go-app:v1` → `ghcr.io/sgrsaga/go-app:v1-golden-base-app`
**Run status:** `golden_base_app`
**Iterations:** 1
**Base artifact:** `cgr.dev/chainguard/go:latest` (published as `ghcr.io/sgrsaga/go:latest-golden-base`)

---

## 1. Executive Summary

| Metric | Before | After | Delta |
|--------|-------:|------:|------:|
| **Overall risk rating** | **CRITICAL** | **CLEAN / PASS** | — |
| Total findings | 483 | 0 | −483 (−100%) |
| CRITICAL | 22 | 0 | −22 |
| HIGH | 461 | 0 | −461 |
| MEDIUM | 0 (not reported) | 0 | — |

**Assessment.** The original image carried a **Critical** risk posture, dominated by an outdated Go toolchain (`stdlib v1.21.13`) that compiled ~440 known Go standard-library CVEs directly into the binary, plus a vulnerable Alpine OS layer (OpenSSL `libssl3`/`libcrypto3` `3.3.1-r3`, `musl 1.2.5-r0`, `zlib 1.3.1-r1`). After a single successful remediation iteration the scanner reports **zero CRITICAL and zero HIGH findings**. The decisive action was a **base image swap** to the Chainguard Go image, which simultaneously refreshed the OS distribution and the Go toolchain used to (re)build the application. **No residual CRITICAL/HIGH vulnerabilities remain** at the time of this scan. Remaining risk is therefore limited to *future* CVE disclosures and to operational/runtime hardening (covered in §5), not to any currently-known unpatched finding.

> ⚠️ **Caveat:** "0 findings" reflects the current vulnerability database at scan time. A minimal/distroless base substantially shrinks attack surface but does **not** make the image permanently CVE-free. Continuous rescanning remains mandatory (see §5).

---

## 2. What Changed

The reduction from **483 → 0** was achieved through an automated, test-gated remediation loop. Every step was validated by a **full rebuild → application test suite → Trivy rescan** cycle; non-improving steps were rolled back but retained as adjudication candidates.

### Remediation trail

| Step | Strategy | Action | Result (C,H) | Kept? |
|------|----------|--------|--------------|-------|
| 1 | `os-patch` | Alpine blanket upgrade in base stage | (22, 461) → (20, 440) | ✅ improved |
| 2 | `os-patch` | Alpine blanket upgrade (repeat) | (20, 440) → (20, 440) | ❌ no gain, rolled back |
| 3 | `llm-base` | Swap base to `cgr.dev/chainguard/go:latest` | (20, 440) → **(0, 0)** | ✅ **selected** |

### Plain-language account

1. **OS package upgrade (partial win).** Upgrading Alpine packages in place cleared 2 CRITICAL and 21 HIGH findings — mostly the OpenSSL, `musl`, and `zlib` OS-level CVEs that had a fix version available in Alpine repos. This could **not** touch the ~440 Go `stdlib` CVEs, because those are **compiled into the application binary** and are only fixed by rebuilding with a newer Go toolchain.

2. **Base image swap (decisive win).** Replacing the base with **Chainguard's Go image** delivered two effects at once:
   - A **modern, continuously-patched minimal/distroless OS layer** (no `musl`/OpenSSL/`zlib` at the vulnerable versions).
   - A **current Go toolchain**, so the rebuilt binary embeds a patched `stdlib`, eliminating every `CVE-2024-*`, `CVE-2025-*`, and `CVE-2026-*` Go standard-library finding (net/url, crypto/x509, crypto/tls, net/http, html/template, encoding/*, mime, net/mail, os.Root, etc.).

**Effort:** 1 iteration, 3 evaluated steps (2 retained as effective, 1 rolled back). The golden base is now republished as `ghcr.io/sgrsaga/go:latest-golden-base` for reuse across other services.

---

## 3. Remaining Risk Breakdown

### OS packages with no fix available
**None.** All OS-level findings (`libssl3`, `libcrypto3`, `musl`, `musl-utils`, `zlib`) were resolved by the base swap. The Chainguard base does not ship these packages at vulnerable versions.

### Compiled-in / application-level CVEs
**None currently present.** The entire block of Go `stdlib` CVEs was resolved by rebuilding on the newer toolchain. For reference, the classes that were compiled-in and are now fixed:

| CVE class | Package/module | Prior fix requirement |
|-----------|----------------|-----------------------|
| `CVE-2025-68121` (crypto/tls cert validation) | `stdlib` | Go rebuild ≥ 1.24.13 / 1.25.7 |
| `CVE-2026-27145`, `-32280/81`, `-56862` (crypto/x509, crypto/tls DoS) | `stdlib` | Go rebuild |
| `CVE-2025-61726`, `-56860`, `-25679` (net/url) | `stdlib` | Go rebuild |
| `CVE-2026-33814`, `-56853` (net/http, HTTP/2) | `stdlib` | Go rebuild |
| `CVE-2026-56858` (html/template XSS) | `stdlib` | Go rebuild |
| `CVE-2026-56859`, `-33818` (encoding/xml, encoding/asn1) | `stdlib` | Go rebuild |
| `CVE-2026-39820`, `-42499` (net/mail) | `stdlib` | Go rebuild |
| `CVE-2026-39822` (os.Root symlink) | `stdlib` | Go rebuild |

> **Guidance for the future:** Because these are *statically linked into the binary*, they will **not** be fixed by patching the OS layer. Whenever a new Go `stdlib` CVE is disclosed, the remediation is always: **bump the builder Go version and rebuild** — not `apk upgrade`. Keep the CI pipeline pinned to "latest patched Go minor/patch" and rescan on every release.

**Net residual known-vulnerability risk: 0 CRITICAL / 0 HIGH.**

---

## 4. Risk Acceptance Template

No CVEs currently require acceptance. **Retain this template** for any future finding that cannot be immediately remediated (e.g., a newly disclosed `stdlib` CVE before the toolchain patch lands, or an OS CVE with no upstream fix):

```
CVE: <ID>
Status: Risk Accepted
Reason: <why this is acceptable in this deployment — e.g., vulnerable code path
        not reachable; feature not compiled/enabled; mitigated by network
        isolation; no fix available upstream and exploitation requires
        conditions not present in this environment>
Reviewed by: <name / team>
Review date: <YYYY-MM-DD>
Next review: <YYYY-MM-DD (review date + 90 days)>
```

Example (illustrative — not currently active):
```
CVE: CVE-2026-XXXXX
Status: Risk Accepted
Reason: Affects net/mail parser; service does not parse untrusted email input.
        No inbound path reaches the vulnerable function. Fix expected in next
        Go patch release; will be picked up automatically on next rebuild.
Reviewed by: A. Engineer, Platform Security
Review date: 2026-06-01
Next review: 2026-08-30
```

---

## 5. Residual Risk Guidance (Compensating Controls)

Even at 0 known CVEs, apply defense-in-depth so the workload stays resilient to **future** disclosures and runtime attacks.

### Runtime hardening (Pod/Container)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532                # Chainguard 'nonroot' UID
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true    # image already minimal; make FS immutable
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault          # or a custom profile scoped to the app's syscalls
```

- **Read-only root filesystem** — mount `emptyDir` only for required writable paths (`/tmp`); blocks tampering/persistence.
- **Drop all Linux capabilities** — the Go service needs none by default.
- **seccomp `RuntimeDefault`** (or a tighter custom profile) — limits the syscall surface exploitable by a memory-safety bug in a future `stdlib` CVE.
- **AppArmor / SELinux** — apply a confinement profile restricting file and network access to only what the app requires.

### Network controls

- **NetworkPolicy** — default-deny ingress/egress; explicitly allow only required peers and ports. This is a strong mitigation for the DoS-class CVEs (net/url, net/http, crypto/tls) that dominated the original findings — restricting who can send traffic reduces exploitability.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: go-app-default-deny }
spec:
  podSelector: { matchLabels: { app: go-app } }
  policyTypes: ["Ingress", "Egress"]
  # add explicit allow rules for known peers/ports below
```

- **mTLS enforcement** — terminate/enforce mutual TLS at a service mesh (Istio/Linkerd) or gateway. Given the historical `crypto/tls` cert-validation CVE (`CVE-2025-68121`), enforcing mesh-level identity verification adds an independent trust layer beyond the app's own TLS stack.

### Supply-chain & continuous assurance

- **Rescan on a schedule** (daily) and on every build — "0 today" ≠ "0 tomorrow."
- **Pin and auto-bump the builder** — keep the Go builder at the latest patched release so newly disclosed `stdlib` CVEs are cleared by the next rebuild automatically.
- **Reuse the golden base** — standardize other Go services on `ghcr.io/sgrsaga/go:latest-golden-base` and rebuild them from it.
- **Sign & attest** — generate SBOM and sign images (cosign) so the golden base's provenance is verifiable.
- **Admission control** — enforce the above `securityContext` and image-provenance requirements via a policy engine (Kyverno/OPA Gatekeeper) so non-compliant images cannot deploy.

---

*Report generated for automated multi-iteration remediation run. Final image `ghcr.io/sgrsaga/go-app:v1-golden-base-app` passed rebuild, application test suite, and Trivy rescan with 0 CRITICAL / 0 HIGH findings.*