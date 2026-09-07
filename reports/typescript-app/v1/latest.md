# Remediation Summary

> Original image: `ghcr.io/sgrsaga/typescript-app:v1`
> Final image: `ghcr.io/sgrsaga/typescript-app:v1-golden-base-app`
> Status: `golden_base_app`

# Container Security Remediation Summary

**Image:** `ghcr.io/sgrsaga/typescript-app:v1` → `ghcr.io/sgrsaga/typescript-app:v1-golden-base-app`
**Run status:** `golden_base_app`
**Iterations:** 1 (3 validated remediation steps)
**Scanner:** Trivy

---

## 1. Executive Summary

| Metric | Before | After | Delta |
|--------|--------|-------|-------|
| **Overall risk rating** | **CRITICAL** | **CLEAN** | ✅ |
| CRITICAL | 7 | 0 | −7 |
| HIGH | 83 | 0 | −83 |
| **Total** | **90** | **0** | **−90 (100%)** |

The initial scan of `typescript-app:v1` returned a **Critical** overall risk posture: 7 CRITICAL and 83 HIGH findings, dominated by unfixable Debian 12 (bookworm) OS-package CVEs (`util-linux`, `perl-base`, `ncurses`, `systemd`, `zlib1g`) and a large cluster of vulnerable JavaScript dependencies bundled into the runtime layer (`node-tar`, `minimatch`, `brace-expansion`, `cross-spawn`, `glob`, `sigstore`, `pacote`).

After remediation the final image scans **completely clean (0/0)**. The decisive action was a **base image swap** from a Debian-based Node image to the distroless-style **Chainguard `cgr.dev/chainguard/node:latest-dev`** base, which eliminated both the unfixable Debian OS CVEs and refreshed the bundled Node toolchain packages to non-vulnerable versions. **No residual OS or application CVEs remain**, and no new vulnerabilities were introduced. Sections 3–5 below are provided as forward-looking guidance because Chainguard images are rebuilt frequently — new CVEs *will* appear over time, and this posture must be re-validated on each rebuild.

---

## 2. What Changed

The reduction was achieved in **one automated iteration comprising three validated steps**. Every step was gated by a full image rebuild, the application's own test suite, and a Trivy rescan. Non-improving steps were rolled back but retained as adjudication candidates.

| # | Strategy | Action | Result | (CRIT, HIGH) |
|---|----------|--------|--------|--------------|
| 1 | `os-patch` | Debian blanket `apt-get upgrade` in base stage | ✅ Passed | (7, 83) → (5, 71) |
| 2 | `os-patch` | Second Debian blanket upgrade | ⏪ No improvement, rolled back | (5, 71) → (5, 71) |
| 3 | `llm-base` | **Base swap → `cgr.dev/chainguard/node:latest-dev`** | ✅ Passed | (5, 71) → **(0, 0)** |

**Plain-language account:**

1. **OS package upgrade (partial win).** A blanket Debian upgrade cleared vulnerabilities that had published fixes in bookworm-security (`libgnutls30`, `libpam-*`, `libcap2`, `gpgv`, and the perl CPAN TLS issue), reducing the count from 90 → 76. A second upgrade pass yielded nothing further — the remaining Debian CVEs were **`NO FIX`** upstream, so `apt` could not resolve them. This step was rolled back to avoid a redundant layer.

2. **Base image swap (decisive win).** Replacing the Debian-derived base with **Chainguard's minimal, continuously-patched Node base** removed the entire Debian userland responsible for the unfixable CVEs (`util-linux`/`mount`/`libblkid1`/`libmount1`/`libuuid1`/`libsmartcols1`, `perl-base`, `ncurses`, `libsystemd0`/`libudev1`, `libacl1`, `zlib1g`, `gzip`, `bsdutils`). Because Chainguard tracks upstream and ships current npm toolchain packages, it simultaneously resolved the bundled JS-package CVEs (`node-tar`, `minimatch`, `brace-expansion`, `cross-spawn`, `glob`, `ip-address`, `pacote`, `sigstore`).

**Final base artifact:**

- Selected base: `cgr.dev/chainguard/node:latest-dev`
- Published golden base: `ghcr.io/sgrsaga/node:latest-dev-golden-base`

All **90 CVEs resolved**, **0 still present**, **0 newly introduced**.

---

## 3. Remaining Risk Breakdown

**Current residual count: 0 CRITICAL / 0 HIGH.**

There are no OS-level, compiled-in, or application-level CVEs present in the final image at scan time. The subsections below are retained as **operational guidance** because the clean state is a point-in-time result against a rolling base tag.

### 3.1 OS packages with no fix available
**None present.** For historical context, the following classes were removed entirely by the base swap (they had no bookworm fix and could not have been patched in place):

| Original CVE(s) | Package family | Why it was unfixable | How it was resolved |
|-----------------|----------------|----------------------|---------------------|
| CVE-2026-53613, -76642, -78408, -78409, -78410 | `util-linux` / `mount` / `libblkid1` / `libmount1` / `libuuid1` / `libsmartcols1` / `bsdutils` | No Debian fix published | Package family absent from Chainguard base |
| CVE-2026-13221, -42496, -8376, -42497, -48962, -57432, -57433, -9538 | `perl-base` | No Debian fix published | Perl not present in runtime base |
| CVE-2025-69720 | `ncurses` (`libtinfo6`, `ncurses-base`, `ncurses-bin`) | No Debian fix published | Absent from base |
| CVE-2026-16742 | `systemd` (`libsystemd0`, `libudev1`) | No Debian fix published | Absent from base |
| CVE-2023-45853 | `zlib1g` | No Debian fix published | Superseded by Chainguard-managed zlib |
| CVE-2026-41992 | `gzip` | No Debian fix published | Absent from base |
| CVE-2026-54369 | `libacl1` | No Debian fix published | Absent from base |

