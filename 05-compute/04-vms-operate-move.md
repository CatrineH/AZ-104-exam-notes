# Virtual machines: operate and move

AZ-104 domain: Deploy and manage Azure compute resources

Date learned: 2026-10-19

[← Back to README](../README.md)

## In one line
Operating a VM means **getting commands onto it** (VM agent, extensions, Run Command, cloud-init), understanding **power states and cost**, and knowing that moving between RGs/subscriptions **changes labels** while moving regions **needs a copy**.

## When to use it (and when not)
| Need | Use |
|---|---|
| Install software / run a script after deployment | **Custom Script Extension** |
| Configure a Linux VM at first boot | **cloud-init** (`--custom-data`) |
| Run a command now, without RDP/SSH or a public IP | **Run Command** |
| Stop paying for compute | **Deallocate** (not shut down inside the OS) |
| VM has a host problem / can't be reached | **Redeploy** (moves to a new host – temp disk lost) |
| Move to another RG or subscription | **Move** (same region, new resource ID) |
| Move to another region | **Azure Resource Mover** / Site Recovery / redeploy |

---

What is a VM in Azure.

```
Azure datacentre
  Physical server (the "host")
     +-- My VM's CPU and memory (borrowed from this host)
     +-- Temporary disk (a real disk INSIDE this host)

  Storage system (separate from the host, kept safe and copied 3 times)
     +-- OS disk (C: or /)
     +-- Data disks
```

My VM **borrows** CPU, memory and a small local disk from one physical server (the host). My OS disk and data disks are **not** on that server. They live in Azure's storage system, reached over the network.

 This is the same setup as a Hyper-V or VMware cluster:

- The **host** is the physical server that runs the VM right now. The VM can be moved to another host.
- The **temporary disk** is like a disk physically inside that host. It only exists on that one server.
- The **OS and data disks** are like disks on a SAN (shared storage). Any host can reach them, so the VM keeps them wherever it runs.

Picture

| Question | Answer from the picture |
|---|---|
| Why is the temporary disk lost on redeploy? | Redeploy moves the VM to a new host. The local disk stays behind in the old host. |
| Why is it lost on deallocate? | Deallocate releases the host. When the VM starts again, it may get a different host, with an empty local disk. |
| Why is it lost on resize? | A new size may need a different host. |
| Why does the OS disk always survive? | It is on shared storage, not on the host. The new host simply connects to it. |
| Why do you still pay when you shut down from inside Windows? | The VM still has the host's CPU and memory reserved. Azure can't give them to anyone else. |


---

## Part 1: Getting software and commands onto a VM

### The situation

I have just created 10 Linux VMs. Each one needs NGINX installed. Later, one VM stops answering, and I need to check what is wrong, but SSH doesn't work.

I don't want to log in to each VM by hand. How do I get software and commands onto them?

### The idea

Azure has three ways to reach inside a VM. The difference between them is **when** they run.

| Tool | When it runs | On-premises equivalent |
|---|---|---|
| **cloud-init** | Once, at the very **first boot** (Linux) | A server built from an automated install script |
| **Custom Script Extension** | After the VM exists, when you **set or change** it | A configuration script pushed out by a management tool |
| **Run Command** | **Any time** you ask | Running a single command on a server from a remote console |

### The VM agent: the messenger

Custom Script Extension and Run Command both need a small program inside the VM called the **VM agent** (waagent on Linux, Guest Agent on Windows).

The agent is the messenger between Azure and the VM:

```
I or a person  ->  Azure (control plane)  ->  VM agent inside the VM  ->  runs the script
```

This explains two rules:

1. **The VM must be running.** A stopped VM has no running agent to pass the message to.
2. **Run Command works even if the network is broken.** My command doesn't travel over SSH or RDP. It goes through Azure's own channel to the agent.

### Custom Script Extension: why it only runs once

Azure treats an extension as a description of how the VM **should be**: "this VM should have run this script with these settings."

