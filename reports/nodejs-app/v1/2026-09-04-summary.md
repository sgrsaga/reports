# Remediation Summary

> Original image: `ghcr.io/sgrsaga/nodejs-app:v1`
> Final image: `ghcr.io/sgrsaga/nodejs-app:v1-golden-base-app`
> Status: `golden_base_app`

# Container Security Remediation Summary

**Image:** `ghcr.io/sgrsaga/nodejs-app:v1` → `ghcr.io/sgrsaga/nodejs-app:v1-golden-base-app`
**Final Status:** `golden_base_app`
**Iterations Run:** 1
**Base Artifact:** `cgr.dev/chainguard/node:latest` (published as `ghcr.io/sgrsaga/node:latest-golden-base`)

---

## 1. Executive Summary

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| **Overall Risk Rating** | **CRITICAL** | **NONE** | ✅ Fully remediated |
| Critical CVEs | 7 | 0 | −7 |
| High CVEs | 83 | 0 | −83 |
| **Total** | **90** | **0** | **−90 (100%)** |

The original `nodejs-app:v1` image carried an **overall risk rating of CRITICAL**, driven by 7 CRITICAL and 83 HIGH severity findings. A significant portion of these findings were unfixable at the OS layer on the incumbent Debian 12 (bookworm) base — notably the `util-linux` TOCTOU/mount family (`CVE-2026-53613`, `-76642`, `-78408/09/10`), `perl-base` regex/Archive-Tar defects, `ncurses`, `gzip`, `libacl1`, and `libsystemd0` — meaning package-level patching alone could not clear them.

The automated remediation escalated from OS patching to a **base image swap** to Chainguard's minimal, distroless-style `node` image. This **eliminated 100% of findings**, reducing the image to a clean scan of **0 CRITICAL / 0 HIGH**. **No residual risk remains** from a scanner perspective, and **no new CVEs were introduced** by the swap. The final image now achieves `golden_base_app` status.

> ⚠️ **Note:** The residual-risk, acceptance, and compensating-control sections below are retained as **standing guidance** for future scan cycles, since a `:latest` base tag will accrue new CVEs over time and re-open remediation obligations.

---

## 2. What Changed

The reduction was achieved in **1 iteration** comprising **3 validated steps**. Every step was gated by a full image rebuild, the application's own test suite, and a Trivy rescan; non-improving steps were rolled back but retained as adjudication candidates.

| Step | Strategy | Action | Result (C, H) | Verdict |
|------|----------|--------|---------------|---------|
| 1 | `os-patch` | Debian blanket upgrade in base stage | (7, 83) → (5, 71) | ✅ Kept (−14) |
| 2 | `os-patch` | Second Debian blanket upgrade | (5, 71) → (5, 71) | ↩️ Rolled back (no improvement) |
| 3 | `llm-base` | Base swap to `cgr.dev/chainguard/node:latest` | (5, 71) → (0, 0) | ✅ Kept (−76) |

**Plain-language account:**

1. **OS package upgrade (partial win).** The first pass ran a distribution-wide `apt` upgrade against the Debian base. This cleared 14 findings — chiefly the fixable `libgnutls30`, `libpam*`, `libcap2`, `gpgv`, `cross-spawn`, and `perl` (`CVE-2023-31484`) CVEs that had published fix versions. It could **not** touch the large cluster of `NO FIX` OS CVEs.

2. **OS package upgrade (redundant).** A second blanket upgrade produced no delta — the remaining OS CVEs had no available fixed packages in the Debian tree. This step was **rolled back** to avoid layer bloat while being retained for audit.

3. **Base image swap (decisive win).** The remediation engine replaced the Debian base with the Chainguard minimal `node` image. Chainguard images are built on a hardened, minimal userland (Wolfi) that **does not ship** the vulnerable `util-linux`, `perl-base`, `ncurses`, `gzip`, `libacl1`, or `libsystemd0` packages, and rebuilds first-party Node.js tooling (`tar`, `minimatch`, `brace-expansion`, `glob`, `cross-spawn`, `sigstore`, `pacote`, `ip-address`) against current, patched releases. This removed the entire remaining surface of 76 findings.

