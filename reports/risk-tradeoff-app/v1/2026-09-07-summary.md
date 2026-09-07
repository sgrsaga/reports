# Remediation Summary

> Original image: `ghcr.io/sgrsaga/risk-tradeoff-app:v1`
> Final image: `ghcr.io/sgrsaga/risk-tradeoff-app:v1-optimized-app`
> Status: `optimized_app`

# Container Security Remediation Summary

**Image:** `ghcr.io/sgrsaga/risk-tradeoff-app:v1` → `ghcr.io/sgrsaga/risk-tradeoff-app:v1-optimized-app`
**Base image:** `python:3.11-slim-bookworm` (published as `ghcr.io/sgrsaga/python:3.11-slim-bookworm-optimized-base`)
**Run status:** `optimized_app` · **Iterations:** 1 · **Report date:** _fill on publish_

---

## 1. Executive Summary

| Metric | Before | After | Delta |
|--------|--------|-------|-------|
| **CRITICAL** | 22 | 5 | **−17 (−77%)** |
| **HIGH** | 1,719 | 58 | **−1,661 (−97%)** |
| **Total** | 1,741 | 63 | **−1,678 (−96%)** |
| Resolved | — | 1,679 | — |
| Still present | — | 62 | — |
| Newly introduced | — | 1 | — |

**Overall risk rating: `CRITICAL` → `MEDIUM`.**

The remediation run reduced the total vulnerability surface by **96%**, eliminating essentially the entire kernel-header (`linux-libc-dev`) attack surface — which alone accounted for the overwhelming majority of the original 1,719 HIGH findings — along with all fixable OpenSSL, GnuTLS, krb5, glibc, expat, and PAM CVEs. This was achieved with a **single effective base-image swap** plus a Debian security-patch pass, with the application's own test suite passing on the winning candidate.

What remains is a small, well-characterized residual set: **5 CRITICAL and 58 HIGH** findings, dominated by (a) OS packages with **no upstream fix yet available** (`NO FIX`), and (b) a cluster of `util-linux`/`bsdutils` and Perl-module CVEs. One low-impact HIGH (`CVE-2026-23949`, `jaraco.context`) was **newly introduced** by the base swap and should be resolved by a trivial dependency bump. None of the remaining items reflect a regression in the app's runtime dependencies of concern; most are triage-and-accept or await-upstream candidates.

---

## 2. What Changed

The reduction was achieved through **base-image selection driven by an automated, test-gated remediation loop**. Each step was validated by a full rebuild, the app's test suite, and a Trivy rescan; non-improving or breaking steps were rolled back but retained for adjudication.

### Step-by-step trail

| Step | Action | Result | (CRIT, HIGH) |
|------|--------|--------|--------------|
| 1 | `os-patch` — Debian blanket upgrade in base stage | ✅ passed | (22, 1719) → (6, 274) |
| 2 | `os-patch` — repeat blanket upgrade | ⚪ no improvement | (6, 274) → (6, 274) |
| 3 | `llm-base` — swap to **`python:3.11-slim-bookworm`** | ✅ passed | (6, 274) → (5, 58) |
| 4 | `os-patch` — blanket upgrade on new base | ⚪ no improvement | (5, 58) → (5, 58) |
| 5 | `llm-base` — `distroless/python3-debian12:nonroot` | ❌ build/test failed (`/bin/sh` missing during `pip install`) | — |
| 6 | `llm-base` — `cgr.dev/chainguard/python:latest` | ❌ build/test failed (`/bin/sh` missing during `pip install`) | — |
| 7 | `llm-base` — `python:3.11-slim-bullseye` | ⚪ regressed | (5, 58) → (7, 88) |

### Plain-language account

1. **OS package upgrade (Step 1)** — A Debian security-update pass on the original base collapsed the count from 1,741 to 280 by patching the huge bank of fixable kernel-header, TLS, and crypto CVEs.
2. **Base swap (Step 3)** — Replacing the base with the current, slimmer **`python:3.11-slim-bookworm`** dropped it further to **63**, primarily by shipping already-patched system libraries and a smaller package footprint.
3. **Rejected paths** — Distroless and Chainguard images broke the build because the Dockerfile invokes a shell during `pip install` (no `/bin/sh` in those images). The older `bullseye` base *increased* vulnerabilities and was discarded.

**Adjudication:** The balanced pick was the `bookworm` candidate — lowest counts, tests green, current Debian release, no breaking changes.

---

## 3. Remaining Risk Breakdown

63 findings remain (5 CRITICAL, 58 HIGH). They fall into three groups.

### 3a. OS packages — NO FIX available yet

