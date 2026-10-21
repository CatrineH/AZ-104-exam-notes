# Hybrid Connectivity: VPN Gateway, ExpressRoute, and Virtual WAN

[← Back to README](../README.md)

## In one line
**VPN Gateway** connects on-prem or users to Azure through an **encrypted tunnel over the internet**, **ExpressRoute** is a **private dedicated line** into Microsoft, and **Virtual WAN** is a **Microsoft-managed global hub** that ties all of it together.

- **ExpressRoute** = *how* traffic gets to Azure
- **Virtual WAN** = *how* traffic moves once it is connected

They are **complementary, not competing**. Many enterprise designs use ExpressRoute as the connectivity layer and Virtual WAN as the global transit and routing layer.

## When to use it (and when not)
| Need | Use |
|---|---|
| Connect an office network to Azure over the internet, encrypted | **Site-to-site (S2S) VPN** |
| Individual laptops connect to the VNet | **Point-to-site (P2S) VPN** |
| Private, predictable, high-bandwidth connection, no internet | **ExpressRoute** |
| Backup path if ExpressRoute fails | **S2S VPN** as failover |
| Many branches + many regions + central routing | **Virtual WAN** |
| Just connect two Azure VNets | **VNet peering** (not a gateway) |

## Key concepts

### Azure VPN Gateway
A **virtual network gateway** of type **VPN** sends encrypted (IPsec/IKE) traffic over the internet.

| Connection | Connects | Needs |
|---|---|---|
| **Site-to-site (S2S)** | On-prem network ↔ VNet | On-prem **VPN device** with a public IP, a **local network gateway** in Azure |
| **Point-to-site (P2S)** | One computer ↔ VNet | VPN client + authentication (certificate, Entra ID, RADIUS) |
| **VNet-to-VNet** | VNet ↔ VNet | A gateway in each VNet |

| Building block | What it is |
|---|---|
| **GatewaySubnet** | Required subnet with that **exact name**, /27 or larger recommended |
| **Virtual network gateway** | The Azure side of the VPN (takes **20–45 minutes** to deploy) |
| **Local network gateway** | Azure's description of **your on-prem side**: its public IP + on-prem address ranges |
| **Connection** | Links the two gateways with a **shared key** (PSK) |

- **Route-based** VPN type is the default and needed for P2S, VNet-to-VNet, and most scenarios. **Policy-based** is legacy (one S2S tunnel)
- **SKUs:** VpnGw1–5 (AZ versions for zone redundancy). **Basic** is legacy – limited features
- **Active-active** mode: two instances for higher availability
- One VPN gateway per VNet

---

### Azure ExpressRoute

**Analogy:** a private highway into Microsoft.

#### Key facts
- **Private, dedicated connection** between your datacenter and the Microsoft network
- With **private peering**, traffic does **not** go over the public internet
- **Predictable performance**, lower jitter (latency variation), and higher reliability than internet connections
- Mainly used for **hybrid connectivity** between on-premises and Azure
- Traffic is private but **not encrypted by default** – add IPsec VPN over ExpressRoute or MACsec (ExpressRoute Direct) if encryption is required

#### Peering types
| Peering | Connects to |
|---|---|
| **Private peering** | Your Azure VNets (VMs and other resources with private IPs) |
| **Microsoft peering** | Microsoft public services, such as Microsoft 365 and Azure PaaS public endpoints |

#### Typical use cases
- Large enterprise datacenters
- Migrating workloads to Azure
- High-bandwidth workloads, such as SAP, databases, or VDI
- Regulatory or compliance requirements for private connectivity

```
Head Office
     |
ExpressRoute
     |
Azure Region A
```

You get a private connection to Azure, but you still design the routing between multiple regions and networks yourself.

#### Connection models

**1. Service provider model** – you connect through a connectivity partner:

