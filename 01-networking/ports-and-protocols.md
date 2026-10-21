# Ports and Protocols

[← Back to README](../README.md)

## In one line
An **IP address** finds the machine, a **port** finds the app on it, and the **protocol** (TCP or UDP) sets the delivery rules – and NSG rules are written in exactly these terms.

## When to use it (and when not)
| You're writing / reading… | You need to know |
|---|---|
| An NSG or firewall rule | The **destination port** and **protocol** of the service |
| A load balancer rule or probe | Frontend port, backend port, TCP/UDP |
| A "can't connect" troubleshooting question | Is the right port open on every hop (NSG, OS firewall, app listening)? |

## Key concepts

### Ports
One server can run several apps. Each app **listens on its own port**.
- A port number can only be used by **one app at a time** on a device
- Port range: 0–65535
- **Analogy:** IP address = the building, port = the apartment. A full destination = `10.0.1.4:443`
- The **client** uses a random high **source port** (that's why NSG source port is usually `*`)

### Standard ports

| Port | Protocol / service | Used for |
|---|---|---|
| 22 | SSH (TCP) | Remote admin for Linux |
| 53 | DNS (UDP/TCP) | Name lookups |
| 80 | HTTP (TCP) | Web (unencrypted) |
| 443 | HTTPS (TCP) | Web (encrypted), Azure APIs, Bastion, File Sync |
| 445 | SMB (TCP) | **Azure Files** mount |
| 1433 | SQL Server (TCP) | Database |
| 3306 | MySQL (TCP) | Database |
| 3389 | RDP (TCP) | Remote admin for Windows |
| 5985 / 5986 | WinRM | PowerShell remoting |
| 65200–65535 | App Gateway v2 infra | Must be allowed from **GatewayManager** |

Know **22, 80, 443, 445, 3389** by heart – they appear in NSG rule questions.

### Protocols

| | TCP | UDP |
|---|---|---|
| Analogy | Registered mail | Postcard |
| Delivery confirmed | Yes | No |
| Resends lost data | Yes | No |
| Keeps order | Yes | No |
| Speed | Slower | Faster |
| Used for | Web pages, file transfer, email, RDP | Video calls, streaming, gaming, DNS |

**Rule of thumb:** TCP when accuracy matters, UDP when speed matters more than perfection. **ICMP** (ping) is neither – it has no ports.

## Exam traps
- NSG rule: **destination port** = the service's port; **source port** = `*`
- Azure Files from on-prem fails → **445 blocked** outbound
- Ping failing doesn't prove the network is broken – **ICMP** is often blocked by the OS firewall. Test the real **TCP port**
- Bastion needs only **443** to the browser – no 3389/22 open to the internet

## Do it – test a port
```powershell
Test-NetConnection 10.1.1.4 -Port 3389            # TcpTestSucceeded : True / False
Test-NetConnection mystorage.file.core.windows.net -Port 445
Get-NetTCPConnection -State Listen | Sort-Object LocalPort   # what's listening on this Windows VM?
```
```bash
nc -zv 10.1.1.4 22                                # Linux: is TCP 22 open?
curl -v http://10.1.1.4:80
sudo ss -tlnp                                     # what's listening on this Linux VM?
```
```bash
# Azure side: is the NSG letting it through?
az network watcher test-ip-flow -g $RG --vm vm1 --direction Inbound --protocol TCP \
  --local 10.1.1.4:3389 --remote 203.0.113.10:50000
az network watcher test-connectivity -g $RG --source-resource vm1 --dest-address 10.1.1.5 --dest-port 22
```

## Troubleshooting
Work from the outside in: **NSG (subnet + NIC) → route → OS firewall → app listening on the port**.

| Symptom | Likely cause | Fix |
|---|---|---|
| Connection **times out** | Something **drops** it: NSG, firewall, route | IP flow verify, effective rules/routes |
| Connection **refused** immediately | Reached the VM, but **nothing listens** on that port | Start the service / check its port |
| Ping fails, TCP works | ICMP blocked in the OS | Allow ICMP, or just test TCP |
| Works from inside the VNet, not from the internet | No public IP / inbound NSG rule / LB rule | Add what's missing, or use Bastion |

## Check yourself
<details><summary>1. Which port must be open to mount an Azure file share over SMB?</summary>

**445** (TCP).
</details>

<details><summary>2. `Test-NetConnection` returns "connection refused" quickly. Is it the NSG?</summary>

Unlikely – an NSG **drops** (timeout). "Refused" means the VM answered but nothing listens on that port.
</details>

## Related
- [Firewalls](firewalls.md)
- [NSG Rules](nsg-inbound-outbound-rules.md)
- [Network Fundamentals](network-fundamentals.md)
