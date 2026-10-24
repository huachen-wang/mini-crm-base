# Mini CRM — GitHub Actions security testbed

A deliberately tiny CRM (Java + Spring Boot backend, one static HTML page) whose
real purpose is to be a **controlled stage** for the INFO5995 Part B
investigation of GitHub Actions security. The application is not the subject of
testing — the files under `.github/workflows/` are.

> ⚠️ The `lab-*` workflows are **intentionally vulnerable test fixtures**. Only
> ever run them on **your own** test repositories, with **canary** values
> (`FAKE_SECRET=canary123`), never real secrets. This is authorised, scoped,
> non-destructive testing of the GitHub Actions platform.

---

## 1. The app (the stage)

| Part | What | Where |
| --- | --- | --- |
| Backend | Spring Boot REST API, in-memory contacts | `src/main/java/com/example/minicrm/` |
| Frontend | single static page, vanilla JS | `src/main/resources/static/index.html` |
| Tests | two smoke tests (give CI something real to run) | `src/test/java/...` |

Run locally:

```bash
mvn spring-boot:run
# open http://localhost:8080
```

Build / test:

```bash
mvn -B test
mvn -B package
```

---

## 2. The workflows (the subject under test)

Two kinds of files. The **baselines** show how a careful team writes CI/CD —
use them as the "secure" reference. The **lab fixtures** are the misconfigured
or attack-surface versions you actually probe.

### Safe baselines (realistic reference)

| File | Trigger | Maps to | What to study |
| --- | --- | --- | --- |
| `ci.yml` | `pull_request`, `push` | **C** baseline | `cache: maven` is the cache write surface; safe `pull_request` trigger |
| `deploy.yml` | `push` main | **D**, **E** | many secrets flowing into `run:`/URLs; boolean-gated `if:`; mutable action tags |
| `bot.yml` | `issue_comment`, `issues` | **B** baseline | attacker-reachable trigger, but read-only perms + no fork-code checkout |

### Lab fixtures (⚠️ intentionally vulnerable)

| File | Trigger | Maps to | Demonstrates |
| --- | --- | --- | --- |
| `lab-a-script-injection.yml` | `issues` | **A** | issue title interpolated into `run:` → command execution (our-config-error / Not Applicable) |
| `lab-b-pwn-request.yml` | `pull_request_target` | **B** | checks out & runs fork PR code while holding base secrets |
| `lab-b-pwn-request-fixed.yml` | `pull_request` | **B** | the recommended fix, for the fix-vs-residual-risk comparison |
| `lab-c-cache-poison.yml` | `pull_request_target` | **C** | read-only job still writes the base repo's shared cache |
| `lab-d-secret-masking.yml` | `workflow_dispatch` | **D** | echoes a canary secret through encodings to find masking bypasses |

### Task E (supply chain) — no dedicated file needed

Every `uses:` across all workflows is pinned to a **mutable tag**
(`actions/checkout@v4`, `actions/setup-java@v4`, `actions/cache@v4`,
`actions/upload-artifact@v4`). List them and argue the pin-to-SHA remediation.
Find them with:

```bash
grep -rn "uses:" .github/workflows/
```

---

## 3. The A–E → focus-area mapping (for the report's scope section)

Two eligible focus areas, one shared trust boundary: **can a fork's untrusted
code cross the fork↔base boundary to reach secrets / token / cache it
shouldn't?**

- **Focus 1 — GITHUB_TOKEN / secrets越界:** tasks **B** (`lab-b-*`) and **D** (`lab-d-*`).
- **Focus 2 — cross-repo isolation:** task **C** (`lab-c-*`).
- **A** (`lab-a-*`) = the Not-Applicable baseline that defines "our config error vs platform defect".
- **E** = remediation / supply-chain review, drawn from the `uses:` tags above.

---

## 4. Setting up base ↔ fork (the controlled stage)

Both repos must be **yours**. Do not fork someone else's project.

1. Create a new repo under your account, e.g. `mini-crm-base`. Push this project
   into it (it is the "victim" / base repo).
2. In `mini-crm-base` → Settings → Secrets, add `FAKE_SECRET = canary123`
   (plus other canaries like `DEPLOY_TOKEN=canary123` if you probe `deploy.yml`).
3. From a **second account**, fork `mini-crm-base` → `mini-crm-fork`.
4. Open fork PRs from `mini-crm-fork` back to `mini-crm-base` to trigger the
   `pull_request_target` fixtures (B, C).
5. For D, run `lab-d-secret-masking.yml` manually from the Actions tab.
6. For A, open an issue in the base repo with the title
   `$(echo injected > proof.txt)`.

---

## 5. Evidence format (per the rubric)

For every finding, record the chain and label each line as **our config** or
**the platform's response**:

```
Assumption → Evidence (I set up X → I did input Y → GitHub returned Z)
→ How verified → Failure/attack condition → Impact → Fix → Residual risk
```

Keep only the minimal workflow snippet + trigger + the relevant log line. A
negative result ("configured per X, platform behaved per its promise") is still
a complete, gradeable finding.

## INFO5995 defensive workflow validation
