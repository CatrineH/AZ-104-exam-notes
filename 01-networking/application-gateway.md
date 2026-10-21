# Application Gateway

[← Back to README](../README.md)

## In one line
**Azure Application Gateway** is a **layer 7 load balancer** for web traffic (HTTP/HTTPS) that can route on **URL path** or **host name**, end TLS, and add a **WAF**.

## When to use it (and when not)
| Need | Use |
|---|---|
| `/api/*` → API servers, `/images/*` → image servers | **Application Gateway** (path-based routing) |
| Several websites on one IP (`a.contoso.com`, `b.contoso.com`) | **Application Gateway** (multi-site listeners) |
| Protect web apps against OWASP attacks | **Application Gateway WAF_v2** |
| Balance any TCP/UDP traffic (RDP, SQL, DNS) | **Azure Load Balancer** (layer 4) |
| Global web traffic across regions, CDN | **Azure Front Door** |
| DNS-based routing between regions | **Traffic Manager** |

## Key concepts
- Needs its **own dedicated subnet** – no other resources in it
- Microsoft recommends a **/24** subnet for v2, so it has room to scale
- Use the **v2 SKU** (Standard_v2 or WAF_v2). v1 is retired
- Can have a **public frontend IP, a private frontend IP, or both**
- Supports **autoscaling** and **zone redundancy** (v2)

### Load Balancer vs Application Gateway

| | Azure Load Balancer | Application Gateway |
|---|---|---|
| OSI layer | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| Routes on | IP and port | URL path, host name |
| TLS termination | No | Yes |
| WAF | No | Yes (WAF_v2) |
| Dedicated subnet | No | Yes |

### Main components

| Component | What it does |
|---|---|
| **Frontend IP** | The address clients connect to (public, private, or both) |
| **Listener** | Listens on a port and protocol, for example HTTP 80. **Basic** (one site) or **Multi-site** (by host name) |
| **Routing rule** | Connects a listener to backends. **Basic** or **path-based** |
| **URL path map** | Sends paths to different backends, for example `/api/*` → pool 1, `/images/*` → pool 2 |
| **Backend pool** | The servers that receive traffic (VMs, scale sets, IP addresses, App Services) |
| **Backend settings** | Port, protocol, cookie affinity, timeout, probe used to reach the backend |
| **Health probe** | Checks if backends are healthy. Only healthy backends get traffic |

- **App Gateway** decides *which backend* receives the request
- **The web server** on the backend (for example nginx) decides *how it responds*

### Health probes
- The default probe checks **`/`** on the backend and expects **200–399**
- If the backend doesn't answer on `/`, create a **custom probe**, for example `/health`

### NSG requirements

| Where | Rule |
|---|---|
| **App Gateway subnet** (if it has an NSG) | Allow inbound TCP **65200–65535** from **GatewayManager** (v2), allow **AzureLoadBalancer**, allow the listener ports from clients |
| **Backend VMs** | Allow inbound on the backend port (for example 80) from the App Gateway subnet |

## Exam traps
- Layer 7 / URL path / host name / WAF → **Application Gateway**. Plain TCP/UDP → **Load Balancer**
- **Dedicated subnet** – you can't put VMs in it
- v2 NSG must allow **65200–65535 from GatewayManager**
- **502 Bad Gateway** = App Gateway can't get a healthy answer from the backend
- Path rules need a **default** backend for paths that don't match
- Public + private frontend at the same time → **v2**

## Do it

### Portal
See the step-by-step [Application Gateway lab](labs/application-gateway-lab.md). In short: **Application gateways → + Create** → Standard V2, dedicated subnet → **Frontends** → **Backends** (pools) → **Configuration: + Add a routing rule** → listener + backend targets + **path-based** rules → Create. Then **Backend health**.

### Azure CLI
```bash
az network public-ip create -g $RG -n pip-appgw --sku Standard
az network application-gateway create -g $RG -n appgw-lab --sku Standard_v2 --capacity 2 \
  --vnet-name vnet-appgw-cat --subnet snet-appgw --public-ip-address pip-appgw \
  --http-settings-port 80 --http-settings-protocol Http --frontend-port 80 \
  --servers 10.10.1.4 10.10.1.5 --priority 100

# Custom probe on /health
az network application-gateway probe create -g $RG --gateway-name appgw-lab -n probe-health \
  --protocol Http --host-name-from-http-settings true --path /health
az network application-gateway http-settings update -g $RG --gateway-name appgw-lab \
  -n appGatewayBackendHttpSettings --probe probe-health --host-name-from-backend-pool true

az network application-gateway show-backend-health -g $RG -n appgw-lab -o table
```

### PowerShell
```powershell
Get-AzApplicationGatewayBackendHealth -ResourceGroupName $rg -Name appgw-lab
$gw = Get-AzApplicationGateway -ResourceGroupName $rg -Name appgw-lab
Stop-AzApplicationGateway -ApplicationGateway $gw     # stop billing between lab sessions
Start-AzApplicationGateway -ApplicationGateway $gw
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| **502 Bad Gateway** | All backends **unhealthy** | Check **Backend health** |
| Backend unhealthy | Probe path returns 404 (e.g. no `/` page), NSG blocks the gateway, wrong port, web server down | Custom probe (`/health`), allow from App GW subnet, `curl localhost` on the VM |
| Deployment fails: subnet in use | Other resources in the App GW subnet | Dedicated subnet |
| Deployment/updates fail with v2 | NSG blocks **65200–65535** from GatewayManager | Allow it |
| Wrong backend answers a path | Path map order / pattern (`/api` vs `/api/*`) | Fix path patterns, check default pool |
| 403 from App Gateway | **WAF** blocked the request | WAF logs, exclusions / Detection mode while testing |

## Clean up
App Gateway v2 is billed per hour – **stop** it between sessions, delete when done:
```bash
az network application-gateway stop -g $RG -n appgw-lab
az network application-gateway delete -g $RG -n appgw-lab
```

## Check yourself
<details><summary>1. `/video/*` must go to one pool, everything else to another. Load Balancer or App Gateway?</summary>

**Application Gateway** with a **path-based** rule.
</details>

<details><summary>2. Users get 502. What do you check first?</summary>

**Backend health** – probe failing, NSG, or backend down.
</details>

<details><summary>3. Backends have no page on `/`. Default probe?</summary>

Fails (404) → create a **custom probe** on a path that returns 200.
</details>

## Related
- [Application Gateway lab](labs/application-gateway-lab.md)
- [Firewalls](firewalls.md) – WAF
- [Availability](../05-compute/05-availability.md) – Azure Load Balancer
- [Subnets and Routing](subnets-and-routing.md)