The dependency-level Node CVEs (e.g., the `tar` family `CVE-2026-59873/23745/…`, `minimatch`, `brace-expansion`, `sigstore`) were resolved because the swapped base ships patched versions of these packages in its bundled npm toolchain; no separate `package.json` bump was required in this run.

---

## 3. Remaining Risk Breakdown

**Current residual findings: 0 (CRITICAL: 0, HIGH: 0).**

There are **no OS packages with unfixed CVEs** and **no application-level CVEs** remaining in the final image. All 90 original findings are resolved; none are still present; none newly introduced.

The following table documents the **classes of risk that were remediated by the base swap** rather than by an upstream fix — this is important for forward-looking triage, because these classes are the ones most likely to reappear if the image reverts to a full Debian base or if the `:latest` tag drifts.

### 3.1 OS packages that had *no fix available* (resolved by base swap, not by patch)

| Package family | Representative CVEs | Why unfixable on Debian | How resolved |
|----------------|--------------------|-------------------------|--------------|
| `util-linux` / `libmount1` / `libblkid1` / `libuuid1` / `libsmartcols1` / `mount` / `bsdutils` | `CVE-2026-53613`, `CVE-2026-76642`, `CVE-2026-78408/09/10` | No Debian fixed package published | Package family **not present** in Chainguard minimal base |
| `perl-base` | `CVE-2026-13221`, `CVE-2026-42496`, `CVE-2026-8376`, `CVE-2026-42497`, `CVE-2026-48962`, `CVE-2026-57432/33`, `CVE-2026-9538` | No Debian fix | Perl **not present** in minimal Node base |
| `ncurses` (`libtinfo6`, `ncurses-base/bin`) | `CVE-2025-69720` | No Debian fix | Not present |
| `gzip` | `CVE-2026-41992` | No Debian fix | Not present / rebuilt |
| `libacl1` | `CVE-2026-54369` | No Debian fix | Not present |
| `libsystemd0` / `libudev1` | `CVE-2026-16742` | No Debian fix | systemd **not present** in distroless base |
| `zlib1g` | `CVE-2023-45853` | No Debian fix | Rebuilt against patched zlib |

### 3.2 Application / compiled-in CVEs (resolved by upstream-patched base toolchain)

These required an upstream release or dependency bump — delivered via the base image's refreshed npm toolchain:

| Package | CVEs | Remediation guidance (for future recurrence) |
|---------|------|----------------------------------------------|
| `tar` (node-tar) | `CVE-2026-59873/59874`, `-23745`, `-23950`, `-24842`, `-26960`, `-29786`, `-31802`, `-73566` | Pin `tar >= 7.5.21` |
| `minimatch` | `CVE-2026-26996/27903/27904` | Pin `minimatch >= 10.2.3` |
| `brace-expansion` | `CVE-2026-13149/14257/69152` | Pin `>= 5.0.9` (or backport line ≥ 2.1.4) |
| `glob` | `CVE-2025-64756` | Pin `glob >= 11.1.0` |
| `cross-spawn` | `CVE-2024-21538` | Pin `>= 7.0.5` |
| `sigstore` | `CVE-2026-48815` | Pin `>= 4.1.1` |
| `pacote` | `CVE-2026-9496` | Pin `>= 21.5.1` |
| `ip-address` | `CVE-2026-69192` | Pin `>= 10.3.1` |

**Guidance:** Keep these as `overrides`/`resolutions` in `package.json` so that a future base regression cannot silently reintroduce an old, vulnerable transitive version.

---

## 4. Risk Acceptance Template

No CVEs currently require acceptance. Retain the following template for any future finding a team elects to accept rather than remediate (e.g., a `NO FIX` OS CVE that reappears with a subsequent base change):

