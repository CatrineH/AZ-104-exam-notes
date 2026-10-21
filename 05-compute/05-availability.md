# Availability

AZ-104 domain: Deploy and manage Azure compute resources

Date learned: 2026-10-19

[← Back to README](../README.md)

## In one line
To survive failures, **spread VMs out**: an **availability set** spreads them across racks in one datacentre (99.95%), **availability zones** across buildings in a region (99.99%), and **region pairs** across regions – with a **load balancer** sending traffic only to healthy VMs.

## When to use it (and when not)
| Protect against | Use | SLA (2+ VMs) |
|---|---|---|
| Rack (power/network) failure, planned maintenance | **Availability set** | 99.95% |
| A whole datacentre/zone failing | **Availability zones** | 99.99% |
| A whole region failing | Second region (**region pair**) + Site Recovery / Traffic Manager / Front Door | – |
| One VM only | **Premium SSD/Ultra** on all disks | 99.9% |
| VMs must be very close (low latency) | **Proximity placement group** | – |
| Guarantee capacity for a size | **Capacity reservation** | – |

---

Availability is about one question: **what can break, and how much breaks with it?**

Picture a normal on-premises setup, from small to big:

```
Region (a city or area, for example Oslo)
  Datacentre building (own power, cooling and network)
    Rack (a row of servers sharing one power supply and one network switch)
      Host (one physical server)
        My VM
```

Each level can fail, and each failure takes down **everything below it**:

| What fails | What goes down | Example |
|---|---|---|
| A **host** | The VMs on that one server | A motherboard dies |
| A **rack** | Every host in that rack | The rack's power supply or top-of-rack switch fails |
| A **building** | Every rack in that datacentre | A fire, a flood, a total power cut |
| A **region** | Every datacentre in that area | A major natural disaster |

There is also a failure I **plan**: **maintenance**. Microsoft has to update the software on the hosts, and sometimes that needs a reboot.

Azure gives me one tool for each level:

| To survive... | Use | Covered in |
|---|---|---|
| A host failure | Nothing extra. Azure restarts the VM on another host automatically ("service healing"). | Part 1 |
| A rack failure or maintenance | **Availability set** | Part 2 |
| A building failure | **Availability zones** | Part 3 |
| A region failure | **A second region** (Site Recovery, region pairs) | Part 5 |

The rule to remember: **spread my VMs across the level I want to survive.** If both copies of my app sit in the same rack, the rack is a single point of failure, just like on-premises.

---

## Part 1: SLAs and maintenance

### The situation

My manager asks: "The business wants our web app to be up 99.99% of the time. What do we need to build?"

### The idea: the SLA follows the protection

An SLA (Service Level Agreement) is Microsoft's promise of uptime. The more failure levels I protect against, the higher the promise.

| How I deploy | SLA | Downtime allowed per year | Survives |
|---|---|---|---|
| Single VM, Standard HDD | 95% | about 18 days | Nothing extra |
| Single VM, Standard SSD | 99.5% | about 1.8 days | Nothing extra |
| Single VM, **Premium SSD or Ultra on all disks** | **99.9%** | about 8.8 hours | Nothing extra, but better disks |
| **2+ VMs in an availability set** | **99.95%** | about 4.4 hours | Rack failure, maintenance |
| **2+ VMs across 2+ availability zones** | **99.99%** | about 53 minutes | Building failure, too |

Notice the pattern: each extra level of spreading moves me up the table. I can work out the order from Part 0, instead of memorising it.

**The answer to the manager:** at least 2 VMs, spread across at least 2 availability zones.

Exam shortcut: when a question says **"99.99% for VMs"**, the answer is **availability zones**.

### One more thing: a composite SLA

My app depends on more than VMs, for example a database and a load balancer. The app's real uptime is all of their SLAs **multiplied** together, so it is always a bit lower than the weakest part.

### Planned maintenance: three kinds

Most maintenance happens without me noticing.

| Type | What happens to my VM | What can I do |
|---|---|---|
| **Live migration** | The VM moves to an updated host while running. It pauses for a few seconds. | Nothing |
| **Memory-preserving update** | The VM pauses for up to 30 seconds while the host updates. Memory is kept. | Nothing |
| **Reboot-required update** | The host must restart, so the VM restarts. | You get a **self-service window** to redeploy at a time that suits you |

