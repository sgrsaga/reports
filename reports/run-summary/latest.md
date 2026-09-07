# Run Summary — external & internal scopes

# Container Vulnerability Remediation — Run-Level Summary

## External images

### Summary of improvements
One external (third-party) image was in scope this run:

- `ghcr.io/sgrsaga/risk-tradeoff-app:v1` — status `scan_error`; final artifact unchanged at `ghcr.io/sgrsaga/risk-tradeoff-app:v1`; remaining HIGH/CRITICAL: 0.

No remediation improvement can be attributed to this run. The terminal status is `scan_error`, which means the scan itself did not complete successfully. The reported "remaining HIGH/CRITICAL: 0" is therefore **not** a trustworthy attestation of a clean image — with a scan error, a zero count most plausibly reflects the absence of a successful scan rather than a verified absence of vulnerabilities. The evidence does not record the cause of the scan error.

### Risk factors still present
- **Unverified security state.** Because the scan errored, we have no reliable vulnerability inventory for this image. Treat its actual HIGH/CRITICAL exposure as **unknown**, not zero.
- **Third-party provenance.** As an external image we do not own the build, so remediation is gated on upstream and cannot be fixed by rebuilding here.
- The root cause of the `scan_error` is not recorded in the evidence.

### Concrete mitigations for what remains
- **Re-run and triage the scan first.** The immediate action is to resolve the `scan_error` (e.g., verify registry pull/auth, image manifest/media-type compatibility, and scanner timeouts) and obtain a real vulnerability inventory before drawing any conclusion. Do not report this image as clean until a scan completes.
- **Compensating controls until verified:** pin by digest, run the workload with a restrictive runtime posture (non-root, read-only root FS, dropped capabilities, seccomp/AppArmor), and constrain network egress via NetworkPolicy to limit blast radius of an unknown-state image.
- **Upstream-watch guidance:** subscribe to the upstream repository's release/security advisories, track new tags/digests, and re-scan on every upstream publish so a future working scan can replace the current unknown state.
- **Admission gating:** consider blocking promotion of images whose latest scan status is `scan_error` so an errored scan cannot be mistaken for a passing one.

---

## Internal images

Six internal (owned) images were in scope. **Five were SKIPPED** — their digests were unchanged since the last run, so no new work was performed; the statuses below are recorded outcomes from prior runs, reported here as unchanged-since-last-run. **One (`pr-demo-app`) received active remediation this run.**

### Unchanged since last run (SKIPPED — no work this run)
| Image | Recorded status | Final artifact | Remaining H/C | Recorded outcome time |
|---|---|---|---|---|
| `go-app:v1` | `golden_base_app` | `go-app:v1-golden-base-app` | 0 | 2026-09-07T07:45:32Z |
| `java-app:v1` | `golden_base_app` | `java-app:v1-golden-base-app` | 0 | 2026-09-07T07:48:49Z |
| `nodejs-app:v1` | `golden_base_app` | `nodejs-app:v1-golden-base-app` | 0 | 2026-09-07T07:51:31Z |
| `python-app:v1` | `optimized_app` | `python-app:v1-optimized-app` | **60** | 2026-09-07T07:55:31Z |
| `typescript-app:v1` | `golden_base_app` | `typescript-app:v1-golden-base-app` | 0 | 2026-09-07T07:59:12Z |

Note on `python-app:v1`: its last recorded outcome still carries **60 remaining HIGH/CRITICAL** at status `optimized_app` (i.e., not driven to a golden base). This is a standing exposure carried forward unchanged; it was not re-worked this run because the digest was unchanged. It should be prioritized for a forced re-run.

### Actively remediated this run: `pr-demo-app:v1`
**Final status:** `golden_base_app` → `ghcr.io/sgrsaga/pr-demo-app:v1-golden-base-app`, remaining HIGH/CRITICAL: **0**.
**Starting base artifact:** `cgr.dev/chainguard/python:latest-dev`.