| Model | How it works |
|---|---|
| **CloudExchange colocation** | Your equipment is colocated at a cloud exchange facility, and you connect to Microsoft through the exchange provider |
| **Point-to-point Ethernet** | A dedicated Ethernet link from your site directly to Microsoft |
| **Any-to-any (IPVPN)** | Azure joins your company's existing WAN (MPLS network), like another branch office |

**2. ExpressRoute Direct** – you connect **directly** to Microsoft's network at a peering location, without a provider in between.
- Very high bandwidth (10 Gbps or 100 Gbps ports)
- Supports MACsec encryption
- For very large organizations with large data volumes

---

### Azure Virtual WAN

**Analogy:** a global, managed transit network.

#### Key facts
- **Virtual hubs** – central, Microsoft-managed hubs in Azure regions
- Connects VNets, branch offices, site-to-site VPNs, ExpressRoute, remote users (point-to-site VPN), and security services
- Creates a **hub-and-spoke architecture** without building and managing your own transit network
- Simplifies routing and connectivity management

#### SKUs
| SKU | Supports |
|---|---|
| **Basic** | Site-to-site VPN only |
| **Standard** | Everything: ExpressRoute, point-to-site VPN, VNet-to-VNet transit, hub-to-hub |

#### Typical use cases
- Global enterprise networks
- Multi-region Azure deployments
- Connecting many branch offices
- Central security with Azure Firewall or partner NVAs (network virtual appliances)
- Large-scale VPN and SD-WAN deployments

```
Branch A ─────┐
Branch B ─────┼── Virtual WAN Hub ── Azure VNets
Remote users ─┘
```

You get central connectivity and routing, while the connection to Azure can use VPN or internet-based transport.

---

### Using them together

An ExpressRoute circuit connects directly to a Virtual WAN hub through an **ExpressRoute gateway** in the hub. This is a very common enterprise design.

```
     Datacenter
         |
    ExpressRoute
         |
  Virtual WAN Hub ─── Other regions
    /    |    \
 VNet1 VNet2 VNet3
```

#### Example 1: Global enterprise
- Datacenter in Oslo
- Azure workloads in Norway East, West Europe, and UK South
- 50 branch offices

```
Branches → SD-WAN/VPN → Virtual WAN ← ExpressRoute ← Datacenter
```

**Benefits:** private connection to the datacenter, automatic transit routing, and simpler management than dozens of separate gateways.

#### Example 2: SAP migration
Requirements: low latency, private connectivity, and disaster recovery across regions.

```
On-prem SAP users
        |
   ExpressRoute
        |
   Virtual WAN
     /      \
 Prod        DR
 region      region
```

**Benefits:** ExpressRoute gives dedicated transport, and Virtual WAN gives inter-region connectivity and central routing.

#### Example 3: Retail
- 500 stores using SD-WAN
- Applications hosted in Azure

```
Stores → SD-WAN → Virtual WAN Hub ← ExpressRoute ← Headquarters
```

**Benefits:** stores reach Azure apps through Virtual WAN, headquarters connects privately through ExpressRoute, and routing rules are central.

---

### When to choose which

| Choose | When |
|---|---|
| **ExpressRoute** | You need private connectivity, predictable performance, or compliance rules discourage using the internet |
| **Virtual WAN** | You need a global network, have many locations, VNets, or regions, and want central routing management |
| **Both** | Medium-to-large enterprise with on-prem datacenters, multiple Azure regions, and a need for cloud-scale hub-and-spoke |

## Exam traps
- **GatewaySubnet** – exact name, /27 recommended. No NSG/UDR on it (in general)
- **Local network gateway** = the **on-prem** side (public IP of the VPN device + on-prem prefixes)
- P2S → **route-based** gateway, not Basic (for modern auth such as Entra ID / IKEv2 / OpenVPN)
- VPN gateway deploy = **20–45 min** – start it first in labs
- Changing on-prem address ranges → update the **local network gateway**
- ExpressRoute is **private, not encrypted** by default
- Spokes using the hub's gateway → peering **gateway transit / use remote gateways** ([VNet peering](vnet-peering.md))
- One VPN gateway **per VNet**

