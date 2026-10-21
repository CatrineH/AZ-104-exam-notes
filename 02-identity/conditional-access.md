# Conditional Access

[← Back to README](../README.md) · Lab 14

## In one line
**Conditional Access** is Entra ID's **"if–then" policy engine**: **IF** a user, app, and condition match → **THEN** block, or allow with requirements (MFA, compliant device…).

**Requires Entra ID P1.**

## When to use it (and when not)
| Requirement | Use |
|---|---|
| Require MFA for admins / for Azure management | **Conditional Access** |
| Block sign-ins from certain countries | CA with **named locations** |
| Only allow company (compliant) devices | CA grant: **Require compliant device** (Intune) |
| Block risky sign-ins automatically | CA with **sign-in risk** condition (needs **P2** for risk) |
| Small tenant, no P1, just want MFA for everyone | **Security defaults** (free) |
| Admin access only when needed | **PIM**, not CA |

## Key concepts

### Policy structure

| Part | Contains | Examples |
|---|---|---|
| **Assignments – users** | Include/exclude users, groups, roles | All users, except break-glass |
| **Target resources** | Cloud apps / actions | Microsoft Azure Management (Windows Azure Service Management API), Office 365 |
| **Conditions** | When the policy applies | Location, device platform, sign-in risk, client app |
| **Access controls – Grant** | Block, or grant with requirements | Require MFA, compliant device, (require one / all) |
| **Access controls – Session** | Limits during the session | Sign-in frequency, persistent browser |

### Rules for how policies work
- **Exclusions win over inclusions**
- If **several policies apply**, **all** of them must be satisfied
- **Block always wins**
- **Report-only** mode logs what *would* happen, without enforcing anything
- The **What If** tool simulates the result for a user without a real sign-in

### Microsoft Azure Management
- Covers the **Azure portal, Azure CLI, Azure PowerShell, and ARM**
- Be careful when blocking it – you can **lock yourself out**

### Break-glass account
- An **emergency admin account** that is **excluded** from Conditional Access policies
- Used if a policy locks out all other admins
- Long, strong password, stored safely, and its sign-ins are monitored

### Security defaults vs Conditional Access
| | Security defaults | Conditional Access |
|---|---|---|
| License | Free | P1 |
| Flexible | No – on/off for everyone | Yes – per user, app, condition |
| Together? | **Must turn security defaults off** to use CA | |

## Exam traps
- Conditional Access = **P1**. Risk-based conditions (sign-in/user risk) = **P2**
- Exclude beats include, Block beats everything
- "Test a policy without affecting users" → **Report-only** or **What If**
- Always **exclude a break-glass account**
- **Security defaults** must be **disabled** before creating CA policies
- Policies apply **after** first-factor authentication – they don't stop a wrong password
- Country blocking needs a **named location** first

## Do it

### Portal
1. **Entra ID → Protection → Conditional Access → Named locations → + Countries location** (if needed)
2. **Policies → + New policy**
3. **Users:** include *All users* → exclude **break-glass** account
4. **Target resources:** *Microsoft Azure Management*
5. **Grant:** *Require multifactor authentication*
6. **Enable policy: Report-only** → Create
7. **What If:** choose a user + app → see which policies apply
8. Check **Sign-in logs → Conditional Access / Report-only** tab → then set the policy to **On**

### Azure CLI / PowerShell
Usually done in the portal. To list policies (Microsoft Graph):
```powershell
Connect-MgGraph -Scopes "Policy.Read.All"
Get-MgIdentityConditionalAccessPolicy | Select-Object DisplayName, State
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Can't create policies – "security defaults are enabled" | Security defaults on | Turn off: **Entra ID → Properties → Manage security defaults** |
| Admins locked out of the portal | Block / MFA policy on Azure Management without exclusion | Sign in with **break-glass** account, fix the policy |
| User unexpectedly blocked | Another policy also applies (all must be satisfied) | **Sign-in logs → Conditional Access** tab, or **What If** |
| Policy doesn't apply to a user | User or a group they're in is **excluded** | Exclusions win – check excludes |
| Country block doesn't work | User on VPN with IP in another country, or location not configured | Check sign-in log IP / named location |

**Sign-in logs** (Entra ID → Monitoring → Sign-in logs) show every policy result per sign-in – the first place to look.

## Clean up
Set lab policies to **Off** or delete them – especially anything targeting Azure Management.

## Check yourself
<details><summary>1. Require MFA for everyone using the Azure portal, CLI, or PowerShell. Don't risk lockout.</summary>

CA policy: All users (exclude break-glass) → **Microsoft Azure Management** → Require MFA. Start in **Report-only**.
</details>

<details><summary>2. One policy requires MFA, another blocks access from outside Norway. A user signs in from Spain with MFA. Result?</summary>

**Blocked** – all policies apply and block wins.
</details>

<details><summary>3. A small company with no P1 licenses wants MFA for everyone.</summary>

**Security defaults**.
</details>

## Related
- [Entra Licenses](entra-licenses.md)
- [Entra Roles vs Azure RBAC](entra-roles-vs-rbac.md)
- [PIM](pim.md)
