# App Registrations and Enterprise Applications

[← Back to README](../README.md) · Lab 12

## In one line
An **app registration** is the app's global **definition**, its **service principal** is the app's local **instance** in a tenant that gets permissions, and an **enterprise application** is just the **portal view** of that service principal.

## When to use it (and when not)
| Situation | Where |
|---|---|
| Your own app needs to sign users in or call APIs | **App registrations → New registration** |
| Configure SSO, assign users/groups, see sign-ins for an app (incl. SaaS from the gallery) | **Enterprise applications** |
| Grant admin consent for permissions | App registration → **API permissions**, or Enterprise app → **Permissions** |
| Code running in Azure needs an identity | **Managed identity**, not a new app registration |

## Key concepts

### Three concepts that are often confused

| Concept | What it is | Where |
|---|---|---|
| **App registration** (application object) | The **global definition** of the app. Exists **once**, in the app's home tenant | Entra ID → App registrations |
| **Service principal** | The app's **local instance in a tenant**. This is what gets **permissions and roles** | One per tenant that uses the app |
| **Enterprise application** | The **portal view of the service principal** | Entra ID → Enterprise applications |

```
App registration (home tenant)  ── definition, one per app
        │
        ├── Service principal in tenant A  ── gets roles here
        └── Service principal in tenant B  ── gets roles here
                (shown as Enterprise applications)
```

### IDs
- **Same Client ID** (Application ID) for the app registration and its service principals
- **Different Object IDs** – they are different objects

### API permissions

| Type | The app acts… | Example |
|---|---|---|
| **Delegated** | **On behalf of a signed-in user** – can never do more than the user can | A web app reads the user's own email |
| **Application** | **On its own**, with no user signed in | A background job reads all groups |

- Some delegated permissions need **admin consent**, for example **`Group.Read.All`**
- **Application permissions always need admin consent**
- **User consent settings** (Enterprise applications → Consent and permissions) decide whether users may consent on their own

### Enterprise app settings worth knowing
- **Assignment required?** = Yes → only **assigned** users/groups can sign in
- **Visible to users?** → shows in My Apps (myapps.microsoft.com)
- **Single sign-on** → SAML / OIDC / password-based

## Exam traps
- App registration = **definition**, service principal = **instance that gets access**, enterprise application = **portal view of the service principal**
- Same Client ID, different Object IDs
- Delegated = **with a user**, Application = **without a user**
- Application permissions → **admin consent** always. "Grant admin consent" needs a role like **Global Admin, Privileged Role Admin, or Cloud Application Administrator**
- Restrict an app to some users → Enterprise app → **Assignment required = Yes** + assign groups
- Deleting the **app registration** deletes the app in the home tenant – service principals in other tenants stop working

## Do it

### Portal
1. **App registrations → + New registration** → name, supported account types (single tenant / multitenant), redirect URI
2. **API permissions → + Add a permission → Microsoft Graph** → Delegated or Application → `User.Read.All` → **Grant admin consent**
3. **Certificates & secrets** → secret, certificate, or **federated credential**
4. **Enterprise applications** → the same app → **Properties: Assignment required = Yes** → **Users and groups → + Add**

### Azure CLI
```bash
APPID=$(az ad app create --display-name app-az104-demo --sign-in-audience AzureADMyOrg --query appId -o tsv)
az ad sp create --id $APPID                       # create the service principal (enterprise app)

az ad app credential reset --id $APPID --append    # add a secret

# Add Graph application permission User.Read.All and grant admin consent
az ad app permission add --id $APPID --api 00000003-0000-0000-c000-000000000000 \
  --api-permissions df021288-bdef-4463-88db-98f22de89214=Role
az ad app permission admin-consent --id $APPID

az ad app show --id $APPID --query "{appId:appId, objectId:id}"
az ad sp show --id $APPID  --query "{appId:appId, objectId:id}"   # same appId, different object id
```

### PowerShell
```powershell
$app = New-AzADApplication -DisplayName app-az104-demo
$sp  = New-AzADServicePrincipal -ApplicationId $app.AppId
New-AzADAppCredential -ApplicationId $app.AppId -EndDate (Get-Date).AddMonths(6)

"$($app.AppId)  app objectId: $($app.Id)  sp objectId: $($sp.Id)"
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| **AADSTS65001** "The user or administrator has not consented" | Permission needs consent | **Grant admin consent** |
| **AADSTS50105** "user is not assigned to a role for the application" | **Assignment required = Yes** and user not assigned | Assign the user/group in Enterprise apps |
| **AADSTS50011** redirect URI mismatch | Redirect URI in the app doesn't match the request | Add the exact URI under **Authentication** |
| App can't read data with Application permission | Admin consent missing (status shows "Not granted") | Grant admin consent |
| "Grant admin consent" button greyed out | You lack a consent-capable role | Cloud Application Administrator / Global Admin |
| Users can't add new apps themselves | User consent disabled | Expected – admin consent workflow |

## Clean up
```bash
az ad app delete --id $APPID    # also removes the service principal in your tenant
```

## Check yourself
<details><summary>1. A nightly job must read all users' profiles with no one signed in. Which permission type?</summary>

**Application** permission (`User.Read.All`) + **admin consent**.
</details>

<details><summary>2. Only the Finance group may sign in to an enterprise app.</summary>

**Assignment required = Yes** and assign the Finance group.
</details>

<details><summary>3. The app registration and enterprise app show the same Application ID but different Object IDs. Is that a problem?</summary>

No – they're two objects (definition and instance) for the same app.
</details>

## Related
- [Service Principals](service-principals.md)
- [Managed Identities](managed-identities.md) – a service principal managed by Azure
- [Federation](federation.md)