When I send settings, Azure compares them with what it already has:

- **Same settings as before** - "Already done." Nothing happens.
- **Different settings** - "Something changed." The script runs again.

So to force a re-run, I **change something**: the script, a parameter, or a **timestamp** value that exists only for this purpose.

Other facts:

- One Custom Script Extension per VM, maximum 90 minutes per run
- Put passwords and secrets in **protected settings**, which are encrypted
- Removing the extension does **not** undo what the script did. The software stays installed.
- On a **scale set**, it runs on every VM. That makes it the answer to *"install NGINX on every scale set VM from a template"*.

### cloud-init: setup at first boot

cloud-init is a YAML file you pass as **custom data** when I create a Linux VM. On the very first boot, Linux reads it and installs packages, writes files and runs commands.

```yaml
#cloud-config
package_update: true
packages:
  - nginx
runcmd:
  - systemctl enable --now nginx
```

It runs **only at first boot**, so it is for the **starting setup**, not for changes later.

### Choosing between the three

Ask myself one question: **When do I need this to happen?**

| I need... | Use |
|---|---|
| The VM is ready the moment it is created | **cloud-init** |
| Configure a VM that already exists, in a repeatable way | **Custom Script Extension** |
| Check or fix something right now, one time | **Run Command** |

### Old names I might see

Microsoft has replaced some older tools. Old exam questions may still use the old names.

| Old (legacy) | Replaced by |
|---|---|
| DSC extension | Machine Configuration (through Azure Policy) |
| Log Analytics agent (MMA) | Azure Monitor Agent |

If an answer option uses the old name, that is fine. Choose the option that matches the job.

### When the VM won't even start properly

Sometimes the problem is so bad that nothing works: no network, no SSH, no RDP, no agent.

| Tool | What it shows me | On-premises equivalent |
|---|---|---|
| **Boot diagnostics** | A **screenshot** and log of the screen while the VM starts | Looking at the monitor plugged into the server |
| **Serial console** | A **text keyboard and screen** for the VM, in the portal | A console cable (or iLO/iDRAC) that works even when the network is down |

Serial console needs boot diagnostics turned on first.

### Check your understanding

**Q1.** You sent a Custom Script Extension to VM1 yesterday. Today you send exactly the same settings again, but nothing happens. Why, and how do you fix it?

<details>
<summary>Answer</summary>

Azure sees the same settings and decides the job is already done. Change something, such as a timestamp value, and it will run again.

</details>

**Q2.** The question says "you need to enable an extension on VM1". The exhibit shows VM1 is **Stopped**. What do you do first?

<details>
<summary>Answer</summary>

Start VM1. Extensions are run by the VM agent, and the agent only runs when the VM is running.

</details>

**Q3.** SSH to a VM fails because of a bad firewall rule inside Linux. Which tool lets you run a command anyway?

<details>
<summary>Answer</summary>

Run Command (or the serial console). Run Command goes through Azure's control plane to the VM agent, not over the network.

</details>

---

## Part 2: Power states, cost and what I lose

### The situation

My manager asks: "We shut down all the test VMs on Friday. Why is the bill still high?"

### The idea

Go back to the picture from Part 0. The question is always: **does the VM still have a host reserved?**

| State | How I get there | Host still reserved? | Pay for CPU and memory? | Temporary disk |
|---|---|---|---|---|
| Running | Start | Yes | Yes | Kept |
| **Stopped** | Shut down **from inside** Windows or Linux | **Yes** | **Yes** | Kept |
| **Stopped (deallocated)** | **Stop** in the portal, or `az vm deallocate` | **No** | **No** | **Lost** |

When I shut down from inside the OS, Azure doesn't know whether I plan to start again in a minute. So it **keeps the host reserved for you**, and keeps charging.

When I **deallocate**, I tell Azure: "Release the host." The charges for CPU and memory stop. Because the VM no longer has that host, the local temporary disk is gone.

