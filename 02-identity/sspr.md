# Self-Service Password Reset (SSPR)

[← Back to README](../README.md) · Lab 05

## In one line
**SSPR** lets users **reset their own password** (after proving who they are with registered methods) without calling the helpdesk.

## When to use it (and when not)
| Situation | Use |
|---|---|
| Too many helpdesk calls for forgotten passwords | **SSPR** |
| Try it with a pilot group first | SSPR **Enabled for: Selected** |
| Hybrid users whose password lives in on-prem AD | SSPR + **password writeback** (P1, Entra Connect) |
| Guests | Not SSPR – they reset at their home tenant |

## Key concepts

| Setting | Options |
|---|---|
| **Enabled for** | **None**, **Selected** (one group), or **All** |
| **Number of methods required** | **1 or 2** |
| **Methods** | Authenticator app, SMS, phone call, email, security questions, and more |
| **Registration** | Require users to register methods at next sign-in. Ask them to reconfirm every X days |
| **Notifications** | Notify users when their password is reset, notify admins when another admin resets |

### Administrators have stricter rules
- Admins **always** need **two methods** to reset their password, whatever the setting says
- Admins **can't use security questions**
- This applies even if SSPR is set to *None* for users

### Password writeback
- Sends the new password back to **on-premises Active Directory** (hybrid environments with Entra Connect / Cloud Sync)
- Requires **Entra ID P1**

### License
| Feature | License |
|---|---|
| Password **change** (user knows the old password) | Free |
| Password **reset** for cloud users (forgot it) | **P1** (or Microsoft 365 Business Standard/Premium) |
| Reset/unlock with **writeback** to on-prem AD | **P1** |

## Exam traps
- **Selected** = **one** group only. Need two groups → nest them or create a new group
- **Admins: always two methods, no security questions**, even if SSPR is off for users
- Writeback to on-premises AD → **P1** + Entra Connect
- "Users can't reset, but the feature is enabled" → they haven't **registered** methods, or they're not in the **selected** group
- Security questions can't be the **only** method used by admins and are the weakest option

## Do it

### Portal
1. **Entra ID → Password reset → Properties** → Self service password reset enabled: **Selected** → pick group
2. **Authentication methods** → number of methods (1 or 2) → tick methods
3. **Registration** → require registration on sign-in = Yes
4. **On-premises integration** → write back passwords (hybrid only)
5. Test: in a private browser go to `https://aka.ms/sspr` as a test user
6. **Audit logs** and **Usage & insights** to see registrations and resets

Users register methods at `https://aka.ms/mysecurityinfo`.

### Azure CLI / PowerShell
SSPR is configured in the **portal** (or via Graph policy APIs). The exam tests the portal settings.
```powershell
# See which methods a user has registered (Microsoft Graph)
Connect-MgGraph -Scopes "UserAuthenticationMethod.Read.All"
Get-MgUserAuthenticationMethod -UserId sarah@contoso.onmicrosoft.com
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| User gets "You can't reset your password" | Not in the **selected** group, or SSPR is None | Add to group / enable |
| User can't complete reset | Not enough **registered methods** | Make them register at aka.ms/mysecurityinfo |
| New password doesn't work on-prem | **Writeback** off, or Entra Connect not healthy | Enable writeback, check Entra Connect |
| Admin can't reset with security questions | Admins can't use them | Register two other methods |
| SSPR settings missing or greyed out | Tenant has no license that includes SSPR | Assign **P1** (or a trial) |

## Clean up
Set **Self service password reset enabled = None** after the lab if the tenant is shared.

## Check yourself
<details><summary>1. Pilot SSPR for the IT department only.</summary>

**Enabled for: Selected** → the IT group.
</details>

<details><summary>2. SSPR requires 1 method. How many does a Global Administrator need?</summary>

**Two** – admins always need two, and no security questions.
</details>

<details><summary>3. Hybrid users reset their password in the cloud, but the new password doesn't work on their PC at the office.</summary>

**Password writeback** isn't enabled (needs P1 + Entra Connect).
</details>

## Related
- [Entra Licenses](entra-licenses.md)
- [Entra Users](entra-users.md)
