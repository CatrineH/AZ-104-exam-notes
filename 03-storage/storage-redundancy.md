# Azure Storage Redundancy

[← Back to README](../README.md)

## In one line
Azure always keeps **multiple copies** of your data. The redundancy option decides **where** those copies live: one datacenter, three zones, or also a second region.

## When to use it (and when not)
| Requirement | Answer |
|---|---|
| Cheapest, data can be recreated (test, cache, copies) | **LRS** |
| Survive a datacenter/zone failure, data must stay in one region | **ZRS** |
| Survive a regional outage | **GRS** or **GZRS** |
| **Read** data during a regional outage | **RA-GRS** or **RA-GZRS** |
| Survive both zone and regional failure, best availability | **RA-GZRS** |

Don't pay for geo-redundancy if the data can be rebuilt, or if data residency rules say it can't leave the region.

## Key concepts

| Option | Copies and location | Protects against |
|---|---|---|
| **LRS** | 3 copies in one datacenter | Disk or rack failure |
| **ZRS** | 3 copies across 3 availability zones | Datacenter or zone failure |
| **GRS** | LRS in primary + LRS in paired region | Regional outage |
| **RA-GRS** | GRS + read access to secondary | Regional outage, reads still work |
| **GZRS** | ZRS in primary + LRS in paired region | Zone failure and regional outage |
| **RA-GZRS** | GZRS + read access to secondary | Maximum availability |

- **RA-** = read access to the secondary region, for example `mystorage-secondary.blob.core.windows.net`
- Replication to the secondary region is **asynchronous** – recent writes can be lost in a failover
- **Last Sync Time** = everything written before this time is safely in the secondary
- Cost order, cheapest first: LRS → ZRS → GRS → RA-GRS → GZRS → RA-GZRS

### Failover
- With GRS/GZRS (no RA), the secondary is **not readable** until a **failover** happens
- **Customer-managed failover:** you start it yourself (portal: **Storage account → Redundancy → Prepare for failover**)
- After an unplanned failover, the account in the new primary region is **LRS**. You must set geo-redundancy again
- Data written after the **Last Sync Time** is lost

## Exam traps
- **Premium** accounts support only **LRS and ZRS** – no geo-redundancy
- The **Archive** tier works only with **LRS, GRS, RA-GRS** – not with ZRS, GZRS, or RA-GZRS
- **GRS ≠ readable secondary.** Only the **RA-** options let you read before a failover
- Geo-replication is **asynchronous** → "no data loss" is never guaranteed with GRS
- **LRS ↔ GRS/RA-GRS** is a simple setting change. Moving **to or from ZRS/GZRS** is a **conversion** that takes time (or a manual migration with AzCopy)
- The secondary region is the **paired region** – you don't choose it (for example Norway East → Norway West)
- Redundancy protects against **hardware/region failure**, not against someone **deleting** a file. That's soft delete, versioning, and backup

## Do it

### Portal
**Storage account → Data management → Redundancy** → choose the new option → **Save**.
The same page shows the primary and secondary region and **Last sync time**, and has **Prepare for failover**.

### Azure CLI
```bash
# See current redundancy
az storage account show -n $SA -g $RG --query sku.name -o tsv

# LRS → RA-GRS
az storage account update -n $SA -g $RG --sku Standard_RAGRS

# Check Last Sync Time for the secondary
az storage account show -n $SA -g $RG --expand geoReplicationStats \
  --query geoReplicationStats.lastSyncTime

# Start a customer-managed failover (only in a lab!)
az storage account failover -n $SA -g $RG
```

SKU names: `Standard_LRS`, `Standard_ZRS`, `Standard_GRS`, `Standard_RAGRS`, `Standard_GZRS`, `Standard_RAGZRS`, `Premium_LRS`, `Premium_ZRS`.

### PowerShell
```powershell
Set-AzStorageAccount -ResourceGroupName $rg -Name $sa -SkuName Standard_RAGRS

$acc = Get-AzStorageAccount -ResourceGroupName $rg -Name $sa -IncludeGeoReplicationStats
$acc.GeoReplicationStats.LastSyncTime

Invoke-AzStorageAccountFailover -ResourceGroupName $rg -Name $sa
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| ZRS/GZRS not in the list | The region doesn't have availability zones, or it's a Premium account (no GZRS) | Choose a region with zones |
| Can't change to ZRS/GZRS with `update` | That change needs a **conversion** | Use **Redundancy → Change** (conversion) in the portal, or copy data to a new account |
| Can't switch to ZRS / GZRS | Account contains **Archive** blobs | Rehydrate or delete archive blobs first |
| Reads fail from the `-secondary` endpoint | Account is GRS, not **RA-GRS** | Change to RA-GRS / RA-GZRS |
| Data missing after failover | Writes after **Last Sync Time** weren't replicated | Expected with async replication. Check Last Sync Time before failing over |
| Account is LRS after failover | Normal after unplanned failover | Set GRS/GZRS again |

## Clean up
Set the lab account back to LRS so it costs less, or delete the resource group:
```bash
az storage account update -n $SA -g $RG --sku Standard_LRS
```

## Check yourself
<details><summary>1. Data must survive a zone outage and must never leave Norway East. Cheapest choice?</summary>

**ZRS**. GRS/GZRS would copy data to the paired region.
</details>

<details><summary>2. An app must keep reading blobs during a regional outage without anyone doing a failover. Which options work?</summary>

**RA-GRS** or **RA-GZRS** – read access to the secondary.
</details>

<details><summary>3. You want to move old blobs to Archive. The account is GZRS. What happens?</summary>

Archive isn't supported on GZRS. You'd need LRS, GRS, or RA-GRS.
</details>

## Related
- [Storage Overview](storage-overview.md)
- [Blob Tiers and Lifecycle](blob-tiers-lifecycle.md)
- [Availability](../05-compute/05-availability.md) – availability zones for VMs
