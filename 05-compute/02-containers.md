# Azure Containers

AZ-104 domain: Deploy and manage Azure compute resources - containers

Date learned: 2026-10-15

[← Back to README](../README.md)

## In one line
**ACR** stores container images, **ACI** runs them quickly with nothing to manage (no scaling), and **Container Apps** runs them with automatic scaling – even to zero.

## When to use it (and when not)
| Need | Use |
|---|---|
| Private place to store images | **Container Registry (ACR)** |
| Run one container (or a small group) for a task, test, or batch job | **Container Instances (ACI)** |
| Microservices/APIs that scale on HTTP or queue length, revisions, traffic split | **Container Apps** |
| Full Kubernetes control (kubectl, node pools) | **AKS** |
| One web app as a container with slots and custom domains | **App Service** (Web App for Containers) |

---

## The big picture

A container packages my app with everything it needs, so the same image runs the same way everywhere.

Three container services are on the exam:

| Service | Remember it as |
|---|---|
| **Container Registry (ACR)** | The warehouse where my images are stored |
| **Container Instances (ACI)** | Run a container quickly. No scaling. |
| **Container Apps** | Run containers that scale automatically, even down to zero |

AKS (Kubernetes) is only an overview, covered in Deck 8.

---

## 1. Container vs virtual machine

| | Virtual machine | Container |
|---|---|---|
| Starts in | Minutes | Seconds |
| Size | GBs | MBs |
| Operating system | Its own full OS | Shares the host's kernel |
| My patch | The guest OS | The image |

Think of a VM as a whole house, and a container as a flat in a shared building. Flats share the foundation (the kernel), so they are smaller and quicker to set up.

### The words the exam uses

- **Dockerfile:** the recipe for building an image
- **Image:** the template. It is immutable (never changes).
- **Tag:** a label on an image, such as `v1` or `latest`. `latest` is just a tag, not a guarantee of the newest version.
- **Repository:** all the tags of one image name
- **Container:** a running copy of an image

The flow is **Dockerfile -> image -> registry -> container.**

---

## 2. Container Registry (ACR)

A **registry** holds **repositories**, and a repository holds **tags**.

```
Registry:    catrineregistry.azurecr.io
Repository:  catalogue
Tags:        v1, v2, latest
```

- Registry name: 5-50 lower-case letters and digits, globally unique
- Image reference format: `<registry>.azurecr.io/<repository>:<tag>`

### Tiers

| Tier | Remember it as |
|---|---|
| Basic | Learning and low usage |
| Standard | Most production workloads |
| **Premium** | Adds **geo-replication** and **private endpoint** |

I can change the tier at any time.

### Three ways to get an image into ACR

| Method | When to use it |
|---|---|
| `docker push` | I build the image locally with Docker |
| **`az acr build`** | Builds the image **in the cloud** and pushes it. No Docker needed on my machine. |
| `az acr import` | Copies an image from another registry, such as Docker Hub |

**ACR Tasks** can rebuild images automatically, for example on a Git commit or when the base image gets patched.

### Who can pull images?

| Option | Use it? |
|---|---|
| Admin user | **No.** One shared password with full access, off by default. Demos only. |
| **Managed identity with the AcrPull role** | **Yes.** This is the right answer on the exam. |

| Role | Allows |
|---|---|
| **AcrPull** | Pull images |
| AcrPush | Pull and push images |
| AcrDelete | Delete images |

---

## 3. Container Instances (ACI)

ACI is the quickest way to run a container. There are no servers or clusters to manage.

### The container group

I deploy a **container group**: one or more containers that share:

- an IP address and ports (they talk over `localhost`)
- volumes (storage)
- a restart policy

Key rules:

- **One OS type per group:** Linux or Windows
- **Multi-container groups are Linux only**
- Maximum **4 vCPU and 16 GB** per group
- I pay **per second** while the group runs
- **No scaling.** For scaling, use Container Apps.

### Restart policy

| Policy | What happens when the process stops | Use for |
|---|---|---|
| **Always** (default) | Always restarts | Web apps and services |
| OnFailure | Restarts only after an error | Jobs that may fail |
| Never | Never restarts | Run-once tasks |

