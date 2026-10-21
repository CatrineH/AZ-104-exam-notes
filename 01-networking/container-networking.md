# Container Networking

[← Back to README](../README.md)

## In one line
Containers get their own small networks: **Docker** uses bridge/overlay networks and **port mapping**, **Kubernetes** gives each pod an IP and uses **Services** for stable addresses – and in Azure, ACI and Container Apps can join your **VNet** through a delegated subnet.

## When to use it (and when not)
| Need | Use |
|---|---|
| Containers on one host talk by name | Docker **user-defined bridge** network |
| Reach a container from outside the host | **Port mapping** `-p host:container` |
| Stable address for a set of pods | Kubernetes **Service** |
| ACI / Container Apps must reach private resources | Deploy into a **VNet** (delegated subnet / internal environment) |
| Only internal access to a Container App | **Internal** ingress / internal environment |

## Key concepts

### Docker
**Bridge network** – a private network that exists only on **one host**. Containers on the same user-defined bridge reach each other **by container name**.
```
Server (Company A)
└── Bridge network
    ├── web container
    └── payment container (port 9090)
```
**Port mapping** – containers aren't reachable from outside by default:
```bash
docker run -p 9090:9090 company-a/payment
```
`-p <host port>:<container port>` – traffic to the host on 9090 goes to 9090 in the container.

**Overlay network** – connects containers **across multiple hosts**.

### Kubernetes

| Concept | Key facts |
|---|---|
| **Pod** | Smallest unit. Gets **one IP**. Pods are temporary – a recreated pod gets a **new IP** |
| **Service** | Gives a group of pods a **permanent IP and DNS name** |
| **Cluster** | A group of servers (nodes) running pods |

Apps connect to the **Service**, not to pods, because pod IPs change all the time.

### In Azure
| Service | Networking options |
|---|---|
| **ACI** | Public IP + DNS label (`<label>.<region>.azurecontainer.io`), or **private IP in a delegated subnet** |
| **Container Apps** | **External** or **internal** environment; ingress external/internal/disabled |
| **AKS** | kubenet / Azure CNI (pods get VNet IPs) – overview only for AZ-104 |
| **ACR** | Public, firewall rules, or **private endpoint** (Premium) |

## Exam traps
- ACI in a VNet → the subnet is **delegated** to `Microsoft.ContainerInstance/containerGroups` and can't host other resources
- Container Apps environment network type (external/internal) and VNet are **fixed at creation**
- Containers in one **ACI group** share an IP and talk over **localhost**
- ACR **private endpoint** needs **Premium**

## Do it
```bash
# ACI with a private IP in your VNet
az network vnet subnet create -g $RG --vnet-name vnet-lab -n snet-aci --address-prefixes 10.0.5.0/24 \
  --delegations Microsoft.ContainerInstance/containerGroups
az container create -g $RG -n aci-private --image mcr.microsoft.com/azuredocs/aci-helloworld \
  --vnet vnet-lab --subnet snet-aci --os-type Linux --cpu 1 --memory 1.5
az container show -g $RG -n aci-private --query ipAddress.ip -o tsv
```

## Troubleshooting
| Symptom | Likely cause | Fix |
|---|---|---|
| ACI in VNet fails to deploy | Subnet not **delegated**, or has other resources | Dedicated, delegated subnet |
| Docker container not reachable from outside | No **port mapping** | `-p host:container` |
| App works, then breaks after pod restart | Connecting to a **pod IP** | Use a **Service** |
| Container App reachable only from VNet | Internal ingress/environment | Expected – or recreate environment as external |

## Related
- [Containers](../05-compute/02-containers.md)
- [Network Fundamentals](network-fundamentals.md)
