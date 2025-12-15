Below is a **security-hardening process for GitHub Actions self-hosted runner infrastructure**, organized **infra-wise** (VM on Azure/AWS vs Kubernetes/AKS/EKS). I’m treating this like an architected control set: **threat model → build/provision → runtime controls → maintenance/patching → monitoring/response**.

GitHub’s core warning is the big one: **self-hosted runners do not provide the same “clean ephemeral VM per job” guarantee as GitHub-hosted runners**, so you must assume a runner can be **persistently compromised** if it runs untrusted code. ([GitHub Docs][1])

---

## 0) Baseline threat model (applies everywhere)

### Primary risks to design against

* **Untrusted code execution** (PRs, forks, third-party actions) → runner compromise → lateral movement into your cloud/VNet/VPC.
* **Credential theft** (GitHub tokens, cloud creds, SSH keys, kubeconfigs).
* **Cross-job contamination** (job A leaves malware/artifacts for job B).
* **Supply chain** (actions you run, containers you pull, dependencies).
* **Privilege escalation** (runner has admin/sudo, hostPath mounts, overly broad IAM/RBAC).

### Non-negotiable GitHub controls

1. **Never use self-hosted runners for public repos / untrusted PRs.** Isolate by trust level using runner groups. ([wiz.io][2])
2. Prefer **ephemeral / just-in-time runners** (one job then destroy). GitHub recommends ephemeral autoscaling; persistent autoscaling is discouraged. ([GitHub Docs][3])
3. **Least privilege for `GITHUB_TOKEN`** (set default permissions to read; elevate only per job).
4. **Pin actions** (commit SHA, not mutable tags) and avoid unreviewed third-party actions.
5. Consider runner hardening instrumentation (network/file/process monitoring) (e.g., “Harden Runner” style tooling) for visibility. ([GitHub][4])

---

## 1) VM-based runners (Azure VM / AWS EC2)

### A. Provisioning & image strategy (golden image + immutable)

**Goal:** treat runner VMs like cattle, not pets.

* Build a **hardened golden image** (Packer) with:

  * runner binaries, required toolchains, baseline security agents
  * locked-down OS config (CIS-like)
  * no long-lived secrets baked in
* Deploy runners via **autoscaling** and **terminate after 1 job** (ephemeral). This is repeatedly recommended as it reduces persistent compromise and job contamination. ([GitHub Docs][3])

### B. Identity & access (cloud IAM)

**Goal:** the runner should have *exactly* the permissions to do CI/CD and nothing else.

* Use **short-lived credentials** (OIDC federation) instead of static keys where possible.
* Scope IAM to:

  * specific resources (RG/Subscription, specific buckets, specific registries)
  * least privilege actions (no `*` permissions)
* In Azure, follow identity best practices (MFA, conditional access, strong separation of duties). ([Microsoft Learn][5])

### C. Network controls (blast-radius shaping)

* Put runners in a **dedicated subnet** with **egress control**:

  * allow outbound only to what’s needed (GitHub endpoints, artifact repos, registries, package mirrors)
* **No inbound** to runners (ideally). Use outbound-only connectivity; if SSH is needed, restrict by source IP + JIT access + MFA/bastion.
* Prevent lateral movement:

  * block access from runner subnet to sensitive private endpoints unless required
  * separate “build” runners from “deploy” runners (different subnets, different IAM)

### D. Host hardening (OS)

* Run runner service as **non-root** where feasible.
* Minimize installed packages (reduce attack surface).
* Enforce:

  * disk encryption
  * host firewall
  * audit logging
  * kernel hardening defaults where supported
* Defender/Vuln mgmt:

  * Use cloud posture/vulnerability recommendations (e.g., Microsoft Defender for Cloud compute recommendations). ([Microsoft Learn][6])

### E. Secrets management (VM)

* **Never store secrets on disk** in the runner (especially SSH keys).
* Prefer:

  * GitHub OIDC → cloud secret store (Azure Key Vault / AWS Secrets Manager) → fetch at runtime
  * short TTL, rotate frequently
* If SSH to targets is required:

  * best: **ephemeral SSH certificates** (CA-signed short-lived certs) instead of static private keys
  * if you must use keys: keep them in a secrets manager and write them to tmpfs for job duration only