```
CVE: <ID>
Status: Risk Accepted
Reason: <why this is acceptable in this deployment — e.g., vulnerable code path
         not reachable; component not invoked at runtime; exploit requires local
         privileged access not available in the pod security context>
Reviewed by: <name / security team>
Review date: <YYYY-MM-DD>
Next review: <YYYY-MM-DD (review date + 90 days)>
```

**Worked example (illustrative only — not currently active):**

```
CVE: CVE-2026-53613
Status: Risk Accepted
Reason: util-linux mount TOCTOU requires local mount privileges; container runs
        as non-root with a read-only rootfs and no CAP_SYS_ADMIN, so the mount
        helper is not invokable. Not present in current base image.
Reviewed by: Container Security Team
Review date: 2026-01-15
Next review: 2026-04-15
```

---

## 5. Residual Risk Guidance — Compensating Controls

Although the current scan is clean, the following controls should be enforced as **defense-in-depth** and as **standing mitigations** for any residual OS/app CVE that reappears (especially the `NO FIX` classes above). Apply at the Kubernetes/runtime layer regardless of scan state.

### 5.1 Pod Security Context (read-only FS, non-root, dropped caps)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532          # Chainguard nonroot UID
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

- **`readOnlyRootFilesystem: true`** neutralizes the `tar`/`node-tar` arbitrary-file-overwrite and path-traversal class (`CVE-2026-23745`, `-24842`, `-26960`, `-29786`, `-31802`) by removing writable target surfaces.
- **`drop: ["ALL"]` + `allowPrivilegeEscalation: false`** removes `CAP_SYS_ADMIN`/`CAP_DAC_OVERRIDE`, defeating the `util-linux` mount TOCTOU and `libcap`/`libacl1` privilege-escalation classes.
- **`runAsNonRoot`** contains the `libsystemd0`/systemd-homed and PAM directory-traversal escalation paths.

### 5.2 seccomp / AppArmor

- Enforce `seccompProfile: RuntimeDefault` (above) to block the `mount`, `nsenter`, and namespace-manipulation syscalls central to the `util-linux` CVE family.
- Apply an AppArmor profile denying write to `/`, `/etc`, `/usr`, and mount operations:

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
```

### 5.3 Network Policy (default-deny + explicit egress)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nodejs-app-default-deny
spec:
  podSelector:
    matchLabels: { app: nodejs-app }
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector: { matchLabels: { role: gateway } }
  egress:
    - to:
        - namespaceSelector: { matchLabels: { name: platform } }
      ports:
        - { protocol: TCP, port: 443 }
```

- Restricts the DoS blast radius of the GnuTLS/DTLS (`CVE-2026-42009/42010/33845`) and `ip-address` parsing (`CVE-2026-69192`) classes by limiting who can reach the service and where it can egress.

### 5.4 mTLS Enforcement (service mesh)

- Enforce **strict mTLS** (Istio `PeerAuthentication: STRICT` or Linkerd default) so that the GnuTLS certificate-validation / authentication-bypass class (`CVE-2026-42010`, `CVE-2025-32988/32990`, `CVE-2026-48815` sigstore trust) is mitigated by the mesh's separate, independently-patched TLS stack rather than the app's linked library.

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata: { name: default }
spec:
  mtls: { mode: STRICT }
```

### 5.5 Ongoing hardening for the `:latest` base

- **Pin by digest**, not tag: replace `cgr.dev/chainguard/node:latest` with `cgr.dev/chainguard/node@sha256:<digest>` in the golden base to make rebuilds reproducible and prevent silent drift.
- **Schedule a recurring rescan** (e.g., nightly Trivy) against the published `ghcr.io/sgrsaga/node:latest-golden-base` so newly-disclosed CVEs against the `:latest` base are caught before they propagate to app images.
- **Enforce `resolutions`** for the Node dependency CVEs in §3.2 to prevent transitive regression.

---

*Report generated for the `golden_base_app` remediation run. Final image is scan-clean (0/0); all guidance above is retained for forward-looking risk management.*