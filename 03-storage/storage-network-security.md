# Storage Network Security

[← Back to README](../README.md)

## In one line
Control **which networks** can reach a storage account. A request must come from an **allowed network** *and* be **authorized** (key, SAS, or Entra ID) – both checks must pass.

## When to use it (and when not)
| Requirement | Answer |
|---|---|
| Only a few public IPs (an office) may connect | **Storage firewall** with IP rules |
| A subnet in Azure should reach storage, cheaply | **Service endpoint** + firewall rule |
| Access from **on-premises** or **peered VNets** over a private IP | **Private endpoint** |
| Storage must have **no public access at all** | Private endpoint + **public network access = Disabled** |
| Azure Backup / Event Grid must still reach a locked account | Allow **trusted Azure services** exception |

## Key concepts

### Public network access – three options
Portal: **Storage account → Security + networking → Networking → Public network access**

| Option | What it means |
|---|---|
| **Enabled from all networks** | Default. Anyone on the internet can reach the endpoint (but still needs a key, SAS, or Entra ID access) |
| **Enabled from selected virtual networks and IP addresses** | Storage firewall: only the subnets and IP addresses you list are allowed |
| **Disabled** | No public access at all. Only **private endpoints** work |

> **!!!** When you restrict access, also add **your own client IP address**, or you will get "This request is not authorized" when browsing data in the portal from your PC.

**Exceptions:** you can allow **trusted Azure services** (for example Azure Backup) to bypass the firewall.

### Service endpoints
Lets a **subnet** reach the storage account over the **Azure backbone network**, and lets the storage firewall recognize that subnet.

```
VM in snet-vm ──(Azure backbone)──► mystorage.blob.core.windows.net
                                     (still a public IP)
```

**How to set it up:**
1. Enable the service endpoint **`Microsoft.Storage`** on the subnet
2. Add the VNet and subnet to the storage account firewall
3. Set public network access to **Enabled from selected virtual networks and IP addresses**

The portal enables the service endpoint on the subnet automatically when you add the subnet in step 2.

- The storage account **keeps its public IP address** – traffic just takes an optimized route
- The source of the traffic is seen as the VM's **private IP** and subnet
- **Free**
- Only works for resources **in that subnet** – not from on-premises or peered VNets

### Private endpoints
Gives the storage account a **private IP address inside your VNet**. The storage account is reached as if it were a local resource.

```
VM in snet-vm ──► 10.10.3.4 (private endpoint in snet-pe) ──► storage account
```

- Built on **Azure Private Link**
- The private endpoint is a **network interface (NIC)** with a private IP from your subnet
- Works from the VNet, **peered VNets**, and **on-premises** (through VPN or ExpressRoute)
- Lets you set public network access to **Disabled**
- **Costs money** (per hour and per GB)
- **Needs DNS** configuration (see below)
- One private endpoint per **sub-resource**: `blob`, `file`, `queue`, `table`, `dfs`, `web`

### Private DNS zones
Apps still connect to the normal name, `mystorage.blob.core.windows.net`. DNS must return the **private IP** instead of the public IP.

```
mystorage.blob.core.windows.net
        │  CNAME (created by Azure)
        ▼
mystorage.privatelink.blob.core.windows.net
        │  A record in the private DNS zone
        ▼
10.10.3.4
```

**What you need:**
1. A **private DNS zone** named `privatelink.blob.core.windows.net`
2. An **A record** for the storage account pointing to the private IP (created automatically with a DNS zone group)
3. A **virtual network link** from the zone to every VNet that needs to resolve the name

| Sub-resource | Private DNS zone |
|---|---|
| Blob | `privatelink.blob.core.windows.net` |
| File | `privatelink.file.core.windows.net` |
| Queue | `privatelink.queue.core.windows.net` |
| Table | `privatelink.table.core.windows.net` |
| Data Lake (dfs) | `privatelink.dfs.core.windows.net` |

### Service endpoint vs private endpoint

| | Service endpoint | Private endpoint |
|---|---|---|
| Storage IP address | Public | **Private, in your VNet** |
| Configured on | The subnet | A NIC in a subnet |
| Access from peered VNets | No | Yes |
| Access from on-premises | No | Yes (VPN/ExpressRoute) |
| Public access can be disabled | No | Yes |
| DNS changes needed | No | Yes |
| Cost | Free | Paid |

## Exam traps
- If the question mentions **on-premises access**, **disabling public access**, or a **private IP** → **private endpoint**. If it only needs a subnet in Azure to reach storage cheaply → **service endpoint**
- Network rules are **on top of** authorization. Allowed network + no role/key = still denied
- The firewall's **IP rules take public IPs only** – you can't add private IPs (10.x, 192.168.x). Use VNet rules for Azure subnets
- A private endpoint covers **one sub-resource**. Blob + File = **two** private endpoints and **two** DNS zones
- The private DNS zone must be **linked to every VNet** that resolves the name (including hub VNets, and peered VNets that use their own DNS)
- On-premises clients need a **DNS forwarder / Private DNS Resolver** to resolve the private zone, or they get the public IP
- Service endpoint policies can restrict a subnet to **specific** storage accounts (data exfiltration protection)