### 3.2 Compiled-in / application-level CVEs
**None present.** The previously-bundled Node dependency CVEs were all resolved by the toolchain refresh in the new base. Forward-looking remediation guidance for the packages most likely to regress:

| Package | Prior CVE(s) | If it reappears — fix action |
|---------|--------------|------------------------------|
| `tar` / `node-tar` | CVE-2026-59873, -23745, -23950, -24842, -26960, -29786, -31802, -59874, -73566 | Pin `tar >= 7.5.21` in `package.json`; run `npm audit fix` and rebuild |
| `minimatch` | CVE-2026-26996, -27903, -27904 | Upgrade to `>= 10.2.3` (or patched line, e.g. `9.0.7`) |
| `brace-expansion` | CVE-2026-13149, -14257, -69152 | Upgrade to `>= 2.1.4` / `5.0.9`; use `overrides` for transitive pins |
| `cross-spawn` | CVE-2024-21538 | Upgrade to `>= 7.0.5` |
| `glob` | CVE-2025-64756 | Upgrade to `>= 11.1.0` (or `10.5.0`) |
| `ip-address` | CVE-2026-69192 | Upgrade to `>= 10.3.1` |
| `pacote` | CVE-2026-9496 | Upgrade to `>= 21.5.1` |
| `sigstore` | CVE-2026-48815 | Upgrade to `>= 4.1.1` |

> **Guidance:** Enforce these via `package.json` `overrides` / `resolutions` and commit a `package-lock.json`, so a base rebuild cannot silently reintroduce a downgraded transitive dependency.

---

## 4. Risk Acceptance Template

No CVEs currently require acceptance. Use the template below if a future rescan surfaces a `NO FIX` finding the team elects to accept:

```
CVE: <ID>
Status: Risk Accepted
Reason: <why this is acceptable in this deployment — e.g. affected code path
         not reachable; component not invoked at runtime; exploit requires
         local privileged access not present in this workload>
Reviewed by: <name / security owner>
Review date: <YYYY-MM-DD>
Next review: <YYYY-MM-DD (review date + 90 days)>
```

Example (illustrative, for a hypothetical future unfixable OS CVE):

```
CVE: CVE-XXXX-XXXXX
Status: Risk Accepted
Reason: Vulnerable binary (e.g. mount) is not invoked by the application;
        container runs non-root with read-only rootfs, removing the local
        privilege-escalation precondition. No fix available upstream.
Reviewed by: A. Engineer (Container Security)
Review date: 2026-01-15
Next review: 2026-04-15
```

---

## 5. Residual Risk Guidance — Compensating Controls

Even with a clean scan, apply defense-in-depth so that any *future* CVE surfacing in the rolling base tag is contained until the next rebuild.

### 5.1 Pod / container hardening (Kubernetes `securityContext`)
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532            # Chainguard 'nonroot' UID
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```
- **`readOnlyRootFilesystem: true`** neutralizes the class of *arbitrary file write / overwrite* CVEs (the historical `node-tar` and `util-linux` findings). Mount an `emptyDir` for any writable scratch path the app genuinely needs.
- **`drop: ALL` + `allowPrivilegeEscalation: false`** removes the precondition for the local privilege-escalation CVEs (`libpam`, `libcap`, `systemd-homed`, `libacl`).

### 5.2 seccomp / AppArmor
- Ship a **custom seccomp profile** (start from `RuntimeDefault`, tighten to the syscall set the Node process actually uses) to block `mount`/`nsenter`-style syscalls exploited by the former `util-linux` CVEs.
- Where AppArmor is available, apply a **restrictive profile** denying write to `/proc`, `/sys`, and binary paths.

### 5.3 Network policies (default-deny)
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: typescript-app-default-deny
spec:
  podSelector:
    matchLabels: { app: typescript-app }
  policyTypes: ["Ingress", "Egress"]
  # Add explicit allow rules only for required peers/ports below.
```
- Restrict egress to only the endpoints the app requires. This limits the impact of the former **DoS / parsing CVEs** (`minimatch`, `brace-expansion`, `ip-address`, GnuTLS) by controlling who can send crafted input.

### 5.4 mTLS enforcement
- Enforce **mTLS for all service-to-service traffic** via a service mesh (Istio `PeerAuthentication: STRICT`, or Linkerd). This provides an independent authentication layer that mitigates the historical **GnuTLS authentication-bypass / cert-validation** class (CVE-2026-42010, CVE-2026-48815) regardless of the in-image TLS library state.

### 5.5 Continuous verification (essential for a rolling base)
- **Re-scan on every rebuild.** Because `latest-dev` is a moving tag, pin to a **digest** (`cgr.dev/chainguard/node@sha256:...`) for reproducible deployments and re-run Trivy in CI before promotion.
- **Fail the pipeline** on any new CRITICAL/HIGH, and regenerate this before/after report per run.
- **Sign and attest** the golden base (`ghcr.io/sgrsaga/node:latest-dev-golden-base`) and verify signatures at admission (e.g. Sigstore/cosign + Kyverno policy).

---

### Sign-off

| Field | Value |
|-------|-------|
| Final image | `ghcr.io/sgrsaga/typescript-app:v1-golden-base-app` |
| Final base | `cgr.dev/chainguard/node:latest-dev` (`ghcr.io/sgrsaga/node:latest-dev-golden-base`) |
| Result | **0 CRITICAL / 0 HIGH — 90/90 resolved, 0 introduced** |
| Recommendation | **Promote.** Pin base by digest and enforce Section 5 controls + per-rebuild rescan. |