This is the same as patching a Hyper-V cluster on-premises: My live-migrate VMs off a host when I can, and reboot only when I must.

### Controlling and seeing maintenance

| Tool | What it does |
|---|---|
| **Maintenance configurations** | I choose a schedule (time window, time zone, recurrence) for updates. Also used by Azure Update Manager. |
| **Scheduled events** | The VM itself can ask "is anything about to happen to me?" through the metadata endpoint `169.254.169.254`. You get up to **15 minutes' warning** (Spot VMs get 30 seconds). |
| **Service Health > Planned maintenance** | Where you see upcoming maintenance in the portal |

### Check my understanding

**Q1.** You have one VM with Premium SSD disks. What is its SLA, and what is the cheapest way to get a higher one?

<details>
<summary>Answer</summary>

99.9%. Add a second VM and put both in an availability set (99.95%). For 99.99%, spread them across availability zones instead.

</details>

**Q2.** A question asks for 99.99% uptime for VMs. Is an availability set enough?

<details>
<summary>Answer</summary>

No. An availability set gives 99.95%. You need 2 or more VMs across 2 or more availability zones.

</details>

---

## Part 2: Availability sets

### The situation

You run two web servers. One night, a rack loses power and **both** servers go down, because Azure happened to put them in the same rack. How do you make sure that never happens again?

### The idea: two ways to spread VMs inside one datacentre

An availability set tells Azure: "Never put all of these VMs in the same place." It spreads them in **two directions at once**:

| Spread across | What it is | Protects against |
|---|---|---|
| **Fault domains (FD)** | **Racks.** Each has its own power and network switch. | A **rack failure** (unplanned) |
| **Update domains (UD)** | **Groups of hosts that Microsoft updates at the same time.** Only one group is updated at a time. | **Maintenance** (planned) |

Think of it as a grid. Fault domains are the **columns** (racks). Update domains are the **rows** (maintenance waves).

```
             FD 0 (rack A)   FD 1 (rack B)
UD 0         VM1             VM2
UD 1         VM3             VM4
UD 2         VM5             VM6
```

- A **rack fails**: one **column** goes down
- A **maintenance wave** runs: one **row** goes down

Azure places VMs **round-robin** when you create them. VM1 goes to FD0/UD0, VM2 to FD1/UD1, VM3 to FD0/UD2, and so on, so they spread out evenly.

### The numbers

| Setting | Maximum | Portal default |
|---|---|---|
| Fault domains | **3** (2 in some regions) | **2** |
| Update domains | **20** | **5** |

- The counts are **fixed when you create the set**. To change them, create a new set.
- An availability set is **free**. I only pay for the VMs.
- Use **Aligned** (managed disks), so each VM's disks are spread the same way as the VM.

Exam phrase: *"50 VMs, as many available as possible during maintenance"* = set the **maximum** fault domains and **20** update domains. More domains means fewer VMs in each one.

### Working out how many VMs go down

This is the most common availability-set question on the exam. The method is always the same:

**VMs divided by the number of domains, rounded up.**

- For a **rack failure**, divide by **fault domains**
- For **maintenance**, divide by **update domains**

**Example 1: 14 VMs, 2 fault domains, 7 update domains**

| Event | Calculation | Most VMs down at once |
|---|---|---|
| Rack failure | 14 ÷ 2 = 7 | **7** |
| Maintenance | 14 ÷ 7 = 2 | **2** |

**Example 2: 10 VMs, 3 fault domains, 5 update domains**

| Event | Calculation | Most VMs down at once |
|---|---|---|
| Rack failure | 10 ÷ 3 = 3.33, rounded **up** to 4 | **4** |
| Maintenance | 10 ÷ 5 = 2 | **2** |

Why round **up**? With 10 VMs in 3 racks, the spread is 4 + 3 + 3. The question asks for the **worst** case, so it is the rack with 4.

### A VM can only join a set when it is created

This is the most important rule, and the exam tests it often.

| Allowed | Not allowed |
|---|---|
| Choose the set when you **create** the VM | Add an **existing** VM to a set |
| Mix different VM sizes in one set | Move a VM between sets, or out of a set |
| Delete the VM, keep its disks, and recreate it in the set | Use a set **and** a zone on the same VM |
| | Put the set in a different region or resource group from its VMs |

