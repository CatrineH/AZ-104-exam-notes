# Virtual machines: plan and deploy 

AZ-104 domain: Deploy and manage Azure compute resources -VM's planning and deployment

Date learned: 2026-10-16

[← Back to README](../README.md)

## In one line
A **VM** is IaaS: you get a full computer in Azure (built from VM + disks + NIC + optional public IP + NSG), and **you** manage the OS, patching, and software.

## When to use it (and when not)
| Need | Use |
|---|---|
| Full OS control, custom/legacy software, lift-and-shift of a server | **Virtual machine** |
| Many identical VMs that scale | **VM Scale Set** |
| Web app/API without managing the OS | **App Service** |
| Run a container | **ACI / Container Apps** |
| Admin access without public IPs | **Bastion** (not a public IP on every VM) |

---

## The big picture

A virtual machine (VM) is IaaS: I get a full computer in Azure, and **I** manage the OS, patching and software.

This deck covers:

1. What a VM is made of
2. Sizes, quotas and images
3. Disks and encryption
4. How to connect to a VM safely
5. Azure Policy guardrails

---

## 1. What a VM is made of

A VM is not one resource. It is **six resources** that work together.

| Resource | What it is |
|---|---|
| Virtual machine | The size, image and OS |
| OS disk | The system drive |
| Data disks | Extra drives (optional) |
| Network interface (NIC) | Gives the VM a private IP in **one subnet** |
| Public IP | Optional, attached to the NIC |
| Network security group (NSG) | The firewall, on the NIC or the subnet |

### The rules the exam tests

1. **The NIC and the VM must be in the same region and subscription.** The resource group can be different.
2. **All NICs on a VM must be in the same VNet.**
3. **A VM cannot change VNet.** Delete the VM, keep the disks, and recreate it with a NIC in the new VNet.
4. **A resource group's location is only metadata.** A VM in East Asia can be in a resource group located in West US.

The classic exam question: *VM1 is in VNET1. Connect it to VNET2.*

| Solution | Works? |
|---|---|
| Add a new NIC in VNET2 | No (same VNet rule) |
| Turn off VM1, then add a NIC in VNET2 | No (still the same VNet rule) |
| Delete VM1 and recreate it with a NIC in VNET2 | **Yes** |

### Counting NICs and NSGs

5 VMs in one subnet with identical firewall rules need:

- **5 NICs** (one per VM; a NIC can carry both a private and a public IP)
- **1 NSG** on the subnet (one NSG can serve many subnets and NICs)

NSG facts:

- Attach it to a subnet, a NIC, or both
- If both, inbound traffic must pass the subnet NSG **and then** the NIC NSG
- Rules have priorities 100-4096. **The lowest number wins.**

### Deleting a VM leaves parts behind

When I delete only the VM, the **disks, NIC and public IP stay**, and disks and static public IPs **keep costing money**.

To avoid this:

- Tick **"Delete with VM"** on the Disks and Networking tabs in the portal
- Or use `--os-disk-delete-option Delete` and `--nic-delete-option Delete` in CLI
- Simplest: one resource group per workload, then delete the whole group

The NSG and VNet are never deleted with the VM.

---

## 2. Sizes, quotas and images

### VM families

| Family | Remember it as | Use for |
|---|---|---|
| **B** | Burstable | Dev, test, small servers. Saves credits while idle. Not for steady load. |
| **D** | General purpose | Most production servers |
| **E** | Memory optimised | Databases and caches |
| **F** | Compute optimised | High CPU, batch jobs |
| L | Storage optimised | Big local disks |
| M | Large memory | SAP HANA |
| N | GPU | Machine learning, rendering |
| H | High performance | Simulations |

### Reading a size name

`Standard_D2s_v5` = tier, **D** family, **2** vCPUs, **s** features, version 5

The two letters to remember:

- **s** = can use **Premium SSD** (and Ultra) disks
- **d** = has a **local temporary disk**

