# Managed Identities

[← Back to README](../README.md) · Lab 10

## In one line
A **managed identity** gives an Azure resource (VM, App Service, Function…) its own identity in Entra ID, so it can sign in to other Azure services **without any password, key, or secret in code**.

## When to use it (and when not)
| Situation | Use |
|---|---|
| One VM/app needs to read storage or Key Vault | **System-assigned** |
| Many VMs/apps need the **same** access, or the identity must survive the resource being recreated | **User-assigned** |
| Identity needed **before** the resource exists (for example CMK at storage creation) | **User-assigned** |
| Code runs **outside Azure** (on-prem server, GitHub Actions) | Not a managed identity – use a **service principal** (preferably with [federation](federation.md)) |

## Key concepts

| | System-assigned | User-assigned |
|---|---|---|
| Created by | Azure, when enabled on a resource | You, as a separate resource |
| Tied to | One resource – follows its lifecycle | Independent |
| Shared between resources | No | Yes |
| Deleted when resource is deleted | Yes | No |
| One resource can have | One | Many |

- Under the hood, a managed identity is a **special service principal** that Azure manages – you never see or rotate its credentials
- Inside the resource, code gets a token from the local **Instance Metadata Service (IMDS)** endpoint `http://169.254.169.254/metadata/identity/oauth2/token`
- Having an identity gives **no access by itself** – it needs an **RBAC role** on the target

### Lab: VM reading from storage with its managed identity
1. Enable a managed identity on the VM
2. Assign a data role on the storage account or container
3. Sign in **from the VM** and work with blobs – no keys or secrets

```powershell
Connect-AzAccount -Identity                                       # sign in as the VM
$ctx = New-AzStorageContext -StorageAccountName staz104cat -UseConnectedAccount
Get-AzStorageBlob -Container training -Context $ctx                # list
Get-AzStorageBlobContent -Container training -Blob training.txt -Context $ctx   # download
Set-AzStorageBlobContent -Container training -File .\ny.txt -Context $ctx       # upload
Remove-AzStorageBlob -Container training -Blob ny.txt -Context $ctx             # delete
```

| Role on the storage account | List and read | Upload and delete |
|---|---|---|
| **Storage Blob Data Reader** | Yes | No |
| **Storage Blob Data Contributor** | Yes | Yes |

## Exam traps
- System-assigned is **deleted with the resource**, user-assigned is **not**
- The managed identity still needs an **RBAC role** – having an identity gives no access by itself
- "**Without storing credentials**" in the question → managed identity
- Several VMs that need identical access and are recreated often → **user-assigned**
- `-UseConnectedAccount` / `--auth-mode login` = use the signed-in identity (Entra ID) instead of a key
- A managed identity only works **from inside** the Azure resource – you can't use it from your laptop

## Do it

### Portal
- **System-assigned:** VM → **Security → Identity → System assigned → Status On** → Save → **Azure role assignments**
- **User-assigned:** **Create a resource → User Assigned Managed Identity** → then VM → Identity → **User assigned → + Add**
- **Give access:** storage account → **Access control (IAM) → Add role assignment** → Storage Blob Data Reader → **Members: Managed identity** → choose the VM

### Azure CLI
```bash
# System-assigned on a VM
az vm identity assign -g $RG -n vm-app01
PRINCIPAL=$(az vm show -g $RG -n vm-app01 --query identity.principalId -o tsv)

az role assignment create --assignee-object-id $PRINCIPAL --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" --scope $(az storage account show -n $SA -g $RG --query id -o tsv)

# User-assigned, shared by two VMs
az identity create -g $RG -n id-app
ID=$(az identity show -g $RG -n id-app --query id -o tsv)
az vm identity assign -g $RG -n vm-app01 --identities $ID
az vm identity assign -g $RG -n vm-app02 --identities $ID

# Inside the VM (Linux)
az login --identity
az storage blob list --account-name $SA --container-name training --auth-mode login -o table
```

### PowerShell
```powershell
$vm = Get-AzVM -ResourceGroupName $rg -Name vm-app01
Update-AzVM -ResourceGroupName $rg -VM $vm -IdentityType SystemAssigned

$uami = New-AzUserAssignedIdentity -ResourceGroupName $rg -Name id-app -Location norwayeast
Update-AzVM -ResourceGroupName $rg -VM $vm -IdentityType UserAssigned -IdentityId $uami.Id

New-AzRoleAssignment -ObjectId $vm.Identity.PrincipalId -RoleDefinitionName "Storage Blob Data Reader" `
  -Scope (Get-AzStorageAccount -ResourceGroupName $rg -Name $sa).Id
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| **AuthorizationPermissionMismatch** right after assigning a role | Role still propagating | Wait a few minutes and try again |
| Same error after waiting | Role given at the wrong scope, or to the wrong identity (system vs user-assigned) | Check IAM → Role assignments on the target |
| `Connect-AzAccount -Identity` fails on your PC | Managed identity only works **inside** Azure | Run it on the VM |
| VM with several user-assigned identities picks the wrong one | No client ID specified | `az login --identity --client-id <id>` / `Connect-AzAccount -Identity -AccountId <client-id>` |
| Access stopped after recreating the VM | System-assigned identity was deleted with the old VM | Use **user-assigned**, or reassign the role to the new identity |
| Can't reach IMDS (169.254.169.254) | Proxy or firewall on the VM intercepts it | Bypass proxy for 169.254.169.254 |

## Clean up
```bash
az vm identity remove -g $RG -n vm-app01
az identity delete -g $RG -n id-app
```

## Check yourself
<details><summary>1. Ten VMs in a scale set must read the same Key Vault secret. VMs are replaced often.</summary>

One **user-assigned** managed identity with **Key Vault Secrets User**, attached to the scale set.
</details>

<details><summary>2. You enabled a system-assigned identity on a VM, but the app gets 403 from storage.</summary>

The identity has **no RBAC role** yet – assign Storage Blob Data Reader/Contributor.
</details>

<details><summary>3. What happens to a system-assigned identity when the VM is deleted?</summary>

It's **deleted too**, along with its role assignments.
</details>

## Related
- [Service Principals](service-principals.md)
- [Azure RBAC](../04-governance/azure-rbac.md)
- [Storage Access](../03-storage/storage-access.md)