## Do it

### Portal
- **Firewall:** Storage account → **Networking → Firewalls and virtual networks** → "Enabled from selected virtual networks and IP addresses" → **+ Add existing virtual network** → add **your client IP** → tick **Allow Azure services on the trusted services list** → Save
- **Private endpoint:** **Networking → Private endpoint connections → + Private endpoint** → sub-resource `blob` → VNet/subnet → **Integrate with private DNS zone = Yes**
- **Disable public access:** **Networking → Public network access → Disabled**

### Azure CLI
```bash
# Service endpoint on the subnet + firewall rule
az network vnet subnet update -g $RG --vnet-name vnet-lab --name snet-vm \
  --service-endpoints Microsoft.Storage
az storage account network-rule add -g $RG --account-name $SA \
  --vnet-name vnet-lab --subnet snet-vm

# Allow my own public IP
MYIP=$(curl -s https://api.ipify.org)
az storage account network-rule add -g $RG --account-name $SA --ip-address $MYIP

# Deny everything else, but let trusted Azure services through
az storage account update -g $RG -n $SA --default-action Deny --bypass AzureServices

# See the rules
az storage account network-rule list -g $RG --account-name $SA

# Disable public access (private endpoint only)
az storage account update -g $RG -n $SA --public-network-access Disabled
```

Full private endpoint + DNS commands: see the [private access lab](lab/storage-account-with-private-access.md), steps 5–7.

### PowerShell
```powershell
$subnet = Get-AzVirtualNetwork -ResourceGroupName $rg -Name vnet-lab |
          Get-AzVirtualNetworkSubnetConfig -Name snet-vm

Add-AzStorageAccountNetworkRule -ResourceGroupName $rg -Name $sa -VirtualNetworkResourceId $subnet.Id
Add-AzStorageAccountNetworkRule -ResourceGroupName $rg -Name $sa -IPAddressOrRange "203.0.113.10"

Update-AzStorageAccountNetworkRuleSet -ResourceGroupName $rg -Name $sa `
  -DefaultAction Deny -Bypass AzureServices

Get-AzStorageAccountNetworkRuleSet -ResourceGroupName $rg -Name $sa
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "This request is not authorized to perform this operation" (403 AuthorizationFailure) | Your network/IP isn't allowed | Add your **public** IP or the subnet. Check you didn't add a private IP |
| Portal shows containers but "access denied" on blobs from home | You're browsing from an IP not on the list | Add client IP (your home IP can change!) |
| VM in the allowed subnet is still denied | Service endpoint not enabled on that subnet, or the VM is in a **different** subnet / peered VNet | Enable `Microsoft.Storage` on the right subnet, or use a private endpoint |
| `nslookup` returns a **public** IP from the VM | Private DNS zone not **linked** to the VNet, A record missing, or VNet uses custom DNS that doesn't forward to 168.63.129.16 | Link the zone, add zone group, fix DNS forwarding |
| Works from Azure, not from on-premises | On-prem DNS can't resolve the private zone | Conditional forwarder to Azure DNS Private Resolver |
| Azure Backup / other service fails after locking down | Trusted services not allowed | `--bypass AzureServices` |
| Private endpoint works for blobs but not files | Only a `blob` private endpoint exists | Create a second private endpoint for `file` + its DNS zone |

**Test commands:**
```bash
nslookup $SA.blob.core.windows.net        # private IP from inside the VNet?
az storage blob list --account-name $SA --container-name data --auth-mode login -o table
```
```powershell
Resolve-DnsName "$sa.blob.core.windows.net"
Test-NetConnection "$sa.blob.core.windows.net" -Port 443
```

## Clean up
```bash
az storage account update -g $RG -n $SA --default-action Allow --public-network-access Enabled
az network private-endpoint delete -g $RG -n ${SA}-pe
az network private-dns zone delete -g $RG -n privatelink.blob.core.windows.net --yes
```
Private endpoints cost money per hour – delete them after the lab.

## Check yourself
<details><summary>1. On-premises servers connect through VPN and must reach storage over a private IP. Public access must be disabled.</summary>

**Private endpoint** + private DNS zone (with DNS forwarding from on-prem) + public network access **Disabled**.
</details>

<details><summary>2. You added 10.0.1.4 as an IP rule in the storage firewall but the VM is still denied. Why?</summary>

IP rules accept **public** IPs only. Add the VM's **subnet** as a VNet rule (service endpoint) instead.
</details>

<details><summary>3. A VM in a peered VNet can't reach storage that allows the hub subnet via a service endpoint. Fix?</summary>

Service endpoints don't work across peering. Add a service endpoint on the VM's own subnet, or use a **private endpoint**.
</details>

## Related
- [Lab: Storage account with private access](lab/storage-account-with-private-access.md)
- [Storage Access](storage-access.md)
- [DNS](../01-networking/dns.md)
- [Subnets and Routing](../01-networking/subnets-and-routing.md)
