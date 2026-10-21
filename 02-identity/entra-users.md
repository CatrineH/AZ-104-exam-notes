# Entra ID and Users

[← Back to README](../README.md) · Lab 01

## In one line
**Microsoft Entra ID** is the cloud directory that holds your users, groups, and apps. As an admin you **create, block, delete, and restore** users – one at a time or in bulk.

## When to use it (and when not)
| Situation | Action |
|---|---|
| New employee | **Create** user (or sync from on-prem AD with Entra Connect) |
| Many new users at once | **Bulk create** with CSV |
| Employee on leave, or suspected compromise | **Block sign-in** (reversible, keeps everything) |
| Employee left | **Delete** (restorable for 30 days) |
| Deleted by mistake | **Restore** from Deleted users (within 30 days) |
| External partner | Don't create a member – **invite a guest** ([guest-users.md](guest-users.md)) |

## Key concepts

### User lifecycle

| Action | What happens |
|---|---|
| **Create** | One user at a time, or **bulk create** with a CSV file |
| **Block sign-in** | The user still exists but can't sign in. Good when someone leaves or is on leave |
| **Delete** | The user moves to **Deleted users** |
| **Restore** | Possible for **30 days** after deletion |
| **Permanently delete** | Happens automatically after 30 days, or manually from Deleted users |

### Bulk operations
- **Bulk create**, **bulk invite** (guests), and **bulk delete** use a **CSV template downloaded from the portal**
- Portal: **Users → Bulk operations → Download** the template, fill it in, then upload it
- Check **Bulk operation results** afterwards for rows that failed

### User types

| Type | Who |
|---|---|
| **Member** | Employees in your own tenant |
| **Guest** | External users invited through B2B (see [guest-users.md](guest-users.md)) |

### Source of the user
| Source | Where you manage it |
|---|---|
| **Cloud only** | In Entra ID |
| **Synced from on-premises AD** (Entra Connect) | In **on-premises AD** – many properties are read-only in Entra |

### Important properties
- **User principal name (UPN):** the sign-in name, `cat@contoso.com`. The domain must be a **verified domain** in the tenant
- **Usage location:** country – **required before assigning a license**
- **Job title, department, manager:** used by **dynamic groups** and access reviews

## Exam traps
- **Usage location must be set before a license can be assigned.** Licensing rules differ between countries
- Deleted users can be restored for **30 days** – after that, they're gone
- Blocking sign-in is reversible and keeps the user's group memberships and roles
- A **synced** user can't be edited in Entra (name, department…) – change it in on-prem AD
- UPN with a domain that isn't **verified** → you can't use it. Add and verify the custom domain first (TXT/MX record)
- Creating users needs an **Entra role** like **User Administrator** – not an Azure RBAC role

## Do it

### Portal
- **Microsoft Entra ID → Users → + New user → Create new user** → UPN, display name, password → **Properties** tab: job title, department, usage location
- **User → Edit properties → Settings → Account enabled** (untick to block sign-in)
- **Users → Bulk operations → Bulk create**
- **Users → Deleted users → Restore users**

### Azure CLI
```bash
az ad user create --display-name "Sarah Hansen" \
  --user-principal-name sarah@contoso.onmicrosoft.com \
  --password 'TempP@ss123!' --force-change-password-next-sign-in true

az ad user list --query "[].{name:displayName, upn:userPrincipalName}" -o table
az ad user show --id sarah@contoso.onmicrosoft.com

# Block sign-in
az ad user update --id sarah@contoso.onmicrosoft.com --account-enabled false

az ad user delete --id sarah@contoso.onmicrosoft.com
```

### PowerShell
Two modules you'll see: **Az** (`New-AzADUser`) for basic work, and **Microsoft Graph** (`New-MgUser`) for everything in Entra – licenses, usage location, deleted items.

```powershell
# Az module
New-AzADUser -DisplayName "Sarah Hansen" -UserPrincipalName sarah@contoso.onmicrosoft.com `
  -MailNickname sarah -Password (ConvertTo-SecureString 'TempP@ss123!' -AsPlainText -Force)

# Microsoft Graph module
Connect-MgGraph -Scopes "User.ReadWrite.All","Directory.ReadWrite.All"

New-MgUser -DisplayName "Sarah Hansen" -UserPrincipalName sarah@contoso.onmicrosoft.com `
  -MailNickname sarah -AccountEnabled -UsageLocation NO -Department Development `
  -PasswordProfile @{ Password = 'TempP@ss123!'; ForceChangePasswordNextSignIn = $true }

Update-MgUser -UserId sarah@contoso.onmicrosoft.com -AccountEnabled:$false   # block sign-in

# Deleted users and restore
Get-MgDirectoryDeletedItemAsUser
Restore-MgDirectoryDeletedItem -DirectoryObjectId <object-id>
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "License cannot be assigned… usage location" | Usage location empty | Set **usage location** first |
| Can't change a user's department / name | User is **synced** from on-prem AD | Change it in on-prem AD |
| UPN domain not in the list | Custom domain not **verified** | Add domain, create the TXT record, verify |
| Can't find a deleted user to restore | More than **30 days** passed, or permanently deleted | Recreate the user |
| You can't create users | You only have Azure RBAC roles (Owner) | Need **User Administrator** (Entra role) |
| Bulk create partly failed | Bad CSV rows (UPN taken, missing field) | Download the **results file**, fix those rows |
| Blocked user still signed in | Existing tokens are still valid | **Revoke sessions** on the user |

## Clean up
```bash
az ad user delete --id sarah@contoso.onmicrosoft.com
```
Then **Deleted users → Delete permanently** if you want the UPN free right away.

## Check yourself
<details><summary>1. An employee goes on 6 months' leave. Access must stop, but nothing should be lost.</summary>

**Block sign-in**. The account, groups, and roles stay.
</details>

<details><summary>2. A license assignment fails for a new user. What's the first thing to check?</summary>

**Usage location**.
</details>

<details><summary>3. A user was deleted 45 days ago. Can you restore them?</summary>

No – restore only works for **30 days**.
</details>

## Related
- [Groups](groups.md)
- [Entra Licenses](entra-licenses.md)
- [Entra Roles vs Azure RBAC](entra-roles-vs-rbac.md)
