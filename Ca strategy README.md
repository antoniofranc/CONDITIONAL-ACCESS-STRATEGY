# Conditional Access Strategy
 📄 [View Full Document on Google Docs](https://docs.google.com/document/d/1s5iz44bpKn7-VXx_OgaJfAlKl7ORzl-Z/)

**Baseline & Admin Policy Design — Microsoft Entra ID**  
Final IAM Project — Day 2 Portfolio Artifact
 
> Prepared as a presentation to a Security / IAM Lead  
> May 2026
 
---
 
## Table of Contents
 
1. [Executive Summary](#1-executive-summary)
2. [What Is Conditional Access?](#2-what-is-conditional-access)
   - [Policy Anatomy](#21-policy-anatomy)
   - [Key Concepts](#22-key-concepts-for-this-strategy)
3. [Tier 1 — Baseline Policies (All Users)](#3-tier-1--baseline-policies-all-users)
   - [Policy 1 — Block Legacy Authentication](#31-policy-1--block-legacy-authentication)
   - [Policy 2 — Require MFA for All Users](#32-policy-2--require-mfa-for-all-users)
   - [Policy 3 — Secure Security Info Registration](#33-policy-3--secure-security-info-registration)
   - [Policy 4 — Require Compliant Device or MFA Fallback](#34-policy-4--require-compliant-device-or-mfa-fallback)
4. [Tier 2 — Admin Policies (Privileged Roles)](#4-tier-2--admin-policies-privileged-roles)
   - [Policy 5 — Phishing-Resistant MFA for Admins](#41-policy-5--phishing-resistant-mfa-for-admins)
   - [Policy 6 — Require MFA for Microsoft Admin Portals](#42-policy-6--require-mfa-for-microsoft-admin-portals)
   - [Policy 7 — Require Compliant Device for Admins (No Fallback)](#43-policy-7--require-compliant-device-for-admins-no-fallback)
   - [Policy 8 — Block Unknown or Unsupported Device Platforms](#44-policy-8--block-unknown-or-unsupported-device-platforms)
5. [Deployment Approach](#5-deployment-approach)
6. [Design Decisions & Rationale](#6-design-decisions--rationale)
7. [Monitoring & Ongoing Maintenance](#7-monitoring--ongoing-maintenance)
---

# Conditional Access Strategy
 
**Baseline & Admin Policy Design — Microsoft Entra ID**  
Final IAM Project — Day 2 Portfolio Artifact
 
> Prepared as a presentation to a Security / IAM Lead  
> May 2026
 
---
 
## 1. Executive Summary
 
This document presents a Conditional Access (CA) strategy for Microsoft Entra ID, structured as a presentation to a security or IAM lead. It defines two policy tiers:
 
- **Baseline layer** — protecting all users
- **Elevated layer** — stricter controls protecting privileged administrator accounts
The strategy is built from Microsoft's Secure Foundation template recommendations and follows a **Zero Trust model** — every sign-in is verified regardless of network location, device, or user familiarity. Policies are deployed in **report-only mode first** to assess impact before enforcement.
 
The goal is to reduce the attack surface against the three most common identity attack vectors:
 
1. Credential phishing
2. Legacy authentication abuse
3. Lateral movement via compromised admin accounts
---
 
## 2. What Is Conditional Access?
 
Conditional Access is the **policy engine in Microsoft Entra ID** that makes real-time access decisions based on identity signals. Rather than a simple password gate, it evaluates a set of conditions on every sign-in and enforces controls before granting or blocking access.
 
Think of it as an **if-then policy engine**:
 
> *IF a user signs in from an unmanaged device outside the corporate network, THEN require MFA and limit the session to read-only.*
 
This allows security to be context-aware rather than binary.

### 2.1 Policy Anatomy
 
| Component | Description |
|---|---|
| **Assignments — Users** | Who the policy applies to: all users, specific groups, or directory roles (e.g., Global Administrator) |
| **Assignments — Cloud apps** | Which apps trigger the policy: all cloud apps, Microsoft admin portals, specific SaaS apps |
| **Conditions** | Additional signals: sign-in risk level, device platform, device compliance state, location (named or not) |
| **Grant controls** | What is required to proceed: MFA, compliant device, hybrid-joined device, terms of use |
| **Session controls** | Ongoing access limits: sign-in frequency, persistent browser session, app-enforced restrictions |
 
### 2.2 Key Concepts for This Strategy
 
- **Named locations** — Define trusted IP ranges (e.g., corporate office). Used to restrict MFA registration to known networks.
- **Report-only mode** — Policy is evaluated but not enforced. Sign-in logs show what *would have* happened. Always deploy in report-only first.
- **Break-glass accounts** — Emergency admin accounts that must be excluded from all CA policies to prevent lockout. Monitor their usage with alerts.
- **Workload identities** — Service principals and managed identities are not covered by user-targeted CA policies. Use CA for workload identities separately.
---
 
## 3. Tier 1 — Baseline Policies (All Users)
 
These four policies form the **secure foundation**. Microsoft recommends deploying them as a group. They apply to every user in the tenant regardless of role, department, or device state. No user is exempt except break-glass accounts and service principals.
 
### 3.1 Policy 1 — Block Legacy Authentication
 
Legacy authentication protocols (`SMTP AUTH`, `POP3`, `IMAP`, basic auth) cannot be intercepted by Conditional Access — they bypass MFA entirely. This policy closes that gap permanently.
 
| Setting | Value |
|---|---|
| Users | All users |
| Cloud apps | All cloud apps |
| Conditions — Client apps | Exchange ActiveSync clients + Other clients (legacy protocols) |
| Grant | Block access |
| Rationale | Legacy auth is responsible for the majority of password spray attacks. No legitimate modern workflow requires it. |
 
> **Screenshot:** CA policy — Block legacy authentication — Conditions tab showing client apps selected
 <img width="283" height="469" alt="image" src="https://github.com/user-attachments/assets/2e85417b-cbe7-44a3-9455-65db98703a9b" />

---

### 3.2 Policy 2 — Require MFA for All Users
 
MFA is the single most effective control against credential theft. This policy ensures every user must prove their identity with a second factor on every new sign-in, regardless of location or device.
 
| Setting | Value |
|---|---|
| Users | All users *(exclude break-glass accounts)* |
| Cloud apps | All cloud apps |
| Conditions | None — applies universally |
| Grant | Require authentication strength: Multifactor authentication |
| Rationale | 99.9% of account compromise attacks are stopped by MFA. This is non-negotiable for any modern tenant. |
 
> **Screenshot:** CA policy — Require MFA — Grant tab showing MFA requirement
<img width="308" height="360" alt="image" src="https://github.com/user-attachments/assets/6e9b03ff-d799-437b-9718-eeabbd0b9bc6" />

---
 
### 3.3 Policy 3 — Secure Security Info Registration
 
This policy prevents an attacker who has stolen a password from immediately registering their own MFA method and locking out the legitimate user. MFA registration is restricted to trusted locations or sessions where MFA is already satisfied.
 
| Setting | Value |
|---|---|
| Users | All users *(exclude break-glass accounts)* |
| Cloud apps | Microsoft Azure Information Protection (registration app) |
| Conditions — Locations | Any location **EXCEPT** named trusted locations |
| Grant | Require MFA (or existing MFA session) |
| Rationale | Without this, a phished user's account can be fully taken over before they even notice. |
 
> **Screenshot:** CA policy — Secure MFA registration — Locations condition excluding named trusted locations
<img width="614" height="512" alt="image" src="https://github.com/user-attachments/assets/4fd1f79a-198a-4821-a736-04846d93dcb8" />

---
 
### 3.4 Policy 4 — Require Compliant Device or MFA Fallback
 
Managed, Intune-enrolled devices meet a baseline security posture. This policy encourages device enrollment by granting seamless access to compliant devices, while still allowing unmanaged devices access if they complete MFA. Over time, the MFA fallback can be removed to enforce full device compliance.
 
| Setting | Value |
|---|---|
| Users | All users *(exclude break-glass accounts)* |
| Cloud apps | All cloud apps |
| Grant — operator | Require **one of**: compliant device, hybrid-joined device, **OR** MFA |
| Session — unmanaged | App-enforced restrictions (read-only in SharePoint/Exchange) |
| Rationale | Unmanaged devices are a significant endpoint risk. This nudges the organization toward full MDM enrollment. |
 
> **Screenshot:** CA policy — Compliant device or MFA — Grant tab showing three options with OR operator
<img width="292" height="800" alt="image" src="https://github.com/user-attachments/assets/3be52dba-4425-4276-859b-0ac790ba5ee6" />


---
 
## 4. Tier 2 — Admin Policies (Privileged Roles)
 
Administrator accounts are the **highest-value targets** for attackers. A compromised Global Administrator can permanently backdoor a tenant, disable all security controls, and exfiltrate all data. The baseline policies apply to admins, but we layer on four additional stricter controls.
 
**Target roles include:**
- Global Administrator
- Privileged Role Administrator
- Security Administrator
- Exchange Administrator
- SharePoint Administrator
- Helpdesk Administrator
- Any other role with significant tenant-wide impact
---

### 4.1 Policy 5 — Phishing-Resistant MFA for Admins
 
Standard MFA (push notification or TOTP) is vulnerable to real-time phishing: an attacker can proxy a session and intercept the MFA approval. Phishing-resistant methods (**FIDO2 security keys** or **certificate-based authentication**) are bound to the origin and cannot be proxied.
 
| Setting | Value |
|---|---|
| Users | Members of privileged directory roles |
| Cloud apps | All cloud apps |
| Grant | Require authentication strength: **Phishing-resistant MFA** |
| Rationale | Admins are targeted by adversary-in-the-middle (AiTM) attacks specifically because their MFA is bypassable. FIDO2 eliminates this attack surface. |
 
> **Screenshot:** CA policy — Phishing-resistant MFA — Grant tab showing Authentication Strength: Phishing-resistant MFA
<img width="291" height="591" alt="image" src="https://github.com/user-attachments/assets/47cefb37-3e98-404b-824c-8414f713669d" />

---

### 4.2 Policy 6 — Require MFA for Microsoft Admin Portals
 
The Microsoft Entra admin center, Azure portal, Microsoft 365 admin center, and Exchange admin center are high-value targets. This policy specifically scopes to those applications and enforces MFA even if a broader policy would have been satisfied.
 
| Setting | Value |
|---|---|
| Users | All users *(note: applies to anyone accessing admin portals, not just admins)* |
| Cloud apps | Microsoft Admin Portals (Entra admin, Azure portal, M365 admin, Exchange admin) |
| Grant | Require MFA |
| Rationale | Admin portals are the blast radius. Even a non-admin who somehow navigates there should face MFA. |
 
> **Screenshot:** CA policy — MFA for admin portals — Cloud apps tab showing Microsoft Admin Portals selected

<img width="305" height="377" alt="image" src="https://github.com/user-attachments/assets/f5fcdfdb-c2ef-4225-9f3e-fd1a24086844" />

---

### 4.3 Policy 7 — Require Compliant Device for Admins (No Fallback)
 
Unlike the baseline policy which allows MFA as a fallback, admin accounts **must always use a compliant, Intune-enrolled device**. There is no MFA-only fallback. This ensures admin actions are taken from a known, managed, secured endpoint.
 
| Setting | Value |
|---|---|
| Users | Members of privileged directory roles |
| Cloud apps | All cloud apps |
| Grant | Require compliant device **OR** hybrid-joined device *(no MFA fallback)* |
| Rationale | An admin on an unmanaged personal laptop with malware is a critical risk even with MFA. Device posture matters as much as identity. |
 
> **Screenshot:** CA policy — Compliant device for admins — Grant tab showing compliant OR hybrid-joined, no MFA option
<img width="290" height="700" alt="image" src="https://github.com/user-attachments/assets/9c921b4d-d9d9-464a-9bff-7d8f2846529c" />
 
---

### 4.4 Policy 8 — Block Unknown or Unsupported Device Platforms
 
Devices running unsupported or unrecognized operating systems cannot receive Intune compliance policies and therefore cannot satisfy device compliance requirements. This policy blocks them outright for admin accounts.
 
| Setting | Value |
|---|---|
| Users | Members of privileged directory roles |
| Cloud apps | All cloud apps |
| Conditions — Device platforms | Any device — **EXCLUDE** Windows, macOS, iOS, Android |
| Grant | Block access |
| Rationale | Linux, ChromeOS, and unrecognized platforms cannot be MDM-enrolled into Intune. Blocking prevents compliance bypass via unusual OS. |
 
> **Screenshot:** CA policy — Block unknown platforms — Conditions showing excluded platforms
<img width="273" height="453" alt="image" src="https://github.com/user-attachments/assets/d60d5764-c4fe-495a-97d4-0bfc83832b64" />

---

## 5. Deployment Approach
 
A poorly deployed Conditional Access policy can lock users out of the tenant. The following phased approach minimizes risk while building toward full enforcement.
 
### 5.1 Deployment Phases
 
| Phase | Action | Duration |
|---|---|---|
| **1 — Prepare** | Create named locations, break-glass accounts, and emergency exclusion groups | Day 1 |
| **2 — Report-only** | Deploy all policies in report-only mode. Review sign-in logs daily. | Week 1–2 |
| **3 — Pilot** | Enable policies for a pilot group (IT staff). Monitor for false positives. | Week 2–3 |
| **4 — Rollout** | Enable policies tenant-wide for all users. Keep admins in pilot scope longer. | Week 3–4 |
| **5 — Admin tier** | Enable Tier 2 admin policies after confirming FIDO2 keys are issued. | Week 4–5 |
| **6 — Harden** | Remove MFA fallback from device compliance policy. Review monthly. | Ongoing |
 
> **Tip:** Always validate sign-in logs in **Entra ID → Monitoring → Sign-in logs** with the filter `Conditional Access = Report-only` before enabling any policy.
 
---

### 5.2 Pre-Deployment Checklist
 
- [ ] Create at least **two break-glass (emergency access) accounts** with long random passwords. Store credentials offline in a secure location. Exclude them from **ALL** CA policies.
- [ ] Create a **named location** in **Entra ID → Security → Named Locations** for your corporate IP ranges.
- [ ] Identify and document all **service accounts and service principals** that authenticate interactively. Exclude them from user-targeted policies and create workload identity CA policies instead.
- [ ] Confirm at least one global administrator has a **FIDO2 hardware key enrolled** before enabling the phishing-resistant MFA policy.
- [ ] Set up an alert in **Microsoft Sentinel** or **Entra ID → Diagnostic Settings** to notify on break-glass account sign-in.
---
 
### 5.3 Lab Steps — Building the Baseline
 
1. Navigate to **Entra ID → Protection → Conditional Access → Policies → New policy from templates**.
2. Select `Block legacy authentication`. Review settings. Click **Create policy** — it defaults to Report-only.
3. Repeat for:
   - `Require multifactor authentication for all users`
   - `Securing security info registration`
   - `Require compliant or Microsoft Entra hybrid joined device or MFA`
4. Sign in as a test user in **InPrivate mode**. Navigate to `portal.microsoft.com`.
5. Check **Entra ID → Monitoring → Sign-in logs**. Filter by the test user. Review the **Conditional Access** tab on the sign-in event — confirm each policy shows as `Report-only: Would have succeeded/failed`.
6. After confirming expected results, change policy state from `Report-only` to `On` for the **legacy auth block first** (lowest false positive risk), then MFA.
> **Screenshot:** Sign-in logs → Conditional Access tab — showing all 4 baseline policies evaluated in report-only mode

<img width="732" height="133" alt="image" src="https://github.com/user-attachments/assets/1f691f5f-fe7f-4e95-a615-2ae2bd1db157" />

---

### 5.4 Lab Steps — Building the Admin Tier
 
1. Create a new CA policy *(not from template)*. Name it `Require phishing-resistant MFA — admins`.
2. **Assignments → Users:** Include — `Directory roles`. Select: Global Administrator, Privileged Role Administrator, Security Administrator. **Exclude your break-glass accounts.**
3. **Cloud apps:** All cloud apps.
4. **Grant:** Require authentication strength → Phishing-resistant MFA.
5. Set to **Report-only**. Save.
6. Repeat the process for the admin portals MFA policy, the compliant device policy, and the block unknown platforms policy using the settings in [Section 4](#4-tier-2--admin-policies-privileged-roles).
7. Test by signing in as an account with a privileged role. Confirm sign-in logs show the admin policies evaluating on top of the baseline policies.
> **Screenshot:** CA policies list — showing all 8 policies with status (Report-only or On)

<img width="1218" height="363" alt="image" src="https://github.com/user-attachments/assets/beb3af40-1e1b-4b04-b316-c864ee3f39db" />

---

## 6. Design Decisions & Rationale
 
| Decision | Security Benefit | Trade-off Accepted |
|---|---|---|
| Deploy in report-only first | Zero risk of accidental lockout; validates policy logic against real sign-in patterns | Policies are not enforced during the monitoring window — risk is accepted for a limited period |
| MFA fallback on device compliance (Tier 1) | Allows users on personal devices to access work resources; avoids productivity disruption during MDM rollout | Unmanaged device risk is not fully mitigated until the fallback is removed in Phase 6 |
| No MFA fallback for admins (Tier 2) | Admin actions must come from a known, managed endpoint — no workaround possible | Admins without enrolled devices cannot work; requires hardware provisioning before rollout |
| Phishing-resistant MFA for admins only | FIDO2 keys cost $30–80 each; full-tenant rollout is expensive and logistically complex | Standard MFA risk remains for non-admin users; AiTM attacks targeting standard users are still possible |
| Exclude service accounts from user CA | Prevents breaking automation workflows that use interactive auth *(should migrate to managed identities)* | Service principals outside CA for workload identities have no access controls from this strategy |
| Block unknown device platforms | Prevents compliance bypass on unmanageable OS types | Legitimate Linux users cannot access resources as admins; must use a managed Windows/macOS machine |
| Sign-in frequency not set explicitly | Entra ID defaults provide reasonable session lengths; avoids re-auth friction for normal users | Long-lived sessions increase risk if tokens are stolen; consider 1-hour frequency for high-risk apps in future |
 
---
 
## 7. Monitoring & Ongoing Maintenance
 
Conditional Access policies are not set-and-forget. The following monitoring practices keep the strategy effective as the environment evolves.
 
### 7.1 Key Metrics to Monitor
 
| Metric | Where to Find It / What to Look For |
|---|---|
| **Policy interruptions** | Entra ID → Sign-in logs → filter `Conditional Access = Failure`. Investigate unexpected blocks. |
| **Legacy auth sign-ins** | Workbooks → Authentication methods → filter by legacy protocols. Should trend to zero after policy enablement. |
| **MFA registration rate** | Entra ID → Security → Authentication methods → Registration and reset events. Target: 100% of licensed users. |
| **Device compliance rate** | Microsoft Intune → Devices → Monitor → Device compliance. Track % compliant over time. |
| **Break-glass account usage** | Alert rule: any sign-in from break-glass UPN. Should be zero in normal operations. |
| **Admin sign-ins from non-compliant devices** | Sign-in logs filtered by directory role + `device compliant = false`. Should drop to zero after Tier 2 rollout. |
 
### 7.2 Review Cadence
 
- **Weekly** — Review sign-in log failures for new false positives or blocked legitimate users.
- **Monthly** — Review named locations list; add or remove IP ranges as offices change.
- **Quarterly** — Review the list of excluded accounts. Ensure service account exclusions are still necessary.
- **Annually** — Review all policies against the latest Microsoft Secure Foundation recommendations for new templates.
---
 
*Conditional Access Strategy | Microsoft Entra ID IAM Lab — Day 2 Portfolio Artifact*










