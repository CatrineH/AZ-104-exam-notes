Questions:

You plan to deploy an Azure App Service web app that will have the following settings:
- Name: WebApp1
- Publish: Docker Container
- Operating system: Windows
- Region: West US
- Windows Plan (West US): ASP-RG1-8bcf
You need to ensure that WebApp1 can use the ASP.NET V4.8 runtime stack.
Which setting should you modify?

Select only one answer.

- Operating system
- Publish
- Region
- Windows Plan

---

You have an Azure App Service web app named Contoso2025 that runs in the Standard App Service plan. Contoso2025 has five deployment slots in use.
A user named User1 has the Contributor role for Contoso2025.
You need to ensure that User1 can create additional deployment slots to Contoso2025.
What should you do?

Select only one answer.

- Assign User1 the Owner role for Contoso2025.
- Assign User1 the Website Contributor role for Contoso2025.
- Scale out the Contoso2025 App Service plan.
- Scale up the Contoso2025 App Service plan.

---

You have an Azure subscription that includes a virtual network named VNet1.
You plan to create an App Service web app named WebApp1 and deploy it to VNet1.
You need to prepare the environment for the planned web app. The solution must minimize costs.
Which app service plan size should you use?

Select only one answer.

- Dev / Test B1
- Dev / Test F1
- Isolated I1V2
- Production P1V2
- Production P3V3

---

You have an Azure App Service web app named WebApp1 that runs in the Standard App Service plan.
You plan to add deployment slots for WebApp1.
What is the maximum number of deployment slots that you can add to WebApp1?

Select only one answer.

- 1
- 4
- 6
- 9

---

You have an Azure container registry that stores an image named Image1 and a Windows Server Azure virtual machine named VM1.
You need to ensure that you can run Image1 in VM1.
What should you install in VM1?

Select only one answer.

- Azure Storage Explorer
- Docker
- Hyper-V role
- .NET Framework 4.8

---

You have Azure subscription that includes virtual network named VNet1 in West US region.

You plan to deploy following container instances:
- instance1, running Windows container image in West US region
- instance2, running Linux container image in West US region
- instance3, running Windows container image in East US region
Which container instances can be deployed to VNet1?

Select only one answer.

- instance1 and instance2 only
- instance1 and instance3 only
- instance1, instance2 and instance3
- instance1 only
- instance2 only

---

You have Azure subscription that includes virtual network with following subnets:
- Subnet1, which has connected virtual machine
- Subnet2, which has connected app service web app
- Subnet3, which has deployed container instance
You plan to deploy container instance named container1.
To which subnets can you deploy container1?

Select only one answer.

- Subnet1 and Subnet3 only
- Subnet1, Subnet2 and Subnet3
- Subnet2 and Subnet3 only
- Subnet3 only

---

You have an Azure storage account that contains a blob container named container1.
You need to configure access to container1.
Which authorization types can you use?

Select only one answer.

- Microsoft Entra ID only
- Microsoft Entra ID, shared access signature or certificate only
- Microsoft Entra ID, storage key or shared access signature only
- Microsoft Entra ID, storage key, shared access signature or certificate
- Storage key or shared access signature only

---

You have an Azure subscription.

You plan to create a storage account that contains the following settings:
- Name: storage1
- Performance: Standard
- Redundancy: Zone-redundant storage (ZRS)
What is the minimum number of copies of storage1 data stored in Azure?

Select only one answer.

- 2
- 3
- 6
- 9

---

You have an Azure storage account named storage1.
You need to ensure that a user named User1 can access storage1 only from January 1st to January 31st 2026.
What should you do?

Select only one answer.

- Create a conditional access policy for User1
- Provide User1 with a shared access signature (SAS)
- Provide User1 with a storage1 access key
- Use a condition when assigning User1 an RBAC role on storage1

---

You have an Azure subscription that contains the following storage accounts:
- storage1, configured as StorageV2 kind
- storage2, configured as BlobStorage kind
- storage3, configured as FileStorage kind
Which storage account or storage accounts can you use Lifecycle management?

Select only one answer.

- storage1 and storage2 only
- storage1 and storage3 only
- storage1 only
- storage1, storage2 and storage3
- storage2 and storage3 only

---

You have an Azure subscription that contains a storage account named storage1 and the following virtual machines:
- VM1 has a public IP address of 13.68.158.24 and is connected to VNET1/Subnet1
- VM2 has a public IP address of 52.255.145.76 and is connected to VNET1/Subnet1
- VM3 has a public IP address of 13.68.158.50 and is connected to VNET1/Subnet2
The subnets have the following service endpoints:
- Subnet1 has a Microsoft.Storage service endpoint
- Subnet2 does NOT have any service endpoint
Storage1 has a firewall configured to allow access from the 13.68.158.0/24 IP address range only.
Which virtual machines can access storage1.

