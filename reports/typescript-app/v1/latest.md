# Remediation Summary

> Original image: `ghcr.io/sgrsaga/typescript-app:v1`
> Final image: `ghcr.io/sgrsaga/typescript-app:v1-golden-base-app`
> Status: `golden_base_app`

# Container Security Remediation Summary

**Image:** `ghcr.io/sgrsaga/typescript-app:v1` → `ghcr.io/sgrsaga/typescript-app:v1-golden-base-app`
**Status:** `golden_base_app`
**Iterations run:** 1
**Report generated for:** Multi-iteration automated remediation run

---

## 1. Executive Summary

| Phase | Total | CRITICAL | HIGH | Overall Risk Rating |
|-------|-------|----------|------|---------------------|
| **Before** | 90 | 7 | 83 | **CRITICAL** |
| **After** | 0 | 0 | 0 | **CLEAN / LOW** |

The remediation run achieved a **complete elimination** of all detected CRITICAL and HIGH severity vulnerabilities — a **100% reduction (90 → 0)**. The original image, built on Debian 12 (bookworm) with an aged Node.js toolchain, carried a heavy load of OS-level CVEs (notably `util-linux`, `perl-base`, `libgnutls30`, and `systemd` components) plus a cluster of application-tier JavaScript dependency CVEs (`tar`, `minimatch`, `brace-expansion`, `sigstore`, `pacote`, etc.).

Critically, **41 of the 90 findings had no upstream Debian fix available** (`NO FIX`), meaning an in-place patch-only strategy could never have reached a clean state. The decisive remediation was a **base image swap to Chainguard's hardened, minimal Node.js image** (`cgr.dev/chainguard/node:latest-dev`), which eliminated the entire Debian package surface and shipped current, low-CVE dependency versions.

**What remains:** Nothing in the scanned SBOM. However, "0 findings" reflects the state of the scanner's vulnerability database at scan time and the reduced package surface — it does not guarantee immunity to future disclosures. Ongoing monitoring and the residual-risk controls in Sections 4–5 remain mandatory.

---

## 2. What Changed

The reduction was achieved over **1 iteration comprising 3 validated remediation steps**. Every step was gated by a full rebuild, the application's own test suite, and a Trivy rescan. Non-improving steps were rolled back but retained as adjudication candidates.

| Step | Type | Action | Result (C, H) | Verdict |
|------|------|--------|---------------|---------|
| 1 | `os-patch` | Debian blanket upgrade in base stage | (7, 83) → (5, 71) | ✅ Kept (partial improvement) |
| 2 | `os-patch` | Repeat Debian blanket upgrade | (5, 71) → (5, 71) | ↩️ Rolled back (no improvement) |
| 3 | `llm-base` | **Base swap → `cgr.dev/chainguard/node:latest-dev`** | (5, 71) → (0, 0) | ✅ Kept (decisive) |

**Plain-language account:**

1. **OS package upgrade (partial win).** A blanket `apt-get upgrade` in the base stage cleared 2 CRITICALs and 12 HIGHs — primarily the GnuTLS, PAM, and libcap CVEs that *did* have Debian fixes. This proved that patch-only remediation plateaued quickly because ~41 findings were `NO FIX` in the Debian stream.

2. **Second upgrade attempt (no-op, rolled back).** A repeated upgrade produced zero delta, confirming the Debian package tree was fully patched to the extent upstream allowed. This step was reverted to avoid dead layers.

3. **Base image swap (decisive win).** Replacing the Debian-based Node image with **Chainguard's minimal, distroless-style hardened Node image** eliminated the remaining 5 CRITICAL and 71 HIGH findings in one move. Chainguard images:
   - Ship a **drastically smaller package surface** (no `util-linux`, `perl-base`, `systemd`, `ncurses`, `gzip`, etc. — the source of the bulk of `NO FIX` CVEs).
   - Track **current dependency versions**, resolving the `tar`, `minimatch`, `brace-expansion`, `sigstore`, `pacote`, `glob`, `cross-spawn`, and `ip-address` application-tier CVEs.

**Base artifact published:**
- Final base: `cgr.dev/chainguard/node:latest-dev`
- Republished as: `ghcr.io/sgrsaga/node:latest-dev-golden-base`

---

## 3. Remaining Risk Breakdown

**Current scan state: 0 CRITICAL / 0 HIGH — no findings present.**

There is no residual OS-package or application-level CVE to enumerate in the post-remediation SBOM. For completeness and future-proofing, the following captures the *classes* of risk that were eliminated and where they will most likely re-emerge:

### 3.1 OS packages that had NO FIX in the original image (now eliminated by base swap)

These were **unfixable in Debian** and only disappeared because the packages themselves are absent from the Chainguard base:

| Package family | Representative CVEs | Why unfixable before | Status now |
|----------------|---------------------|----------------------|------------|
| `util-linux` / `mount` / `libblkid1` / `libmount1` / `libuuid1` / `libsmartcols1` / `bsdutils` / `util-linux-extra` | CVE-2026-53613, -76642, -78408, -78409, -78410 | No Debian fix released | Removed (not present in base) |
| `perl-base` | CVE-2026-13221, -42496, -8376, -42497, -48962, -57432, -57433, -9538 | No Debian fix released | Removed (Perl not shipped) |
| `libsystemd0` / `libudev1` | CVE-2026-16742 | No Debian fix released | Removed |
| `libtinfo6` / `ncurses-*` | CVE-2025-69720 | No Debian fix released | Removed |
| `gzip` | CVE-2026-41992 | No Debian fix released | Removed |
| `libacl1` | CVE-2026-54369 | No Debian fix released | Removed |
| `zlib1g` | CVE-2023-45853 | No Debian fix released | Removed / superseded |