**The answer to the manager:** the VMs were shut down from inside the OS, not deallocated.

### What you always pay for

Even when deallocated, you still pay for things that keep existing:

- **Managed disks** (they stay on shared storage)
- **Public IPs** (the address is reserved for me)
- Snapshots

### Restart, redeploy and reapply

These three sound similar, but they do very different things.

| Action | What happens | Same host? | Temporary disk | Use it when |
|---|---|---|---|---|
| **Restart** | The OS reboots | Yes | Kept | The OS is frozen |
| **Redeploy** | Azure moves the VM to a **new host** | **No** | **Lost** | I still can't connect after a restart. Maybe the host itself has a problem. |
| **Reapply** | Azure re-applies the VM's settings, without moving it | Yes | Kept | The VM shows provisioning state **Failed** |

Redeploy is the same as live-migrating a VM off a faulty host on-premises, except that the VM is shut down during the move.

### Worked example: what does a redeploy lose?

> User1 creates files on C:, creates files on D:, changes the screen saver, and changes the desktop background. The VM is redeployed. What is lost?

Work it out step by step:

1. **Where does each change live?**
   - Files on C: - on the OS disk
   - Files on D: - on the temporary disk (on Windows, D: is the temporary disk)
   - Screen saver and background - these are user settings, saved in the Windows registry, which is on the OS disk (C:)
2. **What does redeploy do?** It moves the VM to a new host.
3. **What stays with the old host?** Only the temporary disk.
4. **Answer:** only the files on D: are lost.

This is the most common VM question on the exam, and it comes straight from the picture in Part 0.

### Resizing

Resizing means choosing a bigger or smaller VM size.

- **The VM always restarts.**
- If the current host can run the new size, Azure resizes in place.
- If not, the VM needs a **different host**, so you must **deallocate, resize, then start**.
- The temporary disk may be lost (a new host means a new, empty local disk).

Before resizing, check:

- **Disks:** if you use Premium SSD, the new size must have an **"s"** in its name
- **Quota:** you have enough vCPUs left
- **Zone:** the size exists in the VM's zone

### Two ways to save money

| Option | How it saves money | The catch |
|---|---|---|
| **Spot VM** | Uses Azure's spare capacity, up to around **90% cheaper** | Azure can take the VM back (**evict** it) with only **30 seconds' notice**. No SLA. |
| **Auto-shutdown** | **Deallocates** the VM at a set time every day | It only stops the VM. It won't start it again in the morning. |

A Spot VM is like a standby airline ticket: very cheap, but you can be removed if a full-price passenger needs the seat. Use it only for work that can stop and start again, such as batch jobs, tests and rendering.

### Check your understanding

**Q4.** Which is cheaper: a VM that is "Stopped", or one that is "Stopped (deallocated)"? Why?

<details>
<summary>Answer</summary>

Deallocated. A "Stopped" VM still holds its host, so you pay for CPU and memory. A deallocated VM has given the host back.

</details>

**Q5.** A VM is deallocated for the weekend. On Monday, is the data on the temporary disk still there?

<details>
<summary>Answer</summary>

No. Deallocating gives the host back. When the VM starts again it may be on a different host, so the temporary disk starts empty.

</details>

**Q6.** A VM's status shows "Provisioning state: Failed". It is not frozen and you can still reach it. Which action fits best?

<details>
<summary>Answer</summary>

Reapply. It re-applies the VM's settings without moving it.

</details>

---

## Part 3: Moving a VM

### The situation

Three different requests arrive in one week:

1. "Move the web VM from the **test** resource group to the **production** resource group."
2. "Move it to the **other subscription**, because the finance team will pay for it."
3. "Move it from **Norway East** to **Sweden Central**."

They all say "move", but in Azure they work completely differently.

### The idea: labels vs physical location

