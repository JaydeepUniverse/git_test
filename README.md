Below is a **generic, industry-standard branching + release + deployment** model that works well for **many application teams** on **GitHub + GitHub Actions** with **dev / preprod / prod**, with **tag-based promotion**, strong **audit/back-trace**, and **ServiceNow change gating**.

---

## 1) Branching strategy (keep it boring on purpose)

### Base branch

* **`main`** = *the* base branch (default branch) and the **source of truth**.
* `main` is always **deployable** (no broken builds allowed).

### Working branches (short-lived)

* `feature/<ticket>-<short-desc>` for new work
* `bugfix/<ticket>-<short-desc>` for defects found before release
* `hotfix/<ticket>-<short-desc>` for urgent prod fixes (branch point explained later)

### Why this is the “standard” pattern

* It’s basically “GitHub Flow + release tags”: fewer long-lived branches, easier history, less merge-hell, and works cleanly with GitHub environments + protections.

---

## 2) PR + branch governance (Rulesets / protections)

Use **GitHub Rulesets** (preferred) or classic branch protection. Rulesets can target **branches and tags** and enforce merge policy + status checks. ([GitHub Docs][1])

### Protect `main` (must-have)

Configure on `main` (and optionally on `release/*` if you add them later):

* **Require PRs** (no direct pushes)
* **Minimum approvals** (typically 1–2; security-critical repos often 2)
* **CODEOWNERS required review** for sensitive paths (infra, security, pipelines)
* **Require conversations resolved** (no “hanging” review comments) ([GitHub Docs][2])
* **Require status checks** (strict mode recommended so PR branch must be up-to-date) ([GitHub Docs][2])
* **Block force pushes** (protect history integrity) ([GitHub Docs][2])
* **Restrict merge method**: pick one and standardize org-wide

  * Common: **Squash merge** for clean history, or **Rebase** for linear history. ([GitHub Docs][2])

> Practical note: GitHub warns that **required status checks depend on clear/unique job names** across workflows—otherwise you can accidentally block merges. ([GitHub Docs][3])

### Protect release tags (must-have for “tag-based promotion”)

Apply a ruleset to tags like:

* `v*` (production releases)
* `v*-rc.*` (preprod candidates)

Recommended rules:

* only CI/bot/service-account can create/move tags
* block force-push / updates to tags (immutability)
* optionally require signed tags/commits (if your compliance model needs it)

Rulesets explicitly support required status checks + force-push blocking on **tags** too. ([GitHub Docs][2])

---

## 3) Standard PR checks (generic across all apps)

Make these checks **reusable** via a central workflow template repo (platform team), but the checks themselves are pretty universal:

### Quality gates (baseline)

* Build (compile/package)
* Unit tests + coverage threshold
* Lint/format
* SAST (e.g., CodeQL) + dependency vulnerability scanning
* Secret scanning / policy checks (where applicable)
* IaC checks (terraform fmt/validate + security) if repo includes infra

### Delivery gates (baseline)

* Versioning rules (no duplicate version tags, enforce SemVer)
* Release note / changelog fragment required (optional but great for traceability)
* “No breaking change without label” policy (optional)

---

## 4) Environments in GitHub (dev / preprod / prod)

Create three GitHub **Environments**: `dev`, `preprod`, `prod`.

GitHub Environments give you:

* per-environment secrets
* deployment history in GitHub UI
* “gates” via **deployment protection rules**
* restriction to **selected branches/tags**
* approvals / timers / custom third-party gates ([GitHub Docs][4])

### Environment protection rules (recommended)

**dev**

* Allow deploys from: `main`
* No manual approvals (fast feedback)
* Concurrency group: `dev` (prevents overlapping deploy collisions) ([GitHub Docs][5])

**preprod**

* Allow deploys from tags: `v*-rc.*`
* Optional required reviewers: QA / Release team
* Optional wait timer (useful for “cooldown” or batching; wait time doesn’t count as billable runtime) ([GitHub Docs][4])
* Concurrency group: `preprod` ([GitHub Docs][5])

**prod**

* Allow deploys from tags: `v*.*.*` (no `-rc`)
* Required reviewers: Release manager / Ops / CAB representative (if you want a human gate)
* Additionally: ServiceNow gate (details below)
* Concurrency group: `prod` ([GitHub Docs][5])

---

## 5) Release & deployment process (tag-based promotion, “build once deploy many”)

The core idea: **the artifact is built from `main`**, then **promoted by tags** to higher environments. This gives clean back-trace and avoids “it worked in dev but not in prod” rebuild drift.

