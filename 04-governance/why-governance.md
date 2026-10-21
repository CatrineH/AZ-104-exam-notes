# Why Governance

[← Back to README](../README.md) · Governance 01

## In one line
**Azure governance** is the set of rules, structure, and controls that keep an Azure environment **secure, compliant, and affordable** as it grows – without checking every resource by hand.

The key question:
> **How do you control *this* for every subscription and resource, with the least possible admin effort?**
>
> **Answer: scopes.** Assign rules once, high up in the hierarchy, and let them flow down.

## When to use it (and when not) – pick the right tool

| Without governance | Governance tool that stops it | Note |
|---|---|---|
| Resources created in regions the company doesn't allow | **Azure Policy** (allowed locations) | [azure-policy.md](azure-policy.md) |
| Expensive VM sizes created by mistake | **Azure Policy** (allowed SKUs) + **budgets** | |
| Nobody knows who owns a resource or which cost center pays | **Tags** + policy that requires tags | [tags.md](tags.md) |
| A production database deleted by accident | **Resource locks** | [resource-locks.md](resource-locks.md) |
| Too many people with Owner rights | **RBAC** + **PIM** | [azure-rbac.md](azure-rbac.md) |
| Storage accounts open to the internet | **Azure Policy** (deny public access) | |
| Surprise bills at the end of the month | **Budgets** and **Cost Management** | [cost-advisor-budget.md](cost-advisor-budget.md) |
| Security settings different in every subscription | Policy and RBAC assigned at **management group** level | [scope-hierarchy.md](scope-hierarchy.md) |

<!-- Add the ten failures from the course slide here -->

### Which tool answers which question?
| Tool | Answers |
|---|---|
| **RBAC** | **Who** can do what? |
| **Policy** | **What** is allowed to exist / how must it be configured? |
| **Locks** | What must **not be deleted or changed** – even by Owners? |
| **Tags** | **Who owns it** / who pays? |
| **Budgets & Cost Management** | **How much** are we spending? |
| **Management groups** | **Where** do we assign all of the above once? |

## Key concepts

### The scope levels (details in [Scope Hierarchy](scope-hierarchy.md))
```
Tenant root group
└── Management groups      ← group subscriptions, e.g. Platform, Landing zones, Sandbox
    └── Subscriptions      ← e.g. sub-prod, sub-dev
        └── Resource groups
            └── Resources
```

### Microsoft cloud security benchmark (MCSB)
- A set of **security recommendations** from Microsoft for Azure
- Assigned automatically as a **built-in policy initiative** when you use **Microsoft Defender for Cloud**
- Gives a starting point for how companies structure their security policies

<!-- Follow up: how companies structure policy with MCSB -->

### Landing zones
A **landing zone** is a **pre-planned management group hierarchy** with built-in rules and structure, so new workloads "land" in a place that is already governed. It comes from Microsoft's **Cloud Adoption Framework (CAF)**.

```
Tenant root group
└── Contoso (intermediate root)
    ├── Platform
    │   ├── Management     ← logging, monitoring
    │   ├── Connectivity   ← hub network, firewall, DNS
    │   └── Identity       ← domain controllers
    ├── Landing zones
    │   ├── Corp           ← internal apps, connected to the company network
    │   └── Online         ← internet-facing apps
    ├── Sandbox            ← testing, few restrictions, no connection to prod
    └── Decommissioned     ← subscriptions being shut down
```

- Policies and RBAC are assigned to these **management groups**, not to each subscription
- A new subscription placed under *Corp* **automatically gets** all of Corp's rules

## Exam traps
- "**Least administrative effort**" for many subscriptions → assign at a **management group**
- Policy = **what**, RBAC = **who**. A policy Deny stops even an Owner
- Locks stop **accidents** by the right people. RBAC stops the **wrong people**
- **Tags aren't inherited** – a policy (Modify) copies them
- **Budgets only alert** – they don't stop spending

## Do it – governance practice lab
Build this in a sandbox subscription, step by step. Each step links to the note with commands.

1. Create management groups `mg-contoso` → `mg-sandbox` and move your subscription under `mg-sandbox` ([Scope Hierarchy](scope-hierarchy.md))
2. Assign **Allowed locations = Norway East** at `mg-sandbox` and try to create a storage account in West Europe ([Azure Policy](azure-policy.md))
3. Assign **Inherit a tag from the resource group** (Modify) + run a **remediation task** ([Tags](tags.md))
4. Put a **CanNotDelete** lock on a resource group and try to delete it ([Resource Locks](resource-locks.md))
5. Give a test user **Reader** at the subscription and **Contributor** on one RG, then **Check access** ([Azure RBAC](azure-rbac.md))
6. Create a **budget** of 100 kr with an 80% actual alert to your email ([Cost Management](cost-advisor-budget.md))
7. Look at **Advisor** recommendations
8. Clean up in reverse order: remove lock → delete RG → remove policy assignments → move subscription back → delete management groups

## Troubleshooting – "why was my deployment blocked?"

| Error | Cause | Where to look |
|---|---|---|
| `RequestDisallowedByPolicy` | A policy **Deny** | Error shows the assignment name → **Policy → Compliance** |
| `ScopeLocked` | A **lock** on the resource, RG, or subscription | **Locks** blade on each level |
| `AuthorizationFailed` | Missing **RBAC** permission | **IAM → Check access** |
| `QuotaExceeded` / `OperationNotAllowed` (cores) | Subscription **quota** reached | **Subscription → Usage + quotas** |
| `SkuNotAvailable` | Size not offered in that region/zone | `az vm list-skus -l <region>` |

## Related
- [Scope Hierarchy](scope-hierarchy.md)
- [Inheritance](inheritance.md)
- [Azure RBAC](azure-rbac.md)
- [Role cheat sheet](cheat-sheet.md)
- [Deployment checklist](../00-guides/az-project-skeleton.md)
