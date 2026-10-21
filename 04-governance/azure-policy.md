# Azure Policy

[← Back to README](../README.md) · Governance

## In one line
**Azure Policy** decides **what is allowed to exist** in Azure – which regions, which VM sizes, which settings, which tags – and can **block**, **report**, or **fix** resources that break the rules.

## When to use it (and when not)
| Requirement | Use |
|---|---|
| Only allow certain regions / VM sizes / SKUs | Policy **Deny** (Allowed locations, Allowed virtual machine size SKUs) |
| Find storage accounts that allow public access, without blocking anyone | Policy **Audit** |
| Automatically add/copy tags | Policy **Modify** |
| Automatically deploy diagnostic settings / extensions | Policy **DeployIfNotExists** |
| A whole compliance standard (ISO 27001, MCSB) | Policy **initiative** |
| Decide **who** may do something | **RBAC**, not policy |
| Stop one important resource from being deleted | **Lock** (simpler) |

## Key concepts
- The **most powerful tool** in the governance toolbox
- Can **block** (Deny), **report** (Audit), or **fix** (Modify, DeployIfNotExists) resources
- Works on **resource properties**, not on users

### Policy vs RBAC

| | Azure Policy | Azure RBAC |
|---|---|---|
| Answers | **What** can exist? | **Who** can do it? |
| Example | Only Norway East is allowed | Cat is Contributor on rg-dev |
| Focus | Resource properties | User actions |

 **Both must allow it.** No role bypasses a policy Deny – **not even an Owner**.


### The building blocks

```
Definition ──► Initiative ──► Assignment ──► Remediation
```

| Block | What it is | Example |
|---|---|---|
| **Definition** | **One rule** in JSON: *if* a condition matches, *then* an effect | "Allowed locations" |
| **Initiative** (policy set) | A **group of definitions** with one goal | **Microsoft cloud security benchmark**, ISO 27001 |
| **Assignment** | A definition or initiative **applied to a scope**, with parameters | "Allowed locations = Norway East" on management group *Landing zones* |
| **Remediation** | A **task** that fixes **existing** non-compliant resources | Add a missing tag to 200 old resources |

#### Definition example – require a tag

```json
{
  "if": {
    "field": "tags['costCenter']",
    "exists": "false"
  },
  "then": {
    "effect": "deny"
  }
}
```

> **Benchmark:** the Microsoft cloud security benchmark (MCSB) is a built-in **initiative**. Defender for Cloud assigns it by default and shows the results as security recommendations.


### Effects

#### Ranked from mildest to strongest

| Effect | What it does |
|---|---|
| **Disabled** | Turns the policy off |
| **Audit** | **Reports** non-compliant resources, blocks nothing |
| **AuditIfNotExists** | Reports if a **related resource** is missing (e.g. a VM without an antimalware extension) |
| **Append** | **Adds** properties at create/update (older – use Modify for tags) |
| **Modify** | **Adds, changes, or removes** properties and tags – fixes them |
| **DeployIfNotExists** (DINE) | **Deploys** a missing related resource (e.g. diagnostic settings) |
| **DenyAction** | **Blocks an action**, such as **delete** |
| **Deny** | **Blocks** create or update requests that don't comply |

#### Which effect? Read the wording

| The question says… | Effect |
|---|---|
| "prevent", "block", "must not be created", "only allow" | **Deny** |
| "identify", "report", "track", "without affecting users" | **Audit** |
| "report VMs **missing** an extension / diagnostic setting" | **AuditIfNotExists** |
| "**automatically add** a tag", "inherit a tag", "change a setting" | **Modify** |
| "**automatically deploy** an extension / diagnostic setting" | **DeployIfNotExists** |
| "prevent **deletion**" | **DenyAction** (or a **CanNotDelete lock**) |
| "turn off without removing the assignment" | **Disabled** effect, or enforcement mode **DoNotEnforce** |

#### Effect vs enforcement mode

| | Effect | Enforcement mode |
|---|---|---|
| Set on | The definition (often as a parameter) | The **assignment** |
| Options | Audit, Deny, Modify… | **Default** (enforce) or **DoNotEnforce** |
| DoNotEnforce means | – | Compliance is **still evaluated and reported**, but nothing is **blocked or fixed** |


### Exclusions vs exemptions

