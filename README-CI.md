# CI GitHub runner — working Azure CLI / PowerShell steps

This is the **CI** pool only (`runs-on: [ci]`). It matches what works in this POC: Container Apps **job** in a VNet, **ACR private endpoint**, **Azure Firewall** + **route table** + **DNS proxy**.

Use **PowerShell**. Set variables in **every new window**. Do not commit the PAT.

Portal-only version: [README-CI-GUI.md](README-CI-GUI.md)  
Deploy pool (same environment): [README-deploy.md](README-deploy.md)

---

## What you end up with

| Item | Working name |
|------|----------------|
| Resource group | `containerapps-jobs-rg` |
| Region | `westus` |
| VNet | `vnet-github-runners` (`10.20.0.0/16`) |
| PE subnet | `snet-aca-pe` (`10.20.0.0/27`) |
| Job subnet | `snet-aca` (`10.20.1.0/27`, delegated `Microsoft.App/environments`) |
| Firewall subnet | `AzureFirewallSubnet` (`10.20.2.0/26`) |
| Environment | `dev-github-runners` |
| Job | `github-actions-runner-job` |
| Label | `ci` |
| ACR | `containerappsgithubacr` |
| Image | `github-actions-runner:1.0` |
| Identity | `id-github-runner` |
| ACR PE | `pe-acr` |
| Firewall | `fw-github-runners` / policy `fwpol-github-runners` |
| Route table | `rt-snet-aca` on **`snet-aca` only** |
| VNet DNS | `10.20.2.4` (firewall private IP) |

A custom VNet is set **when the environment is created**. You cannot attach `snet-aca` later. If an environment already exists with `vnetConfiguration: null`, delete the job and environment, then recreate the environment with `--infrastructure-subnet-resource-id`.

Do **not** use `az acr build` (this subscription returns `TasksOperationsNotAllowed`). Use local `docker build` + `docker push`. Push **before** you turn ACR public access off, or push from a VM in the VNet.

---

## 0. Session variables

```powershell
$RESOURCE_GROUP = "containerapps-jobs-rg"
$LOCATION = "westus"
$ENVIRONMENT = "dev-github-runners"
$JOB_NAME = "github-actions-runner-job"
$VNET_NAME = "vnet-github-runners"
$PE_SUBNET_NAME = "snet-aca-pe"
$ACA_SUBNET_NAME = "snet-aca"
$CONTAINER_REGISTRY_NAME = "containerappsgithubacr"
$CONTAINER_IMAGE_NAME = "github-actions-runner:1.0"
$IDENTITY = "id-github-runner"
$REPO_OWNER = "azdevopstraining"
$REPO_NAME = "containerapp-jobs-poc-githubrunner"
$FW_NAME = "fw-github-runners"
$FW_PIP = "pip-fw-github-runners"
$FW_POLICY = "fwpol-github-runners"
$ROUTE_TABLE = "rt-snet-aca"

$GITHUB_PAT = "github_pat_PASTE_YOUR_TOKEN_HERE"
```

PAT: **Administration** Read and write, **Actions** Read-only, **Metadata** Read-only. Store the **PAT** in Azure, not the one-hour registration `token`.

---

## 1. Login and providers

```powershell
az login
az account set --subscription "YOUR_SUBSCRIPTION_ID"
az extension add --name containerapp --upgrade
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.ManagedIdentity
```

---

## 2. Prove the PAT can register runners

```powershell
$headers = @{
  Accept        = "application/vnd.github+json"
  Authorization = "Bearer $GITHUB_PAT"
}
Invoke-RestMethod -Method Post -Headers $headers `
  -Uri "https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/actions/runners/registration-token"
```

Expect `token` and `expires_at`. `403` means Administration is not Read and write.

---

## 3. Resource group, identity, ACR (public on until first push)

```powershell
az group create --name $RESOURCE_GROUP --location $LOCATION

