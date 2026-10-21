# Moving Data: AzCopy, Storage Explorer, Import/Export, and Data Box

[← Back to README](../README.md)

## In one line
**AzCopy** is the command-line tool for copying data to, from, and between storage accounts. **Storage Explorer** is the desktop app that does the same with a GUI (and uses AzCopy under the hood).

## When to use it (and when not)
| Situation | Tool |
|---|---|
| Script or schedule a copy of many files / large data over the network | **AzCopy** |
| Copy between two storage accounts (server to server, no local download) | **AzCopy** `copy` with two URLs |
| Keep a local folder and a container the same | **AzCopy** `sync` |
| Browse, upload, download, manage SAS and access policies by clicking | **Storage Explorer** |
| A few files, quick | **Portal** upload |
| Tens of TB, slow or expensive network | **Azure Data Box** (Microsoft ships you a device) |
| Ship your **own** disks to an Azure datacenter | **Azure Import/Export** |
| Continuous sync from on-prem Windows file servers | **[Azure File Sync](azure-file-sync.md)** |

## Key concepts

### AzCopy
- A single executable (**AzCopy v10**), Windows/Linux/macOS. No install – download and run
- Works with **Blob**, **Files**, and **Data Lake (ADLS Gen2)**. Not Queue or Table
- Two ways to authorize:

| Auth | How | Works for |
|---|---|---|
| **Entra ID** | `azcopy login` (or managed identity / service principal) | Blob and Data Lake. Needs **Storage Blob Data Contributor** (or Reader for download) |
| **SAS token** | Add `?<sas>` to the URL | Blob and **Files** |

- **Jobs:** every run is a job with a log. Failed jobs can be **resumed**
- **Server-to-server copy:** between accounts, Azure copies directly – your PC doesn't download the data

| Command | Does |
|---|---|
| `azcopy copy` | Copy files/folders/containers (use `--recursive` for folders) |
| `azcopy sync` | Make destination match source (only changed files). `--delete-destination=true` also deletes extras |
| `azcopy make` | Create a container or share |
| `azcopy list` | List blobs/files |
| `azcopy remove` | Delete blobs/files |
| `azcopy jobs list` / `jobs resume <id>` | See and resume jobs |

### Azure Storage Explorer
- Free desktop app for Windows, macOS, Linux
- Connect with: **Entra ID sign-in**, **account name + key**, **connection string**, **SAS URL**, or local **Azurite** emulator
- Manage blobs, files, queues, tables, **SAS tokens**, **access policies**, snapshots, soft-deleted blobs, blob tiers
- Uses **AzCopy** for transfers

### Offline transfer

| Service | Who provides the disks | Typical size |
|---|---|---|
| **Data Box Disk** | Microsoft (SSDs) | Up to ~35 TB per order |
| **Data Box** | Microsoft (rugged device) | ~80 TB+ usable per device |
| **Import/Export** | **You** (your own disks, BitLocker encrypted, prepared with the **WAImportExport** tool) | Varies |

## Exam traps
- AzCopy with **Entra ID** (`azcopy login`) needs a **data role**, not Contributor
- **Azure Files** with AzCopy → use a **SAS** (or Entra ID with newer versions). Account keys can't be passed directly to AzCopy
- `copy` vs `sync`: `sync` compares **last modified** times and only copies differences. `copy` with `--overwrite` decides what to do with existing files
- `sync` deletes files at the destination **only** with `--delete-destination=true`
- Copy between accounts → AzCopy uses **server-to-server** APIs (fast, no local bandwidth)
- Huge data + slow network → **Data Box**, not AzCopy. **Your own disks** → **Import/Export**
- Import/Export disks must be **BitLocker** encrypted, prepared with **WAImportExport**
- Storage Explorer "can't see the subscription" → you signed in with the wrong account or tenant

## Do it

### Install AzCopy
```bash
# Linux
wget -O azcopy.tar.gz https://aka.ms/downloadazcopy-v10-linux
tar -xf azcopy.tar.gz --strip-components=1
sudo mv azcopy /usr/local/bin/
azcopy --version
```
```powershell
# Windows
Invoke-WebRequest -Uri https://aka.ms/downloadazcopy-v10-windows -OutFile azcopy.zip
Expand-Archive azcopy.zip -DestinationPath C:\azcopy
Get-ChildItem C:\azcopy -Recurse -Filter azcopy.exe   # add that folder to PATH
```
In **Cloud Shell**, AzCopy is already installed.

