# AZ-104 Role Cheat Sheet — ARM, Entra ID and Data Plane

[← Back to README](../README.md) · See also [Azure RBAC](azure-rbac.md) · [Entra Roles vs RBAC](../02-identity/entra-roles-vs-rbac.md)

*Version 1.1 — October 2026 · Orange Business Azure Trainee Programme*

**Why this file exists:** many AZ-104 questions come down to one thing: *"Which role, at which scope, gives exactly this permission and nothing more?"* To answer, you need to know two things: **which plane** the task lives on, and **the smallest role** that covers it.

**How to study this file**

| Tier | What | How much effort |
|---|---|---|
| **Tier 1** | 25 roles | **Know them cold** |
| **Tier 2** | Naming patterns | **Learn the patterns** — they decode roles you've never seen |
| **Tier 3** | Everything else (appendix) | **Recognise only** — don't memorise |

> The exam regularly includes roles you haven't studied. Someone who knows Tier 1 and the Tier 2 patterns will usually beat someone who half-memorised a list of 60+ roles.

---

## 0. The three planes at a glance

| | **ARM (control plane)** | **Entra ID** | **Data plane** |
|---|---|---|---|
| **Controls** | Azure *resources*: create, configure, delete | The *directory*: users, groups, apps, licences | The *content inside* a resource: blobs, files, secrets, VM sign-in |
| **Role system** | Azure RBAC — `Actions/NotActions` | Entra ID roles (a separate system) | Azure RBAC — `DataActions/NotDataActions` |
| **Scope** | Management group · subscription · resource group · resource | Tenant, or an **administrative unit** | Usually the resource or below (e.g. a container or file share) |
| **Assigned in** | Resource → **Access control (IAM)** | Entra admin center → **Roles & admins** | Resource → **Access control (IAM)** |
| **Example** | Create a storage account | Create a user | Read a blob in that storage account |

> **Key point:** data-plane roles are **Azure RBAC roles too**. They are assigned in the same IAM blade as ARM roles; they just contain `DataActions`. Entra ID roles are the *only* truly separate system.

**Analogy (keycards):**
- **ARM roles** open the doors to rooms and let you move, add or remove the furniture.
- **Data roles** let you open the filing cabinets inside and read or change the papers.
- **Entra ID roles** are like HR: they decide who works here and who gets a keycard at all.

---

## 1. Tier 1 — know these cold (25 roles)

### 1.1 ARM — control plane (11)

