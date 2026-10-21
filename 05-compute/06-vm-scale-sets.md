# 06 - Virtual Machine Scale Sets (VMSS)

Part of: Deploy and manage Azure compute resources (20-25% of AZ-104)
Labs: L12 (build a scale set), L13 (autoscale and upgrades)

[← Back to README](../README.md)

## In one line
A **VM Scale Set** creates and manages **many identical VMs from one model**, and can add or remove them automatically (**autoscale**) based on load or a schedule.

## When to use it (and when not)
| Need | Use |
|---|---|
| Web tier that grows and shrinks with load | **VMSS + autoscale + load balancer** |
| Many identical VMs updated together | **VMSS** with an **upgrade policy** |
| Replace unhealthy VMs automatically | **VMSS automatic repairs** (needs health probe/extension) |
| A few different, hand-built servers | Individual **VMs** |
| Web app without managing VMs | **App Service** |
| Containers that scale | **Container Apps / AKS** |

---

## 1. The big idea: one recipe, many identical VMs

I need a scale set when I want many VMs that are all the same, and I want Azure to create, spread, heal and scale them for me.

**Ex:** A web shop needs 3 web servers on a normal day and 10 during Black Friday. Without a scale set, I create and configure each VM by hand, add each one to the load balancer, place each one in a zone, and watch the metrics myself. With a scale set, I describe one VM once (the **model**), set a count, and Azure does the rest.

Think of it like a furniture factory: the **model** is the drawing, each **instance** is a piece built from that drawing. If I change the drawing, the pieces already in the showroom do not change by themselves. Only new pieces follow the new drawing.

What the scale set does for me:
- Builds every instance from the same model
- Spreads instances across zones or fault domains automatically
- Adds new instances to the load balancer backend pool automatically
- Scales out and in (manually, or with autoscale)
- Replaces broken instances (automatic repairs)
- Rolls out changes in safe batches (rolling upgrades)

The scale set itself is **free**. I pay for the VMs, disks and networking it creates. Max **1,000 instances** per scale set.

---

## 2. Orchestration modes: Flexible vs. Uniform

I need to pick the mode when I create the scale set, because **I can never change it later**.

| | Uniform (classic) | Flexible (default) |
|---|---|---|
| What the instances are | Scale set instances with IDs 0, 1, 2 ... | Normal VM resources |
| How I manage them | Scale set APIs only | Normal VM portal and APIs, plus scale set APIs |
| Sizes | One size | Mixed sizes possible |
| Overprovisioning | Yes, on by default | No |
| Spot priority mix | No | Yes |
| Max instances | 1,000 (100 with single placement group) | 1,000 |

**When I choose Flexible:** almost always for anything new. The instances look and behave like normal VMs, so I can use everything I learned in the VM decks.

**When I see Uniform:** older setups and older exam questions. The words "latest model: Yes/No", "overprovisioning" and "placement group" point to Uniform.

**One Flexible gotcha:** Flexible instances have no default outbound internet access. I need a load balancer outbound rule or a NAT gateway if they must reach the internet (ex: `apt-get install`).

---

## 3. The model and "latest model"

The model holds: image, size, extensions, custom data (cloud-init), networking.

The key rule to understand:

> When I change the model, the instances that already exist keep the old version. New instances always get the current model.

**Ex:** I have 3 instances on model v1. I change the size, so the model is now v2. Instance 1, 2 and 3 still run the old size and show **Latest model: No**. If autoscale adds instance 4, it is built from v2 and shows **Latest model: Yes**.

How the old instances catch up depends on the **upgrade policy** (section 6).

---

## 4. Where the instances come from (images)

- **Marketplace image:** quick to start, but every instance has to install its software at boot. Slower scale-out.
- **Azure Compute Gallery image:** a "golden image" with the software already inside. Fast, identical scale-out, shareable across subscriptions and regions.