### Storage

- **Azure Files share:** files **survive** when the group is deleted (Linux only)
- Other volume types are lost when the group is deleted
- Use **secure environment variables** for secrets. They are hidden in the portal and in logs.

### Networking

| Option | What I get |
|---|---|
| Public | Public IP and a DNS name: `<label>.<region>.azurecontainer.io` |
| Private (VNet) | Private IP only, in a subnet delegated to container groups |

### What can I change without deleting?

| Update in place | Must delete and recreate |
|---|---|
| Image and tag | **OS type** |
| Environment variables | **CPU and memory** |
| Ports and DNS label | **Restart policy** |
| Volumes | **Network type** (public or VNet) |

---

## 4. Container Apps

Container Apps runs containers that scale automatically. Azure does the scaling for me, based on the scale rules I set (see Scaling below).

Under the hood, Container Apps runs on Kubernetes, but Azure manages and hides it completely. I never see or touch Kubernetes: no clusters, nodes or kubectl. 
I only need to know that Kubernetes exists underneath. I will learn Kubernetes itself with AKS in Deck 8.

### The four layers

```
Environment   (the boundary: network and logs)
  App         (my workload: ingress, secrets, identity)
    Revision  (a snapshot of one version)
      Replica (a running copy, scaled 0 to many)
```

### Environment

- **External:** has a public endpoint (default)
- **Internal:** private, inside my VNet
- External or internal, the VNet and the region are **fixed** when I create the environment.

### Revisions

A revision is a **version** of my app. Changing the image, environment variables, CPU, memory or scale rules creates a **new revision**.

| Mode | What happens |
|---|---|
| Single (default) | The new revision replaces the old one |
| Multiple | Run versions side by side and **split traffic**, for example 80% / 20% |

To roll back, send 100% of traffic to the old revision. I don't need to redeploy.

### Ingress (who can reach the app)

| Type | Who can reach it |
|---|---|
| External | Anyone, through a public HTTPS URL |
| Internal | Only from inside the environment or VNet |
| Disabled | No one. Used for background workers and jobs. |

IP restrictions work like App Service: blocked users get **403**.

### Scaling

| Rule type | Can it scale to zero? |
|---|---|
| **HTTP** (concurrent requests) | **Yes** |
| **TCP** (concurrent connections) | **Yes** |
| **Custom / KEDA** (queues, Service Bus, cron) | **Yes** |
| CPU or memory | **No.** The minimum is 1. |

Defaults: **minimum 0, maximum 10 replicas.**

Worked example: an HTTP rule of 10 requests per replica gets 35 requests. 35 ÷ 10 = 3.5, which rounds up to **4 replicas**.

### Size

- Fixed CPU and memory pairs, always **1 vCPU : 2 GiB**
- Maximum **4 vCPU and 8 GiB** per replica (Consumption plan)

### Jobs

Container Apps **jobs** run a task and then stop. They can be started manually, run on a schedule (cron), or be triggered by an event such as a queue message.

---

## 5. Which service should I choose?

Match the phrase in the question to the service.

| The question says... | Answer |
|---|---|
| Run one container for a task or demo, no scaling | **Container Instances** |
| Several containers sharing localhost and a volume | **Container Instances** |
| Scale on HTTP traffic or a queue, down to zero | **Container Apps** |
| Revisions, traffic splitting, rollback | **Container Apps** |
| A web app as an image, with slots, backups, custom domains | **App Service** |
| kubectl, pods, node pools | **AKS** |
| An image on my workstation must reach the service. What first? | **Push or build it into Container Registry** |

---

## 6. Top exam traps

| Wrong | Right |
|---|---|
| A Windows container group only accepts Windows clients | The OS type is the host's. Any device can connect. |
| Enable the admin user so the app can pull | Give the app's managed identity **AcrPull** |
| Add a private endpoint to a Basic registry | Private endpoint and geo-replication need **Premium** |
| Scale out an ACI group under load | ACI has **no scaling**. Use Container Apps. |
| Update ACI CPU or memory in place | **Delete and recreate** the group |
| Use a CPU rule to scale a Container App to zero | CPU and memory rules stop at 1. Use HTTP, TCP or custom rules. |