| Role | Can | Cannot | Exam trigger |
|---|---|---|---|
| **Owner** | Everything, including assigning roles | — | "Full control *and* manage access" — rarely the least-privilege answer |
| **Contributor** | Create and manage all resources | Assign roles, manage locks or policy | "Manage all resources", nothing about access |
| **Reader** | View everything | Change anything; list storage keys | "View only", auditors |
| **User Access Administrator** | Everything under Microsoft.Authorization/*: role assignments, custom roles, locks — and policy (but better to use Resource Policy Contributor for that)| Manage resources | "Manage access"; "remove a lock" (with Owner) |
| **Role Based Access Control Administrator** | Create and delete role assignments only. Can be limited with **conditions** | Custom roles, locks, policy, resources | "Least privilege to assign roles"; "may only assign *specific* roles" |
| **Virtual Machine Contributor** | Create and manage VMs and their disks | Access management; the VNet or storage account the VM uses; **signing in to the VM** | "Manage VMs but not access" |
| **Network Contributor** | Manage VNets, subnets, NSGs, peerings, load balancers | Access management | "Manage networking only" |
| **Storage Account Contributor** | Manage storage accounts — **including listing the access keys** | Access management | "Manage storage accounts" (see Trap 2) |
| **Tag Contributor** | Manage tags | Change the resources themselves | "Update tags only" |
| **Cost Management Contributor** | View costs; manage budgets and exports | Change resources | "Create budgets" |
| **Cost Management Reader** | View costs and budgets | Change anything | "View costs only" |

### 1.2 Entra ID — directory (5 + administrative units)

These are **not** Azure RBAC roles. They don't appear in the IAM blade and give **no access to Azure resources**.

| Role | Can | Exam trigger |
|---|---|---|
| **Global Administrator** | Everything in Entra ID. **No Azure resource access by default** — must *elevate access* to get User Access Administrator at root scope `/` | "Can't see any subscriptions" → elevate access |
| **User Administrator** | Create and manage users and groups; reset passwords for non-admins and some limited admins | "Create users and reset passwords" |
| **Helpdesk Administrator** | Reset passwords for non-admins and some limited admins | "The service desk resets passwords" |
| **License Administrator** | Assign and remove licences on users and groups, including group-based licensing | "Assign licences only" |
| **Guest Inviter** | Invite B2B guest users, even when ordinary members can't | "Invite external users only" |

**Administrative units (AU).** An AU restricts an Entra role to a *subset* of users, groups or devices, such as one country office. It is the Entra equivalent of a narrower scope.

> **Exam trigger:** *"The Oslo service desk may reset passwords only for Oslo users"* → an administrative unit for Oslo, with **Helpdesk Administrator scoped to that AU**.

### 1.3 Data plane — content (9)

**Blob storage**

| Role | Can | Exam trigger |
|---|---|---|
| **Storage Blob Data Owner** | Everything Contributor can, **plus** set POSIX ACLs (ADLS Gen2) | "Full control of blob data, including ACLs" |
| **Storage Blob Data Contributor** | Read, write and delete blobs and containers | "Upload and delete blobs" |
| **Storage Blob Data Reader** | Read blobs and list containers | "Read blob data only" |

**Azure Files (identity-based access)**

Access is checked in **two layers**: first the **share-level** RBAC role, then the **NTFS ACLs** on folders and files. A user needs both.

| Role | Can | Exam trigger |
|---|---|---|
| **Storage File Data SMB Share Contributor** | Read, write and delete files over SMB | "Users must edit files on the share" |
| **Storage File Data SMB Share Elevated Contributor** | As Contributor, **plus modify NTFS ACLs** | "Admins must change NTFS permissions" |

**Key Vault** (only when the vault uses the **Azure RBAC** permission model)

| Role | Can | Exam trigger |
|---|---|---|
| **Key Vault Secrets User** | **Read secret values** — nothing more | "An app / managed identity must read a secret" |

**Virtual machine sign-in with Entra ID**

| Role | Can | Exam trigger |
|---|---|---|
| **Virtual Machine Administrator Login** | Sign in to the VM with Entra ID **as a local administrator** | "Admins sign in to the VM with Entra ID" |

**Azure Container Registry — learn these two together**

| Role | Can | Exam trigger |
|---|---|---|
| **AcrPull** | **Pull** images | AKS, App Service or Container Apps must pull images → give the workload's **managed identity** AcrPull |
| **AcrPush** | **Push and pull** images — AcrPush is a **superset** of AcrPull | A CI/CD pipeline builds and pushes images → AcrPush |

> **Rule:** pull only → **AcrPull**. Push (with or without pull) → **AcrPush**. Never assign both to the same identity, because AcrPush already includes pull. Least privilege: don't give a cluster AcrPush "just in case".
>
> *Note:* both roles are technically defined with `Actions`, not `DataActions`. They still grant access to the registry's *content*, which is why they are listed here.

---

## 2. Tier 2 — decode any role from its name

| Pattern | Meaning | Examples |
|---|---|---|
| ***X* Contributor** | Manage resources of type X. **Can't grant access** | Backup Contributor, Monitoring Contributor, Website Contributor |
| ***X* Reader** | View X only | Monitoring Reader, Backup Reader, Billing Reader |
| ***X* Operator** | Run X's day-to-day operations, but not create, delete or grant | Backup Operator, Managed Identity Operator |
| **"Data" in the name** | **Data plane** — the content, not the resource | Storage Queue Data Reader, Storage Table Data Contributor |
| **Owner > Contributor > Reader** | The same ladder in every family. Owner usually adds ACL or access control on top | Storage Blob Data Owner → Contributor → Reader |
| **Key Vault *Officer* vs *User*** | Officer **manages** the object; User only **uses or reads** it | Secrets Officer vs Secrets User; Crypto Officer vs Crypto User |
| **"Administrator" with no Azure service in the name** | Almost always an **Entra ID** role | User Administrator, Groups Administrator, Password Administrator |
| **"Administrator" with an Azure service in the name** | An Azure RBAC role with broad rights to that service | Key Vault Administrator |
| **"Login" in the name** | Signing in to a resource | Virtual Machine User Login |
| **Push vs Pull** | Push includes pull | AcrPush ⊃ AcrPull |

**Try it:** you've never seen *Storage Queue Data Message Sender*. Data → content. Queue → messages. Sender → can only *add* messages. That's all you need on exam day.

---

## 3. The traps the exam loves ★★★

1. **Owner ≠ data access.** A storage account Owner can't read blobs with **Entra ID authentication** without a *Storage Blob Data* role.
2. **…but Owner, Contributor and Storage Account Contributor can list the account keys,** and a key opens all the data. That's why disabling **shared key access** matters in production.
3. **Key Vault Contributor manages the vault, not its secrets.** Under the older access policy model, though, it can edit access policies and grant itself access. Another reason to prefer the RBAC model.
4. **Global Administrator ≠ Azure access.** Elevate access first.
5. **Subscription Owner ≠ directory admin.** An Owner can't create users; that needs an Entra role such as User Administrator.
6. **Reader can't list storage keys.** `listKeys` is a POST action, not a read.
7. **VM Contributor can't sign in to the VM with Entra ID.** That needs *Virtual Machine Administrator Login* or *User Login*.
8. **Contributor can't assign roles, locks or policies.** All three live under `Microsoft.Authorization`.
9. **Wrong plane = wrong answer.** If the task is about *content* (blobs, secrets, messages, files), the answer is almost always a role with *Data* in its name, a Key Vault data role, or AcrPull/AcrPush.
10. **Least privilege beats "it works".** Contributor *works* for almost everything, which is exactly why it's usually the wrong answer.

---

## 4. Quick decision table — "Which role?"

| The requirement says … | Plane | Role |
|---|---|---|
| Create users and reset their passwords | Entra ID | User Administrator |
| Reset passwords only, for the Oslo office | Entra ID | Helpdesk Administrator, scoped to an administrative unit |
| Assign licences | Entra ID | License Administrator |
| Invite external partners | Entra ID | Guest Inviter |
| Admin can't see any subscriptions | Entra ID → ARM | Global Administrator → **elevate access** |
| Manage VMs, nothing else | ARM | Virtual Machine Contributor |
| Manage VNets and NSGs in 25 subscriptions | ARM | Network Contributor at the management group |
| Grant access to others — least privilege | ARM | Role Based Access Control Administrator |
| Grant only Reader to others | ARM | RBAC Administrator **with a condition** |
| Remove a resource lock | ARM | Owner or User Access Administrator |
| Update tags only | ARM | Tag Contributor |
| Create budgets | ARM | Cost Management Contributor |
| Upload blobs with an Entra identity | Data | Storage Blob Data Contributor |
| An app reads a connection string from Key Vault | Data | Key Vault Secrets User (on the app's managed identity) |
| Users edit files on an Azure Files share | Data | Storage File Data SMB Share Contributor |
| Admins change NTFS permissions on the share | Data | Storage File Data SMB Share Elevated Contributor |
| Admins sign in to a VM with Entra ID | Data | Virtual Machine Administrator Login |
| AKS pulls images from ACR | Data | AcrPull (on the cluster's identity) |
| A pipeline builds and pushes images to ACR | Data | AcrPush (includes pull) |

---

## 5. How to check a role yourself

- **Portal:** Access control (IAM) → **Roles** → pick a role → **View** → **Permissions** / **JSON**. This shows `Actions`, `NotActions`, `DataActions` and `NotDataActions`.
- **CLI:** `az role definition list --name "Storage Blob Data Contributor"`
- **Entra roles:** Entra admin center → **Roles & admins** → pick a role → **Description**.
- **Reference:** Microsoft Learn — *Azure built-in roles* and *Microsoft Entra built-in roles*. Role names and permissions change over time; when in doubt, these pages win over this file.

---

## Appendix — Tier 3: recognise only

**Don't memorise this list.** Read it once, and check that the Tier 2 patterns explain each name. Roles marked ◆ have appeared in exam-style questions more often than the rest — worth a second look.

### ARM

| Role | One line |
|---|---|
| Backup Contributor | Manage backups; can't create vaults or grant access |
| Backup Operator ◆ | Like Backup Contributor, but also can't **remove** backups |
| Backup Reader | View backup management |
| Monitoring Contributor | Read monitoring data and edit monitoring settings (alerts, diagnostic settings) |
| Monitoring Reader | Read all monitoring data |
| Log Analytics Contributor / Reader | Manage / read Log Analytics workspaces and their data |
| Website Contributor | Manage web apps, but not their App Service plans |
| Web Plan Contributor | Manage App Service plans, but not the web apps |
| Key Vault Contributor ◆ | Manage the vault resource — **not** its secrets (Trap 3) |
| Resource Policy Contributor ◆ | Manage policy definitions and assignments |
| Billing Reader | Read billing data |
| Managed Identity Operator | Assign user-assigned managed identities to resources |

### Entra ID

| Role | One line |
|---|---|
| Privileged Role Administrator | Manage Entra role assignments, administrative units and PIM |
| Groups Administrator | Manage groups and group settings |
| Password Administrator | Reset passwords for non-admins (narrower than Helpdesk) |
| Authentication Administrator | Manage non-password authentication methods, e.g. MFA re-registration |
| Application Administrator | Manage app registrations and enterprise apps, including Application Proxy |
| Cloud Application Administrator | As Application Administrator, without Application Proxy |
| Conditional Access Administrator | Manage Conditional Access policies |
| Security Administrator / Security Reader | Manage / read security settings and reports |
| Global Reader | Read everything a Global Administrator can see; change nothing |
| Hybrid Identity Administrator | Manage Entra Connect / cloud sync and federation |

### Data plane

| Role | One line |
|---|---|
| Storage Blob Delegator | Get a user delegation key to create a user delegation SAS |
| Storage Queue Data Contributor / Reader | Read, write, delete / read queue messages |
| Storage Queue Data Message Sender / Processor | Only add / only read and delete queue messages |
| Storage Table Data Contributor / Reader | Read-write / read table data |
| Storage File Data SMB Share Reader | Read files on the share over SMB |
| Key Vault Administrator | All data operations on keys, secrets and certificates |
| Key Vault Secrets Officer | Create, update and delete secrets |
| Key Vault Crypto Officer / Crypto User | Manage keys / use keys (encrypt, decrypt, sign) |
| Key Vault Certificates Officer | Manage certificates |
| Key Vault Reader | Read vault and object **metadata** — not secret values |
| Virtual Machine User Login | Sign in to the VM with Entra ID as a regular user |
