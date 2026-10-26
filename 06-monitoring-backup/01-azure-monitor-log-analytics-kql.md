# Azure Monitor, Log Analytics & KQL

## The big picture
**Azure Monitor** is the umbrella service for collecting, analyzing and acting on telemetry from Azure, on-prem and other clouds. Almost every other monitoring feature (alerts, insights, workbooks) sits on top of it.

Two data types — learn this split first:

| | **Metrics** | **Logs** |
|---|---|---|
| What | Numeric values over time (CPU %, disk IOPS) | Records with many properties (events, traces, sign-ins) |
| Stored in | Azure Monitor Metrics database (time-series) | Log Analytics workspace |
| Latency | Near real-time (~1 min) | Minutes |
| Default retention | 93 days | 30 days in workspace (configurable, up to 2 years interactive; longer with long-term retention/archive) |
| Query with | Metrics Explorer | **KQL** in Log Analytics |
| Cost | Platform metrics are free | Pay per GB ingested + retention |

> Mental model: **metrics = "how much / how fast right now"**, **logs = "what happened and why"**.

## Where data comes from
- **Platform metrics** – emitted automatically by Azure resources. No setup.
- **Activity log** – subscription-level control-plane events: *who* created/deleted/changed *what*, when. Kept 90 days. Free. Send to a workspace to keep longer and query with KQL.
- **Resource logs** (formerly diagnostic logs) – data-plane operations *inside* a resource (e.g., Key Vault access, storage reads). **Not collected by default** → you must create a **diagnostic setting**.
- **Guest OS data** (inside the VM: event logs, syslog, perf counters) – needs the **Azure Monitor Agent (AMA)** + a **Data Collection Rule (DCR)**.
- **Microsoft Entra logs** – sign-in and audit logs, routed via Entra diagnostic settings.

### Diagnostic settings – destinations
A diagnostic setting sends resource logs/metrics to one or more of:
1. **Log Analytics workspace** – query with KQL, alerting
2. **Storage account** – cheap archive / compliance
3. **Event Hubs** – stream to external SIEM (Splunk, etc.)
4. Partner solutions

### Azure Monitor Agent (AMA)
- Replaces the legacy **Log Analytics agent (MMA/OMS)**, which is retired — if you see MMA on the exam, AMA is the modern answer.
- Configuration lives in **Data Collection Rules (DCRs)**: *what* to collect and *where* to send it. One DCR can apply to many VMs; one VM can have many DCRs.
- **VM Insights** uses AMA to give performance charts and a dependency map.

## Log Analytics workspace
- The container that stores log data. Region-bound.
- Access control: workspace-level RBAC **or** resource-context (users see only logs for resources they can access). Roles to know: **Log Analytics Reader**, **Log Analytics Contributor**, **Monitoring Reader**, **Monitoring Contributor**.
- Design tip: fewer workspaces = easier cross-resource queries. Split only for data residency, billing or strict access reasons.
- Pricing: pay-as-you-go per GB, or **commitment tiers** (100 GB/day+) for discounts. A **daily cap** can limit cost (but drops data after the cap!).

## KQL – Kusto Query Language
Read-only query language. Data flows top to bottom through the pipe `|`.

### Core operators (know these cold)
| Operator | Purpose | SQL equivalent |
|---|---|---|
| `where` | Filter rows | WHERE |
| `project` | Choose/rename columns | SELECT |
| `extend` | Add calculated column | SELECT ..., expr AS x |
| `summarize` | Aggregate | GROUP BY |
| `count` | Count rows | COUNT(*) |
| `sort by` / `order by` | Sort (default **descending**!) | ORDER BY |
| `top N by` | First N sorted rows | TOP |
| `take` / `limit` | Any N rows (not sorted) | — |
| `distinct` | Unique values | DISTINCT |
| `join` | Combine tables | JOIN |
| `render` | Chart the result | — |
| `ago()` | Relative time | — |
| `bin()` | Bucket time/numbers | — |

