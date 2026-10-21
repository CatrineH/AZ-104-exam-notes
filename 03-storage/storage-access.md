# Storage Account Access

[← Back to README](../README.md)

## In one line
There are three ways to prove you may touch the data: an **account key** (full access to everything), a **SAS token** (limited and time-bound), or **Entra ID + a data role** (the recommended way).

## When to use it (and when not)
| Situation | Use |
|---|---|
| Users, admins, and Azure resources inside your tenant | **Entra ID + RBAC data role** (managed identity for apps) |
| Give an outside partner read access to one container for 7 days | **SAS** (user delegation SAS if possible) |
| Need to revoke many SAS tokens at once | **Stored access policy** behind a service SAS |
| Legacy tool that only understands a connection string | **Account key** – stored in Key Vault, rotated |
| Lock down so keys can't be used at all | **Disable shared key access** on the account |

Avoid account keys in code or scripts. Anyone with the key owns all the data.

## Key concepts

### Storage account keys
- Every storage account has **two keys**: key1 and key2
- Each key gives **full access** to the whole account
- Two keys allow **rotation without downtime**: move apps to key2 → regenerate key1 → move back
- Store keys in **Azure Key Vault**, never in code
- **Regenerating a key** breaks every SAS that was signed with it
- You can **turn off** key access completely: **Configuration → Allow storage account key access = Disabled**

### Shared Access Signatures (SAS)
A SAS is a token added to a URL that gives **limited, delegated access**: which permissions, for how long, and from which IPs.

```
https://az104catrinesta.blob.core.windows.net/data/file.txt?sv=...&sp=r&se=2026-11-01T00:00Z&sip=1.2.3.4&spr=https&sig=...
                                                                 │      │                    │            │          └ signature
                                                       permissions  expiry               allowed IP    HTTPS only
```

| Type | Signed with | Notes |
|---|---|---|
| **User delegation SAS** | Entra ID credentials | Blob (and Data Lake) only, **most secure** |
| **Service SAS** | Account key | One service. Can use a **stored access policy** |
| **Account SAS** | Account key | One or more services, can include service-level operations |

### Stored access policy
A policy saved **on a container** (or share/queue/table) that holds the permissions and expiry. A service SAS can point to it.
- Change or delete the policy → **all SAS tokens using it change or stop working** immediately
- Max **5** stored access policies per container

### Full access vs delegated access
| | Account key | SAS |
|---|---|---|
| Access level | Everything | Only what the token allows |
| Time limit | None | Has an expiry time |
| Revoke | Regenerate the key | Expiry, stored access policy, or regenerate the signing key |

### Entra ID and data roles (preferred)
Control plane roles (Owner, Contributor) manage the **account**. Data plane roles give access to the **data**:

| Role | Can |
|---|---|
| **Storage Blob Data Reader** | Read and list blobs |
| **Storage Blob Data Contributor** | Read, write, delete blobs |
| **Storage Blob Data Owner** | Everything + set POSIX ACLs (Data Lake) |
| **Storage Queue Data Contributor** | Read, write, delete queue messages |
| **Storage File Data SMB Share Contributor** | Read/write files in a share over SMB |
| **Storage Account Contributor** | Manage the account **and** read the keys (so indirectly all data) |
| **Reader and Data Access** | See the account and **list keys** |

- Assign at the **narrowest scope**: a single **container** is possible
- Role assignments can take a few minutes to work

### Azure Storage Explorer
Desktop app for working with storage. I used it to:
- Connect to storage accounts (Entra ID sign-in, account key, connection string, or SAS URL)
- Browse containers
- List blobs
- Download files

More in [AzCopy and Storage Explorer](azcopy-storage-explorer.md).

## Exam traps
- **Contributor/Owner ≠ data access** with Entra ID. With `--auth-mode login` they still need a **Storage Blob Data** role. (They *can* read the keys though, so the portal may fall back to key access)
- **User delegation SAS** is the most secure SAS and works for **Blob only**
- To revoke **one group of SAS tokens** without breaking everything → **stored access policy**. To revoke **all** account-key SAS tokens → **regenerate the key**
- To revoke all **user delegation SAS** tokens → `revoke-delegation-keys`
- A SAS needs **both** the right permission letters (`r`, `w`, `d`, `l`, `c`, `a`) **and** a valid time window. Missing `l` (list) = can't list the container
- **Anonymous (public) access** must be allowed on the **account** *and* set on the **container** before anyone can read without auth
- Managed identity + data role is the answer when the question says "**without storing credentials**"

## Do it

### Portal
- **Keys:** Storage account → **Security + networking → Access keys** → Show / **Rotate key**
- **SAS for the account:** **Security + networking → Shared access signature**
- **SAS for one blob/container:** open the container → **Shared access tokens** → choose **Signing method** (Account key / User delegation key)
- **Stored access policy:** container → **Access policy → + Add policy**
- **Data role:** Storage account (or container) → **Access control (IAM) → Add role assignment** → Storage Blob Data Reader → choose user/group/managed identity
- **Disable key access:** **Settings → Configuration → Allow storage account key access → Disabled**

