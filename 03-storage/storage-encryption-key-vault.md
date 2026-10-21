# Storage Encryption and Azure Key Vault

[← Back to README](../README.md)

## In one line
Storage data is **always encrypted at rest** – you only choose **who manages the key**: Microsoft (default) or you, with a key in **Azure Key Vault**.

## When to use it (and when not)
| Situation | Use |
|---|---|
| No special requirement | **Microsoft-managed keys** (default, nothing to do) |
| Compliance says the company must own, rotate, and be able to revoke the key | **Customer-managed keys (CMK)** in Key Vault |
| Need two layers of encryption | **Infrastructure encryption** (must be chosen when the account is created) |
| Different keys for different containers/customers in one account | **Encryption scopes** |
| Store passwords, connection strings, certificates | **Key Vault secrets/certificates** |

CMK adds work (Key Vault, roles, rotation). Don't choose it without a reason.

## Key concepts

### Encryption at rest
All data in Azure Storage is **always encrypted at rest** with 256-bit AES. I can't turn it off.

| Option | Key managed by | Notes |
|---|---|---|
| **Microsoft-managed keys** | Microsoft | **The default.** Nothing to configure |
| **Customer-managed keys (CMK)** | **I**, in **Azure Key Vault** (or Managed HSM) | I control rotation, and can revoke access to the data |

Why choose customer-managed keys?
- **Compliance** rules that require the company to own and control the keys
- **Control over rotation** – when and how often
- **Revoke access:** disable the key, and the storage data becomes **inaccessible**

**Encryption in transit** is separate: **Secure transfer required** (HTTPS only) and **minimum TLS version 1.2** on the account.

### Azure Key Vault

| Object | Example |
|---|---|
| **Keys** | Encryption keys, for example for storage CMK |
| **Secrets** | Passwords, connection strings, API keys |
| **Certificates** | TLS/SSL certificates |

**Two permission models:**

| Model | How access is given |
|---|---|
| **Azure RBAC** (recommended) | Role assignments, just like other Azure resources |
| **Vault access policy** (older) | Permissions set inside the vault |

**Control plane vs data plane:**

| | Control plane | Data plane |
|---|---|---|
| Is about | **The vault itself** – create, delete, settings, network | **What's inside** – keys, secrets, certificates |
| Example roles | Owner, Contributor, Key Vault Contributor | Key Vault Crypto Officer, Key Vault Secrets User |

A **Contributor can create the vault**, but **can't create keys** in it. A **data plane role** is needed for that.

| Role | Can |
|---|---|
| **Key Vault Administrator** | Everything on the data plane (keys, secrets, certificates) |
| **Key Vault Crypto Officer** | Create and manage **keys** |
| **Key Vault Crypto Service Encryption User** | **Use** a key to encrypt and decrypt – for a **service's managed identity** |
| **Key Vault Secrets Officer** | Create and manage **secrets** |
| **Key Vault Secrets User** | **Read** secrets |

**Network:** Key Vault has its own **firewall**, just like storage. If I restrict the network, turn on **"Allow trusted Microsoft services to bypass this firewall"** so Azure Storage can still reach the key.

### How customer-managed keys work with storage

```
Storage account
  │ system-assigned (or user-assigned) managed identity
  │ role: Key Vault Crypto Service Encryption User
  ▼
Key Vault ── key: cmk-storage
```

1. The storage account gets a **managed identity** – in the portal, the system-assigned identity is enabled automatically when I choose CMK
2. The identity gets the role **Key Vault Crypto Service Encryption User** on the vault (or the key)
3. The storage account uses the key to **protect its own encryption key** (key wrapping). My data isn't re-encrypted

**Requirements:**
- The Key Vault must have **soft delete** and **purge protection** enabled
- Key type: **RSA** or **RSA-HSM** (2048, 3072, or 4096 bits)

### Key rotation
- **Rotating a key creates a new version** – the key name stays the same, the old versions remain
- With **automatic key version updates** (no version chosen), the storage account **starts using the newest version automatically**, normally within 24 hours
- If I chose a **specific version**, I must update the storage account manually after rotating
- Rotation doesn't re-encrypt the data, so it's quick
- A **rotation policy** in Key Vault can rotate keys automatically, for example every 90 days

### Revoking access
Disable or delete the key, or remove the role assignment → the storage account can't use the key → **data can't be read** until access is restored.

## Exam traps
- Encryption at rest is **always on** – CMK only changes who manages the key
- CMK needs **soft delete + purge protection** on the Key Vault. **Purge protection can't be turned off** again
- The storage account's **managed identity** needs **Key Vault Crypto Service Encryption User**
- Contributor = control plane only. Creating keys needs **Key Vault Crypto Officer** (or Administrator)
- Rotation = **new version** of the same key. Pinned version = manual update
- **Infrastructure encryption** can only be enabled **when the account is created**
- Key Vault firewall on → tick **trusted Microsoft services**, or storage can't reach the key
- To set CMK **while creating a new account**, you must use a **user-assigned** identity – the system-assigned one doesn't exist until the account does

