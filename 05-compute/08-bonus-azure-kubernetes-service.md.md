# AKS overview (Bonus, compute)

> **Exam status:** the deck marks AKS as *not in the current AZ-104 exam outline*. Learn it at overview level: it still shows up in older practice questions, and you'll meet it at work. Don't spend review-week time here over storage, identity, networking or VMs.

---

## One-liner

**AKS = managed Kubernetes.** Azure runs the control plane (the brain). You run the node pools (the worker VMs) and the apps on them.

## When to use

| Need | Pick |
|---|---|
| Many containers, microservices, full control over scheduling, networking, scaling | **AKS** |
| One container or a quick job, no orchestration | Container Instances (ACI) |
| Containers without managing Kubernetes, scale to zero, simple ingress | Container Apps |
| A web app or API, no containers needed | App Service |

---

## 1. Kubernetes in plain words

| Object | Plain meaning |
|---|---|
| **Cluster** | Control plane + worker nodes. The whole thing. |
| **Node** | A worker VM. In AKS, an instance in a **VM scale set**. |
| **Pod** | The smallest unit. One or more containers sharing **one IP** and volumes. |
| **Deployment** | "Keep 3 copies of this pod running." Also rolls out new image versions. |
| **Service** | A stable IP + DNS name in front of pods (pods come and go, the service stays). |
| **Namespace** | A folder inside the cluster to separate teams or apps. |

**Memory hook:** container → lives in a **pod** → copies kept by a **deployment** → reached through a **service** → all running on **nodes** → inside a **cluster**.

A minimal YAML (deployment + LoadBalancer service):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: catalogue }
spec:
  replicas: 3
  selector: { matchLabels: { app: catalogue } }
  template:
    metadata: { labels: { app: catalogue } }
    spec:
      containers:
        - name: web
          image: catrinecmptlab04cr.azurecr.io/catalogue:v1
          ports: [{ containerPort: 80 }]
---
apiVersion: v1
kind: Service
metadata: { name: catalogue }
spec:
  type: LoadBalancer
  selector: { app: catalogue }
  ports: [{ port: 80 }]
