# Remediation Summary

> Original image: `ghcr.io/sgrsaga/python-app:v1`
> Final image: `ghcr.io/sgrsaga/python-app:v1-optimized-app`
> Status: `optimized_app`

# Container Security Remediation Report

**Image:** `ghcr.io/sgrsaga/python-app:v1` → `ghcr.io/sgrsaga/python-app:v1-optimized-app`
**Base artifact:** `python:3.9-slim` (published as `ghcr.io/sgrsaga/python:3.9-slim-optimized-base`)
**Run status:** `optimized_app` · **Iterations:** 1 · **Newly introduced CVEs:** 0

---

## 1. Executive Summary

| Metric | Before | After | Delta |
|--------|-------:|------:|------:|
| **CRITICAL** | 6 | 3 | **−3 (−50%)** |
| **HIGH** | 109 | 57 | **−52 (−48%)** |
| **Total** | 115 | 60 | **−55 (−48%)** |

**Overall risk rating: `CRITICAL` → `HIGH`.**

This single-iteration run cut the overall vulnerability count roughly in half with **zero regressions and no application code changes**, driven almost entirely by a low-risk Debian blanket OS package upgrade in the base stage (validated by rebuild + test suite + Trivy rescan). All three remaining CRITICALs and the bulk of the residual HIGHs are concentrated in a small set of packages — `perl-base`, `util-linux`, `ncurses`, `sqlite`, `systemd`, `gzip`, `libacl1` — where **no upstream fix is currently published**, plus three application-level Python CVEs (`Flask`, `Werkzeug`) that a validated dependency bump can eliminate. The image is materially safer but should **not** yet be treated as "clean": the 3 residual CRITICALs (all `perl-base`) require a reachability assessment and formal risk acceptance or a base-image change before this can be considered production-hardened.

---

## 2. What Changed

The reduction was achieved through the automated remediation pipeline, which evaluated multiple candidate strategies and adjudicated the best-balanced one. Each step was validated by a full rebuild, the application's own test suite, and a Trivy rescan; non-improving or failing steps were rolled back but retained as adjudication candidates.

**Candidate evaluation summary:**

| Step | Strategy | Result | (CRIT, HIGH) |
|------|----------|--------|--------------|
| `os-patch` **[0]** ✅ | Debian blanket upgrade in base stage | **Passed — selected** | (6,109) → **(3,57)** |
| `os-patch` | Repeat blanket upgrade | No improvement | (3,57) → (3,57) |
| `llm-base` | `cgr.dev/chainguard/python:latest-dev` | Build/test failed (`pytest: not found` — PATH issue in nonroot) | — |
| `llm-base` | `python:3.12-slim-bookworm` | No improvement / **regressed CRITICAL** | (3,57) → (5,58) |
| `llm-base` | `redhat/ubi9-python-39:latest` | Build failed (`pull access denied`) | — |
| `dep-bump#1` **[3]** ✅ | `Flask==2.3.2`, `Werkzeug==3.0.3` | Passed | (3,57) → **(3,54)** |

**What actually landed:** The **Debian blanket OS upgrade** (candidate [0]) on the `python:3.9-slim` base was the highest-value, best-balanced move. It:

- Upgraded **OpenSSL** (`libssl3t64`, `openssl`, `openssl-provider-legacy`) from `3.5.1-1+deb13u1` to a fixed build, clearing **8 distinct OpenSSL CVEs** (`CVE-2026-31789`, `CVE-2025-15467`, `CVE-2025-69421`, `CVE-2026-14456`, `CVE-2026-28387/28388/28389/28390`, `CVE-2026-45447`).
- Upgraded **util-linux** family to `2.41.5-0+deb13u1`, clearing the **fixed** TOCTOU/SUID mount CVEs (`CVE-2026-53612/53613/53614`).
- Upgraded **libcap2** to `1:2.75-10+deb13u1`, clearing `CVE-2026-4878`.

> **Note / open action:** The adjudicator recommended layering candidate [3]'s dependency bump (`Flask==2.3.2`, `Werkzeug==3.0.3`) on top of [0] to capture an additional 3 HIGH reductions. The **final scanned image still shows `Flask 2.2.2` / `Werkzeug 2.2.2`** (`CVE-2023-30861`, `CVE-2023-25577`, `CVE-2024-34069` still present), so this dependency bump was **not yet merged into the shipped artifact** — it is the top follow-up item.

---

## 3. Remaining Risk Breakdown

60 findings remain (3 CRITICAL, 57 HIGH). They fall into three categories.

### 3a. OS packages — **NO FIX available yet** (cannot patch by upgrade)

These are Debian packages with no fixed version published. They can only be resolved by an upstream security update, a base-image swap to a distro that has patched, or removal of the package.