Why? Azure picks the physical placement (which rack and which update group) when it first places the VM. Changing that later would mean physically moving it, so Azure makes you recreate it.

### What an availability set does NOT protect against

All the racks in an availability set are in **one datacentre building**. If the building fails, the whole set fails. For that, you need zones (Part 3).

### Check your understanding

**Q3.** An availability set has 12 VMs, 3 fault domains and 4 update domains. How many VMs can go down in a rack failure? During maintenance?

<details>
<summary>Answer</summary>

Rack failure: 12 ÷ 3 = 4. Maintenance: 12 ÷ 4 = 3.

</details>

**Q4.** VM1 already exists. You need it in availability set AS1. What do you do?

<details>
<summary>Answer</summary>

Delete VM1 but keep its disks, then recreate VM1 from those disks and choose AS1 during creation. You can't add an existing VM to a set.

</details>

**Q5.** A question says "a rack loses power". Do you divide by fault domains or update domains?

<details>
<summary>Answer</summary>

Fault domains. A rack is a fault domain. Update domains are for maintenance.

</details>

---

## Part 3: Availability zones

### The situation

The whole datacentre building that hosts your availability set loses power. Both VMs go down, even though they were in different racks. How do I survive the loss of an entire building?

### The idea: separate buildings in one region

An **availability zone** is a separate datacentre (or group of datacentres) inside one region, with its **own power, cooling and network**.

```
Region: Norway East
  Zone 1 (building with own power)    -> web-1
  Zone 2 (building with own power)    -> web-2
  Zone 3 (building with own power)
```

- A zone-enabled region has **at least 3 zones**
- The zones are connected by fast, low-latency links
- If one zone fails, the others keep running
- 2+ VMs across 2+ zones gives **99.99%**

On-premises, this is like having servers in **two separate buildings** on the same campus, instead of two racks in the same server room.

### A VM is in a zone OR in a set, never both

This follows from Part 0. A zone already spreads VMs further than a set does (buildings instead of racks), so Azure doesn't let you combine them on one VM.

### Logical zones vs physical zones

Here is a surprise: **zone 1 in your subscription may be a different building from zone 1 in another subscription.**

```
Subscription A:  zone 1 -> building X
Subscription B:  zone 1 -> building Z
```

Azure maps the zone numbers **per subscription**. The numbers you see are **logical** zones. The real buildings are **physical** zones.

- Inside **one** subscription, zone 1 is always the same building
- Across subscriptions, compare the **physical zone mappings** if you need resources in the same building

Exam words to look for: *"logical zone"*, *"physical zone"*, *"per subscription"*.

### Zonal vs zone-redundant

There are two ways a resource can use zones:

- **Zonal** = pinned to **one** zone. If that zone fails, the resource fails.
- **Zone-redundant** = spread across **all** zones. It survives one zone failing.

| Resource | Zonal | Zone-redundant |
|---|---|---|
| Virtual machine | Yes (`--zone 1`) | No. Deploy one VM per zone instead (or use a scale set). |
| Managed disk | LRS: stays in the VM's zone | ZRS (Premium SSD or Standard SSD): can attach from any zone |
| Public IP (Standard) | Yes, pinned to one zone | Yes, **the default** when no zone is chosen |
| Load balancer (Standard) | Front end pinned to one zone | Front end spans zones, **the default** |

The **Basic** SKU of public IP and load balancer has **no zone support at all**.

### The standard pattern: zonal VMs behind a zone-redundant load balancer

```
         Zone-redundant Standard public IP
                      |
      Zone-redundant Standard load balancer
           /                         \
  Zone 1: web-1                  Zone 2: web-2
```

If zone 1 fails:

1. The load balancer and its public IP keep working (they are zone-redundant)
2. The health probe sees that web-1 is down
3. All traffic goes to web-2
4. Your site stays up, but with **half the capacity**. A third VM in zone 3 gives you headroom.

### What a zonal VM needs

| Setting | Must be |
|---|---|
| Availability options | **Availability zone**, then zone 1, 2 or 3 |
| Disks | **Managed disks** (unmanaged disks are not supported) |
| Size | Offered in that zone (check with `az vm list-skus`) |
| Public IP and load balancer | **Standard** SKU |
| Availability set | Not allowed at the same time |