Gallery words I need to know:
- **Image definition:** the description (OS, Gen 2, generalised or specialised)
- **Image version:** `major.minor.patch`, ex `1.2.0`, replicated to the regions I choose
- **Latest:** if I point at the definition, I get the newest version
- **Exclude from latest:** keeps a version out of "latest" (ex: a test build)
- **End of life date:** still deployable, just flagged

---

## 5. Configuring the instances

Three ways to get software onto every instance:

| Method | When it runs | I use it when... |
|---|---|---|
| cloud-init (custom data) | First boot, Linux | I want a simple base setup at creation (lab L12 web page) |
| Extensions (Custom Script, health, Entra login, AMA) | After provisioning, Linux and Windows | I want configuration after creation |
| Baked gallery image | Before deployment | I want the fastest, most identical scale-out |

**Exam phrase:** "install NGINX on every instance of the scale set from a template" = **Custom Script Extension** (older questions also accept the DSC extension).

All three live in the model, so changing any of them is a model change.

---

## 6. Upgrade policy: how old instances catch up

This is the setting that decides what happens after a model change.

| Policy | What happens to existing instances | Needs |
|---|---|---|
| **Manual** (default) | Nothing until I upgrade them myself | Nothing |
| **Automatic** | All upgraded at the same time, in any order (downtime) | Nothing |
| **Rolling** | Upgraded in batches (default 20%); next batch only when the last is healthy | Health probe or health extension |

**Ex (Manual):** An admin changes the VM size in the model. How many VMs change at once? **0.** The model changes; the instances wait for me to run `az vmss update-instances`.

**Ex (Rolling):** 10 instances, default 20% batch = 2 at a time, 5 batches. Each batch must report healthy before the next one starts.

Rolling upgrade guard rails (all default 20%): max batch size, max unhealthy instances before it starts, max unhealthy upgraded instances before it stops. **MaxSurge** creates new instances first, so I lose no capacity during the rollout.

### Automatic OS image upgrade (a different thing)

Do not mix this up with the "Automatic" upgrade policy.
- Upgrade policy = what happens when **I** change the model
- Automatic OS image upgrade = what happens when the **publisher** releases a new OS image

The OS upgrade always rolls in batches of **20%, minimum 1 instance**, and needs a health signal.

**Ex:** 4 instances. 20% of 4 = 0.8, rounded up to the minimum of 1. So **1 VM at a time**.

---

## 7. Health and automatic repairs

The scale set needs to know whether an instance is healthy. Two ways:

1. **Load balancer health probe** - the outside view (TCP port or HTTP path). Needs the instances in an LB pool. Also stops traffic to a failing instance.
2. **Application Health extension** - the inside view (checks a localhost endpoint). No load balancer needed. Gives richer states: Healthy, Unhealthy, Initializing, Unknown.

I pick one, not both.

Health is used by: rolling upgrades, automatic OS upgrades and automatic repairs.

**Automatic repairs**, step by step:
1. An instance reports unhealthy
2. Azure waits the **grace period** (10 minutes default and minimum, max 90)
3. Azure repairs it: **Replace** (default), Restart or Reimage
4. The new instance is built from the current model

At most 5% of the set is repaired at once. Protected instances are never repaired.

**Ex (lab L12):** I stop the web server on one instance. The health extension reports it unhealthy, and after 10 minutes it is replaced.

---

## 8. Autoscale

I can scale **manually** (set a count) or with **autoscale** (an Azure Monitor setting, one per scale set). While autoscale is on, my manual changes get overwritten.

### Profiles: which limits apply right now

| Profile | Ex | Limits |
|---|---|---|
| Default | Always there | min 1, max 3 |
| Recurrence | Mon-Fri 08:00-18:00 | min 2, max 6 |
| Fixed date | A sales weekend | min 4, max 10 |

Precedence: **fixed date > recurrence > default**. Each profile has its own min, max, default count and rules.

### Rules: when to scale

A rule has: metric, statistic, **duration**, operator, threshold, action, **cooldown**.

**Ex:** Percentage CPU, average, over 10 minutes, greater than 70% → increase by 1, then cool down 5 minutes.

