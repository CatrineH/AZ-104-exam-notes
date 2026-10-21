# NSG Inbound and Outbound Rules

[← Back to README](../README.md)

## In one line
A **Network Security Group (NSG)** is a free, **stateful** firewall on a **subnet** or **NIC** that allows or denies traffic with **priority-ordered rules** – first match wins.

## When to use it (and when not)
| Need | Use |
|---|---|
| Allow/deny by IP, port, protocol for subnets and VMs | **NSG** |
| Group VMs by role (web, db) instead of IPs in rules | **ASG** (application security group) in NSG rules |
| Allow traffic to an Azure service without knowing its IPs | **Service tag** (`Storage`, `Sql`, `AzureLoadBalancer`…) |
| Filter by **domain name** (FQDN), central logging, threat intel | **Azure Firewall** |
| Protect a website from SQL injection / XSS | **WAF** on Application Gateway / Front Door |

## Key concepts

| Direction | Controls traffic… | Example |
|---|---|---|
| **Inbound** | **Coming in** to a subnet or VM | Allow RDP (3389) from my IP to a VM |
| **Outbound** | **Going out** from a subnet or VM | Deny the database subnet access to the internet |

The direction is always seen **from the VM's or subnet's point of view**.

### Rule properties

| Property | Options / example |
|---|---|
| **Name** | `Allow-HTTP-Inbound` |
| **Priority** | **100–4096**. **Lower number = checked first** |
| **Source** | IP address, CIDR range, **service tag**, **application security group (ASG)**, or Any |
| **Source port** | Usually `*` (clients use random ports) |
| **Destination** | IP address, CIDR range, service tag, ASG, or Any |
| **Destination port** | `443`, `80,443`, or a range like `8000-8100` |
| **Protocol** | TCP, UDP, ICMP, ESP, AH, or Any |
| **Action** | **Allow** or **Deny** |

The **source port** is almost always `*`. The **destination port** is the port the service listens on.

### How rules are evaluated
1. Rules are checked in **priority order**, lowest number first
2. The **first rule that matches** is used – processing **stops** there
3. If nothing you created matches, the **default rules** decide

| Priority | Name | Port | Source | Action |
|---|---|---|---|---|
| 100 | Allow-RDP-MyIP | 3389 | 192.168.2.10 | Allow |
| 200 | Deny-RDP-All | 3389 | Any | Deny |

- RDP from **192.168.2.10** → matches 100 → **allowed**
- RDP from any other IP → skips 100, matches 200 → **denied**
- If the priorities were swapped, **everyone** would be denied – including me

### Default rules
Every NSG has default rules. I **can't delete** them, but I can **override** them with my own rules, since mine always have a lower number.

**Inbound**

| Priority | Name | Source → Destination | Action |
|---|---|---|---|
| 65000 | AllowVnetInBound | VirtualNetwork → VirtualNetwork | Allow |
| 65001 | AllowAzureLoadBalancerInBound | AzureLoadBalancer → Any | Allow |
| 65500 | **DenyAllInBound** | Any → Any | **Deny** |

**Outbound**

| Priority | Name | Source → Destination | Action |
|---|---|---|---|
| 65000 | AllowVnetOutBound | VirtualNetwork → VirtualNetwork | Allow |
| 65001 | **AllowInternetOutBound** | Any → Internet | **Allow** |
| 65500 | DenyAllOutBound | Any → Any | Deny |

**What this means:**
- **Inbound from the internet is blocked** by default
- **Outbound to the internet is allowed** by default
- Traffic **inside the VNet** is allowed both ways – and `VirtualNetwork` includes **peered VNets** and connected on-premises networks

### NSGs are stateful
If a connection is allowed in **one** direction, the **reply** is allowed back automatically.
- Allow inbound 443 → the response to the client goes out **without** an outbound rule
- Allow outbound 443 (default) → the answer from the website comes in **without** an inbound rule

I only write rules for **who starts** the connection.