Other letters: **a** = AMD, **p** = ARM, **i** = isolated hardware.

### Size, quota or capacity?

| Error | Means | Fix |
|---|---|---|
| SkuNotAvailable | The size is not offered in that region or zone | Choose another size or region |
| **QuotaExceeded** | I have used my vCPU allowance | Request a quota increase |
| AllocationFailed | Azure has no capacity right now | Retry, or change size or zone |

Quotas are per **region** and per **VM family**. Check them under **Subscription > Usage + quotas**.

### Images

- Images come from the **Marketplace**, or my own from an **Azure Compute Gallery**
- **Generation 2** (UEFI) is the default for new VMs
- **Trusted Launch** (Secure Boot and virtual TPM) needs Gen 2
- **Generalised** image = a template for many VMs (sysprep or deprovision first)
- **Specialised** image = a copy of one machine, including its name and accounts

---

## 3. Disks and encryption

### Three kinds of disk

| Disk | Survives a restart? | Remember |
|---|---|---|
| OS disk | Yes | The system drive |
| Data disk | Yes | Extra drives; can move between VMs |
| **Temporary disk** | **No** | Lost on redeploy, resize and deallocate. **Never store data on it.** |

The temporary disk is `D:` on Windows and `/mnt` on Linux, and only exists on sizes with a **d**.

### Five disk types

| Type | Use for | Single-VM SLA |
|---|---|---|
| Standard HDD | Backup, archive | 95% |
| Standard SSD | Web, dev, test | 99.5% |
| **Premium SSD** | Production | **99.9%** |
| Premium SSD v2 | IO-heavy production (not an OS disk) | 99.9% |
| Ultra Disk | Top-tier databases (not an OS disk) | 99.9% |

- Premium SSD, v2 and Ultra need an **"s" size**
- The 99.9% single-VM SLA needs Premium SSD or Ultra on **every** disk
- On Premium SSD, a bigger disk = more IOPS. For **more IOPS without more space**, set a **higher performance tier** instead of making the disk bigger.

### Changing disks

| Change | VM state |
|---|---|
| Grow a data disk | Can stay running |
| Grow an OS disk | **Deallocated** |
| Change type (for example Standard to Premium) | **Deallocated** |
| Set a higher performance tier | Can stay running |
| **Shrink a disk** | **Not possible.** Copy the data to a smaller disk. |

After growing a disk, I must also grow the partition and file system inside the OS.

### Snapshots and shared disks

- A **snapshot** is a point-in-time copy of a disk. **Incremental** snapshots are the cheaper default.
- To restore: create a new disk from the snapshot and attach it.
- A snapshot is **not** a backup plan, because it has no schedule or retention. Use Azure Backup for that.
- **Shared disks** are for cluster software only. To share files between VMs, use **Azure Files**.

### Encryption: four options

| Option | Remember it as |
|---|---|
| **SSE with platform keys** | **Always on.** Microsoft manages the keys. Nothing to set up. |
| SSE with customer keys | My own key in Key Vault, through a disk encryption set |
| **Encryption at host** | Also encrypts the **temporary disk and caches** |
| Azure Disk Encryption (ADE) | Encrypts inside the OS (BitLocker or dm-crypt). Being retired in 2028, but still in older exam questions. |

To turn on **encryption at host**:

1. Register the `EncryptionAtHost` feature on the subscription
2. Choose a size that supports it
3. **Deallocate** the VM
4. Enable encryption at host, then start the VM

I cannot use encryption at host and ADE on the same VM.

---

## 4. Connecting to a VM

| Method | Public IP on the VM? | Remember |
|---|---|---|
| SSH or RDP directly | Yes | Ports 22 or 3389 open to the internet. **Avoid.** |
| **Azure Bastion** | **No** | Connect through the portal over HTTPS (443) |
| Just-in-time (JIT) access | Optional | Opens the port only for a set time |
| Entra ID login | Either | Sign in with my Entra account instead of a local password |
| Run Command, serial console | No | Fallbacks when the network is broken |