| | Exclusion (`notScopes`) | Exemption |
|---|---|---|
| Is | A **property of the assignment** | A **separate object** at a lower scope |
| Effect | The scope is **not evaluated at all** | Still evaluated, shown as **Exempt** |
| Categories | – | **Waiver** (accepted risk) or **Mitigated** (solved another way) |
| Expiry date | NO | YES, Optional |
| Use for | "This resource group should never be part of the policy" | "This one resource is allowed to break the rule – for now" |


### Who can assign policy?

| Role | Can assign policy? |
|---|---|
| **Resource Policy Contributor** | YES **Least privilege** for policy |
| **Owner** | YES |
| **Contributor** | NO Contributor alone **is not enough** – it can't write to `Microsoft.Authorization` |

Two questions to ask:
- **What** can you do? → the **role** (least privilege)
- **Where** can you do it? → the **scope** (blast radius)


### Modify and DINE need an identity

Modify and DeployIfNotExists **change** resources, so the assignment needs a **managed identity** with permission to do it.

```
Assign ──► Identity ──► Role ──► Remediate
```

1. **Assign** the policy
2. The assignment gets a **managed identity** (system- or user-assigned)
3. The identity gets a **role**, e.g. Contributor or Tag Contributor, on the scope
4. Create a **remediation task** for existing resources

| Resources | When they're fixed |
|---|---|
| **New or updated** resources | **On the spot**, during the request |
| **Existing** resources | **Only via a remediation task** |

> **Exam:** "The policy works for new resources, but existing ones still lack the tag" → **create a remediation task**.

> To assign a policy with a managed identity and role, you also need permission to **create role assignments** – Owner, or Resource Policy Contributor + User Access Administrator.


### How policy is evaluated

- **Any applicable Deny wins** – the **most restrictive** outcome applies when several assignments match
- Evaluation happens:
  1. **On every request** – create or update
  2. **On assignment change** – new or updated assignment
  3. **Every 24 hours** – a full compliance scan
  4. **On demand** – `az policy state trigger-scan`
- **A new assignment is not instant.** Allow some minutes before testing (often up to 30 minutes)

### What Deny does to existing resources

```
Keeps running ──► Reported as non-compliant ──► Updates can be denied ──► You fix it yourself
```

**Policy never deletes.** Existing non-compliant resources keep running and are flagged.


### Pitfalls from real life
- **Deny on tags breaks platform resources** – some resources are created by Azure services without your tags
- **An untested Deny high in the tree** – a big blast radius
- **Expecting instant results** – evaluation takes time
- **Assigning where Owners can undo it** – a subscription Owner can remove a policy assigned at the subscription, but not one assigned at a management group

> **Rule of thumb: Test low, roll out high.** Audit in a sandbox first.

#### Audit first, Deny later

```
Audit ──► Review ──► Deny ──► Remediate
```


## Exam traps
- **Policy says what, RBAC says who.** No role bypasses a Deny – not even an Owner
- **Contributor can't assign policy** → **Resource Policy Contributor** (least privilege) or Owner
- New assignment takes **up to ~30 minutes** to take effect – "it didn't block yet" is normal
- **Existing** non-compliant resources aren't deleted or fixed by Deny/Modify → **remediation task** (Modify/DINE) or fix them yourself
- Modify/DINE need a **managed identity** on the assignment + a role
- Exclusion (`notScopes`) = not evaluated at all. **Exemption** = evaluated, shown as Exempt, can expire
- Several assignments on the same scope → the **most restrictive** wins (any Deny blocks)
- **Initiative** = group of definitions. "Assign many policies at once / track compliance against a standard" → initiative
- Test with **Audit** or **DoNotEnforce** before Deny

## Do it

### Portal
1. **Policy → Definitions** → search "Allowed locations" → **Assign**
2. **Basics:** scope (MG/subscription/RG) + **exclusions** → enforcement **Enabled/Disabled**
3. **Parameters:** Norway East
4. **Remediation** (Modify/DINE only): create a managed identity + **Create a remediation task**
5. **Non-compliance messages:** text the user sees when denied
6. **Policy → Compliance** → open the assignment → non-compliant resources
7. **Policy → Exemptions** / **Remediation**