- A **resource group** is a **folder**: a way to organise and manage resources.
- A **subscription** is a **billing account**: who pays.
- A **region** is a **physical place**: a real datacentre building with real hardware.

Moving to another folder or billing account is just **changing a label**. Nothing physical moves.

Moving to another region means the data must **physically travel** to another building. That is a different job.

Think of a furniture warehouse: you can move a sofa from the "Living room" list to the "Sale" list on the computer in seconds, but moving the sofa to another city needs a truck.

| Move | What really happens | Downtime? | What changes |
|---|---|---|---|
| Another **resource group** | A label change | No | Only the resource IDs |
| Another **subscription** | A label change (who pays) | No | Only the resource IDs |
| Another **region** | A **copy** is made in the new region, then you delete the old one | Yes, at the switch-over | Everything is new: IDs, IP addresses, NIC |

Because a resource group move only changes labels, the **region, IP addresses and data all stay the same**. The only thing that changes is the **resource ID**, because the resource group name is part of the ID:

```
/subscriptions/<sub>/resourceGroups/test-rg/providers/Microsoft.Compute/virtualMachines/web01
                                    ^^^^^^^
                                    this part changes
```

Any script that uses the old ID must be updated.

### Moving to another resource group or subscription

#### Things that move together

Some resources depend on each other so closely that they must move **as a group**:

- A **VM** moves with its **disks and NICs** (to another subscription, they must all move in one operation)
- A **VNet** moves with **everything connected to it**
- An **availability set** moves with **all its VMs**

#### What I need

- **Permission on both sides:** for example **Contributor** on the source resource group and on the target resource group. I am taking something out of one folder and putting it into another.
- **For a subscription move, the extra rules:**
  - Both subscriptions must be in the **same Entra tenant**. A tenant is my organisation's identity boundary, and resources can't simply cross into another organisation.
  - The target subscription needs the **resource providers registered** (for example Microsoft.Compute) and enough **quota**.

During the move, both resource groups are **locked against changes** for up to 4 hours.

#### What blocks a move

| Blocker | Why | Fix |
|---|---|---|
| A **VPN or ExpressRoute gateway** in the VNet | Azure doesn't support moving it | Recreate it in the target |
| **VNet peering** | The VNet is connected to another VNet | Remove the peering first |
| A **read-only lock** on either resource group | A move is a change, and read-only blocks changes | Remove the lock |
| A vault with active **Site Recovery** replication | It is busy copying data | Stop the replication |
| Different **tenant** | Different organisation | Transfer the subscription to my tenant first |

Always **validate** before moving. The portal does this for me and tells me what would fail.

### Moving to another region: I copy instead

Because a region is a physical place, there is no "move" button that simply relabels. I **copy** the VM to the new region, then delete the original myself.

| Method | Best for | How it works |
|---|---|---|
| **Azure Resource Mover** | Moving a VM **with everything it needs** | A free, guided tool. It finds the dependencies (VNet, NSG, NIC, disks) and copies them together. |
| **Snapshot copy** | One VM, done by hand | Snapshot the disk, copy it to the new region, create a disk, then create a VM from it |
| **Gallery image** | Many identical VMs | Make an image, copy it to other regions, create VMs from it |
| **Site Recovery** | Disaster recovery | Keeps copying all the time, so you can fail over |

Resource Mover works in steps:

1. **Select** the resources and the target region
2. **Resolve** dependencies (it adds the VNet, NIC and so on)
3. **Prepare** (copy the disks)
4. **Initiate move** (create the new VMs)
5. **Commit** (or discard if something is wrong)
6. **Delete the source** - you do this yourself. Resource Mover never deletes the original.

### Worked example: which resources can move?

> Subscription AZPT1 has: VM1, its managed disk VM1Managed, storage1, VNET1 and a Recovery Services vault RVAULT1. You create AZPT2 in the **same tenant**. Which resources can move to AZPT2?

Go through the blockers list for each resource:

| Resource | Any blocker? | Can it move? |
|---|---|---|
| VM1 | No. It must move together with its disk. | Yes |
| VM1Managed | No. It moves with VM1. | Yes |
| storage1 | No | Yes |
| VNET1 | The question mentions no gateway and no peering | Yes |
| RVAULT1 | The question mentions no active Site Recovery replication | Yes |

**Answer:** all five. Same tenant, no blockers.

### Check your understanding

**Q7.** You move a VM from resource group RG-Test to RG-Prod. Does its private IP address change? Does its region change?

<details>
<summary>Answer</summary>

Neither changes. A resource group move only changes the label (and so the resource ID). Nothing physical moves.

</details>

**Q8.** Someone says "just move the resource group to West Europe". Why doesn't that work?

<details>
<summary>Answer</summary>

A resource group is a folder, not a place. Moving it doesn't move the data. To change region you must copy the resources, for example with Azure Resource Mover.

</details>

**Q9.** A VNet can't be moved, and validation says it is connected to another VNet. What do you do?

<details>
<summary>Answer</summary>

Remove the VNet peering, then move it.

</details>

---

## Part 4: Bringing VMs in from on-premises

### The situation

Your company runs servers on Hyper-V in its own server room. You need to bring one into Azure, and you need to know which Azure sizes to choose for the others.

### Uploading a disk: Azure is strict about the format

Azure stores VM disks in a fixed-size format. So a disk you upload must be:

| Azure needs | What Hyper-V often has | Fix |
|---|---|---|
| **VHD** format | VHDX (a newer format) | Convert it |
| **Fixed** size | **Dynamic** (grows as you add data) | Convert it to fixed |
| A whole number of MiB | Any size | Resize it |

Exam phrase: *"Use VM1's disks as a template for Azure VMs. What should you modify?"* Answer: **the hard drive**, changing it from a dynamic VHDX to a fixed VHD.

### Template or copy?

| Type | What it is | When to use it |
|---|---|---|
| **Generalised** | The unique details are removed (with sysprep on Windows) | As a **template** for many new VMs |
| **Specialised** | An exact copy, including the computer name and accounts | To recreate **one** specific machine |

Think of it as a blank order form (generalised) versus a photocopy of a completed one (specialised).

### Azure Migrate: measure before you move

Before moving many servers, you want to know: *How big should each Azure VM be? What will it cost? Is the server ready to move?*

**Azure Migrate** answers this:

1. Create an **Azure Migrate project** in the portal
2. Install a **discovery appliance** on-premises. Older exam questions call it the **collector VM**.
3. The appliance **discovers** your servers and measures their CPU, memory and disk use
4. Run an **assessment**: you get a recommended Azure size, disk type, monthly cost and readiness
5. **Migrate** the servers

Exam phrase: *"Identify the Azure sizes for Server2"*. Answer: **create an Azure Migrate project, then the collector VM.**

### Check your understanding

**Q10.** You try to upload a dynamically expanding VHDX disk to Azure. What two things must you change?

<details>
<summary>Answer</summary>

Convert the format from VHDX to VHD, and change it from dynamic to fixed size.

</details>

---

## Part 5: Locks and deleting resource groups

### The situation

A colleague accidentally deletes a production VNet. Your manager asks: "How do we stop that from ever happening again, even for admins?"

### The idea: a lock is a safety catch, not a permission

Normal access control (RBAC) answers: **Who is allowed to do this?**

A **lock** answers a different question: **Is this action allowed at all, right now?**

A lock is like a padlock on a cupboard. It doesn't matter whose key you have. Even the owner of the building can't open it until someone **removes the padlock**.

| Lock | Read | Change | Delete |
|---|---|---|---|
| **CanNotDelete** | Yes | Yes | **No** |
| **ReadOnly** | Yes | **No** | **No** |

Rules that follow from this:

