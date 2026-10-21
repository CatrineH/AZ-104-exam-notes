# Azure App Service

AZ-104 domain: Deploy and manage Azure compute resources - App services concept
Date learned: 2026-10-12

[← Back to README](../README.md)

## In one line
**App Service** hosts web apps and APIs as **PaaS**: you pay for an **App Service plan** (the servers), and your apps run on it while Microsoft handles the OS and runtime.

## When to use it (and when not)
| Need | Use |
|---|---|
| Web app or API in .NET, Node, Python, Java, PHP – no servers to patch | **App Service** |
| Staging + zero-downtime swap, custom domains, TLS, backups | **App Service** (Standard+ for slots) |
| Full control of the OS, custom software, legacy app | **Virtual machine** |
| Containers that scale on HTTP/queues down to zero | **Container Apps** |
| Small event-driven code | **Azure Functions** |

---

## The big picture

App Service is where I host **web apps and APIs** without managing servers.

- **Microsoft manages:** the servers, the operating system, patching, and the runtime (Node, .NET, Python and so on)
- **I manage:** my code, my settings, my secrets, and who can reach the app

Think of it as renting a fully furnished flat. I bring my belongings (the code), and the landlord looks after the building (the servers).

---

## 1. App Service plan = the servers

The **plan** is the group of servers I pay for. My **apps** run on those servers.

```
Plan (the servers)
  Server 1: app-a, app-b
  Server 2: app-a, app-b
```

Remember three things:

1. Every app in the plan runs on **every** server in the plan.
2. I pay for the **plan**, not for each app.
3. An app can only move to another plan in the **same resource group, region and OS**.

---

## 2. Tiers: what I need to remember

I don't need the whole table. These five lines cover most exam questions.

| Tier | Remember it as |
|---|---|
| Free / Shared | Testing only. No custom domain TLS, no scaling. |
| **Basic** | The first "real" tier: custom domain with TLS, backups, VNet integration, private endpoint. Max 3 servers. |
| **Standard** | Adds **deployment slots (5)** and **autoscale**. Max 10 servers. |
| Premium v3 | More of everything: 20 slots, max 30 servers. |
| Isolated v2 | Runs inside my own VNet, single-tenant. |

The two lines to learn by heart:

- **Basic** = lowest tier for TLS, backups and networking
- **Standard** = lowest tier for slots and autoscale

---

## 3. Scaling

| | Scale **up** | Scale **out** |
|---|---|---|
| What changes | Bigger servers (a higher tier) | More servers |
| Example | B1 to S1 | 1 server to 3 servers |
| Gives me new features? | **Yes** (for example slots) | No |

**Autoscale** (Standard and higher) adds or removes servers automatically, based on rules I write.

Example: add 1 server when CPU is above 70%, and remove 1 when CPU is below 30%.

- Always pair a scale-out rule with a scale-in rule.
- Set a minimum and a maximum number of servers.

---

## 4. Deployment slots

A slot is a **second copy of my app** with its own web address. The most common use is a **staging** slot.

```
Production slot   -> my users      (version 1)
Staging slot      -> I test here   (version 2)
```

**Swap** = swap staging and production. Users get version 2 with no downtime, because Azure warms it up first.

**Roll back** = swap again. Don't restore a backup.

### What moves during a swap?

- **Moves:** the code and normal app settings
- **Stays with the slot:** custom domain, TLS certificate, scaling, network settings, and any setting marked **"deployment slot setting"**

Memory hook: *the code moves; the address and security stay.*

---

## 5. App settings

- App settings reach my code as **environment variables**.
- Saving a setting **restarts the app**.
- Store secrets in **Key Vault** and reference them from the setting.
- Mark a setting as a **deployment slot setting** if it must not move during a swap (for example a database connection for staging).

---

## 6. Custom domain and HTTPS

To add my own domain, such as `www.catrineh.com`:

1. Create the DNS records:
   - **CNAME** `www` pointing to `<app>.azurewebsites.net`
   - **TXT** `asuid.www` containing the verification ID
   - For a root domain (`catrineh.com`), use an **A record** instead of the CNAME.
2. Add the domain to the app.
3. Add a certificate. The **free managed certificate** is the simplest option, but it doesn't support wildcards.
4. Turn on **HTTPS Only** and set the **minimum TLS version to 1.2**.

The order matters: **DNS first, then certificate.**

---

## 7. Backup

- Basic tier and higher.
- **Automatic** backups run every hour and are kept for 30 days, with nothing to set up.
- **Custom** backups go to my own storage account.
- A restore stops the app. To avoid downtime, restore to a slot and then swap.
- Custom domains, TLS, slots and network settings are **not** restored.

---

## 8. Networking: in or out?

Always ask first: **is the traffic coming into the app, or going out of it?**

| Direction | Feature | What it does |
|---|---|---|
| **In** | Access restrictions | Allow or block IP addresses. Blocked users get **403**. |
| **In** | Private endpoint | Gives the app a private IP inside my VNet. |
| **Out** | VNet integration | Lets the app reach resources inside my VNet, such as a VM or database. Needs its own subnet (/26 recommended). |

The classic trap: VNet integration is **outbound only**. If users need to reach the app privately, use a **private endpoint**.

---

## 9. Logs

| I want to see... | Use |
|---|---|
| Messages my code writes | Application logging |
| HTTP requests and errors like 500 | Web server logging |
| Live output right now | Log stream |
| Help finding the problem | Diagnose and solve problems |

---

## 10. Top exam traps

| Wrong | Right |
|---|---|
| Scale out to get slots | **Scale up** to Standard |
| Restore a backup to roll back | **Swap again** |
| Add the certificate before the domain | **DNS records first** |
| VNet integration lets users in privately | It is outbound only. Use a **private endpoint**. |
| Moving to another resource group changes region | A move **never** changes the region |

