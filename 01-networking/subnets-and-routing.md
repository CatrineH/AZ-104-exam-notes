# Subnets and Routing

[← Back to README](../README.md)

## In one line
**Subnets** split a VNet into zones, Azure **system routes** connect them automatically, and **user-defined routes (UDRs)** override the default path – for example to force traffic through a firewall.

## When to use it (and when not)
| Need | Use |
|---|---|
| Separate web, app, and database tiers | **Subnets** (+ NSGs) |
| Subnets in the same VNet talk to each other | Nothing – **system routes** do it |
| Send all outbound/inter-subnet traffic through a firewall/NVA | **Route table (UDR)** with next hop **Virtual appliance** |
| Block a subnet from reaching the internet by routing | UDR `0.0.0.0/0` → next hop **None** |
| Connect two VNets | **Peering** (not routes) |
| Learn on-prem routes automatically | **BGP** via VPN/ExpressRoute gateway |

## Key concepts

### Subnets
- A virtual network can be divided into one or more subnets
- Subnets improve **security**, **performance**, and **manageability**
- Each subnet must have a **unique address range** that does not overlap with other subnets in the same VNet

**Analogy:** a hospital with separate departments.

| Subnet | Purpose | Address range |
|---|---|---|
| subnet1 | Frontend | 10.0.1.0/24 |
| subnet2 | Application | 10.0.2.0/24 |
| subnet3 | Database | 10.0.3.0/24 |

For how address ranges and CIDR work, see [IP Addressing and Subnetting](ip-addressing-subnetting.md).

### Routing
**Routing** directs traffic between network segments. **Rule:** devices in different subnets need a router to talk – in Azure, that router is built in.

**System routes** (created automatically):

| Destination | Next hop |
|---|---|
| The VNet's own address space | **Virtual network** |
| `0.0.0.0/0` | **Internet** |
| Peered VNet ranges | **VNet peering** |
| On-prem ranges (with a gateway) | **Virtual network gateway** |
| Some private ranges (10/8, 172.16/12, 192.168/16) not in the VNet | **None** (dropped) |

### User-defined routes (UDR)
A **route table** holds your routes and is **associated with subnets** (not NICs).

| Next hop type | Use for |
|---|---|
| **Virtual appliance** | Send to a firewall/NVA – you give its **private IP** |
| **Virtual network gateway** | Send to the VPN gateway (forced tunnelling to on-prem) |
| **Virtual network** | Override back to normal VNet routing |
| **Internet** | Send straight to the internet |
| **None** | **Drop** the traffic |

**Which route wins?**
1. **Longest prefix match** – `10.0.2.0/24` beats `10.0.0.0/16`
2. Same prefix → **UDR** beats **BGP** beats **system route**

**NVA requirement:** the NVA's NIC needs **IP forwarding enabled** (and forwarding in its OS) or it drops traffic not addressed to itself.

### Hub-and-spoke with a firewall
```
Spoke1 subnet ─ UDR 0.0.0.0/0 → 10.0.0.4 (Azure Firewall in hub)
Spoke2 subnet ─ UDR 0.0.0.0/0 → 10.0.0.4
          └──── peering ────► Hub VNet (Azure Firewall 10.0.0.4)
```
Spoke-to-spoke traffic works **through** the firewall – that's how you get around "peering isn't transitive".

## Exam traps
- Subnets in the **same VNet** talk by default – no route needed
- Separate VNets **can't** talk by default → **peering** or a gateway
- Route tables are associated with **subnets**, not NICs or VNets
- Next hop **Virtual appliance** needs the NVA's **private IP** and **IP forwarding** on its NIC
- **Longest prefix wins**, then UDR > BGP > system
- Forcing internet traffic through a firewall → UDR `0.0.0.0/0` → Virtual appliance
- Don't put a UDR with `0.0.0.0/0` on **GatewaySubnet** or **AzureBastionSubnet** (breaks them)
- "Why does traffic go this way?" → **Effective routes** (NIC) or **Network Watcher → Next hop**