## Do it

### Portal (S2S VPN)
1. VNet → **Subnets → + Gateway subnet** (`GatewaySubnet`, /27)
2. **Virtual network gateways → + Create** → VPN, Route-based, VpnGw1(AZ) → new public IP → wait 20–45 min
3. **Local network gateways → + Create** → on-prem public IP + address spaces (e.g. `192.168.0.0/16`)
4. Virtual network gateway → **Connections → + Add** → Site-to-site (IPsec) → local network gateway → **shared key**
5. Configure the on-prem device (download config from the connection) → Status **Connected**

### Azure CLI
```bash
az network vnet subnet create -g $RG --vnet-name vnet-hub -n GatewaySubnet --address-prefixes 10.10.255.0/27
az network public-ip create -g $RG -n pip-vgw --sku Standard

az network vnet-gateway create -g $RG -n vgw-hub --vnet vnet-hub --public-ip-addresses pip-vgw   --gateway-type Vpn --vpn-type RouteBased --sku VpnGw1 --no-wait

az network local-gateway create -g $RG -n lgw-oslo --gateway-ip-address 203.0.113.10   --local-address-prefixes 192.168.0.0/16

az network vpn-connection create -g $RG -n conn-oslo --vnet-gateway1 vgw-hub   --local-gateway2 lgw-oslo --shared-key 'S3cretKey!'

az network vpn-connection show -g $RG -n conn-oslo --query connectionStatus
```

### PowerShell
```powershell
$vgw = Get-AzVirtualNetworkGateway -ResourceGroupName $rg -Name vgw-hub
$lgw = New-AzLocalNetworkGateway -ResourceGroupName $rg -Name lgw-oslo -Location norwayeast `
         -GatewayIpAddress 203.0.113.10 -AddressPrefix 192.168.0.0/16
New-AzVirtualNetworkGatewayConnection -ResourceGroupName $rg -Name conn-oslo -Location norwayeast `
  -VirtualNetworkGateway1 $vgw -LocalNetworkGateway2 $lgw -ConnectionType IPsec -SharedKey 'S3cretKey!'
Get-AzVirtualNetworkGatewayConnection -ResourceGroupName $rg -Name conn-oslo | Select-Object ConnectionStatus
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Can't create the gateway | No subnet named **GatewaySubnet** | Create it (exact name) |
| Connection **Not connected** | **Shared key** mismatch, wrong on-prem public IP, IKE/IPsec settings differ | Match the key and settings on both sides |
| Connected but no traffic to some on-prem networks | **Local network gateway** missing those prefixes | Add address prefixes |
| Spokes can't reach on-prem | Peering not set with gateway transit / use remote gateways | Set both flags |
| P2S option greyed out | **Basic** SKU or policy-based gateway | Route-based, VpnGw1+ |
| Overlapping on-prem and VNet ranges | Address conflict | Re-address, or NAT rules on the gateway |
| Tool to diagnose | | **VPN troubleshoot** in Network Watcher, connection **Diagnose and solve problems** |

## Clean up
VPN gateways cost per hour – delete when done:
```bash
az network vpn-connection delete -g $RG -n conn-oslo
az network vnet-gateway delete -g $RG -n vgw-hub
az network local-gateway delete -g $RG -n lgw-oslo
```

## Check yourself
<details><summary>1. In a S2S VPN, which Azure resource holds the on-prem VPN device's public IP and on-prem address ranges?</summary>

The **local network gateway**.
</details>

<details><summary>2. A company needs a private connection that never goes over the internet with predictable latency.</summary>

**ExpressRoute** (private peering).
</details>

<details><summary>3. 20 laptops of remote admins must reach a VNet.</summary>

**Point-to-site VPN** on a route-based gateway.
</details>

## Related
- [Virtual Networks and VNet Peering](vnet-peering.md)
- [Subnets and Routing](subnets-and-routing.md)
- [Firewalls](firewalls.md)
- [Network Fundamentals](network-fundamentals.md)
