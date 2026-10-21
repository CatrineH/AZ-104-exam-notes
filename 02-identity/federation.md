# Federation

[← Back to README](../README.md)

## In one line
**Federation** is a **trust** between identity systems: instead of sharing passwords or secrets, one system accepts **tokens** issued by another.

## When to use it (and when not)
| Situation | Type of federation |
|---|---|
| GitHub Actions / Azure DevOps / Kubernetes must deploy to Azure **without a stored secret** | **Workload identity federation** (federated credential on an app registration or user-assigned managed identity) |
| Users sign in to Entra with an on-prem identity provider (AD FS) | **Federated domain** (hybrid identity) |
| Guests from another company sign in with their own accounts | **B2B** – trust their home tenant ([Guest Users](guest-users.md)) |
| Users sign in to a SaaS app (Salesforce) with their Entra account | **SAML/OIDC SSO** via an enterprise app |

## Key concepts
- No long-term secrets are exchanged between systems
- One system trusts tokens issued by another
- Example: a GitHub Actions pipeline authenticates to Azure using **workload identity federation**, with no client secret stored in GitHub

### Workload identity federation – how it works
```
GitHub Actions job
  │ 1. GitHub issues an OIDC token: "repo CatrineH/notes, branch main"
  ▼
Microsoft Entra ID
  │ 2. Checks the federated credential: issuer + subject match?
  │ 3. Yes → issues an Azure access token for the app/managed identity
  ▼
Azure (deploys with the RBAC role the app has)
```

A **federated credential** contains:
| Field | Example |
|---|---|
| **Issuer** | `https://token.actions.githubusercontent.com` |
| **Subject** | `repo:CatrineH/notes:ref:refs/heads/main` |
| **Audience** | `api://AzureADTokenExchange` |

### Hybrid sign-in options (for context)
| Option | Password checked by |
|---|---|
| **Password hash sync (PHS)** | Entra ID (hash of the hash synced) – Microsoft's recommended default |
| **Pass-through authentication (PTA)** | On-prem AD via an agent |
| **Federation (AD FS)** | On-prem AD FS servers |

## Exam traps
- "No secrets in the pipeline" / "secretless" → **workload identity federation**
- Federated credential must match **exactly**: a workflow on another branch or an environment has a **different subject**
- Federated credentials can be added to an **app registration** or a **user-assigned** managed identity – not a system-assigned one
- Federation still needs an **RBAC role** – trust gives authentication, not authorization

## Do it

### Portal
**App registrations → your app → Certificates & secrets → Federated credentials → + Add credential** → scenario **GitHub Actions deploying Azure resources** → organization, repository, entity (branch/environment/PR) → Add. Then give the app an RBAC role.

### Azure CLI
```bash
cat > fic.json <<'EOF'
{
  "name": "github-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:CatrineH/notes:ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}
EOF
az ad app federated-credential create --id <appId> --parameters fic.json

# For a user-assigned managed identity
az identity federated-credential create -g $RG --identity-name id-github -n github-main \
  --issuer https://token.actions.githubusercontent.com \
  --subject repo:CatrineH/notes:ref:refs/heads/main --audiences api://AzureADTokenExchange
```

GitHub workflow uses `azure/login` with `client-id`, `tenant-id`, `subscription-id` – no secret – and `permissions: id-token: write`.

### PowerShell
```powershell
New-AzADAppFederatedCredential -ApplicationObjectId <app-object-id> -Name github-main `
  -Issuer "https://token.actions.githubusercontent.com" `
  -Subject "repo:CatrineH/notes:ref:refs/heads/main" -Audience "api://AzureADTokenExchange"
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| **AADSTS70021** "No matching federated identity record found" | **Subject** doesn't match (other branch, environment, PR) | Add a federated credential for that subject |
| GitHub: "Unable to get ACTIONS_ID_TOKEN_REQUEST_URL" | Workflow lacks `id-token: write` permission | Add it under `permissions:` |
| Login works, deployment fails with AuthorizationFailed | App has no role on the target | Assign RBAC role |

## Clean up
```bash
az ad app federated-credential delete --id <appId> --federated-credential-id github-main
```

## Check yourself
<details><summary>1. A GitHub workflow must deploy without any secret stored in GitHub. What do you configure?</summary>

A **federated credential** on an app registration (or user-assigned MI) + an RBAC role.
</details>

<details><summary>2. The workflow works on `main` but fails on `dev` with AADSTS70021.</summary>

The federated credential's **subject** only matches `main`. Add one for `dev`.
</details>

## Related
- [Service Principals](service-principals.md)
- [App Registrations](app-registration.md)
- [Managed Identities](managed-identities.md)