Exam pattern: a planned VM shows *"No infrastructure redundancy required"* and *"Use managed disks: No"*. Which two settings must you change to put it in a zone? **Use managed disks** and **Availability options**. Not the image, size or disk type.

### Sets or zones?

| | Availability set | Availability zones |
|---|---|---|
| Spreads VMs across | Racks and update groups in **one building** | **Separate buildings** in one region |
| Survives | Host, rack, maintenance | All of that, plus a **building failure** |
| SLA (2+ VMs) | 99.95% | **99.99%** |
| Chosen | At VM creation, fixed | At VM creation, fixed |
| Load balancer | Basic or Standard | **Standard only** |
| Use when | The region has no zones, or for legacy setups | **The default for anything new** |

### Check your understanding

**Q6.** Your VMs are in an availability set. The whole datacentre loses power. Does the set protect you?

<details>
<summary>Answer</summary>

No. An availability set spreads VMs across racks inside one building. To survive a building failure, use availability zones.

</details>

**Q7.** You create a Standard public IP and don't choose a zone. Is it zonal or zone-redundant?

<details>
<summary>Answer</summary>

Zone-redundant. That is the default for Standard public IPs and load balancers when no zone is chosen.

</details>

**Q8.** You and a colleague both deploy a VM to "zone 1" in Norway East, from different subscriptions. Are they in the same building?

<details>
<summary>Answer</summary>

Not necessarily. Zone numbers are logical and mapped per subscription. Compare the physical zone mappings to find out.

</details>

---

## Part 4: Placement and capacity

### The situation

Two problems:

1. Your app server and database server talk to each other constantly, and every millisecond of network delay slows the app down.
2. Your company is launching a product next month and needs 50 extra VMs that day. What if Azure doesn't have that size available when you need it?

### Proximity placement groups: keep VMs close

A **proximity placement group** tells Azure: "Put these VMs physically **close together** in the same datacentre." That gives the **lowest possible network latency** between them.

- Works for VMs, availability sets and scale sets
- Everything in the group is in **one region**, and at most **one zone**
- It is **free**, but the trade-off is that Azure has fewer places to put your VMs, so you may get **allocation failures**
- Tip: create the most demanding VM first, and use similar size families

Exam phrase: *"lowest possible network latency between VM1 and VM2"* = **proximity placement group**.

Notice this is the **opposite** of availability: availability spreads VMs apart, proximity pulls them together. You trade some resilience for speed.

### Capacity reservations: guarantee the size is there

A **capacity reservation** keeps a number of VMs of one size **reserved for you** in a region or zone, whether you use them or not.

- You pay for it **as if the VMs were running**
- No long commitment. Delete it any time.
- Use it for disaster-recovery targets, launches, or planned scale events

Don't confuse it with **reserved instances**:

| | Capacity reservation | Reserved instances |
|---|---|---|
| Guarantees the capacity exists | **Yes** | No |
| Gives a discount | No | **Yes** |
| Commitment | None | 1 or 3 years |

One guarantees **space**, the other gives you a **lower price**. You can combine them.

### Check your understanding

**Q9.** You need the lowest possible latency between an app VM and a database VM. What do you use?

<details>
<summary>Answer</summary>

A proximity placement group.

</details>

**Q10.** You want a discount for running VMs for 3 years. Capacity reservation or reserved instances?

<details>
<summary>Answer</summary>

Reserved instances. Capacity reservations guarantee space but give no discount.

</details>

---

## Part 5: Regions and the load balancer

### Region pairs

#### The situation

A huge storm takes out a whole region. Where does my data go, and where does Microsoft recover first?

#### The idea

Microsoft links many regions into **pairs** inside the same geography:

| Region | Pair |
|---|---|
| West Europe | North Europe |
| Norway East | Norway West |
| Sweden Central | Sweden South |
| UK South | UK West |
| East US | West US |

What a pair gives me:

- **Updates one region at a time.** Microsoft never updates both regions of a pair at once.
- **Recovery priority.** In a large outage, one region of each pair is restored first.
- **GRS and GZRS storage** copy my data to the paired region, so my data stays in the same geography.

The pairs are fixed by Microsoft. Many newer regions have zones but no pair.

