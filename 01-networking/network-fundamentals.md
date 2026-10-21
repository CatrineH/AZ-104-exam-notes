# Network Fundamentals – Overview

[← Back to README](../README.md)

## In one line
Azure networking uses the **same concepts** as an office network – addresses, subnets, routing, firewalls, NAT, DNS – they just have Azure names and you configure them as resources.

This topic follows **Business-A**, a company whose networking needs grow over time. The goal is to understand **why each networking piece exists** and which real problem it solves.

## How the pieces build on each other

| Step | Problem | Solution | Notes |
|---|---|---|---|
| 1 | Users need to find the website | DNS | [dns.md](dns.md) |
| 2 | One server runs several apps | Ports | [ports-and-protocols.md](ports-and-protocols.md) |
| 3 | Data must be delivered by shared rules | TCP / UDP | [ports-and-protocols.md](ports-and-protocols.md) |
| 4 | Everything is on one flat network | Subnets | [subnets-and-routing.md](subnets-and-routing.md) |
| 5 | Subnets need to talk to each other | Routing | [subnets-and-routing.md](subnets-and-routing.md) |
| 6 | Not everything should talk to everything | Firewalls, NSGs | [firewalls.md](firewalls.md), [nsg-inbound-outbound-rules.md](nsg-inbound-outbound-rules.md) |
| 7 | Private servers need internet access | NAT | [nat.md](nat.md) |
| 8 | Buying and running servers is slow | The cloud | See below |
| 9 | Two networks must talk | Peering / VPN / ExpressRoute | [vnet-peering.md](vnet-peering.md), [expressroute-and-virtual-wan.md](expressroute-and-virtual-wan.md) |
| 10 | Web traffic must be spread over servers | Load balancing | [application-gateway.md](application-gateway.md), [Availability](../05-compute/05-availability.md) |
| 11 | Apps run in containers | Container networking | [container-networking.md](container-networking.md) |

## Moving to the cloud

### Why?
On-premises, the company must predict capacity months ahead, buy and maintain servers, and wait weeks for new capacity. In the cloud, resources are created in minutes and you pay for what you use.

### The concepts stay the same – only the names change

| Concept | Azure service |
|---|---|
| Network | Virtual Network (VNet) |
| Subnets | Subnets in a VNet |
| Routing | System routes, User-Defined Routes (UDR) |
| Firewalls | Network Security Groups (NSG), Azure Firewall |
| NAT | NAT Gateway |
| DNS | Azure DNS, Azure Private DNS |
| IP addresses | Private and public IP addresses |
| Ports | Allowed or blocked in NSG rules |
| Jump server | Azure Bastion |
| Load balancer | Azure Load Balancer (L4), Application Gateway (L7) |
| WAN link | VPN Gateway, ExpressRoute, Virtual WAN |

## The network troubleshooting toolkit (Network Watcher)
Learn these – AZ-104 asks "which tool?" a lot.

| Question | Tool |
|---|---|
| Is this packet allowed or denied, and by **which NSG rule**? | **IP flow verify** |
| Where does traffic from this VM **go next** (which route)? | **Next hop** |
| Can VM A reach B on port X – end to end? | **Connection troubleshoot** / Connection monitor |
| Which rules apply to this NIC in total? | **Effective security rules** |
| Which routes apply to this NIC? | **Effective routes** |
| Log all traffic flows | **VNet flow logs** (+ Traffic Analytics) |
| Capture packets on a VM | **Packet capture** |
| Diagnose a VPN connection | **VPN troubleshoot** |
| See the resources and how they connect | **Topology** |

## Hands-on practice track (Week 7)
1. Plan an address space: hub `10.0.0.0/16`, spoke `10.1.0.0/16` ([IP addressing](ip-addressing-subnetting.md))
2. Create both VNets with subnets, incl. `AzureBastionSubnet` in the hub
3. Peer hub ↔ spoke ([VNet peering](vnet-peering.md))
4. Deploy a VM in the spoke with **no public IP** and connect with **Bastion**
5. NSG on the spoke subnet: deny 22 from VirtualNetwork, then use **IP flow verify** to find the rule ([NSG](nsg-inbound-outbound-rules.md))
6. Private DNS zone `corp.internal` with auto-registration on both VNets ([DNS](dns.md))
7. Route table on the spoke with `0.0.0.0/0` → None, check **Next hop**, then remove it ([Routing](subnets-and-routing.md))
8. NAT Gateway on the spoke subnet, check the outbound IP ([NAT](nat.md))
9. Two web VMs behind a Standard Load Balancer ([Availability](../05-compute/05-availability.md))
10. Clean up: Bastion, NAT Gateway, gateways and firewalls cost per hour!
