# Azure Project Skeleton – Deployment Checklist

[← Back to README](../README.md)

A general order of work for setting up a new Azure project with Infrastructure as Code (IaC). 

- Steps **1, 3, 5, and 8** stay essentially the same for every customer
- Step **2** varies – some customers have no management groups at all
- Steps **6–7** can vary depending on which networking and compute pieces the project needs

## Checklist

- [ ] 1. Define IP plan, naming, and tagging conventions
- [ ] 2. Confirm subscription and management group structure
- [ ] 3. Create the resource group(s)
- [ ] 4. Set up the deployment backend
- [ ] 5. Configure identity for deployments
- [ ] 6. Build the core networking layer
- [ ] 7. Author modular IaC templates
- [ ] 8. Validate, what-if, then deploy via pipeline

---

## 1. Define IP plan, naming, and tagging conventions

Agree on standards **before creating anything**. Retrofitting names later is painful, because many Azure resources can't be renamed.

**Naming** – Microsoft's Cloud Adoption Framework (CAF) recommends starting with the resource type:

```
<resourcetype>-<workload>-<env>-<region>-<instance>
rg-lab104-catrine-prod-norwayeast-001
vnet-labcatrine-prod-norwayeast-001
```

**Tagging schema** – for example:

| Tag | Example |
|---|---|
| `owner` | cat@company.com |
| `costCenter` | 1234 |
| `environment` | dev / test / prod |
| `project` | lab104-catrine |

**IP plan** – decide address ranges for each vNet up front, with no overlaps between vNets that will be connected.

Put naming and tags into my IaC as **variables/parameters**, so every resource is consistent from day one.

Ref also: [Naming Conventions](../01-networking/naming-conventions.md), [IP Addressing and Subnetting](../01-networking/ip-addressing-subnetting.md)

## 2. Confirm subscription and management group structure

Find where the project sits in the organization's hierarchy:

```
Management groups
└── Subscriptions
    └── Resource groups
        └── Resources
```

- **Azure Policy** and **RBAC** assigned at a higher level are **inherited** by everything below
- Check which policies apply, so my deployments comply with governance set higher up (for example, allowed regions or required tags)

## 3. Create the resource group(s)

- Typically **one resource group per workload and environment**, for example `rg-appname-dev` and `rg-appname-prod`
- Set the **location** and **tags** here
- In Bicep, either:
  - Target the **subscription** and create the resource group in the template, or
  - Target the **resource group** directly if it already exists

## 4. Set up the deployment backend

- **Bicep/ARM does not need a state file** – Azure tracks deployment history itself
- **Terraform does need state** – store it in a storage account (blob container)
- A storage account is also used by **deployment scripts** (Bicep scripts that run PowerShell or CLI during deployment)
- Create this early, since other pipelines depend on it

## 5. Configure identity for deployments

The CI/CD pipeline needs its own identity – **never use personal credentials**.

| Option | Notes |
|---|---|
| Service principal with a secret | Works, but the secret must be stored and rotated |
| **Workload identity federation** (service principal or user-assigned managed identity) | **Preferred** – no secrets. GitHub Actions or Azure DevOps sign in using a trusted token |

- Follow **least privilege**: assign **Contributor scoped to the resource group**, not the subscription
- !*! Contributor **cannot create role assignments**. If my templates assign RBAC roles (for example, giving a managed identity access to storage), the pipeline also needs **Role Based Access Control Administrator** or **User Access Administrator**

Ref also: [Service Principals](../02-identity/service-principals.md), [Managed Identities](../02-identity/managed-identities.md), [Federation](../02-identity/federation.md)

## 6. Build the core networking layer

- vNet, subnets, NSGs, and route tables
- Deploy as a separate **foundation template**, because networking changes less often than application resources
- Later templates **reference the vNet by ID or output** instead of redefining it

Ref also: [Subnets and Routing](../01-networking/subnets-and-routing.md), [Firewalls](../01-networking/firewalls.md)

## 7. Author modular IaC templates

Split Bicep into **modules** with clear parameters and outputs, instead of one giant template:

```
infra/
├── main.bicep              ← orchestrates the modules
├── main.dev.bicepparam     ← dev values
├── main.prod.bicepparam    ← prod values
└── modules/
    ├── network.bicep
    ├── compute.bicep
    ├── storage.bicep
    └── identity.bicep
```

The same `main.bicep` is deployed to every environment. Only the parameter file changes.

## 8. Validate, what-if, then deploy via pipeline

Check before deploying:

```bash
# Catch errors in the template
az deployment group validate --resource-group rg-appname-dev \
  --template-file main.bicep --parameters main.dev.bicepparam

# Preview what will be created, changed, or deleted
az deployment group what-if --resource-group rg-appname-dev \
  --template-file main.bicep --parameters main.dev.bicepparam

# Deploy
az deployment group create --resource-group rg-appname-dev \
  --template-file main.bicep --parameters main.dev.bicepparam
```

For subscription-scope deployments (for example, when the template creates the resource group), use `az deployment sub ...` with `--location` instead.

Ref also: [ARM Templates and Bicep](../05-compute/07-arm-templates-and-bicep.md)

Run this through a **CI/CD pipeline** (GitHub Actions or Azure DevOps), so deployments are **repeatable and reviewed** – not run ad hoc from someone's laptop.

---

## How this maps to AZ-104

| Step | AZ-104 domain |
|---|---|
| 1, 2 | Manage Azure identities and governance – tags, Azure Policy, management groups |
| 5 | Manage Azure identities and governance – RBAC, managed identities |
| 6 | Implement and manage virtual networking |
| 3, 7, 8 | Deploy and manage Azure compute resources – ARM templates and Bicep |
