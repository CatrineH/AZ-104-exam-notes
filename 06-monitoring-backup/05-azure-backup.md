# Azure Backup

## What it is
Microsoft's managed backup service: backs up Azure VMs, files, databases and on-prem workloads to Azure with no backup infrastructure for you to run. Data is encrypted, and soft delete protects against accidental or malicious deletion.

> Backup = **restore data** after loss/corruption (point-in-time copies).
> Site Recovery = **keep the business running** after a regional/site outage (replication + failover).

## Two vault types — key exam topic

| | **Recovery Services vault** | **Backup vault** |
|---|---|---|
| Azure VMs | YES | NO |
| SQL Server / SAP HANA in Azure VM | YES | NO |
| Azure Files (file shares) | YES NO (snapshot-based in RSV) |
| On-prem via MARS agent / MABS / DPM | YES | NO|
| **Azure Site Recovery** | YES | NO |
| Azure Blobs | NO | YES |
| Azure Managed Disks (disk backup) | NO | YES|
| Azure Database for PostgreSQL | NO | YES |
| AKS | NO | YES |

**Backup center / Business Continuity Center** gives one management view across both vault types and many subscriptions.

### Vault rules
- Vault must be in the **same region** as the data source (for VMs).
- Can protect resources across resource groups in the same region and subscription.
- **Storage redundancy** – LRS, ZRS or GRS (default **GRS**). **Must be set before the first backup**; it can't be changed afterwards (without stopping protection).
- **Cross Region Restore (CRR)** – with GRS, allows restoring in the paired secondary region.

## Backing up Azure VMs
- Uses the **VM backup extension**, installed automatically when backup is enabled.
- Process: snapshot of the disks → transferred to the vault.
- **Application-consistent** (Windows via VSS; Linux via pre/post scripts), **file-system consistent**, or **crash-consistent** (if VM is off).
- **Instant restore** – snapshots kept locally for 1–5 days (default 2) for fast restores, before/alongside vault tier.
- Policy types: **Standard** (one backup per day) and **Enhanced** (multiple backups per day, needed for Trusted Launch VMs and some disk types).

### Restore options for a VM
1. **Create a new VM** from a restore point
2. **Restore disks** (then attach or build a VM yourself, e.g., with a template)
3. **Replace existing** disks on the current VM
4. **File-level restore** (File Recovery) – mount the recovery point as a drive via a script and copy single files
5. **Cross Region Restore** (if enabled)

## Backup policies
- **Schedule** – daily/weekly (or hourly with Enhanced policy).
- **Retention** – daily, weekly, monthly, yearly points (GFS style). Max 9,999 recovery points per protected item.
- One policy can cover many VMs.

## On-premises options
| Option | Use |
|---|---|
| **MARS agent** (Microsoft Azure Recovery Services agent) | Files, folders, system state from Windows machines **directly** to the vault. No app-aware backup (no SQL). Max 3 backups/day. |
| **MABS** (Microsoft Azure Backup Server) | Free, app-aware backup of on-prem VMs, SQL, Exchange, SharePoint → disk, then to Azure. |
| **System Center DPM** | Same idea as MABS if you already own System Center. |

## Security features
- **Soft delete** – deleted backup data kept for 14 days (free), can be extended; can be made "always on".
- **Immutable vaults** – prevent reducing retention or deleting recovery points.
- **Multi-user authorization (MUA)** – critical operations need approval via Resource Guard.
- **Encryption** – at rest with platform-managed keys by default; customer-managed keys optional.
- RBAC roles: **Backup Contributor**, **Backup Operator** (can back up and restore, but not delete or change policies), **Backup Reader**.

## Azure Files backup
- Snapshot-based (snapshots stay in the storage account) + optional vaulted backup.
- Managed from the Recovery Services vault.
- Restore full share or individual files.

## CLI / PowerShell
```bash
# Create a Recovery Services vault
az backup vault create -g rg-backup -n rsv-cat --location westeurope

# Set storage redundancy BEFORE first backup
az backup vault backup-properties set -g rg-backup -n rsv-cat --backup-storage-redundancy LocallyRedundant

# Enable backup on a VM with the default policy
az backup protection enable-for-vm -g rg-backup --vault-name rsv-cat \
  --vm vm01 --policy-name DefaultPolicy

# Run an on-demand backup, keep until a date
az backup protection backup-now -g rg-backup --vault-name rsv-cat \
  --container-name vm01 --item-name vm01 --retain-until 30-12-2026
```
```powershell
New-AzRecoveryServicesVault -Name rsv-cat -ResourceGroupName rg-backup -Location westeurope
$vault = Get-AzRecoveryServicesVault -Name rsv-cat
Set-AzRecoveryServicesVaultContext -Vault $vault
$pol = Get-AzRecoveryServicesBackupProtectionPolicy -Name DefaultPolicy
Enable-AzRecoveryServicesBackupProtection -ResourceGroupName rg-vm -Name vm01 -Policy $pol
```

## Exam traps
- Back up a VM in **West Europe** → vault must be in **West Europe**.
- Back up **blobs or managed disks** → **Backup vault**, not Recovery Services vault.
- Change vault redundancy from GRS to LRS → only possible **before** any item is protected.
- Restore a single file from a VM backup → **File Recovery** (mount script), no need to restore the whole VM.
- Back up on-prem **files/folders** with no extra server → **MARS agent**. Need **SQL/Exchange app-aware** on-prem → **MABS/DPM**.
- You can't delete a vault that still has protected items or soft-deleted data — stop protection, delete data, wait out/undelete soft delete first.
- Backup Operator can't delete backups or modify policies.

## Self-test
1. You need daily backups of 20 Azure VMs in North Europe and blob backups for a storage account. How many vault types do you need?
2. What's the default storage redundancy for a Recovery Services vault, and when can you change it?
3. A user deleted one Word file from a VM's data disk. What's the fastest restore?
4. Which on-prem option backs up SQL Server in an application-aware way?
5. Someone deleted a VM's backup data yesterday. Is it gone?

<details><summary>Answers</summary>

1. Two: a Recovery Services vault (VMs) and a Backup vault (blobs).
2. GRS; only before the first item is protected.
3. File-level restore (File Recovery) from a recovery point.
4. MABS (or DPM). MARS can't.
5. No — soft delete keeps it for 14 days by default.
</details>
