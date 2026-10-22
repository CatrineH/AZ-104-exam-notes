# Module assessment: Microsoft Entra ID

> **Active Directory provides the core service of identity management. AD DS is the traditional on-premises solution, whereas Microsoft Entra ID is the cloud-based solution. Microsoft Entra ID is frequently adopted at first to facilitate authentication for cloud-based apps, but is capable of providing authentication services for the entire infrastructure. While they provide similar solutions, each offer different capability and are often used together to provide a best-of-breed solution. Microsoft Entra ID is offered as a free service, with paid tiers for additional capabilities, depending on an organization's needs.*

---

## 1. Detecting irregular sign-in activity

Your company is using Microsoft Entra ID to manage cloud-based resources and needs to identify irregular sign-in activities. Which feature of Microsoft Entra ID should be utilized for this purpose?

- A. Conditional Access
- B. Identity Protection
- C. Application Proxy

<details>
<summary>Show answer</summary>

**Correct: B. Identity Protection**

| Option | What it is | Why right or wrong |
|---|---|---|
| **A. Conditional Access** | The policy engine: "if this user/app/location/device/risk, then require MFA, block, or allow". | Wrong. It *enforces* decisions but does not *detect* irregular sign-ins itself. It can consume risk signals from Identity Protection. |
| **B. Identity Protection** | Uses Microsoft's signals and machine learning to detect risky sign-ins (anonymous IP, impossible travel, unfamiliar sign-in properties, leaked credentials) and risky users. | Right. Detecting irregular sign-in activity is exactly its job. Requires P2. |
| **C. Application Proxy** | Publishes on-premises web apps securely to remote users through Entra ID, without opening inbound firewall ports. | Wrong. It is about remote access to on-prem apps, not sign-in risk. |

**Remember:** Identity Protection *detects*, Conditional Access *enforces*.
</details>

---

## 2. P2-only feature

Which feature is unique to the Microsoft Entra ID P2 plan and not included in the P1 plan or the free version?

- A. Cloud App Discovery
- B. Self-service group management
- C. Microsoft Entra Identity Protection

<details>
<summary>Show answer</summary>

**Correct: C. Microsoft Entra Identity Protection**

| Option | What it is | Why right or wrong |
|---|---|---|
| **A. Cloud App Discovery** | Finds which cloud (SaaS) apps users actually use, including shadow IT. | Wrong. Included from P1. |
| **B. Self-service group management** | Lets users create groups and request to join groups, with owners approving. | Wrong. Available in Free/P1, not P2-only. |
| **C. Identity Protection** | Risk-based detection of risky users and sign-ins, plus risk policies. | Right. A P2 feature. |

**Remember the P2 pair:** Identity Protection and Privileged Identity Management (PIM).
</details>

---

## 3. Advantage of Entra ID over AD DS for cloud apps

An organization is considering transitioning from its on-premises Active Directory Domain Services (AD DS) to Microsoft Entra ID for its cloud-based applications. Which feature of Microsoft Entra ID would provide an advantage over AD DS in this scenario?

- A. Hierarchical structure with Organizational Units
- B. Support for multi-factor authentication
- C. Use of Kerberos protocol for authentication

<details>
<summary>Show answer</summary>

**Correct: B. Support for multi-factor authentication**

| Option | What it is | Why right or wrong |
|---|---|---|
| **A. OUs in a hierarchy** | Containers in AD DS used to organise objects and link Group Policy. | Wrong. That is an AD DS feature. Entra ID has a flat structure (no OUs). |
| **B. Multi-factor authentication** | A second factor (Authenticator app, FIDO2 key, SMS) on top of the password, built into the cloud identity service. | Right. Native, cloud-scale MFA for web/SaaS apps is an Entra ID strength. AD DS has no built-in MFA. |
| **C. Kerberos** | Ticket-based authentication protocol used on-premises in a domain. | Wrong. That is what AD DS uses. Entra ID uses web protocols: SAML, WS-Federation, OAuth 2.0, OpenID Connect. |

</details>

---

## 4. User risk and sign-in risk policies

Your company needs a feature that allows you to define user risk policies and sign-in policies. Which Microsoft Entra ID plan should you consider?

- A. Microsoft Entra ID P1
- B. Microsoft Entra ID P2
- C. Microsoft Entra ID Free

<details>
<summary>Show answer</summary>

**Correct: B. Microsoft Entra ID P2**

| Option | What it includes | Why right or wrong |
|---|---|---|
| **A. P1** | Conditional Access, dynamic groups, SSPR with writeback, hybrid features, Cloud App Discovery. | Wrong. P1 has Conditional Access, but not the *risk* signals (user risk, sign-in risk). |
| **B. P2** | Everything in P1 plus Identity Protection (risk policies) and PIM. | Right. User risk and sign-in risk policies come from Identity Protection, which is P2. |
| **C. Free** | Users, groups, basic reports, SSO for apps, security defaults. | Wrong. No risk policies, no Conditional Access. |

**Exam keyword:** "risk" in the question → P2.
</details>

---

## 5. Organisational structure: Entra ID vs AD DS

What is a significant difference between how organizational structures are managed in Microsoft Entra ID compared to AD DS?

- A. Microsoft Entra ID uses a flat structure without OUs, while AD DS uses a hierarchical structure with OUs.
- B. Microsoft Entra ID and AD DS use Group Policies to manage organizational structures.
- C. Both Microsoft Entra ID and AD DS use hierarchical structures with Organizational Units (OUs).

