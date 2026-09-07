# Remediation Summary

> Original image: `ghcr.io/sgrsaga/nodejs-app:v1`
> Final image: `ghcr.io/sgrsaga/nodejs-app:v1-golden-base-app`
> Status: `golden_base_app`

# Container Security Remediation Summary

**Image:** `ghcr.io/sgrsaga/nodejs-app`
**Original tag:** `v1` → **Final tag:** `v1-golden-base-app`
**Status:** `golden_base_app` ✅
**Iterations run:** 1 (3 discrete remediation steps)
**Base artifact:** `cgr.dev/chainguard/node:latest` → published as `ghcr.io/sgrsaga/node:latest-golden-base`

---

## 1. Executive Summary

| Metric | Before | After | Delta |
|--------|--------|-------|-------|
| **Overall risk rating** | **CRITICAL** | **PASS / Minimal** | ▼ |
| Total findings | 90 | 0 | **−90 (100%)** |
| CRITICAL | 7 | 0 | −7 |
| HIGH | 83 | 0 | −83 |
| MEDIUM | 0 | 0 | 0 |

The image began in a **Critical** posture: 7 CRITICAL and 83 HIGH findings, dominated by unfixable Debian 12 (`bookworm`) OS-level CVEs (util-linux, ncurses, perl-base, systemd, gzip, libacl1) and a cluster of application-level Node.js dependency CVEs (`tar`, `minimatch`, `brace-expansion`, `glob`, `cross-spawn`, `sigstore`, `pacote`, `ip-address`).

Remediation achieved a **complete elimination of all 90 findings**. The decisive action was a **base image swap** from the Debian-based Node runtime to the distroless, minimal-CVE **Chainguard Node** image. This removed the entire Debian package surface (including the many `NO FIX` OS CVEs that a package upgrade could never resolve) and shipped current, patched versions of the bundled Node tooling.

**What remains:** Nothing in the current scan. However, "zero findings" reflects a point-in-time scan against a rolling `:latest` base. The residual risk is **operational, not static** — the pinned base must be re-pulled, re-scanned, and re-published on a cadence, since new CVEs will be disclosed against even minimal images over time.

---

## 2. What Changed

The reduction was achieved through a **single automated iteration containing three validated steps**. Each step was gated by a full rebuild + the application's own test suite + a Trivy rescan; non-improving steps were rolled back but retained as adjudication candidates.

| Step | Technique | Result | (CRITICAL, HIGH) |
|------|-----------|--------|------------------|
| 1 | `os-patch` — Debian blanket `apt-get upgrade` in base stage | ✅ passed | (7, 83) → (5, 71) |
| 2 | `os-patch` — repeated Debian blanket upgrade | ⏹ no improvement (rolled back) | (5, 71) → (5, 71) |
| 3 | `llm-base` — swap base to `cgr.dev/chainguard/node:latest` | ✅ passed | (5, 71) → **(0, 0)** |

**Plain-language account:**

1. **OS package upgrade (partial win).** A blanket Debian upgrade cleared the CVEs that *had* fix versions available in the `bookworm` repositories — e.g. `libgnutls30` (→ `3.7.9-2+deb12u7`), `libpam*` (→ `1.5.2-6+deb12u2`), `libcap2` (→ `1:2.66-4+deb12u3`), `gpgv` (→ `2.2.40-1.1+deb12u2`). This knocked out 2 CRITICAL and 12 HIGH findings but **left every `NO FIX` OS CVE untouched** (util-linux family, perl-base, ncurses, systemd libs, gzip, libacl1, zlib1g).
2. **Second upgrade attempt (no-op).** A repeated upgrade produced no further improvement, confirming the Debian package repositories had no additional fixes to offer. Rolled back.
3. **Base image swap (decisive win).** Rebasing onto **Chainguard's distroless Node image** eliminated the remaining findings entirely by:
   - **Removing the Debian userland** — the entire `util-linux`, `ncurses`, `perl-base`, `systemd`, `bsdutils`, `mount`, `gzip`, `libacl1` surface no longer ships in the image, so all their `NO FIX` CVEs vanish.
   - **Shipping current Node tooling** — the bundled/global npm packages (`tar`, `minimatch`, `brace-expansion`, `glob`, `cross-spawn`, `sigstore`, `pacote`, `ip-address`) arrive at patched versions.

