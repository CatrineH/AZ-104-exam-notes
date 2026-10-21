# Azure Files

[← Back to README](../README.md)

## In one line
**Azure Files** gives you fully managed **file shares in the cloud** that you mount like a normal network drive (`Z:` in Windows, `/mnt/share` in Linux) – Azure handles the servers, disks, and redundancy.

## When to use it (and when not)
| Use Azure Files for | Use something else for |
|---|---|
| Replacing or extending **on-premises file servers** | Huge amounts of unstructured data read by apps over HTTP → **Blob** |
| **Shared folders** for apps running on several VMs | Disk for one VM → **managed disk** |
| "Lift and shift" of apps that expect a file share | Users who need local speed in a branch office → add **[Azure File Sync](azure-file-sync.md)** |
| Shared config files, logs, and tools for admins | |

## Key concepts

- Lives inside a **storage account**, next to blob containers, queues, and tables
- Can be mounted **at the same time** by many VMs and on-premises computers
- Endpoint: `https://<storageaccount>.file.core.windows.net/<share>`, UNC path `\\<storageaccount>.file.core.windows.net\<share>`

### Protocols

| Protocol | Used by | Notes |
|---|---|---|
| **SMB** (2.1, 3.0, 3.1.1) | Windows, Linux, macOS | Most common. Uses **TCP port 445** |
| **NFS 4.1** | Linux | **Premium only**, and only through a private network (private endpoint or service endpoint) |
| **REST (HTTPS)** | Portal, CLI, PowerShell, Storage Explorer | Used when you upload or list files with commands |

> **!!!** **Port 445** must be open **outbound**. Many home internet providers and company networks block it, so mounting from your own PC can fail even though everything in Azure is correct. From an Azure VM it normally works.

### Blob storage vs Azure Files

| | Blob storage | Azure Files |
|---|---|---|
| Structure | Containers with objects (flat) | Real folders and files |
| Access | HTTP/REST, SDKs | **Mounted as a drive** (SMB/NFS) + REST |
| Best for | Images, videos, backups, data lakes | Shared folders, file server replacement |
| Several VMs at once | Through the app | Mounted on all of them |

### Tiers

| Tier | Storage | Account type | Use for |
|---|---|---|---|
| **Premium** | SSD | **FileStorage** (premium) | Low latency, databases, high performance. Supports NFS |
| **Transaction optimized** | HDD | General-purpose v2 | Many transactions, little storage |
| **Hot** | HDD | General-purpose v2 | General team shares |
| **Cool** | HDD | General-purpose v2 | Archive-like data that is rarely used |

- Premium requires its **own storage account type** (FileStorage). You can't move a share between standard and premium – you must copy the data
- Standard tiers can be changed on an existing share
- Premium supports only **LRS and ZRS** redundancy

### Size
- A standard share is up to **5 TiB** by default, or **100 TiB** with **large file shares** enabled on the storage account
- The **quota** sets the maximum size of a share

### Authentication

| Method | How it works |
|---|---|
| **Storage account key** | Full access. The portal's **Connect** script uses this. Good for labs, not for users |
| **Identity-based (Kerberos)** | Users sign in with their own identity through **AD DS**, **Microsoft Entra Domain Services**, or **Microsoft Entra Kerberos** (for hybrid users) |

**Permissions with identity-based access – two levels:**

| Level | Set with | Example |
|---|---|---|
| **Share level** | **Azure RBAC** roles | Storage File Data SMB Share **Reader** / **Contributor** / **Elevated Contributor** |
| **Folder and file level** | **Windows ACLs** (NTFS permissions) | Only HR can open the `HR` folder |

A user needs access at **both** levels. RBAC lets them into the share, NTFS permissions decide what they can do inside. **Elevated Contributor** is needed to **change** NTFS permissions.

### Data protection

| Feature | What it does |
|---|---|
| **Soft delete** | A deleted **share** can be restored within the retention period (1–365 days, default 7). On by default |
| **Share snapshots** | Read-only, point-in-time copies of the whole share. Up to **200** per share. Shown as **Previous Versions** in Windows |
| **Azure Backup** | Scheduled snapshots with retention, managed from a Recovery Services vault |

### Azure File Sync
Keeps an **on-premises Windows Server** in sync with an Azure file share. Full notes: [azure-file-sync.md](azure-file-sync.md).

### Network security
The same options as for blob storage (see [Storage Network Security](storage-network-security.md)):
- Service endpoints and the storage firewall
- Private endpoint with sub-resource **`file`** and private DNS zone **`privatelink.file.core.windows.net`**

## Exam traps
- Mount fails from home/office but works from an Azure VM → **port 445 is blocked outbound**. Answer: VPN/ExpressRoute, or Azure File Sync (sync uses 443)
- **NFS** = Premium (FileStorage) + private network + **"Secure transfer required" turned off** on the account. No Windows clients
- You can't **convert** a Standard share to Premium – new account + copy
- Share-level **RBAC** alone isn't enough when NTFS ACLs deny access – both levels count
- Share **snapshots** are per **share**, not per file. Soft delete protects the **share**, not single files (use snapshots / backup for files)
- Shares over 5 TiB need **large file shares** enabled on the account
- With identity-based access, the **default share-level permission** can give all authenticated users a role without per-user assignments

