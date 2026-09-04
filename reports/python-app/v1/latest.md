# Remediation Summary

> Original image: `ghcr.io/sgrsaga/python-app:v1`
> Final image: `ghcr.io/sgrsaga/python-app:v1-golden-base-app`
> Status: `golden_base_app`

# Container Security Remediation Summary

**Image:** `ghcr.io/sgrsaga/python-app:v1` → `ghcr.io/sgrsaga/python-app:v1-golden-base-app`
**Run status:** `golden_base_app`
**Iterations:** 1 (multi-step base-selection + dependency remediation)
**Scanner:** Trivy (rebuild + application test suite validated at every step)

---

## 1. Executive Summary

| Metric | Before | After |
|--------|-------:|------:|
| **CRITICAL** | 6 | **0** |
| **HIGH** | 109 | **0** |
| **Total** | 115 | **0** |
| **Overall risk rating** | **Critical** | **Low / Clean** |

The remediation run achieved a **complete elimination of all 115 detected findings** (6 CRITICAL, 109 HIGH), moving the image from a **Critical** risk posture to a **clean** state at the scanned severities. This was accomplished primarily through a **base-image swap** to `python:3.12-alpine` — which dropped the entire Debian `util-linux`/`perl-base`/`openssl`/`ncurses` attack surface that dominated the original findings — followed by a targeted **Python dependency upgrade** (`Flask`, `Werkzeug`) to clear the last three application-layer HIGHs.

**What remains:** At the scanned severity tiers (CRITICAL/HIGH), **nothing**. However, "zero findings" is a point-in-time result against the current vulnerability database. The Alpine (musl) base and the pinned application dependencies will accrue new CVEs over time, so the residual risk is entirely **future drift**, not present exposure. Continuous rescanning and the compensating controls in Section 5 remain mandatory.

---

## 2. What Changed

The reduction was **not** a single action — it was an adjudicated search across candidate bases and package strategies. Each step was validated by a **full rebuild + application test suite + Trivy rescan**, and non-improving or failing steps were rolled back but retained as adjudication candidates.

### Step-by-step trail

| # | Strategy | Action | Result | (C, H) |
|---|----------|--------|--------|--------|
| 1 | `os-patch` | Debian blanket `apt-get upgrade` in base stage | ✅ Passed | (6,109) → **(3,57)** |
| 2 | `os-patch` | Repeat Debian blanket upgrade | ⚪ No improvement | (3,57) → (3,57) |
| 3 | `llm-base` | `cgr.dev/chainguard/python:latest-dev` | ❌ Build/test failed — `pytest: not found` (script dir not on PATH) | — |
| 4 | `llm-base` | **`python:3.12-alpine`** | ✅ **Passed** | (3,57) → **(0,3)** |
| 5 | `os-patch` | Debian upgrade against alpine base | ❌ Failed — `apt-get: not found` (alpine uses `apk`) | — |
| 6 | `llm-base` | `redhat/ubi9/python-312:latest` | ❌ Failed — pull access denied (auth required) | — |
| 7 | `llm-base` | `gcr.io/distroless/python3-debian12:debug` | ❌ Failed — no `/bin/sh` for pip layer | — |
| 8 | `llm-base` | `debian:12-slim` | ❌ Failed — `pip: not found` | — |
| 9 | `dep-bump#1` | `Flask==2.3.2`, `Werkzeug==3.0.3` | ✅ **Passed** | (0,3) → **(0,0)** |

### Plain-language account