az identity create --name $IDENTITY --resource-group $RESOURCE_GROUP --location $LOCATION

$IDENTITY_ID = az identity show --name $IDENTITY --resource-group $RESOURCE_GROUP --query id -o tsv
$PRINCIPAL_ID = az identity show --name $IDENTITY --resource-group $RESOURCE_GROUP --query principalId -o tsv
$IDENTITY_ID

az acr create `
  --name $CONTAINER_REGISTRY_NAME `
  --resource-group $RESOURCE_GROUP `
  --location $LOCATION `
  --sku Premium

az acr update --name $CONTAINER_REGISTRY_NAME --data-endpoint-enabled true
az acr config authentication-as-arm update --registry $CONTAINER_REGISTRY_NAME --status enabled

$ACR_ID = az acr show --name $CONTAINER_REGISTRY_NAME --query id -o tsv

az role assignment create --assignee $PRINCIPAL_ID --role AcrPull --scope $ACR_ID
```

If `az acr create` returns **AlreadyInUse**, the name is taken globally. Either use that registry in **its** subscription, or pick a new name and `az acr check-name`.

---

## 4. Build and push the runner image

Dockerfile lives in the sample clone (do **not** `git push` to Azure-Samples):

`C:\Users\HP\Documents\Shravya\acr\container-apps-ci-cd-runner-tutorial`

`entrypoint.sh` must register with labels:

```text
./config.sh ... --ephemeral --labels "${RUNNER_LABELS:-ci}" && ./run.sh
```

```powershell
cd C:\Users\HP\Documents\Shravya\acr\container-apps-ci-cd-runner-tutorial

$entrypoint = "github-actions-runner\entrypoint.sh"
$text = [System.IO.File]::ReadAllText((Resolve-Path $entrypoint))
[System.IO.File]::WriteAllText((Resolve-Path $entrypoint), ($text -replace "`r`n", "`n"))

az acr login --name $CONTAINER_REGISTRY_NAME
docker build -f Dockerfile.github -t "$CONTAINER_REGISTRY_NAME.azurecr.io/$CONTAINER_IMAGE_NAME" .
docker push "$CONTAINER_REGISTRY_NAME.azurecr.io/$CONTAINER_IMAGE_NAME"
```

Tag must be `containerappsgithubacr.azurecr.io/github-actions-runner:1.0` — do not double `.azurecr.io`.

---

## 5. VNet and subnets

```powershell
az network vnet create `
  --resource-group $RESOURCE_GROUP `
  --name $VNET_NAME `
  --location $LOCATION `
  --address-prefix 10.20.0.0/16

az network vnet subnet create `
  --resource-group $RESOURCE_GROUP `
  --vnet-name $VNET_NAME `
  --name $PE_SUBNET_NAME `
  --address-prefixes 10.20.0.0/27

az network vnet subnet update `
  --resource-group $RESOURCE_GROUP `
  --vnet-name $VNET_NAME `
  --name $PE_SUBNET_NAME `
  --private-endpoint-network-policies Disabled

az network vnet subnet create `
  --resource-group $RESOURCE_GROUP `
  --vnet-name $VNET_NAME `
  --name $ACA_SUBNET_NAME `
  --address-prefixes 10.20.1.0/27 `
  --delegations Microsoft.App/environments

az network vnet subnet create `
  --resource-group $RESOURCE_GROUP `
  --vnet-name $VNET_NAME `
  --name AzureFirewallSubnet `
  --address-prefixes 10.20.2.0/26

$ACA_SUBNET_ID = az network vnet subnet show `
  --resource-group $RESOURCE_GROUP --vnet-name $VNET_NAME --name $ACA_SUBNET_NAME --query id -o tsv
$PE_SUBNET_ID = az network vnet subnet show `
  --resource-group $RESOURCE_GROUP --vnet-name $VNET_NAME --name $PE_SUBNET_NAME --query id -o tsv

$ACA_SUBNET_ID
$PE_SUBNET_ID
```

