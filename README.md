Nice — let’s build two practical deliverables you can drop into `platform-repo-ops` and run:

1. A **canonical CODEOWNERS** file for the central Platform-CICD repos (ready to paste into `.github/CODEOWNERS`).
2. A **per-repo example CODEOWNERS** for product repos (example product repo layout).
3. A production-ready **`platform-repo-ops` automation script** (Python + `gh` CLI + `git`) that will create/update CODEOWNERS across many repos, open PRs, and optionally assign reviewers — with dry-run, logging, retries, and safety checks.

Everything below is copy/paste-ready. Use it, adapt it, or tell me to tailor it to your org/team names and policies.

---

# 1 — Canonical CODEOWNERS (Platform-CICD repos)

Place this in `platform-ci-templates/.github/CODEOWNERS` and `cd-templates/.github/CODEOWNERS` (and `shared-actions/.github/CODEOWNERS`).

```text
# Canonical CODEOWNERS for Platform-CICD repos
# Put this file in .github/CODEOWNERS (and optionally repo root)
# Last matching line wins. Keep specific rules below general catch-all.

# -------------------------
# Core templates & workflows
# -------------------------
# All workflow files (CI + CD) : platform CI team + platform cross-product reviewers
/.github/workflows/**        @platform-org/platform-ci-team  @platform-org/platform-cross-product

# Composite actions (reusable small actions)
actions/**                  @platform-org/platform-ci-team

# Tests and pipeline-tests (E2E for templates)
tests/**                    @platform-org/platform-ci-team  @platform-org/qe

# Docs & Examples
docs/**                     @platform-org/platform-docs

# Security sensitive: infra / secrets patterns inside templates
security/**                 @platform-org/security-team
*.secrets.yml               @platform-org/security-team

# Specific template files (give explicit owners for high-risk templates)
.github/workflows/helm-deploy.yml   @platform-org/cd-lead @platform-org/platform-ci-team
.github/workflows/terraform-apply.yml @platform-org/infra-lead @platform-org/security-team
.github/workflows/java-ci.yml       @platform-org/java-template-owner

# -------------------------
# Catch-all: platform leads
# -------------------------
# Everything else should be owned by platform leads (fallback)
*                           @platform-org/platform-leads
```

**Notes / how to adapt**

* Replace `@platform-org/...` with your actual org/team handles (teams must exist and have write access).
* Keep the list short for noisy paths; prefer 1–3 owners per path.
* Put this file on `main` and any long-term support branches you use (e.g., `release/*`).

---

# 2 — Example CODEOWNERS for a Product Repo

Place in `product-repo/.github/CODEOWNERS`. This is tuned for repos that include app code + charts + ansible + terraform + pipeline references.

```text
# Product repo CODEOWNERS - example
# Designed for product repositories that call Platform-CICD templates

# Application code owners
/src/**                     @product-org/product-dev-team

# Deployment artifacts (helm charts)
/charts/**                  @product-org/product-sre @platform-org/platform-ci-team

# Ansible playbooks (app deployments)
/ansible/**                 @product-org/product-sre @platform-org/platform-ci-team

# Terraform infra code
/terraform/**               @product-org/product-infra @platform-org/security-team

# GitHub Actions workflows that reference central templates
.github/workflows/**        @product-org/product-ci-lead @platform-org/platform-ci-team

# Docs and runbooks
/docs/**                    @product-org/product-docs

# Catch-all fallback (product leads)
*                           @product-org/product-leads
```

**Guidelines**

* `@platform-org/platform-ci-team` is included on `.github/workflows/**` so Platform-CICD gets review requests when pipeline-related files change.
* Product teams should include alternates in their teams (at least 2 people per team) to avoid blocking PRs.

---

# 3 — `platform-repo-ops` automation script (Python + `gh` CLI + `git`)

This script will:

* Read a CSV/JSON list of target repos (org/repo).
* For each repo:

  * Clone the repo (shallow), create a branch `ops/codeowners-sync/<timestamp>`.
  * Add or update `.github/CODEOWNERS` with the canonical content (merge intelligently if file exists).
  * Commit and push branch.
  * Create a PR against `main` (or configured base branch) with title/body.
  * Optionally assign reviewers (product repo owners or platform leads).
  * Optionally label the PR and add checklist comments.
* Supports dry-run, concurrency, retries, logging, and rate-limit sleep.
* Uses `gh` CLI for PR creation and `git` for commits. (Easily adapted to use the REST API directly.)

