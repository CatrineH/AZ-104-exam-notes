# Learning Log

[← Back to README](README.md)

## September 2026

### Week 36–38 · Mon 1 – Fri 18 September · Networking
- [Network Fundamentals – Overview](01-networking/network-fundamentals.md) – the Business-A scenario, moving to the cloud
- [Virtual Networks and VNet Peering](01-networking/vnet-peering.md)
- [Lab: VNet Peering with PowerShell](01-networking/labs/vnet-peering-lab.md) – two VNets, peering in both directions, NSG, VM, and testing
- [DNS](01-networking/dns.md)
- [Ports and Protocols](01-networking/ports-and-protocols.md) – standard ports, TCP vs UDP
- [Subnets and Routing](01-networking/subnets-and-routing.md)
- [Firewalls](01-networking/firewalls.md) – host and network firewalls
- [NAT](01-networking/nat.md)
- [ExpressRoute and Virtual WAN](01-networking/expressroute-and-virtual-wan.md) – hybrid connectivity and global transit
- [Container Networking](01-networking/container-networking.md) – Docker and Kubernetes

### Week 39 · Mon 21 – Fri 25 September · Microsoft Entra ID
- [Entra ID Labs – Overview](02-identity/entra-id-labs.md) – 14 labs: identities, access to Azure resources, access governance

### Week 40 · Mon 28 September
- [Naming conventions](01-networking/naming-conventions.md) – RG, vNet, sNet
- [IP addressing](01-networking/ip-addressing-subnetting.md#ip-addressing) – host ranges, bits and decimal positions, bit calculation
- [CIDR notation](01-networking/ip-addressing-subnetting.md#cidr-notation) – how many bits to mask per network
- [Subnetting](01-networking/ip-addressing-subnetting.md#subnetting-in-azure-29-example) – /29 example with Azure reserved addresses


### Week 40 · Tue 29 September - Storage
- [Service Principals](02-identity/service-principals.md)
- [Storage redundancy](03-storage/storage-redundancy.md) – LRS, ZRS, GRS, RA-GRS, GZRS, RA-GZRS
- [Storage Account Keys](03-storage/storage-access.md#storage-account-keys)
- [Shared Access Signatures (SAS)](03-storage/storage-access.md#shared-access-signatures-sas)
- [Azure Storage Explorer](03-storage/storage-access.md#azure-storage-explorer)
- [PowerShell for storage access](03-storage/storage-access.md#powershell)
- [Managed Identities](02-identity/managed-identities.md) – system-assigned and user-assigned
- [Federation](02-identity/federation.md)

### Week 40 · Wed 30 September - Storage
- [Storage Network Security](03-storage/storage-network-security.md) – public network access, service endpoints, private endpoints, private DNS zones
- [Lab: Storage Account with Private Access](03-storage/lab/storage-account-with-private-access.md) – jump server, Azure CLI on Linux, PowerShell on Windows, service endpoint, private endpoint, `privatelink.blob.core.windows.net`

## October 2026

### Week 40 · Thu 1 October - Storage
- [Azure Files](03-storage/azure-files.md) – SMB/NFS, tiers, authentication, snapshots, Azure File Sync
- [Lab: Azure File Share with CLI](03-storage/azure-files.md#do-it) – create share, upload with CLI, mount on Linux and Windows VMs

### Week 41 · Mon 5-13 October - Governance and ID
- [NSG Inbound and Outbound Rules](01-networking/nsg-inbound-outbound-rules.md)
- [Azure file-sync](03-storage/azure-file-sync.md)
- [Why Governance?](04-governance/why-governance.md)
- [Azure RBAC](04-governance/azure-rbac.md)
- [Tags](04-governance/tags.md)
- [AzCopy and Storage Explorer](03-storage/azcopy-storage-explorer.md)

### Week 42 - Wed 14-16 October - Compute
- [App Service](05-compute/01-app-services.md)
- [Containers](05-compute/02-containers.md)
- [VM's planning and deployment](05-compute/03-vms-plan-deployment.md)

### Week 43 - Mon 19-22 October - Compute
- [VM's operate and move](05-compute/04-vms-operate-move.md)
- [Availability](05-compute/05-availability.md)
- [Virtual Machine Scale-Sets](05-compute/06-vm-scale-sets.md)
- [ARM templates and Bicep](05-compute/07-arm-templates-and-bicep.md)
- AKS overview (notes coming)