- A lock blocks **everyone, even Owners**
- To remove a lock you need **Owner** or **User Access Administrator**. **Contributor can't** remove locks.
- Locks are **inherited downwards**: a lock on a resource group protects everything in it

**The answer to the manager:** put a **CanNotDelete** lock on the production VNet.

### ReadOnly surprises

ReadOnly blocks more than you might expect, because many normal actions count as "changes":

- You can't **start or stop** a VM
- You can't **list storage account keys**
- You can't **move** resources into or out of the group

### Why can't I delete this resource group?

| Blocker | Why | Fix |
|---|---|---|
| A **lock** on the group or a resource in it | The safety catch is on | Remove the lock |
| A **Recovery Services vault** still holding backups | Azure protects your backup data | Stop backup and delete the backup data first |
| A resource in **another** group uses something in this group | Deleting would break the other resource | Delete or move that resource first |

### Check your understanding

**Q11.** User1 is an **Owner** of the subscription. A VNet has a CanNotDelete lock. Can User1 delete the VNet?

<details>
<summary>Answer</summary>

Not straight away. The lock blocks everyone, including Owners. User1 must remove the lock first, which an Owner is allowed to do.

</details>

**Q12.** User2 is a **Contributor**. Can User2 remove the lock?

<details>
<summary>Answer</summary>

No. Contributor doesn't include permission to manage locks. You need Owner or User Access Administrator.

</details>

---

## Summary: the ideas that explain the facts

| Idea | Facts it explains |
|---|---|
| **The temporary disk lives on the host.** Leaving the host loses it. | Lost on redeploy, deallocate and resize. The OS disk always survives. |
| **You pay for CPU and memory while you hold the host.** | Shutting down inside the OS is still billed. Deallocate is not. |
| **Extensions are run by the VM agent.** | The VM must be running. Run Command works when the network is broken. |
| **An extension describes the desired state.** | The same settings don't run again. Change something to re-run. |
| **Resource groups and subscriptions are labels. Regions are places.** | Group and subscription moves only change the IDs. Region moves need a copy. |
| **A lock is a padlock, not a key.** | It blocks even Owners. Only Owner or User Access Administrator can remove it. |

---

## Hands-on practice

Each step says **what to expect**. Compare that with what I see.

```bash
RG=catrineh-vmops-rg
DST=catrineh-vmops-moved-rg
LOC=norwayeast
VM=catrineh-vm-02
SIZE=Standard_FX2mds_v2     # the "d" means it has a temporary disk
```

**Step 1. Create a VM that installs NGINX on first boot (cloud-init)**

```bash
cat > cloud-init.yaml <<'EOF'
#cloud-config
package_update: true
packages:
  - nginx
runcmd:
  - systemctl enable --now nginx
EOF

az group create -n $RG -l $LOC
az vm create -g $RG -n $VM -l $LOC \
  --image Ubuntu2404 --size $SIZE \
  --admin-username azureuser --generate-ssh-keys \
  --public-ip-address "" \
  --custom-data cloud-init.yaml \
  --os-disk-delete-option Delete --nic-delete-option Delete
```

Expect: the VM is created with no public IP. NGINX installs by itself during the first boot.

**Step 2. Check NGINX with Run Command**

```bash
az vm run-command invoke -g $RG -n $VM \
  --command-id RunShellScript --scripts "curl -s http://localhost | head -5"
```

Expect: the start of the NGINX welcome page. This works without SSH or a public IP because it goes through the VM agent.

**Step 3. Test the picture from Part 0: write one file to each disk**

```bash
az vm run-command invoke -g $RG -n $VM --command-id RunShellScript \
  --scripts "echo temp | sudo tee /mnt/test.txt; echo os > /home/azureuser/test.txt"
```

`/mnt` is the temporary disk (local to the host). `/home` is on the OS disk (on shared storage).

**Step 4. Redeploy, then check which file survived**

Before you run this, predict which file will be gone.

