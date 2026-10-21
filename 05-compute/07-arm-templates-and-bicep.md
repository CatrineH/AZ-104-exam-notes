# ARM Templates and Bicep

[← Back to README](../README.md)

AZ-104 domain: Deploy and manage Azure compute resources – automate deployment with ARM templates and Bicep

> Starter note in the agreed format – add my own course notes from the lesson in "Key concepts".

## In one line
**ARM templates** (JSON) and **Bicep** (a simpler language that compiles to ARM JSON) describe Azure resources as **code**, so the same deployment can be repeated, reviewed, and previewed with **what-if** before anything changes.

## When to use it (and when not)
| Need | Use |
|---|---|
| Deploy the same environment again and again (dev/test/prod) | **Bicep/ARM** with parameter files |
| Preview what a deployment will change | **what-if** |
| Save the setup of something built by clicking | **Export template** (then clean it up) |
| One-off quick test | Portal or a single CLI command |
| Multi-cloud IaC | Terraform (not on AZ-104) |

## Key concepts

### ARM template structure (JSON)
```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageName": { "type": "string", "minLength": 3, "maxLength": 24 },
    "location":    { "type": "string", "defaultValue": "[resourceGroup().location]" }
  },
  "variables": {
    "skuName": "Standard_LRS"
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2023-05-01",
      "name": "[parameters('storageName')]",
      "location": "[parameters('location')]",
      "sku": { "name": "[variables('skuName')]" },
      "kind": "StorageV2"
    }
  ],
  "outputs": {
    "blobEndpoint": {
      "type": "string",
      "value": "[reference(parameters('storageName')).primaryEndpoints.blob]"
    }
  }
}
```

| Section | Holds | Required? |
|---|---|---|
| `$schema`, `contentVersion` | Template version info | Yes |
| **parameters** | Values given **at deployment** (name, size, environment) | No |
| **variables** | Values **calculated inside** the template | No |
| **resources** | What to deploy (`type`, `apiVersion`, `name`, `location`, properties) | **Yes** |
| **outputs** | Values returned after deployment | No |
| `functions` | User-defined functions | No |

- **Expressions** use `[ ]`: `[parameters('x')]`, `[variables('y')]`, `[resourceGroup().location]`, `[concat(...)]`, `[uniqueString(resourceGroup().id)]`
- **dependsOn** – deploy this resource after another (e.g. NIC before VM)
- **Secure values:** `securestring` parameter type, or a **Key Vault reference** in the parameter file

### The same in Bicep
```bicep
@minLength(3)
@maxLength(24)
param storageName string
param location string = resourceGroup().location

var skuName = 'Standard_LRS'

resource sa 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageName
  location: location
  sku: { name: skuName }
  kind: 'StorageV2'
}

output blobEndpoint string = sa.properties.primaryEndpoints.blob
```
- Shorter, no `[ ]`, **dependencies are automatic** when you reference another resource (`sa.id`)
- `@secure()` decorator for passwords
- **Modules:** split into files (`module net 'modules/network.bicep' = { ... }`)
- `az bicep build` → ARM JSON; `az bicep decompile` → JSON to Bicep

### Parameter files
| Format | Example |
|---|---|
| ARM JSON | `azuredeploy.parameters.json` with `"parameters": { "storageName": { "value": "stdev001" } }` |
| Bicep | `main.dev.bicepparam` with `using 'main.bicep'` + `param storageName = 'stdev001'` |

### Deployment scopes
| Scope | CLI | `targetScope` in Bicep |
|---|---|---|
| Resource group (default) | `az deployment group create` | `'resourceGroup'` |
| Subscription (e.g. create RGs, policy) | `az deployment sub create --location` | `'subscription'` |
| Management group | `az deployment mg create` | `'managementGroup'` |
| Tenant | `az deployment tenant create` | `'tenant'` |

### Deployment modes
| Mode | Resources in the RG **not** in the template |
|---|---|
| **Incremental** (default) | **Left alone** |
| **Complete** | **Deleted** (careful!) |

## Exam traps
- **Incremental** is the default – existing resources not in the template stay. **Complete** deletes them
- Only **resources** is a required section (plus schema/contentVersion)
- **Parameters** = input at deploy time; **variables** = computed inside
- Reading a template question: look at `type` (what), `name`, `location`, `sku`, and **`dependsOn`**/order
- A template that creates a **resource group** must be deployed at **subscription** scope
- Redeploying the same template is **idempotent** – same result, no duplicates
- **Export template** from a RG gives JSON that often needs cleanup (hard-coded values, unsupported types)
- Deployment history: **RG → Deployments** (max 800 per RG; old ones are auto-deleted)
- Passwords: `securestring` / `@secure()` or **Key Vault reference** – never plain text in the parameter file

## Do it

