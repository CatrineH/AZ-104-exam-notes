# Custom Roles

[← Back to README](../README.md) · Lab 08

## In one line
When no built-in Azure role fits, write your own **JSON role definition** with exactly the actions needed – **least privilege**.

## When to use it (and when not)
| Situation | Use |
|---|---|
| Operators may only **start/restart/stop** VMs, not change them | **Custom role** |
| A built-in role is close enough (Reader, VM Contributor, Storage Blob Data Reader) | **Built-in role** – always check first |
| Need to manage users or other directory objects | Not an Azure custom role – that's an **Entra custom role** (P1) |
| Need to **block** something for everyone, including Owners | Not a role – use **Azure Policy** or a **lock** |

## Key concepts

### The JSON definition

```json
{
  "Name": "VM Power Operator",
  "IsCustom": true,
  "Description": "Can start, restart, and stop VMs",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/powerOff/action",
    "Microsoft.Compute/virtualMachines/deallocate/action"
  ],
  "NotActions": [],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/<subscription-id>/resourceGroups/rg-dev"
  ]
}
```

| Property | Meaning |
|---|---|
| **Actions** | Allowed **control plane** operations |
| **NotActions** | Operations **removed** from Actions (for example `*` minus delete) |
| **DataActions** | Allowed **data plane** operations (for example reading blobs) |
| **NotDataActions** | Data operations removed from DataActions |
| **AssignableScopes** | **Where** the role can be assigned (and seen) |

**NotActions is not a deny.** If another role assignment gives the same permission, the user still has it.

**Operation format:** `{Provider}/{resourceType}/{action}` – for example `Microsoft.Storage/storageAccounts/listKeys/action`. Wildcards: `Microsoft.Compute/*`, `*/read`.

### Minimum permissions for a VM restart
- `Microsoft.Compute/virtualMachines/read`
- `Microsoft.Compute/virtualMachines/restart/action`

## Exam traps
- Creating custom roles requires `Microsoft.Authorization/roleDefinitions/write` → you must be **Owner** or **User Access Administrator**
- **Actions** (control plane) vs **DataActions** (data plane). "Read blob contents" = **DataActions** `Microsoft.Storage/storageAccounts/blobServices/containers/blobs/read`
- **NotActions ≠ deny.** Another role can still grant it
- A custom role can only be assigned **inside its AssignableScopes**. Want to use it in another subscription → add that subscription to AssignableScopes
- Only **one management group** can be in AssignableScopes, and a custom role with a management group scope can't use **DataActions**
- You can't delete a custom role while it still has **role assignments**
- Up to **5,000** custom roles per tenant
- Start by **cloning** a built-in role – fastest way in the portal

## Do it

### Portal
**Subscription (or RG) → Access control (IAM) → + Add → Add custom role** → Baseline: **Clone a role** / Start from scratch / Start from JSON → **Permissions → + Add permissions** → **Assignable scopes** → **JSON** tab to review → Create.
Then **Add role assignment** → your custom role appears under the **Job function roles** tab.

### Azure CLI
```bash
# Look at a built-in role to copy from
az role definition list --name "Virtual Machine Contributor" -o json > vm-contrib.json

# List provider operations to find action names
az provider operation show --namespace Microsoft.Compute \
  --query "resourceTypes[?name=='virtualMachines'].operations[].name" -o tsv

az role definition create --role-definition vm-power-operator.json
az role definition update --role-definition vm-power-operator.json
az role definition list --custom-role-only true -o table

az role assignment create --assignee sarah@contoso.onmicrosoft.com \
  --role "VM Power Operator" --resource-group rg-dev
```

### PowerShell
```powershell
# Start from a built-in role
$role = Get-AzRoleDefinition "Virtual Machine Contributor"
$role | ConvertTo-Json | Out-File .\vm-contrib.json

Get-AzProviderOperation "Microsoft.Compute/virtualMachines/*" | Select-Object Operation, Description

New-AzRoleDefinition -InputFile .\vm-power-operator.json
Set-AzRoleDefinition -InputFile .\vm-power-operator.json       # update (needs "Id" in the JSON)
Get-AzRoleDefinition -Custom | Select-Object Name, Id
New-AzRoleAssignment -SignInName sarah@contoso.onmicrosoft.com -RoleDefinitionName "VM Power Operator" -ResourceGroupName rg-dev
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "AuthorizationFailed … roleDefinitions/write" | You're Contributor, not Owner/UAA | Ask an Owner or **User Access Administrator** |
| Custom role not shown when assigning | Current scope is outside **AssignableScopes** | Add the scope to AssignableScopes and update the role |
| User can see VMs but restart fails | Missing `restart/action` (or the role isn't propagated yet) | Add the action, wait a few minutes, sign out/in |
| User has a permission you removed with NotActions | Another role assignment grants it | Check **IAM → Check access** for all their assignments |
| "Role definition has assignments, cannot delete" | Role still assigned | Remove the assignments first |
| JSON fails with "DataActions not allowed" | Management group in AssignableScopes | Use subscription/RG scopes for data roles |

## Clean up
```bash
az role assignment delete --assignee sarah@contoso.onmicrosoft.com --role "VM Power Operator" --resource-group rg-dev
az role definition delete --name "VM Power Operator"
```

## Check yourself
<details><summary>1. Helpdesk must restart VMs in rg-prod and nothing else. Built-in roles give too much.</summary>

**Custom role** with `virtualMachines/read` and `restart/action`, AssignableScopes = rg-prod.
</details>

<details><summary>2. A user with Contributor on the subscription can't create a custom role. Why?</summary>

Needs `roleDefinitions/write` – **Owner** or **User Access Administrator**.
</details>

<details><summary>3. The role has `"NotActions": ["*/delete"]` but the user can still delete. Why?</summary>

NotActions isn't a deny – **another role assignment** grants delete.
</details>

## Related
- [Azure RBAC](../04-governance/azure-rbac.md)
- [Entra Roles vs Azure RBAC](entra-roles-vs-rbac.md)