---

## Quick practice (Azure CLI)

```bash
RG=catrineh-containers-rg
ACR=catrinehacr$RANDOM          # must be globally unique, lower-case letters and digits
LOC=norwayeast
echo $ACR                  # write it down

# 1. Create a resource group and a Basic registry
az group create -n $RG -l $LOC
az acr create -g $RG -n $ACR --sku Basic

# 2. Import a public image into your registry
az acr import -n $ACR \
  --source mcr.microsoft.com/azuredocs/aci-helloworld:latest \
  --image hello:v1

# 3. List repositories and tags
az acr repository list -n $ACR -o table
az acr repository show-tags -n $ACR --repository hello -o table

# 4. Create a managed identity and give it AcrPull on the registry
az identity create -g $RG -n catrineh-pull-id
ID=$(az identity show -g $RG -n catrineh-pull-id --query id -o tsv)
PRINCIPAL=$(az identity show -g $RG -n catrineh-pull-id --query principalId -o tsv)
az role assignment create --assignee $PRINCIPAL --role AcrPull \
  --scope $(az acr show -n $ACR --query id -o tsv)

# 5. Run the image in Container Instances, pulling with the identity
#    (if the pull fails, wait a minute for the role assignment to take effect)
az container create -g $RG -n catrineh-aci \
  --image $ACR.azurecr.io/hello:v1 \
  --assign-identity $ID --acr-identity $ID \
  --os-type Linux --cpu 0.5 --memory 1 \
  --ports 80 --dns-name-label catrineh-aci-$RANDOM \
  --restart-policy Always

az container show -g $RG -n catrineh-aci --query ipAddress.fqdn -o tsv

# 6. Run a Container App with external ingress that scales to zero
az containerapp up -g $RG -n catrineh-app \
  --image mcr.microsoft.com/k8se/quickstart:latest \
  --ingress external --target-port 80

az containerapp update -g $RG -n catrineh-app \
  --min-replicas 0 --max-replicas 5 \
  --scale-rule-name http10 --scale-rule-type http \
  --scale-rule-http-concurrency 10

# 7. Clean up
az group delete -n $RG --yes --no-wait
```

---

## PowerShell equivalents

```powershell
$rg = "catrineh-containers-rg"; $acr = "catrinehacr$(Get-Random -Maximum 99999)"

New-AzResourceGroup -Name $rg -Location norwayeast
New-AzContainerRegistry -ResourceGroupName $rg -Name $acr -Sku Basic
Import-AzContainerRegistryImage -ResourceGroupName $rg -RegistryName $acr `
  -SourceRegistryUri mcr.microsoft.com -SourceImage azuredocs/aci-helloworld:latest -TargetTag "hello:v1"
Get-AzContainerRegistryRepository -RegistryName $acr

# Container Instances (Az.ContainerInstance)
$port   = New-AzContainerInstancePortObject -Port 80 -Protocol TCP
$c      = New-AzContainerInstanceObject -Name hello -Image mcr.microsoft.com/azuredocs/aci-helloworld:latest `
            -RequestCpu 1 -RequestMemoryInGb 1.5 -Port $port
$ipPort = New-AzContainerGroupPortObject -Port 80 -Protocol TCP
New-AzContainerGroup -ResourceGroupName $rg -Name catrineh-aci -Location norwayeast -Container $c `
  -OsType Linux -RestartPolicy Always -IPAddressType Public -IPAddressPort $ipPort -IPAddressDnsNameLabel "catrineh-aci-$(Get-Random)"