### Step A — Merge to `main` → auto deploy to **dev**

1. Dev opens PR from `feature/*` → `main`
2. PR checks must pass + approvals
3. Merge to `main`
4. Workflow on `push` to `main`:

   * build + test
   * publish artifact (container image / package) with immutable identifier (image digest)
   * deploy automatically to `dev` environment

GitHub explicitly supports deployment workflows triggered by `push/pull_request`, and environments for gating and secrets. ([GitHub Docs][5])

### Step B — Create **preprod candidate** tag → deploy to **preprod**

1. Release manager (or automation) creates a tag on the chosen `main` commit:

   * `v1.8.0-rc.1`
2. Workflow on `push` tags `v*-rc.*`:

   * **do not rebuild** (or rebuild only as a verification step)
   * fetch the already published artifact for that commit (by digest)
   * deploy to `preprod`

GitHub environments can restrict which **tags** can deploy. ([GitHub Docs][4])

### Step C — Create **release** tag → deploy to **prod** (gated)

1. After validation in preprod, create final release tag:

   * `v1.8.0`
2. Workflow on `push` tags `v*.*.*`:

   * promote the *same artifact digest*
   * deployment job targets `prod` environment, which applies approvals/gates before secrets are accessible and before execution proceeds ([GitHub Docs][4])

---

## 6) Back-tracing / audit / “where did this come from?”

This model gives very strong traceability because each deployment ties together:

* **environment deployment record** in GitHub (who/what/when) ([GitHub Docs][6])
* release **tag** → exact commit SHA
* workflow run → logs + provenance
* artifact digest (immutable)
* related PRs (via merge commit + references)

Extra “grown-up” practices:

* Generate release notes automatically from merged PRs (label-driven)
* Store SBOM + provenance attestations with the release artifact
* Require “no force push” and immutable tags (rulesets) ([GitHub Docs][2])

---

## 7) Automation patterns (what triggers what)

### PR workflow (CI only)

Triggers: `pull_request` to `main`

* build/test/lint/security scans
* optional: ephemeral preview environments (recommended for web apps)

### Main workflow (CD to dev)

Triggers: `push` to `main`

* build + publish artifact
* deploy to `dev`

### Preprod workflow (tag promotion)

Triggers: `push` tags `v*-rc.*`

* deploy to `preprod`

### Prod workflow (tag promotion + gates)

Triggers: `push` tags `v*.*.*`

* gated deploy to `prod`

---

## 8) ServiceNow change integration for prod (2 practical options)

### Option 1 — Use GitHub Environment “custom deployment protection rules” (best gate architecture)

GitHub supports **custom deployment protection rules** (powered by GitHub Apps) that can gate deployments using third-party systems like **ServiceNow**. ([GitHub Docs][4])
This is clean because the workflow is “paused” at the environment gate level.

A community explanation of how the **deployment_protection_rule** webhook interacts with ServiceNow’s deployment-gate behavior is described here (useful for implementation reasoning). ([servicenow.com][7])

### Option 2 — Use ServiceNow’s GitHub Action to create/change + pause/resume the workflow (easiest to implement)

ServiceNow provides a GitHub Action: **“ServiceNow DevOps Change Automation”** that:

* creates/updates a change in ServiceNow
* **pauses/resumes** the workflow based on approval status ([GitHub][8])

This is very direct, but be aware: if you literally “wait” in a runner for approval, it can have cost/runner-occupancy implications (people have raised this). ([servicenow.com][9])
If you need long CAB lead times, prefer the **Environment gate** approach.

---

## 9) Hotfix process (when prod is burning)

1. Identify last prod release tag: `v1.8.0`
2. Create hotfix branch from that tag:

   * `hotfix/INC1234-critical-fix`
3. PR back into `main` (same checks)
4. Tag a patch release on `main` after merge:

   * `v1.8.1-rc.1` → preprod
   * `v1.8.1` → prod (gated via ServiceNow)

This preserves history and keeps `main` as the single truth.

---

## 10) Make it reusable for *all* application teams (platform approach)

Standardize these as “golden paths”:

* Org-wide **rulesets** for `main` and tag patterns (v*, v*-rc.*) ([GitHub Docs][1])
* Central repo of **reusable workflows** (build/test/deploy templates)
* Standard environment names and tag patterns across repos:

  * `dev`, `preprod`, `prod`
  * `vX.Y.Z-rc.N` (preprod)
  * `vX.Y.Z` (prod)

