# Blob Storage – Access Tiers, Lifecycle Rules, and Data Protection

[← Back to README](../README.md)

## In one line
Pay less for data you rarely use by putting it in a **colder tier**, and let **lifecycle rules** move or delete blobs automatically based on age.

## When to use it (and when not)
| Situation | Use |
|---|---|
| Data used every day | **Hot** |
| Data used now and then, kept at least 30 days (older backups, short-term archive) | **Cool** |
| Rarely read, kept at least 90 days | **Cold** |
| Almost never read, kept 180+ days, can wait hours to read it (compliance archive) | **Archive** |
| "Move logs to Cool after 30 days and delete after a year" | **Lifecycle management rule** |
| Someone deleted or overwrote a blob by mistake | **Soft delete / versioning** |
| Data must not be changed or deleted for X years (legal) | **Immutable storage (WORM)** |

Don't use Archive for anything you might need quickly – it's **offline** and takes hours to read.

## Key concepts

### Containers and blob types
- Blobs live in **containers** (like folders at the top level). Container names: lowercase, 3–63 characters
- **Block blobs:** normal files – documents, images, backups. The only type that supports access tiers
- **Append blobs:** you can only add to the end – good for logs
- **Page blobs:** random read/write in 512-byte pages – VM disks

### Access tiers

| Tier | Storage cost | Read cost | Minimum days | Online? |
|---|---|---|---|---|
| **Hot** | Highest | Lowest | – | Yes |
| **Cool** | Lower | Higher | **30** | Yes |
| **Cold** | Even lower | Even higher | **90** | Yes |
| **Archive** | Lowest | Highest | **180** | **No – offline** |

- The **account default tier** can be Hot, Cool, or Cold (never Archive). New blobs get this tier unless you set another one
- You can set the tier **per blob**
- **Early deletion fee:** delete or move a blob before the minimum days → you pay for the remaining days

### Archive and rehydration
An archived blob **can't be read**. You must **rehydrate** it first:

| Method | How |
|---|---|
| **Change the tier** to Hot/Cool/Cold | The blob itself is rehydrated |
| **Copy** it to a new blob in an online tier | The archived original stays |

| Priority | Time |
|---|---|
| **Standard** | Up to **15 hours** |
| **High** | Usually **under 1 hour** for blobs under 10 GB (costs more) |

### Lifecycle management rules
Set on the **storage account** (**Data management → Lifecycle management**). A rule has:
- **Filters:** blob type (block/append) and **prefix** (for example `logs/`), or blob index tags
- **Actions:** `tierToCool`, `tierToCold`, `tierToArchive`, `delete` – for base blobs, snapshots, and versions
- **Conditions:** days since **last modified**, **created**, or **last accessed** (needs last access time tracking turned on)

- Rules run **about once a day**. New or changed rules can take **up to 24 hours** before they act
- Rules are **JSON** under the hood – the portal "Code view" shows it

### Data protection for blobs

| Feature | Protects against | Notes |
|---|---|---|
| **Blob soft delete** | Deleted blobs | Keep deleted blobs 1–365 days, then **Undelete** |
| **Container soft delete** | Deleted containers | Restore the whole container |
| **Versioning** | Overwrites and deletes | Every change keeps the previous version automatically |
| **Snapshots** | Changes | Manual, read-only copy of a blob at a point in time |
| **Point-in-time restore** | Big mistakes in block blobs | Restore containers to an earlier time. Needs **versioning + change feed + soft delete** |
| **Immutable storage (WORM)** | Changing or deleting at all | **Time-based retention** or **legal hold**. A **locked** policy can't be removed, not even by Owner |
| **Resource lock** | Deleting the **account** | Doesn't protect the data inside. See [Resource Locks](../04-governance/resource-locks.md) |

### Object replication
Copies block blobs **asynchronously** from a source container to a destination container (can be another region or account). Needs **versioning on both** and **change feed on the source**.

## Exam traps
- Tiers apply to **block blobs** only
- **Archive is offline.** "Read the file now" + archive → wrong answer. Rehydrate first (up to 15 hours on Standard)
- Archive is **not supported** on ZRS, GZRS, or RA-GZRS accounts
- **Account default tier** can be Hot, Cool, or Cold – **never Archive**
- Moving Cool → Hot after 10 days still costs the **early deletion fee** for the remaining 20 days
- Lifecycle rules can take **24 hours** to start. "Why didn't my rule run yet?" → wait
- "Last accessed" conditions need **access time tracking** enabled first
- **Soft delete ≠ backup.** If the whole **storage account** is deleted, blob soft delete doesn't help (a delete lock does)
- **Point-in-time restore** needs versioning, change feed, **and** soft delete

## Do it

### Portal
- **Container:** Storage account → **Data storage → Containers → + Container** (keep anonymous access **Private**)
- **Change a blob's tier:** open the blob → **Change tier**
- **Lifecycle rule:** **Data management → Lifecycle management → Add a rule** → filter by prefix → choose "Move to cool storage" after X days → "Delete the blob" after Y days
- **Data protection:** **Data management → Data protection** → soft delete for blobs/containers, versioning, point-in-time restore