## Do it

### Portal
1. Storage account → **Data storage → File shares → + File share** → name, tier, (quota)
2. Open the share → **Upload** files, **+ Add directory**
3. **Connect** → choose Windows/Linux/macOS → copy the script (uses the account key)
4. **Snapshots → + Add snapshot**
5. Account → **Data protection** → soft delete for file shares

### Azure CLI
```bash
# Create a share (management plane – no key needed)
az storage share-rm create -g $RG --storage-account $SA \
  --name teamshare --quota 100 --access-tier Hot

# Upload a file (data plane)
KEY=$(az storage account keys list -n $SA -g $RG --query [0].value -o tsv)
az storage directory create --account-name $SA --account-key $KEY --share-name teamshare --name docs
az storage file upload --account-name $SA --account-key $KEY \
  --share-name teamshare --source ./readme.txt --path docs/readme.txt
az storage file list --account-name $SA --account-key $KEY --share-name teamshare --path docs -o table

# Snapshot
az storage share snapshot --account-name $SA --account-key $KEY --name teamshare

# Change tier / quota
az storage share-rm update -g $RG --storage-account $SA --name teamshare --access-tier Cool --quota 200
```

**Mount on Linux (SMB 3.1.1):**
```bash
sudo mkdir -p /mnt/teamshare
nc -zvw3 $SA.file.core.windows.net 445          # is port 445 open?
sudo mount -t cifs //$SA.file.core.windows.net/teamshare /mnt/teamshare \
  -o vers=3.1.1,username=$SA,password=$KEY,serverino,dir_mode=0777,file_mode=0777
```

### PowerShell
```powershell
New-AzRmStorageShare -ResourceGroupName $rg -StorageAccountName $sa `
  -Name teamshare -QuotaGiB 100 -AccessTier Hot

$key = (Get-AzStorageAccountKey -ResourceGroupName $rg -Name $sa)[0].Value
$ctx = New-AzStorageContext -StorageAccountName $sa -StorageAccountKey $key
New-AzStorageDirectory -ShareName teamshare -Path docs -Context $ctx
Set-AzStorageFileContent -ShareName teamshare -Source .\readme.txt -Path docs/readme.txt -Context $ctx

# Snapshot
$share = Get-AzStorageShare -Name teamshare -Context $ctx
$share.ShareClient.CreateSnapshot()
```

**Mount on Windows:**
```powershell
Test-NetConnection -ComputerName "$sa.file.core.windows.net" -Port 445   # TcpTestSucceeded : True?

cmdkey /add:"$sa.file.core.windows.net" /user:"localhost\$sa" /pass:"$key"
New-PSDrive -Name Z -PSProvider FileSystem -Root "\\$sa.file.core.windows.net\teamshare" -Persist
# or: net use Z: \\<sa>.file.core.windows.net\teamshare /user:localhost\<sa> <key>
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| **System error 53** ("network path was not found") / `TcpTestSucceeded: False` | **Port 445 blocked** outbound (ISP or company firewall) | Mount from an Azure VM, use VPN, or Azure File Sync |
| Same error from an Azure VM | Storage firewall blocks the subnet, or the **NSG** denies outbound 445 | Allow the subnet in storage networking, check NSG outbound |
| **System error 5** / access denied | Wrong key, or identity lacks **share-level RBAC** / NTFS permission | Check key, assign Storage File Data SMB Share Contributor, check NTFS |
| "Mount error(13): Permission denied" on Linux | Wrong username/key, or SMB version too old | `username=<account>`, `vers=3.1.1` |
| "Mount error(112): Host is down" on Linux | Port 445 blocked or SMB 3 encryption not supported by client | Check `nc -zv ... 445`, update cifs-utils |
| Can't create an NFS share | Account is Standard, or secure transfer is on | FileStorage account, disable "Secure transfer required", private network |
| Upload fails: share full | **Quota** reached | Increase quota (and enable large file shares for > 5 TiB) |
| Can't delete the storage account | A share has a snapshot or a backup lease | Delete snapshots / stop backup first |

## Clean up
```bash
sudo umount /mnt/teamshare
az storage share-rm delete -g $RG --storage-account $SA --name teamshare --include snapshots --yes
```
```powershell
Remove-PSDrive -Name Z
cmdkey /delete:"$sa.file.core.windows.net"
```

## Check yourself
<details><summary>1. Users in an office can't map an Azure file share. Mapping from an Azure VM works. Most likely cause?</summary>

The office network blocks **outbound TCP 445**.
</details>

<details><summary>2. Linux servers need an NFS 4.1 share. What account and network setup do you need?</summary>

**Premium FileStorage** account, private endpoint or service endpoint, and **secure transfer disabled**.
</details>

<details><summary>3. A user has Storage File Data SMB Share Contributor but can't open the `Finance` folder. Why?</summary>

**NTFS permissions** on the folder deny them. Both levels must allow access.
</details>

## Related
- [Azure File Sync](azure-file-sync.md)
- [Storage Access](storage-access.md)
- [Storage Network Security](storage-network-security.md)
- [Storage Redundancy](storage-redundancy.md)
