# 📓 Engineering Journal: Multi-Tiered Conditional Access Strategy & Zero-Trust Perimeter Build
 
**Author:** Identity & Access Management (IAM) Engineer  
**Environment:** Microsoft Entra ID Hybrid-Cloud Sandbox (P2 Tier)  
**Security Framework:** Zero-Trust Architecture (NIST 800-207 Aligned)
 
---
 
## Table of Contents
 
1. [Entry 1 — Project Scope & Foundation Engineering](#-entry-1-project-scope--foundation-engineering)
   - [Step 1 — Break-Glass Accounts](#️-step-1-mitigating-the-blast-radius-break-glass-accounts)
   - [Step 2 — Named Locations](#-step-2-mapping-the-corporate-perimeter-named-locations)
   - [Step 3 — Service Account Isolation](#️-step-3-protecting-automation-service-account-isolation)
2. [Entry 2 — Tier 1: Baseline Policies (All Users)](#-entry-2-engineering-tier-1--baseline-policies-all-users)
   - [Policy 1 — Block Legacy Authentication](#-policy-1-block-legacy-authentication)
   - [Policy 2 — Require MFA for All Users](#-policy-2-require-mfa-for-all-users)
   - [Policy 3 — Secure Security Info Registration](#️-policy-3-secure-security-info-registration)
   - [Policy 4 — Device-Aware Access Control](#-policy-4-device-aware-access-control-compliance-or-mfa)
   - [Troubleshooting — Policy 4 UI Quirks](#-troubleshooting-intermission-policy-4-ui-quirks)
3. [Entry 3 — Tier 2: Administrator Policies (Privileged Roles)](#-entry-3-engineering-tier-2--administrator-policies-privileged-roles)
   - [Policy 5 — Phishing-Resistant MFA for Admins](#️-policy-5-phishing-resistant-mfa-for-admins)
   - [Policy 6 — Require MFA for Microsoft Admin Portals](#️-policy-6-require-mfa-for-microsoft-admin-portals)
   - [Policy 7 — Strict Administrative Hardware Isolation](#-policy-7-strict-administrative-hardware-isolation-no-fallback)
   - [Policy 8 — Platform Whitelisting for Privileged Roles](#-policy-8-platform-whitelisting-for-privileged-roles)
4. [Entry 4 — Validation, Telemetry Ingestion & Log Diagnosis](#-entry-4-validation-telemetry-ingestion--log-diagnosis)
   - [Scenario 1 — "What If" Simulation Engine](#-scenario-1-the-what-if-simulation-engine)
   - [Scenario 2 — Live Traffic Telemetry Audit](#-scenario-2-live-traffic-telemetry-audit--diagnosis)
5. [Project Summary & Architectural Sign-Off](#-project-summary--architectural-sign-off)
---
 
## 📅 Entry 1: Project Scope & Foundation Engineering
 
### 🎯 Objective
 
Architect a robust perimeter defense using Microsoft Entra Conditional Access. The architecture implements a **multi-tiered security posture**:
 
- **Tier 1** — Baseline rules for all identities
- **Tier 2** — Strict hardware isolation for privileged administrators
---
 <img width="600" height="600" alt="image" src="https://github.com/user-attachments/assets/b7f2b6a8-a4f3-4651-99e3-ad098bb6d6f4" />

### 🛠️ Step 1: Mitigating the Blast Radius (Break-Glass Accounts)
 
Before writing a single restrictive policy, I engineered an **emergency recovery strategy** to prevent permanent tenant lockout in the event of an accidental global rule misconfiguration.
 
**Objects Created:**
 
| Account | UPN |
|---|---|
| `EMERGENCY-BreakGlass-01` | `breakglass01@<tenant>.onmicrosoft.com` |
| `EMERGENCY-BreakGlass-02` | `breakglass02@<tenant>.onmicrosoft.com` |
 
**Configuration rules:**
- **Role:** Elevated to Permanent, Active Global Administrators
- **Sync:** Pure cloud anchors — no on-premises synchronization dependencies
- **Policy exclusion:** Strictly excluded from **every** Conditional Access policy built in this project
---
 
### 🌐 Step 2: Mapping the Corporate Perimeter (Named Locations)
 
To enable location-aware security logic, I defined the company's safe network boundary.
 
1. Navigate to **Protection → Conditional Access → Named locations**.
2. Create object: `Corporate-Trusted-IPs`
3. Enter public IP ranges in **CIDR notation**.
4. Mark as **Trusted Location**.
---
 
### ⚙️ Step 3: Protecting Automation (Service Account Isolation)
 
Non-human service accounts run automated background tasks and **cannot respond to interactive MFA prompts**.
 
- **Action:** Created an assigned security group named `SG-ServiceAccounts-ExcludeFromCA`.
- **Purpose:** Acts as an administrative safety wrapper to prevent business-critical script failures when CA policies enforce interactive authentication.
---
 
## 📅 Entry 2: Engineering Tier 1 — Baseline Policies (All Users)
 
With the foundation secure, I built the foundational security tier. To guarantee business continuity, **all policies were initialized in Report-only mode** to collect background telemetry safely before enforcement.
 
---
 
### 🔒 Policy 1: Block Legacy Authentication
 
**Technical Goal:** Block obsolete protocols (`IMAP`, `POP3`, `SMTP Auth`, older ActiveSync) that cannot process interactive MFA prompts — and are actively exploited to bypass MFA entirely.
 
| Setting | Value |
|---|---|
| **Name** | `CA-Baseline-Block-Legacy-Auth` |
| **Users** | All users — exclude Break-Glass and Service Accounts |
| **Cloud apps** | All cloud apps |
| **Conditions — Client apps** | Exchange ActiveSync clients + Other clients (legacy protocols) |
| **Grant** | Block access |
| **State** | Report-only |
 
---
 
### 🔑 Policy 2: Require MFA for All Users
 
**Technical Goal:** Establish a baseline security posture across the directory to neutralize credential theft.
 
| Setting | Value |
|---|---|
| **Name** | `CA-Baseline-Require-MFA-AllUsers` |
| **Users** | All users — exclude Break-Glass and Service Accounts |
| **Cloud apps** | All cloud apps |
| **Grant** | Grant access → Require multifactor authentication |
| **State** | Report-only |
 
---
 
### 🛡️ Policy 3: Secure Security Info Registration
 
**Technical Goal:** Close the **"MFA Hijack" loophole** — where an attacker steals a password and registers their own MFA method before the legitimate user onboards.
 
| Setting | Value |
|---|---|
| **Name** | `CA-Baseline-Secure-MFA-Registration` |
| **Users** | All users — exclude Break-Glass |
| **Target resources** | User Actions → Register security information |
| **Conditions — Locations** | Any location, **EXCEPT** `Corporate-Trusted-IPs` |
| **Grant** | Grant access → Require multifactor authentication |
| **State** | Report-only |
 
---
 
### 💻 Policy 4: Device-Aware Access Control (Compliance or MFA)
 
**Technical Goal:** Reward corporate-managed devices while enforcing data boundaries on unmanaged personal machines.
 
| Setting | Value |
|---|---|
| **Name** | `CA-Baseline-CompliantDevice-or-MFA` |
| **Users** | All users — exclude Break-Glass and Service Accounts |
| **Target resources** | Office 365 application bundle |
| **Grant — operator** | Require **one of** (OR): compliant device, hybrid-joined device, MFA |
| **Session control** | Use app-enforced restrictions → read-only browser state in cloud documents |
| **State** | Report-only |
 
---
 
### 🔧 Troubleshooting Intermission: Policy 4 UI Quirks
 
Two issues surfaced during Policy 4 configuration — documented here for reproducibility.
 
**Issue 1 — App-Enforced Session Control Grayed Out**
 
- **Symptom:** The session restrictions checkbox was unclickable.
- **Root cause:** The policy was scoped to `All Cloud Apps`, but Microsoft's read-only browser wrapper only operates on Office 365, Exchange, and SharePoint.
- **Resolution:** Scoped Target Resources down to the **Office 365 application bundle**, which immediately unlocked the session control.
**Issue 2 — Device Platform Warning**
 
- **Symptom:** The Entra engine threw a high-friction warning flagging that Apple/Android devices would receive disruptive digital certificate prompts in Report-only mode.
- **Resolution:** Modified **Conditions → Device Platforms** to explicitly exclude `macOS`, `iOS`, `Android`, and `Linux` — isolating enforcement cleanly to **Windows devices**.
---
 
## 📅 Entry 3: Engineering Tier 2 — Administrator Policies (Privileged Roles)
 
To protect high-value directories, I engineered a highly restrictive admin perimeter layered **directly on top of Tier 1**. These policies target the following directory roles:
 
`Global Administrator` · `Security Administrator` · `Privileged Role Administrator` · `Exchange Administrator` · `SharePoint Administrator` · `Helpdesk Administrator`
 
---
 
### 🛡️ Policy 5: Phishing-Resistant MFA for Admins
 
**Technical Goal:** Mandate cryptographic hardware-backed tokens (**FIDO2 security keys**, **Windows Hello for Business**) for high-privilege roles — stopping MFA fatigue attacks and session-hijack proxy (AiTM) attacks.
 
| Setting | Value |
|---|---|
| **Name** | `CA-Admin-PhishingResistant-MFA` |
| **Users** | Targeted directory roles (see list above) — exclude Break-Glass |
| **Cloud apps** | All cloud apps |
| **Grant** | Require authentication strength → **Phishing-resistant MFA** |
| **State** | Report-only |
 
---
 
### 🏛️ Policy 6: Require MFA for Microsoft Admin Portals
 
**Technical Goal:** Secure the management plane by forcing an immediate MFA prompt the moment anyone touches a cloud administration URL.
 
| Setting | Value |
|---|---|
| **Name** | `CA-Admin-MFA-AdminPortals` |
| **Users** | All users — exclude Break-Glass |
| **Target resources** | Microsoft Admin Portals application container |
| **Grant** | Grant access → Require multifactor authentication |
| **State** | Report-only |
 
---
 
### 💻 Policy 7: Strict Administrative Hardware Isolation (No Fallback)
 
**Technical Goal:** Ban admins from accessing the environment via personal, unmanaged home devices. **There is no MFA fallback.**
 
| Setting | Value |
|---|---|
| **Name** | `CA-Admin-CompliantDevice-NoFallback` |
| **Users** | Targeted directory roles — exclude Break-Glass |
| **Cloud apps** | All cloud apps |
| **Grant — operator** | Require **ALL** (AND): compliant device + hybrid-joined device *(MFA left unchecked)* |
| **State** | Report-only |
 
> **Note:** Switching the logic engine to **AND operator** (rather than OR) is what eliminates the MFA fallback. Both device conditions must be satisfied simultaneously.
 
---
 
### 🚫 Policy 8: Platform Whitelisting for Privileged Roles
 
**Technical Goal:** Prevent compliance bypass by blocking admins from connecting via unmanageable or unknown OS platforms (e.g., custom Linux builds, spoofed user-agent strings).
 
| Setting | Value |
|---|---|
| **Name** | `CA-Admin-Block-UnknownPlatforms` |
| **Users** | Targeted directory roles — exclude Break-Glass |
| **Conditions — Device platforms** | Any device — **EXCLUDE** Windows, macOS, iOS |
| **Grant** | Block access |
| **State** | Report-only |
 
---
 
## 📅 Entry 4: Validation, Telemetry Ingestion & Log Diagnosis
 
### 🧪 Scenario 1: The "What If" Simulation Engine
 
Before live user testing, I used the **Conditional Access diagnostic simulator** to verify logical architecture.
 
**Test Case A — Regular User (`Casey Newhire`)**
 
| Variable | Value |
|---|---|
| Target app | Office 365 |
| Platform | Windows |
| Client app | Browser |
| **Result** | ✅ Policies 1–4 evaluated as active; Policies 5–8 remained completely silent |
 
**Test Case B — Global Administrator**
 
| Variable | Value |
|---|---|
| Platform | Windows |
| Client app | Browser |
| **Result** | ✅ All 8 policies registered under "Policies that will apply" — confirming role-based scoping was tight |
 
---
 
### 🔍 Scenario 2: Live Traffic Telemetry Audit & Diagnosis
 
I spun up real-world authenticated browser sessions via isolated **InPrivate windows** to audit the Entra ID Sign-in logs.
 
#### 🚨 Log Anomaly Diagnosis
 
**Symptom:** Casey Newhire's initial authentication event returned a status of `Not Applicable` with the message: *"No authentication events were triggered."*
 
**Root cause analysis:**
 
Casey was a freshly created user, so her initial sign-in was intercepted by Microsoft's default **first-time password reset flow**. Because her session had not yet reached a downstream target resource (Office 365), the CA engine correctly marked the transaction as `Not Applicable` for that temporary state.
 
**Resolution:**
 
1. Completed the password remediation workflow to establish a healthy post-onboarding user state.
2. Launched a secondary InPrivate session and authenticated directly to `https://office.com`.
3. Refreshed the **Interactive Sign-in Logs** console.
**Result:** Casey's authentication ledger updated to `Report-only: Success` across all Tier 1 baseline policies — confirming the perimeter engine was functioning correctly.
 
---
 
## 🏁 Project Summary & Architectural Sign-Off
 
By completing this journal, I successfully **designed, built, and audited** a comprehensive two-tiered Conditional Access architecture.
 
| Milestone | Status |
|---|---|
| Break-glass infrastructure deployed | ✅ Complete |
| Corporate named location defined | ✅ Complete |
| Service account exclusion group created | ✅ Complete |
| Tier 1 baseline policies (Policies 1–4) deployed in Report-only | ✅ Complete |
| Tier 2 admin policies (Policies 5–8) deployed in Report-only | ✅ Complete |
| What-If simulation validated for standard and admin users | ✅ Complete |
| Live telemetry audit passed with log anomaly resolved | ✅ Complete |
 
Through defensive engineering (break-glass infrastructure and report-only deployment phases) and advanced logging diagnostics, the tenant perimeter has transitioned into a robust, high-governance **Zero-Trust environment**.
 
---
 
*Engineering Journal — Microsoft Entra ID IAM Lab | NIST 800-207 Zero-Trust Aligned*