#### Why the labs use Sweden Central, not Norway West

| Region | Zones | Open to everyone? | Role |
|---|---|---|---|
| Norway East | 3 | Yes | Primary lab region |
| Norway West | None | **No**, restricted access, few sizes | Only the GRS storage target |
| Sweden Central | 3 | Yes | Second lab region |

Norway West is Norway East's pair, but it has no zones and you can't deploy freely there. So I choose **zones first**, then **a second region that suits you**. 
My disaster-recovery region doesn't have to be the official pair.

#### Surviving a region failure

| Tool | What it does | Remember it as |
|---|---|---|
| **Azure Site Recovery** | Keeps copying VMs to a second region, so you can **fail over** | Keeps the **service running** |
| **Azure Backup** | Keeps backups, and can restore in the paired region | Restores **data**, but slower |
| **Traffic Manager / Front Door** | Sends users to the region that is still healthy | Routes **users** |
| **Compute Gallery** | Copies images to other regions | Rebuilds VMs **from images** |

Exam phrase: *"available if the region fails"* = **Site Recovery**. Backup brings data back; it doesn't keep the service running.

### The load balancer

#### The situation

I have three web VMs. Users need one address, and traffic should only go to VMs that are healthy.

#### The idea: four parts

```
Users
  -> Front end (the address users connect to, e.g. 20.1.2.3:80)
     -> Rule (TCP 80 on the front end -> port 80 on the VMs, using a probe)
        -> Backend pool (the VMs)
           Health probe checks each VM: healthy VMs get traffic, failed VMs don't
```

| Part | What it does |
|---|---|
| **Front end** | The IP address users connect to: public, or private for an **internal** load balancer |
| **Backend pool** | The VMs that receive the traffic |
| **Health probe** | Checks each VM regularly (a TCP port, or an HTTP path). A VM that fails gets **no new traffic**. |
| **Rule** | Connects front end port to backend port. **No rule = no traffic at all.** |

The health probe is how the "zone 1 fails" example in Part 3 works: the probe notices the VM is down and stops sending users there.

#### Basic vs Standard

The Basic load balancer was **retired in September 2025**, but older exam questions still use it.

| | Basic (retired) | Standard |
|---|---|---|
| Which VMs can be in the backend pool | VMs in **one availability set** or **one scale set** | **Any VM in the same VNet** |
| Availability zones | No | Yes |
| SLA | None | 99.99% |
| Security | Open by default | **Closed by default**: an NSG must allow the traffic |

The rule to remember for old questions: **a Basic pool holds one availability set (or one scale set) only.**

#### Reading a load balancer exam question

Always check the **SKU first**. Then:

- **Basic:** are all the VMs in the **same VNet** and the **same availability set**?
- **Standard:** are all the VMs in the **same VNet**?

### Check your understanding

**Q11.** A Basic internal load balancer is in VNET1. VM1 is in VNET1 and set AS1. VM2 is in VNET1 and set AS2. Can it balance between them?

<details>
<summary>Answer</summary>

No. A Basic load balancer's pool can only hold one availability set, and these are in two different sets. A Standard load balancer could balance them.

</details>

**Q12.** You delete the only rule on a load balancer. What happens?

<details>
<summary>Answer</summary>

No traffic is balanced at all. Deleting the rule doesn't mean "balance all ports". Balancing all ports needs a special HA ports rule, which only exists on Standard.

</details>

**Q13.** The health probe checks `/probe.htm`, but that file is missing on VM2. What happens to VM2?

<details>
<summary>Answer</summary>

The probe gets a 404 error, marks VM2 as unhealthy, and VM2 receives no new traffic.

</details>

---

## Summary: the ideas that explain the facts

| Idea | Facts it explains |
|---|---|
| **Spread VMs across the level you want to survive.** | Set = racks and maintenance. Zones = buildings. Second region = region failure. |
| **More spreading, higher SLA.** | 99.9% single VM, 99.95% set, 99.99% zones |
| **Fault domain = rack (columns). Update domain = maintenance wave (rows).** | Rack failure: divide by FDs. Maintenance: divide by UDs. Round up. |
| **Placement is decided when the VM is created.** | You can't add an existing VM to a set, or move it between sets. |
| **Zones are bigger than sets.** | A VM uses a zone or a set, never both. |
| **Zone numbers are per subscription.** | Zone 1 may be a different building in another subscription. |
| **Basic load balancer = one set only. Standard = any VM in the VNet.** | Most load balancer exam questions |