Both IDs must print. Empty `$ACA_SUBNET_ID` causes `--infrastructure-subnet-resource-id: expected one argument`.

---

## 6. ACR private endpoint, then disable public access

```powershell
az network private-endpoint create `
  --resource-group $RESOURCE_GROUP `
  --location $LOCATION `
  --name pe-acr `
  --subnet $PE_SUBNET_ID `
  --private-connection-resource-id $ACR_ID `
  --connection-name pe-conn-acr `
  --group-id registry

az network private-dns zone create --resource-group $RESOURCE_GROUP --name privatelink.azurecr.io

az network private-dns link vnet create `
  --resource-group $RESOURCE_GROUP `
  --zone-name privatelink.azurecr.io `
  --name dns-link-acr `
  --virtual-network $VNET_NAME `
  --registration-enabled false

az network private-endpoint dns-zone-group create `
  --resource-group $RESOURCE_GROUP `
  --endpoint-name pe-acr `
  --name acr-zone-group `
  --private-dns-zone privatelink.azurecr.io `
  --zone-name privatelink.azurecr.io

az acr update --name $CONTAINER_REGISTRY_NAME --public-network-enabled false
```

After this, `docker push` from your laptop fails unless you allow your IP or push from the VNet.

---

## 7. Container Apps environment **on** `snet-aca`

If `dev-github-runners` already exists **without** a VNet, delete the job, then the environment, then create:

```powershell
az containerapp env create `
  --name $ENVIRONMENT `
  --resource-group $RESOURCE_GROUP `
  --location $LOCATION `
  --infrastructure-subnet-resource-id $ACA_SUBNET_ID
```

Wait until it succeeds. Confirm the subnet is set (must not be null):

```powershell
az containerapp env show -n $ENVIRONMENT -g $RESOURCE_GROUP `
  --query "properties.vnetConfiguration.infrastructureSubnetId" -o tsv
```

Optional inbound lock-down (jobs do not need this to run):

```powershell
$ENVIRONMENT_ID = az containerapp env show -n $ENVIRONMENT -g $RESOURCE_GROUP --query id -o tsv
az containerapp env update --id $ENVIRONMENT_ID --public-network-access Disabled

az network private-endpoint create `
  --resource-group $RESOURCE_GROUP --location $LOCATION `
  --name pe-dev-github-runners `
  --subnet $PE_SUBNET_ID `
  --private-connection-resource-id $ENVIRONMENT_ID `
  --connection-name pe-conn-dev-github-runners `
  --group-id managedEnvironments
```

---

## 8. Event-driven CI job (label `ci`)

```powershell
az containerapp job create `
  --name $JOB_NAME `
  --resource-group $RESOURCE_GROUP `
  --environment $ENVIRONMENT `
  --trigger-type Event `
  --replica-timeout 1800 `
  --replica-retry-limit 0 `
  --replica-completion-count 1 `
  --parallelism 1 `
  --image "$CONTAINER_REGISTRY_NAME.azurecr.io/$CONTAINER_IMAGE_NAME" `
  --min-executions 0 `
  --max-executions 10 `
  --polling-interval 30 `
  --scale-rule-name "github-runner" `
  --scale-rule-type "github-runner" `
  --scale-rule-metadata "githubAPIURL=https://api.github.com" "owner=$REPO_OWNER" "runnerScope=repo" "repos=$REPO_NAME" "labels=ci" "noDefaultLabels=true" "targetWorkflowQueueLength=1" `
  --scale-rule-auth "personalAccessToken=personal-access-token" `
  --cpu "2.0" `
  --memory "4Gi" `
  --secrets "personal-access-token=$GITHUB_PAT" `
  --env-vars "GITHUB_PAT=secretref:personal-access-token" "RUNNER_LABELS=ci" "GH_URL=https://github.com/$REPO_OWNER/$REPO_NAME" "REGISTRATION_TOKEN_API_URL=https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/actions/runners/registration-token" `
  --registry-server "$CONTAINER_REGISTRY_NAME.azurecr.io" `
  --mi-user-assigned $IDENTITY_ID `
  --registry-identity $IDENTITY_ID
