# Entra ID Labs – Overview

[← Back to README](../README.md)

## In one line
Identity answers two questions: **who are you** (Entra ID: users, groups, apps) and **what may you do** (Entra roles for the directory, Azure RBAC for resources).

All 14 labs belong to the exam domain **"Manage Azure identities and governance"** (about **20–25%** of AZ-104). The labs build on each other.

## Part 1: Identities – *who are the users?*

| Lab | Topic | Notes |
|---|---|---|
| 01 | Entra ID and users | [entra-users.md](entra-users.md) |
| 02 | Groups and membership | [groups.md](groups.md) |
| 03 | Entra licenses | [entra-licenses.md](entra-licenses.md) |
| 04 | Guest users | [guest-users.md](guest-users.md) |
| 05 | Self-service password reset (SSPR) | [sspr.md](sspr.md) |

## Part 2: Access to Azure resources – *what are they allowed to do?*

| Lab | Topic | Notes |
|---|---|---|
| 06 | Azure RBAC | [azure-rbac.md](../04-governance/azure-rbac.md) |
| 07 | RBAC scope and inheritance | [inheritance.md](../04-governance/inheritance.md) |
| 08 | Custom roles | [custom-roles.md](custom-roles.md) |
| 09 | Entra roles vs Azure RBAC roles | [entra-roles-vs-rbac.md](entra-roles-vs-rbac.md) |
| 10 | Managed identity | [managed-identities.md](managed-identities.md) |

## Part 3: Access governance and security

| Lab | Topic | Notes |
|---|---|---|
| 11 | Privileged Identity Management (PIM) | [pim.md](pim.md) |
| 12 | App registrations and enterprise applications | [app-registration.md](app-registration.md) |
| 13 | Access reviews | [access-review.md](access-review.md) |
| 14 | Conditional Access | [conditional-access.md](conditional-access.md) |

Extra: [Service Principals](service-principals.md) · [Federation](federation.md)

## Three themes that come back again and again

| Theme | Labs | Remember |
|---|---|---|
| **Entra roles vs Azure RBAC roles** | 9, 11 | Two separate systems: directory vs resources |
| **Control plane vs data plane** | 6, 7, 10 | Managing a resource ≠ accessing its data |
| **Required licenses** | 3, 5, 11, 13, 14 | P1 for Conditional Access and dynamic groups, P2 for PIM, access reviews, and Identity Protection |

## Hands-on practice track
Do this end-to-end in one sitting, from memory, after you've done the labs. It touches almost every identity topic.

1. Create two users (one with CLI, one with PowerShell). Set **usage location**
2. Create a **dynamic** group `department -eq "Development"` and an **assigned** group `sg-ops`
3. Give `sg-ops` **Reader** on a resource group, and the dev group **Contributor**
4. Invite a **guest** (your own personal email) and add them to `sg-ops`
5. Create a **custom role** "VM Power Operator" and assign it to one user on one RG
6. Make one user **eligible** for User Administrator in **PIM** with approval
7. Enable a **system-assigned managed identity** on a VM and give it **Storage Blob Data Reader**
8. Create a **Conditional Access** policy (Report-only) requiring MFA for Azure Management, excluding a break-glass account. Test with **What If**
9. Use **IAM → Check access** to prove what each user can and can't do
10. Clean up: delete users, groups, role assignments, custom role, CA policy

## Common identity troubleshooting – where to look

| Question | Look here |
|---|---|
| Why was this sign-in blocked? | **Entra ID → Sign-in logs** → Conditional Access tab |
| Who changed this user/group/role? | **Entra ID → Audit logs** |
| What can this user do on this resource? | Resource → **IAM → Check access** |
| Who changed an Azure role assignment? | **Activity log** on the subscription/RG |
| Why didn't a license apply? | **User → Licenses** / **Group → Licenses** (errors) |
