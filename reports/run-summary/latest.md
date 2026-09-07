# Run Summary — external & internal scopes

# Container Vulnerability Remediation — Run Summary

## External images

No third-party images were in scope for this remediation run. The sweep operated exclusively against internally owned application images.

**Improvements achieved:** None applicable — there were no external images to remediate this cycle.

**Risk factors still present:** The absence of external images from *this run* is not evidence that the cluster is free of third-party image risk. It typically means one of the following, each of which carries residual risk:

- External images may be pinned/cached and were simply not re-scanned this cycle.
- Third-party workloads (databases, sidecars, ingress controllers, service meshes, observability agents) may be deployed via Helm charts or operators whose images sit outside this pipeline's discovery scope.
- Vendor images are, by definition, not rebuildable by us — we cannot patch their layers directly.

**Concrete mitigation techniques for the external attack surface going forward:**

- **Upstream-watch guidance:** Subscribe to vendor security advisories and CVE feeds for every third-party image in use. Track upstream digests and gate promotion on a fresh scan of the *pulled digest*, not the mutable tag.
- **Pin by digest, not tag:** Replace `image: repo/name:tag` with `image: repo/name@sha256:...` to prevent silent drift and to make scan results reproducible.
- **Compensating controls where no patch exists:**
  - Run with a restricted `securityContext` (`runAsNonRoot`, `readOnlyRootFilesystem`, dropped Linux capabilities, `seccompProfile: RuntimeDefault`).
  - Apply `NetworkPolicy` egress/ingress restrictions to reduce exploitability of any latent vuln.
  - Enforce admission policy (Kyverno/OPA Gatekeeper) to block images above a CVE severity threshold.
- **Mirror and re-base where licensing permits:** Pull vendor images into an internal registry, and where the vendor ships on an outdated distro base, evaluate distroless/minimal re-basing if the vendor supports it.

**Action:** Confirm whether external images are genuinely absent from the cluster or merely out of the scanner's discovery scope. If the latter, expand asset discovery before drawing any assurance from this run.

## Internal images

Four of five internal applications reached **`golden_base_app`** status with **zero remaining HIGH/CRITICAL** vulnerabilities. One application (`python-app`) reached only **`optimized_app`** status and retains a significant residual load.

### Base image selections and rationale

| Image | Final tier | Base strategy | HIGH/CRITICAL |
|---|---|---|---|
| `go-app:v1` | golden_base_app | Golden/minimal (distroless or scratch-class) base | 0 |
| `java-app:v1` | golden_base_app | Golden minimal JRE base | 0 |
| `nodejs-app:v1` | golden_base_app | Golden minimal Node base | 0 |
| `typescript-app:v1` | golden_base_app | Golden minimal Node base | 0 |
| `python-app:v1` | optimized_app | Optimized (slimmed) base only — golden not achieved | 60 |

**Why the golden bases improve posture:** The `golden_base_app` tier reflects a move to a minimal, curated base image (distroless/scratch-class for Go; slimmed language runtimes for Java/Node/TypeScript). This eliminates the OS package manager, shells, and general-purpose userland utilities that account for the majority of CVE-bearing packages in a typical image. The security wins are structural, not just cosmetic:

- **Reduced package count → reduced CVE surface.** Fewer installed packages means fewer things to patch and fewer future CVEs.
- **No shell / no package manager → materially harder post-exploitation.** An attacker who achieves RCE cannot trivially pivot, install tooling, or run arbitrary binaries.
- **Sustained zero HIGH/CRITICAL** across four images demonstrates the golden base is a durable, repeatable pattern for the Go, Java, and Node/TypeScript stacks.

**Go in particular** benefits most: a statically compiled binary on a scratch/distroless base carries essentially no OS-level attack surface, which is consistent with the clean result.

### Test-case impact

The four golden-base images reached golden status, indicating their application test suites passed under the minimal base — the runtime behavior was preserved despite removing the shell and userland. No functional regressions are evidenced for `go-app`, `java-app`, `nodejs-app`, or `typescript-app`.

**`python-app` is the exception.** It stalled at `optimized_app` and did **not** progress to the golden base. This pattern is characteristic of a Python workload that:

- Depends on native/C-extension wheels (e.g. `numpy`, `cryptography`, `pillow`, `psycopg2`) that need system shared libraries or a build toolchain not present on a golden/distroless base, **or**
- Executes shell-outs / `subprocess` calls, or relies on runtime package resolution, which break when the shell and package manager are removed.

The 60 remaining HIGH/CRITICAL findings almost certainly originate from the fuller OS base that the optimized tier still carries.

### Justification for code-base changes on `python-app`

Moving `python-app` from 60 → near-0 HIGH/CRITICAL requires the same golden base the other four already run on, and that likely forces code and build changes. This is justified:

- **The residual risk is concentrated.** `python-app` now represents effectively 100% of the internal HIGH/CRITICAL exposure. It is the single highest-leverage target in the cluster.
- **The required changes are bounded and well-understood:**
  - Adopt a **multi-stage build** — compile/install wheels in a full builder stage, then copy only the resolved virtualenv/artifacts into a distroless Python runtime. This removes the build toolchain and OS package surface from the final image without sacrificing native dependencies.
  - Replace any `subprocess`/shell-outs with native library calls or vendored static binaries, so removing the shell doesn't break runtime paths.
  - Pin Python dependencies to patched versions where the CVEs are in the app's own wheels rather than the OS layer.
- **Cost/benefit:** The engineering cost is a one-time refactor of the Dockerfile and a small number of runtime call sites. The benefit is elimination of the cluster's entire remaining critical-vuln backlog plus the same post-exploitation hardening the other four services already enjoy. That trade strongly favors making the change.

**Interim compensating controls for `python-app` until golden base lands:**
- Enforce `readOnlyRootFilesystem`, `runAsNonRoot`, drop all capabilities, `seccompProfile: RuntimeDefault`.
- Apply restrictive `NetworkPolicy` to limit blast radius of the 60 open findings.
- Ensure admission control does not block this image only if an explicit, time-boxed exception is recorded — do not let it become permanent drift.

## Holistic assessment

The trend is strongly positive. Four of five owned applications are on golden, minimal bases with **zero HIGH/CRITICAL** vulnerabilities and no evidenced functional regression — the golden-base pattern is proven across Go, Java, and Node/TypeScript stacks. The entire remaining internal critical-severity exposure is now isolated to a **single service, `python-app` (60 findings)**, which converts a diffuse problem into one well-scoped, high-leverage engineering task with a clear remediation path (multi-stage build → distroless Python). The principal blind spot is external/third-party imagery, which was out of scope this run and must be brought under continuous discovery and digest-pinned scanning before the cluster's posture can be considered fully understood. Net: internal posture is near-hardened and trending toward complete; close out `python-app` and expand scan coverage to external images to reach steady state.