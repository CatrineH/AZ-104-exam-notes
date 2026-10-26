# Azure Site Recovery (ASR)

## What it is
A **disaster recovery (DR)** service. It continuously **replicates** workloads from a primary location to a secondary location, and lets you **fail over** when the primary goes down, then **fail back** when it's healthy.

| | Azure Backup | Azure Site Recovery |
|---|---|---|
| Goal | Recover **data** | Recover **service availability** |
| How | Scheduled point-in-time copies | Continuous replication |
| Typical RPO | Hours (daily backups) | Seconds–minutes |
| Typical RTO | Hours | Minutes |
| Protects against | Deletion, corruption, ransomware | Regional/site outage |

Both live in a **Recovery Services vault**. They're complementary — production systems usually use both.

### RPO and RTO (always on the exam)
- **RPO – Recovery Point Objective**: how much *data* (measured in time) you can afford to lose. "We can lose at most 15 minutes of data."
- **RTO – Recovery Time Objective**: how long you can be *down*. "We must be running again within 2 hours."

## Supported scenarios
| Source | Target |
|---|---|
| Azure VM (region A) | Azure (region B) — **Azure-to-Azure** |
| Azure VM (zone 1) | Azure (zone 2) — zone-to-zone |
| On-prem **VMware** VMs / **physical** servers | Azure |
| On-prem **Hyper-V** VMs | Azure |

For AZ-104, focus on **Azure-to-Azure**.

## Key components / terms
- **Recovery Services vault** – must be in the **target (DR) region**, not the source region. (Opposite of Backup!)
- **Mobility service extension** – installed on Azure VMs automatically when replication is enabled; captures disk writes.
- **Cache storage account** – in the **source** region; writes are staged here before being sent to the target.
- **Replica managed disks** – in the target region.
- **Target resources** – target resource group, VNet, subnet, availability set/zone. Created automatically if you don't choose.
- **Replication policy** – recovery point retention (e.g., 24 h–15 days) and **app-consistent snapshot frequency**.
- **Recovery points**
  - **Crash-consistent** – like pulling the power; taken every few minutes.
  - **App-consistent** – applications flushed (VSS), taken at the configured frequency (e.g., every 4 h).
- **Multi-VM consistency** – VMs in a replication group share consistent recovery points (for multi-tier apps).

## Recovery plans
A recovery plan groups VMs and orders the failover:
- **Groups** fail over in sequence (Group 1 = databases, Group 2 = app tier, Group 3 = web tier).
- Add **Azure Automation runbooks** and **manual actions** before/after groups (e.g., update DNS, attach a public IP to the LB).

## Failover types
| Type | When | Effect |
|---|---|---|
| **Test failover** | DR drills | Creates VMs in an **isolated VNet** — no impact on production or replication. Clean up afterwards. |
| **Planned failover** | Known event (maintenance, migration) | Source shut down first → **zero data loss**. (Mainly for on-prem → Azure.) |
| **Unplanned failover** | Real outage | Uses latest available recovery point → possible data loss up to RPO. |

### Failover lifecycle (Azure-to-Azure)
1. **Enable replication** (initial replication, then continuous)
2. **Test failover** → clean up test failover
3. **Failover** to the secondary region (choose recovery point: latest, latest processed, latest app-consistent, custom)
4. **Commit** the failover
5. **Re-protect** – start replicating from secondary back to primary
6. **Fail back** to primary when ready → re-protect again

## Networking considerations
- Map source VNet → target VNet. Keep **IP addresses** consistent if possible (same subnet range in target VNet allows retaining IPs).
- Use a **different address space** for the test failover VNet so it's isolated.
- Plan DNS, public IPs, load balancers and NSGs in the target region (recovery plan scripts help).
- Outbound connectivity: VMs must reach ASR service endpoints (via service tags like `AzureSiteRecovery`, `Storage`, `AzureActiveDirectory`) — blocked outbound traffic is a common replication failure.

## Portal steps (Azure-to-Azure)
VM → **Disaster recovery** (under Operations) → choose target region → review/customize target settings → **Review + Start replication**. That's literally it for a single VM.

## CLI / PowerShell
Azure-to-Azure setup via PowerShell uses `Az.RecoveryServices` cmdlets (fabrics, protection containers, mappings) — it's long. Know the portal flow for the exam, and recognize names like:
```powershell
New-AzRecoveryServicesAsrFabric
New-AzRecoveryServicesAsrProtectionContainerMapping
New-AzRecoveryServicesAsrReplicationProtectedItem
Start-AzRecoveryServicesAsrTestFailoverJob
Start-AzRecoveryServicesAsrUnplannedFailoverJob
Update-AzRecoveryServicesAsrProtectionDirection   # re-protect
```

## Exam traps
- Vault location for ASR → **target (DR) region**. Vault location for Backup → **same region as the VM**.
- DR drill with no production impact → **Test failover** into an isolated VNet.
- Bring up DB first, then app, then web → **Recovery plan** with ordered groups.
- Need to run a script after failover (e.g., change DNS) → **runbook in the recovery plan**.
- After failover to region B, you must **re-protect** before you can fail back.
- "Lose no more than 5 minutes of data" = RPO; "be back within 1 hour" = RTO.
- Backup alone doesn't meet minutes-level RTO for a regional outage — that's ASR.

## Self-test
1. Your VMs run in Norway East and DR is Norway West. Where do you create the vault for ASR?
2. Define RPO and RTO in one sentence each.
3. Management wants a quarterly DR test without touching production. Which feature?
4. A three-tier app must fail over in the order SQL → app → web, and DNS must be updated at the end. What do you build?
5. After a successful failover, what step starts replication back towards the original region?

<details><summary>Answers</summary>

1. Norway West (the target region).
2. RPO = maximum acceptable data loss measured in time; RTO = maximum acceptable downtime.
3. Test failover (into an isolated VNet).
4. A recovery plan with three ordered groups and an Automation runbook (or manual action) at the end.
5. Re-protect.
</details>