<details>
<summary>Show answer</summary>

**Correct: A. Flat structure in Entra ID, hierarchy with OUs in AD DS**

| Option | Why right or wrong |
|---|---|
| **A** | Right. Entra ID has a flat directory of users and groups (you can scope admin roles with *administrative units*, but there are no OUs). AD DS is built as forests, domains and nested OUs. |
| **B** | Wrong. Group Policy (GPOs) is an AD DS feature. Entra ID has no GPOs; device settings are managed with Microsoft Intune instead. |
| **C** | Wrong. Only AD DS has OUs. |

</details>

---

## 6. A similarity between Entra ID and AD DS

Which statement correctly identifies a similarity between Microsoft Entra ID and Active Directory Domain Services (AD DS)?

- A. Both provide directory services for managing user identities and access.
- B. Both use LDAP for querying directory services.
- C. Both use the same authentication protocols.

<details>
<summary>Show answer</summary>

**Correct: A. Both provide directory services for managing identities and access**

| Option | Why right or wrong |
|---|---|
| **A** | Right. At heart, both are identity directories: they store users and groups and control access to resources. |
| **B** | Wrong. AD DS is queried with LDAP. Entra ID is queried through REST APIs (Microsoft Graph). If you need LDAP in Azure, that is a separate service: Microsoft Entra Domain Services. |
| **C** | Wrong. AD DS uses Kerberos and NTLM. Entra ID uses SAML, WS-Federation, OAuth 2.0 and OpenID Connect. |

</details>

---

## 7. Authentication across on-premises and cloud

An organization using AD DS is planning to extend its on-premises infrastructure to Microsoft Entra ID. What feature of Microsoft Entra ID can help manage user authentication across both environments?

- A. Implementing Kerberos authentication
- B. Enabling federation between organizations
- C. Using Group Policy Objects (GPOs)

<details>
<summary>Show answer</summary>

**Correct: B. Enabling federation**

| Option | Why right or wrong |
|---|---|
| **A. Kerberos** | Wrong. Kerberos is the on-premises AD DS protocol; it does not bridge on-prem and Entra ID by itself. |
| **B. Federation** | Right. Federation sets up a trust so an identity provider (for example AD FS on-prem) authenticates users, and Entra ID accepts that result. Users sign in with one identity in both environments. |
| **C. GPOs** | Wrong. GPOs configure computers and users inside an AD DS domain. They do not handle cloud authentication. |

**Good to know (not an option here):** in practice, hybrid identity is usually set up with **Microsoft Entra Connect** (or Connect cloud sync), using **password hash sync** (most common, simplest), **pass-through authentication**, or **federation**.
</details>

---

## 8. Multi-tenancy

How does Microsoft Entra ID handle multi-tenancy compared to AD DS?

- A. Microsoft Entra ID is inherently multi-tenant, allowing multiple directory instances across organizations.
- B. AD DS supports multiple tenants through its hierarchical structure.
- C. Both Microsoft Entra ID and AD DS are inherently single-tenant services.

<details>
<summary>Show answer</summary>

**Correct: A. Entra ID is inherently multi-tenant**

| Option | Why right or wrong |
|---|---|
| **A** | Right. Entra ID is a global cloud service where each organisation gets its own isolated tenant. Tenants can also collaborate, for example with B2B guest users. |
| **B** | Wrong. AD DS forests and domains organise *one* organisation's objects. That hierarchy is not multi-tenancy. |
| **C** | Wrong. AD DS is effectively single-organisation, but Entra ID is multi-tenant by design. |

</details>

---

## 9. Managing administrator privileges (P2)

Which feature provided by Microsoft Entra ID P2 enhances security by managing privileges for administrators?

- A. Microsoft Entra Privileged Identity Management
- B. Self-service password reset with writeback
- C. Advanced security reports and alerts

<details>
<summary>Show answer</summary>

**Correct: A. Privileged Identity Management (PIM)**

| Option | What it is | Why right or wrong |
|---|---|---|
| **A. PIM** | Just-in-time admin access: users are *eligible* for a role and activate it for a limited time, optionally with approval, MFA and a justification. Includes access reviews and audit history. | Right. Its purpose is managing admin privileges. P2. |
| **B. SSPR with writeback** | Users reset their own password in the cloud and it is written back to on-prem AD DS. | Wrong. About passwords, not admin privileges. Available from P1. |
| **C. Advanced security reports and alerts** | Reports on sign-ins and suspicious activity. | Wrong. It reports and alerts; it does not manage who holds admin rights. |

**Remember:** PIM = "eligible, not permanent" admin roles.
</details>

---

## Quick summary

| Need | Feature | Plan |
|---|---|---|
| Detect risky sign-ins / users | Identity Protection | P2 |
| User risk and sign-in risk policies | Identity Protection | P2 |
| Just-in-time admin roles | PIM | P2 |
| Conditional Access, dynamic groups, SSPR writeback, Cloud App Discovery | — | P1 |
| Users, groups, SSO, security defaults | — | Free |

| | AD DS | Entra ID |
|---|---|---|
| Structure | Hierarchical (forests, domains, OUs) | Flat (users, groups, admin units) |
| Policy | Group Policy (GPOs) | Conditional Access, Intune |
| Protocols | Kerberos, NTLM, LDAP | SAML, WS-Fed, OAuth 2.0, OIDC, Microsoft Graph |
| Tenancy | Single organisation | Multi-tenant cloud service |