1. **Debian in-place OS patching** (Step 1) resolved the OpenSSL cluster (`CVE-2026-31789`, `-28387/8/9/90`, `-45447`, `-14456`, `CVE-2025-15467`, `-69421`) and `libcap2` (`CVE-2026-4878`), taking CRITICAL 6→3 and HIGH 109→57. It **could not** touch the large `NO FIX` residue (perl-base, util-linux, ncurses, sqlite, gzip, libacl1, systemd).
2. **Base swap to `python:3.12-alpine`** (Step 4) was the decisive move. By replacing the Debian userland with Alpine/musl, the entire block of unfixable Debian CVEs — `perl-base` (6 CVEs incl. 3 CRITICAL), the `util-linux` family (`CVE-2026-53612/13/14`, `-76642`, `-78408/09/10` × 9 packages each), `ncurses`, `libsqlite3-0`, `gzip`, `libacl1`, `libsystemd0` — **ceased to exist in the image**. This dropped remaining findings to just 3 HIGHs, all Python-layer.
3. **Dependency upgrade** (Step 9) bumped `Flask` 2.2.2→2.3.2 and `Werkzeug` 2.2.2→3.0.3, clearing `CVE-2023-30861`, `CVE-2023-25577`, and `CVE-2024-34069` — reaching **(0, 0)**.

> **Note on the diff list:** The reported "Resolved (115)" list contains duplicate CVE IDs because a single CVE (e.g. `CVE-2026-53612`) was recorded against multiple installed packages (bsdutils, libblkid1, libmount1, login, mount, util-linux, …). These were all eliminated together by the base swap.

**Final base artifact:** `python:3.12-alpine`, published as `ghcr.io/sgrsaga/python:3.12-alpine-golden-base`.

---

## 3. Remaining Risk Breakdown

### At scanned severities (CRITICAL/HIGH): **none**

There are **no OS packages with unfixed CVEs** and **no application-level CVEs** remaining in the final image at the CRITICAL/HIGH tiers. The `NO FIX` Debian packages that previously drove risk (perl-base, util-linux, ncurses, sqlite, gzip, libacl1, systemd) are **no longer present** because the Alpine base does not ship them in this footprint.

### Residual (structural / non-severity) risk to track

Although the finding count is zero, the following structural risks persist and should be treated as *managed*, not *eliminated*:

| Area | Residual concern | Remediation guidance |
|------|------------------|----------------------|
| **musl libc (Alpine)** | Alpine uses musl rather than glibc. Some CVE feeds under-report musl; and musl-specific defects can appear. | Keep Trivy DB current; add a secondary scanner (Grype) for cross-validation; pin the digest, not just the `:3.12-alpine` tag. |
| **Python interpreter (3.12)** | Interpreter CVEs (e.g. future CPython advisories) will surface as the base ages. | Rebuild from `python:3.12-alpine` on a scheduled cadence to absorb upstream interpreter patches; track CPython security releases. |
| **Pinned app deps** | `Flask==2.3.2`, `Werkzeug==3.0.3` are fixed *now* but will drift. Note: Werkzeug 3.0.3 is the fix line for `CVE-2024-34069`; there is no newer requirement today. | Enable Dependabot/Renovate; re-run `dep-bump` on each release; keep `requirements.txt` version-pinned + hash-pinned. |
| **Transitive build deps** | `wheel`, `jaraco.context`, `setuptools`-family were only present in the Debian build; verify they are not baked into the runtime layer. | Confirm the runtime stage is dependency-minimal (multi-stage build) and does not carry build-time packages like `wheel` (`CVE-2026-24049`) or `jaraco.context` (`CVE-2026-23949`). |

There are currently **no CVEs requiring an upstream release or code change** to fix in the final image.

---

## 4. Risk Acceptance Template

Because the final image reports **0 CRITICAL / 0 HIGH**, no risk acceptance entries are required today. Retain the template below for any **MEDIUM/LOW** findings, or any future CVE that a team elects to accept between remediation cycles.

```
CVE: <ID>
Status: Risk Accepted
Reason: <why this is acceptable in this deployment — e.g. affected code path
         not reachable, package present but binary not invoked, mitigated by
         compensating control X, no fixed version available upstream>
Reviewed by: <name / team>
Review date: <YYYY-MM-DD>
Next review: <YYYY-MM-DD + 90 days>
```

**Worked example (illustrative — for a hypothetical future no-fix MEDIUM):**