This gives you consistent dashboards, consistent audit, and minimal cognitive load across teams.

---

If you implement only one “big idea” from all of this: **promote the same artifact by tag**, and let **GitHub Environments** enforce who/what can deploy to each environment. That’s where most real-world CI/CD programs quietly become sane.

[1]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets?utm_source=chatgpt.com "About rulesets"
[2]: https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets "Available rules for rulesets - GitHub Docs"
[3]: https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches?utm_source=chatgpt.com "About protected branches"
[4]: https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments "Deployments and environments - GitHub Docs"
[5]: https://docs.github.com/actions/deployment/about-deployments/deploying-with-github-actions "Deploying with GitHub Actions - GitHub Docs"
[6]: https://docs.github.com/en/actions/concepts/workflows-and-actions/deployment-environments?utm_source=chatgpt.com "Deployment environments"
[7]: https://www.servicenow.com/community/devops-forum/devops-change-velocity-github-deployment-gates-for-servicenow/td-p/3160270?utm_source=chatgpt.com "DevOps Change Velocity - GitHub Deployment Gates f..."
[8]: https://github.com/marketplace/actions/servicenow-devops-change-automation?utm_source=chatgpt.com "ServiceNow DevOps Change Automation · Actions"
[9]: https://www.servicenow.com/community/devops-forum/servicenow-devops-change-github-action-is-billed-for-entire/m-p/2624493?utm_source=chatgpt.com "ServiceNow DevOps Change GitHub Action is billed f..."



