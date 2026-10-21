# Firewalls

[← Back to README](../README.md)

## In one line
Firewalls **decide what traffic is allowed**. Routing makes it *possible* for networks to talk; firewalls decide what is *permitted*. In Azure: **NSGs** (free, per subnet/NIC), **Azure Firewall** (central, FQDN filtering), **WAF** (web attacks).

## When to use it (and when not)
| Need | Use |
|---|---|
| Allow/deny by IP, port, protocol per subnet or VM | **NSG** ([NSG rules](nsg-inbound-outbound-rules.md)) |
| Central filtering for many VNets, allow outbound only to certain **domains** | **Azure Firewall** in the hub |
| Publish an internal server to the internet on a port | **Azure Firewall DNAT** (or load balancer) |
| Protect web apps from SQL injection / XSS (OWASP) | **WAF** on Application Gateway / Front Door |
| Simple, free, stateful filtering | NSG – don't buy a firewall for that alone |

## Key concepts

### Types by location
| Type | Where | Example rule |
|---|---|---|
| **Host firewall** | On an individual server (Windows Firewall, ufw) | Only accept 3306 from the frontend subnet |
| **Network firewall** | Between network segments | Allow 443 in, block everything else |

### Types by how they inspect traffic
| Type | Analogy | How it works |
|---|---|---|
| **Stateless** | "Do you have the key?" | Checks **each packet on its own**. Return traffic needs its own rule |
| **Stateful** | "Where are you going?" | **Tracks connections.** Replies are allowed automatically |
| **Application firewall** | "Where are you going, and what are you doing there?" | Inspects the **content** (layer 7): website name, URL, HTTP request |

| | Stateless | Stateful |
|---|---|---|
| Remembers connections | No | Yes |
| Return traffic | Needs its own rule | Allowed automatically |
| Examples | Simple router ACLs | NSGs, Azure Firewall |

### In Azure
| Azure service | Type | Notes |
|---|---|---|
| **Network Security Group (NSG)** | Stateful, basic network firewall | IP, port, protocol. On a subnet or NIC. Free |
| **Azure Firewall** | Stateful, managed, central firewall | IP, port, **and domain name**. Usually in a hub VNet |
| **Web Application Firewall (WAF)** | Application firewall | OWASP rules. Runs on Application Gateway or Front Door |