## Do it

### Portal
1. **Route tables → + Create** (same region as the VNet)
2. **Routes → + Add** → destination `0.0.0.0/0` → next hop **Virtual appliance** → `10.0.0.4`
3. **Subnets → + Associate** → VNet + subnet
4. NVA VM → **Networking → NIC → IP configurations → IP forwarding: Enabled**
5. VM NIC → **Effective routes** to verify

### Azure CLI
```bash
az network route-table create -g $RG -n rt-spoke1 -l norwayeast
az network route-table route create -g $RG --route-table-name rt-spoke1 -n default-to-fw \
  --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.0.4
az network vnet subnet update -g $RG --vnet-name vnet-spoke1 -n snet-app --route-table rt-spoke1

# IP forwarding on the NVA's NIC
az network nic update -g $RG -n nic-nva1 --ip-forwarding true

# Troubleshoot
az network nic show-effective-route-table -g $RG -n nic-app1 -o table
az network watcher show-next-hop -g $RG --vm vm-app1 --source-ip 10.1.1.4 --dest-ip 8.8.8.8
```

### PowerShell
```powershell
$rt = New-AzRouteTable -ResourceGroupName $rg -Name rt-spoke1 -Location norwayeast
Add-AzRouteConfig -RouteTable $rt -Name default-to-fw -AddressPrefix 0.0.0.0/0 `
  -NextHopType VirtualAppliance -NextHopIpAddress 10.0.0.4 | Set-AzRouteTable

$vnet = Get-AzVirtualNetwork -ResourceGroupName $rg -Name vnet-spoke1
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name snet-app -AddressPrefix 10.1.1.0/24 -RouteTable $rt
$vnet | Set-AzVirtualNetwork

Get-AzEffectiveRouteTable -ResourceGroupName $rg -NetworkInterfaceName nic-app1 | Format-Table
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| VM lost internet after adding a UDR | `0.0.0.0/0` → appliance that is down / doesn't allow it | Check firewall rules, NVA health, effective routes |
| Traffic reaches the NVA but goes no further | **IP forwarding** off on the NIC or in the OS | Enable both |
| Return traffic takes another path (asymmetric) | UDR only on one side | Add matching UDRs on both subnets |
| Route has no effect | Route table not **associated** with the subnet, or a longer prefix wins | Associate, check **effective routes** |
| Spoke1 can't reach Spoke2 | Peering isn't transitive | Route both via a hub firewall/NVA (UDR + forwarded traffic) or peer directly |
| Bastion/VPN gateway broke | UDR on its special subnet | Remove it |

**Network Watcher – Next hop** tells you the next hop type, IP, and **which route table** decided it.

## Clean up
```bash
az network vnet subnet update -g $RG --vnet-name vnet-spoke1 -n snet-app --route-table ""
az network route-table delete -g $RG -n rt-spoke1
```

## Check yourself
<details><summary>1. All traffic from the app subnet to the internet must go through Azure Firewall (10.0.0.4).</summary>

Route table with `0.0.0.0/0` → **Virtual appliance 10.0.0.4**, associated with the app subnet.
</details>

<details><summary>2. Routes: 10.0.0.0/16 → VNet, 10.0.2.0/24 → Virtual appliance. Where does traffic to 10.0.2.5 go?</summary>

To the **virtual appliance** – longest prefix (/24) wins.
</details>

<details><summary>3. Traffic arrives at the NVA VM but is not forwarded. What's missing?</summary>

**IP forwarding** on the NVA's NIC (and in the OS).
</details>

## Related
- [Firewalls](firewalls.md) – routing allows traffic, firewalls decide what is permitted
- [IP Addressing and Subnetting](ip-addressing-subnetting.md)
- [Virtual Networks and VNet Peering](vnet-peering.md)
- [NSG Rules](nsg-inbound-outbound-rules.md)
