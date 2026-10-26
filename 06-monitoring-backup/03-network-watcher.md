# Azure Network Watcher

## What it is
A **regional** service with tools to monitor, diagnose and view logs for IaaS networking (VMs, VNets, NSGs, gateways, load balancers). It's **enabled automatically** per region when you create a VNet in that region — the instance lives in a resource group called `NetworkWatcherRG`.

> Network Watcher is for *IaaS network* troubleshooting. It does not monitor PaaS app performance (that's Application Insights).

## The tools — "which tool for which question?"

| Question | Tool |
|---|---|
| Is traffic from VM A to IP X on port Y **allowed or denied**, and by which NSG rule? | **IP flow verify** |
| What is the **next hop** for a packet from this VM to that IP? (Is a UDR sending it to a firewall?) | **Next hop** |
| What are the **effective security rules** on this NIC (all NSGs combined)? | **Effective security rules** (NSG diagnostics) |
| Can VM A **connect** to VM B / a URL / an IP on port 443, and where does it fail? Latency? | **Connection troubleshoot** |
| Continuously monitor connectivity and latency between endpoints over time | **Connection monitor** |
| Visual **diagram** of resources in a VNet and how they connect | **Topology** |
| Capture packets on a VM for deep analysis (Wireshark) | **Packet capture** |
| Why is my **VPN gateway / connection** down? | **VPN troubleshoot** |
| Log all IP traffic flowing through a VNet / NSG | **Flow logs** (VNet flow logs; NSG flow logs are legacy) |
| Analyze flow logs visually (top talkers, malicious IPs, traffic patterns) | **Traffic analytics** |

### IP flow verify vs Connection troubleshoot vs Next hop
- **IP flow verify** – checks **NSG rules only**: allowed/denied + the rule name. Fast, 5-tuple (source IP, dest IP, source port, dest port, protocol).
- **Next hop** – checks **routing only**: next hop type (Internet, VirtualAppliance, VirtualNetworkGateway, VnetLocal, None) + the route table used. "None" means traffic is dropped by routing.
- **Connection troubleshoot** – end-to-end test: considers NSGs, routing *and* whether the destination actually answers. Shows hops and latency.

### Flow logs
- **VNet flow logs** – the current approach; captured at VNet level (covers more scenarios than NSG flow logs).
- **NSG flow logs** – legacy; Microsoft is retiring creation of new NSG flow logs in favor of VNet flow logs. If the exam asks the classic question, the idea is the same: "log traffic allowed/denied by NSGs".
- Stored in a **storage account** (same region). Optional retention policy.
- **Traffic analytics** processes flow logs into a **Log Analytics workspace** for dashboards.

### Packet capture
- Requires the **Network Watcher agent VM extension** on the VM.
- Output saved to a storage account or the VM's local disk as a `.cap` file.
- Can use filters (IP, port, protocol) and limits (time, size).

### Connection monitor
- Replaces the older Connection Monitor (classic) and Network Performance Monitor.
- Uses test groups: sources (Azure VMs, on-prem machines with the agent) → destinations (VMs, URLs, IPs).
- Data goes to Log Analytics; integrates with alerts.

## CLI / PowerShell
```bash
# IP flow verify: can vm01 receive RDP from the internet?
az network watcher test-ip-flow -g rg-vm --vm vm01 \
  --direction Inbound --protocol TCP \
  --local 10.0.0.4:3389 --remote 203.0.113.10:60000

# Next hop
az network watcher show-next-hop -g rg-vm --vm vm01 \
  --source-ip 10.0.0.4 --dest-ip 10.1.0.4

# Connection troubleshoot
az network watcher test-connectivity -g rg-vm --source-resource vm01 \
  --dest-address www.microsoft.com --dest-port 443

# Effective NSG rules on a NIC
az network nic list-effective-nsg -g rg-vm -n vm01-nic
```
```powershell
$nw = Get-AzNetworkWatcher -Location westeurope
Test-AzNetworkWatcherIPFlow -NetworkWatcher $nw -TargetVirtualMachineId $vm.Id `
  -Direction Inbound -Protocol TCP -LocalIPAddress 10.0.0.4 -LocalPort 3389 `
  -RemoteIPAddress 203.0.113.10 -RemotePort 60000
```

## Exam traps
- "Which NSG rule is blocking traffic?" → **IP flow verify** (or effective security rules).
- "Traffic is going to the wrong place / not reaching the firewall" → **Next hop**.
- "Test connectivity including whether the app port responds" → **Connection troubleshoot**.
- "Capture traffic for analysis" → **Packet capture** (needs the agent extension).
- "Visualize which IPs talk to which, find malicious traffic" → **Traffic analytics** on flow logs.
- Network Watcher must exist **in the same region** as the resources being diagnosed.
- Flow logs storage account must be in the **same region** as the NSG/VNet.

## Self-test
1. A user can't RDP to a VM. You suspect an NSG. Which tool tells you the exact rule?
2. You added a UDR to send traffic to an NVA, but traffic seems to bypass it. Which tool?
3. You need a timeline of latency between an on-prem server and an Azure VM. Which tool?
4. What must be installed on a VM before running packet capture?
5. Where are flow logs stored, and what turns them into dashboards?

<details><summary>Answers</summary>

1. IP flow verify.
2. Next hop.
3. Connection monitor.
4. The Network Watcher agent VM extension.
5. In a storage account; Traffic analytics (with a Log Analytics workspace).
</details>