## Do it

### Portal
1. **Key Vault → Create** → permission model **Azure RBAC**, **purge protection = Enable**
2. Key Vault → **Access control (IAM)** → give yourself **Key Vault Crypto Officer**
3. **Objects → Keys → + Generate/Import** → RSA 3072
4. Storage account → **Security + networking → Encryption** → **Customer-managed keys** → select vault and key → identity: system-assigned → **Save**
5. Key → **Rotation policy** → rotate every 90 days

Full step-by-step: [Lab: Storage Encryption with CMK](lab/storage-cmk-lab.md).

### Azure CLI
```bash
az keyvault create -n $KV -g $RG -l norwayeast \
  --enable-rbac-authorization true --enable-purge-protection true
az keyvault key create --vault-name $KV --name cmk-storage --kty RSA --size 3072

az storage account update -n $SA -g $RG --assign-identity
az storage account update -n $SA -g $RG \
  --encryption-key-source Microsoft.Keyvault \
  --encryption-key-vault https://$KV.vault.azure.net \
  --encryption-key-name cmk-storage

# Which key is in use?
az storage account show -n $SA -g $RG --query encryption.keyVaultProperties

# Secure transfer + TLS
az storage account update -n $SA -g $RG --https-only true --min-tls-version TLS1_2

# Store and read a secret
az keyvault secret set --vault-name $KV --name sql-password --value 'P@ssw0rd!'
az keyvault secret show --vault-name $KV --name sql-password --query value -o tsv
```

### PowerShell
```powershell
New-AzKeyVault -Name $kv -ResourceGroupName $rg -Location norwayeast -EnablePurgeProtection
Add-AzKeyVaultKey -VaultName $kv -Name cmk-storage -Destination Software -KeyType RSA -Size 3072

Set-AzStorageAccount -ResourceGroupName $rg -Name $sa -AssignIdentity
Set-AzStorageAccount -ResourceGroupName $rg -Name $sa -KeyvaultEncryption `
  -KeyVaultUri "https://$kv.vault.azure.net" -KeyName cmk-storage

Set-AzKeyVaultSecret -VaultName $kv -Name sql-password `
  -SecretValue (ConvertTo-SecureString 'P@ssw0rd!' -AsPlainText -Force)
Get-AzKeyVaultSecret -VaultName $kv -Name sql-password -AsPlainText
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| **Forbidden** when creating a key or secret | You have Owner/Contributor but no **data plane** role | Key Vault Crypto Officer / Secrets Officer. Wait a few minutes |
| "Purge protection must be enabled" | Vault lacks purge protection | Key Vault → Properties → enable purge protection |
| Storage can't access the key / CMK save fails | Managed identity missing the **Crypto Service Encryption User** role, or **Key Vault firewall** blocks it | Assign role, allow trusted Microsoft services |
| All storage data suddenly inaccessible | Key **disabled/deleted/expired**, or role removed | Re-enable key, restore from soft delete, re-add role |
| Still the old key version after rotation | Account is pinned to a **specific version**, or < 24 h passed | Update account to the new version, or remove version for auto-update |
| "Vault name already in use" after deleting it | Vault is **soft-deleted** | `az keyvault recover` or wait for retention to end (purge is blocked by purge protection) |

## Clean up
Switch storage back to Microsoft-managed keys **before** deleting the vault, or the data becomes unreadable:
```bash
az storage account update -n $SA -g $RG --encryption-key-source Microsoft.Storage
az keyvault delete -n $KV -g $RG     # stays soft-deleted for the retention period
```

## Check yourself
<details><summary>1. An admin with Contributor on the subscription created a Key Vault but can't create a key. Why?</summary>

Contributor is control plane. Creating keys needs **Key Vault Crypto Officer** (data plane) when the vault uses Azure RBAC.
</details>

<details><summary>2. The company wants to make storage data unreadable immediately if they leave Azure. How?</summary>

Use **CMK**. Disabling the key (or removing the identity's role) makes the data inaccessible.
</details>

<details><summary>3. After rotating the key, the storage account still uses the old version a week later. Why?</summary>

The CMK config uses a **specific key version**. Update it, or remove the version to get automatic updates.
</details>

## Related
- [Lab: Storage Encryption with Customer-Managed Keys](lab/storage-cmk-lab.md)
- [Managed Identities](../02-identity/managed-identities.md)
- [Azure RBAC](../04-governance/azure-rbac.md)
- [Storage Network Security](storage-network-security.md)
