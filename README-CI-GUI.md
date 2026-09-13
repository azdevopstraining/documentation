# CI GitHub runner — Azure portal and GitHub website

Same **working** CI setup as [README-CI.md](README-CI.md), using **only** the [Azure portal](https://portal.azure.com) and [GitHub](https://github.com/azdevopstraining/containerapp-jobs-poc-githubrunner). No Azure CLI in this file.

Create in this **order**. The environment must be created **on** `snet-aca`. You cannot add that subnet later. Attach the **route table last** (after firewall DNS proxy and GitHub FQDNs), or the runner goes Offline.

Deploy pool (same environment): [README-deploy.md](README-deploy.md)

---

## Names (working POC)

| Item | Name |
|------|------|
| Resource group | `containerapps-jobs-rg` |
| Region | West US |
| VNet | `vnet-github-runners` (`10.20.0.0/16`) |
| Private endpoint subnet | `snet-aca-pe` (`10.20.0.0/27`) |
| Container Apps subnet | `snet-aca` (`10.20.1.0/27`) |
| Firewall subnet | `AzureFirewallSubnet` (`10.20.2.0/26`) — this name is required |
| Environment | `dev-github-runners` |
| Job | `github-actions-runner-job` |
| Label | `ci` |
| ACR | `containerappsgithubacr` |
| Image | `github-actions-runner:1.0` |
| Identity | `id-github-runner` |
| Firewall | `fw-github-runners` |
| Firewall policy | `fwpol-github-runners` |
| Route table | `rt-snet-aca` |

PAT: GitHub → Settings → Developer settings → Personal access tokens. Fine-grained, this repo: **Administration** Read and write, **Actions** Read-only, **Metadata** Read-only. You will paste it into an Azure **secret**. Never commit it.

Keep the repo **private**.

The runner image is built on your PC (Docker Desktop) and pushed to ACR. The portal cannot replace that first push. After ACR public access is off, push only from the VNet or with your IP allowed.

---

## 1. Resource group

1. Portal → **Resource groups** → **Create**.
2. Name `containerapps-jobs-rg`, region **West US** → **Review + create**.

---

## 2. Managed identity

1. Search **Managed Identities** → **Create**.
2. Resource group `containerapps-jobs-rg`, region **West US**, name **`id-github-runner`** → **Create**.

---

## 3. Azure Container Registry (Premium, public on for the first push)

1. **Create a resource** → **Container Registry**.
2. Name `containerappsgithubacr` (must be globally unique). SKU **Premium**. Region **West US**.
3. Networking: leave **public** access enabled until the image is pushed.
4. After create: registry → **Networking** / **Public access** — still **All networks** for step 5.
5. **Settings** → enable **anonymous pull** off; enable data endpoint if shown.
6. **Access control (IAM)** → **Add role assignment** → **AcrPull** → managed identity **`id-github-runner`**.

If the name is taken, choose another name and use it everywhere below.

---

## 4. Build and push the image (local Docker, once)

This is the only non-portal step besides GitHub.

1. Clone or use `C:\Users\HP\Documents\Shravya\acr\container-apps-ci-cd-runner-tutorial`.
2. `entrypoint.sh` must use `--labels "${RUNNER_LABELS:-ci}"`.
3. Convert that script to Linux line endings if it was edited on Windows (otherwise the container fails with `Illegal option -`).
4. Docker Desktop running. In a terminal: log in to ACR from the portal (**Access keys** / **Login server**) or `az acr login`, then build `Dockerfile.github` and push `containerappsgithubacr.azurecr.io/github-actions-runner:1.0`.
5. Portal → ACR → **Repositories** → confirm the tag **1.0**.

Do not `git push` to `Azure-Samples/container-apps-ci-cd-runner-tutorial` (403).

---

## 5. Virtual network and three subnets

1. **Create a resource** → **Virtual network**.
2. Name `vnet-github-runners`, resource group `containerapps-jobs-rg`, region **West US**.
3. Address space **`10.20.0.0/16`**. Remove the default subnet or replace it as follows.

**Subnet 1 — private endpoints**

- Name `snet-aca-pe`, range **`10.20.0.0/27`**
- Delegate to: none
- Private endpoint network policies: **Disabled**

**Subnet 2 — Container Apps (jobs)**

- Name `snet-aca`, range **`10.20.1.0/27`**
- Subnet delegation: **`Microsoft.App/environments`**
- No private endpoints on this subnet

**Subnet 3 — firewall**

- Name **`AzureFirewallSubnet`** (exact name)
- Range **`10.20.2.0/26`** (minimum `/26`)
- No delegation

---

## 6. ACR private endpoint, then turn public access off

1. ACR → **Networking** → **Private access** → **+ Create** (or **Private endpoints** → **Create**).
2. Name `pe-acr`, region **West US**.
3. Resource type **Microsoft.ContainerRegistry/registries**, target your ACR, subresource **registry**.
4. VNet `vnet-github-runners`, subnet **`snet-aca-pe`**.
5. Integrate private DNS zone **`privatelink.azurecr.io`**, link to this VNet.
6. After the PE is **Approved**, ACR → **Networking** → **Public access** → **Disabled**.

---

## 7. Container Apps environment on `snet-aca`

If `dev-github-runners` already exists **without** a VNet, delete the CI **job**, then the **environment**, then create again.

1. **Create a resource** → **Container Apps Environment**.
2. Name `dev-github-runners`, resource group `containerapps-jobs-rg`, region **West US**.
3. **Networking**: use your virtual network. Infrastructure subnet **`snet-aca`**.
4. Workload profiles: default **Consumption** is fine.
5. Create and wait until **Succeeded**.
6. Open the environment → **Networking**. Subnet must be **`snet-aca`**.

Optional: **Public network access** **Disabled**, then a private endpoint on **`snet-aca-pe`** for the environment (inbound apps only). Jobs do not need this to run.

---

## 8. Container Apps **job** (CI)

1. **Create a resource** → **Container App Job**.
2. **Basics**: name **`github-actions-runner-job`**, environment **`dev-github-runners`**.
3. **Container**: ACR, image `github-actions-runner:1.0`, CPU **2**, memory **4 Gi**. Identity **`id-github-runner`**, pull with that identity.
4. **Trigger**: **Event**. Timeout **1800**, retry **0**, parallelism **1**, completion count **1**.
5. **Scale**: min **0**, max **10**, polling **30**.
6. **Scale rule**: type **github-runner**.
   - Auth: secret `personal-access-token` → parameter `personalAccessToken`
   - Metadata:

| Name | Value |
|------|--------|
| githubAPIURL | `https://api.github.com` |
| owner | `azdevopstraining` |
| runnerScope | `repo` |
| repos | `containerapp-jobs-poc-githubrunner` |
| labels | `ci` |
| noDefaultLabels | `true` |
| targetWorkflowQueueLength | `1` |

7. **Secret** `personal-access-token` = your **PAT**.
8. **Environment variables**:
   - `GITHUB_PAT` → secret `personal-access-token`
   - `RUNNER_LABELS` = `ci`
   - `GH_URL` = `https://github.com/azdevopstraining/containerapp-jobs-poc-githubrunner`
   - `REGISTRATION_TOKEN_API_URL` = `https://api.github.com/repos/azdevopstraining/containerapp-jobs-poc-githubrunner/actions/runners/registration-token`
9. **Review + create**.

If the wizard blocks scale auth, create the job, add the secret, then edit **Scale**.

---

## 9. Firewall policy and firewall (before the route table)

### Policy and DNS proxy

1. Search **Firewall policies** → **Create**.
2. Name `fwpol-github-runners`, region **West US**, SKU **Standard**.
3. **DNS**: enable **DNS proxy**.
4. Create the policy.

### Rule collection group `rcg-aca` (priority 200)

Add **filter** collections:

**net-dns** (priority 100, Allow, **Network** rule)

- Source `10.20.1.0/27`
- Destination `168.63.129.16`
- Protocol UDP, TCP, port **53**

**app-github** (priority 200, Allow, **Application** rule, HTTPS 443)

- Source `10.20.1.0/27`
- FQDNs: `github.com`, `api.github.com`, `codeload.github.com`, `*.actions.githubusercontent.com`, `results-receiver.actions.githubusercontent.com`, `objects.githubusercontent.com`, `objects-origin.githubusercontent.com`, `github-releases.githubusercontent.com`, `github-registry-files.githubusercontent.com`

**app-github-extra** (priority 205, Allow, Application, HTTPS 443)

- Source `10.20.1.0/27`
- FQDNs: `*.githubusercontent.com`, `*.github.com`, `*.blob.core.windows.net`, `*.pkg.github.com`, `ghcr.io`, `pkg-containers.githubusercontent.com`

**app-aca** (priority 210, Allow, Application, HTTPS 443)

- Source `10.20.1.0/27`
- FQDNs: `mcr.microsoft.com`, `*.data.mcr.microsoft.com`, `packages.aks.azure.com`, `acs-mirror.azureedge.net`, `login.microsoft.com`, `login.microsoftonline.com`, `*.identity.azure.net`

### Firewall

1. **Create a resource** → **Firewall**.
2. Name `fw-github-runners`, policy `fwpol-github-runners`.
3. VNet `vnet-github-runners` (uses **AzureFirewallSubnet**).
4. New **Standard** public IP `pip-fw-github-runners`.
5. Wait until **Succeeded**. Overview private IP should be **`10.20.2.4`**.

---

## 10. VNet DNS, then route table on `snet-aca` only

1. `vnet-github-runners` → **DNS servers** → Custom **`10.20.2.4`** → Save.
2. **Create a resource** → **Route table** → name `rt-snet-aca`, region **West US**.
3. **Routes** → **Add**:
   - Name `default-to-firewall`
   - Prefix `0.0.0.0/0`
   - Next hop **Virtual appliance**
   - Next hop address **`10.20.2.4`**
4. **Subnets** → **Associate** → `vnet-github-runners` / **`snet-aca`**.

Do **not** associate `snet-aca-pe` or `AzureFirewallSubnet`.

---

## 11. GitHub workflow (browser)

1. Repo → **Code** → `.github/workflows/multistage-cicd-pipeline.yml` → **Edit**.
2. Lint job **`runs-on: [ci]`** (not `self-hosted`).
3. Leave deploy on `ubuntu-latest` unless you follow [README-deploy.md](README-deploy.md).
4. **Commit** to `main` (or merge a PR). Local files do not count until they are on GitHub.

---

## 12. Check that it works

1. GitHub **Actions** → **Bicep CI/CD** → **Run workflow** or push.
2. **Lint and security scan** → **Requested labels: `ci`**, then the job **starts** (not waiting for many minutes).
3. Azure → job `github-actions-runner-job` → **Execution history** → **Running** then **Succeeded**.
4. **Logs**: Connected to GitHub, Listening for Jobs.
5. GitHub **Settings** → **Actions** → **Runners**: name like `github-actions-runner-job-…`, label **`ci`**, only while running. Empty afterward is normal.

**Firewall (optional):** `fw-github-runners` → **Logs** / **Diagnostic settings** → application rules **Allow** for `api.github.com` and `*.actions.githubusercontent.com` from `10.20.1.x`.

---

## If lint waits or the runner is Offline

| What you see | What to check in the portal |
|--------------|-----------------------------|
| Requested labels `self-hosted` | Workflow on **main** is still `runs-on: self-hosted` |
| Requested labels `ci`, wait forever | Scale metadata `labels=ci`; env `RUNNER_LABELS=ci`; PAT Administration write |
| Azure **Running**, GitHub **Offline** | DNS proxy on; `app-github` / `app-github-extra`; route table on `snet-aca` **after** those rules |
| Image pull failed | ACR PE Approved; public access disabled is OK if the env is on `snet-aca` |
| `Illegal option -` | Rebuild image after fixing `entrypoint.sh` line endings |

Do not point this pool at `runs-on: self-hosted` while `noDefaultLabels` is true.

---

## Traffic (working)

```text
Job replica in snet-aca
  → DNS 10.20.2.4 (firewall proxy)
  → internet via rt-snet-aca → firewall (GitHub FQDNs only)
  → image pull via pe-acr (private)
```

GitHub is still GitHub’s **public** HTTPS endpoints. The firewall **allowlists** them; it does not make GitHub private.
