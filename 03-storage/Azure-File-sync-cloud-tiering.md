# Azure File Sync cloud tier - Implement and manage storage

[← Back to README](../README.md)

## In one line
Azure always keeps **multiple copies** of your data. The redundancy option decides **where** those copies live: one datacenter, three zones, or also a second region.

| Component | What it is |
|---|---|
| **Storage Sync Service** | The top-level Azure resource. A server can be registered with only **one** Storage Sync Service. |
| **Sync group** | Defines what gets synced. It contains **one** cloud endpoint and one or more server endpoints. |
| **Cloud endpoint** | An Azure file share. There can be only one per sync group. |
| **Server endpoint** | A path on a registered server, for example `D:\Data`. |
| **Registered server** | A Windows Server with the **Azure File Sync agent** installed. This establishes trust between the server and the Storage Sync Service. |

## 2. What does cloud tiering do?

Cloud tiering is an **optional feature on a server endpoint**. It turns the local server into a smart cache:

- **Hot files**, the ones accessed frequently, are kept in full on local disk.
- **Cold files**, the ones rarely accessed, are *tiered*. Their content lives only in Azure, and locally only a **reparse point** remains. It acts as a pointer that keeps the file name, metadata and ACLs.
- To the user, the file looks completely normal in File Explorer. When someone opens it, it is **recalled** automatically from Azure. This is transparent, but with somewhat higher latency.
- The sync engine maintains a **heatmap** of access patterns based on last access time. This decides which files are hot and which are cold.

## 3. The two policies

### Volume free space policy
- Defines how much of the **entire volume** must be kept free. **The default is 20%.**
- When free space drops below the threshold, the coldest files are tiered until the threshold is met again.
- The policy applies to the **volume**, not just the server endpoint. If several server endpoints share a volume, **the most restrictive (highest) value applies**.

### Date policy
- Tiers files that **have not been read or written for X days**, for example 30 days.
- Optional, and can be combined with the volume free space policy.

### Conflict rule (important for the exam)
> **The volume free space policy always takes precedence.**
> Even if a file is "recent enough" under the date policy, it is tiered when the disk needs space.

**Example:** The date policy is 60 days and free space is 20%. The disk fills up, so files last used 10 days ago are also tiered until 20% of the disk is free again.
## 4. Limitations and pitfalls

| Rule | Detail |
|---|---|
| NO - Not on the system volume | Cloud tiering is **not** supported on the system volume (usually `C:`). |
| Yes -  Server endpoint only | Tiering is configured on the **server endpoint**, never on the cloud endpoint. |
|  Antivirus and backup | These tools must respect the **offline attribute**. Otherwise a full scan or backup can trigger a recall of *every* file. Back up the **Azure file share** with Azure Backup, not the local tiered folder. |
|  Changes made directly in Azure | Changes made directly in the Azure file share are picked up by a **change detection job** that runs once every 24 hours. They do not show up locally right away. |

---

## 5. Useful PowerShell commands (on the server)

```powershell
# Load the Azure File Sync module on the registered server
Import-Module "C:\Program Files\Azure\StorageSyncAgent\StorageSync.Management.ServerCmdlets.dll"

# Recall all tiered files in a folder, e.g. before a migration
Invoke-StorageSyncFileRecall -Path "D:\Data\Projects"

# Force tiering of a specific folder/file manually
Invoke-StorageSyncCloudTiering -Path "D:\Data\Archive"
```

---

## 6. Typical exam questions

| Question | Answer |
|---|---|
| What does cloud tiering do? | Keeps hot files local, tiers cold files to Azure and leaves a pointer (reparse point) |
| Which policy wins in a conflict? | **Volume free space policy** |
| Default volume free space? | **20%** |
| Where is cloud tiering configured? | On the **server endpoint** |
| Can it be used on `C:`? | **No**, not on the system volume |
| How many cloud endpoints per sync group? | **One** |
| How many Storage Sync Services can a server be registered with? | **One** |
| How do you bring all files back locally? | `Invoke-StorageSyncFileRecall` |

---

## OBS - Common wrong answers to recognise

- *"Moves the entire share to Azure and deletes local files."* Wrong. Only cold files are tiered, and the pointers stay.
- *"Replicates synchronously to a second on-premises server."* Wrong. Sync goes through Azure, and tiering is about space, not redundancy.
- *"Compresses local files."* Wrong. Tiering moves file content and does not compress anything.

---

*Source: Microsoft Learn – Azure File Sync cloud tiering overview*
## Related
- [Storage Overview](storage-overview.md)
- [Blob Tiers and Lifecycle](blob-tiers-lifecycle.md)
- [Availability](../05-compute/05-availability.md) – availability zones for VMs