```

The **selector/label** match (`app: catalogue`) is what links the deployment's pods to the service.

---

## 2. Who manages what

| Azure manages | You manage |
|---|---|
| Control plane: API server, scheduler, etcd, controllers | Node pools: VM size, count, OS |
| Patching and availability of the control plane | Scaling, upgrades, workloads |

- **System node pool:** at least one, **Linux only**, runs critical pods (CoreDNS, metrics, tunnel).
- **User node pool:** your apps, **Linux or Windows**.
- **Tiers:** Free (no SLA) · Standard (uptime SLA) · Premium (long-term support, LTS).

## 3. The node resource group `MC_…`

AKS creates a second resource group, e.g. `MC_catrine-cmpt-lab-16-rg_<cluster>_norwayeast`, holding:
VM scale set · OS disks · load balancer · public IPs · default VNet · managed identities.

- Created **and deleted** with the cluster.
- **Don't edit by hand.** AKS may revert it, or upgrades break.
- Change it through `az aks`, `kubectl`, or the cluster's settings.
- Custom name only at creation: `--node-resource-group`.
- Can be locked down so nobody can change it.

---

## 4. Networking

### Three network models: the difference is where **pod IPs** come from

| | Kubenet (legacy) | Azure CNI (VNet IPs) | **Azure CNI Overlay (default)** |
|---|---|---|---|
| Node IPs | Subnet | Subnet | Subnet |
| Pod IPs | Private CIDR, NAT to node IP | **Real VNet IPs from the subnet** | Private overlay CIDR |
| Reach pods directly from VNet | No | **Yes** | No |
| IP use | Low | **High, plan the subnet** | Low |
| Status | Retiring March 2028 | Supported | Default for new clusters |

**Exam cue:** "pods must be reachable from other VNet resources *by their own IP*" → **Azure CNI with VNet IPs**.

### Four address ranges

| Range | Default | Used by |
|---|---|---|
| Subnet | e.g. 10.240.0.0/16 | **Nodes** |
| Pod CIDR | **10.244.0.0/16** | **Pods / containers** |
| Service CIDR | **10.0.0.0/16** | **Services** (cluster IPs) |
| DNS service IP | **10.0.0.10** | CoreDNS, sits *inside* the service CIDR |
| Docker bridge | 172.17.0.1/16 | Nothing in the cluster (legacy, a distractor) |

Rules:
- No overlap between any range, the VNet, peered VNets or on-premises.
- Service CIDR is virtual. Don't use it anywhere else.
- DNS IP must be inside the service CIDR and not its first address.

**Hook:** "**4**4 for **pods**, **0** for **services**": 10.2**44** = containers, 10.**0** = services.

### Service types

| Type | Reachable from | Azure resource created |
|---|---|---|
| ClusterIP | Inside the cluster only | None |
| NodePort | Node IP + port | None |
| **LoadBalancer** | Public IP (or internal IP with annotation) | Rule on the Standard LB in `MC_…` |
| Ingress | HTTP host/path routing | App routing add-on or App Gateway for Containers |

- Cluster IP comes from the **service CIDR**. External IP is an Azure **public IP in `MC_…`**.
- Internal LB: annotation `service.beta.kubernetes.io/azure-load-balancer-internal: "true"`.
- **Private cluster:** the API server has no public endpoint.

---

## 5. Identity and the registry

### Sign-in flow

1. **Authenticate** with Entra ID (via `kubelogin`). Disable local accounts.
2. **Authorise** with Kubernetes RBAC (RoleBindings) **or** Azure RBAC for Kubernetes.
3. **Act:** `kubectl` calls the API server as that user.

| Azure role | Allows |
|---|---|
| AKS Cluster **User** Role | Download **user** kubeconfig |
| AKS Cluster **Admin** Role | Download **admin** kubeconfig |
| AKS RBAC Reader / Writer | Read / change objects (Azure RBAC) |
| AKS RBAC Cluster Admin | Everything in the cluster (Azure RBAC) |

Week 2 link: "Cluster User/Admin Role" = permission to *get credentials*. "RBAC Reader/Writer/Cluster Admin" = permission to *do things inside* the cluster. Same idea as control plane vs data plane in storage (Contributor vs Storage Blob Data Contributor).

**Exam trap:** older questions mark "create an OAuth 2.0 endpoint" as the answer. Outdated. Today: **enable Entra integration on the existing cluster**.

### Attaching Container Registry

- `--attach-acr` gives the **kubelet identity** (lives in `MC_…`, used by every node) the **AcrPull** role on the registry.
- No admin user, no pull secret needed.
- You need rights to **create role assignments** on the registry (Owner or User Access Administrator, for example).
- Order: **image into registry first, then the deployment.**

---

## 6. Scaling

| Scales | Manual | Automatic |
|---|---|---|
| **Pods** | `kubectl scale deployment catalogue --replicas=3` | **HPA** (CPU/memory) · **KEDA** add-on (events like queue length, can scale to zero) |
| **Nodes** | `az aks nodepool scale --node-count 2` | **Cluster autoscaler**: adds nodes for *pending* pods, removes idle ones, min/max per pool |

**Hook:** HPA = more **pods**. Cluster autoscaler = more **VMs**. They work together: HPA adds pods → no room → pods pending → cluster autoscaler adds a node.

## 7. Upgrades

Order: **check → control plane → surge node → cordon and drain old node → repeat per node, per pool.**

- Control plane first, then node pools.
- **No skipping minor versions** (1.29 → 1.30 → 1.31, not 1.29 → 1.31).
- **Max surge:** extra nodes created during the upgrade so apps keep running.
- Auto-upgrade channels: none, patch, stable, rapid, node-image.
- Support: roughly the last three minor versions.

## 8. Add-ons worth knowing

- **Key Vault Secrets Provider** (Secrets Store CSI driver): mounts Key Vault secrets/keys/certs as files in pods. Auth with workload identity. Optional rotation polling.
- **Azure Policy for AKS** (Gatekeeper): audit or deny on Kubernetes objects, e.g. no privileged containers, images only from your registry. Compliance shows in Azure Policy like any other resource (Week 3 link).

---

## Exam traps (summary)

1. Deploy a YAML file → **`kubectl apply -f`**. Not `azcopy` (that's storage), not `az aks create`, not `docker push`.
2. Containers get the **pod CIDR (10.244.0.0/16)**. Services get the **service CIDR (10.0.0.0/16)**. Docker bridge 172.17.0.1/16 is a distractor.
3. Pods reachable by own IP from the VNet → **Azure CNI (VNet IPs)**, not kubenet or overlay.
4. Entra integration → enable on existing cluster (`--enable-aad`), not "OAuth 2.0 endpoint".
5. Registry access → `--attach-acr` = **AcrPull** for the **kubelet identity**.
6. Upgrade → control plane first, no skipping minor versions.
7. Don't edit `MC_…` by hand.
8. System node pool must be **Linux**. Windows only in user pools.

---

## Steps

### Portal

**Kubernetes services › Create › Kubernetes cluster**

| Tab | You decide |
|---|---|
| Basics | Preset, tier, region, version, system pool size and count |
| Node pools | User pools, OS, size, Spot, zones, autoscaler |
| Networking | Network model, ranges, private cluster |
| Integrations | Container registry, monitoring, policy |
| Security | Entra ID auth, Azure RBAC, local accounts |

### Azure CLI

```bash
RG=catrine-cmpt-lab-16-rg
AKS=catrine-cmpt-lab-16-aks