### Common tables
| Table | Contains |
|---|---|
| `Heartbeat` | Agent heartbeats – is the VM alive? |
| `Perf` | Performance counters |
| `Event` | Windows event logs |
| `Syslog` | Linux syslog |
| `AzureActivity` | Activity log |
| `AzureDiagnostics` | Resource logs (legacy/shared mode) |
| `SigninLogs`, `AuditLogs` | Entra ID |
| `InsightsMetrics` | VM Insights perf data |

### Example queries to practice
```kql
// VMs that haven't sent a heartbeat in 15 minutes
Heartbeat
| summarize LastSeen = max(TimeGenerated) by Computer
| where LastSeen < ago(15m)
```

```kql
// Average CPU per computer in 5-minute buckets, last hour, as a chart
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize AvgCPU = avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart
```

```kql
// Who deleted resources in the last 7 days?
AzureActivity
| where TimeGenerated > ago(7d)
| where OperationNameValue endswith "DELETE"
| project TimeGenerated, Caller, ResourceGroup, OperationNameValue, ActivityStatusValue
```

```kql
// Windows error events, top 10 noisiest sources
Event
| where EventLevelName == "Error"
| summarize Count = count() by Source
| top 10 by Count
```

```kql
// Failed sign-ins per user
SigninLogs
| where ResultType != "0"
| summarize Failures = count() by UserPrincipalName
| sort by Failures
```

## Visualizing
- **Metrics Explorer** – charts of metrics, pin to dashboards.
- **Workbooks** – interactive reports mixing KQL, metrics, text and parameters. The go-to for rich reports.
- **Azure dashboards** – pin tiles from many places.
- **Insights** – curated experiences: VM Insights, Container Insights, Storage Insights, Network Insights, Application Insights (APM for apps).

## CLI / PowerShell
```bash
# Create a workspace
az monitor log-analytics workspace create -g rg-monitor -n law-cat --location westeurope

# Diagnostic setting: send Key Vault logs to the workspace
az monitor diagnostic-settings create \
  --name kv-to-law \
  --resource /subscriptions/<sub>/resourceGroups/rg/providers/Microsoft.KeyVault/vaults/kv-cat \
  --workspace law-cat \
  --logs '[{"categoryGroup":"allLogs","enabled":true}]'

# Run a KQL query from the CLI
az monitor log-analytics query -w <workspace-guid> \
  --analytics-query "Heartbeat | summarize count() by Computer"
```
```powershell
New-AzOperationalInsightsWorkspace -ResourceGroupName rg-monitor -Name law-cat -Location westeurope -Sku PerGB2018
Invoke-AzOperationalInsightsQuery -WorkspaceId <guid> -Query "Heartbeat | take 10"
```

## Exam traps
- **Resource logs are off by default** – the fix is almost always "create a diagnostic setting".
- Need guest OS logs/counters? → **AMA + DCR**, not a diagnostic setting.
- "Who stopped the VM?" → **Activity log**.
- Keep activity log longer than 90 days → send it to a workspace or storage account.
- `sort by` defaults to **descending**; `take` returns arbitrary rows, not "the first".
- Stream to a third-party SIEM → **Event Hubs**. Cheap long-term archive → **Storage account**.

## Self-test
1. A Key Vault is deployed but you can't find any access logs in Log Analytics. Why?
2. Which table tells you whether an agent-connected VM is still reporting?
3. Write a query that counts events by `Computer` for the last 24 hours.
4. You must collect the Windows System event log from 50 VMs. What two components do you configure?
5. What is the difference between `take 10` and `top 10 by X`?
6. Where would you look to find out who changed an NSG rule yesterday?

<details><summary>Answers</summary>

1. Resource logs aren't collected by default — no diagnostic setting sends them to the workspace.
2. `Heartbeat`.
3. `Event | where TimeGenerated > ago(24h) | summarize count() by Computer`
4. Azure Monitor Agent on the VMs and a Data Collection Rule defining the event log and destination workspace.
5. `take` returns any 10 rows with no ordering; `top` sorts by X and returns the highest 10.
6. The Activity log (or `AzureActivity` in a workspace).
</details>
