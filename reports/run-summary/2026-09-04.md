# Run Summary — external & internal scopes

# Container Vulnerability Remediation Sweep — Run Summary

## External images

**No external (third-party) images were processed in this run.**

There is no scan delta, remediation activity, or residual risk to report for
externally-sourced images. This is either because no third-party images are
currently in scope for the cluster, or because none were selected for this
sweep.

**Recommended posture for external images going forward:**

- **Pin and digest-lock.** Reference third-party images by immutable digest
  (`@sha256:...`) rather than mutable tags to prevent silent drift and to make
  scan results reproducible.
- **Upstream-watch guidance.** Subscribe to upstream release/security channels
  (GitHub security advisories, vendor CVE feeds) and mirror images into an
  internal registry so remediation cadence is decoupled from upstream
  availability.
- **Compensating controls for un-fixable CVEs.** Where an external image ships
  a vulnerability with no upstream fix, apply runtime mitigations:
  - restrictive `seccomp`/AppArmor profiles,
  - drop all Linux capabilities and run non-root,
  - network policy egress/ingress restriction to shrink blast radius,
  - admission-policy gating (e.g., block deploys exceeding a HIGH/CRITICAL
    threshold).
- **Re-scan on every ingest.** Treat external images as untrusted until scanned;
  gate promotion into the cluster on a passing scan.

> **Action:** Confirm whether external images are genuinely out of scope. A
> run with zero external results should be explicitly validated, not assumed.

---

## Internal images

Five owned application images were processed. Four were successfully migrated to
hardened **golden base** images with **zero remaining HIGH/CRITICAL findings**.
One image could not be improved.

| Image | Status | Final artifact | Remaining HIGH/CRITICAL |
|---|---|---|---|
| `go-app:v1` | `no_improvement` | `go-app:v1` | **483** |
| `java-app:v1` | `golden_base_app` | `java-app:v1-golden-base-app` | 0 |
| `nodejs-app:v1` | `golden_base_app` | `nodejs-app:v1-golden-base-app` | 0 |
| `python-app:v1` | `golden_base_app` | `python-app:v1-golden-base-app` | 0 |
| `typescript-app:v1` | `golden_base_app` | `typescript-app:v1-golden-base-app` | 0 |

### Successful golden-base migrations (Java, Node.js, Python, TypeScript)

**Base image selection and rationale.**
Each of these four applications was rebased onto a curated **golden base image**
(distroless/minimal runtime-only base). This drives HIGH/CRITICAL counts to zero
for two structural reasons:

- **Reduced package surface.** Golden bases strip shells, package managers, and
  general-purpose OS utilities. Fewer OS packages means fewer CVE-bearing
  components — most residual HIGH/CRITICALs in typical app images originate from
  the base OS layer, not application code.
- **Maintained provenance.** Golden bases are centrally patched and version-
  controlled, so future sweeps inherit fixes automatically rather than requiring
  per-app remediation.

**Security-posture improvement.**
- Eliminates entire classes of exploit tooling at runtime (no `sh`, no `apt`),
  raising the cost of post-exploitation and lateral movement.
- The `0` residual count means these images can pass a strict admission gate
  with no exceptions or waivers required.

**Application impact / test-case evidence.**
The `golden_base_app` status with `0` residuals indicates the rebuild completed
and passed validation. Note the following operational caveats that must be
verified in the app teams' CI before promotion:

- **Distroless has no shell** — any container that relied on `exec`-ing a shell
  for health checks, entrypoint scripts, or debugging will break. Health probes
  must use HTTP/gRPC or a compiled binary, not `sh -c`.
- **Runtime-only bases** may omit CA certificate bundles, timezone data, or libc
  variants — confirm TLS egress and time-dependent logic still function.
- **Non-root by default** — verify the app does not write to privileged paths or
  bind to ports <1024.

Where these caused **test-case failures**, the fix is a manifest/entrypoint
change (probe reconfiguration, adding `ca-certificates`/`tzdata` layers), not a
regression in the security choice. **Recommendation:** attach the per-app test
results to this run before promoting the `-golden-base-app` tags to production.

### `go-app:v1` — no improvement (483 HIGH/CRITICAL remaining)

**Assessment.**
The remediation engine could not rebase this image onto a golden base, and
483 HIGH/CRITICAL findings persist. A count this high strongly suggests the
image is built on a **full/legacy OS base** (or a fat base with a bundled OS
toolchain), rather than something intrinsic to Go — Go compiles to a static
binary and is one of the *easiest* stacks to ship on a minimal base.

**Why this is likely fixable — and why code changes are justified.**
Go's static-linking model makes it an ideal candidate for `scratch` or
`gcr.io/distroless/static`. Moving to such a base should collapse the 483
findings dramatically, because nearly all of them are almost certainly
inherited from the current base OS layer, not the Go application.

The blockers that require code-base / build changes, and their justification:

- **Static build flags.** Requires `CGO_ENABLED=0` and a fully static binary.
  If the app currently depends on CGO (e.g., certain DB drivers, `net` with
  system resolver), those dependencies must be swapped for pure-Go equivalents.
  *Justification:* eliminating a 483-finding attack surface is a decisive
  security win that categorically outweighs the engineering cost of a pure-Go
  dependency swap.
- **CA certs / tzdata.** Must be copied explicitly into the minimal image.
  *Justification:* a trivial Dockerfile change relative to the risk removed.
- **No shell for debugging/health checks.** Same probe migration as above.

**Interim compensating controls (until rebased):**
- Quarantine via admission policy — block promotion of `go-app:v1` to
  production namespaces until the count is remediated.
- Apply tight `seccomp`, capability-drop, read-only root filesystem, and
  restrictive NetworkPolicy to constrain blast radius in the meantime.
- Triage the 483 findings for *reachable/exploitable* subset — but treat this
  as stopgap, not resolution.

> **Action (owner: go-app team):** Rebuild on `distroless/static` or `scratch`
> with `CGO_ENABLED=0`. Re-scan and target `0` residuals, matching the other
> four services.

---

## Holistic assessment

The cluster's internal security posture is trending **strongly positive**:
**4 of 5 owned images (80%) reached zero HIGH/CRITICAL** via golden-base
adoption, demonstrating that the golden-base program is effective and
repeatable. The single outlier, `go-app`, is not a limitation of the approach
but an un-migrated legacy base — and it is the **lowest-effort, highest-payoff**
remaining item given Go's static-binary suitability for minimal bases.

**Net direction:** posture improving and consolidating around a standardized,
centrally-patched base strategy. Two gaps must be closed to declare the sweep
complete: (1) remediate `go-app` to eliminate the concentrated 483-finding risk,
and (2) validate per-app test results and probe/entrypoint compatibility before
promoting the four `-golden-base-app` artifacts to production. Once `go-app` is
rebased, the cluster is positioned to enforce a **zero HIGH/CRITICAL admission
gate** on all internal workloads without waivers.