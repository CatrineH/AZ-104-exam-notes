# Virtual Networks and VNet Peering

[← Back to README](../README.md)

## In one line
A **VNet** is your private network in Azure (one region, one subscription). **Peering** connects two VNets over the Azure backbone so they talk with **private IPs** – it must be created in **both directions**, and it's **not transitive**.

## When to use it (and when not)
| Need | Use |
|---|---|
| Connect two VNets, fastest and simplest | **VNet peering** (same region) / **global peering** (different regions) |
| Many spokes share one VPN gateway to on-prem | Hub-and-spoke: **gateway transit** + **use remote gateways** |
| Spoke-to-spoke traffic | Hub with firewall/NVA + UDRs, or direct peering, or **Virtual WAN** |
| Encrypted tunnel between VNets | **VNet-to-VNet VPN** (slower, costs gateways) |
| Overlapping address spaces | Peering **isn't possible** – re-address |

## Key concepts

### Virtual network (VNet)
- Resources in the **same VNet** can communicate with each other by default
- A VNet is divided into **subnets** for segmentation and security
- A VNet belongs to **one region** and **one subscription**
- Separate VNets **cannot** communicate by default

```
Azure
└── VNet
    ├── Frontend subnet
    ├── Backend subnet
    └── Database subnet
```

### VNet peering
**Peering** connects two VNets privately. **Analogy:** a private highway between two networks.
```
Before:   VNet A   ✕   VNet B
After:    VNet A ◄────► VNet B
```
- **Low latency** – traffic uses the **Azure backbone network**, never the public internet
- **No VPN gateway** required
- Works across subscriptions and Microsoft Entra tenants

| Type | Connects |
|---|---|
| **VNet peering** | VNets in the **same region** |
| **Global VNet peering** | VNets in **different regions** |

### Peering settings

| Setting | What it does |
|---|---|
| **Allow access to remote VNet** | Allows traffic between the VNets (on by default) |
| **Allow forwarded traffic** | Accepts traffic that didn't start in the peered VNet, for example from an NVA |
| **Allow gateway transit** | Lets the peered VNet use **this** VNet's VPN/ExpressRoute gateway |
| **Use remote gateways** | Uses the **peered** VNet's gateway instead of having its own |

**Hub-and-spoke:** one hub VNet with a VPN gateway is peered to many spoke VNets. The hub enables **gateway transit**, and the spokes enable **use remote gateways**. Then all spokes can reach on-premises through one gateway.

### Peering vs VPN gateway

| | VNet peering | VNet-to-VNet VPN |
|---|---|---|
| Traffic path | Azure backbone | Encrypted tunnel through a VPN gateway |
| Speed | Highest, lowest latency | Limited by the gateway SKU |
| Setup | Simple | Needs a gateway in each VNet (20–45 min each) |
| Cost | Data transfer only | Gateway hours + data transfer |

## Exam traps
- **Peering must be created in both directions.** Status goes from *Initiated* to **Connected** when both exist. The portal creates both; CLI/PowerShell don't
- **Address spaces cannot overlap**
- **Peering is not transitive.** A ↔ B and B ↔ C does **not** mean A ↔ C
- **Data transfer costs money** – inbound and outbound over the peering
- **NSGs still apply.** `AllowVnetInBound` includes peered VNets, so traffic is allowed unless you deny it
- A spoke can **use remote gateways** only if it has **no gateway of its own** and the hub has **allow gateway transit**
- Adding an address space to a peered VNet → **sync** the peering
- Private DNS zone links and service endpoints are **not** shared over peering
- A VM can't move to another VNet – recreate it ([VMs](../05-compute/03-vms-plan-deployment.md))

## Do it

### Portal
VNet A → **Peerings → + Add** → name for *this* side and the *remote* side → remote VNet → settings (forwarded traffic, gateway transit) → Add. The portal creates **both** links. Check **Peering status = Connected**.

### Azure CLI
```bash
VNET_A_ID=$(az network vnet show -g $RG -n vnet-prod --query id -o tsv)
VNET_B_ID=$(az network vnet show -g $RG -n vnet-dev  --query id -o tsv)

az network vnet peering create -g $RG -n prod-to-dev --vnet-name vnet-prod \
  --remote-vnet $VNET_B_ID --allow-vnet-access
az network vnet peering create -g $RG -n dev-to-prod --vnet-name vnet-dev \
  --remote-vnet $VNET_A_ID --allow-vnet-access

az network vnet peering list -g $RG --vnet-name vnet-prod \
  --query "[].{name:name, state:peeringState, sync:peeringSyncLevel}" -o table

# Hub-and-spoke gateway sharing
az network vnet peering update -g $RG -n hub-to-spoke --vnet-name vnet-hub --set allowGatewayTransit=true
az network vnet peering update -g $RG -n spoke-to-hub --vnet-name vnet-spoke --set useRemoteGateways=true

# After adding an address space
az network vnet peering sync -g $RG -n prod-to-dev --vnet-name vnet-prod
```

### PowerShell
See the full [VNet Peering lab](labs/vnet-peering-lab.md). Short version:
```powershell
$a = Get-AzVirtualNetwork -ResourceGroupName $rg -Name vnet-prod
$b = Get-AzVirtualNetwork -ResourceGroupName $rg -Name vnet-dev
Add-AzVirtualNetworkPeering -Name prod-to-dev -VirtualNetwork $a -RemoteVirtualNetworkId $b.Id
Add-AzVirtualNetworkPeering -Name dev-to-prod -VirtualNetwork $b -RemoteVirtualNetworkId $a.Id
Get-AzVirtualNetworkPeering -ResourceGroupName $rg -VirtualNetworkName vnet-prod | Select-Object Name, PeeringState
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Peering stuck at **Initiated** | Only one direction created | Create the reverse peering |
| Peering **Disconnected** | The other side was deleted | Delete and recreate both sides |
| "Address space overlaps" | Same/overlapping ranges | Re-address one VNet |
| Peered, but VMs can't talk | NSG deny, OS firewall (ICMP!), wrong IP | IP flow verify, `Test-NetConnection -Port` |
| A can't reach C through B | **Not transitive** | Peer A–C, or route via hub NVA/firewall |
| Spoke can't reach on-prem through hub gateway | Gateway transit / use remote gateways not set, or spoke has its own gateway | Set both flags; remove spoke gateway |
| New address range not reachable | Peering not **synced** | `az network vnet peering sync` |
| Name resolution fails across peering | Private DNS zone not linked to both VNets | Link the zone |

## Clean up
```bash
az network vnet peering delete -g $RG -n prod-to-dev --vnet-name vnet-prod
az network vnet peering delete -g $RG -n dev-to-prod --vnet-name vnet-dev
```

## Check yourself
<details><summary>1. VNet A is peered with B, B with C. VM in A can't reach VM in C. Why?</summary>

Peering is **not transitive**.
</details>

<details><summary>2. You ran one `az network vnet peering create`. Status is Initiated. Why?</summary>

The **other direction** is missing.
</details>

<details><summary>3. Ten spokes must reach on-prem using the hub's VPN gateway. Which settings?</summary>

Hub → spoke: **Allow gateway transit**. Spoke → hub: **Use remote gateways**.
</details>

## Related
- [Lab: VNet Peering with PowerShell](labs/vnet-peering-lab.md)
- [Subnets and Routing](subnets-and-routing.md)
- [ExpressRoute, VPN, and Virtual WAN](expressroute-and-virtual-wan.md)