How rules combine (the exam loves this):
- **Scale out if ANY** scale-out rule is met
- **Scale in only if ALL** scale-in rules are met
- Every scale-out rule should have a matching scale-in rule, with a gap between the thresholds (ex: out at 70%, in at 30%) so it does not flap up and down

### How I solve an autoscale question

I check in this order:
1. **Duration:** did the load last as long as the rule's duration? If not, nothing happens.
2. **Threshold:** is it above the out-threshold, below the in-threshold, or in between (nothing)?
3. **Limits:** clamp to min and max.
4. **Cooldown:** no new action during the cooldown.

**Worked example (Scale1):** start 4, min 2, max 20, out CPU > 80% for 10 min +2, in CPU < 30% for 10 min -4.

| Load | Reasoning | Instances |
|---|---|---|
| 85% for 6 min | 6 < 10 min duration, no action | 4 |
| 25% for 6 min, then 50% for 6 min | Too short to scale in, then between thresholds | 4 |
| 85% for 12 min | Above 80% for 10 min, +2 | 6 |
| 20% for 15 min (starting at 6) | Below 30% for 10 min, -4 | 2 |

---

## 9. Scale-in: who gets removed

When the count goes down, the **scale-in policy** decides which instance goes. Zone balance always comes first.

| Policy | Removes |
|---|---|
| Default | Balances zones, then fault domains, then the highest instance ID |
| NewestVM | The newest instance (after balancing zones) |
| OldestVM | The oldest instance (after balancing zones) |

**Exam phrase:** "remove the oldest VMs first" = **OldestVM**.

### Instance protection

I need instance protection when one instance must not be removed (ex: it is running a long job).

| Protection | Scale-in | Model upgrades | Automatic repair |
|---|---|---|---|
| None | Can be removed | Applied | Repaired |
| Protect from scale-in | Kept | Applied | Skipped |
| Protect from scale set actions | Kept | Skipped | Skipped |

Protection does **not** stop me from deleting the instance directly.

---

## 10. Zones, Spot and Uniform extras

**Zones:** chosen at creation. Instances are spread so zones differ by at most 1 (ex: 7 instances = 3, 2, 2). Scale-in takes from the fullest zone first.
- Best effort (default): allows imbalance if a zone has no capacity
- Strict (`zoneBalance: true`): scale-out fails rather than become unbalanced

**Spot priority mix (Flexible only):** a base of regular VMs, then a percentage of regular VMs above the base, and the rest are cheap Spot VMs.

**Ex:** 10 instances, base 2, 25% regular above base. Base = 2 regular. Above base = 8 x 25% = 2 regular. The rest = 6 Spot.

**Overprovisioning (Uniform only):** Azure creates extra instances and deletes the surplus, so scale-out is faster. The extras are **not billed** and not counted against quota.

**Placement groups (Uniform only):** single placement group = max 100 instances; turn it off for up to 1,000. I can change true to false, never back.

---

## 11. Match the requirement

| When I need... | I use |
|---|---|
| Many identical VMs that grow and shrink with load | Scale set + autoscale |
| To survive a datacentre failure | Scale set across zones 1-3 |
| To update all instances with no downtime | Rolling upgrade policy + health signal |
| Broken instances replaced automatically | Health extension or probe + automatic repairs |
| NGINX on every instance from a template | Custom Script Extension |
| The lowest cost for work that can be interrupted | Spot, or Spot priority mix (Flexible) |
| Mixed sizes and normal VM APIs | Flexible orchestration |

---

## 12. Key exam facts

- **Flexible** is the default mode; the mode is fixed at creation
- **Manual** is the default upgrade policy; a model change touches **0** instances
- **20%** is the default rolling batch and the automatic OS upgrade batch (minimum **1**)
- Rolling upgrades and automatic repairs **need a health signal** (probe or extension)
- Repair grace period: **10 min** default, 10-90
- Autoscale: scale out on **any** rule, scale in only on **all** rules
- Load shorter than the rule's **duration** changes nothing
- Profile precedence: fixed date > recurrence > default
- Uniform: **100** instances with single placement group, **1,000** without; Flexible: 1,000
- Overprovisioned surplus is **not billed** (Uniform only)
- "Remove the oldest first" = **OldestVM** scale-in policy

