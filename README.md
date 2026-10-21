# AZ-104 Study Notes

My study notes for the **Microsoft Azure Administrator (AZ-104)** certification.

The exam is scenario-based: it checks that I can **deploy, manage, monitor, and troubleshoot** Azure in a live environment – not just repeat theory. So every note is built for hands-on practice:

> **In one line → When to use it → Key concepts → Exam traps → Do it (Portal / CLI / PowerShell) → Troubleshooting → Clean up → Check yourself**

Labs also have a **Break it and fix it** section. How I study with this format: [How to study hands-on](00-guides/how-to-study-hands-on.md).

## Exam domains

| Domain | Weight | Folder |
|---|---|---|
| Manage Azure identities and governance | 20–25% | [02-identity](02-identity/), [04-governance](04-governance/) |
| Implement and manage storage | 15–20% | [03-storage](03-storage/) |
| Deploy and manage Azure compute resources | 20–25% | [05-compute](05-compute/) |
| Implement and manage virtual networking | 15–20% | [01-networking](01-networking/) |
| Monitor and maintain Azure resources | 10–15% | *Week 4 – notes coming* |

## Contents

### 0. Guides / best practice
- [How to study hands-on](00-guides/how-to-study-hands-on.md) – note template, study day, lab hygiene, troubleshooting method, error codes
- [Deployment checklist](00-guides/az-project-skeleton.md)
- [Command recognition (governance)](00-guides/command-recognition-gov.md)
- [VM commands – CLI and PowerShell](00-guides/CLI-azure-commands.md)
- [Naming conventions](01-networking/naming-conventions.md)

### 1. Networking
- [Network Fundamentals – Overview](01-networking/network-fundamentals.md) – incl. Network Watcher toolkit and a practice track
- [IP Addressing and Subnetting](01-networking/ip-addressing-subnetting.md)
- [DNS](01-networking/dns.md) – Azure DNS, private DNS zones
- [Ports and Protocols](01-networking/ports-and-protocols.md)
- [Virtual Networks and VNet Peering](01-networking/vnet-peering.md)
- [Subnets and Routing](01-networking/subnets-and-routing.md) – UDRs, next hop
- [NSG Inbound and Outbound Rules](01-networking/nsg-inbound-outbound-rules.md) – NSGs, ASGs, service tags
- [Firewalls](01-networking/firewalls.md) – Azure Firewall, WAF
- [NAT and NAT Gateway](01-networking/nat.md)
- [Application Gateway](01-networking/application-gateway.md)
- [Hybrid Connectivity: VPN Gateway, ExpressRoute, Virtual WAN](01-networking/expressroute-and-virtual-wan.md)
- [Container Networking](01-networking/container-networking.md)
- Labs: [VNet Peering with PowerShell](01-networking/labs/vnet-peering-lab.md) · [Application Gateway](01-networking/labs/application-gateway-lab.md)

### 2. Identity
- [Entra ID Labs – Overview](02-identity/entra-id-labs.md) – incl. an end-to-end practice track
- [Entra ID and Users](02-identity/entra-users.md)
- [Groups and Membership](02-identity/groups.md)
- [Entra ID Licenses](02-identity/entra-licenses.md)
- [Guest Users](02-identity/guest-users.md)
- [Self-Service Password Reset](02-identity/sspr.md)
- [Custom Roles](02-identity/custom-roles.md)
- [Entra Roles vs Azure RBAC](02-identity/entra-roles-vs-rbac.md)
- [Managed Identities](02-identity/managed-identities.md)
- [Privileged Identity Management](02-identity/pim.md)
- [App Registrations and Enterprise Applications](02-identity/app-registration.md)
- [Access Reviews](02-identity/access-review.md)
- [Conditional Access](02-identity/conditional-access.md)
- [Service Principals](02-identity/service-principals.md)
- [Federation](02-identity/federation.md)

### 3. Storage
- [Storage Overview](03-storage/storage-overview.md)
- [Storage Redundancy](03-storage/storage-redundancy.md)
- [Blob Tiers, Lifecycle Rules, and Data Protection](03-storage/blob-tiers-lifecycle.md)
- [Storage Access](03-storage/storage-access.md) – keys, SAS, Entra ID data roles
- [Storage Network Security](03-storage/storage-network-security.md)
- [Storage Encryption and Azure Key Vault](03-storage/storage-encryption-key-vault.md)
- [Azure Files](03-storage/azure-files.md)
- [Azure File Sync](03-storage/azure-file-sync.md)
- [AzCopy, Storage Explorer, and Data Box](03-storage/azcopy-storage-explorer.md)
- Labs: [Storage Account with Private Access](03-storage/lab/storage-account-with-private-access.md) · [Customer-Managed Keys](03-storage/lab/storage-cmk-lab.md)

### 4. Governance
- [Why Governance?](04-governance/why-governance.md) – incl. a governance practice lab
- [Scope Hierarchy](04-governance/scope-hierarchy.md) – management groups, subscriptions, moving resources
- [Inheritance](04-governance/inheritance.md)
- [Azure RBAC](04-governance/azure-rbac.md)
- [Azure Policy](04-governance/azure-policy.md)
- [Resource Locks](04-governance/resource-locks.md)
- [Tags](04-governance/tags.md)
- [Cost Management, Advisor, and Budgets](04-governance/cost-advisor-budget.md)
- [Role cheat sheet](04-governance/cheat-sheet.md)

### 5. Compute
- [App Services](05-compute/01-app-services.md)
- [Containers](05-compute/02-containers.md)
- [VMs – Planning and Deployment](05-compute/03-vms-plan-deployment.md)
- [VMs – Operate and Move](05-compute/04-vms-operate-move.md)
- [Availability](05-compute/05-availability.md) – sets, zones, load balancer
- [VM Scale Sets](05-compute/06-vm-scale-sets.md)
- [ARM Templates and Bicep](05-compute/07-arm-templates-and-bicep.md)

### Still to write
- **Monitoring and maintenance:** Azure Monitor, Log Analytics and KQL, alerts and action groups, Network Watcher, Azure Backup, Site Recovery
- **Networking extras:** Azure Load Balancer (own note), Azure Bastion, service endpoints vs private endpoints (beyond storage)
- **AKS overview**

## Learning Log
See [learning-log.md](learning-log.md) for what I learned each day.

## 🔗 Resources
- [AZ-104 exam page](https://learn.microsoft.com/en-us/credentials/certifications/azure-administrator/)
- [AZ-104 learning paths on Microsoft Learn](https://learn.microsoft.com/en-us/training/courses/az-104t00)
- [Azure CLI reference](https://learn.microsoft.com/en-us/cli/azure/reference-docs-index?view=azure-cli-latest)
- [Azure PowerShell reference](https://learn.microsoft.com/en-us/powershell/azure/)