---

## Quick practice (Azure CLI)

```bash
RG=catrine-appsvc-rg
PLAN=catrine-appsvc-plan
APP=catrine-appsvc-$RANDOM

# 1. Create a Basic plan and a web app
az group create -n $RG -l norwayeast
az appservice plan create -g $RG -n $PLAN --sku B1 --is-linux
az webapp create -g $RG -p $PLAN -n $APP --runtime "NODE:24-lts"

# 2. Add an app setting
az webapp config appsettings set -g $RG -n $APP --settings GREETING="Hello"

# 3. Try to create a slot (this fails on Basic. Why?)
az webapp deployment slot create -g $RG -n $APP --slot staging

# 4. Scale up to Standard, then create the slot again
az appservice plan update -g $RG -n $PLAN --sku S1
az webapp deployment slot create -g $RG -n $APP --slot staging

# 5. Swap staging into production, then swap back
az webapp deployment slot swap -g $RG -n $APP --slot staging --target-slot production

# 6. Force HTTPS and TLS 1.2
az webapp update -g $RG -n $APP --https-only true
az webapp config set -g $RG -n $APP --min-tls-version 1.2

# 7. Watch live logs
az webapp log tail -g $RG -n $APP

# 8. Clean up
az group delete -n $RG --yes --no-wait
```

---

## PowerShell equivalents

```powershell
$rg = "catrine-appsvc-rg"; $plan = "catrine-appsvc-plan"; $app = "catrine-appsvc-$(Get-Random)"

New-AzResourceGroup -Name $rg -Location norwayeast
New-AzAppServicePlan -ResourceGroupName $rg -Name $plan -Location norwayeast -Tier Basic -WorkerSize Small -Linux
New-AzWebApp -ResourceGroupName $rg -Name $app -Location norwayeast -AppServicePlan $plan

# App settings (replaces ALL settings – include the ones you want to keep)
Set-AzWebApp -ResourceGroupName $rg -Name $app -AppSettings @{ GREETING = "Hello" }

# Scale up to Standard, scale out to 2 instances
Set-AzAppServicePlan -ResourceGroupName $rg -Name $plan -Tier Standard
Set-AzAppServicePlan -ResourceGroupName $rg -Name $plan -NumberofWorkers 2

# Slots and swap
New-AzWebAppSlot -ResourceGroupName $rg -Name $app -Slot staging
Switch-AzWebAppSlot -ResourceGroupName $rg -Name $app -SourceSlotName staging -DestinationSlotName production

# HTTPS only
Set-AzWebApp -ResourceGroupName $rg -Name $app -HttpsOnly $true
```

---

## Troubleshooting (symptom → cause → fix)

| Symptom | Likely cause | Fix |
|---|---|---|
| **403 Forbidden** | **Access restrictions** block your IP – or the app is **stopped** ("This web app is stopped") | Check Networking → Access restrictions, start the app |
| **503 Service Unavailable** | App crashed/restarting, or **Free/Shared quota** used up | Log stream, Diagnose and solve problems, scale up |
| **500 Internal Server Error** | Bug or missing setting in your code | **Application logging** + **log stream** |
| "Application Error" for a container app | Wrong port – container listens on another port | Set app setting **`WEBSITES_PORT`** |
| Can't create a slot / autoscale greyed out | Tier is **Free/Basic** | **Scale up** to Standard |
| Can't scale **down** to Basic | Slots exist (Basic has none) | Delete the slots first |
| Custom domain validation fails | **TXT `asuid`** or CNAME/A record missing, or DNS not propagated | Add records, wait, re-validate |
| Staging setting showed up in production after swap | Setting wasn't a **deployment slot setting** | Tick "Deployment slot setting" |
| Key Vault reference shows a red error | App's **managed identity** lacks **Key Vault Secrets User**, or KV firewall blocks it | Enable identity, assign role, allow access |
| App can't reach a VM/database in the VNet | **VNet integration** missing, NSG/DNS on the target | Add VNet integration (own subnet), check NSG and private DNS |
| Users can't reach the app privately | VNet integration is **outbound only** | **Private endpoint** |
| Can't move app to another plan | Plan is in another region / RG / OS | Same RG, region, and OS – or redeploy |

---

## Break it and fix it
Use the lab from "Quick practice". Note the error, then fix it.

| # | Break it | What you should see | Fix |
|---|---|---|---|
| 1 | Add an **access restriction** that denies your IP (priority 100) | **403** in the browser | Remove the rule – or allow your IP with a lower number |
| 2 | `az webapp stop` | **403 – This web app is stopped** | `az webapp start` |
| 3 | Set `DB=staging-db` in the staging slot **without** slot setting, swap | Production now has `DB=staging-db` | Mark as **deployment slot setting**, set again, swap |
| 4 | Try `az appservice plan update --sku B1` while the staging slot exists | Error – Basic has no slots | Delete the slot first |
| 5 | Tail logs while you browse a page that doesn't exist | 404 lines in the **web server log** | – (practice reading logs) |

---

## Test myself

1. What is the difference between a plan and an app?
2. Which tier is the lowest with deployment slots?
3. Scale up or scale out: which one gives me new features?
4. How do I roll back a bad release?
5. Does the custom domain move during a swap?
6. Which DNS record do I use for a root domain like `contoso.com`?
7. Users must reach the app privately from my VNet. Which feature do I use?
8. Which log shows HTTP 500 errors?

<details>
<summary>Answers</summary>

1. The plan is the servers I pay for. The app is the code that runs on them.
2. Standard.
3. Scale up.
4. Swap the slots again.
5. No. It stays with the slot.
6. An A record (plus a TXT `asuid` record).
7. A private endpoint.
8. Web server logging.

</details>