---

## Hands-on practice

This lab creates 4 VMs, so remember the clean-up step at the end.

Each step says **what to expect**. Compare that with what you see.

```bash
RG=cat-avail-rg
LOC=norwayeast
SIZE=Standard_FX2ms_v2     # in Norway East this size is only in zones 1 and 3
```

**Step 1. Network: a VNet, a subnet, and an NSG that allows HTTP**

The Standard load balancer is closed by default, so the NSG must allow port 80.

```bash
az group create -n $RG -l $LOC

az network nsg create -g $RG -n web-nsg
az network nsg rule create -g $RG --nsg-name web-nsg -n allow-http \
  --priority 100 --access Allow --direction Inbound --protocol Tcp \
  --destination-port-ranges 80

az network vnet create -g $RG -n avail-vnet --address-prefix 10.20.0.0/16 \
  --subnet-name web --subnet-prefix 10.20.1.0/24 --network-security-group web-nsg
```

**Step 2. A small web page on each VM (cloud-init)**

Each VM serves a page showing its own name, so you can see which VM answers. It uses Python, which is already on Ubuntu, so the VMs don't need internet access.

```bash
cat > web.yaml <<'EOF'
#cloud-config
write_files:
  - path: /etc/systemd/system/web.service
    content: |
      [Unit]
      Description=Simple web page
      [Service]
      ExecStartPre=/bin/sh -c 'mkdir -p /srv/web && hostname > /srv/web/index.html'
      ExecStart=/usr/bin/python3 -m http.server 80 --directory /srv/web
      Restart=always
      [Install]
      WantedBy=multi-user.target
runcmd:
  - systemctl daemon-reload
  - systemctl enable --now web
EOF
```

**Step 3. An availability set with two VMs**

```bash
az vm availability-set create -g $RG -n web-as \
  --platform-fault-domain-count 2 --platform-update-domain-count 5

for VM in as-vm1 as-vm2; do
  az vm create -g $RG -n $VM --availability-set web-as \
    --image Ubuntu2404 --size $SIZE \
    --admin-username azureuser --generate-ssh-keys \
    --vnet-name avail-vnet --subnet web --nsg "" --public-ip-address "" \
    --custom-data web.yaml \
    --os-disk-delete-option Delete --nic-delete-option Delete
done
```

**Step 4. Check which fault and update domain each VM got**

Before you run this, predict the result from the round-robin rule.

```bash
for VM in as-vm1 as-vm2; do
  az vm get-instance-view -g $RG -n $VM \
    --query "{vm:name, faultDomain:instanceView.platformFaultDomain, updateDomain:instanceView.platformUpdateDomain}" -o table
done
```

Expect: as-vm1 in FD 0 / UD 0, as-vm2 in FD 1 / UD 1. They are in different racks and different maintenance waves.

**Step 5. Two zonal VMs, in zone 1 and zone 3**

```bash
az vm create -g $RG -n zone-vm1 --zone 1 \
  --image Ubuntu2404 --size $SIZE \
  --admin-username azureuser --generate-ssh-keys \
  --vnet-name avail-vnet --subnet web --nsg "" --public-ip-address "" \
  --custom-data web.yaml \
  --os-disk-delete-option Delete --nic-delete-option Delete

az vm create -g $RG -n zone-vm3 --zone 3 \
  --image Ubuntu2404 --size $SIZE \
  --admin-username azureuser --generate-ssh-keys \
  --vnet-name avail-vnet --subnet web --nsg "" --public-ip-address "" \
  --custom-data web.yaml \
  --os-disk-delete-option Delete --nic-delete-option Delete
```

**Step 6. Try to add an existing VM to the availability set**

```bash
AS_ID=$(az vm availability-set show -g $RG -n web-as --query id -o tsv)
az vm update -g $RG -n zone-vm1 --set availabilitySet.id=$AS_ID
```

Expect: an error. A VM can only join a set when it is created, and a zonal VM can never be in a set.

**Step 7. A zone-redundant Standard load balancer in front of the zonal VMs**