**Net:** 90 → 0 in one iteration. The winning strategy was **surface reduction via base swap**, not incremental patching.

---

## 3. Remaining Risk Breakdown

**Current scan: 0 findings.** There are no OS-package, compiled-in, or application-level CVEs present in `v1-golden-base-app`.

Because there is nothing to remediate today, this section instead documents the **structural risks introduced by the chosen remediation** and the classes of CVE to watch for in future scans.

### 3.1 OS packages with no fix available yet
- **None present.** The base swap removed the Debian userland entirely. The previous `NO FIX` cluster is listed here for the historical record — these are no longer in the image and require **no action**:

| CVE(s) | Component (removed) | Prior status |
|--------|--------------------|--------------|
| `CVE-2026-53613`, `CVE-2026-76642`, `CVE-2026-78408/9/10` | util-linux family (`mount`, `libmount1`, `libblkid1`, `libuuid1`, `libsmartcols1`, `bsdutils`, `util-linux`, `util-linux-extra`) | NO FIX → removed |
| `CVE-2025-69720` | ncurses (`libtinfo6`, `ncurses-base`, `ncurses-bin`) | NO FIX → removed |
| `CVE-2026-16742` | systemd (`libsystemd0`, `libudev1`) | NO FIX → removed |
| `CVE-2026-13221`, `-42496`, `-8376`, `-42497`, `-48962`, `-57432`, `-57433`, `-9538` | perl-base | NO FIX → removed |
| `CVE-2026-41992` | gzip | NO FIX → removed |
| `CVE-2026-54369` | libacl1 | NO FIX → removed |
| `CVE-2023-45853` | zlib1g | NO FIX → removed |

### 3.2 Compiled-in / application-level CVEs
- **None present.** The Node dependency CVEs (`tar`, `minimatch`, `brace-expansion`, `glob`, `cross-spawn`, `ip-address`, `sigstore`, `pacote`) were resolved via the current tooling shipped in the Chainguard base.

**Forward-looking remediation guidance** (for when future rescans surface new items):

| Risk class | If it reappears, remediate by… |
|------------|-------------------------------|
| Chainguard base OS CVE | Re-pull `cgr.dev/chainguard/node:latest`, rebuild, rescan. Chainguard rebuilds continuously; a fresh pull is usually the fix. Consider `-fips`/`-dev` variants only if functionally required. |
| Bundled Node global tool CVE (`npm`, `tar`, `pacote`, etc.) | Bump the base tag; these travel with the image. Do not vendor older copies. |
| **Application `package.json` dependency CVE** | Pin and upgrade in the app manifest (`npm audit fix`, or explicit `overrides`), commit lockfile, rebuild. This is *not* solved by the base swap. |

---

## 4. Risk Acceptance Template

No CVEs currently require acceptance. Retain this template for any finding a future rescan surfaces that cannot be immediately remediated (e.g. a `NO FIX` upstream CVE reachable only in an unused code path).

```
CVE: <ID>
Status: Risk Accepted
Reason: <why this is acceptable in this deployment — e.g. vulnerable code path
         is unreachable; component not exposed to untrusted input; compensating
         control X in place; no upstream fix and severity mitigated by runtime
         hardening>
Reviewed by: <name / security owner>
Review date: <YYYY-MM-DD>
Next review: <YYYY-MM-DD + 90 days>
```

Example (illustrative — not currently applicable):

```
CVE: CVE-XXXX-XXXXX
Status: Risk Accepted
Reason: NO FIX from upstream. Affected binary (mount) is not invoked by the
        workload; container runs read-only and non-root, eliminating the
        privilege-escalation vector. Reachability confirmed unexploitable.
Reviewed by: A. Engineer (Container Security)
Review date: 2026-01-15
Next review: 2026-04-15
```

