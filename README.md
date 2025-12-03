Here’s a **practical, “can-be-implemented-tomorrow” RACI** for **GitHub + GitHub Actions** with **dev / preprod / prod**, **tag-based promotion**, and **ServiceNow change control**.

**Legend**: **R** = Responsible (does the work) · **A** = Accountable (owns outcome/decision) · **C** = Consulted (gives input) · **I** = Informed

### Standard roles (use these across all teams)

| Role | Meaning (in plain English)                                            |
| ---- | --------------------------------------------------------------------- |
| PO   | Product Owner / Business Owner                                        |
| EM   | Engineering Manager (delivery + people + process ownership)           |
| ARCH | Solution/Tech Architect (standards + architecture fitness)            |
| TL   | Tech Lead / Service Owner (codebase & runtime ownership)              |
| DEV  | Developer (implements changes)                                        |
| QA   | QA / SDET (quality validation, automation)                            |
| PLAT | Platform/DevOps team (CI/CD platform, runners, templates, guardrails) |
| SEC  | AppSec/InfoSec (security controls, risk)                              |
| REL  | Release Manager (release planning, coordination)                      |
| CAB  | Change Manager / CAB approvers (change governance)                    |
| OPS  | SRE/Operations (prod reliability, incident response, runbooks)        |

---

## 1) Governance & setup (org-wide standards + repo bootstrap)

| Activity                                                                  | PO | EM    | ARCH  | TL    | DEV | QA | PLAT    | SEC | REL   | CAB     | OPS   |
| ------------------------------------------------------------------------- | -- | ----- | ----- | ----- | --- | -- | ------- | --- | ----- | ------- | ----- |
| Define org branching strategy + tag conventions                           | C  | **A** | **R** | C     | I   | I  | C       | C   | C     | I       | C     |
| Create standard PR policy (approvals, required checks, merge method)      | I  | **A** | C     | C     | I   | C  | **R**   | C   | I     | I       | C     |
| Create org rulesets (protect `main`, protect `v*`/`rc` tags)              | I  | **A** | C     | C     | I   | I  | **R**   | C   | I     | I       | C     |
| Define environment model (`dev/preprod/prod`) + deploy-from rules         | I  | **A** | **R** | C     | I   | C  | C       | C   | C     | I       | C     |
| Create reusable workflow templates (CI/CD “golden paths”)                 | I  | C     | C     | C     | I   | C  | **A/R** | C   | I     | I       | C     |
| Runner strategy (hosted vs self-hosted), hardening, scaling               | I  | C     | C     | I     | I   | I  | **A/R** | C   | I     | I       | C     |
| Secrets strategy (ownership, rotation, environment segregation)           | I  | C     | C     | **A** | I   | I  | R       | C   | I     | I       | **R** |
| ServiceNow change process mapping (what requires CR, lead time, evidence) | C  | **A** | C     | C     | I   | I  | C       | C   | **R** | **A/R** | C     |
| Define release policy (cadence, freeze windows, emergency path)           | C  | **A** | C     | C     | I   | C  | C       | C   | **R** | C       | C     |

---

## 2) Day-to-day development (branch → PR → merge → dev deploy)

| Activity                                                               | PO | EM | ARCH | TL      | DEV     | QA    | PLAT  | SEC   | REL | CAB | OPS |
| ---------------------------------------------------------------------- | -- | -- | ---- | ------- | ------- | ----- | ----- | ----- | --- | --- | --- |
| Create feature/bugfix branch (`feature/*`, `bugfix/*`)                 | I  | I  | I    | C       | **A/R** | I     | I     | I     | I   | I   | I   |
| Commit changes + unit tests updated                                    | I  | I  | I    | C       | **R**   | C     | I     | I     | I   | I   | I   |
| Open PR to `main`                                                      | I  | I  | I    | C       | **R**   | I     | I     | I     | I   | I   | I   |
| PR review (code/quality)                                               | I  | I  | C    | **A/R** | R       | C     | I     | C     | I   | I   | I   |
| Security review for sensitive changes (auth, crypto, infra, pipelines) | I  | I  | C    | **A**   | R       | I     | C     | **R** | I   | I   | C   |
| Resolve PR comments + update PR                                        | I  | I  | I    | C       | **R**   | C     | I     | C     | I   | I   | I   |
| Approve PR (CODEOWNERS + required approvals)                           | I  | I  | C    | **A/R** | I       | C     | I     | C     | I   | I   | I   |
| Merge PR to `main` (only after checks green)                           | I  | I  | I    | **A/R** | C       | I     | I     | I     | I   | I   | I   |
| Maintain PR checks/workflows in app repo (as code evolves)             | I  | I  | I    | **A**   | **R**   | C     | C     | C     | I   | I   | I   |
| Auto deploy to **dev** on merge (GitHub Actions)                       | I  | I  | I    | **A**   | I       | I     | **R** | I     | I   | I   | C   |
| Verify dev deployment success (basic validation)                       | I  | I  | I    | **A**   | R       | **R** | I     | I     | I   | I   | C   |

> Note: even though the workflow “executes the deploy,” accountability for dev deploy quality usually sits with the **TL** (service owner). Platform is responsible for the delivery mechanism.

---

## 3) Release promotion (RC tag → preprod → release tag → prod with ServiceNow)