### Azure Bastion

- A managed jump host inside my VNet, so the VM needs **no public IP**
- Needs a subnet named exactly **AzureBastionSubnet**, at least **/26**
- SKUs: Developer (free, shared), Basic, Standard (native client and file transfer), Premium (session recording)

### Just-in-time access

- The port stays **closed** until someone requests access
- Defender for Cloud opens it for a set time (up to 24 hours), then closes it again
- Needs **Defender for Servers Plan 2**
- Exam phrase: *"allow RDP only when needed, for 3 hours"* = JIT

### Entra ID login

- Install the extension on the VM
- Give users one of these roles:
  - **Virtual Machine Administrator Login** (admin rights)
  - **Virtual Machine User Login** (normal user)
- I get MFA and Conditional Access, and no local passwords to manage

---

## 5. Azure Policy guardrails

Azure Policy checks every deployment against rules I assign.

| Effect | What happens |
|---|---|
| **Deny** | The request fails. Nothing is deployed. |
| **Audit** | Allowed, but marked non-compliant |
| Modify | Adds or changes properties, such as tags |
| DeployIfNotExists | Deploys something extra afterwards, such as an extension |
| Disabled | The rule is switched off |

- I assign a policy to a **management group, subscription or resource group**
- Policies **inherit downwards** to child scopes
- An **initiative** is a group of policies assigned together

### Three useful VM policies

| Policy | Blocks |
|---|---|
| Allowed locations | Resources outside the regions I list |
| Allowed virtual machine size SKUs | Sizes not on my list |
| Network interfaces should not have public IPs | NICs with a public IP |

When blocked, I get the error **RequestDisallowedByPolicy**.

### The important rule

**Deny only blocks new requests.** Resources that already exist are **not deleted or changed**. They are only flagged as **non-compliant**.

---

## 6. Top exam traps

| Wrong | Right |
|---|---|
| Turn the VM off and add a NIC in another VNet | All NICs share one VNet. **Delete and recreate** the VM. |
| A VM must be in the same region as its resource group | The resource group's location is **only metadata** |
| Create the NIC in another region | The NIC and VM must be in the **same region** |
| Shrink a disk to save money | Disks **never shrink** |
| Make the disk bigger to get more IOPS | Set a **higher performance tier** |
| Assign a deny policy to remove existing VMs | Deny only blocks **new** requests |
| Store data on the temporary disk | It is **lost** on redeploy, resize and deallocate |

---

## Quick practice (Azure CLI)

My trainee subscription may block B and D sizes, so this example uses the lab size. Change `SIZE` if needed.

```bash
RG=catrineh-vm-rg
LOC=norwayeast
VM=catrineh-vm-01
SIZE=Standard_FX2ms_v2

# 1. Resource group and a VNet with one subnet
az group create -n $RG -l $LOC
az network vnet create -g $RG -n catrineh-vnet --address-prefix 10.10.0.0/16 \
  --subnet-name subnet-web --subnet-prefix 10.10.1.0/24

# 2. Check that the size exists in the region, and check your quota
az vm list-skus -l $LOC --size $SIZE --output table
az vm list-usage -l $LOC --output table

# 3. Create a Linux VM with NO public IP, and clean-up options set
az vm create -g $RG -n $VM -l $LOC \
  --image Ubuntu2404 --size $SIZE \
  --admin-username azureuser --generate-ssh-keys \
  --vnet-name catrineh-vnet --subnet subnet-web \
  --public-ip-address "" \
  --storage-sku StandardSSD_LRS \
  --os-disk-delete-option Delete --nic-delete-option Delete

# 4. Run a command without SSH or a public IP
az vm run-command invoke -g $RG -n $VM \
  --command-id RunShellScript --scripts "hostname && df -h"

# 5. Add a 32 GiB data disk, then grow it to 64 GiB
az vm disk attach -g $RG --vm-name $VM -n data1 \
  --new --size-gb 32 --sku StandardSSD_LRS
az disk update -g $RG -n data1 --size-gb 64

# 6. Take an incremental snapshot of the data disk
az snapshot create -g $RG -n data1-snap --source data1 --incremental true

# 7. Change the disk type to Premium (the VM must be deallocated)
az vm deallocate -g $RG -n $VM
az disk update -g $RG -n data1 --sku Premium_LRS
az vm start -g $RG -n $VM

# 8. Assign "Allowed locations" to the resource group, then test it
az policy assignment create -n allowed-locations \
  --policy e56962a6-4747-49cd-b67b-bf8b01975c4c \
  --scope $(az group show -n $RG --query id -o tsv) \
  --params '{"listOfAllowedLocations":{"value":["norwayeast"]}}'

#    This should fail with RequestDisallowedByPolicy
#    (wait a few minutes after assigning the policy)
az disk create -g $RG -n test-disk -l westeurope --size-gb 4

# 9. Clean up
az group delete -n $RG --yes --no-wait
```

