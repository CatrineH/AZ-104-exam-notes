# Scope Hierarchy

[← Back to README](../README.md) · Governance

## In one line
Azure is organised in **four levels** – management groups → subscriptions → resource groups → resources – and everything you assign at a level (RBAC, policy, locks, budgets) applies to what's **below** it.

## When to use it (and when not)
| Need | Level |
|---|---|
| Same rule for many subscriptions (company-wide) | **Management group** |
| Separate billing, quotas, or a hard security boundary | **Subscription** |
| Group resources with the same **lifecycle** (deployed and deleted together) | **Resource group** |
| One-off exception for a single resource | **Resource** (or a policy exemption) |
| Manage **customers'** tenants as a service provider | **Azure Lighthouse** |

"Do this with the **least administrative effort**" → assign it **once**, at the **highest scope** that covers everything needed.

## Key concepts

```
Tenant root group
└── Management groups       e.g. Platform, Landing zones, Sandbox
    └── Subscriptions       e.g. sub-prod, sub-dev
        └── Resource groups e.g. rg-webshop-prod
            └── Resources   e.g. VMs, storage accounts, VNets
```

### Management groups
Group **subscriptions** so you can manage access and policy for many at once.
- The **tenant root group** is created automatically. Its ID is the **tenant ID**. It can't be moved or deleted
- Up to **6 levels** below the root (not counting the root or subscriptions)
- Up to **10 000** management groups per tenant
- Each management group and subscription has **exactly one parent**
- New subscriptions are placed under the **root** by default – this can be changed in **hierarchy settings**
- A **Global Administrator** must **elevate access** to manage the root group the first time

**Moving a subscription to another management group** needs:
- **Owner** (or write access) on the **subscription**
- **Management Group Contributor** on **both** the current and the new parent

Policies and roles from the **new** parent apply right away.

### Subscriptions – five boundaries

| Boundary | What it means |
|---|---|
| **Billing** | Costs are invoiced and reported **per subscription** |
| **Limits and quotas** | For example **vCPU quotas per region** are set per subscription |
| **Trust** | A subscription trusts **exactly one** Entra tenant. A tenant can have **many** subscriptions |
| **Network** | A VNet lives in **one subscription and one region**. Peering **can cross** subscriptions |
| **Access and policy** | A common **scope** for RBAC and policy assignments |

- Linked to a **billing account** – pay-as-you-go (credit card), Enterprise Agreement, or Microsoft Customer Agreement
- Moving a subscription to **another tenant** **removes all RBAC assignments** – they must be recreated

### Resource groups
- Logical containers for resources that share a **lifecycle**
- A resource belongs to **exactly one** resource group
- Resource groups **can't be nested**
- The resource group's **location** only stores its **metadata** – the resources inside can be in other regions
- **Deleting a resource group deletes everything in it**

### Moving resources

| Move | Possible? | Notes |
|---|---|---|
| To another **resource group** | Yes, most resources | Same subscription |
| To another **subscription** | Yes, most resources | Both subscriptions in the **same tenant** |
| To another **region** | Not with a normal move | Use **Azure Resource Mover**, or redeploy |

During a move:
- **Both resource groups are locked** for writes and deletes – the resources keep running
- The **resource ID changes** – update scripts and references
- The **location doesn't change**
- Dependent resources must be moved **together**, e.g. a VM with its disks and NIC
- The resource provider must be **registered** in the target subscription
- Check first with the portal's **validation** step

### Azure Lighthouse
Manage **other tenants'** resources from **your own** tenant.
- Built for **service providers (MSPs)** – for example a company like Orange Business managing customers' Azure
- The customer **delegates** subscriptions or resource groups to the provider's users or groups, with specific **RBAC roles**
- The provider's staff sign in to **their own tenant** – no guest accounts in each customer tenant
- Customers can see and **remove** the delegation at any time