```bash
az vm redeploy -g $RG -n $VM
az vm run-command invoke -g $RG -n $VM --command-id RunShellScript \
  --scripts "ls -l /mnt/test.txt /home/azureuser/test.txt"
```

Expect: `/mnt/test.txt` is missing and `/home/azureuser/test.txt` is still there.

**Step 5. Compare "stopped" and "deallocated"**

```bash
az vm stop -g $RG -n $VM
az vm show -d -g $RG -n $VM --query powerState     # "VM stopped" - still billed

az vm deallocate -g $RG -n $VM
az vm show -d -g $RG -n $VM --query powerState     # "VM deallocated" - not billed for compute
```

**Step 6. See a lock block a delete**

```bash
az lock create -n keep-rg --lock-type CanNotDelete -g $RG
az group delete -n $RG --yes      # Expect: an error saying the scope is locked
az lock delete -n keep-rg -g $RG
```

**Step 7. Move the VM, its disk and its NIC to another resource group**

```bash
az group create -n $DST -l $LOC
VM_ID=$(az vm show -g $RG -n $VM --query id -o tsv)
DISK_ID=$(az vm show -g $RG -n $VM --query storageProfile.osDisk.managedDisk.id -o tsv)
NIC_ID=$(az vm show -g $RG -n $VM --query "networkProfile.networkInterfaces[0].id" -o tsv)

az resource move --destination-group $DST --ids $VM_ID $DISK_ID $NIC_ID

# Check: same region, new resource ID
az vm show -g $DST -n $VM --query "{region:location, id:id}" -o json
```

Expect: the region is still `norwayeast`, and the ID now contains `catrineh-vmops-moved-rg`.

**Step 8. Clean up**

```bash
az group delete -n $DST --yes --no-wait
az group delete -n $RG --yes --no-wait
```

---

## PowerShell equivalents

```powershell
$rg = "catrineh-vmops-rg"; $vmName = "catrineh-vm-02"

# Custom Script Extension (Windows)
Set-AzVMCustomScriptExtension -ResourceGroupName $rg -VMName $vmName -Name InstallIIS -Location norwayeast `
  -FileUri "https://raw.githubusercontent.com/<you>/scripts/main/iis.ps1" -Run "iis.ps1"

# Run Command
Invoke-AzVMRunCommand -ResourceGroupName $rg -VMName $vmName -CommandId RunShellScript -ScriptString "uptime"

# Power states
Stop-AzVM    -ResourceGroupName $rg -Name $vmName -Force                    # deallocated (not billed for compute)
Stop-AzVM    -ResourceGroupName $rg -Name $vmName -StayProvisioned -Force   # stopped (still billed)
Start-AzVM   -ResourceGroupName $rg -Name $vmName
Restart-AzVM -ResourceGroupName $rg -Name $vmName
Set-AzVM     -ResourceGroupName $rg -Name $vmName -Redeploy
Get-AzVM     -ResourceGroupName $rg -Name $vmName -Status | Select-Object -ExpandProperty Statuses

# Resize
$vm = Get-AzVM -ResourceGroupName $rg -Name $vmName
Get-AzVMSize -ResourceGroupName $rg -VMName $vmName          # sizes available on the current hardware
$vm.HardwareProfile.VmSize = "Standard_D4s_v5"
Update-AzVM -ResourceGroupName $rg -VM $vm

