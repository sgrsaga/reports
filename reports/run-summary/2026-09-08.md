# Run Summary — external & internal scopes

# Container Vulnerability Remediation — Run Summary

## External images

No external (third-party) images were evaluated in this run.

- **Improvements achieved:** None to report — there are no external image results for this sweep.
- **Risk factors still present:** Not assessable from this run's evidence; no external image inventory or scan data was recorded.
- **Possible mitigation techniques:** Not applicable this run. When external images are next included, the report should carry their scan deltas and remaining HIGH/CRITICAL counts so that compensating controls (network policy restrictions, runtime admission gating, read-only root filesystems) and upstream-watch guidance (tracking upstream fixed versions, subscribing to vendor advisories) can be recommended against concrete findings.

## Internal images

### Skipped this run (unchanged since last run)

The following images were **not reworked this run** because their digests were unchanged. The statuses below are recorded outcomes carried forward from prior runs, not new work:

| Image | Status | Final artifact | Remaining HIGH/CRITICAL | Recorded outcome timestamp |
|---|---|---|---|---|
| `ghcr.io/sgrsaga/go-app:v1` | `golden_base_app` | `v1-golden-base-app` | 0 | 2026-09-07T07:45:32Z |
| `ghcr.io/sgrsaga/java-app:v1` | `golden_base_app` | `v1-golden-base-app` | 0 | 2026-09-07T07:48:49Z |
| `ghcr.io/sgrsaga/nodejs-app:v1` | `golden_base_app` | `v1-golden-base-app` | 0 | 2026-09-07T07:51:31Z |
| `ghcr.io/sgrsaga/pr-demo-app:v1` | `golden_base_app` | `v1-golden-base-app` | 0 | 2026-09-07T11:54:23Z |
| `ghcr.io/sgrsaga/python-app:v1` | `optimized_app` | `v1-optimized-app` | 60 | 2026-09-07T07:55:31Z |
| `ghcr.io/sgrsaga/risk-tradeoff-app:v1` | see below | — | — | — |
| `ghcr.io/sgrsaga/typescript-app:v1` | `golden_base_app` | `v1-golden-base-app` | 0 | 2026-09-07T07:59:12Z |

Notes on the skipped set:
- Five images (`go-app`, `java-app`, `nodejs-app`, `pr-demo-app`, `typescript-app`) remain at `golden_base_app` with **0 remaining HIGH/CRITICAL** per their recorded outcomes.
- `python-app` remains at `optimized_app` with **60 remaining HIGH/CRITICAL** per its recorded outcome. This is the largest outstanding internal exposure. No reason for the 60 residual findings is recorded in the evidence, and no remediation trail was run this cycle because the digest was unchanged. This image should be prioritized for a forced re-sweep (e.g., a rebuild to change the digest) so remediation can actually execute against it.

### `ghcr.io/sgrsaga/risk-tradeoff-app:v1` — full remediation trail

This is the only internal image with new work evidenced this run. Base artifact of record: `ghcr.io/sgrsaga/chainguard-python:latest-dev-golden-base`.

**Step-by-step trail (Critical, High):**

