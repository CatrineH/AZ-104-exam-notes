# DNS

[← Back to README](../README.md)

## In one line
**DNS** turns names into IP addresses. In Azure you use **Azure DNS** for **public** domains, **Private DNS zones** for names **inside VNets**, and VNet **DNS server settings** to decide who answers.

```
www.example.com  →  DNS lookup  →  93.184.215.14
```
**Analogy:** DNS works like a phone book. You look up a name and get the number.

## When to use it (and when not)
| Need | Use |
|---|---|
| Host `catrineh.com` and its records in Azure | **Azure DNS public zone** + delegate NS at the registrar |
| VMs find each other by name inside one VNet | **Azure-provided DNS** (default, automatic) |
| Custom names like `db01.corp.internal` across VNets | **Private DNS zone** linked to the VNets |
| VMs register their own names automatically | Private DNS zone link with **auto-registration** |
| Private endpoints (`privatelink.*`) | **Private DNS zone** per service |
| On-prem must resolve Azure private names (and the reverse) | **Azure DNS Private Resolver** or a DNS forwarder VM |
| Buy a domain | Not Azure DNS – use a registrar (or App Service Domains) |

## Key concepts
- A domain name **points to** an IP address. The name is for people, the IP address is for computers
- DNS uses **port 53** (UDP, and TCP for large answers)
- Azure's internal DNS IP is always **168.63.129.16** (virtual, reachable from every VNet)
- Azure reserves two addresses in every subnet for DNS (see [subnetting](ip-addressing-subnetting.md#subnetting-in-azure-29-example))

### Record types

| Record | Points to | Example |
|---|---|---|
| **A** | IPv4 address | `www → 20.10.5.4` |
| **AAAA** | IPv6 address | |
| **CNAME** | Another name (alias) | `www → myapp.azurewebsites.net` |
| **MX** | Mail server | `@ → mail.contoso.com` (priority 10) |
| **TXT** | Text – domain verification, SPF | `asuid.www → <id>` |
| **NS** | Name servers for the zone | Set at the registrar to delegate |
| **SOA** | Zone info (created automatically) | |
| **SRV** | Service location | `_sip._tcp` |
| **PTR** | Reverse lookup (IP → name) | |

- **CNAME can't be used at the apex** (`contoso.com` itself) → use an **A record** or an **alias record**
- **Alias record set:** an A/AAAA/CNAME that points to an **Azure resource** (public IP, Front Door, Traffic Manager) and updates automatically if its IP changes – and works at the apex
- **TTL** = how long resolvers cache the answer

### Azure DNS – public zones
1. Create a zone `catrineh.com` in Azure DNS
2. Azure gives you **4 name servers** (`ns1-0x.azure-dns.com` …)
3. At your **domain registrar**, replace the NS records with Azure's → **delegation**
4. Add records in Azure
- **Child zone delegation:** create `dev.catrineh.com` as its own zone and add **NS records** for it in the parent zone

### Private DNS zones
- Resolves names **only inside linked VNets** – not from the internet
- A zone is **global** (not tied to a region) and must be **linked** to each VNet that uses it
- **Auto-registration:** VMs in the linked VNet get A records automatically (created/removed with the VM)
- A VNet can have **only one** private zone with **auto-registration** – but can be linked to many zones for resolution
- Used by **private endpoints**: `privatelink.blob.core.windows.net`, etc. (see [Storage Network Security](../03-storage/storage-network-security.md))

### Who answers DNS for a VNet?
| VNet DNS setting | Result |
|---|---|
| **Default (Azure-provided)** | 168.63.129.16 – resolves internet names, VM names in the same VNet, and linked private zones |
| **Custom** (e.g. your domain controllers) | Your servers answer. They must **forward** to 168.63.129.16 to resolve private zones |

After changing a VNet's DNS servers, VMs pick it up after a **restart** (or DHCP renew).

## Exam traps
- Delegating a domain to Azure DNS = update **NS records at the registrar**
- **Apex** (`contoso.com`) → A record or **alias**, not CNAME
- App Service custom domain → **CNAME** (www) or **A** (apex) **+ TXT `asuid`**
- Private DNS zone resolves only in **linked** VNets. Peering does **not** share zone links
- **One** auto-registration zone per VNet
- Custom DNS servers must forward to **168.63.129.16** for private zones to work
- Changed VNet DNS → **restart VMs**
- Azure DNS public zones **don't buy domains**

## Do it

### Portal
- **DNS zones → + Create** → `catrineh.com` → copy the 4 **name servers** → set them at the registrar → **+ Record set**
- **Private DNS zones → + Create** → `corp.internal` → **Virtual network links → + Add** → tick **Enable auto registration**
- VNet → **DNS servers → Default (Azure-provided) / Custom**

### Azure CLI
```bash
# Public zone and records
az network dns zone create -g $RG -n catrineh.com
az network dns zone show -g $RG -n catrineh.com --query nameServers
az network dns record-set a add-record -g $RG -z catrineh.com -n www -a 20.10.5.4
az network dns record-set cname set-record -g $RG -z catrineh.com -n app -c myapp.azurewebsites.net
az network dns record-set txt add-record -g $RG -z catrineh.com -n asuid.www -v "<verification-id>"

# Private zone + link with auto-registration
az network private-dns zone create -g $RG -n corp.internal
az network private-dns link vnet create -g $RG -z corp.internal -n link-hub \
  --virtual-network vnet-hub --registration-enabled true
az network private-dns record-set a list -g $RG -z corp.internal -o table

# Custom DNS servers on a VNet
az network vnet update -g $RG -n vnet-hub --dns-servers 10.10.1.10 10.10.1.11
```

### PowerShell
```powershell
New-AzDnsZone -ResourceGroupName $rg -Name catrineh.com
(Get-AzDnsZone -ResourceGroupName $rg -Name catrineh.com).NameServers
New-AzDnsRecordSet -ResourceGroupName $rg -ZoneName catrineh.com -Name www -RecordType A -Ttl 3600 `
  -DnsRecords (New-AzDnsRecordConfig -Ipv4Address 20.10.5.4)

New-AzPrivateDnsZone -ResourceGroupName $rg -Name corp.internal
New-AzPrivateDnsVirtualNetworkLink -ResourceGroupName $rg -ZoneName corp.internal -Name link-hub `
  -VirtualNetworkId (Get-AzVirtualNetwork -ResourceGroupName $rg -Name vnet-hub).Id -EnableRegistration
```

### Test
```bash
nslookup www.catrineh.com ns1-01.azure-dns.com   # ask Azure's name server directly
nslookup vm-web1.corp.internal                    # from a VM in a linked VNet
dig +short myaccount.blob.core.windows.net        # private endpoint → 10.x?
```
```powershell
Resolve-DnsName www.catrineh.com -Server ns1-01.azure-dns.com
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Records exist in Azure but the internet doesn't see them | Domain **not delegated** (registrar still has old NS) | Set Azure's NS at the registrar, wait for TTL |
| Can't create CNAME at `contoso.com` | CNAME not allowed at apex | A record or **alias** record |
| VM name doesn't resolve in another VNet | Azure-provided DNS only covers the same VNet | Private DNS zone linked to both VNets |
| Private endpoint resolves to **public IP** | Private zone not linked to this VNet, missing A record, or custom DNS doesn't forward | Link zone, add zone group, forward to 168.63.129.16 |
| Can't enable auto-registration on a second zone | Only **one** registration zone per VNet | Link without registration |
| VMs still use old DNS servers | Setting applies after restart | Restart VMs / `ipconfig /renew` |
| On-prem can't resolve Azure private names | No forwarding path | **DNS Private Resolver** inbound endpoint + conditional forwarder |

## Clean up
```bash
az network private-dns link vnet delete -g $RG -z corp.internal -n link-hub --yes
az network private-dns zone delete -g $RG -n corp.internal --yes
az network dns zone delete -g $RG -n catrineh.com --yes
```
(A private zone with VNet links can't be deleted – remove the links first.)

## Check yourself
<details><summary>1. You created a public DNS zone in Azure, but nobody can resolve your records. What did you forget?</summary>

Update the **NS records at the registrar** (delegation).
</details>

<details><summary>2. VMs in VNet1 and VNet2 (peered) must resolve each other by name, automatically.</summary>

A **private DNS zone** linked to both; **auto-registration** on (one registration zone per VNet).
</details>

<details><summary>3. Your VNet uses custom DNS servers (domain controllers). Private endpoint names resolve to public IPs. Fix?</summary>

Configure the DNS servers to **forward** to **168.63.129.16** (or use Private Resolver).
</details>

## Related
- [Network Fundamentals](network-fundamentals.md)
- [Storage Network Security](../03-storage/storage-network-security.md) – private endpoint DNS
- [App Service](../05-compute/01-app-services.md) – custom domains