### 3.2 Application-level / compiled-in CVEs (now resolved by current dependency versions)

These required an **upstream release or version bump** and are resolved in the Chainguard base's dependency tree. **Remediation guidance is retained here because they will reappear if a build pins these packages back to vulnerable versions:**

| Package | CVEs | Fix guidance (enforce as floor versions) |
|---------|------|------------------------------------------|
| `tar` (node-tar) | CVE-2026-59873, -23745, -23950, -24842, -26960, -29786, -31802, -59874, -73566 | Pin `tar >= 7.5.21` |
| `minimatch` | CVE-2026-26996, -27903, -27904 | Pin `minimatch >= 10.2.3` |
| `brace-expansion` | CVE-2026-13149, -14257, -69152 | Pin `brace-expansion >= 5.0.9` (or backport line `2.1.4`) |
| `sigstore` | CVE-2026-48815 | Pin `sigstore >= 4.1.1` |
| `pacote` | CVE-2026-9496 | Pin `pacote >= 21.5.1` |
| `glob` | CVE-2025-64756 | Pin `glob >= 11.1.0` |
| `cross-spawn` | CVE-2024-21538 | Pin `cross-spawn >= 7.0.5` |
| `ip-address` | CVE-2026-69192 | Pin `ip-address >= 10.3.1` |

> **Action:** Add these as minimum-version constraints / `overrides` in `package.json` (or `resolutions` for Yarn) so a future transitive downgrade is blocked at install time.

---

## 4. Risk Acceptance Template

No open CVEs currently require acceptance. Retain the template below for any CVE that re-emerges in a future scan and the team elects to accept rather than remediate:

```
CVE: <ID>
Status: Risk Accepted
Reason: <why this is acceptable in this deployment — e.g., affected code path not
         reachable, package present but binary not invoked, exploit requires local
         privileged access not granted in this runtime>
Reviewed by: <name>
Review date: <YYYY-MM-DD>
Next review: <YYYY-MM-DD + 90 days>
```

**Example (illustrative, for a hypothetical returning `NO FIX` OS CVE):**

```
CVE: CVE-2026-53613
Status: Risk Accepted
Reason: util-linux mount TOCTOU requires local invocation of mount(8); container
        runs non-root with read-only rootfs and no mount capability (CAP_SYS_ADMIN
        dropped). Attack path not reachable in this deployment.
Reviewed by: <security-engineer>
Review date: 2026-XX-XX
Next review: 2026-XX-XX (+90 days)
```

---

## 5. Residual Risk Guidance — Compensating Controls

Even at 0 findings, defense-in-depth must be enforced to contain future zero-days and any transitively reintroduced CVEs. Apply the following at deploy time:

### 5.1 Pod / Container hardening (Kubernetes `securityContext`)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532          # Chainguard 'nonroot' UID
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

- **`readOnlyRootFilesystem: true`** — directly neutralizes the *class* of arbitrary-file-write CVEs seen in `tar`/`node-tar` (CVE-2026-23745, -24842, -26960, -29786, -31802) by denying writes outside declared `emptyDir`/`tmpfs` mounts.
- **`capabilities: drop: ["ALL"]`** — removes `CAP_SYS_ADMIN`, neutralizing the `util-linux` mount/nsenter privilege-escalation classes.
- **`allowPrivilegeEscalation: false`** + **`runAsNonRoot`** — blocks the PAM (CVE-2025-6020) and libcap (CVE-2026-4878) escalation patterns.

### 5.2 Seccomp / AppArmor

- Enforce `seccompProfile.type: RuntimeDefault` (above) at minimum; author a **custom seccomp profile** allow-listing only syscalls the Node app requires.
- Where AppArmor is available, attach a confinement profile denying `mount`, `ptrace`, and raw socket operations.

### 5.3 Network policy (default-deny + explicit allow)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: typescript-app-default-deny
spec:
  podSelector:
    matchLabels: { app: typescript-app }
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector: { matchLabels: { role: api-gateway } }
      ports:
        - { protocol: TCP, port: 8080 }
  egress:
    - to:
        - namespaceSelector: { matchLabels: { name: data } }
      ports:
        - { protocol: TCP, port: 5432 }
```

- Restricts egress to reduce blast radius of the `ip-address` parsing (CVE-2026-69192) and `sigstore` certificate-validation (CVE-2026-48815) risk classes, should they recur.

### 5.4 mTLS enforcement (service mesh)

- Enforce **STRICT mTLS** for all service-to-service traffic (e.g., Istio `PeerAuthentication mode: STRICT` or Linkerd default). This compensates for the GnuTLS authentication-bypass / policy-bypass classes (CVE-2026-42010, -3833) by moving trust verification to the mesh proxy layer rather than in-app TLS libraries.

### 5.5 Continuous verification

- **Re-scan on every build and on a nightly schedule** — a clean scan today reflects only the current CVE database. Set CI to fail on any new CRITICAL/HIGH.
- **Pin the base by digest**, not `:latest-dev`, and promote digest bumps through the same test-suite + rescan gate used in this run.
- **Enforce the floor versions** from §3.2 via lockfile constraints to prevent silent dependency downgrades.

---

### Sign-off

| Item | Value |
|------|-------|
| Original image | `ghcr.io/sgrsaga/typescript-app:v1` |
| Remediated image | `ghcr.io/sgrsaga/typescript-app:v1-golden-base-app` |
| Golden base | `cgr.dev/chainguard/node:latest-dev` → `ghcr.io/sgrsaga/node:latest-dev-golden-base` |
| Net result | **90 → 0** (7 CRITICAL, 83 HIGH eliminated) |
| Newly introduced | 0 |
| Recommendation | **Approve for promotion**, subject to digest-pinning and residual controls in §5 |