1. **[os-patch] debian blanket upgrade in base stage** → **passed**, (C,H) (23, 1711) → (7, 268); tests passed. *Large initial reduction from OS package upgrades.*
2. **[os-patch] debian blanket upgrade in base stage** → **rejected: no improvement** (7, 268) → (7, 268); tests passed. *No further OS-level gains available.*
3. **[llm-base] debian:12-slim** → **rejected: build/test failed** — `docker build failed (target=test)`; tests FAILED. *Candidate base did not build/test successfully.*
4. **[restructure] builder(python:3.11.4-slim) + runtime(debian:12-slim)** → **passed**, (7, 268) → (5, 56); tests passed. *Multi-stage split (build toolchain out of the runtime) materially reduced HIGH count.*
5. **[os-patch] debian blanket upgrade in base stage** → **rejected: no improvement** (5, 56) → (5, 56); tests passed.
6. **[llm-base] cgr.dev/chainguard/python:latest-dev** → **passed**, (5, 56) → (1, 4); tests passed. **Winning base selection.** *Largest balanced drop, tests green.*
7. **[os-patch] debian blanket upgrade in base stage** → **rejected: final build/rescan failed** — `docker build failed (target=None)`; tests passed. *Post-selection OS patch attempt failed to build.*
8. **[llm-base] redhat/ubi9:latest** → **rejected: no improvement** (1, 4) → (1, 13); tests passed. *Increased HIGH.*
9. **[llm-base] gcr.io/distroless/python3-debian12:latest** → **rejected: no improvement** (1, 4) → (3, 49); tests passed. *Increased both C and H.*
10. **[llm-base] amazonlinux:2023** → **rejected: no improvement** (1, 4) → (1, 6); tests passed. *Increased HIGH.*
11. **[dep-bump#1] httpx==0.23.0, setuptools==78.1.1 (appended, transitive), wheel==0.46.2 (appended, transitive)** → **passed**, (1, 4) → (1, 3); tests passed.
12. **[dep-bump#2] wheel==0.46.2, h11==0.16.0 (appended, transitive), jaraco.context==6.1.0 (appended, transitive)** → **rejected: build/test failed** — `docker build failed (target=test)`; tests FAILED.

**Base selection outcome and rationale (from adjudication):**
Candidate [6] **cgr.dev/chainguard/python:latest-dev** was selected because it delivered the single biggest balanced drop to **CRITICAL=1 / HIGH=4** with tests passing and no restructuring risk, and its remaining CVEs are characterized as low-reachability Python packaging/library issues rather than reachable OS-level exploits. Candidate [8] (dep-bump#2 path referenced in adjudication) technically reaches HIGH=3 but was rejected because it introduces **CVE-2025-43859 (h11, CRITICAL)** — a genuinely reachable request-smuggling issue (adjudication text is truncated in the evidence).

**Application impact (test-case failures evidenced):**
- Candidate base **debian:12-slim** (step 3): build/test failed at `target=test`.
- **dep-bump#2** (step 12): build/test failed at `target=test`.
- Post-selection **os-patch** (step 7): final build/rescan failed at `target=None`.
- All other steps report tests passing.

**Adjudication-supplied code fixes (relayed) and justification:**
- **Pin `httpx` to a patched line that does not drag in vulnerable `h11`** — avoid the 0.23.0 downgrade from candidate [8]; use a current httpx (e.g., `>=0.27`) with `h11>=0.16` to close **CVE-2025-43859** and **CVE-2021-41945** together.
- **Bump `setuptools` to `>=78.1.1`** to remediate **CVE-2024-6345** and **CVE-2025-47273**.
- **Bump `wheel` to `>=0.46.2`** to close **CVE-2026-24049**.
- **For CVE-2024-23342 (ecdsa Minerva timing attack):** assess reachability — if ECDSA signing isn't used, remove the `ecdsa` dependency; otherwise migrate to the `cryptography` library for constant-time operations (adjudication text truncated).
- **Verify these pins in the Chainguard runtime image and re-scan** to confirm CRITICAL=0 without introducing new transitive CRITICALs.

**Net result for this image:** From (C,H) (23, 1711) at start to **(1, 3)** at the last passing step (dep-bump#1), via OS patching, multi-stage restructuring, the Chainguard base selection, and a validated dependency bump. Residual: 1 CRITICAL / 3 HIGH, pending the adjudication code fixes above to drive CRITICAL toward 0.

---

## Holistic assessment

- **Trend is positive but stalled this cycle.** Six of seven internal images did no new work (digests unchanged); their posture is carried forward, not re-verified. Five are at `golden_base_app` with 0 HIGH/CRITICAL.
- **Two concentrated exposures remain:**
  - `python-app` at **60 HIGH/CRITICAL** (unchanged, no trail this run) — the single largest internal risk. Because it was skipped on an unchanged digest, remediation never executed. Force a rebuild/re-sweep to unblock it.
  - `risk-tradeoff-app` reduced dramatically to **(1, 3)** but still carries 1 CRITICAL and 3 HIGH; the adjudication provides concrete, actionable dependency pins to close them.
- **Actionable next steps:** (1) trigger a digest change on `python-app` to force remediation; (2) apply the adjudicated `httpx`/`setuptools`/`wheel`/`ecdsa` fixes to `risk-tradeoff-app` and re-scan to confirm no new transitive CRITICALs; (3) treat the `golden_base_app` images as "verified only as of their recorded timestamps" and schedule periodic re-scans since skip-on-unchanged-digest can mask newly disclosed CVEs against static digests.

*(No claims here are inferred from image names; where the evidence records no reason — e.g., the 60 residual findings on `python-app` — that absence is stated rather than filled in.)*