# Remediation Summary

> Original image: `ghcr.io/sgrsaga/pr-demo-app:v1`
> Final image: `ghcr.io/sgrsaga/pr-demo-app:v1-golden-base-app`
> Status: `golden_base_app`

# Container Security Remediation Report

**Image:** `ghcr.io/sgrsaga/pr-demo-app`
**Original tag:** `v1`
**Remediated tag:** `v1-golden-base-app`
**Run status:** `golden_base_app`
**Iterations:** 1 (multi-step base-selection trail)

---

## 1. Executive Summary

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| **CRITICAL** | 22 | 0 | −22 (−100%) |
| **HIGH** | 1718 | 0 | −1718 (−100%) |
| **TOTAL** | 1740 | 0 | −1740 (−100%) |
| **Overall Risk Rating** | 🔴 **Critical** | 🟢 **Clean** | Fully remediated |

**Assessment:** The image moved from a **Critical** risk posture — driven overwhelmingly by a stale Debian 12 base (`linux-libc-dev` 6.1.38-4 alone accounted for the vast majority of the 1718 HIGH findings) plus outdated OpenSSL, GnuTLS, krb5, expat, Python `perl-base`, and toolchain packages — to a **fully clean** scan with **zero** CRITICAL or HIGH vulnerabilities. The decisive move was **not** patching-in-place (which plateaued) but a **structural rebuild** onto a minimal, continuously-maintained distroless base image (`cgr.dev/chainguard/python:latest-dev`) using a multi-stage build. This eliminated the entire kernel-headers CVE class (which is noise for a non-kernel userland container but inflates scanner counts) and every OS-level finding in one step.

**What remains:** Nothing at CRITICAL/HIGH severity in the final scan. However, because the golden base is a *rolling* tag (`latest-dev`), residual risk is now **operational** (base drift over time) rather than **inventory** (vulnerable packages present today). See §5.

---

## 2. What Changed

The reduction was achieved in **three effective steps** (plus one no-op and one failed experiment that were rolled back but retained for adjudication):

| Step | Technique | Result | (CRIT, HIGH) |
|------|-----------|--------|--------------|
| 1 | **OS package upgrade** — Debian blanket `apt upgrade` in base stage | ✅ Applied | (22, 1718) → (6, 273) |
| 2 | OS package upgrade (repeat) | ⏸️ No improvement — rolled back | (6, 273) → (6, 273) |
| 3 | **Base swap** to `cgr.dev/chainguard/python:latest-dev` (single-stage) | ❌ Build/test failed (`pytest: not found` — PATH/test-dep issue) | rolled back |
| 4 | **Restructure** — multi-stage: builder `python:3.11.4-slim` + runtime `cgr.dev/chainguard/python:latest-dev` | ✅ Applied | (6, 273) → **(0, 0)** |

**Plain-language narrative:**

1. **Patch-in-place first.** A Debian blanket upgrade knocked out ~85% of findings immediately (kernel-header CVEs with published fix versions, OpenSSL, GnuTLS, krb5, expat, glibc, etc.). A second upgrade pass produced no further gain — the remaining 6 CRITICAL / 273 HIGH were pinned to a Debian base that had **no upstream fixes available** for those specific CVEs.

2. **Base swap attempt.** A naive single-stage swap to Chainguard failed CI because the distroless runtime lacked the test tooling on `PATH` — a valid guardrail catch, not a security regression.

3. **Restructure to the win.** A **multi-stage build** (fat `python:3.11.4-slim` builder for compilation/tests, minimal Chainguard runtime for the final artifact) resolved the build/test failure *and* dropped the scan to **zero**. Chainguard images ship a minimal, hardened, continuously-rebuilt package set — the entire `linux-libc-dev` header package (source of nearly all HIGH findings) simply does not exist in the runtime layer, and remaining userland libs are current.

**Net:** 1740 findings resolved, 0 still present, 0 newly introduced. Every applied step was validated by full rebuild + application test suite + Trivy rescan.

---

## 3. Remaining Risk Breakdown

**Final scan: 0 CRITICAL / 0 HIGH.** There are **no vulnerabilities with fixes-unavailable remaining in the shipped image.**

This is worth stating precisely, because the *pre-remediation* inventory contained two important classes that a reviewer should understand were **eliminated by base replacement, not by patching**:

### 3a. OS packages that had NO FIX in the original Debian base (now gone)

In the original image these were unfixable in-place and would have blocked a patch-only strategy. They are resolved because the packages are **absent or replaced** in the Chainguard runtime:

| Example CVE | Package | Original Fix Status | Resolution |
|-------------|---------|--------------------|------------|
| `CVE-2023-45853` | `zlib1g` | NO FIX | Package replaced in golden base |
| `CVE-2025-7458`, `CVE-2026-11822`, `CVE-2026-11824` | `libsqlite3-0` | NO FIX | Not present in runtime |
| `CVE-2026-13221`, `CVE-2026-8376`, `CVE-2026-42496`, `CVE-2026-9538`, `CVE-2026-48962` | `perl-base` | NO FIX | Perl not shipped in runtime |
| `CVE-2025-69720` | `ncurses*` | NO FIX | Not present in runtime |
| `CVE-2026-53613`, `CVE-2026-76642`, `CVE-2026-78408/9/10` | `util-linux` family | NO FIX | Not present in runtime |
| `CVE-2026-41992` | `gzip` | NO FIX | Not present / replaced |
| `CVE-2026-54369` | `libacl1` | NO FIX | Not present / replaced |
| `CVE-2016742` | `systemd`/`libudev1` | NO FIX | No systemd in distroless runtime |
| ~hundreds of | `linux-libc-dev` (kernel headers) | NO FIX for many | Header package absent in runtime |