```

If the job exists, update PAT/env instead of create:

```powershell
az containerapp job update --name $JOB_NAME --resource-group $RESOURCE_GROUP `
  --set-env-vars "RUNNER_LABELS=ci" "GH_URL=https://github.com/$REPO_OWNER/$REPO_NAME" "REGISTRATION_TOKEN_API_URL=https://api.github.com/repos/$REPO_OWNER/$REPO_NAME/actions/runners/registration-token"

az containerapp job secret set --name $JOB_NAME --resource-group $RESOURCE_GROUP `
  --secrets "personal-access-token=$GITHUB_PAT"
```

Confirm `GH_URL` and `RUNNER_LABELS` have **values**, not empty names.

---

## 9. Azure Firewall (FQDN allowlist) — create **before** the UDR

```powershell
az network public-ip create -g $RESOURCE_GROUP -n $FW_PIP --location $LOCATION --sku Standard --allocation-method Static

az network firewall policy create -g $RESOURCE_GROUP -n $FW_POLICY --location $LOCATION --sku Standard --enable-dns-proxy true

az network firewall policy rule-collection-group create `
  -g $RESOURCE_GROUP --policy-name $FW_POLICY --name rcg-aca --priority 200

az network firewall policy rule-collection-group collection add-filter-collection `
  -g $RESOURCE_GROUP --policy-name $FW_POLICY --rule-collection-group-name rcg-aca `
  --name net-dns --collection-priority 100 --action Allow --rule-type NetworkRule `
  --rule-name allow-azure-dns --ip-protocols UDP TCP `
  --source-addresses 10.20.1.0/27 --destination-addresses 168.63.129.16 --destination-ports 53

az network firewall policy rule-collection-group collection add-filter-collection `
  -g $RESOURCE_GROUP --policy-name $FW_POLICY --rule-collection-group-name rcg-aca `
  --name app-github --collection-priority 200 --action Allow --rule-type ApplicationRule `
  --rule-name allow-github --protocols Https=443 --source-addresses 10.20.1.0/27 `
  --target-fqdns github.com api.github.com codeload.github.com "*.actions.githubusercontent.com" results-receiver.actions.githubusercontent.com objects.githubusercontent.com objects-origin.githubusercontent.com github-releases.githubusercontent.com github-registry-files.githubusercontent.com

az network firewall policy rule-collection-group collection add-filter-collection `
  -g $RESOURCE_GROUP --policy-name $FW_POLICY --rule-collection-group-name rcg-aca `
  --name app-github-extra --collection-priority 205 --action Allow --rule-type ApplicationRule `
  --rule-name allow-github-extra --protocols Https=443 --source-addresses 10.20.1.0/27 `
  --target-fqdns "*.githubusercontent.com" "*.github.com" "*.blob.core.windows.net" "*.pkg.github.com" ghcr.io pkg-containers.githubusercontent.com

az network firewall policy rule-collection-group collection add-filter-collection `
  -g $RESOURCE_GROUP --policy-name $FW_POLICY --rule-collection-group-name rcg-aca `
  --name app-aca --collection-priority 210 --action Allow --rule-type ApplicationRule `
  --rule-name allow-aca-platform --protocols Https=443 --source-addresses 10.20.1.0/27 `
  --target-fqdns mcr.microsoft.com "*.data.mcr.microsoft.com" packages.aks.azure.com acs-mirror.azureedge.net login.microsoft.com login.microsoftonline.com "*.identity.azure.net"

az network firewall create `
  -g $RESOURCE_GROUP -n $FW_NAME --location $LOCATION `
  --sku AZFW_VNet --tier Standard --vnet-name $VNET_NAME --firewall-policy $FW_POLICY

az network firewall ip-config create `
  -g $RESOURCE_GROUP --firewall-name $FW_NAME --name fw-ipconfig `
  --public-ip-address $FW_PIP --vnet-name $VNET_NAME

$FW_PRIVATE_IP = az network firewall show -g $RESOURCE_GROUP -n $FW_NAME --query "ipConfigurations[0].privateIPAddress" -o tsv
$FW_PRIVATE_IP
```