### Subnet NSG and NIC NSG together
```
Inbound:   Internet ──► Subnet NSG ──► NIC NSG ──► VM
Outbound:  VM ──► NIC NSG ──► Subnet NSG ──► Internet
```
- When both exist, traffic must be **allowed by both**
- **Inbound:** subnet NSG first, then NIC NSG
- **Outbound:** NIC NSG first, then subnet NSG
- Best practice: use **subnet NSGs** for most rules – easier to manage
- One NSG can be attached to **many** subnets and NICs (same region)

### Service tags
A **service tag** is a name for a group of IP addresses managed by Microsoft.

| Service tag | Means |
|---|---|
| `Internet` | Everything outside Azure's virtual networks |
| `VirtualNetwork` | The VNet, peered VNets, and connected on-premises networks |
| `AzureLoadBalancer` | Azure's load balancer health probes |
| `Storage` / `Storage.WestEurope` | Azure Storage (all regions / one region) |
| `Sql` | Azure SQL Database |
| `GatewayManager` | Management traffic for Application Gateway and VPN gateways |
| `AzureCloud` | All Azure public IP addresses |

Example: allow a VM to reach **only** storage in West Europe, and nothing else on the internet:

| Priority | Direction | Destination | Port | Action |
|---|---|---|---|---|
| 100 | Outbound | `Storage.WestEurope` | 443 | Allow |
| 200 | Outbound | `Internet` | Any | Deny |

### Application security groups (ASGs)
An **ASG** groups VMs by **role** instead of by IP address. You use the ASG as source or destination in rules.
```
asg-web  ← web VMs' NICs
asg-db   ← database VMs' NICs
```
| Priority | Direction | Source | Destination | Port | Action |
|---|---|---|---|---|---|
| 100 | Inbound | Internet | `asg-web` | 443 | Allow |
| 110 | Inbound | `asg-web` | `asg-db` | 1433 | Allow |

- New VMs only need to be added to the right ASG – **the rules don't change**
- All NICs in an ASG must be in the **same VNet**

### Example: three-tier app

| Subnet | Inbound rule | Outbound rule |
|---|---|---|
| **Frontend** | Allow 443 from `Internet` | Default |
| **Application** | Allow 8080 from the frontend subnet | Default |
| **Database** | Allow 1433 from the application subnet. Deny everything else from `VirtualNetwork` | Deny `Internet` |

In the database subnet, the deny rule is needed because the default **AllowVnetInBound** would otherwise let the frontend reach the database directly.

## Exam traps
- **Lower priority number = higher priority**. First match wins
- Default: **inbound internet denied**, **outbound internet allowed**, **VNet traffic allowed** (incl. peered VNets)
- NSGs are **stateful** – no rule needed for return traffic
- Subnet + NIC NSG → **both must allow**
- Default rules can't be deleted, only overridden
- "Which rule blocks the traffic?" → **IP flow verify** or **effective security rules**
- The **source port** in a rule is almost always `*`
- NSG and the subnet/NIC must be in the **same region**
- Count questions: 5 VMs in one subnet, same rules → **1 NSG** on the subnet

## Do it

### Portal
- **Network security groups → + Create** → then **Settings → Subnets → Associate** (or **Network interfaces → Associate**)
- **Inbound/Outbound security rules → + Add** → source (My IP address / Service Tag / ASG) → destination port → priority → action
- VM → **Networking** → see NIC and subnet NSG rules side by side, **Effective security rules**
- **Network Watcher → IP flow verify** → VM, direction, protocol, local/remote IP and port → shows **Access allowed/denied + rule name**

### Azure CLI
```bash
az network nsg create -g AZ104-catrine-rg -n nsg-web

az network nsg rule create \
  --resource-group AZ104-catrine-rg \
  --nsg-name nsg-web \
  --name Allow-HTTPS-Inbound \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 443

# Attach to a subnet
az network vnet subnet update -g AZ104-catrine-rg --vnet-name vnet-lab -n snet-web --network-security-group nsg-web

# ASG + rule that uses it
az network asg create -g AZ104-catrine-rg -n asg-web
az network nic ip-config update -g AZ104-catrine-rg --nic-name nic-web1 -n ipconfig1 --application-security-groups asg-web

# Troubleshoot
az network nic list-effective-nsg -g AZ104-catrine-rg -n nic-web1
az network watcher test-ip-flow -g AZ104-catrine-rg --vm vm-web1 --direction Inbound \
  --protocol TCP --local 10.0.1.4:443 --remote 203.0.113.10:50000
```