### F. Runner software updates & patching

* GitHub runners can auto-update the runner app, but you can disable and manage it yourself (common in containerized/controlled environments). ([GitHub Docs][7])
  **Hardening approach:**
* Prefer **immutable updates**:

  * new golden image → roll out new runners → drain old runners → terminate
* OS patching:

  * schedule maintenance windows + staged rings (dev → preprod → prod)
  * in Azure, align with VM security baseline guidance and patch management processes. ([Microsoft Learn][8])

### G. Logging/monitoring/IR

* Collect:

  * runner service logs
  * OS auth logs
  * process/network telemetry (EDR)
* Monitor runner health and job anomalies (GitHub provides runner monitoring/troubleshooting guidance). ([GitHub Docs][9])
* Incident response stance:

  * **assume compromise = rebuild** (terminate runner, rotate secrets, invalidate tokens, review workflow changes)

---

## 2) Kubernetes-based runners (AKS/EKS/GKE; typical with actions-runner-controller)

### A. Architecture pattern: controller + ephemeral runner pods

* Use a runner controller/operator to scale runner pods.
* Ensure **each job gets a fresh pod** and pods are destroyed after completion (ephemeral). This aligns with GitHub’s guidance on ephemeral runners. ([GitHub Docs][3])

### B. Namespace & isolation model

* Dedicated **namespace per trust zone** (e.g., `runners-internal`, `runners-release`).
* Separate clusters or node pools for high-trust runners (prod deployment runners).
* Use **taints/tolerations** so runners only land on hardened nodes.

### C. Pod security hardening (the big rocks)

Use strong **Pod/Container security contexts**:

* `runAsNonRoot: true`
* drop Linux capabilities
* `readOnlyRootFilesystem: true` (where feasible)
* disallow privileged containers
  Kubernetes security context is the standard control point. ([Kubernetes][10])

Also:

* **No hostPath mounts** unless absolutely required.
* Avoid Docker-in-Docker privileged patterns; prefer:

  * kaniko/buildkit rootless where possible
  * dedicated build service if needed

### D. RBAC (least privilege)

* Runner controller SA (service account) gets minimal permissions to create/manage runner pods and nothing else.
* Runner pods themselves should not have broad API access.
  Kubernetes has explicit RBAC good-practice guidance (design to avoid privilege escalation). ([Kubernetes][11])

### E. Network Policies (contain the runner)

* Default deny ingress/egress for runner namespace.
* Allow egress only to:

  * GitHub
  * container registry
  * artifact repo
  * required cloud APIs
* Block access to Kubernetes API from runner pods unless necessary.

### F. Images: minimal + signed + scanned

* Use **minimal base images** (reduce CVEs + tools attackers love).
* Enforce:

  * pull from trusted registries only
  * image signing/verification
  * vulnerability scanning gates
* Don’t bake secrets into images.

### G. PV/PVC & storage

* Prefer **ephemeral storage** (`emptyDir`) for workspaces.
* If PVC is unavoidable (caching):

  * separate caches per repo/trust level to avoid cross-contamination
  * encrypt at rest
  * restrict access via namespace + RBAC + storage class policies
* Avoid sharing PVCs between unrelated repos.

### H. Node security (AKS/EKS)

* Separate node pools for runners:

  * hardened OS image
  * restricted SSH access (or none)
  * disk encryption
  * auto-upgrades and patching cadence aligned with cluster upgrades
* Use managed cluster security baselines and continuously evaluate posture (CSPM).

### I. Secrets in Kubernetes

* Avoid raw Kubernetes Secrets for high-value credentials unless you have envelope encryption + tight RBAC.
* Prefer external secret stores:

  * **Azure Key Vault / AWS Secrets Manager** mounted via CSI driver
  * short-lived tokens via OIDC where possible

### J. Upgrades & maintenance

* Treat controller + runner images like software supply chain:

  * pin versions
  * validate in lower env
  * rollout with canaries
* Cluster upgrades:

  * runbooks for AKS/EKS version upgrades
  * ensure Pod security and RBAC policies remain enforced post-upgrade

---

## 3) “Infra governance” controls (applies to both VM & K8s)

### A. Runner segmentation (by trust + purpose)