These are awaiting an upstream Debian/library patch. Track and re-scan periodically.

| Severity | CVE | Package(s) | Notes / Guidance |
|----------|-----|-----------|------------------|
| CRITICAL | `CVE-2023-45853` | `zlib1g` | zlib integer overflow in MiniZip. Debian assesses as won't-fix / minor; only reachable via MiniZip API, which Python's `zlib` does not expose. Low practical reachability. |
| CRITICAL | `CVE-2025-7458` | `libsqlite3-0` | SQLite integer overflow. Reachable only if app parses untrusted SQL/DB files. |
| HIGH | `CVE-2026-11822`, `CVE-2026-11824` | `libsqlite3-0` | SQLite FTS5 / heap issues. Same reachability caveat as above. |
| HIGH | `CVE-2026-16742` | `libsystemd0`, `libudev1` | systemd-homed local privesc. Not applicable — `systemd-homed` is not run in-container. |
| HIGH | `CVE-2025-69720` | `libncursesw6`, `libtinfo6`, `ncurses-base`, `ncurses-bin` | ncurses buffer overflow. No fix; ncurses not on the app's runtime path. Candidate for package removal. |
| HIGH | `CVE-2026-41992` | `gzip` | gzip global buffer overflow (info disclosure). Reachable only when decompressing attacker-controlled archives. |
| HIGH | `CVE-2026-54369` | `libacl1` | libacl symlink-traversal privesc. Local-only; mitigated by non-root + read-only FS. |
| HIGH | `CVE-2026-53613`, `CVE-2026-76642`, `CVE-2026-78408`, `CVE-2026-78409`, `CVE-2026-78410` | `util-linux` / `bsdutils` / `libblkid1` / `libmount1` / `libsmartcols1` / `libuuid1` / `mount` / `util-linux-extra` | `mount`/`nsenter`/`bind-mount` TOCTOU and helper issues. **All require local `mount` capability**, which containers should not have. Strong accept-with-controls candidates, or remove `util-linux` if unused. |

