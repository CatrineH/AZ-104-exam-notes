# Privileged Identity Management (PIM)

[← Back to README](../README.md) · Lab 11

## In one line
**PIM** gives **just-in-time** admin access: users are **eligible** for a role and **activate** it only when needed, for a limited time, with MFA/approval/justification.

## When to use it (and when not)
| Situation | Use |
|---|---|
| Admins shouldn't be Global Admin / Owner 24/7 | **PIM eligible** assignments |
| Activating Owner on production must be approved | PIM role setting: **Require approval** |
| Contractor needs Contributor for exactly 3 months | PIM **time-bound** assignment (start/end date) |
| Regular check of who holds admin roles | PIM **access reviews** |
| Normal, everyday access for regular users | Plain RBAC/Entra assignment – PIM is for **privileged** roles |

**Requires Entra ID P2** (or Microsoft Entra ID Governance).

## Key concepts

### Assignment types

| Type | Meaning |
|---|---|
| **Eligible** | *Can* activate the role when needed – no access until then |
| **Active** | *Has* the access right now (permanently, or until an end date) |

### Activation requirements (role settings, can be combined)
- **MFA**
- **Approval** from a chosen approver
- **Justification** (a reason)
- **Ticket number**
- **Maximum activation duration**, for example 1–8 hours (max 24)

### Two sections in PIM

| Section | Manages | Assigned at |
|---|---|---|
| **Microsoft Entra roles** | Directory roles, such as **Application Administrator** or Global Administrator | Directory (or administrative unit) |
| **Azure resources** | RBAC roles, such as Owner or Contributor | Management group, subscription, resource group, resource |
| **Groups** (PIM for Groups) | Membership/ownership of a group | The group |

**Application Administrator is an Entra role, not an RBAC role.** It can't be assigned on a resource group – it's assigned in the **Microsoft Entra roles** section at directory level.

## Exam traps
- PIM = **P2**
- **Eligible** = can activate, **Active** = has access now
- "Admins need access only when needed, with approval" → **PIM**, not Conditional Access
- **Approval** is set in **role settings**, per role
- After activation, the user may need to **sign out and in** (or refresh) to get the new token
- PIM handles **both** Entra roles and Azure roles – but in **separate** sections
- To use PIM on an Azure resource, it appears automatically in **Azure resources** when you have Owner/User Access Administrator on it

## Do it

### Portal
1. **Entra ID → Identity Governance → Privileged Identity Management** (or search "PIM")
2. **Microsoft Entra roles → Roles → User Administrator → + Add assignments** → member → **Assignment type: Eligible** → duration
3. **Settings → User Administrator → Edit** → activation max duration, **Require MFA**, **Require justification**, **Require approval** + approvers
4. As the user: **PIM → My roles → Eligible assignments → Activate** → reason → wait for approval
5. As approver: **PIM → Approve requests**
6. **Azure resources → choose subscription/RG → Roles** → same flow for Owner/Contributor

### Azure CLI / PowerShell
PIM is mostly portal-based in the exam. With Az PowerShell you can create **eligible Azure role** assignments:
```powershell
$scope = "/subscriptions/<sub-id>/resourceGroups/rg-prod"
$role  = Get-AzRoleDefinition -Name Contributor
New-AzRoleEligibilityScheduleRequest -Name (New-Guid) -Scope $scope `
  -PrincipalId <user-object-id> -RoleDefinitionId "$scope/providers/Microsoft.Authorization/roleDefinitions/$($role.Id)" `
  -RequestType AdminAssign -ScheduleInfoStartDateTime (Get-Date) `
  -ExpirationType AfterDuration -ExpirationDuration "P90D"

Get-AzRoleEligibilitySchedule -Scope $scope
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| PIM blade says you need a license | No **P2** / ID Governance | Assign P2 (or trial) |
| User activated the role but still gets "access denied" | Old token | Sign out and in, or refresh the portal |
| Activation stuck in "Pending approval" | Approval required, approver hasn't acted | Approver uses **Approve requests** |
| Activation fails with MFA error | Role requires MFA, user not registered | Register MFA at aka.ms/mysecurityinfo |
| Can't find Application Administrator under Azure resources | It's an **Entra** role | Use **Microsoft Entra roles** section |
| Subscription not listed under Azure resources | You aren't Owner/UAA on it | Get the right role or **Elevate access** |

## Clean up
Remove lab eligible/active assignments: **PIM → (section) → Assignments → Remove**.

## Check yourself
<details><summary>1. Owners on production must request Owner, give a reason, and get approval from the CTO. Max 4 hours.</summary>

**PIM eligible** Owner assignment + role settings: justification, **approval** (CTO), max duration 4 h. Needs P2.
</details>

<details><summary>2. What's the difference between eligible and active?</summary>

**Eligible** must activate first; **active** has access right now.
</details>

<details><summary>3. Where do you make someone eligible for Application Administrator?</summary>

PIM → **Microsoft Entra roles** (directory scope) – not Azure resources.
</details>

## Related
- [Entra Licenses](entra-licenses.md)
- [Access Reviews](access-review.md)
- [Entra Roles vs Azure RBAC](entra-roles-vs-rbac.md)
