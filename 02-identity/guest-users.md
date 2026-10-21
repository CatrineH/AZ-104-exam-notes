# Guest Users (B2B Collaboration)

[← Back to README](../README.md) · Lab 04

## In one line
**B2B collaboration** lets **external users** (partners, consultants) into your tenant as **guests** – they sign in with **their own** account, you decide what they can access.

## When to use it (and when not)
| Situation | Use |
|---|---|
| A consultant needs access to one resource group for a project | **Invite a guest**, give an RBAC role (via a group) |
| 200 partner users | **Bulk invite** with CSV |
| Limit which companies can be invited | **Allow/deny domains** in External collaboration settings |
| Regular check whether guests still need access | **Access review** (P2) |
| Customers signing up to your app | Not B2B – that's **External ID for customers** |

## Key concepts
- The guest signs in with **their own identity**, authenticated in **their home tenant** (or with a Microsoft account, Google, or **email one-time passcode**)
- I control **what** they can access in my tenant

### How a guest looks in my tenant

| Property | Value |
|---|---|
| **User type** | Guest |
| **UPN** | `sarah_abc.com#EXT#@contoso.onmicrosoft.com` |

The UPN is the guest's own email with `@` replaced by `_`, followed by `#EXT#` and my tenant domain.

### Inviting guests
1. Portal: **Users → + New user → Invite external user**
2. The guest gets an email and **accepts the invitation** (redemption)
3. Many guests at once: **Bulk invite** with a CSV file

### External collaboration settings
Portal: **External Identities → External collaboration settings**:
- **Guest user access:** what guests can see in the directory (same as members / limited / most restricted)
- **Guest invite settings:** who can invite (only admins and Guest Inviter role, members, or also guests)
- **Collaboration restrictions:** **allowed or blocked domains** for invitations

**Cross-tenant access settings** (inbound/outbound) control B2B with specific other Entra tenants, for example trusting their MFA.

## Exam traps
- Guests can be **added to groups** and **assigned RBAC roles** just like regular users
- Guests authenticate in their **home tenant** – I don't manage their passwords and **can't reset** them
- To limit who can invite → **External collaboration settings**. A non-admin who must invite → **Guest Inviter** role
- To block a specific partner company → **deny list** of domains
- A guest that hasn't accepted has **"Pending acceptance"** – **Resend invitation**
- You **can't** use both an allow list and a deny list at the same time

## Do it

### Portal
- **Entra ID → Users → + New user → Invite external user** → email, display name, message → optionally add to groups/roles right away
- **Users → Bulk operations → Bulk invite**
- **External Identities → External collaboration settings**
- Guest's profile → **B2B collaboration → Resend invitation**

### Azure CLI
No `az ad` command for invitations – use the portal or Microsoft Graph.
```bash
az ad user list --filter "userType eq 'Guest'" --query "[].{name:displayName, mail:mail}" -o table
```

### PowerShell (Microsoft Graph)
```powershell
Connect-MgGraph -Scopes "User.Invite.All","User.Read.All"

New-MgInvitation -InvitedUserEmailAddress "sarah@abc.com" -InvitedUserDisplayName "Sarah (ABC)" `
  -InviteRedirectUrl "https://myapps.microsoft.com" -SendInvitationMessage:$true

Get-MgUser -Filter "userType eq 'Guest'" -Property DisplayName,Mail,ExternalUserState |
  Select-Object DisplayName, Mail, ExternalUserState
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Guest never got the email | Spam filter, wrong address | **Resend invitation**, or send them the redemption link |
| "Your organization doesn't allow…" when inviting | Domain on the **deny list** or not on the allow list | Update collaboration restrictions |
| Users can't invite guests | Invite settings allow only admins | Change setting, or give **Guest Inviter** role |
| Guest signed in but sees no resources | No role assignment / group membership, or wrong tenant chosen in the portal | Assign role, have them **switch directory** to your tenant |
| Guest blocked by Conditional Access / MFA loop | CA requires MFA and cross-tenant settings don't trust their MFA | Guest registers MFA in your tenant, or trust home-tenant MFA in **cross-tenant access** |
| Guest can't reset password | Password lives in **their** home tenant | They reset it at home |

## Clean up
```bash
az ad user delete --id "sarah_abc.com#EXT#@contoso.onmicrosoft.com"
```

## Check yourself
<details><summary>1. Only admins should be able to invite guests, except one project manager.</summary>

Set invite settings to admins only, and give the project manager the **Guest Inviter** role.
</details>

<details><summary>2. Guests from competitor.com must never be invited.</summary>

Add `competitor.com` to the **deny list** in External collaboration settings.
</details>

<details><summary>3. A guest says they forgot their password. Can you reset it?</summary>

No. They reset it in their **home tenant** / identity provider.
</details>

## Related
- [Entra Users](entra-users.md)
- [Access Reviews](access-review.md) – review guest access regularly
- [Conditional Access](conditional-access.md)