### Azure CLI
```bash
# Create a container and upload a file (Entra ID auth)
az storage container create --account-name $SA --name logs --auth-mode login
az storage blob upload --account-name $SA --container-name logs \
  --name app/2026-10-01.log --file ./app.log --tier Cool --auth-mode login

# Change one blob's tier
az storage blob set-tier --account-name $SA --container-name logs \
  --name app/2026-10-01.log --tier Archive --auth-mode login

# Rehydrate with high priority
az storage blob set-tier --account-name $SA --container-name logs \
  --name app/2026-10-01.log --tier Hot --rehydrate-priority High --auth-mode login

# Turn on soft delete and versioning
az storage account blob-service-properties update -n $SA -g $RG \
  --enable-delete-retention true --delete-retention-days 14 \
  --enable-container-delete-retention true --container-delete-retention-days 7 \
  --enable-versioning true
```

**Lifecycle rule** – save as `policy.json`:
```json
{
  "rules": [
    {
      "enabled": true,
      "name": "move-old-logs",
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": [ "blockBlob" ], "prefixMatch": [ "logs/" ] },
        "actions": {
          "baseBlob": {
            "tierToCool":    { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete":        { "daysAfterModificationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
```
```bash
az storage account management-policy create --account-name $SA -g $RG --policy @policy.json
az storage account management-policy show --account-name $SA -g $RG
```

> `prefixMatch` starts with the **container name**: `logs/` = everything in the `logs` container.

### PowerShell
```powershell
$ctx = New-AzStorageContext -StorageAccountName $sa -UseConnectedAccount

New-AzStorageContainer -Name logs -Context $ctx
Set-AzStorageBlobContent -File .\app.log -Container logs -Blob "app/2026-10-01.log" `
  -StandardBlobTier Cool -Context $ctx

# Change tier
$blob = Get-AzStorageBlob -Container logs -Blob "app/2026-10-01.log" -Context $ctx
$blob.BlobClient.SetAccessTier("Archive", $null)

# Soft delete
Enable-AzStorageBlobDeleteRetentionPolicy -ResourceGroupName $rg -StorageAccountName $sa -RetentionDays 14
Update-AzStorageBlobServiceProperty -ResourceGroupName $rg -StorageAccountName $sa -IsVersioningEnabled $true

# Lifecycle rule
$action = Add-AzStorageAccountManagementPolicyAction -BaseBlobAction TierToCool -DaysAfterModificationGreaterThan 30
$action = Add-AzStorageAccountManagementPolicyAction -InputObject $action -BaseBlobAction TierToArchive -DaysAfterModificationGreaterThan 90
$action = Add-AzStorageAccountManagementPolicyAction -InputObject $action -BaseBlobAction Delete -DaysAfterModificationGreaterThan 365
$filter = New-AzStorageAccountManagementPolicyFilter -PrefixMatch "logs/" -BlobType blockBlob
$rule   = New-AzStorageAccountManagementPolicyRule -Name move-old-logs -Action $action -Filter $filter
Set-AzStorageAccountManagementPolicy -ResourceGroupName $rg -StorageAccountName $sa -Rule $rule
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "This operation is not permitted on an archived blob" | Blob is in **Archive** (offline) | Rehydrate: set tier to Hot/Cool/Cold, or copy it |
| Rehydration still "pending" after hours | Standard priority takes up to 15 h | Wait, or use **High** priority next time |
| Lifecycle rule did nothing | Rules can take up to **24 h**, wrong **prefix**, or wrong blob type | Check `prefixMatch` includes the container name, wait a day |
| "Last accessed" rule can't be saved | Access time tracking is off | Turn on **last access time tracking** first |
| Archive option greyed out | Account is ZRS/GZRS/RA-GZRS, or Premium | Use LRS/GRS/RA-GRS Standard |
| Unexpected cost after moving blobs | **Early deletion** fee | Respect minimum days: Cool 30, Cold 90, Archive 180 |
| Deleted blob can't be found | Soft delete was off, or retention expired | Turn on soft delete. Use "Show deleted blobs" in the container |
| Can't delete a blob, "immutable" error | Immutability policy or legal hold | Wait for retention to end / remove legal hold (locked policies can't be removed) |

## Clean up
```bash
az storage account management-policy delete --account-name $SA -g $RG
az storage container delete --account-name $SA --name logs --auth-mode login
```
> Don't create a **locked** immutability policy in a lab – you can't delete the data (or the account) until it expires.

## Check yourself
<details><summary>1. Logs must be kept 7 years for audit, almost never read, cheapest option. A reader can wait a day.</summary>

**Archive** tier, moved there by a **lifecycle rule**. Add an **immutability policy** if they must not be changed.
</details>

<details><summary>2. A lifecycle rule was created an hour ago, but no blobs have moved. What's wrong?</summary>

Probably nothing – rules can take up to **24 hours** to run.
</details>

<details><summary>3. A user overwrote an important file in a container. What should have been on to get the old one back?</summary>

**Blob versioning** (or snapshots). Soft delete alone protects deletes; versioning protects overwrites.
</details>

## Related
- [Storage Redundancy](storage-redundancy.md)
- [Storage Access](storage-access.md)
- [AzCopy and Storage Explorer](azcopy-storage-explorer.md)
- [Resource Locks](../04-governance/resource-locks.md)
