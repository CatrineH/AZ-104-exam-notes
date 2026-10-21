# Entra Roles vs Azure RBAC Roles

[← Back to README](../README.md) · Lab 09

## In one line
Two **completely separate** role systems: **Entra roles** manage the **directory** (users, groups, apps), **Azure RBAC roles** manage **Azure resources** (VMs, storage, networks).

**Arguably the most important concept for the identity part of the exam.**

## When to use it (and when not)
| Task | Role system | Example role |
|---|---|---|
| Create users, reset passwords | **Entra** | User Administrator, Helpdesk Administrator |
| Manage app registrations | **Entra** | Application Administrator |
| Assign licenses | **Entra** | License Administrator |
| Create and manage VMs | **Azure RBAC** | Virtual Machine Contributor |
| Read all resources in a subscription | **Azure RBAC** | Reader |
| Give others access to resources | **Azure RBAC** | Owner, User Access Administrator, Role Based Access Control Administrator |

## Key concepts

| | Entra roles | Azure RBAC roles |
|---|---|---|
| Manage | The **directory**: users, groups, passwords, apps, licenses | **Azure resources**: VMs, storage, networks |
| Examples | Global Administrator, User Administrator, Helpdesk Administrator | Owner, Contributor, Reader |
| Scope | Tenant (or **administrative unit**) | Management group, subscription, resource group, resource |
| Where in the portal | Microsoft Entra ID → **Roles and administrators** | Resource → **Access control (IAM)** |
| Custom roles | Yes (P1) | Yes |

### Lab example

| Person | Role | Can | Can't |
|---|---|---|---|
| **Sarah** | User Administrator (Entra) | Create users, reset passwords | Touch VMs |
| **Adam** | Contributor (RBAC) | Create and manage resources | Create users, give others access |

### Global Administrator and Azure
- A **Global Administrator is not automatically an Owner** of Azure subscriptions
- But a Global Administrator can turn on **Elevate access** (Entra ID → Properties → *Access management for Azure resources*)
- This gives them **User Access Administrator at the root scope (`/`)** – all subscriptions and management groups
- From there, they can give themselves any Azure role
- Turn it off again when finished

### Administrative units
Limit an Entra role to **part** of the directory. Example: a Helpdesk Administrator who can reset passwords only for users in the "Oslo" administrative unit.

## Exam traps
- "Needs to create users" → **Entra role** (User Administrator)
- "Needs to manage VMs" → **RBAC role** (Contributor / Virtual Machine Contributor)
- "Global admin can't see the subscription" → **Elevate access**
- **Contributor can't assign roles.** Assigning roles needs Owner, **User Access Administrator**, or **Role Based Access Control Administrator**
- **Application Administrator** is an Entra role – you **can't** assign it on a resource group
- Limit an Entra admin to some users → **administrative unit**, not a resource group

## Do it

### Portal
- **Entra role:** **Entra ID → Roles and administrators** → choose role → **+ Add assignments**
- **RBAC role:** resource/RG/subscription → **Access control (IAM) → + Add → Add role assignment** → role → members → Review + assign
- **Check what someone can do:** IAM → **Check access** → user → see role assignments (including inherited)
- **Elevate access:** Entra ID → **Properties** → Access management for Azure resources = **Yes**

### Azure CLI
```bash
# Azure RBAC
az role assignment create --assignee adam@contoso.onmicrosoft.com --role Contributor --resource-group rg-dev
az role assignment list --assignee adam@contoso.onmicrosoft.com --all --include-inherited -o table

# Elevate access as Global Admin (then sign out/in)
az rest --method post --url "/providers/Microsoft.Authorization/elevateAccess?api-version=2016-07-01"
```
Entra role assignments have no `az` command – use the portal or Microsoft Graph.

### PowerShell
```powershell
# Azure RBAC (Az module)
New-AzRoleAssignment -SignInName adam@contoso.onmicrosoft.com -RoleDefinitionName Contributor -ResourceGroupName rg-dev
Get-AzRoleAssignment -SignInName adam@contoso.onmicrosoft.com

# Entra role (Microsoft Graph)
Connect-MgGraph -Scopes "RoleManagement.ReadWrite.Directory"
$role = Get-MgRoleManagementDirectoryRoleDefinition -Filter "displayName eq 'User Administrator'"
$user = Get-MgUser -UserId sarah@contoso.onmicrosoft.com
New-MgRoleManagementDirectoryRoleAssignment -PrincipalId $user.Id -RoleDefinitionId $role.Id -DirectoryScopeId "/"
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Global Admin sees **no subscriptions** | Entra role ≠ Azure access | **Elevate access**, or have an Owner assign a role |
| Owner of the subscription can't create users | RBAC role ≠ directory access | Assign **User Administrator** (Entra) |
| Contributor can't add a role assignment (button greyed) | Contributor excludes `Microsoft.Authorization/*/write` | Owner / User Access Administrator / RBAC Administrator |
| Role not in the list when assigning at an RG | It's an **Entra** role | Assign it in Roles and administrators |
| New role doesn't work yet | Propagation (up to ~10 min), cached token | Wait, sign out/in, `az account clear` + login |

## Clean up
- Remove lab role assignments: IAM → **Role assignments** → remove
- Turn **Elevate access** off again

## Check yourself
<details><summary>1. A helpdesk user must reset passwords for staff in the Bergen office only.</summary>

**Helpdesk Administrator** (Entra) scoped to a **Bergen administrative unit**.
</details>

<details><summary>2. A Global Administrator must regain control of a subscription after the only Owner left.</summary>

**Elevate access** → User Access Administrator at root → assign themselves Owner.
</details>

<details><summary>3. Adam (Contributor on rg-dev) must let a colleague read the RG. Can he?</summary>

No – Contributor can't assign roles. He needs Owner, User Access Administrator, or RBAC Administrator.
</details>

## Related
- [Azure RBAC](../04-governance/azure-rbac.md)
- [Custom Roles](custom-roles.md)
- [PIM](pim.md) – handles both role types in separate sections
