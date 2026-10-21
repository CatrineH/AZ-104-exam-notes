# Groups and Membership

[← Back to README](../README.md) · Lab 02

## In one line
Put users in **groups** and give access, roles, and licenses to the **group** – then access follows people automatically as they join or leave.

## When to use it (and when not)
| Situation | Use |
|---|---|
| Give access to Azure resources, apps, or licenses | **Security group** |
| Team collaboration (mailbox, Teams, SharePoint) | **Microsoft 365 group** |
| Membership should follow a user property (department, job title) | **Dynamic user** group (P1) |
| Group of devices for policies (Intune, Conditional Access) | **Dynamic device** group (P1) |
| Small, hand-picked list | **Assigned** membership |
| Assign **Entra roles** to a group | **Role-assignable** group (set when created) |

## Key concepts

### Group types

| Type | Used for |
|---|---|
| **Security group** | Giving access to resources (RBAC roles, apps, licenses) |
| **Microsoft 365 group** | Collaboration – shared mailbox, calendar, Teams, SharePoint |

### Membership types

| Type | How members are added | License |
|---|---|---|
| **Assigned** | Manually, by an admin or group owner | Free |
| **Dynamic user** | Automatically, by a rule on user properties | **Entra ID P1** |
| **Dynamic device** | Automatically, by a rule on device properties | **Entra ID P1** |

**Dynamic group rule examples:**
```
user.department -eq "Development"
(user.department -eq "Sales") -and (user.country -eq "Norway")
user.jobTitle -contains "Engineer"
device.deviceOSType -eq "Windows"
```
- You **can't add or remove members manually** in a dynamic group
- One group is either user-based or device-based, not both
- Rule changes can take a while to process in big tenants

### Group owners
- A **group owner** can manage the group's members **without being an administrator**
- Useful for letting team leads manage their own team's access

### Why use groups for access?
Assign RBAC roles and licenses to **groups**, not individual users. When someone joins or leaves the group, their access follows automatically. This also makes [access reviews](access-review.md) work well.

## Exam traps
- **Dynamic groups require Entra ID P1**
- Deleted **Microsoft 365 groups** can be restored for **30 days**. Deleted **security groups can't**
- Group owners manage members, not the group's role assignments
- You **can't change** a group between Assigned and Dynamic and keep manual members – dynamic overwrites membership
- **Group-based licensing** doesn't pass licenses through **nested** groups
- Only **role-assignable** groups (`isAssignableToRole`, set **at creation**) can get **Entra roles**. Azure RBAC roles work with any security group
- "Give access to resources/apps/licenses" → the exam answer is almost always a **security group**. Microsoft 365 groups are for collaboration

## Do it

### Portal
- **Entra ID → Groups → + New group** → type Security → membership **Assigned** / **Dynamic user** → **Add dynamic query** → rule builder → **Validate rules** with a test user
- **Group → Members / Owners → + Add**
- **Groups → Deleted groups** (Microsoft 365 only)

### Azure CLI
```bash
az ad group create --display-name "sg-developers" --mail-nickname sg-developers
az ad group member add --group sg-developers \
  --member-id $(az ad user show --id sarah@contoso.onmicrosoft.com --query id -o tsv)
az ad group member list --group sg-developers -o table
az ad group owner add --group sg-developers --owner-object-id <user-object-id>

# Give the group an Azure role
az role assignment create --assignee $(az ad group show -g sg-developers --query id -o tsv) \
  --role Contributor --resource-group rg-dev
```
Dynamic groups aren't supported by `az ad group create` – use the portal or Graph.

### PowerShell
```powershell
# Az module
New-AzADGroup -DisplayName sg-developers -MailNickname sg-developers
Add-AzADGroupMember -TargetGroupDisplayName sg-developers -MemberUserPrincipalName sarah@contoso.onmicrosoft.com

# Microsoft Graph – dynamic group
Connect-MgGraph -Scopes "Group.ReadWrite.All"
New-MgGroup -DisplayName "dg-development" -MailEnabled:$false -MailNickname dg-development `
  -SecurityEnabled -GroupTypes "DynamicMembership" `
  -MembershipRule 'user.department -eq "Development"' -MembershipRuleProcessingState On
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Dynamic" option greyed out | No **P1** license in the tenant | Get P1 (or trial) |
| User not in the dynamic group | Property doesn't match exactly (spelling, spaces), or processing is still running | **Validate rules** with that user, check the property value, wait |
| Can't add a member manually | Group is **dynamic** | Change the user property instead |
| User has no access after joining the group | Role assignment still propagating, or user's token is old | Wait a few minutes, sign out/in |
| Group doesn't appear when assigning an Entra role | Group isn't **role-assignable** | Create a new group with "Microsoft Entra roles can be assigned" = Yes |
| Deleted group can't be restored | It was a **security** group | Recreate it, re-add members and roles |

## Clean up
```bash
az ad group delete --group sg-developers
```

## Check yourself
<details><summary>1. All users in the Sales department must automatically get access to a resource group. What do you set up?</summary>

A **dynamic user security group** (`user.department -eq "Sales"`, needs P1) with an RBAC role on the resource group.
</details>

<details><summary>2. A security group was deleted yesterday. Can you restore it?</summary>

No – only **Microsoft 365** groups can be restored.
</details>

<details><summary>3. A team lead must add and remove members of the team's group, without being an admin.</summary>

Make them **group owner**.
</details>

## Related
- [Entra Users](entra-users.md)
- [Entra Licenses](entra-licenses.md)
- [Azure RBAC](../04-governance/azure-rbac.md)
- [Access Reviews](access-review.md)