### PowerShell
```powershell
$nsg = Get-AzNetworkSecurityGroup -Name nsg-web -ResourceGroupName AZ104-catrine-rg

$nsg | Add-AzNetworkSecurityRuleConfig `
  -Name "Deny-Internet-Outbound" `
  -Direction Outbound `
  -Priority 4000 `
  -Access Deny `
  -Protocol * `
  -SourceAddressPrefix * `
  -SourcePortRange * `
  -DestinationAddressPrefix Internet `
  -DestinationPortRange *

$nsg | Set-AzNetworkSecurityGroup

# Attach to a subnet
$vnet = Get-AzVirtualNetwork -Name vnet-lab -ResourceGroupName AZ104-catrine-rg
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name snet-web -AddressPrefix 10.0.1.0/24 -NetworkSecurityGroup $nsg
$vnet | Set-AzVirtualNetwork

Get-AzEffectiveNetworkSecurityGroup -NetworkInterfaceName nic-web1 -ResourceGroupName AZ104-catrine-rg
```

In PowerShell, `Add-AzNetworkSecurityRuleConfig` only changes the object in memory. **`Set-AzNetworkSecurityGroup`** saves it to Azure.

## Troubleshooting

| Tool | Use it to |
|---|---|
| **Effective security rules** (on the NIC) | See the **combined** rules from both the subnet NSG and the NIC NSG |
| **IP flow verify** (Network Watcher) | Test if a specific packet is allowed or denied – and **which rule** decided it |
| **VNet flow logs** (Network Watcher) | Log all traffic for later analysis (replacing the older NSG flow logs) |

**Checklist when traffic is blocked:**
1. Is there an NSG on **both** the subnet and the NIC? Both must allow it
2. Is a **deny rule with a lower number** matching first?
3. Is the **destination port** correct (the service's port, not the client's)?
4. Is the **OS firewall** (Windows Firewall, `ufw`) blocking it? The NSG isn't the only firewall
5. Is the service actually **listening** on that port in the VM?
6. Use **IP flow verify** to find the rule

| Symptom | Likely cause | Fix |
|---|---|---|
| RDP/SSH times out from home | No allow rule for your IP – DenyAllInBound hits | Allow 3389/22 from **your** public IP (or use Bastion) |
| Worked yesterday, not today | Your **home IP changed** | Update the rule |
| Ping between VMs fails, TCP works | Windows Firewall blocks ICMP | Allow ICMP in the OS |
| Frontend can reach the DB directly | **AllowVnetInBound** default | Add a deny rule for VirtualNetwork on the DB subnet |
| Rule saved but no effect (PowerShell) | Forgot **`Set-AzNetworkSecurityGroup`** | Run it |
| Load balancer probe fails after your deny-all rule | Your rule blocked `AzureLoadBalancer` | Allow `AzureLoadBalancer` above the deny |

## Clean up
```bash
az network vnet subnet update -g AZ104-catrine-rg --vnet-name vnet-lab -n snet-web --network-security-group ""
az network nsg delete -g AZ104-catrine-rg -n nsg-web
```
An NSG that is still associated can't be deleted – detach it first.

## Check yourself
<details><summary>1. Rule 200 allows 3389 from Any, rule 100 denies 3389 from Any. Can anyone RDP?</summary>

No – **100** is checked first and denies.
</details>

<details><summary>2. Subnet NSG allows 443, NIC NSG has only default rules. Can internet clients reach the VM on 443?</summary>

No – the **NIC NSG's DenyAllInBound** blocks it. Both must allow.
</details>

<details><summary>3. You allow inbound 443. Do you need an outbound rule for the replies?</summary>

No – NSGs are **stateful**.
</details>

## Related
- [Firewalls](firewalls.md)
- [Ports and Protocols](ports-and-protocols.md)
- [Subnets and Routing](subnets-and-routing.md)
- [Virtual Networks and VNet Peering](vnet-peering.md)
