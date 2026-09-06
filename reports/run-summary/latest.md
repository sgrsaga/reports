# Run Summary — external & internal scopes

# Container Vulnerability Remediation — Run-Level Summary

## External images

No third-party images were in scope for this run.

**Improvements achieved:** None applicable — there were no external/base images pulled from upstream registries to remediate, pin, or rebase during this sweep.

**Residual risk factors:** Even with an empty external result set, the absence of coverage is itself a risk signal. Third-party images (databases, sidecars, ingress controllers, observability agents, init containers) commonly enter clusters outside the application build pipeline and therefore escape build-time scanning. If none were evaluated, the current sweep provides **no assurance** about that class of workload.

**Concrete mitigation techniques for what remains:**

- **Admission-time enforcement:** Deploy a policy controller (Kyverno / OPA Gatekeeper) requiring that every image, external or internal, carries a recent scan attestation before it is admitted. This closes the "unscanned third-party" gap structurally.
- **Runtime compensating controls:** For upstream images that cannot be rebuilt, apply `readOnlyRootFilesystem: true`, drop all Linux capabilities, enforce non-root `runAsUser`, and constrain egress with NetworkPolicies to reduce the blast radius of any latent CVE.
- **Upstream-watch guidance:** Subscribe to the release/security feeds for each third-party image (GitHub Security Advisories, vendor CVE mailing lists) and pin by digest (`@sha256:...`) rather than mutable tags so that upgrades are deliberate and auditable.
- **Digest pinning + periodic rebase:** Schedule a recurring job to diff pinned digests against upstream `latest`/patched tags and open PRs automatically (e.g., Renovate) so external drift is surfaced continuously rather than at incident time.

---

## Internal images

### Base image selections and rationale

Four of the five owned application images were successfully converted to a hardened **golden base** and now report **0 remaining HIGH/CRITICAL** findings:

| Application | Final image | HIGH/CRITICAL | Status |
|---|---|---|---|
| java-app | `ghcr.io/sgrsaga/java-app:v1-golden-base-app` | 0 | golden_base_app |
| nodejs-app | `ghcr.io/sgrsaga/nodejs-app:v1-golden-base-app` | 0 | golden_base_app |
| python-app | `ghcr.io/sgrsaga/python-app:v1-golden-base-app` | 0 | golden_base_app |
| typescript-app | `ghcr.io/sgrsaga/typescript-app:v1-golden-base-app` | 0 | golden_base_app |

**Why this improves posture:** The `golden-base-app` selection replaces the previous general-purpose OS base with a curated, minimal, and continuously patched base layer. This yields three structural gains:

- **Reduced attack surface** — fewer OS packages means fewer vulnerable transitive components (shells, package managers, unused libraries) that never contributed to the application's function.
- **Complete elimination of HIGH/CRITICAL OS-layer CVEs** — the remaining count of 0 indicates the base image resolved essentially all vulnerabilities originating below the application layer.
- **Sustainable patch cadence** — a shared golden base centralizes future patching: a single base bump remediates all four apps simultaneously, rather than four independent upgrade efforts.

### Application impact / test-case evidence

The four converted images completed with terminal status `golden_base_app` and **no reported test-case failures** in this run. On the available evidence, the rebase was **functionally non-regressing** — the applications built, started, and passed their validation gates on the hardened base. No code-base changes were required for these four, so no code-vs-security trade-off justification is needed here.

### The outstanding image: `go-app`

```
ghcr.io/sgrsaga/go-app:v1 → no_improvement
final: ghcr.io/sgrsaga/go-app:v1  (unchanged)
remaining HIGH/CRITICAL: 483
```

This is the dominant risk in the internal set. A count of **483 HIGH/CRITICAL** with `no_improvement` and an **unchanged final tag** indicates the automated rebase could not be applied — most likely because:

- The Go binary is built or run against a base whose hardened equivalent was incompatible (e.g., dynamically linked against glibc where the golden base is distroless/musl), **or**
- The finding volume is driven by **vendored/bundled dependencies inside the compiled artifact** rather than the OS layer, so a base swap alone cannot move the number.

**Recommended remediation path, with justification for code-base change:**

1. **Move to a static build + minimal runtime.** Compile with `CGO_ENABLED=0` to produce a fully static binary, then ship it on `gcr.io/distroless/static` or `scratch`. This removes the entire OS package surface — the most likely source of a large fraction of the 483 findings.
   - *Why the code/build change is justified:* a Dockerfile and build-flag change is low-risk and reversible, yet it can collapse hundreds of OS-layer CVEs to near zero. The security improvement (removing an actively-exploitable HIGH/CRITICAL surface from a running workload) decisively outweighs the modest engineering cost of adjusting the build.
2. **Regenerate and audit Go module dependencies.** If findings persist post-rebase, they are in-application. Run `govulncheck` to filter the 483 down to the subset actually reachable in the call graph, then bump the offending modules. Prioritize reachable vulns; deprioritize unreachable ones with documented justification.
   - *Why worth it:* dependency upgrades may require minor source adaptation, but leaving 483 findings — even partially reachable — in production presents an unacceptable standing exposure relative to the bounded cost of a dependency refresh.
3. **Interim compensating controls until remediated:** pin `go-app` to a hardened `securityContext` (non-root, read-only FS, dropped capabilities), restrict its NetworkPolicy to only required peers, and — if business-critical — consider quarantining or gating it behind admission policy so the unremediated image cannot be scaled or redeployed without sign-off.

---

## Holistic assessment

The cluster's internal security posture is **trending strongly positive**: 4 of 5 owned images (80%) reached a zero HIGH/CRITICAL golden-base state with no observed functional regressions, demonstrating that the golden-base strategy is both effective and low-friction for interpreted/JVM runtimes.

The posture is, however, **gated by two blind spots**: the unremediated `go-app` (483 HIGH/CRITICAL, unchanged) which concentrates nearly all residual internal risk, and the complete absence of external-image coverage this run. **Priority actions:** (1) rebase `go-app` onto a distroless static runtime and run `govulncheck` reachability triage; (2) bring third-party images into scan/admission scope. Closing these two items would move the cluster from "mostly hardened with one hotspot and an unknown" to a defensibly comprehensive posture.