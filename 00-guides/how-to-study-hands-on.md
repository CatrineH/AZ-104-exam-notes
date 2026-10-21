# How to Study AZ-104 Hands-On

[← Back to README](../README.md)

The exam tests whether I can **design, deploy, monitor, and troubleshoot** Azure – not whether I memorised definitions. Questions are usually **scenarios**: *"A user can't reach the file share. What should you check first?"* The answer comes easily if I have built it, broken it, and fixed it myself.

## The note template
Every topic note in this repo follows the same structure:

```markdown
# Topic name

## In one line            ← the whole idea in one sentence
## When to use it (and when not)   ← requirement → service table
## Key concepts           ← my course notes, tables, diagrams
## Exam traps             ← the things that turn a right answer wrong
## Do it
### Portal
### Azure CLI
### PowerShell
## Troubleshooting        ← symptom → cause → fix
## Clean up               ← stop paying, avoid conflicts
## Check yourself         ← scenario questions, answers hidden
## Related
```

Labs add a **Break it and fix it** section: break one thing on purpose, read the exact error, then fix it.

## A study day (2–4 hours)

| Time | Do |
|---|---|
| 15 min | Read **In one line**, **When to use**, **Key concepts** |
| 60–90 min | **Build it** – Portal first, then the same with CLI or PowerShell |
| 30 min | **Break it and fix it** – write down every error message and what fixed it |
| 15 min | **Exam traps** + **Check yourself** (answer before opening) |
| 10 min | **Clean up**, then a line in the [learning log](../learning-log.md) |

## Lab hygiene – keep my bill near zero
1. **One resource group per lab** (`rg-lab-<topic>`) – delete it when done
2. A **budget** with an 80% alert on my subscription (see [Cost Management](../04-governance/cost-advisor-budget.md))
3. **Auto-shutdown** on every lab VM
4. Delete the expensive-per-hour things the same day: **VPN gateway, Azure Firewall, Application Gateway, Bastion, NAT Gateway, private endpoints**
5. Use the free **Microsoft Learn sandbox** when a module offers one
6. Remove **locks** before deleting a resource group

## My troubleshooting method
Work in this order – it's also the order the exam expects:

| Step | Ask | Tool |
|---|---|---|
| 1. Read the error | What exactly does it say? (`AuthorizationFailed`, `RequestDisallowedByPolicy`, `ScopeLocked`, `QuotaExceeded`…) | Portal notification, **Deployments** blade, CLI output |
| 2. Who changed what? | Did someone change it recently? | **Activity log** |
| 3. Permission? | Does the identity have the right role, at the right scope, on the right plane? | **IAM → Check access** |
| 4. Governance? | Is a policy denying it, or a lock blocking it? | **Policy → Compliance**, **Locks** |
| 5. Network? | NSG, route, firewall, DNS? | **IP flow verify**, **Next hop**, **Effective rules/routes**, `nslookup` |
| 6. Inside the resource? | Service running? OS firewall? Disk full? | **Run Command**, **Boot diagnostics**, **Log stream**, logs |

## Common Azure error codes

| Error | Means | Look at |
|---|---|---|
| `AuthorizationFailed` | Missing RBAC permission (control plane) | IAM |
| `AuthorizationPermissionMismatch` | Missing **data** role (storage) | IAM – Storage Blob Data roles |
| `AuthorizationFailure` (storage) | Storage **firewall** | Storage → Networking |
| `RequestDisallowedByPolicy` | Policy Deny | Policy → Assignments |
| `ScopeLocked` | Resource lock | Locks |
| `QuotaExceeded` | vCPU or other quota | Usage + quotas |
| `SkuNotAvailable` | Size not in that region/zone | `az vm list-skus` |
| `MissingSubscriptionRegistration` | Resource provider not registered | `az provider register` |
| `InvalidTemplate` | Template syntax | `az deployment group validate` |

## Practice with Claude
- *"Give me 5 scenario questions on <topic> like the AZ-104 exam, one at a time, with hints and explanations."*
- *"Here's my lab setup. Give me a broken version to troubleshoot without telling me what's wrong."*
- *"Show me five CLI/PowerShell snippets for <topic> with one thing wrong in each."*

## Related
- [Deployment checklist](az-project-skeleton.md)
- [Command recognition](command-recognition-gov.md)
- [CLI and PowerShell VM commands](CLI-azure-commands.md)