### Portal
- **Deploy a custom template:** search "Deploy a custom template" → **Build your own template in the editor** / load a quickstart → fill parameters → Review + create
- **Export:** resource group → **Automation → Export template** → Download / Deploy
- **History:** resource group → **Settings → Deployments** → open one → **Template** / **Inputs** / **Outputs** / **Redeploy**

### Azure CLI
```bash
RG=rg-iac-lab
az group create -n $RG -l norwayeast

# Validate → what-if → deploy
az deployment group validate -g $RG --template-file main.bicep --parameters storageName=stcatiac$RANDOM
az deployment group what-if  -g $RG --template-file main.bicep --parameters storageName=stcatiac001
az deployment group create   -g $RG --template-file main.bicep --parameters storageName=stcatiac001

# ARM JSON with a parameter file
az deployment group create -g $RG --template-file azuredeploy.json --parameters @azuredeploy.parameters.json

# Outputs and history
az deployment group show -g $RG -n main --query properties.outputs
az deployment group list -g $RG -o table

# Export what exists
az group export -n $RG > exported.json

# Bicep tools
az bicep build --file main.bicep          # → main.json
az bicep decompile --file exported.json   # → exported.bicep
```

### PowerShell
```powershell
New-AzResourceGroup -Name rg-iac-lab -Location norwayeast

Test-AzResourceGroupDeployment -ResourceGroupName rg-iac-lab -TemplateFile .\main.bicep -storageName stcatiac001
New-AzResourceGroupDeployment  -ResourceGroupName rg-iac-lab -TemplateFile .\main.bicep -storageName stcatiac001 -WhatIf
New-AzResourceGroupDeployment  -ResourceGroupName rg-iac-lab -TemplateFile .\main.bicep -storageName stcatiac001

New-AzResourceGroupDeployment -ResourceGroupName rg-iac-lab -TemplateFile .\azuredeploy.json `
  -TemplateParameterFile .\azuredeploy.parameters.json

Export-AzResourceGroup -ResourceGroupName rg-iac-lab -Path .\exported.json
New-AzSubscriptionDeployment -Location norwayeast -TemplateFile .\sub.bicep   # subscription scope
```
(Bicep files from PowerShell need the **Bicep CLI** installed.)

## Troubleshooting

| Error / symptom | Likely cause | Fix |
|---|---|---|
| **InvalidTemplate** | JSON/Bicep syntax error, wrong expression | `validate` / `az bicep build` shows the line |
| **InvalidTemplateDeployment** + policy name | **RequestDisallowedByPolicy** inside | Change location/SKU/tags to comply |
| "The template resource … is not defined" / dependency errors | Missing **dependsOn** (JSON) or wrong reference | Add dependsOn / reference the resource symbol |
| **StorageAccountAlreadyTaken** / name not available | Global name in use | `uniqueString()` in the name |
| **SkuNotAvailable** / **QuotaExceeded** | Region/size problem | Other size/region, quota request |
| **AuthorizationFailed** on role assignments in the template | Deployer is only Contributor | Needs Owner / UAA / RBAC Administrator |
| Resources disappeared after deployment | Deployed in **Complete** mode | Use Incremental (default); always run **what-if** first |
| Template creates an RG but fails at group scope | Wrong scope | `az deployment sub create` |
| Where's the real error? | | RG → **Deployments** → failed deployment → **Error details** / operation details |

## Break it and fix it

| # | Break it | What you should see | Fix |
|---|---|---|---|
| 1 | Deploy with `storageName=Stor-Age` | Validation fails – invalid name | Lowercase, no hyphens |
| 2 | Add `location: 'westeurope'` while an "Allowed locations = Norway East" policy is assigned | RequestDisallowedByPolicy | Use an allowed location |
| 3 | Create a second storage account by hand in the RG, then deploy with `--mode Complete` and **what-if** | what-if shows it as **Delete** | Don't deploy – use Incremental |
| 4 | Deploy the same template twice | Second run: no changes (**idempotent**) | – |

## Clean up
```bash
az group delete -n rg-iac-lab --yes --no-wait
```

## Check yourself
<details><summary>1. A template deploys a VNet into RG1, which also has a VM not in the template. Default mode. What happens to the VM?</summary>

Nothing – **Incremental** mode leaves it alone.
</details>

<details><summary>2. Which section is required in every ARM template?</summary>

**resources** (plus `$schema` and `contentVersion`).
</details>

<details><summary>3. How do you see what a deployment *would* change, without changing anything?</summary>

**what-if** (`az deployment group what-if` / `-WhatIf`).
</details>

<details><summary>4. The template creates resource groups. Which command deploys it?</summary>

`az deployment sub create` (subscription scope, with `--location`).
</details>

## Related
- [Deployment checklist](../00-guides/az-project-skeleton.md)
- [VMs: plan and deploy](03-vms-plan-deployment.md)
- [Azure Policy](../04-governance/azure-policy.md)