| CVE(s) | Package(s) | Nature | Guidance |
|--------|-----------|--------|----------|
| `CVE-2026-76642`, `CVE-2026-78408`, `CVE-2026-78409`, `CVE-2026-78410` | `util-linux` family (`bsdutils`, `libblkid1`, `liblastlog2-2`, `libmount1`, `libsmartcols1`, `libuuid1`, `login`, `mount`, `util-linux`) | mount/nsenter TOCTOU, cgroup leak, bind-mount pinning, subdir resolution | **Runtime-mount abuse.** Container almost certainly does **not** perform privileged mounts. Remove `mount`/`nsenter` binaries if unused, drop `CAP_SYS_ADMIN`, run non-root, read-only rootfs. Track Debian tracker for `util-linux` fix. |
| `CVE-2025-69720` | `libncursesw6`, `libtinfo6`, `ncurses-base`, `ncurses-bin` | ncurses buffer overflow | Only triggerable via terminal handling. Remove ncurses if no interactive TTY tooling is needed in the runtime image. Monitor for fix. |
| `CVE-2026-11822`, `CVE-2026-11824` | `libsqlite3-0` (`3.46.1-7+deb13u1`) | SQLite FTS5 / heap ACE | Reachable **only if the app parses untrusted SQLite input via FTS5**. Confirm SQLite usage; if unused transitively, consider removal. Monitor for upstream fix. |
| `CVE-2026-16742` | `libsystemd0`, `libudev1` (`257.13-1~deb13u1`) | systemd-homed local privesc | Not exploitable without `systemd-homed`, which is absent in a container. **Low practical risk** — good candidate for risk acceptance. |
| `CVE-2026-41992` | `gzip` (`1.13-1`) | Info disclosure via global buffer overflow | Blanket upgrade did **not** pull a fixed `gzip`. **Pin/upgrade `gzip` explicitly** if a fixed Debian build exists; otherwise remove if unused at runtime. |
| `CVE-2026-54369` | `libacl1` (`2.3.2-2+b1`) | Symlink-traversal privesc via libacl | No fix yet. Compensate with read-only fs + non-root. Monitor tracker. |

### 3b. Application-level / compiled-in CVEs — fixable by upstream release or code change

| CVE | Package | Installed | Fix | Guidance |
|-----|---------|-----------|-----|----------|
| `CVE-2026-13221` **(CRIT)** | `perl-base` | 5.40.1-6 | NO FIX | Perl regex processing flaw. |
| `CVE-2026-42496` **(CRIT)** | `perl-base` | 5.40.1-6 | NO FIX | perl-archive-tar path traversal. |
| `CVE-2026-8376` **(CRIT)** | `perl-base` | 5.40.1-6 | NO FIX | Perl regex heap overflow. |
| `CVE-2026-42497`, `CVE-2026-48962`, `CVE-2026-57432`, `CVE-2026-57433`, `CVE-2026-9538` | `perl-base` | 5.40.1-6 | NO FIX | Archive::Tar / IO::Compress / Storable DoS & file-mod issues. |
| `CVE-2023-30861` | `Flask` | **2.2.2** | 2.3.2, 2.2.5 | **Directly fixable now** — bump to `Flask==2.3.2` (candidate [3], already test-validated). |
| `CVE-2023-25577` | `Werkzeug` | **2.2.2** | 2.2.3 | **Directly fixable now** — bump to `Werkzeug==3.0.3`. |
| `CVE-2024-34069` | `Werkzeug` | **2.2.2** | 3.0.3 | Covered by the same `Werkzeug==3.0.3` bump. |
| `CVE-2026-23949` | `jaraco.context` | 5.3.0 | 6.1.0 | Bump `jaraco.context>=6.1.0` (transitive build dep). |
| `CVE-2026-24049` | `wheel` | 0.45.1 | 0.46.2 | Bump `wheel>=0.46.2`; ensure it is not baked into the runtime layer if only used at build time. |

**`perl-base` note:** Perl is not typically required at *runtime* for a Python Flask app — it is pulled in as a base-image dependency. All 3 CRITICALs and 5 additional HIGHs live here. **The single most effective structural fix is to remove `perl-base` from the runtime image** (multi-stage build shipping only the Python venv + minimal libs) or swap to a `perl`-free distroless/Chainguard base once the earlier build failure (missing `pytest` on PATH in the nonroot Chainguard variant) is resolved.

### Recommended follow-up priority

1. **Merge the validated dep-bump** (`Flask==2.3.2`, `Werkzeug==3.0.3`) → eliminates 3 HIGH immediately. *(passed tests, zero regression)*
2. Bump `jaraco.context>=6.1.0`, `wheel>=0.46.2` → 3 more HIGH.
3. **Remove or exclude `perl-base`** from the runtime image → eliminates all 3 CRITICAL + 5 HIGH.
4. Explicitly pin/upgrade `gzip`; strip unused `mount`/`nsenter`/`ncurses`/`sqlite`.
5. Re-attempt a distroless / Chainguard base (fix the `pytest` PATH issue: install test deps into a build stage or add `~/.local/bin` to `PATH`).