# Create
az aks create -g $RG -n $AKS -l norwayeast --tier free \
  --node-count 1 --node-vm-size Standard_FX2ms_v2 \
  --network-plugin azure --network-plugin-mode overlay \
  --attach-acr catrinecmptlab04cr --generate-ssh-keys

# Connect kubectl
az aks get-credentials -g $RG -n $AKS
kubectl get nodes

# Entra ID + Azure RBAC, no local accounts
az aks update -g $RG -n $AKS --enable-aad --enable-azure-rbac --disable-local-accounts

# Registry
az aks update -g $RG -n $AKS --attach-acr catrinecmptlab04cr
az aks check-acr -g $RG -n $AKS --acr catrinemptlab04cr.azurecr.io

# Scaling
kubectl autoscale deployment catalogue --cpu-percent=60 --min=2 --max=10        # HPA
az aks nodepool update -g $RG --cluster-name $AKS -n nodepool1 \
  --enable-cluster-autoscaler --min-count 1 --max-count 3                       # cluster autoscaler

# Upgrades
az aks get-upgrades -g $RG -n $AKS -o table
az aks upgrade -g $RG -n $AKS --kubernetes-version <next-minor>
az aks update -g $RG -n $AKS --auto-upgrade-channel patch --node-os-upgrade-channel NodeImage

# Add-ons
az aks enable-addons -g $RG -n $AKS -a azure-keyvault-secrets-provider
az aks enable-addons -g $RG -n $AKS -a azure-policy
```

### PowerShell (Az.Aks)

```powershell
$RG  = "catrine-cmpt-lab-16-rg"
$AKS = "catrine-cmpt-lab-16-aks"