```bash
# No --zone given, so the Standard public IP is zone-redundant
az network public-ip create -g $RG -n lb-pip --sku Standard

az network lb create -g $RG -n web-lb --sku Standard \
  --public-ip-address lb-pip --frontend-ip-name fe --backend-pool-name pool

az network lb probe create -g $RG --lb-name web-lb -n http-probe \
  --protocol Http --port 80 --path /

az network lb rule create -g $RG --lb-name web-lb -n http-rule \
  --protocol Tcp --frontend-port 80 --backend-port 80 \
  --frontend-ip-name fe --backend-pool-name pool --probe-name http-probe

# Put both zonal VMs' NICs in the backend pool
for VM in zone-vm1 zone-vm3; do
  NIC_ID=$(az vm show -g $RG -n $VM --query "networkProfile.networkInterfaces[0].id" -o tsv)
  NIC=$(az network nic show --ids $NIC_ID --query name -o tsv)
  IPC=$(az network nic show --ids $NIC_ID --query "ipConfigurations[0].name" -o tsv)
  az network nic ip-config address-pool add -g $RG --nic-name $NIC \
    --ip-config-name $IPC --lb-name web-lb --address-pool pool
done

LB_IP=$(az network public-ip show -g $RG -n lb-pip --query ipAddress -o tsv)
echo $LB_IP
```

**Step 8. Test, then "lose a zone"**

Open `http://<LB_IP>` in your browser and refresh a few times. Expect: you see `zone-vm1` or `zone-vm3`.

Now simulate zone 1 failing by deallocating its VM:

```bash
az vm deallocate -g $RG -n zone-vm1
```

Wait about a minute, then refresh again. Expect: the site still answers, but only with `zone-vm3`. The probe noticed zone-vm1 was down and stopped sending traffic to it.

**Step 9. Clean up**

```bash
az group delete -n $RG --yes --no-wait
```

---

## PowerShell equivalents

```powershell
$rg = "cat-avail-rg"; $loc = "norwayeast"

# Availability set (Aligned = for managed disks)
New-AzAvailabilitySet -ResourceGroupName $rg -Name web-as -Location $loc `
  -Sku Aligned -PlatformFaultDomainCount 2 -PlatformUpdateDomainCount 5

# A VM in the set / a VM in a zone
New-AzVM -ResourceGroupName $rg -Name as-vm1 -Location $loc -AvailabilitySetName web-as -Image Ubuntu2404 -Size Standard_D2s_v5 -Credential $cred
New-AzVM -ResourceGroupName $rg -Name zone-vm1 -Location $loc -Zone 1 -Image Ubuntu2404 -Size Standard_D2s_v5 -Credential $cred

# Standard load balancer
$pip   = New-AzPublicIpAddress -ResourceGroupName $rg -Name lb-pip -Location $loc -Sku Standard -AllocationMethod Static
$fe    = New-AzLoadBalancerFrontendIpConfig -Name fe -PublicIpAddress $pip
$be    = New-AzLoadBalancerBackendAddressPoolConfig -Name pool
$probe = New-AzLoadBalancerProbeConfig -Name http-probe -Protocol Http -Port 80 -RequestPath / -IntervalInSeconds 5 -ProbeCount 2
$rule  = New-AzLoadBalancerRuleConfig -Name http-rule -FrontendIpConfiguration $fe -BackendAddressPool $be `
           -Probe $probe -Protocol Tcp -FrontendPort 80 -BackendPort 80
New-AzLoadBalancer -ResourceGroupName $rg -Name web-lb -Location $loc -Sku Standard `
  -FrontendIpConfiguration $fe -BackendAddressPool $be -Probe $probe -LoadBalancingRule $rule