---

## 13. Azure CLI practice

```bash
RG=cat-cmpt-lab-12-rg
VMSS=cat-cmpt-lab-12-vmss

# Create a Flexible scale set across zones 1 and 3, behind a load balancer
az vmss create -g $RG -n $VMSS -l norwayeast \
  --orchestration-mode Flexible --zones 1 3 \
  --platform-fault-domain-count 1 --instance-count 3 \
  --vm-sku Standard_FX2ms_v2 --image Ubuntu2404 \
  --admin-username azureuser --generate-ssh-keys \
  --subnet $SUBNET_ID --custom-data cloud-init.yaml \
  --lb $VMSS-lb --backend-pool-name web-pool \
  --upgrade-policy-mode Manual

# See the instances
az vmss list-instances -g $RG -n $VMSS -o table

# Manual scale
az vmss scale -g $RG -n $VMSS --new-capacity 4

# Health extension, then push it to existing instances
az vmss extension set -g $RG --vmss-name $VMSS \
  -n ApplicationHealthLinux --publisher Microsoft.ManagedServices \
  --version 2.0 \
  --settings '{"protocol":"http","port":80,"requestPath":"/health"}'
az vmss update-instances -g $RG -n $VMSS --instance-ids '*'

# Automatic repairs
az vmss update -g $RG -n $VMSS \
  --enable-automatic-repairs true \
  --automatic-repairs-grace-period 10

# Autoscale: min 1, max 3, +1 when CPU > 70% for 10 min
az monitor autoscale create -g $RG -n autoscale \
  --resource $VMSS \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --min-count 1 --max-count 3 --count 1
az monitor autoscale rule create -g $RG --autoscale-name autoscale \
  --condition "Percentage CPU > 70 avg 10m" --scale out 1 --cooldown 5
az monitor autoscale rule create -g $RG --autoscale-name autoscale \
  --condition "Percentage CPU < 30 avg 10m" --scale in 1 --cooldown 5

# Scale-in policy and switching to Rolling
az vmss update -g $RG -n $VMSS --scale-in-policy OldestVM
az vmss update -g $RG -n $VMSS --set upgradePolicy.mode=Rolling
```

---

## PowerShell equivalents

```powershell
$rg = "cat-cmpt-lab-12-rg"; $name = "cat-cmpt-lab-12-vmss"

# Simple parameter set: creates VNet, LB and public IP if missing
New-AzVmss -ResourceGroupName $rg -VMScaleSetName $name -Location norwayeast `
  -ImageName Ubuntu2404 -VmSize Standard_D2s_v5 -InstanceCount 2 -Credential (Get-Credential) `
  -VirtualNetworkName vmss-vnet -SubnetName web -LoadBalancerName "$name-lb" -UpgradePolicyMode Manual

Get-AzVmssVM -ResourceGroupName $rg -VMScaleSetName $name

# Manual scale
$vmss = Get-AzVmss -ResourceGroupName $rg -VMScaleSetName $name
$vmss.Sku.Capacity = 4
Update-AzVmss -ResourceGroupName $rg -VMScaleSetName $name -VirtualMachineScaleSet $vmss

# Bring instances to the latest model (Manual upgrade policy)
Update-AzVmssInstance -ResourceGroupName $rg -VMScaleSetName $name -InstanceId "0","1"
```
Autoscale is easiest with the portal or `az monitor autoscale` (CLI above).

---

## Troubleshooting (symptom → cause → fix)

