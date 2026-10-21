# Service Principals

[← Back to README](../README.md)

## In one line
A **service principal** is an identity for an **application, script, or pipeline** (not a person) that signs in to Azure with a **secret, certificate, or federated credential** and gets access through **RBAC roles**.

## When to use it (and when not)
| Situation | Use |
|---|---|
| CI/CD pipeline (GitHub Actions, Azure DevOps) deploys to Azure | **Service principal** – ideally with **workload identity federation** (no secret) |
| Script on an **on-premises** server automates Azure | **Service principal** with a **certificate** |
| Code running **in** Azure (VM, App Service, Function) | **Managed identity** instead – no secret to manage |
| A person signs in | A **user** account |

## Key concepts
- Created from an **app registration** in Microsoft Entra ID (see [App Registrations](app-registration.md))
- Identified by an **Application (client) ID** and lives in a **tenant**
- Authenticates with:

| Credential | Security | Notes |
|---|---|---|
| **Client secret** | Lowest | A password. Expires (max 24 months in the portal), must be rotated |
| **Certificate** | Better | Private key stays on the machine |
| **Federated credential** | Best | No secret at all – trusts tokens from GitHub, Kubernetes, etc. See [Federation](federation.md) |

- Gets access to resources through **Azure RBAC** role assignments (and to Graph via API permissions)

### Service principal types
| Type | Example |
|---|---|
| **Application** | From your own or a third-party app registration |
| **Managed identity** | Created and managed by Azure for a resource |
| **Legacy** | Old apps created before app registrations |

## Exam traps
- Code runs **in Azure** → managed identity, not a service principal
- **Secrets expire** → pipeline suddenly fails with "invalid client secret" / AADSTS7000222
- `az ad sp create-for-rbac` creates **both** the app registration **and** the service principal, and can assign a role in one go
- The **secret is shown only once** at creation
- Service principal = **identity**, the app registration = **definition** (same client ID, different object IDs)
- Give the SP the **narrowest scope** (one RG), not the whole subscription

## Do it

### Portal
1. **Entra ID → App registrations → + New registration** → name → Register (the SP is created automatically in your tenant)
2. **Certificates & secrets → + New client secret** → copy the **Value** now
3. Target RG → **Access control (IAM) → Add role assignment** → Contributor → **Members: User, group, or service principal** → search the app name

### Azure CLI
```bash
SUB=$(az account show --query id -o tsv)

# App registration + service principal + role assignment in one command
az ad sp create-for-rbac --name sp-deploy-dev \
  --role Contributor --scopes /subscriptions/$SUB/resourceGroups/rg-dev
# Output: appId, password (secret), tenant – save the password now!

# Sign in as the service principal
az login --service-principal -u <appId> -p <password> --tenant <tenantId>

# Rotate the secret
az ad sp credential reset --id <appId>

az ad sp list --display-name sp-deploy-dev -o table
```

### PowerShell
```powershell
$sp = New-AzADServicePrincipal -DisplayName sp-deploy-dev
$sp.PasswordCredentials.SecretText          # the secret – shown once

New-AzRoleAssignment -ApplicationId $sp.AppId -RoleDefinitionName Contributor -ResourceGroupName rg-dev

$cred = New-Object PSCredential($sp.AppId, (ConvertTo-SecureString $sp.PasswordCredentials.SecretText -AsPlainText -Force))
Connect-AzAccount -ServicePrincipal -Credential $cred -Tenant <tenantId>
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| **AADSTS7000215** "Invalid client secret provided" | Wrong value (you copied the **Secret ID** instead of the **Value**) | Create a new secret, copy the **Value** |
| **AADSTS7000222** "client secret keys … are expired" | Secret expired | New secret, update the pipeline / Key Vault |
| Signs in but **AuthorizationFailed** | No role, or role on the wrong scope | Add RBAC role on the right RG/subscription |
| SP not found when assigning a role | Searching by display name in the wrong tenant, or SP not created yet | Search by app name / client ID, check tenant |
| Pipeline works for one RG, fails for another | SP's role is scoped to one RG | Add a role at the other RG |

## Clean up
```bash
az ad sp delete --id <appId>
az ad app delete --id <appId>
```

## Check yourself
<details><summary>1. A GitHub Actions workflow deploys Bicep to rg-prod. Security doesn't want any secret stored in GitHub.</summary>

Service principal (app registration) with a **federated credential** for the GitHub repo, Contributor on rg-prod.
</details>

<details><summary>2. A Function App must read Key Vault secrets. Service principal or managed identity?</summary>

**Managed identity** – it runs in Azure.
</details>

<details><summary>3. A deployment pipeline that worked for a year suddenly fails to sign in. Most likely cause?</summary>

The **client secret expired**.
</details>

## Related
- [App Registrations](app-registration.md)
- [Managed Identities](managed-identities.md) – a service principal that Azure manages for you
- [Federation](federation.md)
