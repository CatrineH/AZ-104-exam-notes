# Entra ID Licenses

[← Back to README](../README.md) · Lab 03

## In one line
Many exam questions are really about **which license a feature needs**: **P1** to automate and control access, **P2** to govern and protect privileged access.

## When to use it (and when not)
| The company wants… | License |
|---|---|
| Users, groups, guests, basic SSO | **Free** |
| Dynamic groups, Conditional Access, group-based licensing, SSPR writeback | **P1** |
| PIM, access reviews, Identity Protection (risky users/sign-ins) | **P2** |

## Key concepts

| License | Includes, among other things |
|---|---|
| **Free** | Basic user and group management, bulk operations, B2B guests, security defaults |
| **P1** | **Conditional Access**, **dynamic groups**, **group-based licensing**, **SSPR with password writeback** to on-premises AD, role-assignable groups |
| **P2** | Everything in P1 + **PIM**, **Access Reviews**, **Identity Protection** (risk-based policies) |

**How to remember:**
- **P1 = automate and control access** (dynamic groups, Conditional Access)
- **P2 = govern and protect privileged access** (PIM, reviews, risk detection)

### Assigning licenses
- To a **user**, or to a **group** (group-based licensing – needs P1)
- The user's **usage location** must be set first
- You can **disable service plans** inside a license (for example give Microsoft 365 without Exchange)

## Exam traps
- "Dynamic groups / Conditional Access" → **P1**
- "Just-in-time admin access / access reviews / risky sign-in policies" → **P2**
- **Usage location** missing → license assignment fails
- Group-based licensing **doesn't work with nested groups**
- If a user gets the same license from a group **and** directly, removing it from the group doesn't remove the direct one
- **Not enough licenses** in the tenant → group-based assignment shows an **error** on some users

## Do it

### Portal
- **Entra ID → Billing → Licenses → All products** → choose license → **+ Assign** → users or groups → assignment options (turn service plans on/off)
- **User → Licenses** to see direct vs inherited (from group)
- **Group → Licenses** to see errors ("Users with license errors")

### Azure CLI
No `az` commands for licenses – use the portal or Microsoft Graph.

### PowerShell (Microsoft Graph)
```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All","Organization.Read.All"

# Which licenses does the tenant have, and how many are used?
Get-MgSubscribedSku | Select-Object SkuPartNumber, SkuId, ConsumedUnits, @{n='Total';e={$_.PrepaidUnits.Enabled}}

# Set usage location, then assign
Update-MgUser -UserId sarah@contoso.onmicrosoft.com -UsageLocation NO
$sku = Get-MgSubscribedSku | Where-Object SkuPartNumber -eq "AAD_PREMIUM_P2"
Set-MgUserLicense -UserId sarah@contoso.onmicrosoft.com -AddLicenses @{SkuId = $sku.SkuId} -RemoveLicenses @()
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "License cannot be assigned to a user without a usage location" | Usage location empty | Set it |
| Users in a group show **license errors** | Not enough licenses, or conflicting service plans | Buy more / free some, fix conflicts, then **Reprocess** |
| User in a nested group got no license | Nested groups aren't supported | Add the user (or a dynamic rule) to the licensed group directly |
| Feature (PIM, dynamic groups) greyed out | Tenant has no P1/P2 | Assign licenses or start a trial |
| Removed group license, user still licensed | License is also assigned **directly** | Remove the direct assignment |

## Clean up
Remove trial licenses from lab users: **User → Licenses → Remove**.

## Check yourself
<details><summary>1. Admins must request Global Administrator for 2 hours with approval. Which license?</summary>

**P2** (PIM).
</details>

<details><summary>2. Licenses should be assigned automatically to everyone in the Finance department.</summary>

**Dynamic group** + **group-based licensing** → **P1**.
</details>

<details><summary>3. Block sign-ins from outside Norway. Which license?</summary>

**P1** (Conditional Access with named locations).
</details>

## Related
- [Groups](groups.md)
- [SSPR](sspr.md)
- [PIM](pim.md)
- [Conditional Access](conditional-access.md)
