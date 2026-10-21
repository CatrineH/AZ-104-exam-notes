# Azure File Sync

[← Back to README](../README.md)

## In one line
**Azure File Sync** keeps **on-premises Windows file servers** in sync with an **Azure file share**, so users keep local speed while Azure holds the full copy.

## When to use it (and when not)
| Use case | How Azure File Sync helps |
|---|---|
| **Cloud tiering** | Keep only frequently used files on the local server, the rest in Azure – saves local disk space |
| **Multi-site sync** | Several offices work on the same files through their own local server |
| **Replace DFS Replication (DFS-R)** | Azure becomes the hub instead of server-to-server replication |
| **Backup** | Back up the Azure file share with Azure Backup instead of backing up every server |
| **Disaster recovery** | If a server fails, set up a new one and the files come back from Azure |
| **Migration** | Move file server data to Azure while users keep working |

Not for: Linux file servers, Windows 10/11 clients, or users who can mount the Azure share directly and don't need a local cache.

## Key concepts

- Users keep working on their **local file server** – same speed, same drive letters, same NTFS permissions
- Several servers in different offices can sync to the **same** Azure file share

```
Office Oslo                    Azure                     Office Bergen
Windows Server ◄──sync──► Azure file share ◄──sync──► Windows Server
  D:\Data                  (full copy)                  D:\Data
```

### Components

| Component | What it is |
|---|---|
| **Storage Sync Service** | The top-level Azure resource. Everything else is created inside it |
| **Sync group** | Defines **what** to sync – which share and which server folders |
| **Cloud endpoint** | The **Azure file share** in a sync group |
| **Registered server** | A Windows Server with the **Azure File Sync agent** installed, registered to the Storage Sync Service |
| **Server endpoint** | A **folder or volume** on a registered server, for example `D:\Data` |

```
Storage Sync Service
├── Registered servers
│   ├── FS-SERVER1
│   └── FS-SERVER2
└── Sync group: "CompanyData"
    ├── Cloud endpoint  → Azure file share "companydata"
    ├── Server endpoint → FS-SERVER1   D:\Data
    └── Server endpoint → FS-SERVER2   D:\Data
```

### Rules and limits (exam favorites)
- A sync group has **exactly one cloud endpoint** but can have **many server endpoints**
- A server can be registered to **only one Storage Sync Service** at a time
- A server can have **several server endpoints**, in **different sync groups**, as long as the paths **don't overlap**
- The **Storage Sync Service** and the **storage account** must be in the **same region**
- Server endpoints must be on **NTFS** volumes
- **Cloud tiering is not supported on the system volume** (usually `C:`)
- The agent supports **Windows Server only** – not Linux, not Windows 10/11

### Setup order

| Step | What to do | Where |
|---|---|---|
| 1 | Create a **storage account** and an **Azure file share** | Azure portal |
| 2 | Create the **Storage Sync Service** | Azure portal |
| 3 | **Install the Azure File Sync agent** on the Windows Server | The server |
| 4 | **Register the server** with the Storage Sync Service (the agent opens a sign-in window after install) | The server |
| 5 | Create a **sync group** and choose the file share as the **cloud endpoint** | Azure portal |
| 6 | Add a **server endpoint** (server + folder path), and choose cloud tiering settings | Azure portal |

Remember: **share → Storage Sync Service → agent → register → sync group → server endpoint.**

### Network requirements
- The agent communicates over **HTTPS (TCP 443) outbound** only
- **Port 445 (SMB) is not needed** for sync – that's only for mounting the share directly

### Cloud tiering
The local server works like a **cache**. Popular files stay local. Rarely used files are kept **only in Azure**, and the server keeps a small **pointer** (a "tiered file").
- To users, tiered files look like normal files in the folder
- When someone opens a tiered file, it is **downloaded automatically** ("recalled")
- Tiered files have the **Offline** (and Recall-on-data-access) attribute set

| Policy | What it does | Example |
|---|---|---|
| **Volume free space policy** | Keeps a percentage of the volume **free** by tiering the least used files | Keep 20% free on `D:` |
| **Date policy** | Tiers files that haven't been used for a number of days | Tier files not opened in 60 days |

Both can be used together. If they conflict, the **volume free space policy wins**.

### How sync works

| Change made on | When it syncs |
|---|---|
| A **server endpoint** | Almost immediately – the agent detects changes on the server |
| The **Azure file share directly** (portal, SMB mount, REST) | Within **24 hours** – a change detection job scans the share once a day |

**Sync conflicts:** if the same file is changed in two places at the same time, **both versions are kept**. The latest change keeps the original name, the other is saved as `FileName-ServerName.ext`.

### Backup and recovery
- Back up the **Azure file share** with **Azure Backup**, not each server
- Don't use on-premises backup software that reads every file on a server with cloud tiering – it will recall all tiered files
- **Disaster recovery:** install the agent on a new server, register it, and add a server endpoint. The folder structure comes back first, then files are downloaded as they are used

### Azure Files vs Azure File Sync

| | Azure Files | Azure File Sync |
|---|---|---|
| What it is | The file share in Azure | A service that syncs servers with that share |
| Users connect to | The Azure share (SMB, port 445) | Their **local** Windows Server |
| Needs a local server | No | Yes – Windows Server with agent |
| Speed for users | Depends on the internet connection | Local network speed |
| Works without Azure Files | – | No – always needs an Azure file share |