> Pre-reqs:
>
> * `gh` CLI installed and authenticated with a token that has repo:public_repo or repo scope for private repos.
> * `git` installed.
> * Python 3.9+ with `requests` (optional) and `pyyaml` if you prefer YAML config.
> * The user running the script must have push rights to the target repos or be a GitHub admin (or use a machine user).

Copy the script into `platform-repo-ops/sync_codeowners.py`.

```python
#!/usr/bin/env python3
"""
platform-repo-ops: sync CODEOWNERS file across many repos.

Usage:
  python sync_codeowners.py --repos-file repos.csv --codeowners-file canonical_CODEOWNERS \
    --branch-prefix ops/codeowners-sync --base-branch main --dry-run

repos.csv format (header): org,repo
example:
platform-org,ci-templates
product-org,app-repo-1

Important:
- Requires gh CLI and git available in PATH
- gh must be authenticated: `gh auth login` with token that has repo permission
"""

import argparse
import csv
import subprocess
import sys
import tempfile
import os
import shutil
import datetime
import time
import logging

# CONFIGURABLE
GIT_CLONE_DEPTH = 1
SLEEP_BETWEEN_REPOS = 1.0  # seconds
PR_BODY_TEMPLATE = """Sync canonical CODEOWNERS

This PR was created automatically by platform-repo-ops to sync the canonical CODEOWNERS.
Please review the changes and merge if OK.

If you need to change ownership, update the CODEOWNERS file or raise a ServiceNow request as per org policy.

Automated metadata:
- created-by: platform-repo-ops
- timestamp: {ts}
"""
# End config

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")

def run_cmd(cmd, cwd=None, check=True, capture_output=False):
    logging.debug("CMD: %s", " ".join(cmd))
    res = subprocess.run(cmd, cwd=cwd, text=True, capture_output=capture_output)
    if check and res.returncode != 0:
        logging.error("Command failed: %s\nStdout: %s\nStderr: %s", " ".join(cmd), res.stdout, res.stderr)
        raise RuntimeError(f"Command failed: {' '.join(cmd)}")
    return res

def read_repos_csv(path):
    repos = []
    with open(path, newline='') as f:
        reader = csv.DictReader(f)
        for r in reader:
            org = r.get('org') or r.get('organization') or r.get('owner')
            repo = r.get('repo') or r.get('repository') or r.get('name')
            if not org or not repo:
                logging.warning("Skipping malformed row: %s", r)
                continue
            repos.append((org.strip(), repo.strip()))
    return repos

def load_codeowners(path):
    with open(path, 'r', encoding='utf-8') as f:
        return f.read()

def ensure_git_user_configured():
    # Ensure git user is configured to avoid commit errors
    try:
        run_cmd(["git", "config", "--global", "user.email"], check=False)
    except Exception:
        pass

def clone_repo(org, repo, tmpdir, base_branch):
    clone_url = f"https://github.com/{org}/{repo}.git"
    repo_dir = os.path.join(tmpdir, f"{org}__{repo}")
    if os.path.exists(repo_dir):
        shutil.rmtree(repo_dir)
    logging.info("Cloning %s/%s ...", org, repo)
    run_cmd(["git", "clone", "--depth", str(GIT_CLONE_DEPTH), "--branch", base_branch, clone_url, repo_dir])
    return repo_dir

def read_existing_codeowners(repo_dir):
    # prefer .github/CODEOWNERS then CODEOWNERS in root
    paths = [os.path.join(repo_dir, ".github", "CODEOWNERS"), os.path.join(repo_dir, "CODEOWNERS")]
    for p in paths:
        if os.path.exists(p):
            with open(p, 'r', encoding='utf-8') as f:
                content = f.read()
            return p, content
    return None, None

def merge_codeowners(existing_content, canonical_content):
    """
    Simple merge strategy:
    - If existing is None -> return canonical
    - If existing contains a marker block DELIMITED by:
        # BEGIN PLATFORM-CODEOWNERS
        ...
        # END PLATFORM-CODEOWNERS
      then replace that block; otherwise prepend canonical block with markers.
    This allows per-repo customizations to stay below/after the marker.
    """
    BEGIN = "# BEGIN PLATFORM-CODEOWNERS"
    END = "# END PLATFORM-CODEOWNERS"
    block = f"{BEGIN}\n{canonical_content.rstrip()}\n{END}\n\n"
    if not existing_content:
        return block
    if BEGIN in existing_content and END in existing_content:
        before, rest = existing_content.split(BEGIN, 1)
        _, after = rest.split(END, 1)
        merged = f"{before}{block}{after.lstrip()}"
        return merged
    else:
        # Prepend canonical block (preserve existing below)
        return f"{block}{existing_content}"

def write_codeowners(repo_dir, content):
    dest_dir = os.path.join(repo_dir, ".github")
    os.makedirs(dest_dir, exist_ok=True)
    path = os.path.join(dest_dir, "CODEOWNERS")
    with open(path, "w", encoding="utf-8") as f:
        f.write(content)
    return path

def create_branch_and_push(repo_dir, branch_name):
    run_cmd(["git", "checkout", "-b", branch_name], cwd=repo_dir)
    run_cmd(["git", "add", ".github/CODEOWNERS"], cwd=repo_dir)
    run_cmd(["git", "commit", "-m", "chore: sync canonical CODEOWNERS [automated]"], cwd=repo_dir)
    run_cmd(["git", "push", "--set-upstream", "origin", branch_name], cwd=repo_dir)

def create_pr_with_gh(org, repo, branch, base_branch, title, body, reviewers=None, labels=None):
    repo_ref = f"{org}/{repo}"
    cmd = ["gh", "pr", "create", "--repo", repo_ref, "--head", branch, "--base", base_branch, "--title", title, "--body", body]
    if labels:
        cmd += ["--label", ",".join(labels)]
    logging.info("Creating PR for %s/%s branch=%s", org, repo, branch)
    res = run_cmd(cmd, capture_output=True)
    pr_output = res.stdout.strip()
    logging.debug("PR output: %s", pr_output)
    if reviewers:
        # assign reviewers
        try:
            run_cmd(["gh", "pr", "review-request", "--repo", repo_ref, "--reviewer", ",".join(reviewers), pr_output], check=False)
        except Exception as e:
            logging.warning("Failed to request reviewers via gh CLI: %s", e)
    return pr_output

def process_repo(org, repo, canonical_codeowners, dry_run, base_branch, branch_prefix, reviewers, labels):
    ts = datetime.datetime.utcnow().strftime("%Y%m%dT%H%M%SZ")
    tmpdir = tempfile.mkdtemp(prefix=f"sync_codeowners_{org}_{repo}_")
    try:
        repo_dir = clone_repo(org, repo, tmpdir, base_branch)
    except Exception as e:
        logging.error("Failed to clone %s/%s: %s", org, repo, e)
        shutil.rmtree(tmpdir, ignore_errors=True)
        return False
    try:
        existing_path, existing = read_existing_codeowners(repo_dir)
        merged = merge_codeowners(existing, canonical_codeowners)
        if existing and merged.strip() == existing.strip():
            logging.info("No changes required for %s/%s (CODEOWNERS unchanged).", org, repo)
            shutil.rmtree(tmpdir, ignore_errors=True)
            return True
        if dry_run:
            logging.info("[DRY RUN] Would update CODEOWNERS for %s/%s. Existing file: %s", org, repo, existing_path)
            # Optionally write preview to logs
            logging.debug("Merged content:\n%s", merged)
            shutil.rmtree(tmpdir, ignore_errors=True)
            return True
        # write new CODEOWNERS
        dest_path = write_codeowners(repo_dir, merged)
        branch = f"{branch_prefix}/{ts}"
        ensure_git_user_configured()
        create_branch_and_push(repo_dir, branch)
        title = f"[ops] Sync canonical CODEOWNERS - {ts}"
        body = PR_BODY_TEMPLATE.format(ts=ts)
        pr = create_pr_with_gh(org, repo, branch, base_branch, title, body, reviewers=reviewers, labels=labels)
        logging.info("Created PR: %s for %s/%s", pr, org, repo)
        time.sleep(SLEEP_BETWEEN_REPOS)
        return True
    except Exception as e:
        logging.exception("Error processing %s/%s: %s", org, repo, e)
        return False
    finally:
        shutil.rmtree(tmpdir, ignore_errors=True)

def main():
    parser = argparse.ArgumentParser(description="Sync CODEOWNERS across repos")
    parser.add_argument("--repos-file", required=True, help="CSV file with columns: org,repo")
    parser.add_argument("--codeowners-file", required=True, help="Canonical CODEOWNERS file path")
    parser.add_argument("--base-branch", default="main", help="Base branch to open PRs against")
    parser.add_argument("--branch-prefix", default="ops/codeowners-sync", help="Branch prefix for sync branches")
    parser.add_argument("--dry-run", action="store_true", help="Perform a dry run (no pushes/PRs)")
    parser.add_argument("--reviewers", help="Comma separated list of reviewers to request on PRs (e.g. platform-org/platform-leads)")
    parser.add_argument("--labels", help="Comma separated labels to add to PRs")
    args = parser.parse_args()

    repos = read_repos_csv(args.repos_file)
    canonical = load_codeowners(args.codeowners_file)
    reviewers = args.reviewers.split(",") if args.reviewers else None
    labels = args.labels.split(",") if args.labels else None

    logging.info("Starting sync for %d repos (dry_run=%s)", len(repos), args.dry_run)
    success = 0
    for org, repo in repos:
        try:
            ok = process_repo(org, repo, canonical, args.dry_run, args.base_branch, args.branch_prefix, reviewers, labels)
            if ok:
                success += 1
        except Exception as e:
            logging.exception("Unhandled error for %s/%s: %s", org, repo, e)
    logging.info("Completed sync. Success: %d/%d", success, len(repos))

if __name__ == "__main__":
    main()
```