New-AzAksCluster -ResourceGroupName $RG -Name $AKS -NodeCount 1 -GenerateSshKey
Import-AzAksCredential -ResourceGroupName $RG -Name $AKS     # = az aks get-credentials
Get-AzAksNodePool -ResourceGroupName $RG -ClusterName $AKS
Update-AzAksNodePool -ResourceGroupName $RG -ClusterName $AKS -Name nodepool1 -NodeCount 2
Remove-AzAksCluster -ResourceGroupName $RG -Name $AKS -Force
```

CLI is the main tool for AKS in the deck and in practice. Know the PowerShell names exist.

### kubectl basics

| Command | Does |
|---|---|
| `kubectl get nodes` | Nodes and status |
| `kubectl apply -f app.yaml` | Create or update objects from YAML |
| `kubectl get pods -o wide` | Pods, their node and IP |
| `kubectl describe pod <name>` | Details and **events** (image pull errors live here) |
| `kubectl logs <pod>` | Container output |
| `kubectl scale deployment app --replicas=3` | Change replica count |
| `kubectl get service <name> --watch` | Watch for the external IP |
| `kubectl delete -f app.yaml` | Remove what the file created |

---

## Troubleshooting

| Symptom | Likely cause | Check / fix |
|---|---|---|
| Pod stuck in `ImagePullBackOff` / `ErrImagePull` | Registry not attached, or wrong image name/tag | `kubectl describe pod` → events · `az aks check-acr` · `az aks update --attach-acr` |
| Pod stuck in `Pending` | No room on nodes | `kubectl describe pod` → "Insufficient cpu" · scale node pool or enable cluster autoscaler |
| Service `EXTERNAL-IP` shows `<pending>` | LB still being created | Wait, `--watch` · if it stays pending, check quota for public IPs |
| `kubectl` says "Unauthorized" / "Forbidden" | No Cluster User Role, or no RBAC role inside cluster | Check Azure role assignments at cluster scope |
| `--attach-acr` fails | You can't create role assignments on the registry | Ask for Owner / User Access Administrator on the registry |
| Upgrade refused | Trying to skip a minor version | `az aks get-upgrades` and go one minor at a time |
| Things "fix themselves" back after manual change in `MC_…` | AKS reconciles the node RG | Make the change through `az aks` / cluster settings |

---

## Lab L16: AKS first contact (optional, 60 min)

**Bring:** the L04 registry `catrinecmptlab04cr`. One node, at least 2 vCPU / 4 GB, running about an hour.

1. Create Free-tier cluster, 1 node, overlay networking, registry attached.
2. `az aks get-credentials` → `kubectl get nodes`.
3. Open `MC_…` in the portal. **Look only.** Find the scale set, LB, public IP, kubelet identity.
4. `kubectl apply -f app.yaml` (catalogue image, LoadBalancer service).
5. `kubectl get service catalogue --watch` → open the external IP in a browser.
6. `kubectl scale deployment catalogue --replicas=3` → `kubectl get pods -o wide` shows 3 pods.
7. Scale the node pool up (`az aks nodepool scale --node-count 2`).

**Proves:** external IP shows the app · 3 pods listed · cluster and `MC_…` gone after clean-up.

### Break it and fix it

| Break | What you'll see | Fix |
|---|---|---|
| Change the image tag to `catalogue:v99` and apply | `ErrImagePull`, event says "not found" | Fix the tag, re-apply |
| `az aks update --detach-acr catrinecmptlab04cr`, then delete a pod | New pod: `ImagePullBackOff`, "unauthorized" | `az aks update --attach-acr ...` |
| Scale to `--replicas=30` on one small node | Many pods `Pending`, "Insufficient cpu" | Scale back, or enable cluster autoscaler |
| Change the service selector to `app: wrong` | External IP loads nothing | `kubectl describe service` shows no endpoints → fix selector |

## Clean up (same day)

```bash
az aks delete -g $RG -n $AKS --yes --no-wait
az group list --query "[?starts_with(name,'MC_catrine-cmpt-lab-16')].name" -o tsv   # should go empty
kubectl config delete-context $AKS   # tidy the local kubeconfig
```

Deleting the cluster removes `MC_…` too. Don't delete `MC_…` on its own first.

---

## Deck practice questions

**Q1. kubenet profile: pod CIDR 10.244.0.0/16, service CIDR 10.0.0.0/16, DNS 10.0.0.10, Docker bridge 172.17.0.1/16. Containers get IPs from … services from …**
→ **D: containers 10.244.0.0/16, services 10.0.0.0/16.**
Why not the others: 172.17.0.1/16 is the Docker bridge and nothing in the cluster uses it (A). Pods and services never share a range (B, C).

**Q2. AKS1 credentials are in Cloud Shell. Deploy app.yaml. What do you run?**
→ **C: `kubectl apply -f app.yaml`.**
Why not: `azcopy` copies to storage (A). `az aks create` makes clusters, it has no `--yaml` (B). `docker push` pushes images to a registry, not YAML to a cluster (D).