```

---

## Troubleshooting (symptom → cause → fix)

| Symptom | Likely cause | Fix |
|---|---|---|
| Can't add an existing VM to an availability set | Sets can only be chosen **at creation** | Recreate the VM (keep the disks) |
| Can't choose a zone for a VM in a set | A VM is in a **set or a zone**, never both | Pick one |
| Size not available in zone 2 | Not every size is in every zone | `az vm list-skus -l <region> --size <size> --zone` |
| Load balancer IP doesn't answer | **Standard LB is closed by default** – NSG must allow the port | NSG allow 80/443 inbound |
| Some backends never get traffic | **Health probe** fails (app not listening on probe port/path, or blocked) | Fix app/probe path, allow `AzureLoadBalancer` service tag (default rule) |
| Can't attach a public IP / mix SKUs | Basic and Standard can't be mixed | Standard LB ↔ Standard public IP |
| Backend VMs lost internet access | Standard LB backends without public IP have no default outbound | **NAT gateway** or LB **outbound rules** |
| Both VMs went down during maintenance | They were in the **same update domain** (or not in a set at all) | Use a set (or zones) with 2+ VMs |

---

## Break it and fix it

| # | Break it | What you should see | Fix |
|---|---|---|---|
| 1 | Delete the NSG **allow-http** rule | LB IP stops answering | Recreate the rule |
| 2 | Stop the web service on zone-vm3 (`systemctl stop web` via Run Command) | Only zone-vm1 answers – probe removed vm3 | Start the service |
| 3 | Change the probe path to `/health` (doesn't exist) | **No** backend answers | Path back to `/` |
| 4 | `az vm update --set availabilitySet.id=...` on a zonal VM | Error | – sets only at creation, never with zones |

---

## Exam-style questions

Try to work out each answer using the ideas, before opening it.

### Question 1

Availability set WEBPROD-AS has **2 fault domains**, **7 update domains** and **14 VMs**, placed evenly. What is the maximum number of VMs that become unavailable in each case?

- Case 1: Microsoft performs planned maintenance in the region.
- Case 2: The rack hosting part of the set loses power.

Options:

- A. Maintenance: 14, Rack failure: 2
- B. Maintenance: 10, Rack failure: 10
- C. Maintenance: 7, Rack failure: 14
- D. Maintenance: 2, Rack failure: 7

<details>
<summary>Answer and reasoning</summary>

**D.**

1. Maintenance happens one **update domain** at a time: 14 ÷ 7 = **2** VMs.
2. A rack is a **fault domain**: 14 ÷ 2 = **7** VMs.
3. Watch out: option C swaps the two. Always check which kind of domain the event uses.

</details>

### Question 2

You create LB1: SKU **Basic**, type **Internal**, in **VNET1**, subnet Subnet12.

| VM | VNet | Availability set |
|---|---|---|
| VM1 | VNET1 | AS1 |
| VM2 | VNET1 | AS2 |
| VM3 | VNET2 | AS3 |
| VM4 | VNET2 | AS3 |
| VM5 | VNET1 | AS4 |
| VM6 | VNET1 | AS4 |

| Statement | Yes / No |
|---|---|
| 1. LB1 can balance traffic between VM1 and VM2. | ? |
| 2. LB1 can balance traffic between VM3 and VM4. | ? |
| 3. LB1 can balance traffic between VM5 and VM6. | ? |

<details>
<summary>Answer and reasoning</summary>

First, read the SKU: **Basic**. So the VMs must be in **VNET1** and in **one availability set**.

1. **No.** VM1 and VM2 are in VNET1, but in different sets (AS1 and AS2).
2. **No.** VM3 and VM4 share a set, but they are in VNET2, not LB1's VNet.
3. **Yes.** VM5 and VM6 are both in VNET1 and both in AS4.

With a Standard load balancer, any of the VNET1 pairs would work, because Standard accepts any VM in the VNet.

</details>

### Question 3

LB1 is a **Basic** load balancer. VM1 and VM2 run IIS and are in its backend pool. There is an HTTP health probe on `/Probe1.htm`, and Rule1 sends TCP 80 to port 80.

| Statement | Yes / No |
|---|---|
| 1. VM1 is in the same availability set as VM2. | ? |
| 2. If Probe1.htm is present on VM1 and VM2, LB1 balances TCP port 80 between them. | ? |
| 3. If you delete Rule1, LB1 balances all requests between VM1 and VM2 on all ports. | ? |

<details>
<summary>Answer and reasoning</summary>

1. **Yes.** They are both in a Basic load balancer's pool, and a Basic pool can only hold one availability set.
2. **Yes.** The probe gets HTTP 200, so both VMs are healthy, and Rule1 balances port 80.
3. **No.** No rule means no traffic. Balancing all ports needs an HA ports rule, which only Standard has.

</details>