The adjudication trail shows four candidate steps. Reproduced in order with their measured (CRITICAL, HIGH) transitions and test results:

1. **[os-patch] Debian blanket upgrade in base stage — ACCEPTED (partial).**
   `(C,H): (22, 1718) → (6, 273)`; tests passed (C=6, H=273).
   Large reduction in both CRITICAL and HIGH counts with no test regression, so it was retained.

2. **[os-patch] Debian blanket upgrade in base stage (repeat) — REJECTED: no improvement.**
   `(C,H): (6, 273) → (6, 273)`; tests passed. A second blanket upgrade produced no further vulnerability reduction, so it added no value and was not kept.

3. **[llm-base] `cgr.dev/chainguard/python:latest-dev` — REJECTED: build/test failed.**
   The evidence records a `docker build failed (target=test)` with a truncated stdout fragment (`…ng iniconfig-2.3.0-py3-none-any.whl.metadata (2.5 k…`) and `tests FAILED`. Because the candidate could not build/test successfully, it was discarded. The full failure detail beyond this fragment is not recorded.

4. **[restructure] builder(`python:3.11.4-slim`) + runtime(`cgr.dev/chainguard/python:latest-dev`) — ACCEPTED (winner).**
   `(C,H): (6, 273) → (0, 0)`; tests passed (C=0, H=0).
   A multi-stage split — building on `python:3.11.4-slim` and running on the Chainguard `python:latest-dev` runtime — drove HIGH/CRITICAL to zero while all tests continued to pass.

**Why the winning selection improves posture:** the multi-stage restructure separates build-time tooling (on `python:3.11.4-slim`) from the runtime image (Chainguard `python:latest-dev`), eliminating all remaining HIGH/CRITICAL findings (`6C/273H → 0/0`) that the OS blanket upgrade alone could not clear, and it did so with a passing test suite — so the hardening carried no evidenced functional cost.

**Application impact (test evidence):**
- Accepted steps 1 and 4, plus the rejected no-improvement step 2, all recorded **tests passed** — no functional regression from the retained changes.
- The rejected `[llm-base]` candidate (step 3) recorded **tests FAILED** during `docker build (target=test)`. This is the sole test-failure signal, and it caused that candidate to be rejected rather than shipped, so no application impact reaches the final artifact.

**Code fixes supplied by adjudication:** none are recorded in the evidence. The remediation was achieved through base-image/structure selection (OS patching and the builder+runtime restructure), not through source-code changes. No code diffs or justifications for such were provided.

---

## Holistic assessment — cluster security-posture trend

- **Owned images are largely at a hardened steady state.** Five of six internal images are at `golden_base_app` with **0 HIGH/CRITICAL** and were legitimately skipped as unchanged — a stable, low-churn posture rather than newly-earned wins this run.
- **This run's net gain is `pr-demo-app`**, cleanly driven `22C/1718H → 0C/0H` via a disciplined trail (partial OS patch retained, redundant patch and a failing base candidate correctly rejected, restructure winning) with tests green throughout. This is the model outcome to replicate.
- **Two watch items remain, both traceable to evidence:**
  1. **`python-app:v1` carries 60 HIGH/CRITICAL** at `optimized_app`, unresolved and merely carried forward unchanged. **Action:** force a re-run (bypass digest-skip) and target a golden-base outcome, ideally via the same builder+runtime restructure pattern that succeeded for `pr-demo-app`.
  2. **`risk-tradeoff-app:v1` is in `scan_error`** — its 0-count is unverified. **Action:** fix and re-run the scan before trusting any number, and apply the runtime/admission compensating controls above in the interim.

**Trend:** improving and mostly stabilized for owned images, but the true residual risk is currently **understated** by two records — one unresolved (`python-app`, 60 H/C) and one unverifiable (`risk-tradeoff-app`, errored scan). Prioritize forcing re-runs on both so the next report reflects verified reality rather than carried-forward or errored states.