Select only one answer.

- VM1 and VM2 only
- VM1 and VM3 only
- VM1 only
- VM1, VM2, and VM3
- VM3 only

---

You have an Azure subscription that includes a network security group named NSG1.
You plan to add an inbound security rule named Rule1 to NSG1.
You need to configure a priority for Rule1. Rule1 must have the highest priority for inbound security rules in NSG1.
Which priority should you configure for Rule1?

Select only one answer.

- 0
- 1
- 10
- 100
- 1000

---

You have an Azure subscription that includes following resources:
- VNet1, a virtual network
- Subnet1, a subnet in VNet1
- VM1, a virtual machine
- NIC1, a network interface of VM1
- LB1, a load balancer
You create a network security group named NSG1.
To which two Azure resources can you associate NSG1?

Select all answers that apply.

- LB1
- NIC1
- Subnet1
- VM1
- VNet1

---

You have an Azure subscription that contains a virtual machine named VM1 and a storage account named storage1.
You need to ensure that VM1 can access storage1 by using the Azure backbone.
What should you configure?

Select only one answer.

- a routing table
- a service endpoint
- a VPN gateway
- Peering

---

You have an Azure subscription that contains an Azure disk named Disk1.
You plan to use an Azure Backup to backup Disk1.
What should you deploy first?

Select only one answer.

- a Backup vault
- a Storage account
- a Recovery Services vault
- an Azure Backup Server

---

You have the following Azure resources:
- a virtual machine named VM1
- a Recovery Services vault named Vault1
On January 1, you configure backups for VM1 by using the following standard backup policy:
- Frequency: Daily
- Time: 23:00
- Timezone: (UTC) Coordinated Universal Time
- Retain instant recovery snapshot(s) for: 2 Day(s)
- Retention of daily backup point: 7 Day(s)
- Azure Backup Resource Group: Backup1RG
How many restore point collections recovery points will be stored in Backup1RG on January 10?

Select only one answer.

- 2
- 7
- 9
- 10

---

Your company’s Azure subscription contains an Azure virtual machine.
You need to back up the virtual machine every 12 hours.
What should you create first?

Select only one answer.

- a backup policy in a backup vault
- a standard backup policy in a recovery services vault
- an enhanced backup policy in a recovery services vault

---

You have an Azure subscription.
You need to deploy a virtual machine that runs Windows Server 2025 Datacenter: Azure Edition.
Which security type can you select when deploying virtual machine?

Select only one answer.

- Standard only
- Standard or trusted launch only
- Standard, trusted launch or confidential
- Trusted launch or confidential only

---

You have an Azure subscription named Sub1.
You plan to deploy a virtual machine scale set named VMSS1 in East US region. VMSS1 will have 8 instances.
What is the maximum number of availability zones that VMSS1 can use?

Select only one answer.

- 1
- 2
- 3
- 4

---

You have an Azure subscription.
You plan to use fault domains.
From which Azure resource can you configure the fault domains?

Select only one answer.

- Availability set
- Virtual machine
- Virtual machine scale set
- Virtual network

---

You have an Azure subscription that contains the following resources:
- A resource group named RG1 in the West US region
- A resource group named RG2 in the Central US region
- A virtual network named VNet1 that is deployed to the West US region in RG1
- A virtual network named VNet2 that is deployed to the Central US region in RG1
- A virtual network named VNet3 that is deployed to the West US region in RG2
You need to deploy a virtual machine named VM1 to RG1 in the West US region.

To which virtual network or virtual networks can you connect VM1?

Select only one answer.

- VNet1 only
- VNet1 or VNet2 only
- VNet1 or VNet3 only
- VNet1, VNet2 or VNet3

---

You have a Microsoft Entra tenant named contoso.com that contains a user named Ben Smith.

You configure a Password protection for contoso.com that includes the following Custom banned passwords settings:
- Enforce custom list: Yes
- Custom banned password list: Contoso
Which password can be used by Ben Smith?

Select only one answer.

- C0nt0s0123
- CONTOSO123
- Conto123so
- FgRs01

---

You have an Azure subscription that contains a user named User1, a resource group named RG1, and a virtual machine named VM1.
You enable a system-assigned managed identity for VM1.
To which identities can you assign the Reports Reader role?

Select only one answer.

- User1 and RG1 only
- User1 and VM1 only
- User1 only
- User1, RG1, and VM1

---

You have an Azure subscription that contains a resource group named RG1 and a security group named Helpdesk. Helpdesk contains five support engineers.
You need to enable the engineers to start, stop, and configure the virtual machines in RG1. The engineers must NOT be able to grant access to Azure resources.
Which built-in role should you assign to Helpdesk at the RG1 scope?

Select only one answer.

- Owner
- Contributor
- Reader
- User Administrator