Landing zones: see [Why Governance](why-governance.md#landing-zones).

## Exam traps
- **Least administrative effort** → the **highest scope** that covers what's needed
- Management groups: **6 levels**, **one parent**, root can't be deleted
- A subscription trusts **one tenant**. Moving it to another tenant **drops RBAC assignments**
- Moving resources: the **resource ID changes**, the **location doesn't**
- Moving across **regions** = Resource Mover / redeploy, not "Move"
- A VM moves with its **disks and NIC** – dependent resources together
- Some resources **can't** be moved (check the "move support" table) – the validation step tells you
- Manage customers' tenants without guest accounts → **Azure Lighthouse**

## Do it

### Portal
- **Management groups:** search **Management groups → + Create** → ID + display name → open it → **+ Add subscription**
- **Hierarchy settings:** default management group for new subscriptions
- **Move resources:** resource group → select resources → **Move → to another resource group / subscription / region** → validation runs first

### Azure CLI
```bash
# Management groups
az account management-group create --name mg-contoso --display-name "Contoso"
az account management-group create --name mg-sandbox --display-name "Sandbox" --parent mg-contoso
az account management-group subscription add --name mg-sandbox --subscription <sub-id>
az account management-group show --name mg-contoso --expand --recurse

# Resource groups
az group create -n rg-webshop-prod -l norwayeast
az group list -o table

# Move a resource to another RG (or subscription with --destination-subscription-id)
az resource move --destination-group rg-new \
  --ids /subscriptions/<id>/resourceGroups/rg-old/providers/Microsoft.Storage/storageAccounts/az104catrinesta

# Register a resource provider in the target subscription
az provider register --namespace Microsoft.Compute
```

### PowerShell
```powershell
New-AzManagementGroup -GroupName mg-contoso -DisplayName "Contoso"
New-AzManagementGroup -GroupName mg-sandbox -DisplayName "Sandbox" -ParentId "/providers/Microsoft.Management/managementGroups/mg-contoso"
New-AzManagementGroupSubscription -GroupName mg-sandbox -SubscriptionId <sub-id>

$res = Get-AzResource -ResourceGroupName rg-old -Name az104catrinesta
Move-AzResource -DestinationResourceGroupName rg-new -ResourceId $res.ResourceId
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Can't see the tenant root group | Haven't **elevated access** | Global Admin → Entra ID → Properties → Access management for Azure resources |
| Can't move subscription to an MG | Missing Management Group Contributor on old or new parent, or not Owner of subscription | Get both roles |
| Move fails: "resource type not supported" | That type can't be moved | Redeploy, or check move support docs |
| Move fails: dependent resources | VM moved without disks/NIC, or related resources in other RGs | Select all related resources together |
| Move fails: provider not registered | Target subscription lacks the provider | `az provider register` |
| Move fails: `ScopeLocked` | A lock on source or target RG | Remove the lock temporarily |
| Scripts break after a move | **Resource ID changed** | Update IDs |
| New subscription doesn't get company policies | Landed under the **root**, not the right MG | Move it, or set default MG in hierarchy settings |

## Clean up
```bash
az account management-group subscription remove --name mg-sandbox --subscription <sub-id>
az account management-group delete --name mg-sandbox
az account management-group delete --name mg-contoso
```
A management group must be **empty** (no child MGs or subscriptions) before you can delete it.

## Check yourself
<details><summary>1. All 30 subscriptions must only allow Norway East. Least effort?</summary>

Assign the policy once at the **management group** above them (or the root).
</details>

<details><summary>2. You move a storage account to another subscription. What changes: location, resource ID, or both?</summary>

Only the **resource ID**.
</details>

<details><summary>3. A VM must move from West Europe to Norway East. Can you use "Move to another resource group"?</summary>

No – region moves need **Azure Resource Mover** or redeploy.
</details>

## Related
- [Why Governance](why-governance.md)
- [Inheritance](inheritance.md)
- [Azure Policy](azure-policy.md)
- [Azure RBAC](azure-rbac.md)
