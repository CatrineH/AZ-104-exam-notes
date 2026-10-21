# Azure Storage – Overview

[← Back to README](../README.md)

## In one line
A **storage account** is the box that holds all Azure Storage services (Blob, Files, Queue, Table). Redundancy, networking, and keys are set on the box and apply to everything inside it.

## When to use it (and when not)
| Use a storage account for | Use something else for |
|---|---|
| Files, images, backups, logs (Blob) | VM disks → **managed disks** (separate resources) |
| Shared drives for VMs or on-premises (Files) | Relational data → Azure SQL |
| Simple messages between app parts (Queue) | Advanced messaging (topics, ordering) → Service Bus |
| Simple key-value data (Table) | Global NoSQL with low latency → Cosmos DB |

Storage belongs to the exam domain **"Implement and manage storage"** (about **15–20%** of AZ-104).

## Key concepts

### The storage account
- The name must be **globally unique**, **3–24 characters**, **lowercase letters and numbers only**
- The name becomes part of every endpoint, for example `https://az104catrinesta.blob.core.windows.net`
- Redundancy, network rules, and access keys are set **on the account** and apply to all services in it

### Services in a storage account

| Service | Stores | Endpoint | Notes |
|---|---|---|---|
| **Blob** | Objects: files, images, videos, backups | `<account>.blob.core.windows.net` | [blob-tiers-lifecycle.md](blob-tiers-lifecycle.md) |
| **Files** | File shares (SMB/NFS) | `<account>.file.core.windows.net` | [azure-files.md](azure-files.md) |
| **Queue** | Messages between app components | `<account>.queue.core.windows.net` | |
| **Table** | NoSQL key-value data | `<account>.table.core.windows.net` | |

> **Managed disks** for VMs are also Azure Storage, but they're separate resources – not part of a storage account.

### Account types

| Type | Supports | Use for |
|---|---|---|
| **Standard general-purpose v2** | Blob, Files, Queue, Table | **The default** for almost everything |
| **Premium block blobs** | Blob only (SSD) | Low latency, many small transactions |
| **Premium file shares** (FileStorage) | Files only (SSD) | High-performance file shares, NFS |
| **Premium page blobs** | Page blobs only (SSD) | Special cases, like unmanaged VM disks |

### Three themes that come back again and again

| Theme | Remember |
|---|---|
| **Who can access the data?** | Account keys = full access, SAS = limited and time-bound, Entra ID + data roles = preferred |
| **From which network?** | Storage firewall, service endpoints, private endpoints – on top of authorization |
| **How safe is the data?** | Redundancy protects against hardware, zone, or region failure. Soft delete, versioning, and snapshots protect against mistakes |

## Exam traps
- **You can't change the account type** (Standard ↔ Premium) after creation. You must create a new account and copy the data (AzCopy)
- **You can't rename** a storage account
- Settings like redundancy, firewall, and keys are **per account**, not per container. If two containers need different redundancy → two accounts
- **Premium** accounts don't support geo-redundancy (GRS/GZRS)
- Uppercase letters or hyphens in the name → the deployment fails
- **Owner/Contributor** manage the account (control plane) but do **not** automatically read blob data with Entra ID sign-in. That needs a **data role** (Storage Blob Data Reader/Contributor)

## Do it

### Portal
**Storage accounts → + Create**
1. **Basics:** subscription, resource group, name, region, **Performance** (Standard/Premium), **Redundancy**
2. **Advanced:** require secure transfer (HTTPS), minimum TLS 1.2, allow/deny anonymous blob access, enable/disable account key access, default access tier (Hot/Cool/Cold)
3. **Networking:** public access from all networks / selected networks / disabled
4. **Data protection:** soft delete for blobs, containers, and file shares, versioning
5. **Review + create**

### Azure CLI
```bash
RG=AZ104-catrine-rg
SA=az104catrinesta

az group create --name $RG --location norwayeast

az storage account create \
  --name $SA \
  --resource-group $RG \
  --location norwayeast \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false

# Check the name before you create it
az storage account check-name --name $SA

# See the account and its endpoints
az storage account show -n $SA -g $RG --query "{sku:sku.name, kind:kind, endpoints:primaryEndpoints}"
```