### Azure CLI
```bash
SA_ID=$(az storage account show -n $SA -g $RG --query id -o tsv)

# Keys
az storage account keys list -n $SA -g $RG -o table
az storage account keys renew -n $SA -g $RG --key key1

# Give a user read access to ONE container
az role assignment create \
  --assignee cat@contoso.com \
  --role "Storage Blob Data Reader" \
  --scope "$SA_ID/blobServices/default/containers/data"

# User delegation SAS (signed with Entra ID) – read+list for 1 day
az storage container generate-sas --account-name $SA --name data \
  --permissions rl --expiry 2026-11-01T00:00Z \
  --auth-mode login --as-user --https-only -o tsv

# Stored access policy + service SAS that uses it
KEY=$(az storage account keys list -n $SA -g $RG --query [0].value -o tsv)
az storage container policy create --account-name $SA --account-key $KEY \
  --container-name data --name partner-read --permissions rl --expiry 2026-11-30T00:00Z
az storage container generate-sas --account-name $SA --account-key $KEY \
  --name data --policy-name partner-read -o tsv

# Revoke all user delegation SAS tokens
az storage account revoke-delegation-keys -n $SA -g $RG

# Turn off key access completely
az storage account update -n $SA -g $RG --allow-shared-key-access false
```

### PowerShell
```powershell
# Keys
Get-AzStorageAccountKey -ResourceGroupName $rg -Name $sa
New-AzStorageAccountKey -ResourceGroupName $rg -Name $sa -KeyName key1   # regenerate

# Connect with account key (full access)
$ctx = New-AzStorageContext -StorageAccountName $sa -StorageAccountKey "<key>"
# Connect with SAS token (delegated access)
$ctx = New-AzStorageContext -StorageAccountName $sa -SasToken "<sas>"
# Connect with your Entra ID sign-in (needs a data role)
$ctx = New-AzStorageContext -StorageAccountName $sa -UseConnectedAccount

# Browse, list, download
Get-AzStorageContainer -Context $ctx
Get-AzStorageBlob -Container "data" -Context $ctx
Get-AzStorageBlobContent -Container "data" -Blob "file.txt" -Destination "C:\temp" -Context $ctx

# Create a SAS for a container (read + list, 4 hours)
New-AzStorageContainerSASToken -Name data -Permission rl -ExpiryTime (Get-Date).AddHours(4) -Context $ctx

# Data role on one container
$scope = (Get-AzStorageAccount -ResourceGroupName $rg -Name $sa).Id + "/blobServices/default/containers/data"
New-AzRoleAssignment -SignInName cat@contoso.com -RoleDefinitionName "Storage Blob Data Reader" -Scope $scope
```

## Troubleshooting

| Error / symptom | Likely cause | Fix |
|---|---|---|
| **AuthorizationPermissionMismatch** (403) | Network OK, but your identity lacks a **data role** | Assign Storage Blob Data Reader/Contributor. Wait a few minutes |
| **AuthorizationFailure** / "This request is not authorized to perform this operation" | The **storage firewall** blocks your IP/network | See [Storage Network Security](storage-network-security.md) |
| **AuthenticationFailed** – "Signature did not match" or "Signed expiry time … must be after signed start time" | SAS **expired**, not started yet (clock skew), or the key it was signed with was **regenerated** | New SAS. Set start time a few minutes in the past |
| **AuthorizationPermissionMismatch** with a SAS | SAS lacks a permission letter (for example no `l` to list) | Generate a SAS with the right permissions |
| "Key based authentication is not permitted on this storage account" | **Shared key access** is disabled | Use Entra ID (`--auth-mode login`) or user delegation SAS |
| **PublicAccessNotPermitted** | Anonymous access is off on the **account** | Allow it on the account *and* the container – or better, use a SAS |
| App suddenly fails after key rotation | App used the rotated key | Update the app (or Key Vault secret) to the other key |

**Debug order:** network first (firewall) → authentication (key/SAS/token valid?) → authorization (role or SAS permissions).

## Clean up
```bash
az role assignment delete --assignee cat@contoso.com --role "Storage Blob Data Reader" \
  --scope "$SA_ID/blobServices/default/containers/data"
az storage container policy delete --account-name $SA --account-key $KEY --container-name data --name partner-read
az storage account keys renew -n $SA -g $RG --key key1   # invalidates SAS you shared in the lab
```

## Check yourself
<details><summary>1. A partner needs to upload files to one container for 2 weeks. You want to be able to cut them off at any time without affecting other apps. What do you use?</summary>

A **service SAS** linked to a **stored access policy** on that container. Delete the policy to revoke.
</details>

<details><summary>2. A VM's app must read blobs without any secrets in code.</summary>

Enable the VM's **managed identity** and give it **Storage Blob Data Reader** on the container.
</details>

<details><summary>3. A user with Contributor gets AuthorizationPermissionMismatch from `az storage blob list --auth-mode login`. Fix?</summary>

Give them a **data role** (Storage Blob Data Reader). Contributor is control plane only.
</details>

## Related
- [Managed Identities](../02-identity/managed-identities.md)
- [Azure RBAC](../04-governance/azure-rbac.md)
- [Storage Network Security](storage-network-security.md)
- [Storage Encryption and Key Vault](storage-encryption-key-vault.md)