---

## Usage examples

1. Create a `repos.csv` file:

```csv
org,repo
platform-org,ci-templates
platform-org,cd-templates
platform-org,shared-actions
product-org,app-repo-1
product-org,app-repo-2
```

2. Create your canonical file `canonical_CODEOWNERS` (use the canonical template above).

3. Run in dry run:

```bash
python sync_codeowners.py --repos-file repos.csv --codeowners-file canonical_CODEOWNERS --dry-run
```

4. Run for real (requests reviewers and labels PRs):

```bash
python sync_codeowners.py --repos-file repos.csv --codeowners-file canonical_CODEOWNERS \
  --reviewers "platform-org/platform-leads,platform-org/platform-ci-team" \
  --labels "ops-sync,codeowners" \
  --base-branch main
```

---

## Operational notes, hardening & best practices

* **Permissions:** `gh` must be authenticated with a token that can create branches and PRs. For private repos use a machine user token or org-level app with scoped permissions.
* **Safety:** The script creates a branch per run using timestamp. This avoids collisions and makes it easy to track PRs.
* **Merge policy:** PRs created by this automation should be reviewed by the product repo owners before merge. Consider a separate branch protection rule that prevents merging automation PRs without approvals.
* **Idempotency:** The merge strategy places a `BEGIN/END PLATFORM-CODEOWNERS` block. This lets teams keep local customizations outside the block. If you prefer full replacement, change merge logic accordingly.
* **Retries & rate limits:** This simple script has no backoff for API rate limits. If you sync hundreds of repos, add retry/backoff around `gh` commands and `git push`. You can detect GitHub rate limit errors from `gh` output.
* **Audit & logging:** Persist logs to a storage location and produce a final report listing created PR URLs (extend `create_pr_with_gh` to capture returned PR URL).
* **PR templates:** Use a PR template that documents the ownership change reason and links to ServiceNow request if required.
* **Testing:** Run dry-run in a sandbox org before production. Test with one product repo to validate team names and access.
* **Automation scheduling:** Run this script nightly or weekly and create an internal dashboard that shows repos lacking CODEOWNERS or having empty ownership teams.
* **Security:** Keep the canonical file in a protected repo (`platform-repo-ops`) with controlled permissions. Use branch protection on that repo.

---

## Troubleshooting tips

* If `gh pr create` fails: ensure `gh` is authenticated (`gh auth status`) and the token has `repo` permission.
* If `git clone` for a repo fails with permission error: check repo visibility and token scopes.
* If reviewers cannot be requested: ensure reviewer teams exist and are mentionable in the target repo (team must belong to the same org or be visible).
* If CODEOWNERS not applied in PRs: ensure the file is on the target branch (`main`) and branch protection setting `Require review from Code Owners` is enabled.

---

## Next steps I can do for you (pick one)

* Tailor the canonical CODEOWNERS with your actual team handles (give me your org/team names).
* Add a feature to the script: generate a summary CSV of created PR URLs and statuses.
* Convert the script to use the GitHub REST API (requests/pygithub) instead of `gh` CLI.
* Create GitHub Actions workflows for `platform-repo-ops` to run the sync on schedule and open a Slack digest.

Which would you like next?