Get-AzContainerInstanceLog -ResourceGroupName $rg -ContainerGroupName catrineh-aci -ContainerName hello
Remove-AzContainerGroup -ResourceGroupName $rg -Name catrineh-aci
```
Container Apps are usually managed with **`az containerapp`** (CLI) or the portal.

---

## Troubleshooting (symptom → cause → fix)

| Symptom | Likely cause | Fix |
|---|---|---|
| Pull fails: "unauthorized", "InaccessibleImage", "ImagePullBackOff" | Identity lacks **AcrPull**, wrong image name/tag, or role not propagated | Check `az acr repository show-tags`, assign AcrPull, wait a minute |
| Pull fails from ACR with a firewall/private endpoint | Service can't reach the private registry | Allow trusted services / VNet path. Private endpoint needs **Premium** |
| ACI keeps restarting | Process exits and **restart policy = Always** | `az container logs` / `az container attach`; use OnFailure/Never for jobs |
| ACI "Waiting" / not starting | Image very large, or region has no capacity for that CPU/memory | Smaller request, other region |
| Need more CPU on ACI | CPU/memory can't change in place | **Delete and recreate** |
| Container App URL gives 404 / connection refused | **Ingress disabled**, or **target port** doesn't match the container's port | `az containerapp ingress enable --target-port <port>` |
| Container App never scales to zero | **min replicas = 1**, or a CPU/memory rule | min 0 + HTTP/TCP/KEDA rule |
| New version not live | Multiple-revision mode – traffic still on old revision | Move traffic to the new revision |
| `az acr login` fails | Docker not installed/running | `az acr login --expose-token`, or use `az acr build` |

---

## Break it and fix it

| # | Break it | What you should see | Fix |
|---|---|---|---|
| 1 | Remove the identity's **AcrPull** role, recreate the ACI group | Pull fails – image inaccessible | Add AcrPull back, wait, recreate |
| 2 | Run ACI with an image that exits immediately (`--command-line "echo hi"`) and **Always** | Restart count keeps rising | `--restart-policy Never` |
| 3 | `az containerapp ingress update --target-port 8080` on an app listening on 80 | Site stops responding | Set target port back to 80 |
| 4 | `az acr update --sku Basic` then try to add a private endpoint | Error – Premium needed | Premium SKU |

---

## Test yourself

1. What is the difference between an image and a container?
2. Which ACR tier do you need for a private endpoint?
3. Which command builds an image in the cloud with no Docker on your machine?
4. Which role does a managed identity need to pull images?
5. What is the default ACI restart policy?
6. Can a Windows container group have two containers?
7. You need 2 vCPU instead of 1 on an ACI group. What do you do?
8. Which Container Apps rule types can scale to zero?
9. How do you roll back a Container App?
10. Your app must scale down to zero when idle. ACI or Container Apps?

<details>
<summary>Answers</summary>

1. The image is the template. The container is a running copy of it.
2. Premium.
3. `az acr build`.
4. AcrPull.
5. Always.
6. No. Multi-container groups are Linux only.
7. Delete the group and recreate it with the new size.
8. HTTP, TCP and custom (KEDA). CPU and memory rules cannot.
9. Send 100% of traffic back to the old revision.
10. Container Apps.

</details>

### Exam-style question 1

An ACI container group is deployed with `osType: Windows`, `restartPolicy: Always`, one container on port 80, and a public IP.

| Statement | Yes / No |
|---|---|
| 1. Only devices that run Windows can connect to the container. | ? |
| 2. If the container's process stops, the container restarts automatically. | ? |
| 3. Adding a second container requires changing osType to Linux. | ? |

<details>
<summary>Answer</summary>

1. **No.** The OS type is the host's. Any device can connect on port 80.
2. **Yes.** The restart policy is Always.
3. **Yes.** Multi-container groups are Linux only.

</details>

### Exam-style question 2

You create a Container Registry named Registry1 and an AKS cluster named Cluster1. You build a container image named App1 on your workstation. You need to deploy App1 to Cluster1. What should you do first?

- A. Create a host pool on Cluster1
- B. Run the `az acr build` command
- C. Run the `docker build` command
- D. Run the `kubectl apply` command

<details>
<summary>Answer</summary>

**B.** The image has to get into the registry before the cluster can pull it. `az acr build` builds the image and pushes it to Registry1. `kubectl apply` comes after that.

</details>
