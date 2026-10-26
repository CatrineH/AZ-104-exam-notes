# Alerts & Action Groups

## How an alert is built
An **alert rule** = **scope** + **condition** + **actions** + **details**.

```
Scope (which resource)  →  Condition (signal + logic)  →  Action group (who/what to notify)
                                                        ↘  Alert processing rules (suppress / add actions)
```

- **Scope** – resource(s), resource group or subscription being monitored.
- **Condition** – the signal (metric, log query, activity log event) plus threshold and evaluation frequency.
- **Action group** – reusable set of notifications and automated actions.
- **Details** – name, **severity**, description.

### Severity levels
| Sev | Meaning |
|---|---|
| 0 | Critical |
| 1 | Error |
| 2 | Warning |
| 3 | Informational |
| 4 | Verbose |

> Lower number = worse. Sev 0 is the "wake someone up" level.

## Alert types
| Type | Signal | Typical use |
|---|---|---|
| **Metric alert** | Platform/custom metrics | CPU > 80 % for 5 min |
| **Log search alert** | KQL query on a workspace | > 5 failed logins in 10 min |
| **Activity log alert** | Activity log events | Someone deleted a VM / a role assignment was created |
| **Service Health alert** | Azure Service Health (activity log category) | Azure outage or planned maintenance in your region |
| **Resource Health alert** | Resource Health | *Your* VM became unavailable |
| **Smart detection** | Application Insights | Abnormal failure rate in an app |
| **Prometheus alert** | Managed Prometheus | AKS / container metrics |

### Metric alert details
- **Static threshold** – fixed number (CPU > 80).
- **Dynamic threshold** – machine learning learns normal behavior and alerts on deviations. Good when "normal" changes by time of day.
- **Multi-resource** – one rule can watch many VMs of the same type in the same region.
- Evaluated every 1 min by default; *aggregation granularity (period)* defines the window.

### Log search alert details
- Query runs on a schedule (frequency) over a time window.
- Alert logic: number of results, or measure a column (e.g., avg CPU) with **dimensions** to split per computer.
- Costs more than metric alerts and has more delay — use metric alerts when a metric exists.

### Stateful vs stateless
- **Stateful** – fires once, then auto-resolves when the condition clears (metric alerts by default; log alerts optionally).
- **Stateless** – fires every time the condition is met (activity log alerts).

## Action groups
A reusable list of *who to notify* and *what to run*. Many alert rules can share one action group.

**Notification types**
- Email Azure Resource Manager Role (e.g., all Owners of the subscription)
- Email, SMS, Azure mobile app push, Voice call

**Action types**
- **Automation runbook**
- **Azure Function**
- **Logic App** (e.g., post to Teams, create a ticket)
- **Webhook / Secure webhook**
- **ITSM** connector (ServiceNow etc.)
- **Event Hubs**

Rate limits exist (e.g., SMS/voice no more than about one every 5 minutes per number; email has limits too) — so alert storms don't spam endlessly.

Action groups are **global** resources by default, but you can pin them to a region for data residency.

## Alert processing rules
Applied *after* an alert fires. Use them to:
- **Suppress** notifications during planned maintenance (e.g., every Saturday 22:00–02:00).
- **Add action groups** to many alerts at once (e.g., add the on-call group to every Sev 0 in a subscription).

> Suppressing doesn't stop the alert from firing — the alert is still recorded; only the actions are skipped.

## Alert lifecycle (user response state)
**New → Acknowledged → Closed**. The *monitor condition* (Fired / Resolved) is set by the system; the *user response* is set by you.

## CLI / PowerShell
```bash
# Action group with an email receiver
az monitor action-group create -g rg-monitor -n ag-ops --short-name ops \
  --action email cat cat@example.com

# Metric alert: CPU > 80% avg over 5 minutes
az monitor metrics alert create -g rg-monitor -n cpu-high \
  --scopes /subscriptions/<sub>/resourceGroups/rg-vm/providers/Microsoft.Compute/virtualMachines/vm01 \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m --evaluation-frequency 1m \
  --severity 2 --action ag-ops

# Activity log alert: VM deleted
az monitor activity-log alert create -g rg-monitor -n vm-deleted \
  --condition category=Administrative and operationName=Microsoft.Compute/virtualMachines/delete \
  --action-group ag-ops
```
```powershell
$email = New-AzActionGroupEmailReceiverObject -Name cat -EmailAddress cat@example.com
New-AzActionGroup -ResourceGroupName rg-monitor -Name ag-ops -ShortName ops -Location global -EmailReceiver $email
```

## Exam traps
- "Notify when Azure has an outage affecting my region" → **Service Health alert**, not a metric alert.
- "Notify when *my specific VM* is unavailable due to the platform" → **Resource Health alert**.
- "Notify when someone deletes a resource / creates a role assignment" → **Activity log alert**.
- "Stop notifications during the maintenance window" → **Alert processing rule** (suppression), not deleting/disabling the rule.
- "Notify all subscription Owners without listing names" → action group with **Email ARM Role**.
- Automated remediation (restart a service) → action group with **Automation runbook**, **Function** or **Logic App**.
- Short name of an action group is max 12 characters (used in SMS/email).

## Self-test
1. CPU on a VM spikes every morning at 08:00 by design. You want to alert only on *unusual* spikes. Which threshold type?
2. You want one email when a disk is nearly full, and an automatic "resolved" when it's fixed. Stateful or stateless?
3. Every alert in the subscription with Sev 0 must also page the on-call team. How do you do this without editing every rule?
4. Which alert type uses a KQL query?
5. Which severity is "Warning"?

<details><summary>Answers</summary>

1. Dynamic threshold.
2. Stateful (metric alerts auto-resolve).
3. Alert processing rule that adds the on-call action group, filtered on Sev 0.
4. Log search alert.
5. Sev 2.
</details>
