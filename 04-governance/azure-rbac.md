# Azure RBAC

[← Back to README](../README.md) · Labs 06–07

## In one line
**Azure role-based access control (RBAC)** decides **what someone can do with Azure resources**: a **role assignment** = **who** + **what role** + **where (scope)**.

## When to use it (and when not)
| Situation | Use |
|---|---|
| Let a team manage resources in one RG | RBAC role on the **resource group** |
| Let auditors view everything | **Reader** at subscription / management group |
| Let someone hand out access but not manage resources | **User Access Administrator** or **RBAC Administrator** |
| Read/write data inside storage, Key Vault, etc. | A **data plane** RBAC role |
| Create users or reset passwords | Not RBAC – an **Entra role** ([Entra roles vs RBAC](../02-identity/entra-roles-vs-rbac.md)) |
| Block a configuration for everyone | Not RBAC – **Azure Policy** |

## Key concepts

| Part | Question | Example |
|---|---|---|
| **Security principal** | Who? | User, group, service principal, managed identity |
| **Role definition** | What can they do? | Contributor |
| **Scope** | Where? | Resource group `rg-dev` |

### The basic roles

| Role | Create/modify resources | Assign access to others |
|---|---|---|
| **Reader** | No | No |
| **Contributor** | Yes | No |
| **Owner** | Yes | Yes |
| **User Access Administrator** | No | Yes (only manages access) |
| **Role Based Access Control Administrator** | No | Yes (can be limited with **conditions**, e.g. "may only assign Reader") |

### Control plane vs data plane

| | Control plane (management) | Data plane (the data) |
|---|---|---|
| Example | Create a storage account, change settings | Read or upload blobs |
| Roles | Owner, Contributor, Reader | **Storage Blob Data Owner/Contributor/Reader** |

**Even an Owner can't necessarily read blobs** with Entra authentication. A **data role** such as *Storage Blob Data Contributor* is needed. (An Owner or Contributor can list the account **keys**, though, and the keys give full access.)

### Scope and inheritance
```
Management group
└── Subscription
    └── Resource group
        └── Resource
```
- Access is **inherited downwards** ([Inheritance](inheritance.md))
- A **Contributor on the subscription** can change and delete **everything** in it
- Assign roles at the **lowest scope** that does the job (least privilege)
- Assign to **groups**, not individual users
- RBAC is **additive**: the user's permissions = the sum of all their assignments

### Resource locks
Locks protect resources **even from Owners**. Details: [Resource Locks](resource-locks.md).

### Key Vault – same idea as storage
When the Key Vault uses the **RBAC permission model**:

| Task | Role needed |
|---|---|
| Create the Key Vault | Contributor |
| Create and manage secrets | **Key Vault Secrets Officer** |
| Only read secrets (minimum privilege) | **Key Vault Secrets User** |

### Sign in to a VM with Entra ID

| Role | Gives |
|---|---|
| **Virtual Machine User Login** | Sign in as a regular user |
| **Virtual Machine Administrator Login** | Sign in as administrator |

The role alone is not enough. You also need:
- The **Entra ID login extension** on the VM (AADLoginForWindows / AADSSHLoginForLinux)
- **Network access**: an NSG rule for RDP **3389** (or SSH 22) and a public IP, Bastion, or a jump server

More roles: [Role cheat sheet](cheat-sheet.md).

## Exam traps
- **Contributor can't assign roles** – Owner, User Access Administrator, or RBAC Administrator can
- **Least privilege to assign roles** → **Role Based Access Control Administrator** (with a condition if limited to certain roles)
- Inheritance goes **down**, never up
- Locks beat RBAC – even an Owner can't delete a resource with a CanNotDelete lock
- RBAC changes can take **a few minutes** to apply (sign out/in to refresh the token)
- **Reader can't list storage keys** (`listKeys` is a POST action)
- **Virtual Machine Contributor** can manage the VM but **can't sign in** to it with Entra ID
- Max **4,000** role assignments per subscription → use **groups**