### Azure CLI
```bash
# Find a built-in definition (Allowed locations = e56962a6-4747-49cd-b67b-bf8b01975c4c)
az policy definition list --query "[?displayName=='Allowed locations'].name" -o tsv

# Assign it
az policy assignment create \
  --name allowed-locations \
  --policy e56962a6-4747-49cd-b67b-bf8b01975c4c \
  --scope /subscriptions/<id>/resourceGroups/AZ104-catrine-rg \
  --params '{ "listOfAllowedLocations": { "value": ["norwayeast"] } }'

# Assign a Modify policy with a managed identity and role
az policy assignment create \
  --name inherit-costcenter \
  --policy <definition-name> \
  --scope /subscriptions/<id> \
  --mi-system-assigned --location norwayeast \
  --role "Tag Contributor" --identity-scope /subscriptions/<id>

# Fix existing resources
az policy remediation create --name fix-tags --policy-assignment inherit-costcenter

# Check compliance / scan now
az policy state summarize --resource-group AZ104-catrine-rg
az policy state trigger-scan --resource-group AZ104-catrine-rg
```

# Exemption for one RG (waiver, expires)
az policy exemption create --name exempt-legacy --policy-assignment <assignment-id> \
  --exemption-category Waiver --scope /subscriptions/<id>/resourceGroups/rg-legacy \
  --expires-on 2026-12-31T00:00:00Z

# List non-compliant resources
az policy state list --resource-group AZ104-catrine-rg --filter "complianceState eq 'NonCompliant'" \
  --query "[].{resource:resourceId, policy:policyDefinitionName}" -o table
```

### PowerShell
```powershell
$def = Get-AzPolicyDefinition -Name e56962a6-4747-49cd-b67b-bf8b01975c4c   # Allowed locations
New-AzPolicyAssignment -Name "allowed-locations" -PolicyDefinition $def `
  -Scope "/subscriptions/<id>/resourceGroups/AZ104-catrine-rg" `
  -PolicyParameterObject @{ listOfAllowedLocations = @("norwayeast") }

Start-AzPolicyRemediation -Name "fix-tags" -PolicyAssignmentId <assignment-id>
Start-AzPolicyComplianceScan -ResourceGroupName AZ104-catrine-rg
Get-AzPolicyState -ResourceGroupName AZ104-catrine-rg -Filter "ComplianceState eq 'NonCompliant'"
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| **RequestDisallowedByPolicy** when deploying | A **Deny** assignment | Error names the assignment → change the deployment, or add an exemption |
| New policy doesn't block anything yet | Assignment not active yet (up to ~30 min) | Wait, then test |
| Compliance shows "Not started" | Evaluation hasn't run | `az policy state trigger-scan` |
| Existing resources still non-compliant | Policies don't fix existing resources automatically | **Remediation task** (Modify/DINE), or fix manually |
| Remediation task fails | Assignment identity has no / too small role | Give the identity the right role at the scope |
| "Does not have authorization… policyAssignments/write" | You're Contributor | **Resource Policy Contributor** or Owner |
| Can't create the assignment with an identity | You can't create role assignments | Owner, or + User Access Administrator |
| Platform resources (managed RGs) fail to deploy | Deny on tags/locations hits Azure-created resources | Exclude those scopes, or use Audit/Modify |
| Policy shows on a resource but you can't find the assignment in the RG | Assigned at **subscription/MG** | **Policy → Assignments** with a higher scope |

## Clean up
```bash
az policy remediation delete --name fix-tags --resource-group AZ104-catrine-rg
az policy assignment delete --name allowed-locations --scope /subscriptions/<id>/resourceGroups/AZ104-catrine-rg
az policy assignment delete --name inherit-costcenter --scope /subscriptions/<id>
```

## Check yourself
<details><summary>1. Report storage accounts that allow public blob access without blocking anyone.</summary>

Assign the built-in policy with the **Audit** effect.
</details>

<details><summary>2. A Modify policy adds a tag to new resources, but 300 old resources still lack it.</summary>

Create a **remediation task**.
</details>

<details><summary>3. One RG must be allowed to break "Allowed locations" until the end of the year, and it should still show in reports.</summary>

A policy **exemption** (Waiver) with an **expiry date** – not an exclusion.
</details>

<details><summary>4. Least-privilege role to assign policies?</summary>

**Resource Policy Contributor**.
</details>

---

## Policy in five sentences
1. **Policy says what, RBAC says who.** No role bypasses a Deny – not even an Owner
2. **Assign once, high up.** Assignments inherit down. Exclusions and exemptions carve out
3. **Audit first, Deny later.** Test with Audit or DoNotEnforce before you block anything
4. **Existing resources need remediation.** Modify and DINE fix them only through a remediation task
5. **Policy never deletes.** Non-compliant resources keep running and are flagged

## Related
- [Inheritance](inheritance.md)
- [Scope Hierarchy](scope-hierarchy.md)
- [Tags](tags.md)
- [Resource Locks](resource-locks.md)
- [Azure RBAC](azure-rbac.md)