# Move VM + disk + NIC to another RG
$ids = @($vm.Id, $vm.StorageProfile.OsDisk.ManagedDisk.Id, $vm.NetworkProfile.NetworkInterfaces[0].Id)
Move-AzResource -DestinationResourceGroupName catrineh-vmops-moved-rg -ResourceId $ids -Force
```

---

## Troubleshooting (symptom → cause → fix)

| Symptom | Likely cause | Fix |
|---|---|---|
| Extension "Provisioning failed" | Script error, file URL unreachable, or no outbound internet | Check extension status message, logs (`C:\WindowsAzure\Logs` / `/var/log/azure`) |
| Custom Script didn't run again | Same settings = same desired state | Change a setting (timestamp / `forceUpdateTag`) or remove and re-add |
| Run Command / extensions hang | **VM agent** not ready, or VM not running | Start VM, check agent status in the portal |
| Wanted size not in the resize list | Current hardware cluster doesn't have it | **Deallocate**, then all sizes appear |
| Data on D: / `/mnt` gone | **Temporary disk** – lost on redeploy/resize/deallocate | Keep data on OS/data disks |
| Still billed after shutdown | Shut down **inside the OS** = "Stopped" | **Deallocate** |
| VM unreachable, nothing else helps | Host issue | **Redeploy** (new host) |
| Move fails | Dependent resources not included, locks, unsupported type, provider not registered | Move VM + disks + NIC together, remove locks, register provider |
| Uploaded VHD won't boot | Wrong format: must be **fixed-size VHD**, not VHDX/dynamic | Convert (`Convert-VHD -VHDType Fixed`) |
| Can't delete RG | **Lock** on RG or a resource | Owner/UAA removes the lock |

---

## Break it and fix it

| # | Break it | What you should see | Fix |
|---|---|---|---|
| 1 | Run the **same** Custom Script Extension twice | Second time nothing happens | Change a setting to force a re-run |
| 2 | `az vm resize` to a size not in `list-vm-resize-options` while running | Error | Deallocate first, then resize |
| 3 | Move only the VM (not its disk/NIC) | Validation error about dependent resources | Move all together |
| 4 | Put a **ReadOnly** lock on the VM, then `az vm start` | ScopeLocked | Remove the lock |

---

## Exam-style questions

Try to work out each answer using the ideas, before opening it.

### Question 1

VM1 is a **Standard_D2ds_v5** VM running Windows Server. You sign in as User1 and create files on drive C, create files on drive D, change the screen saver timeout and change the desktop background. You then **redeploy** VM1. Which change is lost?

- A. The modified screen saver timeout
- B. The new desktop background
- C. The new files on drive D
- D. The new files on drive C

<details>
<summary>Answer and reasoning</summary>

**C.**

1. The "d" in D2**d**s_v5 tells you the VM has a temporary disk. On Windows, that is drive D.
2. Redeploy moves the VM to a new host.
3. The temporary disk lives on the old host, so its files are lost.
4. Drive C, the screen saver and the background are all on the OS disk, which travels with the VM.

</details>

### Question 2

Subscription AZPT1 contains VM1, VM1Managed (its managed disk), storage1, VNET1 and RVAULT1 (a Recovery Services vault). You create AZPT2 in the **same tenant**. Which resources can be moved to AZPT2?

- A. VM1, storage1, VNET1 and VM1Managed only
- B. VM1 and VM1Managed only
- C. VM1, storage1, VNET1, VM1Managed and RVAULT1
- D. RVAULT1 only

<details>
<summary>Answer and reasoning</summary>

**C.**

1. Same tenant, so the subscription move is allowed.
2. Check each resource against the blockers: no gateway, no peering, no lock, and no active Site Recovery replication are mentioned.
3. VM1 and VM1Managed must move together, and they can.
4. So all five can move.

</details>

### Question 3

TestRG contains VM1, which is backed up to Vault1, and VNET1, which has a lock. You need to delete TestRG. What must you deal with first?

<details>
<summary>Answer and reasoning</summary>

**The lock and the backup data.**

1. A lock blocks deletion, even for Owners. Remove it.
2. A Recovery Services vault can't be deleted while it holds backup data. Stop the backup and delete the backup data first.
3. Turning off VM1 is **not** needed. A running VM doesn't block a resource group delete.

The marked answer in some older questions is "turn off VM1 and remove the lock". If that is the only option that includes removing the lock, choose it. The real blockers are the lock and the backup data.

</details>