Create explicit tiers:

* **Tier 0 (untrusted)**: ideally *no* self-hosted runners; use GitHub-hosted only.
* **Tier 1 (internal CI)**: build/test only; no prod network access.
* **Tier 2 (deployment)**: restricted to deployment workflows; tighter IAM; separate infra.

This is a direct response to the “self-hosted runner can be persistently compromised” reality. ([GitHub Docs][1])

### B. Policy as code

* Enforce workflow policies:

  * required reviewers for workflow changes
  * restrict who can use which runner groups
  * enforce action pinning, token permissions, forbidden patterns (`pull_request_target` misuse, etc.)
* Infra policies:

  * Terraform with OPA/Conftest checks
  * guardrails for network/IAM/RBAC

### C. Supply chain controls

* Dependency provenance, SBOMs, signed artifacts.
* Artifact repositories with immutability and retention policies.

### D. Observability + detections

* Centralize logs:

  * GitHub audit logs + Actions logs
  * cloud control plane logs
  * cluster audit logs (K8s)
* Detect:

  * unusual outbound traffic from runners
  * credential access spikes
  * workflow modifications in protected branches

---

## 4) Practical “minimum hardening checklist” (per infra)

### VM (Azure/AWS) minimum

* Ephemeral autoscaled runners (destroy after job) ([GitHub Docs][3])
* Dedicated subnet, egress restricted
* No inbound; use bastion/JIT if needed
* Least-privilege IAM + short-lived creds
* No secrets on disk; fetch at runtime from secret manager
* Immutable patching via golden image rollout
* Central logs + EDR

### AKS/EKS minimum

* Ephemeral runner pods (one job per pod) ([GitHub Docs][3])
* SecurityContext hardening (non-root, drop caps, RO filesystem) ([Kubernetes][10])
* NetworkPolicies default deny
* Tight RBAC (controller minimal; pods minimal) ([Kubernetes][11])
* Minimal/scanned/signed images
* Prefer emptyDir; isolate caches; encrypt PVC
* External secret store integration

---

If you want to turn this into an **organization-standard** deliverable, the next step is usually: define 2–3 **runner tiers**, then for each tier specify **allowed repos**, **runner group**, **network reach**, **IAM/RBAC**, **secrets model**, **patch cadence**, and **monitoring/alerting**. That becomes your “runner platform contract” (and stops random teams from spinning up runner snowflakes that become haunted).

[1]: https://docs.github.com/en/actions/reference/security/secure-use?utm_source=chatgpt.com "Secure use reference - GitHub Docs"
[2]: https://www.wiz.io/blog/github-actions-security-guide?utm_source=chatgpt.com "Hardening GitHub Actions: Lessons from Recent Attacks"
[3]: https://docs.github.com/en/actions/reference/runners/self-hosted-runners?utm_source=chatgpt.com "Self-hosted runners reference"
[4]: https://github.com/marketplace/actions/harden-runner?utm_source=chatgpt.com "Harden-Runner · Actions · GitHub Marketplace"
[5]: https://learn.microsoft.com/en-us/azure/security/fundamentals/identity-management-best-practices?utm_source=chatgpt.com "Azure identity & access security best practices"
[6]: https://learn.microsoft.com/en-us/azure/defender-for-cloud/recommendations-reference-compute?utm_source=chatgpt.com "Reference table for all compute security recommendations ..."
[7]: https://docs.github.com/actions/hosting-your-own-runners?utm_source=chatgpt.com "Self-hosted runners"
[8]: https://learn.microsoft.com/en-us/security/benchmark/azure/baselines/virtual-machines-linux-virtual-machines-security-baseline?utm_source=chatgpt.com "Azure security baseline for Virtual Machines - Linux ..."
[9]: https://docs.github.com/actions/how-tos/managing-self-hosted-runners/monitoring-and-troubleshooting-self-hosted-runners?utm_source=chatgpt.com "Monitoring and troubleshooting self-hosted runners"
[10]: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/?utm_source=chatgpt.com "Configure a Security Context for a Pod or Container"
[11]: https://kubernetes.io/docs/concepts/security/rbac-good-practices/?utm_source=chatgpt.com "Role Based Access Control Good Practices"
