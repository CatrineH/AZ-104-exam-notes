# Azure Tags

[← Back to README](../README.md) · Governance

## In one line
**Tags are labels that don't inherit**: `name:value` pairs on resources, resource groups, and subscriptions – used to find, organise, and **split costs** (owner, environment, cost center).

## When to use it (and when not)
| Need | Use |
|---|---|
| Split the bill by department / project | Tags + **Cost analysis** grouped by tag |
| Find all prod resources | Tag `environment=prod` + filter |
| Every resource **must** have an owner tag | Policy **Require a tag** (Deny) |
| Copy `costCenter` from the RG to resources | Policy **Inherit a tag from the resource group** (Modify) |
| Store a password or personal data | **Never** in tags – they're plain text and visible to Readers |
| Control access | Not tags – **RBAC** (tags aren't a security boundary) |

## Key concepts

| Fact | Detail |
|---|---|
| **Not inherited** | A tag on an RG does **not** appear on the resources inside it |
| **Limits** | **50** tags per resource/RG/subscription. Name 512 characters (128 for storage accounts), value 256 |
| **Plain text** | Never put secrets, passwords, or personal data in tags |
| **Tag Contributor** | Can manage tags **without** access to the resources themselves |
| **Cost analysis** | Group and filter costs by tag – the financial payoff. Tags only appear in cost data **from when they're added** (not backwards) |
| **Enforced by Policy** | Unenforced tags quickly become unreliable |
| **Not all resource types** | Most – not all – resource types support tags |

### Enforce tags with policy
Pick the effect by the goal: block, add, copy, or fix.

| Goal | Built-in policy | Effect |
|---|---|---|
| Block resources without an `owner` tag | Require a tag on resources | **Deny** |
| Add `environment=dev` if missing | Add or replace a tag on resources | **Modify** |
| Copy `costCenter` from the RG | Inherit a tag from the resource group if missing | **Modify** |
| Fix **existing** untagged resources | Modify policy + **remediation task** | **Modify** |
| Just report missing tags | Require a tag → assign with **Audit** | **Audit** |

## Exam traps
- **Tags don't inherit** – the answer is a **Modify** policy, not "tag the RG"
- Modify fixes **new/updated** resources. **Existing** ones need a **remediation task**
- Modify policies need a **managed identity** with a role (for example **Tag Contributor**)
- "Manage tags only, nothing else" → **Tag Contributor**
- Use **Append/Modify** to add tags, **Deny** to require them – Deny on tags can break platform-created resources (for example managed RGs) → test with Audit first
- `az resource tag` / `Set-AzResource -Tag` **replace all** tags. Use **merge** to add one
- Cost analysis by tag doesn't show costs from before the tag existed

## Do it

### Portal
- Any resource/RG → **Tags** → add name/value → Apply
- **Several resources at once:** RG → select resources → **Assign tags**
- **Search by tag:** **All resources → Add filter → Tag**
- **Costs:** **Cost Management → Cost analysis → Group by → Tag → costCenter**
- **Policy:** **Policy → Definitions** → search "tag" → **Assign** → Remediation tab → **Create a remediation task** + managed identity

### Azure CLI
```bash
RID=$(az storage account show -n $SA -g $RG --query id -o tsv)

# Add/merge a tag (keeps the others)
az tag update --resource-id $RID --operation Merge --tags environment=dev owner=cat
# Replace ALL tags
az tag create --resource-id $RID --tags environment=dev
# Remove one tag
az tag update --resource-id $RID --operation Delete --tags owner=cat

# Tag a resource group
az group update -n $RG --set tags.costCenter=1234
# Find resources by tag
az resource list --tag environment=dev -o table

# Assign "Inherit a tag from the resource group if missing" + fix existing
DEF=$(az policy definition list --query "[?displayName=='Inherit a tag from the resource group if missing'].name" -o tsv)
az policy assignment create --name inherit-costcenter --policy $DEF \
  --scope /subscriptions/<sub-id> --params '{ "tagName": { "value": "costCenter" } }' \
  --mi-system-assigned --location norwayeast --role "Tag Contributor" --identity-scope /subscriptions/<sub-id>
az policy remediation create --name fix-costcenter --policy-assignment inherit-costcenter
```

### PowerShell
```powershell
$rid = (Get-AzStorageAccount -ResourceGroupName $rg -Name $sa).Id

Update-AzTag -ResourceId $rid -Tag @{ environment = "dev"; owner = "cat" } -Operation Merge
Update-AzTag -ResourceId $rid -Tag @{ owner = "cat" } -Operation Delete
Get-AzTag -ResourceId $rid

Set-AzResourceGroup -Name $rg -Tag @{ costCenter = "1234" }     # replaces RG tags
Get-AzResource -TagName environment -TagValue dev
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Resources in the RG have no tags | Tags **don't inherit** | Inherit-tag policy + remediation |
| Old resources still untagged after assigning Modify | Modify only acts on create/update | **Remediation task** |
| Remediation task fails | Assignment's managed identity lacks a role | Give it **Tag Contributor** (or Contributor) at the scope |
| All other tags disappeared | Used a **replace** command | Use `--operation Merge` / `-Operation Merge` |
| Deployment blocked: `RequestDisallowedByPolicy` | Require-tag **Deny** policy | Add the tag in the deployment |
| Tag missing in cost analysis | Tag added recently, or resource type doesn't send tags to billing | Costs are tagged from that day on |

## Clean up
```bash
az policy assignment delete --name inherit-costcenter --scope /subscriptions/<sub-id>
az tag update --resource-id $RID --operation Delete --tags environment=dev
```

## Check yourself
<details><summary>1. Finance wants costs split per department with the least effort, and every new resource must carry the department from its RG.</summary>

Tag RGs with `department`, assign **Inherit a tag from the resource group if missing** (Modify) + remediation, then **Cost analysis → group by tag**.
</details>

<details><summary>2. A user must update tags but must not change any resource settings.</summary>

**Tag Contributor**.
</details>

<details><summary>3. You run `az resource tag --tags owner=cat` on a resource with 5 tags. How many tags does it have now?</summary>

**One** – that command replaces all tags. Use `az tag update --operation Merge`.
</details>

## Related
- [Azure Policy](azure-policy.md)
- [Inheritance](inheritance.md)
- [Cost Management, Advisor, and Budgets](cost-advisor-budget.md)