---

## 4. Risk Acceptance Template

Use one entry per accepted CVE. Store alongside the image SBOM in the repo.

```
CVE: <ID>
Status: Risk Accepted
Reason: <why this is acceptable in this deployment — e.g. "systemd-homed not
         present in container; local privesc vector unreachable" or "package
         used only at build time, not shipped in runtime layer">
Reviewed by: <name>
Review date: <date>
Next review: <date + 90 days>
```

**Pre-filled candidates most defensible for acceptance:**

```
CVE: CVE-2026-16742
Status: Risk Accepted
Reason: systemd-homed is not present or executed in this container; the local
        privilege-escalation vector is unreachable in a non-init container
        context. No fix currently published upstream.
Reviewed by: <name>
Review date: <date>
Next review: <date + 90 days>
```

```
CVE: CVE-2026-76642 / CVE-2026-78408 / CVE-2026-78409 / CVE-2026-78410
Status: Risk Accepted
Reason: util-linux mount/nsenter attack surface unreachable — container runs
        non-root, without CAP_SYS_ADMIN, with a read-only root filesystem, and
        performs no privileged mounts. No fixed version available yet.
Reviewed by: <name>
Review date: <date>
Next review: <date + 90 days>
```

> ⚠️ The three `perl-base` **CRITICALs** (`CVE-2026-13221`, `CVE-2026-42496`, `CVE-2026-8376`) should **not** be blanket-accepted without a documented reachability analysis, since no fix exists and severity is Critical. Prefer package removal.

---

## 5. Residual Risk Guidance — Compensating Controls

Because a large share of the remaining findings (`util-linux`, `libacl1`, `ncurses`, `perl-base`, `sqlite`) are **NO-FIX** and rely on local/privileged execution to exploit, the following runtime hardening directly reduces exploitability while upstream fixes are pending.

### 5a. Pod / container security context (Kubernetes)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532
  allowPrivilegeEscalation: false        # blunts util-linux/libacl privesc paths
  readOnlyRootFilesystem: true           # blocks archive-tar/mount file writes
  capabilities:
    drop: ["ALL"]                         # removes CAP_SYS_ADMIN → mount CVEs inert
  seccompProfile:
    type: RuntimeDefault                  # blocks mount/nsenter/unshare syscalls
```

Mount writable scratch only where needed via `emptyDir` with `readOnly` elsewhere.

### 5b. seccomp / AppArmor

- **seccomp:** Apply `RuntimeDefault` (or a custom profile) that blocks `mount`, `umount2`, `unshare`, `setns`, `pivot_root` — this neutralizes the entire residual `util-linux` mount/nsenter CVE class.
- **AppArmor:** Attach a profile denying `mount`/`pivot_root` and restricting file access to the app's working directories; denies the `perl-archive-tar` and `libacl1` symlink-traversal write paths.

### 5c. Network controls

- **NetworkPolicy:** Default-deny ingress/egress; allow only required service ports. Limits blast radius if a `Flask`/`Werkzeug`/`SQLite` parsing bug is triggered by untrusted input.
- **mTLS:** Enforce service-mesh mutual TLS (Istio/Linkerd `STRICT` mode) so the Flask endpoint only receives authenticated peer traffic, shrinking the untrusted-input surface for the Werkzeug multipart (`CVE-2023-25577`) and Flask session CVEs before the dep-bump lands.

### 5d. Image minimization (removes the CVE, not just its exploitability)

- Ship a **multi-stage / distroless runtime** containing only the Python venv and required shared libs. Removing `perl-base`, `mount`, `nsenter`, `ncurses`, and unused `sqlite` **eliminates** ~40+ of the residual findings outright rather than just accepting them.
- Add a Trivy CI gate (`--severity CRITICAL,HIGH --exit-code 1`) with an explicit allowlist tied to the accepted-risk register in §4, so any *new* CRITICAL/HIGH fails the build.

### 5e. Monitoring & re-scan cadence

- Re-run Trivy on a scheduled basis (weekly) — several NO-FIX packages (`util-linux`, `gzip`, `sqlite`, `ncurses`) will receive Debian security updates; re-running the `os-patch` blanket upgrade will auto-clear them once published.
- Set the risk-acceptance review interval to **90 days** and re-validate reachability assumptions each cycle.

---

*Report generated for run status `optimized_app` — 1 iteration, 55 CVEs resolved, 0 newly introduced. Primary outstanding actions: (1) merge validated Flask/Werkzeug bump, (2) remove `perl-base` from runtime to clear all 3 CRITICALs.*