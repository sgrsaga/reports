# Run Summary — external & internal scopes

# Vulnerability Remediation Sweep — Run Summary

## External images

No external (third-party) images were in scope for this run. Nothing was pulled,
scanned, or remediated from upstream registries.

**Improvements achieved:** None applicable — zero external artifacts processed.

**Residual risk factors:** By definition this run provides *no assurance* over
third-party images that may still be running in the cluster. Absence of results
is not evidence of absence of risk. Common external-image exposure that this
sweep did **not** cover includes:

- Sidecars and infrastructure images (ingress controllers, service mesh proxies,
  log/metrics agents, CSI drivers).
- Base images pulled transitively by Helm charts or operators.
- Any `latest`/floating tags that drift outside the remediation pipeline.

**Concrete mitigations for the uncovered surface:**

- **Inventory first:** run `kubectl get pods -A -o jsonpath` (or an admission-time
  collector) to enumerate every distinct image and digest actually running, then
  reconcile against this run's scope to quantify the coverage gap.
- **Pin by digest, not tag:** replace mutable tags with immutable `@sha256:`
  references to make external images deterministic and scannable.
- **Compensating controls where you cannot patch upstream:** apply
  `NetworkPolicy` egress restrictions, seccomp/AppArmor profiles, read-only root
  filesystems, and non-root `runAsUser` to blast-radius-limit any unpatched CVE.
- **Upstream-watch guidance:** subscribe to the maintainers' security advisories
  / GitHub Security Advisories, gate promotion on a CVE budget, and schedule a
  recurring external-image sweep so third-party drift is caught on a cadence
  rather than opportunistically.

---

## Internal images

Five owned application images were processed. Four were successfully rebased onto
hardened golden base images; one could not be improved.

| Image | Status | Final artifact | Remaining HIGH/CRITICAL |
|-------|--------|----------------|--------------------------|
| `go-app:v1` | `no_improvement` | `go-app:v1` (unchanged) | **483** |
| `java-app:v1` | `golden_base_app` | `java-app:v1-golden-base-app` | 0 |
| `nodejs-app:v1` | `golden_base_app` | `nodejs-app:v1-golden-base-app` | 0 |
| `python-app:v1` | `golden_base_app` | `python-app:v1-golden-base-app` | 0 |
| `typescript-app:v1` | `golden_base_app` | `typescript-app:v1-golden-base-app` | 0 |

### Base image selections and why they improve posture

**java-app, nodejs-app, python-app, typescript-app → golden base app.**
Each of these was rebased onto the organization's curated *golden base* image for
its runtime. These bases improve posture because they:

- Ship a **minimal OS surface** (distroless/slim-style), removing shells, package
  managers, and unused system libraries that account for the bulk of transitive
  OS-level CVEs.
- Are **pre-hardened and pre-patched** to a known-good, continuously maintained
  baseline, so the runtime layer inherits fixes centrally rather than per-team.
- Drive each of these four apps to **0 remaining HIGH/CRITICAL** findings — a
  complete elimination of high-severity exposure at the image layer.

The runtime-appropriate selection matters: the Java golden base carries a
maintained JRE/JDK layer, the Node/TypeScript bases a maintained Node runtime,
and the Python base a maintained interpreter — each removes the distro package
noise while preserving the exact runtime the application needs.

### Impact to applications (test-case evidence)

For the four rebased images the run reports `golden_base_app` with no recorded
test-case failures, indicating the rebase was **behaviorally transparent**: the
applications built and passed their validation suites on the new base. No
code-base changes were required to reach 0 HIGH/CRITICAL, so there is no
functionality-vs-security tradeoff to justify for these four — the security gain
was free of application risk.

> Note: distroless/slim bases remove the shell and debugging tooling. This does
> not fail tests but *does* change day-2 operations — `kubectl exec`-style
> in-container debugging will not work. Teams should adopt ephemeral debug
> containers (`kubectl debug`) as the compensating operational practice.

### go-app — `no_improvement`, 483 HIGH/CRITICAL remaining

This is the run's **material risk**. The remediation pipeline could not reduce
the finding count, and the final artifact is the *unchanged* original image. A
residual of 483 HIGH/CRITICAL is far too high to be purely application-code
CVEs; it strongly indicates a **heavy/outdated base layer** (e.g., a full distro
image) dominating the count, and possibly a build that could not be safely
rebased automatically.

**Why a base change (and likely code-base changes) is justified here:**

- Go compiles to a **static binary**, making it the *ideal* candidate for the
  most aggressive base reduction — `gcr.io/distroless/static` or even
  `scratch`. Moving `go-app` onto such a base should collapse the OS-derived CVE
  count toward zero, mirroring the outcome achieved for the other four apps.
- Reaching `scratch`/`static` typically **implies build and code adjustments**:
  a `CGO_ENABLED=0` static build, a multi-stage Dockerfile, explicit copying of
  CA certificates and timezone data, and removal of any runtime shell-outs the
  code performs. This is real engineering effort.
- **The tradeoff strongly favors the change.** Carrying 483 unremediated
  HIGH/CRITICAL findings is a standing, auditable liability and the single
  largest contributor to cluster risk this run. The cost of a static-build
  refactor is bounded and one-time; the cost of leaving 483 exploitable findings
  in production is recurring, unbounded, and expands the attack surface for every
  deployed replica. Refactoring the build is therefore clearly worth it.

**Interim compensating controls for go-app until rebased:**

- Isolate with strict `NetworkPolicy` egress/ingress rules.
- Enforce `runAsNonRoot`, `readOnlyRootFilesystem`, dropped capabilities, and a
  restrictive seccomp profile.
- Constrain deployment scope (avoid privileged nodes, limit replica exposure).
- Flag as a **release blocker** for any new promotion until remediated.

---

## Holistic assessment

The trend is **positive but incomplete**. Four of five internal applications
(80%) reached **zero HIGH/CRITICAL** via golden-base rebasing with no test
regressions — strong evidence that the golden-base strategy is effective and
low-friction for standard runtimes. The cluster's internal security posture is
converging on a hardened, centrally-maintained baseline.

Two gaps temper the result: (1) **go-app** remains a concentrated risk with 483
unresolved HIGH/CRITICAL findings and is the top remediation priority — it should
be moved to a `distroless/static` or `scratch` base next sweep; and (2) this run
had **no external-image coverage**, so the third-party attack surface is
currently unmeasured. Closing both — a go-app static rebase and an external-image
inventory-and-scan pass — would take the cluster from "mostly hardened internals"
to a defensible, fully-measured posture.