# Task E — Independent reproduction, fix acceptance, and integration

Role: third-party verification. For the main case and one boundary result, reproduce
from the written records alone; check that evidence supports each conclusion, that
fixes address the root cause, and that wording stays within scope. All checks use the
team's own repositories and the canary value `FAKE_SECRET=canary123` only.

Repositories: base `huachen-wang/mini-crm-base`; fork `guowanling124-afk/mini-crm-base`
(branches `a-external-input`, `b-pwn-request`, `c-cache`, `d-secret`, `e-review`).

---

## 1. Independent reproduction of the main case (Task B)

Reproduced from B's records without assistance, by inspecting PR #2 and its Actions runs directly.

| Check | Record says | Independently verified | Result |
| --- | --- | --- | --- |
| PR is a genuine external fork | fork PR | PR #2: `guowanling124-afk:b-pwn-request` → `huachen-wang:main`, author is an external account | Confirmed |
| Controlled-risk config is blocked | stopped at checkout | `lab-b-pwn-request.yml` (run 39551521994, `pull_request_target`) → **Failure**, 8s, at step "Checkout PR head (untrusted)" | Confirmed |
| Cause is a real protection, not a fetch error | checkout protection | Exact error: *"Refusing to check out fork pull request code from a 'pull_request_target' workflow … set 'allow-unsafe-pr-checkout: true' …"* — an explicit security refusal by `actions/checkout` | Confirmed |
| Fixed config runs normally | CI preserved | `lab-b-pwn-request-fixed.yml` (run 39551525650, `pull_request`) → **Success**, 18s; checkout + JDK 21 + `mvn -B test` all passed | Confirmed |

**Conclusion (independently supported):** under the tested configuration the untrusted
fork code never reached the privileged stage — `actions/checkout`'s `allow-unsafe-pr-checkout`
guard (default off) refused the checkout. The fix (switch to `pull_request`, no secret in
the job) keeps normal CI working. This is "protection effective + author-config risk," not
a demonstrated platform vulnerability, and no secret value was read.

Not reproduced in this pass: the opt-in variant (`allow-unsafe-pr-checkout: true`), which would
show the residual risk when an author disables the guard. Recommended as the one addition that
turns B into a complete two-sided result.

## 2. Independent check of one boundary result (Task A)

Task A's package (`fork:a-external-input/verification/`) is self-contained: a platform run
(issue #2 titled `$(date -u +%FT%TZ)`, run #1, 2s) plus a local harness
(`repro-a-injection.py`). Verified that the echoed script already contains the substituted
value (substitution happens before the shell) and that the same issue event left `bot.yml`
skipped — same input, safe handling, no execution. A's classification as **Not Applicable
(our configuration)** is correct and is the control case the other findings are measured against.

## 3. Evidence index

| ID | Task | Evidence | Supports |
| --- | --- | --- | --- |
| EV-A-1 | A | Issue #2 run #1 log (substituted title executed) | `${{ }}` is substituted before the shell → config-level injection (N/A) |
| EV-A-2 | A | `bot.yml` skipped on same event | env-passing is the safe counterpart |
| EV-B-1 | B | `lab-b` run 39551521994 Failure at checkout | default checkout guard blocks fork code under `pull_request_target` |
| EV-B-2 | B | `lab-b-fixed` run 39551525650 Success | fix preserves normal CI |
| EV-B-3 | B | Reproducibility record (SHAs, runner 2.337.0, Ubuntu 24.04) | run is reproducible and precisely versioned |
| EV-C-1 | C | `lab-c` run (poison-cache, Failure 4s) | **incomplete — see §6**; cache write not yet demonstrated |
| EV-D-1 | D | `lab-d-reachability.yml` (presence-only probe) | secret reachability by event type (presence, never value) |
| EV-D-2 | D | `lab-d-secret-masking.yml` (canary masking probe) | masking behaviour by encoding |

(IDs to be reconciled with each author's own numbering before submission.)

## 4. Fix-acceptance table

| Fix | Risk blocked after fix? | Normal function kept? | Evidence |
| --- | --- | --- | --- |
| B: use `pull_request`, no secret in job, `contents: read` | Yes — fork code runs without base secrets | Yes — checkout + JDK + test pass | EV-B-2 |
| Platform default guard (`allow-unsafe-pr-checkout` off) | Yes — checkout refused by default | n/a | EV-B-1 |
| Supply chain: pin actions to SHA (see §5) | Reduces tag-repointing risk | Yes — SHA resolves to same action | §5 |

## 5. Supply-chain / configuration audit

Every third-party action is referenced by a **mutable tag `@v4`**, none pinned to a commit SHA:
`actions/checkout@v4`, `actions/setup-java@v4`, `actions/upload-artifact@v4`, `actions/cache@v4`.

Recommendations, each tied to observed evidence:
- **Pin to commit SHA.** Resolved SHAs already recorded in B's run, e.g. `actions/checkout@v4`
  → `11d5960a326750d5838078e36cf38b85af677262`, `actions/setup-java@v4`
  → `cf277c60eb25467037889841efdb72551f06f6c3`, `actions/upload-artifact@v4`
  → `ea165f8d65b6e75b540449e92b4886f43607fa02` (record `actions/cache@v4`'s SHA from a lab-c run).
- **Keep least privilege.** `deploy.yml` holds `contents: write` + `packages: write`, but only on
  `push` / `workflow_dispatch`, which a fork PR cannot trigger; acceptable, noted for completeness.
- **Right trigger for the job.** Use `pull_request` unless base-context privilege is genuinely
  required; never check out or run PR code in a `pull_request_target` job.

## 6. Residual risk and conditions not tested

- **Task C is not yet substantiated.** `lab-c` failed at 4s; it also uses `pull_request_target`
  + checkout of the PR head, so it was almost certainly stopped by the same checkout guard before
  any cache write — meaning cache isolation was not actually exercised. C must either opt in
  (`allow-unsafe-pr-checkout: true`) or write the cache without checking out PR code, then add a
  positive control (a restore that should succeed) to prove the method before judging isolation.
- **Not attempted:** bypassing the checkout guard; reading/printing/exfiltrating the canary value
  (reachability was tested by presence only).
- **Not tested:** self-hosted runners, other third-party actions, higher token permissions, other
  secret-handling patterns, non-default platform policies.
- **Platform trajectory:** the `lab-b` run also carried a policy notice — from **2 December 2026**
  GitHub will restrict `pull_request_target` on public repositories by default (evaluate mode now).
  This is an additional platform-side hardening layer beyond the checkout guard.
- **Scope of conclusions:** results hold only for the tested repositories, workflow configurations,
  action versions, and runner environment at test time. They support "protection effective / author
  configuration risk / anomaly to confirm," and must not be generalised to "all `pull_request_target`
  workflows are safe."