> **High-value action:** Most `util-linux`/`bsdutils` and `ncurses` findings are **not exercised at runtime**. Removing these packages from the final image (if the app doesn't invoke `mount`/`nsenter`/curses UIs) would eliminate the bulk of the remaining HIGH count outright.

### 3b. Application / compiled-in CVEs — need upstream release or dependency bump

| Severity | CVE | Package | Fix Version | Guidance |
|----------|-----|---------|-------------|----------|
| CRITICAL | `CVE-2026-13221` | `perl-base` | NO FIX | Perl regex processing. Perl is a base-OS dependency, not app-invoked. Await Debian perl update. |
| CRITICAL | `CVE-2026-42496` | `perl-base` (`Archive::Tar`) | NO FIX | Path traversal via crafted tar. Only reachable if app shells out to perl `Archive::Tar`. |
| CRITICAL | `CVE-2026-8376` | `perl-base` | NO FIX | Perl regex heap overflow. Await upstream. |
| HIGH | `CVE-2026-42497`, `CVE-2026-48962`, `CVE-2026-57432`, `CVE-2026-57433`, `CVE-2026-9538` | `perl-base` | NO FIX | Archive::Tar / IO::Compress / Storable / integer-overflow issues. Consider removing perl if not required by any runtime tooling. |
| HIGH | `CVE-2024-23342` | `ecdsa` (Python) | NO FIX | **Minerva timing attack.** `python-ecdsa` has no fix. **Remediation: migrate off `python-ecdsa`** to a constant-time library (e.g. `cryptography`) if ECDSA is used; otherwise remove the transitive dependency. |
| HIGH | `CVE-2026-24049` | `wheel` (Python) | **0.46.2** | **Fixable now.** Bump `wheel` to `>=0.46.2` in the build/tooling environment. |
| HIGH ⚠️ NEW | `CVE-2026-23949` | `jaraco.context` (Python) | **6.1.0** | **Newly introduced** by the base swap (transitive of setuptools tooling). **Bump `jaraco.context` to `>=6.1.0`** or pin `setuptools`/`pip` toolchain to pull the fixed version. |

### 3c. Developer direction (from adjudication)

- **Pin the base by digest**: `python:3.11-slim-bookworm@sha256:...` for reproducibility and drift prevention.
- **Rebuild after latest bookworm apt security updates** to clear residual `bsdutils`/`util-linux` CVEs; **remove `util-linux`/`ncurses`/`perl` if not needed at runtime**.
- **Bump Python tooling** (`wheel>=0.46.2`, `jaraco.context>=6.1.0`) and **replace/remove `python-ecdsa`**.
- **Triage the 5 CRITICALs for reachability** and file VEX/suppressions for non-reachable ones to keep scan signal actionable.

---

## 4. Risk Acceptance Template

Copy-paste one block per accepted CVE. Suggested pre-filled reasons for the strongest accept candidates are shown.

```
CVE: CVE-2026-53613
Status: Risk Accepted
Reason: util-linux mount TOCTOU requires local mount capability; container runs
        non-root with no CAP_SYS_ADMIN and a read-only root filesystem, so the
        mount program is not invokable. No fix currently available upstream.
Reviewed by: <name>
Review date: <date>
Next review: <date + 90 days>
```

```
CVE: CVE-2026-16742
Status: Risk Accepted
Reason: systemd-homed is not installed/run in this container image; the
        vulnerable code path is unreachable at runtime.
Reviewed by: <name>
Review date: <date>
Next review: <date + 90 days>
```

```
CVE: CVE-2023-45853
Status: Risk Accepted
Reason: zlib overflow is confined to the MiniZip component, which is not exposed
        via Python's zlib module or the application. Debian classifies as
        minor/won't-fix. No upstream fix available.
Reviewed by: <name>
Review date: <date>
Next review: <date + 90 days>
```

```
CVE: <ID>
Status: Risk Accepted
Reason: <why this is acceptable in this deployment>
Reviewed by: <name>
Review date: <date>
Next review: <date + 90 days>
```

> **Do not risk-accept** `CVE-2026-24049` (wheel) or `CVE-2026-23949` (jaraco.context) — both have fixes available and should be patched rather than accepted.

---

## 5. Residual Risk Guidance — Compensating Controls

The remaining CVEs are predominantly **local-privilege / requires-local-access** or **requires-untrusted-input-parsing**. The following controls materially reduce exploitability until upstream fixes land.

### 5a. Pod / container hardening (Kubernetes `securityContext`)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true          # blocks util-linux/libacl symlink & mount abuse
  capabilities:
    drop: ["ALL"]                        # removes CAP_SYS_ADMIN -> mount/nsenter CVEs unreachable
  seccompProfile:
    type: RuntimeDefault                 # blocks mount()/related syscalls
```

| Control | Mitigates |
|---------|-----------|
| `readOnlyRootFilesystem: true` | `libacl1` (CVE-2026-54369), util-linux TOCTOU, gzip write paths |
| `capabilities: drop [ALL]` + no `CAP_SYS_ADMIN` | All `util-linux`/`bsdutils` mount/nsenter CVEs (`CVE-2026-53613/76642/78408/78409/78410`) |
| `runAsNonRoot` / `allowPrivilegeEscalation: false` | perl-base local privesc vectors, systemd-homed CVE |
| `seccompProfile: RuntimeDefault` | Blocks `mount`, `unshare`, `setns` syscalls used by util-linux CVEs |

### 5b. AppArmor / seccomp

```yaml
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
```
- Custom AppArmor profile denying `mount`, `pivot_root`, and write to `/etc`, `/bin`, `/usr` reinforces the read-only FS guarantee against util-linux/libacl issues.

### 5c. Network policy & mTLS

- **Default-deny NetworkPolicy** (ingress + egress) scoped to only required peers — limits reachability of any input-parsing CVE (SQLite/expat/gzip) to trusted upstream services.
- **Enforce mTLS** (service mesh, e.g. Istio `PeerAuthentication: STRICT`) so untrusted clients cannot deliver crafted payloads (crafted SQL, tar, compressed data) to the SQLite/perl/gzip code paths.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: { name: risk-tradeoff-default-deny }
spec:
  podSelector: { matchLabels: { app: risk-tradeoff-app } }
  policyTypes: ["Ingress","Egress"]
  # add explicit allow rules only for required services
```

### 5d. Input-handling controls

- Do not pass untrusted archives to any `perl Archive::Tar` / `gzip` invocation; validate/sandbox decompression.
- If ECDSA is used (`python-ecdsa`, CVE-2024-23342), **migrate to `cryptography`** (constant-time) and enforce TLS termination at the mesh to reduce timing-oracle exposure.

### 5e. Operational

- **Re-scan on a schedule** (e.g. weekly) so `NO FIX` items are patched as soon as Debian ships updates.
- **Digest-pin the base image** and rebuild after each bookworm security release to keep the residual `util-linux`/`perl` set shrinking automatically.

---

_Generated from a single-iteration automated remediation run. Base artifact: `ghcr.io/sgrsaga/python:3.11-slim-bookworm-optimized-base`._