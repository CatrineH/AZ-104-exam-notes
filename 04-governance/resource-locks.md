# Resource Locks

[← Back to README](../README.md) · Governance

## In one line
A **resource lock** protects resources from being **deleted or changed by accident** – and it applies to **everyone**, even Owners.

## When to use it (and when not)
| Situation | Use |
|---|---|
| Production database / storage / VNet must never be deleted by mistake | **CanNotDelete** |
| Something must be completely **frozen** (rare) | **ReadOnly** – test first! |
| Stop the **wrong people** from changing things | **RBAC**, not locks |
| Block deletes across many subscriptions centrally | **Policy DenyAction** or locks at subscription level |
| Protect the **data inside** (files, blobs) | **Soft delete, versioning, immutable storage** – locks don't protect data |

## Key concepts
- Can be set on a **subscription**, **resource group**, or **resource**
- **Inherited downwards** – a lock on a resource group protects every resource in it

| Lock | Read | Modify | Delete |
|---|---|---|---|
| **CanNotDelete** (Delete in the portal) | Yes | Yes | No |
| **ReadOnly** (Read-only in the portal) | Yes | No | No |

- When several locks apply, the **most restrictive lock wins**
- To delete or change a locked resource, **remove the lock first**

### Who can create and remove locks?
You need `Microsoft.Authorization/locks/*`:
- **Owner** – yes
- **User Access Administrator** – yes
- **Contributor** – **no**, can't create or remove locks

### ReadOnly has side effects
Some operations **look like reads** but are technically **POST requests** – and ReadOnly **blocks them**.

| Resource | What ReadOnly breaks |
|---|---|
| **Storage accounts** | **Listing account keys** is blocked – so Storage Explorer, the portal's data view with keys, and scripts using keys stop working |
| **VMs** | **Start, stop, and restart** are blocked |
| **App Service** | Scaling and some management tools stop working |
| **Resource groups** | Can't **add** new resources. ARM can't clean up old deployment history |

**Use CanNotDelete for production. Use ReadOnly only when you truly want it frozen – and test first.**

### Locks protect the control plane, not the data
- A lock on a storage account stops the **account** from being deleted
- It does **not** stop someone with a data role from **deleting blobs or files** inside it
- Protect data with **soft delete**, **versioning**, or **immutable storage**

### Locks vs RBAC vs Policy

| Tool | Protects against |
|---|---|
| **RBAC** | The **wrong people** making changes |
| **Locks** | **Accidental** delete or change – even by the right people |
| **Policy (DenyAction)** | Delete actions across **many resources**, set centrally |

## Exam traps
- Locks apply **even to Owners**
- **Most restrictive** lock wins
- Contributor **can't** create or remove a lock
- ReadOnly on a storage account **blocks listing keys**
- ReadOnly on a VM **blocks start/restart**
- ReadOnly on a **resource group** → you can't create new resources in it
- CanNotDelete on an RG → you can't delete the RG **or any resource** in it, but you **can** add resources
- Deleting a resource group with a locked resource inside → the **whole delete fails**
- Locks don't protect **data** (blobs/files)

## Do it

### Portal
**Resource / RG / subscription → Settings → Locks → + Add** → name, lock type (Delete / Read-only), notes → OK.

### Azure CLI
```bash
# Lock a resource group
az lock create --name lock-no-delete --lock-type CanNotDelete \
  --resource-group AZ104-catrine-rg --notes "Production – do not delete"

# Lock one resource
az lock create --name lock-readonly --lock-type ReadOnly \
  --resource-group AZ104-catrine-rg \
  --resource-name az104catrinesta \
  --resource-type Microsoft.Storage/storageAccounts

# Subscription-level lock
az lock create --name lock-sub --lock-type CanNotDelete

# List and delete
az lock list --resource-group AZ104-catrine-rg -o table
az lock delete --name lock-no-delete --resource-group AZ104-catrine-rg
```

### PowerShell
```powershell
New-AzResourceLock -LockName "lock-no-delete" -LockLevel CanNotDelete `
  -ResourceGroupName AZ104-catrine-rg -LockNotes "Production – do not delete" -Force

New-AzResourceLock -LockName "lock-readonly" -LockLevel ReadOnly `
  -ResourceGroupName AZ104-catrine-rg -ResourceName az104catrinesta `
  -ResourceType Microsoft.Storage/storageAccounts -Force

Get-AzResourceLock -ResourceGroupName AZ104-catrine-rg
Remove-AzResourceLock -LockName "lock-no-delete" -ResourceGroupName AZ104-catrine-rg -Force
```

## Troubleshooting

| Symptom / error | Likely cause | Fix |
|---|---|---|
| **ScopeLocked** "…cannot perform delete operation because following scope(s) are locked" | A lock on the resource, RG, or subscription | Error shows the scope → remove that lock (Owner/UAA) |
| Can't delete the RG | A resource inside (or the RG) has a lock | Find and remove the lock(s) |
| Storage Explorer / script with keys suddenly fails | **ReadOnly** lock → listKeys blocked | Use CanNotDelete, or Entra ID auth |
| Can't start a VM | **ReadOnly** lock on VM or its RG | Remove/relax lock |
| Can't add a resource to an RG | **ReadOnly** lock on the RG | Remove it |
| Contributor can't remove the lock | Contributor lacks `locks/*` | Owner / User Access Administrator |
| Lock exists but blobs got deleted | Locks don't protect data | Soft delete / versioning / immutability |

## Clean up
Always remove lab locks **before** `az group delete`, or the delete fails:
```bash
az lock list -g AZ104-catrine-rg --query "[].id" -o tsv | xargs -r -n1 az lock delete --ids
az group delete -n AZ104-catrine-rg --yes --no-wait
```

## Check yourself
<details><summary>1. Prevent accidental deletion of a production RG while still letting admins update resources.</summary>

**CanNotDelete** lock on the RG.
</details>

<details><summary>2. After adding a ReadOnly lock to a storage account, an app using the account key fails. Why?</summary>

ReadOnly blocks **listKeys** (a POST action).
</details>

<details><summary>3. A Contributor must remove a lock. Can they?</summary>

No – needs **Owner** or **User Access Administrator**.
</details>

## Related
- [Inheritance](inheritance.md)
- [Azure Policy](azure-policy.md)
- [Azure RBAC](azure-rbac.md)
- [Blob Tiers and Lifecycle](../03-storage/blob-tiers-lifecycle.md) – data protection
