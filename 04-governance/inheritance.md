# Inheritance

[← Back to README](../README.md) · Governance 02

## In one line
What you assign at a scope flows **down** to everything below it – but each tool inherits differently, and **tags and budgets don't inherit at all**.

```
Management group  ──►  Subscription  ──►  Resource group  ──►  Resource
```

## When to use it (and when not)
| Want | Assign at |
|---|---|
| Rules for the whole company: allowed regions, security baseline, required tags | **Root / management groups** |
| Rules for one environment or workload | **Subscriptions and resource groups** |
| An exception for one RG or resource | **Exclusion / exemption** on the policy assignment |
| Tags on every resource | Not inheritance – a **Modify policy** |

## Key concepts

### How each tool inherits

| Tool | Inherited downwards? | Can a lower scope weaken it? |
|---|---|---|
| **RBAC** | Yes | No. Lower assignments can only **add** permissions |
| **Azure Policy** | Yes | No. Only **exclusions** or **exemptions**, set by someone with rights at the assignment scope |
| **Resource locks** | Yes | No. The **most restrictive** lock in the chain wins |
| **Tags** | **No** | – Use a **Modify policy** to inherit or copy tags from the resource group or subscription |
| **Budgets** | No | – They **roll up**: a subscription budget measures **everything below it** |

**Examples:**
- **RBAC:** Reader on the subscription + Contributor on `rg-dev` → Contributor in `rg-dev`, Reader everywhere else
- **Policy:** "Allowed locations: Norway East" on a management group → no subscription below it can create resources in West Europe
- **Locks:** CanNotDelete on a resource group + ReadOnly on one VM inside it → the VM is **ReadOnly**
- **Tags:** `environment=prod` on a resource group → resources inside it **don't** get the tag automatically
- **Budgets:** a 10 000 kr budget on `sub-dev` counts the costs of **all** resource groups in it

### Control plane vs data plane

| | Control plane | Data plane |
|---|---|---|
| Manages | The **resource** – e.g. the storage account | The **data** – e.g. files and blobs in a container |
| Roles | Owner, Contributor, Reader | Storage Blob Data Owner/Contributor/Reader |
| Deleting a single file | No – needs a data role | Yes |

A control plane **Owner or Contributor can delete the whole storage account** – and all the data with it. That's why production storage often gets a **CanNotDelete lock**.

### Govern from above
**Assign once, high up:**
- **Consistent** – every subscription below follows the same rules
- **Low effort** – one assignment instead of one per subscription
- **Out of the owner's reach** – a **subscription Owner can't remove** a policy or role assigned at a management group, because they have no rights at that scope

**But not too high:**
- **Big blast radius** – a bad **Deny policy at the root breaks every subscription**
- Test new policies with the **Audit** effect, or on a **Sandbox** management group, before assigning them high up

## Exam traps
- Tags are the classic trap – they are **not inherited**. The answer is usually the built-in policy **"Inherit a tag from the resource group if missing"** (Modify)
- RBAC is **additive** – you can't take away an inherited role with a lower assignment (only **deny assignments**, which you can't create yourself)
- **Most restrictive lock wins**
- A subscription Owner **can't** remove an MG-level policy or role
- Inheritance goes **down**, never up: Contributor on an RG gives nothing on the subscription

## Do it – see inheritance with your own eyes
1. Give a test user **Reader** on the subscription and **Contributor** on `rg-dev`
2. Resource in `rg-dev` → **Access control (IAM) → Role assignments** → the Reader row shows **Scope: "Subscription (Inherited)"**
3. **IAM → Check access** → choose the user → see both assignments
4. Put a tag on `rg-dev`, create a storage account inside → no tag → assign the inherit-tag policy → create another → tag is there

### Azure CLI
```bash
az role assignment list --assignee sarah@contoso.onmicrosoft.com --all --include-inherited -o table
az lock list --resource-group rg-dev -o table          # locks on the RG and its resources
az lock list -o table                                   # every lock in the subscription (incl. subscription-level)
```

### PowerShell
```powershell
Get-AzRoleAssignment -SignInName sarah@contoso.onmicrosoft.com -ExpandPrincipalGroups
Get-AzResourceLock -ResourceGroupName rg-dev
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| User has more access than expected | Inherited from a higher scope or from a **group** | IAM → Check access → look at **Scope** column / group memberships |
| You can't remove a role assignment on an RG | It's **inherited** – it lives at a higher scope | Remove it at the scope shown |
| Resource can't be modified, no lock on it | Lock on the **RG or subscription** | Check locks at each level |
| New resources don't get the RG's tags | Tags don't inherit | Inherit-tag Modify policy |
| Policy blocks you, but it's not in the RG's assignments | Assigned at **subscription/MG** | Policy → Assignments → change scope filter |

## Check yourself
<details><summary>1. Reader at subscription, Contributor on rg-app. What can the user do in rg-db?</summary>

Only **read** – Contributor applies only to rg-app.
</details>

<details><summary>2. CanNotDelete on the RG, ReadOnly on a VM inside. Can you restart the VM?</summary>

No – ReadOnly (most restrictive) blocks restart.
</details>

<details><summary>3. The RG has tag `costCenter=1234`. Do new resources inside it get the tag?</summary>

No – use the **Inherit a tag from the resource group** policy.
</details>

## Related
- [Why Governance](why-governance.md)
- [Azure RBAC](azure-rbac.md)
- [Azure Policy](azure-policy.md)
- [Resource Locks](resource-locks.md)
- [Tags](tags.md)