---

## PowerShell equivalents

```powershell
$rg = "catrineh-vm-rg"; $loc = "norwayeast"

New-AzResourceGroup -Name $rg -Location $loc

# Check sizes and quota
Get-AzVMSize -Location $loc | Where-Object Name -like "*D2s*"
Get-AzVMUsage -Location $loc | Where-Object { $_.CurrentValue -gt 0 }

# Create a Windows VM (simple parameter set creates VNet/NSG/public IP if missing)
$cred = Get-Credential
New-AzVM -ResourceGroupName $rg -Name vm-win01 -Location $loc `
  -Image Win2022Datacenter -Size Standard_D2s_v5 -Credential $cred `
  -VirtualNetworkName catrineh-vnet -SubnetName subnet-web -OpenPorts 3389

# Add a data disk
$vm = Get-AzVM -ResourceGroupName $rg -Name vm-win01
$vm = Add-AzVMDataDisk -VM $vm -Name data1 -CreateOption Empty -DiskSizeInGB 32 -Lun 0
Update-AzVM -ResourceGroupName $rg -VM $vm

# Grow it
New-AzDiskUpdateConfig -DiskSizeGB 64 | Update-AzDisk -ResourceGroupName $rg -DiskName data1

# Incremental snapshot
$disk = Get-AzDisk -ResourceGroupName $rg -DiskName data1
$cfg  = New-AzSnapshotConfig -SourceUri $disk.Id -Location $loc -CreateOption Copy -Incremental
New-AzSnapshot -ResourceGroupName $rg -SnapshotName data1-snap -Snapshot $cfg

# Run a command without RDP
Invoke-AzVMRunCommand -ResourceGroupName $rg -VMName vm-win01 -CommandId RunPowerShellScript -ScriptString 'hostname; Get-Volume'
```

---

## Troubleshooting (symptom → cause → fix)

| Symptom | Likely cause | Fix |
|---|---|---|
| **SkuNotAvailable** | Size not offered in that region/zone (or restricted for your subscription) | `az vm list-skus -l <region> --size <size>` – choose another |
| **QuotaExceeded** (cores) | vCPU quota per region/family used up | Usage + quotas → request increase |
| **AllocationFailed** | No capacity right now | Retry, other size/zone |
| **RequestDisallowedByPolicy** | A policy (allowed sizes/locations, no public IP) | Change the deployment or ask for an exemption |
| Can't RDP/SSH | VM stopped, no public IP, **NSG** denies 3389/22, OS firewall, wrong IP | Check power state, **IP flow verify**, effective security rules, use **Bastion** |
| RDP works but password unknown | Forgotten local admin password | **Reset password** blade / `az vm user update` |
| VM doesn't boot | OS/disk problem | **Boot diagnostics** (screenshot), **Serial console** |
| Premium SSD not offered | Size has no **s** | Choose an "s" size |
| "Maximum number of data disks exceeded" | Size limit on data disks | Bigger size |
| Can't change OS disk size/type | VM is running | **Deallocate** first |
| Can't make a disk smaller | Not supported | New smaller disk + copy data |
| Disks/public IP still billed after deleting VM | Not set to "Delete with VM" | Delete them, or delete the RG |