### Azure Firewall
- Needs its own subnet **`AzureFirewallSubnet`**, at least **/26**
- Needs a **Standard public IP**
- **Denies all traffic by default** – you allow what you need
- Traffic is sent to it with **UDRs** (`0.0.0.0/0` → Virtual appliance → firewall's **private IP**)
- SKUs: **Basic** (small business), **Standard** (FQDN filtering, threat intel), **Premium** (TLS inspection, IDPS)
- Rules managed through **Firewall Policy** (recommended) or classic rules

| Rule type | Filters on | Example |
|---|---|---|
| **DNAT rules** | Incoming traffic from the internet, translated to a private IP | Forward port 3389 on the public IP to a VM |
| **Network rules** | IP address, port, and protocol | Allow TCP 1433 from 10.0.1.0/24 to 10.0.3.4 |
| **Application rules** | Domain names (FQDNs) | Allow `vy.no`, deny `test.com` |

- Processing order: **DNAT → Network → Application**
- Inside a type: rule **collection groups/collections** by **priority**

## Exam traps
- NSGs are **stateful**. Allow inbound 443 → no outbound rule for the reply
- "Allow outbound only to `*.microsoft.com`" → **Azure Firewall application rule** (NSGs can't filter by FQDN)
- Azure Firewall: subnet **AzureFirewallSubnet /26**, traffic sent there with a **UDR**
- Rule order: **DNAT → Network → Application**
- Azure Firewall **denies by default**; NSG denies **inbound** internet by default but **allows outbound**
- WAF = layer 7 attacks, needs **App Gateway WAF_v2** or **Front Door**

## Do it

### Portal
1. Create `AzureFirewallSubnet` (/26) in the hub VNet
2. **Firewalls → + Create** → SKU Standard → **Firewall policy: Add new** → VNet: hub → public IP: new
3. **Firewall Policy → Application rules → + Add a rule collection** → Allow → source `10.1.0.0/16` → protocol `https` → target FQDN `*.microsoft.com`
4. **Route table** on the spoke subnet: `0.0.0.0/0` → Virtual appliance → firewall private IP
5. Test from a spoke VM: `curl https://www.microsoft.com` works, `curl https://example.com` is denied

### Azure CLI
```bash
az extension add --name azure-firewall

az network vnet subnet create -g $RG --vnet-name vnet-hub -n AzureFirewallSubnet --address-prefixes 10.0.0.0/26
az network public-ip create -g $RG -n pip-fw --sku Standard
az network firewall policy create -g $RG -n fwpol-hub
az network firewall create -g $RG -n fw-hub --sku AZFW_VNet --tier Standard --firewall-policy fwpol-hub
az network firewall ip-config create -g $RG -f fw-hub -n fw-ipconfig --public-ip-address pip-fw --vnet-name vnet-hub
FW_IP=$(az network firewall show -g $RG -n fw-hub --query "ipConfigurations[0].privateIPAddress" -o tsv)

az network firewall policy rule-collection-group create -g $RG --policy-name fwpol-hub -n rcg-app --priority 300
az network firewall policy rule-collection-group collection add-filter-collection -g $RG \
  --policy-name fwpol-hub --rcg-name rcg-app -n allow-microsoft --collection-priority 100 \
  --action Allow --rule-type ApplicationRule --rule-name ms --source-addresses 10.1.0.0/16 \
  --protocols Https=443 --target-fqdns "*.microsoft.com"
```

### PowerShell
```powershell
$fw = Get-AzFirewall -ResourceGroupName $rg -Name fw-hub
$fw.IpConfigurations[0].PrivateIPAddress
$pol = Get-AzFirewallPolicy -ResourceGroupName $rg -Name fwpol-hub
Get-AzFirewallPolicyRuleCollectionGroup -AzureFirewallPolicyName fwpol-hub -ResourceGroupName $rg -Name rcg-app
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Spoke VM can't reach anything after adding the firewall | No **allow** rules (firewall denies by default) | Add network/application rules |
| Traffic bypasses the firewall | **UDR** missing or not associated with the subnet | Route table → Virtual appliance → firewall private IP |
| Firewall deploy fails | Subnet not named **AzureFirewallSubnet** or too small | Exact name, /26 |
| DNAT works but reply fails | Asymmetric routing (UDR on the target subnet sends replies elsewhere) | Fix routes |
| Which rule blocked it? | | Firewall **logs** (Diagnostic settings → Log Analytics), `AZFWApplicationRule` / `AZFWNetworkRule` tables |
| Blocked by the OS, not Azure | Windows Firewall / ufw | Check the host firewall too |

## Clean up
Azure Firewall costs **per hour** (a lot for a lab) – delete it right after testing:
```bash
az network firewall delete -g $RG -n fw-hub
az network firewall policy delete -g $RG -n fwpol-hub
az network public-ip delete -g $RG -n pip-fw
```

## Check yourself
<details><summary>1. Servers may only reach `*.windowsupdate.com` on the internet. NSG or Azure Firewall?</summary>

**Azure Firewall** application rule – NSGs can't filter on domain names.
</details>

<details><summary>2. In which order does Azure Firewall process rule types?</summary>

**DNAT → Network → Application**.
</details>

<details><summary>3. You deployed the firewall but spoke traffic doesn't go through it.</summary>

Missing **UDR** `0.0.0.0/0` → Virtual appliance (firewall private IP) on the spoke subnet.
</details>

## Related
- [NSG Rules](nsg-inbound-outbound-rules.md)
- [Ports and Protocols](ports-and-protocols.md)
- [Subnets and Routing](subnets-and-routing.md)
- [Application Gateway](application-gateway.md) – WAF
