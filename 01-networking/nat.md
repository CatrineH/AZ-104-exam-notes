# NAT (Network Address Translation) and NAT Gateway

[← Back to README](../README.md)

## In one line
**NAT** lets many private IPs **share one public IP** to reach the internet. In Azure, **NAT Gateway** does this for a whole subnet – **outbound only**.

## When to use it (and when not)
| Need | Use |
|---|---|
| Many private VMs need outbound internet (updates) with a **fixed** public IP | **NAT Gateway** on the subnet |
| Partner must allow-list your outbound IP | NAT Gateway with a static public IP / **prefix** |
| VMs behind a Standard load balancer have no internet | **NAT Gateway** (or LB outbound rules) |
| Internet users must reach a VM | Not NAT Gateway – **load balancer**, **public IP**, or **Azure Firewall DNAT** |
| Inspect/filter outbound traffic by domain | **Azure Firewall** |

## Key concepts

### The problem
Business-A has **50 backend servers** with private IP addresses (for example `10.0.2.5`). They can talk internally, but not to the internet. The database server needs to download an update. Giving each server a public IP costs money, must be managed, and would need 50 addresses.

### The solution
```
10.0.2.5  ─┐
10.0.2.6  ─┼──►  NAT (router)  ──►  8.34.217.91  ◄──►  Internet
10.0.2.7  ─┘      translates         one public IP
```
- Private servers stay **hidden** – the internet only sees the public address
- They can still **reach the internet** for updates
- **Outbound only:** servers can start connections out, the internet can't start connections in

### Azure NAT Gateway
- Attached to **subnets** (one NAT gateway can serve many subnets in the **same VNet**; a subnet can have only **one** NAT gateway)
- Uses **Standard** public IPs and/or a **public IP prefix** – up to **16** IPs
- **Takes priority** for outbound: once a subnet has a NAT gateway, all outbound internet traffic uses it (instead of the VM's own public IP or LB outbound rules)
- Inbound traffic to a VM's public IP or a load balancer still works – replies go back the same way
- SKUs: **Standard** (zonal or no zone) and **StandardV2** (zone-redundant across all zones, up to 100 Gbps, needs StandardV2 public IPs). You can't upgrade Standard → StandardV2 – create a new one and swap it on the subnet
- **Default outbound access** (the hidden free public IP) has been retired for new VNets – plan explicit outbound: NAT Gateway, LB outbound rules, public IP, or firewall

## Exam traps
- NAT Gateway = **outbound only**
- Associated with a **subnet**, not a VM or NIC
- Needs **Standard** (or StandardV2) public IPs – never Basic
- Zone-redundant outbound → **StandardV2** NAT Gateway
- With NAT Gateway on the subnet, **outbound** uses the NAT gateway's IP even if the VM has its own public IP
- "Fixed outbound IP for many VMs" → **NAT Gateway**
- Can't be used on **GatewaySubnet**

## Do it

### Portal
**NAT gateways → + Create** → zone → **Outbound IP:** create public IP (or prefix) → **Subnet:** choose VNet + subnets → Create.

### Azure CLI
```bash
az network public-ip create -g $RG -n pip-natgw --sku Standard
az network nat gateway create -g $RG -n natgw-backend --public-ip-addresses pip-natgw --idle-timeout 10
az network vnet subnet update -g $RG --vnet-name vnet-lab -n snet-backend --nat-gateway natgw-backend

# Test from a VM – shows the outbound public IP
az vm run-command invoke -g $RG -n vm-backend1 --command-id RunShellScript --scripts "curl -s https://api.ipify.org"
```

### PowerShell
```powershell
$pip = New-AzPublicIpAddress -ResourceGroupName $rg -Name pip-natgw -Location norwayeast -Sku Standard -AllocationMethod Static
$nat = New-AzNatGateway -ResourceGroupName $rg -Name natgw-backend -Location norwayeast -Sku Standard -PublicIpAddress $pip -IdleTimeoutInMinutes 10

$vnet = Get-AzVirtualNetwork -ResourceGroupName $rg -Name vnet-lab
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name snet-backend -AddressPrefix 10.0.2.0/24 -NatGateway $nat
$vnet | Set-AzVirtualNetwork
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| VMs still have no internet | NAT gateway not **associated** with *their* subnet, or an **NSG outbound deny** / UDR to a firewall | Associate subnet, check NSG and effective routes |
| Outbound IP isn't the NAT IP | Subnet has a **UDR** sending 0.0.0.0/0 to a firewall | Expected – the firewall's IP is used |
| Can't attach public IP | Basic SKU | **Standard** public IP |
| Internet can't reach the VM through NAT | NAT is **outbound only** | Public IP / LB / Firewall DNAT |
| Connection drops on long idle sessions | Idle timeout (default 4 min) | Raise idle timeout or use keep-alives |

## Clean up
```bash
az network vnet subnet update -g $RG --vnet-name vnet-lab -n snet-backend --nat-gateway ""
az network nat gateway delete -g $RG -n natgw-backend
az network public-ip delete -g $RG -n pip-natgw
```
NAT Gateway is billed per hour – delete it after the lab.

## Check yourself
<details><summary>1. 40 VMs in a subnet must reach a partner API that only allows one known IP. No inbound traffic.</summary>

**NAT Gateway** with one static public IP on that subnet.
</details>

<details><summary>2. Can internet users connect to a VM through the NAT gateway?</summary>

No – **outbound only**.
</details>

## Related
- [Network Fundamentals](network-fundamentals.md)
- [Subnets and Routing](subnets-and-routing.md)
- [Firewalls](firewalls.md)
