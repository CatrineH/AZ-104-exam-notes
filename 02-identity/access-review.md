# Access Reviews

[← Back to README](../README.md) · Lab 13

## In one line
**Access reviews** check **regularly** whether people should **keep** their access – group memberships, app access, or roles – and can remove what's no longer needed automatically.

## When to use it (and when not)
| Situation | Use |
|---|---|
| Review **guest users** every quarter | Access review on guests in groups/apps |
| Review members of **admin groups** | Access review on the group |
| Review who has **privileged roles** | **PIM** access review (Entra or Azure roles) |
| Remove access the moment someone changes department | **Dynamic group** – reviews are periodic, not instant |

**Requires Entra ID P2** (or Microsoft Entra ID Governance).

## Key concepts

### Who can review?
- **Group owners**
- **Selected users**
- **Managers** of the users
- **The users themselves** (self-review)

### Important settings

| Setting | What it does |
|---|---|
| **Recurrence** | One-time, weekly, monthly, quarterly, semi-annually, yearly |
| **Duration** | How many days reviewers have to respond |
| **Auto apply results** | If on, a **denial actually removes** the access. If off, someone must apply the results manually |
| **If reviewers don't respond** | No change, remove access, approve access, or take recommendations |
| **Recommendations** | Suggests "deny" for users who haven't signed in for 30 days |

### Why group-based access matters
If Azure RBAC roles are assigned to a **group**, removing a user from the group in an access review **also removes their Azure access** – automatically. This is a big advantage of giving access through groups instead of to individual users.

## Exam traps
- Access reviews = **P2**
- **Auto apply results** decides whether denials are actually applied
- Reviewers who don't answer → the **"If reviewers don't respond"** setting decides
- Want guests removed if nobody confirms them → **If reviewers don't respond = Remove access** + auto apply
- Reviews can't target a **dynamic** group's members for removal (membership is rule-based)

## Do it

### Portal
1. **Entra ID → Identity Governance → Access reviews → + New access review**
2. **Review type:** Teams + Groups (or Applications) → scope: **Guest users only** or all users → choose group(s)
3. **Reviewers:** group owners / selected users / managers / self
4. **Recurrence:** quarterly, duration 14 days
5. **Upon completion:** **Auto apply results = On**, **If reviewers don't respond = Remove access**
6. Reviewer opens **My Access** (myaccess.microsoft.com) → Approve / Deny
7. Results: open the review → **Results** / **Audit logs**

For roles: **PIM → Microsoft Entra roles / Azure resources → Access reviews**.

### Azure CLI / PowerShell
Created in the portal (or Microsoft Graph `accessReviews` API). The exam tests the settings, not commands.
```powershell
Connect-MgGraph -Scopes "AccessReview.Read.All"
Get-MgIdentityGovernanceAccessReviewDefinition | Select-Object DisplayName, Status
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Denied users still have access | **Auto apply** was off | Apply results manually, turn auto apply on |
| Reviewer never got an email | Email off, wrong reviewer, or not licensed | Check reviewer settings, send reminder, use My Access |
| Access review option missing | No **P2** / ID Governance | License |
| User removed from group but still has Azure access | Role was assigned **directly**, not via the group | Assign roles to groups |

## Clean up
Delete lab reviews: **Access reviews → choose → Delete**.

## Check yourself
<details><summary>1. Guests in the Partners group must be reviewed every quarter by the group owner. If the owner doesn't answer, remove them.</summary>

Access review: Partners group, **guests only**, reviewer **group owners**, **quarterly**, auto apply on, **if no response → remove access**. P2.
</details>

<details><summary>2. The review finished with denials but nothing changed. Why?</summary>

**Auto apply results** was off.
</details>

## Related
- [Groups](groups.md)
- [Guest Users](guest-users.md)
- [PIM](pim.md)
