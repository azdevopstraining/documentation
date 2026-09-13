# Deploy runner — Azure portal and GitHub website only

Create a **second** GitHub runner pool for **deploy**. Keep the existing **ci** runner.

This document uses **only**:

- [Azure portal](https://portal.azure.com)
- [GitHub](https://github.com/azdevopstraining/containerapp-jobs-poc-githubrunner) in the browser

There is **no Azure CLI** in these steps.

Use the **same** Container Apps environment `dev-github-runners`. Do **not** create another VNet, subnet, firewall, route table, or ACR private endpoint. Replicas of the new job still run in `snet-aca`, so they already use the route table and firewall.

---

## What already exists (do not create again)

| Resource | Name |
|----------|------|
| Resource group | `containerapps-jobs-rg` |
| Environment | `dev-github-runners` (subnet `snet-aca`) |
| CI job | `github-actions-runner-job` (label `ci`) |
| ACR | `containerappsgithubacr` (image `github-actions-runner:1.0`) |
| ACR private endpoint | `pe-acr` on `snet-aca-pe` |
| Identity | `id-github-runner` |
| VNet | `vnet-github-runners` |
| Firewall | `fw-github-runners` + policy `fwpol-github-runners` |
| Route table | `rt-snet-aca` on `snet-aca` |
| VNet DNS | `10.20.2.4` (firewall DNS proxy) |

The deploy job inherits all of that. **Do not add new firewall FQDN rules.**

---

## What you will create

| Item | Value |
|------|--------|
| Container Apps job | `github-actions-deploy-job` |
| GitHub label | `deploy` |
| Max replicas | 5 |
| Identity | `id-github-deploy` (or reuse `id-github-runner`) |

Lint stays on label `ci`. Deploy uses label `deploy`. Azure login in the workflow stays **OIDC** (GitHub Environment secrets). The PAT only **registers** the runner.

---

## Checklist before the portal

- CI already runs with label `ci` through the firewall.
- Repo is **private**.
- Fine-grained PAT: **Administration** Read and write, **Actions** Read-only, **Metadata** Read-only. You will paste it into an Azure **secret**, not into Git.
- Container Apps jobs **cannot run Docker**. GitHub’s `azure/cli` action will fail until Azure CLI is **inside the runner image**. If the image has no `az`, leave deploy on GitHub-hosted `ubuntu-latest` until you rebuild the image.

---

## Part A — Confirm existing security in the portal

Search each name in the portal search bar.

### A1. Environment uses `snet-aca`

1. Open **Container Apps Environment** `dev-github-runners`.
2. Open **Networking** (or Overview).
3. Infrastructure subnet should be **`snet-aca`**.

### A2. Route table

1. Open **Virtual network** `vnet-github-runners`.
2. Left menu **Subnets** → click **`snet-aca`**.
3. **Route table** must be **`rt-snet-aca`**.
4. Open **Route table** `rt-snet-aca` → **Routes**:
   - Prefix `0.0.0.0/0`
   - Next hop **Virtual appliance**
   - Next hop IP **`10.20.2.4`**

Open subnet **`snet-aca-pe`**. It must **not** use `rt-snet-aca`.

### A3. Firewall policy

1. Open **Firewall** `fw-github-runners`.
2. Click the linked **Firewall policy** `fwpol-github-runners`.
3. **DNS** → DNS proxy **Enabled**.
4. **Rule collections** → group **`rcg-aca`**. Confirm:

| Collection | What you should see |
|------------|---------------------|
| `net-dns` | Port 53 to `168.63.129.16` |
| `app-github` | `github.com`, `api.github.com`, `*.actions.githubusercontent.com` |
| `app-github-extra` | `*.githubusercontent.com`, `*.github.com`, `*.blob.core.windows.net`, `ghcr.io` |
| `app-aca` | `mcr.microsoft.com`, `login.microsoft.com` |

Source address **`10.20.1.0/27`**. Leave these as they are.

### A4. VNet DNS

1. `vnet-github-runners` → **DNS servers**.
2. Custom DNS **`10.20.2.4`**.

### A5. ACR private endpoint

1. Open **Private endpoint** `pe-acr`.
2. Subnet **`snet-aca-pe`**, connection **Approved**.
3. Open **Private DNS zone** `privatelink.azurecr.io` → **Recordsets** → A records for your registry.

---

## Part B — Optional identity (portal)

POC: skip and reuse `id-github-runner`.

Stricter:

1. Search **Managed Identities** → **Create**.
2. Resource group `containerapps-jobs-rg`, region **West US**, name **`id-github-deploy`** → **Create**.
3. Open **Container registry** `containerappsgithubacr` → **Access control (IAM)** → **Add** → **Add role assignment**.
4. Role **AcrPull** → Members → **Managed identity** → `id-github-deploy` → **Review + assign**.

Do **not** assign **Contributor**. Deploy to Azure uses GitHub **OIDC**, not this identity.

---

## Part C — Create the job in the portal

1. Portal home → **Create a resource**.
2. Search **Container App Job** → **Create**.

### Basics

- Resource group: `containerapps-jobs-rg`
- Job name: **`github-actions-deploy-job`**
- Region: **West US**
- Container Apps environment: **`dev-github-runners`** (pick the existing one)

### Container

- Image source: **Azure Container Registry**
- Registry: `containerappsgithubacr` (or yours)
- Image and tag: `github-actions-runner` : `1.0`
- CPU **2**, Memory **4 Gi**
- Command: leave empty

### Identity and registry

- **User assigned** managed identity: `id-github-deploy` or `id-github-runner`
- Registry authentication: **Managed identity** (same identity)

### Trigger and scale

- Trigger type: **Event**
- Replica timeout: **1800**
- Replica retry limit: **0**
- Parallelism: **1**
- Replica completion count: **1**
- Minimum executions: **0**
- Maximum executions: **5**
- Polling interval: **30**

### Scale rule

Add a scale rule:

- Name: `github-runner`
- Type: **github-runner** (if the list says **Custom**, type `github-runner`)

**Authentication**

- Secret name: `personal-access-token`
- Trigger parameter: `personalAccessToken`

**Metadata** (add each row)

| Name | Value |
|------|--------|
| githubAPIURL | `https://api.github.com` |
| owner | `azdevopstraining` |
| runnerScope | `repo` |
| repos | `containerapp-jobs-poc-githubrunner` |
| labels | `deploy` |
| noDefaultLabels | `true` |
| targetWorkflowQueueLength | `1` |

### Secrets

- Name: `personal-access-token`
- Type: **Value**
- Paste the **PAT** (starts with `github_pat_` or `ghp_`). Do **not** paste a short-lived registration token.

If the wizard will not take auth until the secret exists: finish **Create**, then open the job → **Secrets** → add the PAT → **Scale** → edit the rule and attach the secret.

### Environment variables

| Name | Value |
|------|--------|
| GITHUB_PAT | Secret reference **`personal-access-token`** |
| RUNNER_LABELS | `deploy` |
| GH_URL | `https://github.com/azdevopstraining/containerapp-jobs-poc-githubrunner` |
| REGISTRATION_TOKEN_API_URL | `https://api.github.com/repos/azdevopstraining/containerapp-jobs-poc-githubrunner/actions/runners/registration-token` |

**Review + create**. Wait until the resource shows **Succeeded**.

---

## Part D — Point only deploy at this pool (GitHub website)

Azure cannot set `runs-on`. Edit the workflow in GitHub.

1. Open the repo in the browser.
2. **Code** → folder **`.github`** → **`workflows`** → **`multistage-cicd-pipeline.yml`**.
3. Pencil **Edit**.
4. Find the **Lint and security scan** job. Leave **`runs-on: [ci]`**.
5. Find the **Deploy** job (`name: 'Deploy ${{ matrix.environment }}'`).
6. Change **only** that job from `runs-on: ubuntu-latest` to:

   `runs-on: [deploy]`

7. Do **not** remove **`azure/login`**. It must still use GitHub secrets `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`.
8. **Commit changes** (commit on a branch and open a pull request, or commit to `main` if that is your process).

GitHub **Settings** → **Environments** → `dev`, `staging`, `production`:

- Add **Required reviewers** on production if you want approvals.
- Confirm the three Azure OIDC secrets exist on each environment.

---

## Part E — Check in the portal and GitHub (no CLI)

### Job in Azure

1. Search **`github-actions-deploy-job`**.
2. **Scale**: min 0, max 5, metadata **labels** = `deploy`.
3. **Containers** / environment variables: **RUNNER_LABELS** = `deploy`.
4. Environment still **`dev-github-runners`**.

### Run a workflow

1. GitHub → **Actions** → **Bicep CI/CD** → **Run workflow** (or merge to `main`).
2. Open the **Deploy** job (not lint).
3. The log header **Requested labels** must say **`deploy`**.

### Execution

1. Azure job → **Execution history** (or **Executions**).
2. A new row **Running**, then **Succeeded**, name like `github-actions-deploy-job-…`.
3. Open **Logs** on that execution. You should see **Connected to GitHub** and **Listening for Jobs**.

### Runners page

GitHub → **Settings** → **Actions** → **Runners** (only while the replica is running):

- Name starts with `github-actions-deploy-job-`
- Labels include **deploy**

Lint should still use `github-actions-runner-job-…` with label **ci**. After the job, the runner list may be empty. That is normal.

### Firewall logs (optional)

1. Open **`fw-github-runners`**.
2. **Monitoring** → **Logs** (or **Diagnostic settings** first: send **Application rule** logs to a Log Analytics workspace).
3. Open recent **Application rule** log entries.
4. **Action** should be **Allow** for GitHub hostnames. Source IPs start with `10.20.1.` (CI and deploy share this range).

---

## What not to create

| Do not create | Why |
|---------------|-----|
| New Container Apps **environment** | Same env already on `snet-aca` |
| New VNet or `snet-aca` | One delegated subnet per environment |
| Job in `snet-aca-pe` | That subnet is only for private endpoints |
| New route table or extra `0.0.0.0/0` | `rt-snet-aca` already applies |
| Extra firewall FQDN collection | Same source `10.20.1.0/27` |
| Contributor on the job identity | Use OIDC in GitHub |

Do not set deploy **`runs-on: self-hosted`**. That fights the `ci` pool and `noDefaultLabels`.

---

## If GitHub says “Waiting for a runner”

1. On GitHub, open the workflow file **on `main`** and confirm Deploy is `runs-on: [deploy]`.
2. In the Azure job, scale metadata **labels** = `deploy`, **noDefaultLabels** = `true`.
3. Env **RUNNER_LABELS** = `deploy`.
4. PAT: Administration **Read and write**.
5. `snet-aca` still has route table `rt-snet-aca`; firewall DNS proxy on; VNet DNS `10.20.2.4`.
6. If Azure shows **Running** but GitHub **Offline**, the firewall is dropping Actions. In firewall **Logs**, confirm **Allow** for `*.actions.githubusercontent.com`.