## Exam traps
- **One cloud endpoint per sync group.** Two shares = two sync groups
- **One Storage Sync Service per server.** A server can't join two Storage Sync Services
- **Same region** for Storage Sync Service and storage account
- **Cloud tiering on C:** isn't supported
- Port **445 is blocked** at the office but they still want Azure Files → File Sync works (uses **443**)
- A file changed directly in Azure shows up on servers within **24 hours** – "expected behaviour"
- To **remove** a server: delete its server endpoints first, **then** unregister it
- Volume free space policy **wins** over date policy

## Do it

### Portal
1. Create the storage account and file share ([Azure Files](azure-files.md))
2. **Create a resource → Azure File Sync** (Storage Sync Service) → same region as the account
3. On the Windows Server: download and install the **Azure File Sync agent** → the **Server Registration** tool opens → sign in → choose subscription, RG, Storage Sync Service
4. Storage Sync Service → **Sync groups → + Sync group** → choose storage account + file share (cloud endpoint)
5. Open the sync group → **Add server endpoint** → registered server + path `D:\Data` → turn on **Cloud tiering** → volume free space %, date policy
6. Watch **Server endpoint health** in the sync group

### Azure CLI
```bash
az extension add --name storagesync

az storagesync create -g $RG -n sss-az104 -l norwayeast
az storagesync sync-group create -g $RG --storage-sync-service sss-az104 -n sg-companydata
az storagesync sync-group cloud-endpoint create -g $RG \
  --storage-sync-service sss-az104 --sync-group-name sg-companydata \
  -n ce-companydata --storage-account $SA --azure-file-share-name companydata
```
Server registration and server endpoints are usually done with the agent / portal / PowerShell.

### PowerShell
```powershell
Install-Module Az.StorageSync -Scope CurrentUser

$sss = New-AzStorageSyncService -ResourceGroupName $rg -Name sss-az104 -Location norwayeast

# On the Windows Server (after installing the agent):
Register-AzStorageSyncServer -ResourceGroupName $rg -StorageSyncServiceName sss-az104

$sg = New-AzStorageSyncGroup -ParentObject $sss -Name sg-companydata

$saId = (Get-AzStorageAccount -ResourceGroupName $rg -Name $sa).Id
New-AzStorageSyncCloudEndpoint -ParentObject $sg -Name ce-companydata `
  -StorageAccountResourceId $saId -AzureFileShareName companydata

$server = Get-AzStorageSyncServer -ParentObject $sss
New-AzStorageSyncServerEndpoint -ParentObject $sg -Name FS-SERVER1-data `
  -ServerResourceId $server.ResourceId -ServerLocalPath "D:\Data" `
  -CloudTiering -VolumeFreeSpacePercent 20 -TierFilesOlderThanDays 60

# Force change detection after editing the Azure share directly (instead of waiting 24 h)
Invoke-AzStorageSyncChangeDetection -ParentObject $sg -Name ce-companydata -DirectoryPath "docs"
```

On the server, recall tiered files: `Invoke-StorageSyncFileRecall -Path D:\Data` (module in `C:\Program Files\Azure\StorageSyncAgent\StorageSync.Management.ServerCmdlets.dll`).

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Server doesn't show up when adding a server endpoint | Server not **registered**, or registered to another Storage Sync Service | Run `ServerRegistration.exe` from `C:\Program Files\Azure\StorageSyncAgent` |
| Can't create the cloud endpoint | Storage account in a **different region**, or storage firewall blocks the service | Same region. Allow **trusted Microsoft services** / the File Sync service |
| Files edited in the portal don't appear on servers | Change detection runs **once every 24 h** | Wait, or `Invoke-AzStorageSyncChangeDetection` |
| Sync errors in the portal ("files not syncing") | Invalid characters, file locked, path too long, or permissions | Check **Server endpoint health** and Event Viewer → *Applications and Services → Microsoft → FileSync → Agent* |
| Can't enable cloud tiering | Server endpoint is on the **system volume** (C:) | Move the endpoint to a data volume |
| Local disk suddenly full after backup | Backup software **recalled** all tiered files | Back up the Azure share with Azure Backup instead |
| Server can't reach the service | Outbound **443** blocked or proxy not set | Open 443 to Azure, configure proxy in the agent |
| Can't unregister the server | Server endpoints still exist | Delete server endpoints first |

## Clean up
Order matters: **server endpoints → cloud endpoint → sync group → unregister server → Storage Sync Service → storage account.**
```powershell
Remove-AzStorageSyncServerEndpoint -ParentObject $sg -Name FS-SERVER1-data -Force
Remove-AzStorageSyncCloudEndpoint -ParentObject $sg -Name ce-companydata -Force
Remove-AzStorageSyncGroup -ParentObject $sss -Name sg-companydata -Force
Unregister-AzStorageSyncServer -ParentObject $sss -ServerId $server.ServerId -Force
Remove-AzStorageSyncService -ResourceGroupName $rg -Name sss-az104 -Force
```

## Check yourself
<details><summary>1. Put in order: create sync group, install agent, create Storage Sync Service, add server endpoint, register server.</summary>

Create Storage Sync Service → install agent → register server → create sync group → add server endpoint.
</details>

<details><summary>2. Two departments need different Azure file shares synced to the same server. How many sync groups?</summary>

**Two** – one cloud endpoint per sync group. The server gets two server endpoints with non-overlapping paths.
</details>

<details><summary>3. Volume free space policy is 30%, date policy is 90 days. The disk is 80% full. What happens?</summary>

Files newer than 90 days can be tiered too, because the **volume free space policy wins**.
</details>

## Related
- [Azure Files](azure-files.md)
- [Storage Redundancy](storage-redundancy.md)
- [Storage Network Security](storage-network-security.md)