Expect `10.20.2.4`. Wait until the firewall is **Succeeded** before the next section.

Enable DNS proxy on an existing policy:

```powershell
az network firewall policy update -g $RESOURCE_GROUP -n $FW_POLICY --enable-dns-proxy true
```

---

## 10. VNet DNS, then attach the route table (**last**)

Attach the UDR only after DNS proxy and GitHub FQDNs exist. Attaching too early made runners **Offline** and lint **Waiting for a runner**.

```powershell
az network vnet update -g $RESOURCE_GROUP -n $VNET_NAME --dns-servers $FW_PRIVATE_IP

az network route-table create -g $RESOURCE_GROUP -n $ROUTE_TABLE --location $LOCATION

az network route-table route create `
  -g $RESOURCE_GROUP --route-table-name $ROUTE_TABLE --name default-to-firewall `
  --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address $FW_PRIVATE_IP

az network vnet subnet update `
  -g $RESOURCE_GROUP --vnet-name $VNET_NAME -n $ACA_SUBNET_NAME --route-table $ROUTE_TABLE
```

Do **not** attach this route table to `snet-aca-pe` or `AzureFirewallSubnet`.

To detach later (recovery only):

```powershell
az network vnet subnet update -g $RESOURCE_GROUP --vnet-name $VNET_NAME -n $ACA_SUBNET_NAME --route-table null
```

---

## 11. GitHub workflow

Lint job on **`main`** must be:

```yaml
runs-on: [ci]
```

Not `self-hosted`. Deploy jobs can stay `ubuntu-latest` (OIDC).

Push that YAML to GitHub. Local-only edits do not change the run.

---

## 12. Verify a run

Re-run **Bicep CI/CD**. Lint **Requested labels** must be **`ci`**.

```powershell
az containerapp job execution list -n $JOB_NAME -g $RESOURCE_GROUP -o table

az containerapp job logs show -n $JOB_NAME -g $RESOURCE_GROUP --container $JOB_NAME --tail 50
```

Good logs: `Connected to GitHub`, `Listening for Jobs`.  
GitHub **Settings → Actions → Runners**: runner with label **`ci`** only while the replica runs. Empty afterward is normal (ephemeral).

A replica **Running** in Azure but **Offline** in GitHub usually means the firewall dropped `*.actions.githubusercontent.com`. Check policy DNS proxy and `app-github` / `app-github-extra`. Stop the stuck execution before retrying:

```powershell
az containerapp job stop -n $JOB_NAME -g $RESOURCE_GROUP --job-execution-name <execution-name>
```

---

## Traffic (working)

```text
snet-aca  →  DNS 10.20.2.4 (firewall proxy)
          →  0.0.0.0/0 → fw-github-runners → allowed GitHub FQDNs
          →  ACR via pe-acr (private, in-VNet; not the firewall)
```

---

## Do not

| Command / action | Why |
|------------------|-----|
| `az acr build` | ACR Tasks blocked |
| `git push` to Azure-Samples | 403; push the **image** to ACR |
| UDR before firewall FQDNs + DNS proxy | Runner Offline; lint waits forever |
| Store curl `token` as the job secret | Expires ~1 hour |
| Docker-in-Docker / `azure/cli@v2` on this runner | Not supported on Container Apps jobs unless `az` is in the image |