| Activity                                                               | PO      | EM | ARCH | TL      | DEV | QA    | PLAT  | SEC   | REL     | CAB     | OPS   |
| ---------------------------------------------------------------------- | ------- | -- | ---- | ------- | --- | ----- | ----- | ----- | ------- | ------- | ----- |
| Plan release scope (what features go in)                               | **A/C** | C  | C    | **R**   | C   | C     | I     | C     | **R**   | I       | C     |
| Cut **RC tag** (`vX.Y.Z-rc.N`) from `main`                             | I       | I  | I    | C       | I   | I     | I     | I     | **A/R** | I       | I     |
| Deploy to **preprod** from RC tag (automated)                          | I       | I  | I    | C       | I   | I     | **R** | I     | **A**   | I       | C     |
| Execute preprod test cycle + sign-off                                  | I       | I  | C    | **A**   | C   | **R** | I     | C     | C       | I       | C     |
| Performance testing (if applicable) in preprod                         | I       | I  | C    | **A**   | C   | **R** | C     | C     | I       | I       | C     |
| DAST / pen-test window (risk-based)                                    | I       | I  | C    | **A**   | C   | C     | C     | **R** | I       | I       | C     |
| Create ServiceNow Change Request for prod release                      | I       | I  | I    | C       | I   | I     | C     | C     | **R**   | **A**   | C     |
| Attach evidence to CR (test results, rollback plan, impacted services) | I       | I  | C    | **A/R** | C   | R     | C     | C     | **R**   | C       | **R** |
| Approve CR / CAB decision                                              | I       | I  | I    | I       | I   | I     | I     | C     | C       | **A/R** | C     |
| Create **release tag** (`vX.Y.Z`) after approval readiness             | I       | I  | I    | C       | I   | I     | I     | I     | **A/R** | I       | I     |
| Deploy to **prod** from release tag (gated by env + ServiceNow)        | I       | I  | I    | C       | I   | I     | **R** | I     | **A**   | C       | **R** |
| Post-prod smoke verification                                           | I       | I  | I    | **A**   | C   | **R** | I     | I     | C       | I       | **R** |
| Release communications (status, notes, stakeholders)                   | **C**   | C  | I    | C       | I   | I     | I     | I     | **A/R** | I       | C     |

---

## 4) Testing strategy (what tests happen where, and who owns them)

| Test type                                    | PR (pre-merge) | DEV             | PREPROD            | PROD                | RACI ownership                       |
| -------------------------------------------- | -------------- | --------------- | ------------------ | ------------------- | ------------------------------------ |
| Lint/format/static checks                    | ✅ Required     | Optional        | Optional           | No                  | **DEV R**, **TL A**, PLAT C          |
| Unit tests + coverage                        | ✅ Required     | ✅               | Optional           | No                  | **DEV R**, **TL A**, QA C            |
| SAST / dependency scan / secret scan         | ✅ Required     | ✅               | ✅ (optional rerun) | No                  | **PLAT R**, **SEC A** (policy), TL C |
| Integration tests (service-to-service)       | Optional       | ✅ (recommended) | ✅ Required         | No                  | **QA R**, **TL A**, DEV C            |
| End-to-end (E2E) tests                       | Optional       | Optional        | ✅ Required         | Optional smoke only | **QA R**, **TL A**                   |
| Performance / soak tests                     | No             | Optional        | ✅ Risk-based       | No                  | **QA R**, **TL A**, OPS C            |
| DAST                                         | No             | Optional        | ✅ Risk-based       | No                  | **SEC R**, **TL A**, QA C            |
| Smoke tests                                  | No             | Optional        | ✅                  | ✅ Required          | **OPS R**, **TL A**, QA C            |
| Observability validation (dashboards/alerts) | No             | ✅ minimal       | ✅ required         | ✅ required          | **OPS R**, **OPS A**, PLAT C         |

---

## 5) Operations, support, continuous improvement

| Activity                                         | PO | EM    | ARCH | TL    | DEV   | QA | PLAT    | SEC   | REL | CAB | OPS   |
| ------------------------------------------------ | -- | ----- | ---- | ----- | ----- | -- | ------- | ----- | --- | --- | ----- |
| Own service reliability SLOs/SLIs                | I  | C     | C    | **A** | I     | I  | C       | I     | I   | I   | **R** |
| On-call & incident response                      | I  | C     | I    | **A** | C     | I  | C       | I     | I   | I   | **R** |
| Runbooks (create + keep updated)                 | I  | I     | C    | **A** | C     | I  | C       | C     | I   | I   | **R** |
| Monitoring/alerts setup & tuning                 | I  | I     | I    | **A** | I     | I  | C       | I     | I   | I   | **R** |
| Release rollback execution (when needed)         | I  | I     | I    | **A** | C     | I  | C       | I     | C   | I   | **R** |
| Patch/security remediation SLAs                  | I  | C     | I    | **A** | **R** | I  | C       | **R** | I   | I   | C     |
| Maintain reusable workflows/templates            | I  | I     | I    | C     | I     | I  | **A/R** | C     | I   | I   | C     |
| Maintain app repo workflows (app-specific)       | I  | I     | I    | **A** | **R** | C  | C       | C     | I   | I   | I     |
| Runner operations (capacity, updates, hardening) | I  | I     | I    | I     | I     | I  | **A/R** | C     | I   | I   | C     |
| Post-incident review (RCA) + action tracking     | I  | **A** | C    | **R** | C     | C  | C       | C     | I   | I   | **R** |

---

### Two policy notes that make this actually work at scale

1. **Only one “A” per row** (you’ll notice TL/REL/OPS/CAB rotate accountability depending on stage). This prevents the classic “everyone owns it so nobody owns it” failure mode.
2. **Platform owns the *paved road*, app teams own the *car***: PLAT provides runners/templates/policies; TL is accountable for what ships; OPS is accountable for production health; REL/CAB govern prod entry.

If you want, I can also output this as:

* a **Confluence-ready RACI table** (same content, better formatting),
* or a **CSV** you can import into Excel/PowerPoint.
