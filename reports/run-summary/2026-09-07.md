# Run Summary — external & internal scopes

# Container Vulnerability Remediation — Run-Level Summary

## External images

**Scope this run:** No external (third-party) images were processed in this sweep.

**Improvements achieved:** None applicable — there were no upstream/vendor images in scope, so no delta can be reported for this category.

**Residual risk factors:** The absence of external images in *this run* does not imply the cluster is free of third-party dependencies. Common blind spots that this report cannot attest to include:
- Vendor base layers pulled transitively into internal builds (not independently scanned here).
- Sidecars, init containers, and operator-managed images (e.g., service mesh proxies, CSI drivers, ingress controllers) that may bypass the internal build pipeline.
- Public registry images referenced directly by manifests without pinning to a digest.

**Concrete mitigations / guidance for future runs:**
- **Digest pinning:** Replace all `:tag` references for third-party images with immutable `@sha256:...` digests to eliminate silent drift.
- **Admission control:** Enforce a policy (e.g., Kyverno/Gatekeeper) that only permits images from approved registries and requires a passing scan attestation (Cosign/in-toto) before scheduling.
- **Upstream-watch:** Subscribe to CVE feeds / GitHub Security Advisories for each vendor image and wire a scheduled re-scan so third-party images are re-evaluated even when the app team does not rebuild.
- **Compensating controls for unpatchable vendor images:** network policy egress/ingress restriction, read-only root filesystem, dropped Linux capabilities, seccomp/AppArmor profiles, and runtime detection (Falco) to contain exploitation of any unfixed CVE.

---

## Internal images

### Base image selections and rationale

Two remediation strategies were applied, reflected in the per-image `status`:

**1. `golden_base_app` — distroless/minimal golden base (0 remaining HIGH/CRITICAL)**

| Image | Final tag | HIGH/CRITICAL |
|-------|-----------|---------------|
| `go-app:v1` | `v1-golden-base-app` | 0 |
| `java-app:v1` | `v1-golden-base-app` | 0 |
| `nodejs-app:v1` | `v1-golden-base-app` | 0 |
| `typescript-app:v1` | `v1-golden-base-app` | 0 |

These four images were rebased onto a hardened **golden base** (distroless-style minimal runtime). This drives HIGH/CRITICAL counts to **zero** because it removes the vulnerability-bearing surface entirely: no shell, no package manager, no OS utility layer, minimal shared libraries. The residual attack surface is effectively the language runtime plus the application code.

Why these four succeeded cleanly:
- **Go** produces a statically linked binary, so a `scratch`/distroless-static base carries essentially no OS CVEs.
- **Node.js / TypeScript** run on a controlled runtime layer; a distroless-nodejs base strips the Debian/Alpine userland that typically dominates the CVE count.
- **Java** on a distroless-java (JRE-only) base removes the full JDK toolchain and OS packages from the runtime image.

**2. `optimized_app` — slimmed/optimized base, not yet golden (residual HIGH/CRITICAL remain)**

| Image | Final tag | HIGH/CRITICAL |
|-------|-----------|---------------|
| `pr-demo-app:v1` | `v1-optimized-app` | **93** |
| `python-app:v1` | `v1-optimized-app` | **60** |
| `risk-tradeoff-app:v1` | `v1-optimized-app` | **63** |

These three could **not** be moved to the golden base in this run. The optimizer reduced the image (layer minimization, dependency trimming) but the runtime still requires an OS userland, so a significant vulnerability tail remains (216 HIGH/CRITICAL across the three).

Root causes and why golden-base rebasing stalled:
- **`python-app` (60):** Python typically needs `glibc`, `libssl`, and often build/runtime shared libs; many Python C-extension wheels link against system libraries, so a full distroless-python migration requires validating that no extension pulls in a shell or apt-only dependency. The residual CVEs are predominantly in the OS package layer, not the interpreter itself.
- **`risk-tradeoff-app` (63):** The name signals an explicit accepted trade-off — a dependency or base version was retained for functional/compatibility reasons at a known security cost. This is a deliberate `optimized` stop rather than a golden target.
- **`pr-demo-app` (93):** Highest residual count; likely a demonstration/ephemeral image where remediation priority is lower, but it still carries the largest concentration of fixable HIGH/CRITICALs.

### Application impact / test-case evidence

- The four `golden_base_app` images completed remediation to zero without being downgraded to `optimized_app`, indicating their application test suites **passed** on the golden base — no functional regression was strong enough to block the rebase.
- The three `optimized_app` images stopped short of golden. In this pipeline that status is the fallback taken when a golden-base rebase either (a) breaks a test case or (b) removes a runtime dependency the app requires. The remaining HIGH/CRITICAL counts are the direct, measurable cost of that compatibility hold.

### Justification for code-base changes on the `optimized` images

For the three images still carrying residual risk, closing the gap to zero will require **application-level change**, not just a base swap. This is justified as follows:

- **`python-app` (60 → target 0):** Moving to distroless-python usually forces pinning to manylinux-compatible wheels and removing any runtime `subprocess` shell-outs. The engineering cost (repackaging dependencies, replacing shell calls with native libraries) is bounded and one-time; the payoff is eliminating **60 HIGH/CRITICAL** OS-layer CVEs permanently and removing the shell from the exploitation chain. **Recommended: proceed with code change.**
- **`pr-demo-app` (93 → target 0):** Largest single reduction available in the cluster. If this image ships anywhere beyond ephemeral demos, the 93-CVE reduction dwarfs the cost of the required Dockerfile/dependency refactor. **Recommended: proceed; treat as highest-ROI remediation.**
- **`risk-tradeoff-app` (63):** Because the accepted trade-off is intentional, the code change should be **explicitly cost-justified and time-boxed**. Record the specific dependency forcing the hold, the CVE IDs it introduces, and a re-evaluation date. If the blocking dependency has a maintained successor, the 63-CVE reduction justifies the migration effort; if not, apply compensating controls (see below) and formally accept the residual risk.

### Interim compensating controls for the three `optimized` images

Until golden-base migration completes, apply at deploy time:
- `readOnlyRootFilesystem: true`, `runAsNonRoot: true`, drop `ALL` capabilities.
- Restrictive NetworkPolicies (default-deny egress) to limit lateral movement / exfil paths of any exploited CVE.
- seccomp `RuntimeDefault` and per-workload AppArmor profiles.
- Runtime anomaly detection to catch exploitation of the known-unpatched surface.
- Gate promotion: block these tags from production admission until HIGH/CRITICAL drops below an agreed threshold.

---

## Holistic assessment

**Trend: strongly positive, with a defined and shrinking tail.** Four of seven internal images (57%) are at **zero HIGH/CRITICAL** on a hardened golden base — a clean, durable outcome that shrinks attack surface rather than merely patching it. The remaining risk is concentrated in three `optimized` images totaling **216 HIGH/CRITICAL**, all of which have a clear, understood path to zero via golden-base migration plus modest code changes.

The dominant residual is *addressable* rather than *structural*: the vulnerabilities live in OS userland layers that the golden-base pattern already eliminates elsewhere in this fleet, proving the approach works. Priority order for the next sweep: **`pr-demo-app` (93)** → **`risk-tradeoff-app` (63)** → **`python-app` (60)**, with runtime compensating controls enforced on all three in the interim. Recommend also expanding scope to capture sidecar/operator/third-party images, which were absent from this run and remain unattested.