---

## 5. Residual Risk Guidance — Compensating Controls

Even at 0 findings, apply defense-in-depth. These controls harden the workload against **future** CVEs disclosed between rescans and against classes of attack (path traversal, TOCTOU, privilege escalation) that dominated the original finding set.

### 5.1 Runtime hardening (Pod/container spec)

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

| Control | Mitigates | Notes |
|---------|-----------|-------|
| `readOnlyRootFilesystem: true` | File-overwrite / arbitrary-write CVEs (the `tar`, `perl-Archive-Tar`, util-linux TOCTOU classes) | Mount `emptyDir` for any writable temp paths the app needs. |
| `runAsNonRoot` + `runAsUser: 65532` | Privilege-escalation CVEs (`libcap`, PAM, systemd-homed) | Chainguard images ship a `nonroot` user by default. |
| `capabilities: drop: [ALL]` | Mount/namespace abuse (`nsenter`, bind-mount CVEs) | Add back only explicitly required caps (usually none). |
| `allowPrivilegeEscalation: false` | setuid/TOCTOU escalation | Blocks `no_new_privs` bypass. |
| `seccompProfile: RuntimeDefault` | Kernel attack surface | Consider a tighter custom profile once syscall set is profiled. |

### 5.2 AppArmor / seccomp
- Enforce `RuntimeDefault` seccomp cluster-wide via admission policy.
- Where AppArmor is available, apply a `runtime/default` or workload-tailored profile:
  ```yaml
  metadata:
    annotations:
      container.apparmor.security.beta.kubernetes.io/<container>: runtime/default
  ```

### 5.3 Network policy (default-deny)
- Distroless images have no shell, but a compromised process can still make outbound calls (relevant to the `CPAN.pm` / `sigstore` cert-validation class of CVE). Enforce egress restriction:
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: nodejs-app-default-deny
  spec:
    podSelector:
      matchLabels: { app: nodejs-app }
    policyTypes: ["Ingress", "Egress"]
    ingress:
      - from:
          - podSelector: { matchLabels: { role: gateway } }
    egress:
      - to:
          - namespaceSelector: { matchLabels: { name: platform } }
        ports:
          - { protocol: TCP, port: 443 }
  ```

### 5.4 mTLS enforcement
- Enforce service-mesh mTLS (Istio `PeerAuthentication: STRICT` or Linkerd default) so all pod-to-pod traffic is mutually authenticated and encrypted — directly compensating for the GnuTLS/authentication-bypass class of CVE.
  ```yaml
  apiVersion: security.istio.io/v1
  kind: PeerAuthentication
  metadata: { name: default }
  spec:
    mtls: { mode: STRICT }
  ```

### 5.5 Supply-chain & operational controls
- **Pin the base by digest**, not `:latest`, in production manifests; promote the digest through CI after rescan. `:latest` is acceptable at build time for freshness but must be resolved to an immutable digest for deployment.
- **Schedule recurring rescans** (e.g. daily Trivy against the deployed digest) so newly disclosed CVEs against the Chainguard base are caught even without a rebuild.
- **Auto-rebuild cadence**: rebuild + rescan + republish the golden base weekly to absorb upstream Chainguard patches.
- **Sign & verify** the golden base (`ghcr.io/sgrsaga/node:latest-golden-base`) with Cosign; enforce signature verification at admission.

---

### Sign-off

| Field | Value |
|-------|-------|
| Final image | `ghcr.io/sgrsaga/nodejs-app:v1-golden-base-app` |
| Final base | `cgr.dev/chainguard/node:latest` (`ghcr.io/sgrsaga/node:latest-golden-base`) |
| Findings resolved | 90 / 90 (100%) |
| Findings remaining | 0 |
| Newly introduced | 0 |
| Residual posture | **Minimal (point-in-time)** — maintain via digest-pinning + scheduled rescans |