### PowerShell
```powershell
$rg = "AZ104-catrine-rg"
$sa = "az104catrinesta"

New-AzResourceGroup -Name $rg -Location norwayeast

New-AzStorageAccount -ResourceGroupName $rg -Name $sa `
  -Location norwayeast -SkuName Standard_LRS -Kind StorageV2 `
  -AccessTier Hot -MinimumTlsVersion TLS1_2 -AllowBlobPublicAccess $false

Get-AzStorageAccountNameAvailability -Name $sa
Get-AzStorageAccount -ResourceGroupName $rg -Name $sa | Select-Object StorageAccountName, Sku, Kind, PrimaryEndpoints
```

**Recognize the pattern:** CLI = `az storage account <verb>`, PowerShell = `<Verb>-AzStorageAccount`.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "The storage account name is already taken" | Names are global across **all** of Azure | Add numbers or initials, check with `check-name` |
| "...is not a valid storage account name" | Uppercase, hyphen, or wrong length | Lowercase letters and numbers, 3–24 characters |
| "You do not have permissions to list the data using your user account" in the portal | You have Owner/Contributor but no **data role** | Assign **Storage Blob Data Reader/Contributor**, or switch the portal to "Access key" auth |
| "This request is not authorized to perform this operation" | The **storage firewall** blocks your network | Add your client IP or connect from an allowed subnet. See [Storage Network Security](storage-network-security.md) |
| Need Premium on an existing Standard account | Account type is fixed | New Premium account + copy with [AzCopy](azcopy-storage-explorer.md) |

## Clean up
```bash
az storage account delete -n $SA -g $RG --yes
# or the whole lab:
az group delete -n $RG --yes --no-wait
```
```powershell
Remove-AzStorageAccount -ResourceGroupName $rg -Name $sa -Force
```

## My storage notes

| Topic | Notes | Covers |
|---|---|---|
| Redundancy | [storage-redundancy.md](storage-redundancy.md) | LRS, ZRS, GRS, RA-GRS, GZRS, RA-GZRS, failover |
| Blob tiers and lifecycle | [blob-tiers-lifecycle.md](blob-tiers-lifecycle.md) | Hot/Cool/Cold/Archive, lifecycle rules, soft delete, versioning |
| Access and authorization | [storage-access.md](storage-access.md) | Account keys, SAS, Entra ID data roles, Storage Explorer, PowerShell |
| Network security | [storage-network-security.md](storage-network-security.md) | Public network access, service endpoints, private endpoints, private DNS zones |
| Encryption | [storage-encryption-key-vault.md](storage-encryption-key-vault.md) | Microsoft-managed vs customer-managed keys, Key Vault |
| Azure Files | [azure-files.md](azure-files.md) | SMB/NFS, tiers, authentication, snapshots |
| Azure File Sync | [azure-file-sync.md](azure-file-sync.md) | Storage Sync Service, sync groups, cloud tiering |
| Moving data | [azcopy-storage-explorer.md](azcopy-storage-explorer.md) | AzCopy, Storage Explorer, Import/Export, Data Box |

| Lab | Covers |
|---|---|
| [Storage Account with Private Access](lab/storage-account-with-private-access.md) | Jump server, Azure CLI and PowerShell on VMs, service endpoint, private endpoint, DNS |
| [Storage Encryption with Customer-Managed Keys](lab/storage-cmk-lab.md) | Key Vault, managed identity, CMK, key rotation |

<!-- Add new storage topics and labs here -->

## Check yourself
<details><summary>1. You need a file share on SSD with NFS support. You already have a Standard GPv2 account. What do you do?</summary>

Create a new **Premium file shares (FileStorage)** account. Account type can't be changed, and NFS needs Premium.
</details>

<details><summary>2. A user is Owner on the storage account but gets an error listing blobs in the portal. Why?</summary>

Owner is a **control plane** role. Reading data with Entra ID needs a **data plane** role like Storage Blob Data Reader.
</details>

<details><summary>3. Which name is valid: `AZ104-Storage`, `az104storage01`, `az`?</summary>

`az104storage01`. The others have uppercase/hyphen or are too short.
</details>

## Related
- [Managed Identities](../02-identity/managed-identities.md) – passwordless access to storage
- [Azure RBAC](../04-governance/azure-rbac.md) – control plane vs data plane
- [DNS](../01-networking/dns.md)
