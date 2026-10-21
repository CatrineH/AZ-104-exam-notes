# IP Addressing and Subnetting

[← Back to README](../README.md)

## In one line
A VNet gets an **address space** (CIDR range), you cut it into **subnets**, and Azure **reserves 5 addresses** in every subnet – so plan sizes and avoid overlaps from day one.

## When to use it (and when not)
| Planning question | Answer |
|---|---|
| How big should a VNet be? | Leave room: /16 for a big hub, /20–/22 for spokes |
| Will this VNet ever be peered / connected to on-prem? | Then its range must **not overlap** with any of them |
| Which ranges can I use? | **Private** ranges (RFC 1918): `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` |
| Smallest subnet? | **/29** (8 addresses, 3 usable) |
| Special subnets? | See the table under Key concepts |

## Key concepts

### IP addressing
- An IPv4 address has **32 bits**, split into 4 octets of 8 bits
- Each octet is written in decimal (0–255)
- Bit positions in an octet: `128 64 32 16 8 4 2 1`

Example: `10` in binary = `00001010`

### CIDR notation
- The number after `/` = how many bits are used for the **network**
- The remaining bits are for **hosts**
- Mask bits: **1 = on (network)**, **0 = off (host)**

| CIDR | Host bits | Total addresses | Usable in Azure |
|---|---|---|---|
| /16 | 16 | 65,536 | 65,531 |
| /24 | 8 | 256 | 251 |
| /26 | 6 | 64 | 59 |
| /27 | 5 | 32 | 27 |
| /28 | 4 | 16 | 11 |
| /29 | 3 | 8 | 3 |

Formula: total addresses = 2^(host bits). **Usable in Azure = total − 5.**

### Subnetting in Azure (/29 example)
Azure reserves **5 addresses** in every subnet. The smallest subnet Azure allows is **/29**.

| Address | Purpose |
|---|---|
| 10.10.1.0 | Network address |
| 10.10.1.1 | Default gateway |
| 10.10.1.2 | Azure DNS |
| 10.10.1.3 | Azure DNS |
| 10.10.1.4 – 10.10.1.6 | Usable hosts (3) |
| 10.10.1.7 | Broadcast |

The **first VM** in a subnet gets **.4**.

### Special subnets (exact names!)

| Subnet name | For | Size |
|---|---|---|
| `GatewaySubnet` | VPN / ExpressRoute gateway | **/27** or larger recommended |
| `AzureBastionSubnet` | Azure Bastion | **/26** or larger |
| `AzureFirewallSubnet` | Azure Firewall | **/26** |
| (any name, dedicated) | Application Gateway v2 | /24 recommended |
| (any name, delegated) | App Service VNet integration, ACI, Container Apps | /26–/27 or as required |

### Public IP addresses
| SKU | Allocation | Zones | Default |
|---|---|---|---|
| **Standard** | Static | Zonal or zone-redundant | **Closed** inbound – needs an NSG allow |
| Basic | Retired (Sept 2025) | – | – |

## Exam traps
- **5 reserved** per subnet: first 4 + last. "How many VMs fit in a /28?" → **11**
- Smallest subnet: **/29**
- **Overlapping** address spaces can't be **peered** or connected with VPN
- You **can't resize a subnet** that has resources in it (unless the new range still contains them) – plan ahead
- You **can add** address spaces to a VNet later (peered VNets then need a **sync**)
- Special subnet names must be **exact**: `GatewaySubnet`, `AzureBastionSubnet`, `AzureFirewallSubnet`
- A NIC's private IP is **dynamic** by default but stays the same until the NIC is deleted. Set **static** for DNS servers / domain controllers

## Do it

### Portal
**Virtual networks → + Create → IP addresses** → address space → **+ Add a subnet** (choose a template like *Azure Bastion* to get the right name/size).
Existing VNet → **Address space** / **Subnets → + Subnet**.

### Azure CLI
```bash
az network vnet create -g $RG -n vnet-hub -l norwayeast \
  --address-prefixes 10.10.0.0/16 --subnet-name snet-web --subnet-prefixes 10.10.1.0/24

az network vnet subnet create -g $RG --vnet-name vnet-hub -n AzureBastionSubnet --address-prefixes 10.10.2.0/26
az network vnet subnet create -g $RG --vnet-name vnet-hub -n GatewaySubnet      --address-prefixes 10.10.3.0/27

# Add another address space later
az network vnet update -g $RG -n vnet-hub --address-prefixes 10.10.0.0/16 10.20.0.0/16

# Is this IP free? (shows a list of free IPs if it isn't)
az network vnet check-ip-address -g $RG -n vnet-hub --ip-address 10.10.1.4

# Static private IP on a NIC
az network nic ip-config update -g $RG --nic-name nic-dc1 -n ipconfig1 --private-ip-address 10.10.1.10
```

### PowerShell
```powershell
$web = New-AzVirtualNetworkSubnetConfig -Name snet-web -AddressPrefix 10.10.1.0/24
$bas = New-AzVirtualNetworkSubnetConfig -Name AzureBastionSubnet -AddressPrefix 10.10.2.0/26
New-AzVirtualNetwork -ResourceGroupName $rg -Name vnet-hub -Location norwayeast -AddressPrefix 10.10.0.0/16 -Subnet $web,$bas

$vnet = Get-AzVirtualNetwork -ResourceGroupName $rg -Name vnet-hub
Add-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name GatewaySubnet -AddressPrefix 10.10.3.0/27
$vnet | Set-AzVirtualNetwork

Test-AzPrivateIPAddressAvailability -VirtualNetwork $vnet -IPAddress 10.10.1.4
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Subnet … is not valid in VNet address space" | Subnet range outside the VNet range | Pick a range inside, or add an address space |
| "…overlaps with subnet…" | Two subnets overlap | Choose a free range |
| Can't peer the VNets | **Overlapping** address spaces | Re-address one VNet (hard!) – plan first |
| Can't change a subnet's size | Resources are using it | Move resources out, or add a new subnet |
| Bastion/Firewall/Gateway deploy fails | Wrong **subnet name** or too small | Exact name, right size |
| "Subnet is full" / no IPs | Remember: **−5** per subnet | Bigger subnet |
| Domain controller IP changed | Dynamic IP and NIC was recreated | Static private IP |

## Clean up
Delete lab VNets with their RG. Subnets with resources can't be deleted – remove the resources first.

## Check yourself
<details><summary>1. How many VMs can you deploy in a /27 subnet?</summary>

32 − 5 = **27**.
</details>

<details><summary>2. What IP will the first VM in 10.0.5.0/24 get?</summary>

**10.0.5.4**.
</details>

<details><summary>3. You need Azure Bastion. What subnet do you create?</summary>

**AzureBastionSubnet**, **/26** or larger.
</details>

## Related
- [Naming Conventions](naming-conventions.md)
- [Subnets and Routing](subnets-and-routing.md)
- [DNS](dns.md)
