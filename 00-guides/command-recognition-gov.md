# Azure Governance (AZ-104) — Appendix: Command Recognition

[← Back to README](../README.md)

> **Don't spend a lot of your study time here.**
> The exam may show you CLI or PowerShell snippets, but it tests whether you can *recognise* the right command or spot the missing parameter — not whether you can write scripts from memory.
> Budget about **one hour** for this page. Learn the patterns below; skip memorising every parameter. Your time is better spent on *why* and *when* (the study guide, sections 1–7).

---

### 1. The grammar — learn this, and most answer options decode themselves

| | Azure CLI | Azure PowerShell |
|---|---|---|
| **Pattern** | `az <noun> <verb>` | `Verb-AzNoun` |
| **Example** | `az lock create` | `New-AzResourceLock` |
| Create | `create` | `New-` |
| Read | `list` / `show` | `Get-` |
| Change | `update` / `set` | `Set-` / `Update-` |
| Delete | `delete` | `Remove-` |
| Parameter style | `--resource-group` (lowercase, dashes) | `-ResourceGroupName` (PascalCase) |

**Exam trick:** Distractors often mix the two styles — e.g. `az lock create -LockLevel`. If CLI syntax and PowerShell parameters appear in the same command, it's wrong.

---

### 2. The scope string — read it from left to right, top to bottom of the hierarchy

| Scope | Scope string |
|---|---|
| Tenant root | `/` |
| Management group | `/providers/Microsoft.Management/managementGroups/<mg-id>` |
| Subscription | `/subscriptions/<sub-id>` |
| Resource group | `/subscriptions/<sub-id>/resourceGroups/<rg-name>` |
| Resource | `/subscriptions/<sub-id>/resourceGroups/<rg-name>/providers/Microsoft.Storage/storageAccounts/<name>` |

If you can read a scope string, you can answer "at which level does this assignment apply?" — a very common question shape.

---

### 3. The commands worth recognising

| Task | Azure CLI | Azure PowerShell |
|---|---|---|
| Create a resource group | `az group create --name RG1 --location norwayeast` | `New-AzResourceGroup -Name RG1 -Location norwayeast` |
| Move resources | `az resource move --destination-group RG2 --ids <id>` | `Move-AzResource -DestinationResourceGroupName RG2 -ResourceId <id>` |
| Register a resource provider | `az provider register --namespace Microsoft.ContainerRegistry` | `Register-AzResourceProvider -ProviderNamespace Microsoft.ContainerRegistry` |
| Create a management group | `az account management-group create --name corp` | `New-AzManagementGroup -GroupName corp` |
| Move a subscription into an MG | `az account management-group subscription add --name corp --subscription <sub-id>` | `New-AzManagementGroupSubscription -GroupName corp -SubscriptionId <sub-id>` |
| Assign a role | `az role assignment create --assignee <user> --role "Reader" --scope <scope>` | `New-AzRoleAssignment -SignInName <user> -RoleDefinitionName "Reader" -Scope <scope>` |
| List role assignments | `az role assignment list --assignee <user> --include-inherited` | `Get-AzRoleAssignment -SignInName <user>` |
| Create a custom role | `az role definition create --role-definition @role.json` | `New-AzRoleDefinition -InputFile role.json` |
| Assign a policy | `az policy assignment create --name <n> --policy <definition> --scope <scope>` | `New-AzPolicyAssignment -Name <n> -PolicyDefinition $def -Scope <scope>` |
| Trigger a compliance scan | `az policy state trigger-scan --resource-group RG1` | `Start-AzPolicyComplianceScan -ResourceGroupName RG1` |
| Remediate existing resources | `az policy remediation create --name <n> --policy-assignment <id>` | `Start-AzPolicyRemediation -Name <n> -PolicyAssignmentId <id>` |
| Create a lock | `az lock create --name L1 --lock-type CanNotDelete --resource-group RG1` | `New-AzResourceLock -LockName L1 -LockLevel CanNotDelete -ResourceGroupName RG1` |
| Remove a lock | `az lock delete --name L1 --resource-group RG1` | `Remove-AzResourceLock -LockName L1 -ResourceGroupName RG1` |
| Add tags (keep existing ones) | `az tag update --resource-id <id> --operation Merge --tags env=dev` | `Update-AzTag -ResourceId <id> -Tag @{env="dev"} -Operation Merge` |

**Budgets** are usually configured in the portal (Cost Management → Budgets). Recognise that they exist in CLI/PowerShell; don't learn the syntax.

---

### 4. The traps — these are the parts actually worth remembering

1. **Tags: replace vs. merge.** Several commands **replace the entire tag set**, silently deleting existing tags:
   - `az group update --tags env=dev` → replaces all tags on the RG. (`--set tags.env=dev` adds/changes just one.)
   - `az resource tag --tags env=dev` → replaces, unless you add `--is-incremental`.
   - `New-AzTag -ResourceId <id>` / `Set-AzResource -Tag` → replace.
   - `Update-AzTag -Operation Merge` → adds without removing. (`Replace` and `Delete` are the other operations.)
2. **Lock names differ between portal and code.** Portal: *Delete* / *Read-only*. Code: `CanNotDelete` / `ReadOnly`.
3. **A role assignment always needs three things:** who (`--assignee` / `-SignInName` or `-ObjectId`), what (`--role` / `-RoleDefinitionName`) and where (`--scope` / `-Scope`). If an answer option is missing one, it's incomplete.
4. **Existing resources need remediation.** A Modify or DeployIfNotExists assignment alone does not fix what already exists — look for the `remediation` / `Start-AzPolicyRemediation` step.
5. **`MissingSubscriptionRegistration` error** → the answer is `az provider register` / `Register-AzResourceProvider`, not a role or a quota.

---

> **Sparring prompt for Claude:** *"Show me five governance CLI or PowerShell snippets with one thing wrong in each. Let me find the error before you tell me."*