| Symptom | Likely cause | Fix |
|---|---|---|
| Changed the model but instances didn't change | **Manual** upgrade policy | `az vmss update-instances --instance-ids '*'` or Rolling/Automatic |
| Rolling upgrade or automatic repairs won't enable | No **health signal** | Add **Application Health extension** or LB health probe |
| Autoscale didn't scale out | Load shorter than the rule's **duration**, already at **max**, still in **cooldown**, or wrong metric | Check autoscale **Run history** (Monitor → Autoscale) |
| Autoscale flaps out and in | Scale-in threshold too close to scale-out | Leave a gap (e.g. out > 70%, in < 30%) |
| Scale out fails | **vCPU quota** reached | Request quota increase |
| Wrong VM removed during scale-in | Default scale-in policy | Set **OldestVM/NewestVM**, or **instance protection** |
| Unhealthy instance not replaced | Automatic repairs off, or still in **grace period** | Enable repairs, wait the grace period |
| Can't switch Uniform ↔ Flexible | Orchestration mode is fixed at creation | Create a new scale set |

---

## Break it and fix it

| # | Break it | What you should see | Fix |
|---|---|---|---|
| 1 | Change the VM size in the model (Manual policy) | Instances show **latest model: No** | `update-instances` |
| 2 | Stress CPU on all instances for 5 min with a 10-min rule (`stress-ng` via Run Command) | No scale out | Run longer than the duration |
| 3 | Set max count = current count, then load it | Stays at max | Raise max |
| 4 | Stop the web server on one instance with automatic repairs on | After the grace period the instance is **replaced** | – (watch it in Activity log) |

## Clean up
```bash
az group delete -n $RG --yes --no-wait
```

---

## 14. Self-test

**1. I change the VM size in the model of a scale set with upgrade policy Manual. How many instances change size right away?**
<details><summary>Answer</summary>

0. Under Manual, the model changes but existing instances wait until I upgrade them (`az vmss update-instances`). They show "latest model: No".
</details>

**2. A scale set has 4 instances and automatic OS image upgrade turned on. A new Windows Server image is released. How many VMs are upgraded at the same time?**
<details><summary>Answer</summary>

1. Automatic OS upgrades go in batches of 20%, minimum 1. 20% of 4 = 0.8, so 1 at a time.
</details>

**3. Scale1: start 4, min 2, max 20, out CPU > 80% for 10 min (+2), in CPU < 30% for 10 min (-4). CPU runs at 85% for six minutes. How many instances?**
<details><summary>Answer</summary>

4. Six minutes is shorter than the 10-minute duration, so the scale-out rule never triggers.
</details>

**4. Same Scale1. CPU runs at 25% for six minutes, then 50% for six minutes. How many instances?**
<details><summary>Answer</summary>

4. 25% for six minutes is too short to scale in. 50% is between the thresholds, so nothing happens.
</details>

**5. I have two scale-in rules. One is met, the other is not. Does the scale set scale in?**
<details><summary>Answer</summary>

No. Scale in needs ALL scale-in rules met. Scale out only needs ANY one.
</details>

**6. I want to switch my existing Uniform scale set to Flexible. How?**
<details><summary>Answer</summary>

I can't. The orchestration mode is fixed at creation. I create a new Flexible scale set and move the workload.
</details>

**7. I select the Rolling upgrade policy but the option is greyed out. What is missing?**
<details><summary>Answer</summary>

A health signal: a load balancer health probe or the Application Health extension.
</details>

**8. One instance runs a long batch job and must not be removed when autoscale scales in. What do I set?**
<details><summary>Answer</summary>

Instance protection: "protect from scale-in". If I also want it skipped by model upgrades and repairs, "protect from scale set actions".
</details>

**9. 10 instances, base regular count 2, 25% regular above base. How many Spot VMs?**
<details><summary>Answer</summary>

6. Base = 2 regular. Above base = 8 x 25% = 2 regular. 10 - 4 = 6 Spot.
</details>

**10. Overprovisioning creates extra instances during scale-out. Do I pay for them?**
<details><summary>Answer</summary>

No. The surplus instances are not billed and do not count against quota. Overprovisioning exists only in Uniform mode.
</details>
