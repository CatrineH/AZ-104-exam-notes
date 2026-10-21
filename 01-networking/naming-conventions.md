# Naming Conventions

[← Back to README](../README.md)

## In one line
Consistent names make resources easy to find, filter, and understand – and some resources have strict naming rules you must know.

## When to use it (and when not)
Use a naming standard for **every** lab and project. Names usually **can't be changed** after creation, so decide before you deploy.

## Key concepts

### My lab format
| Resource | Format | Example |
|---|---|---|
| Resource Group | `Az104-<Name>-Rg` | Az104-Name-Rg |
| Virtual Network | `Az104-<Purpose>-vNet` | Az104-Storage-vNet |
| Subnet | `AZ104-<Purpose>-Snet` | AZ104-VMs-Snet |

### Microsoft's recommended prefixes (Cloud Adoption Framework)
Pattern: `<type>-<workload>-<environment>-<region>-<instance>`, for example `vm-web-prod-noe-001`.

| Resource | Prefix | Example |
|---|---|---|
| Resource group | `rg-` | `rg-webshop-prod` |
| Virtual network | `vnet-` | `vnet-hub-noe` |
| Subnet | `snet-` | `snet-web` |
| Network security group | `nsg-` | `nsg-web` |
| Public IP | `pip-` | `pip-vm-web01` |
| Network interface | `nic-` | `nic-vm-web01` |
| Virtual machine | `vm-` | `vm-web01` |
| Storage account | `st` (no hyphens!) | `stwebshopprod001` |
| Key Vault | `kv-` | `kv-webshop-prod` |
| App Service plan / app | `asp-` / `app-` | `app-webshop-prod` |
| Load balancer | `lbe-` / `lbi-` | `lbe-web` |
| Route table | `rt-` | `rt-spoke1` |
| Log Analytics workspace | `log-` | `log-platform-prod` |

### Naming rules worth knowing

| Resource | Rule |
|---|---|
| **Storage account** | 3–24, **lowercase letters and numbers only**, globally unique |
| **Key Vault** | 3–24, letters, numbers, hyphens, globally unique |
| **Container registry** | 5–50, letters and numbers only, globally unique |
| **Windows VM computer name** | Max **15** characters |
| **Linux VM name** | Max 64 |
| **Special subnets** | Exactly `GatewaySubnet`, `AzureBastionSubnet`, `AzureFirewallSubnet` |

## Exam traps
- Storage account names: **no uppercase, no hyphens**
- Global uniqueness: storage, Key Vault, ACR, App Service (`*.azurewebsites.net`)
- Most resources **can't be renamed** – you redeploy
- Enforce names with **Azure Policy** (Deny on a name pattern) and add context with **tags**

## Do it
```bash
az storage account check-name --name stwebshopprod001
az acr check-name --name catrinehacr01
```

## Troubleshooting
| Error | Cause | Fix |
|---|---|---|
| "AccountNameInvalid" | Uppercase/hyphen/length | Follow the rules above |
| "already taken" / "not available" | Global name in use somewhere in Azure | Add a suffix |
| Windows deployment fails on computer name | Longer than 15 characters | Shorter `--computer-name` |

## Related
- [IP Addressing and Subnetting](ip-addressing-subnetting.md)
- [Tags](../04-governance/tags.md)