## Do it

### Portal
- Scope (MG / subscription / RG / resource) → **Access control (IAM) → + Add → Add role assignment** → **Role** tab (Job function / Privileged administrator roles) → **Members** → user, group, service principal, or managed identity → **Review + assign**
- **Check access** → see a user's effective assignments
- **Role assignments** tab → filter by scope ("This resource" / "Inherited")
- **Roles** tab → open a role → **Permissions / JSON**

### Azure CLI
```bash
# Assign
az role assignment create --assignee sarah@contoso.onmicrosoft.com --role Reader \
  --scope /subscriptions/<sub-id>
az role assignment create --assignee <group-object-id> --role Contributor --resource-group rg-dev

# Inspect
az role assignment list --assignee sarah@contoso.onmicrosoft.com --all --include-inherited -o table
az role assignment list --resource-group rg-dev -o table
az role definition list --name "Contributor" --query "[].{actions:permissions[0].actions, notActions:permissions[0].notActions}"

# Remove
az role assignment delete --assignee sarah@contoso.onmicrosoft.com --role Reader --scope /subscriptions/<sub-id>
```

### PowerShell
```powershell
New-AzRoleAssignment -SignInName sarah@contoso.onmicrosoft.com -RoleDefinitionName Reader -Scope "/subscriptions/<sub-id>"
New-AzRoleAssignment -ObjectId <group-object-id> -RoleDefinitionName Contributor -ResourceGroupName rg-dev

Get-AzRoleAssignment -SignInName sarah@contoso.onmicrosoft.com -ExpandPrincipalGroups | Format-Table Scope, RoleDefinitionName
Get-AzRoleDefinition Contributor | Select-Object -ExpandProperty NotActions

Remove-AzRoleAssignment -SignInName sarah@contoso.onmicrosoft.com -RoleDefinitionName Reader -Scope "/subscriptions/<sub-id>"
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| **AuthorizationFailed** "does not have authorization to perform action X over scope Y" | Missing role **at that scope** – the error tells you the action and scope | Assign a role that includes X at Y or above |
| Role assigned but still denied | Propagation (up to ~10 min), cached token | Wait, sign out/in, `az account clear && az login` |
| **AuthorizationPermissionMismatch** on storage data | Control plane role only | Add **Storage Blob Data** role |
| "Add role assignment" greyed out | You're Contributor | Owner / UAA / RBAC Administrator |
| Can't remove an assignment | It's **inherited** from a higher scope | Remove it at the parent scope |
| User can do more than expected | Access via a **group** or higher scope | **Check access**, `-ExpandPrincipalGroups` |
| Can't assign role: "principal not found" | Brand-new user/SP not replicated yet | Wait a minute, or use `--assignee-object-id` + `--assignee-principal-type` |
| VM Entra sign-in fails with the right role | Login extension missing or network blocked | Install extension, check NSG/Bastion |

**Who changed a role assignment?** → **Activity log** on the scope (filter "Create role assignment").

## Clean up
Remove all lab assignments (they don't cost money but clutter IAM and count toward the 4,000 limit).

## Check yourself
<details><summary>1. Least privilege for a team lead who must give other people Reader on rg-app – and nothing else.</summary>

**Role Based Access Control Administrator** on rg-app with a **condition** limiting it to Reader.
</details>

<details><summary>2. A user is Owner on a storage account but `az storage blob list --auth-mode login` fails. Fix?</summary>

Assign **Storage Blob Data Reader** (data plane).
</details>

<details><summary>3. An app's managed identity must read one secret in Key Vault (RBAC model). Which role?</summary>

**Key Vault Secrets User** – on the vault or on that secret.
</details>

## Related
- [Role cheat sheet](cheat-sheet.md)
- [Custom Roles](../02-identity/custom-roles.md)
- [Entra Roles vs Azure RBAC](../02-identity/entra-roles-vs-rbac.md)
- [Managed Identities](../02-identity/managed-identities.md)
- [Inheritance](inheritance.md)