<mxfile host="app.diagrams.net" modified="2025-12-03T00:00:00.000Z" agent="ChatGPT" version="22.1.0">
  <diagram id="branching-release-process" name="Branching + Release + ServiceNow Gate">
    <mxGraphModel dx="1200" dy="800" grid="1" gridSize="10" guides="1" tooltips="1" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="1400" pageHeight="900" math="0" shadow="0">
      <root>
        <mxCell id="0"/>
        <mxCell id="1" parent="0"/>

        <!-- Title -->
        <mxCell id="t1" value="GitHub Branching Strategy + Tag-based Promotion (dev / preprod / prod) + ServiceNow Gate" style="text;html=1;strokeColor=none;fillColor=none;align=left;verticalAlign=top;fontSize=18;fontStyle=1" vertex="1" parent="1">
          <mxGeometry x="40" y="20" width="1200" height="30" as="geometry"/>
        </mxCell>

        <!-- Swimlane headers -->
        <mxCell id="h_dev" value="Developer" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F5F5F5;strokeColor=#D0D0D0;align=center;fontStyle=1" vertex="1" parent="1">
          <mxGeometry x="40" y="70" width="320" height="50" as="geometry"/>
        </mxCell>
        <mxCell id="h_pr" value="PR Governance (Rulesets / Checks)" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F5F5F5;strokeColor=#D0D0D0;align=center;fontStyle=1" vertex="1" parent="1">
          <mxGeometry x="380" y="70" width="360" height="50" as="geometry"/>
        </mxCell>
        <mxCell id="h_cd" value="CI/CD + Environments" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F5F5F5;strokeColor=#D0D0D0;align=center;fontStyle=1" vertex="1" parent="1">
          <mxGeometry x="760" y="70" width="600" height="50" as="geometry"/>
        </mxCell>

        <!-- Developer lane nodes -->
        <mxCell id="n1" value="Create branch&#xa;feature/* or bugfix/*" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#E8F0FE;strokeColor=#6C8EBF" vertex="1" parent="1">
          <mxGeometry x="60" y="150" width="280" height="70" as="geometry"/>
        </mxCell>

        <mxCell id="n_hotfix" value="Hotfix path (urgent prod fix)&#xa;Branch hotfix/* from last prod tag vX.Y.Z" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#FFF4E5;strokeColor=#D79B00" vertex="1" parent="1">
          <mxGeometry x="60" y="240" width="280" height="90" as="geometry"/>
        </mxCell>

        <mxCell id="n2" value="Open PR &#x2192; main" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#E8F0FE;strokeColor=#6C8EBF" vertex="1" parent="1">
          <mxGeometry x="60" y="350" width="280" height="60" as="geometry"/>
        </mxCell>

        <!-- PR governance lane nodes -->
        <mxCell id="n3" value="Required PR checks (examples)&#xa;- Build + unit tests + coverage&#xa;- Lint/format&#xa;- SAST/dep scan/secret scan&#xa;- Policy checks (CODEOWNERS paths)" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#EAF9F0;strokeColor=#3C8D5A" vertex="1" parent="1">
          <mxGeometry x="410" y="150" width="300" height="140" as="geometry"/>
        </mxCell>

        <mxCell id="n4" value="Approvals + CODEOWNERS&#xa;No direct pushes to main&#xa;No force-push / history protected" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#EAF9F0;strokeColor=#3C8D5A" vertex="1" parent="1">
          <mxGeometry x="410" y="310" width="300" height="100" as="geometry"/>
        </mxCell>

        <mxCell id="n5" value="Merge PR &#x2192; main" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#EAF9F0;strokeColor=#3C8D5A" vertex="1" parent="1">
          <mxGeometry x="410" y="430" width="300" height="60" as="geometry"/>
        </mxCell>

        <!-- CI/CD + environments lane nodes -->
        <mxCell id="n6" value="On push to main&#xa;Build + Test&#xa;Publish artifact (immutable digest)&#xa;&quot;Build once, deploy many&quot;" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" vertex="1" parent="1">
          <mxGeometry x="790" y="150" width="300" height="120" as="geometry"/>
        </mxCell>

        <mxCell id="n7" value="Auto Deploy &#x2192; DEV&#xa;GitHub Environment: dev&#xa;Allowed from: main" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" vertex="1" parent="1">
          <mxGeometry x="1110" y="150" width="220" height="90" as="geometry"/>
        </mxCell>

        <mxCell id="n8" value="Create RC tag&#xa;vX.Y.Z-rc.N&#xa;(immutable tags via rulesets)" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" vertex="1" parent="1">
          <mxGeometry x="790" y="290" width="300" height="90" as="geometry"/>
        </mxCell>

        <mxCell id="n9" value="Deploy &#x2192; PREPROD&#xa;From tags: v*-rc.*&#xa;Use same artifact digest" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" vertex="1" parent="1">
          <mxGeometry x="1110" y="270" width="220" height="110" as="geometry"/>
        </mxCell>

        <mxCell id="n10" value="Create Release tag&#xa;vX.Y.Z" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" vertex="1" parent="1">
          <mxGeometry x="790" y="410" width="300" height="70" as="geometry"/>
        </mxCell>

        <mxCell id="n11" value="Prod Gate&#xa;ServiceNow Change Request&#xa;- Create/Update CR&#xa;- Await CAB approval" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#FFE6E6;strokeColor=#B85450" vertex="1" parent="1">
          <mxGeometry x="1110" y="400" width="220" height="100" as="geometry"/>
        </mxCell>

        <mxCell id="n12" value="Deploy &#x2192; PROD&#xa;From tags: v*.*.*&#xa;Approved in ServiceNow&#xa;Same artifact digest" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#F3E8FF;strokeColor=#9673A6" vertex="1" parent="1">
          <mxGeometry x="1110" y="520" width="220" height="110" as="geometry"/>
        </mxCell>

        <!-- Optional notes -->
        <mxCell id="note1" value="Traceability: Tag &#x2192; Commit SHA &#x2192; Workflow Run &#x2192; Artifact Digest &#x2192; Environment Deployment History" style="rounded=1;whiteSpace=wrap;html=1;strokeWidth=1;fillColor=#FFFFFF;strokeColor=#D0D0D0;align=left" vertex="1" parent="1">
          <mxGeometry x="40" y="660" width="1290" height="60" as="geometry"/>
        </mxCell>

        <!-- Edges -->
        <mxCell id="e1" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n1" target="n2">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e_hotfix_to_pr" value="or" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n_hotfix" target="n2">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e2" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n2" target="n3">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e3" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n3" target="n4">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e4" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n4" target="n5">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e5" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n5" target="n6">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e6" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n6" target="n7">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e7" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n7" target="n8">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e8" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n8" target="n9">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e9" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n9" target="n10">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e10" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n10" target="n11">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <mxCell id="e11" value="Approved" style="endArrow=block;html=1;rounded=1;strokeWidth=2" edge="1" parent="1" source="n11" target="n12">
          <mxGeometry relative="1" as="geometry"/>
        </mxCell>

        <!-- Loop for "waiting approval" -->
        <mxCell id="e12" value="Pending" style="endArrow=block;html=1;rounded=1;strokeWidth=2;dashed=1" edge="1" parent="1" source="n11" target="n11">
          <mxGeometry relative="1" as="geometry">
            <mxPoint x="1350" y="450" as="targetPoint"/>
            <mxPoint x="1350" y="410" as="sourcePoint"/>
          </mxGeometry>
        </mxCell>

      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