---

## Break it and fix it

| # | Break it | What you should see | Fix |
|---|---|---|---|
| 1 | Add an NSG rule **Deny 22, priority 100** on the subnet | SSH/Bastion times out | Use **IP flow verify** to find the rule, delete it |
| 2 | `az disk update --size-gb 16` on the 64 GiB disk | Error – disks can't shrink | – |
| 3 | `az disk update --sku Premium_LRS` on the **OS disk** while running | Error – must be deallocated | Deallocate, retry, start |
| 4 | Create a VM with a size not in `list-skus` | SkuNotAvailable | Pick a listed size |
| 5 | Deny policy "Allowed locations" then deploy elsewhere | RequestDisallowedByPolicy | Deploy in an allowed region |

---

## Test yourself

1. Which two resources must always be in the same region?
2. Can a VM have NICs in two different VNets?
3. How do you move a VM to another VNet?
4. 5 VMs in one subnet need identical firewall rules. How many NSGs do you need?
5. What does the "s" in `Standard_D2s_v5` mean?
6. You get QuotaExceeded. What do you do?
7. Which disk is lost when you deallocate a VM?
8. Can you shrink a managed disk?
9. You need more IOPS but not more space on a Premium SSD. What do you do?
10. Which encryption option is always on?
11. How can you connect to a VM that has no public IP, using only a browser?
12. A deny policy is assigned. What happens to existing VMs that break the rule?

<details>
<summary>Answers</summary>

1. The VM and its NIC (and its disks and VNet).
2. No. All NICs must be in the same VNet.
3. Delete the VM, keep the disks, and recreate it with a NIC in the new VNet.
4. One, on the subnet.
5. It can use Premium SSD disks.
6. Request a quota increase.
7. The temporary disk.
8. No. Copy the data to a smaller disk instead.
9. Set a higher performance tier.
10. Server-side encryption with platform-managed keys.
11. Azure Bastion.
12. Nothing is deleted. They are marked non-compliant.

</details>

### Exam-style question 1

VM1 is connected to VNET1 through NIC1. VNET1 and VNET2 are in the same region and resource group. You need to connect VM1 to VNET2. Does each solution meet the goal?

| Proposed solution | Yes / No |
|---|---|
| 1. Create a new network interface in VNET2 and add it to VM1. | ? |
| 2. Turn off VM1, then add a new network interface in VNET2 to VM1. | ? |
| 3. Delete VM1, then recreate it with a new network interface in VNET2. | ? |

<details>
<summary>Answer</summary>

1. **No.** All NICs on a VM must be in the same VNet.
2. **No.** Turning the VM off doesn't change the same-VNet rule.
3. **Yes.** A VM cannot change VNet, so you recreate it.

</details>

### Exam-style question 2

RG1 contains a virtual network named VNET1. The "Not allowed resource types" policy is assigned to RG1, listing `Microsoft.Network/virtualNetworks` and `Microsoft.Compute/virtualMachines`. You need to create VM2 in RG1 and connect it to VNET1. What should you do first?

- A. Remove `Microsoft.Network/virtualNetworks` from the policy
- B. Create an Azure Resource Manager template
- C. Remove `Microsoft.Compute/virtualMachines` from the policy
- D. Add a subnet to VNET1

<details>
<summary>Answer</summary>

**C.** The policy blocks new virtual machines, so VM2 cannot be created until you remove that entry. VNET1 already exists, so the policy only flags it as non-compliant. It doesn't stop VM2 from connecting to it.

</details>