### AzCopy – common commands
```bash
# Sign in with Entra ID (needs Storage Blob Data Contributor)
azcopy login --tenant-id <tenant-id>

# Create a container
azcopy make "https://$SA.blob.core.windows.net/backup"

# Upload a folder
azcopy copy "./reports" "https://$SA.blob.core.windows.net/backup" --recursive

# Upload directly to Cool tier
azcopy copy "./old" "https://$SA.blob.core.windows.net/backup" --recursive --block-blob-tier Cool

# Download one blob
azcopy copy "https://$SA.blob.core.windows.net/backup/reports/q3.pdf" "./q3.pdf"

# Keep a container the same as a local folder
azcopy sync "./reports" "https://$SA.blob.core.windows.net/backup/reports" --delete-destination=true

# Copy a whole container to another account (server to server, SAS on both)
azcopy copy "https://src.blob.core.windows.net/data?<sas>" \
            "https://dst.blob.core.windows.net/data?<sas>" --recursive

# Upload to Azure Files (SAS)
azcopy copy "./docs" "https://$SA.file.core.windows.net/teamshare?<sas>" --recursive

# Jobs
azcopy jobs list
azcopy jobs show <job-id> --with-status=Failed
azcopy jobs resume <job-id>
```

> Always put URLs with SAS in **quotes** – the `&` in a SAS breaks the shell otherwise.

### Storage Explorer
1. Download from Microsoft and install
2. **Connect** (plug icon) → choose **Subscription** (Entra ID) or **Storage account or service** (key/SAS/connection string)
3. Expand **Storage Accounts → Blob Containers** → upload/download, right-click → **Get Shared Access Signature**, **Manage Access Policies**, **Change Access Tier**
4. **Activities** pane at the bottom shows transfers and errors (and the AzCopy command)

### Data Box (portal)
**Create a resource → Azure Data Box → Import to Azure** → choose type → destination storage account → Microsoft ships the device → copy data → ship back → data is uploaded → device is wiped.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `403 AuthorizationPermissionMismatch` after `azcopy login` | No **data role** | Storage Blob Data Contributor on account/container. Wait a few minutes |
| `403 AuthorizationFailure` | Storage **firewall** blocks your IP | Add your IP or run from an allowed subnet |
| `403 AuthenticationFailed` with SAS | SAS expired, wrong permissions, or URL not quoted | New SAS with `rwdlac` as needed, put URL in quotes |
| "No such file" / nothing copied from a folder | Forgot `--recursive` | Add `--recursive` |
| Job failed halfway | Network drop | `azcopy jobs resume <id>` |
| `sync` didn't delete removed files | `--delete-destination` not set | Add `--delete-destination=true` (use `prompt` first to be safe) |
| Storage Explorer shows no subscriptions | Wrong account/tenant or filtered | **Account management** → sign in again / choose tenant and subscriptions |
| Logs needed | | AzCopy logs are in `~/.azcopy` (Linux) or `%USERPROFILE%\.azcopy` (Windows) |

## Clean up
```bash
azcopy remove "https://$SA.blob.core.windows.net/backup" --recursive
azcopy logout
azcopy jobs clean
```

## Check yourself
<details><summary>1. You need to copy 2 TB from one storage account to another in a different region as fast as possible without downloading it locally.</summary>

**AzCopy `copy`** with source and destination URLs – it does a **server-to-server** copy.
</details>

<details><summary>2. 60 TB must move to Azure. The office has a 100 Mbps line. Best option?</summary>

**Azure Data Box**. Over the network it would take weeks.
</details>

<details><summary>3. `azcopy login` worked but uploads fail with 403. The user is Owner on the account.</summary>

Owner isn't a data role. Assign **Storage Blob Data Contributor**.
</details>

## Related
- [Storage Access](storage-access.md) – SAS and data roles
- [Blob Tiers and Lifecycle](blob-tiers-lifecycle.md)
- [Azure Files](azure-files.md)