> ⚠️ **Important nuance:** A large number of the `linux-libc-dev` findings are **kernel header** CVEs. A userland container **does not run its own kernel** — it uses the host's. These findings are *scanner inventory artifacts* and were never runtime-exploitable from inside this container. Removing the package (via the minimal base) is the correct hygiene fix and also silences the false noise.

### 3b. Application / compiled-in CVEs

The original image carried Python-ecosystem findings that required dependency upgrades:

| CVE | Component | Original Fix | Guidance (already applied) |
|-----|-----------|-------------|---------------------------|
| `CVE-2024-6345` | `setuptools` | 70.0.0 | Pin `setuptools>=78.1.1` |
| `CVE-2025-47273` | `setuptools` | 78.1.1 | Pin `setuptools>=78.1.1` |
| `CVE-2026-24049` | `wheel` | 0.46.2 | Pin `wheel>=0.46.2` |

**These are resolved.** Going forward, keep these pins in `requirements.txt`/`constraints.txt` so a future base with older build tooling cannot silently reintroduce them.

---

## 4. Risk Acceptance Template

No CVEs currently require acceptance (scan is clean). Retain this template for any future finding that lacks an upstream fix at the next rescan:

```
CVE: <ID>
Status: Risk Accepted
Reason: <e.g. kernel-header CVE not reachable in userland container / no upstream
         fix available / component not invoked at runtime / mitigated by control X>
Component: <package@version>
Severity: <CRITICAL|HIGH|MEDIUM>
Fix available: <yes-but-deferred | no>
Compensating controls: <list, cross-ref §5>
Reviewed by: <name>
Review date: <YYYY-MM-DD>
Next review: <YYYY-MM-DD (review date + 90 days)>
```

**Pre-filled example** for the kernel-header class, should any reappear:

```
CVE: CVE-2026-XXXXX
Status: Risk Accepted
Reason: linux-libc-dev kernel-header CVE. Container runs on host kernel; the
        vulnerable code path is not present in the container's userland runtime.
        Header package retained only if a build dependency requires it.
Component: linux-libc-dev@6.1.38-4
Severity: HIGH
Fix available: no (Debian, at scan time)
Compensating controls: seccomp default profile, non-root user, read-only rootfs
Reviewed by: <name>
Review date: <YYYY-MM-DD>
Next review: <+90 days>
```

---

## 5. Residual Risk Guidance & Compensating Controls

Although the image is clean **today**, the golden base is a **rolling tag** (`latest-dev`) and the deployment should assume future CVEs will surface. Apply defense-in-depth:

### 5.1 Base image hygiene (highest leverage)
- **Pin by digest**, not `latest-dev`, in production: `cgr.dev/chainguard/python@sha256:...`. Promote new digests through CI after rescan.
- **Rescan on a schedule** (daily Trivy in CI/registry) — the value of a rolling minimal base only materializes if you rebuild regularly.
- Consider the **non-`-dev` variant** for the final runtime if the app does not need a shell/package manager at runtime — it removes even more surface than `latest-dev`.

### 5.2 Runtime hardening (Kubernetes `securityContext`)
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532            # Chainguard nonroot UID
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```
- **`readOnlyRootFilesystem: true`** neutralizes the entire path-traversal / file-modification CVE class (e.g. the former `setuptools`, `perl-Archive-Tar`, `util-linux` mount TOCTOU findings) even if reintroduced. Mount an `emptyDir` for any writable scratch paths.
- **`drop: ALL` capabilities** blunts privilege-escalation CVEs (the former `libcap`, `linux-pam`, kernel priv-esc classes).

### 5.3 Mandatory Access Control
- Ship a **seccomp** `RuntimeDefault` profile (above) — blocks obscure syscalls exercised by many kernel/driver CVEs.
- Apply an **AppArmor** (or SELinux) profile restricting the container to its expected file and network footprint.

### 5.4 Network controls
- Default-deny **NetworkPolicy**; allow only required ingress/egress:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: pr-demo-app-default-deny }
spec:
  podSelector: { matchLabels: { app: pr-demo-app } }
  policyTypes: ["Ingress", "Egress"]
  ingress: [...]   # only from the app's front door
  egress:  [...]   # only to required services/DNS
```
- Enforce **mTLS** via a service mesh (Istio/Linkerd) for all east-west traffic — mitigates the TLS/crypto CVE class (former OpenSSL/GnuTLS/krb5 findings) by constraining who can even initiate a TLS handshake.

### 5.5 Supply-chain assurance
- Generate and store an **SBOM** at build; sign the image (**cosign**) and enforce signature verification at admission.
- Gate deployments on a **Trivy policy**: fail build on any new CRITICAL/HIGH with a fix available.

---

### Appendix — Remediation trail summary
- **1740** findings resolved · **0** still present · **0** newly introduced
- Effective path: `os-patch` (22→6 CRIT) → `restructure` to multi-stage with Chainguard runtime (→0)
- Final base artifact: `cgr.dev/chainguard/python:latest-dev` *(pin to digest for production)*