```
CVE: CVE-YYYY-NNNNN
Status: Risk Accepted
Reason: Package ships in base but the vulnerable subcommand is never executed;
        container runs read-only, non-root, with seccomp default profile.
        No upstream fix available as of review date.
Reviewed by: Container Security Team
Review date: 2025-06-01
Next review: 2025-08-30
```

---

## 5. Residual Risk Guidance — Compensating Controls

Even at zero findings, deploy defense-in-depth to contain any **as-yet-undisclosed** vulnerability in the Alpine base, interpreter, or dependencies.

### 5.1 Pod / container hardening (`securityContext`)

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 65532            # match a non-root uid present in the image
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

- **`readOnlyRootFilesystem: true`** — neutralizes any residual path-traversal / arbitrary-file-write class defect (the profile of the eliminated perl-archive-tar and util-linux mount CVEs). Mount `emptyDir` for `/tmp` if the app needs scratch space.
- **`drop: ["ALL"]` + `allowPrivilegeEscalation: false`** — directly counters the privilege-escalation class (`libcap` TOCTOU, `systemd-homed`, SUID `mount`) that dominated the original Debian findings, should any analogue appear on Alpine.
- **`runAsNonRoot`** — ensures no root-owned cgroup / namespace leak (the `nsenter`/`X-mount.subdir` class) is reachable.

### 5.2 Seccomp / AppArmor

- Enforce `RuntimeDefault` seccomp cluster-wide via a `PodSecurity` `restricted` namespace label:
  ```yaml
  pod-security.kubernetes.io/enforce: restricted
  ```
- Where supported, attach a tuned **AppArmor** profile denying `mount`, `ptrace`, and raw socket syscalls — the app is a Python web service and needs none of them.

### 5.3 Network policy (default-deny + explicit allow)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: python-app-default-deny
spec:
  podSelector:
    matchLabels:
      app: python-app
  policyTypes: ["Ingress", "Egress"]
  ingress:
    - from:
        - podSelector:
            matchLabels: { role: gateway }
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - namespaceSelector:
            matchLabels: { kubernetes.io/metadata.name: kube-system }
      ports:
        - protocol: UDP
          port: 53   # DNS only
```

- Limits blast radius of any future RCE (the profile of the eliminated OpenSSL/SQLite/ncurses code-exec CVEs) by preventing lateral movement and unexpected egress (C2 callbacks).

### 5.4 mTLS enforcement

- Enforce **STRICT mTLS** at the mesh layer (Istio/Linkerd) so the workload only accepts authenticated peer connections:
  ```yaml
  apiVersion: security.istio.io/v1
  kind: PeerAuthentication
  metadata:
    name: python-app-mtls
  spec:
    selector:
      matchLabels: { app: python-app }
    mtls:
      mode: STRICT
  ```
- Complements the OpenSSL remediation: even with a patched TLS stack, mesh-managed mTLS removes reliance on the app's own certificate handling and closes the PKCS7/PKCS12 parsing exposure class at the platform layer.

### 5.5 Supply-chain & drift controls

- **Pin by digest**, not tag: `ghcr.io/sgrsaga/python:3.12-alpine-golden-base@sha256:...`.
- **Admission control**: enforce signed images (Cosign) + a policy that **blocks any image with CRITICAL/HIGH** at admission (Kyverno/OPA Gatekeeper).
- **Scheduled rescan**: nightly Trivy scan of the running digest against the latest DB; alert on any new CRITICAL/HIGH so the `golden_base` can be rebuilt.
- **Rebuild cadence**: regenerate the golden base weekly (or on upstream `python:3.12-alpine` publish) to continuously absorb Alpine `apk` and CPython patches.

---

### Sign-off

| Field | Value |
|-------|-------|
| Final image | `ghcr.io/sgrsaga/python-app:v1-golden-base-app` |
| Golden base | `ghcr.io/sgrsaga/python:3.12-alpine-golden-base` |
| Findings after | **0 CRITICAL / 0 HIGH / 115 resolved / 0 introduced** |
| Recommendation | **Approve for promotion**, contingent on digest-pinning + the Section 5 controls being enforced in the target namespace. |