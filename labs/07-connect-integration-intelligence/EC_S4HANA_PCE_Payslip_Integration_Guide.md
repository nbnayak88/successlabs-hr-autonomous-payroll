# SAP SuccessFactors EC ↔ S/4HANA Private Cloud Edition (RISE)  
## Payslip Integration Configuration Guide — No ECP Scenario

> **SuccessLabs Academy | Advisory Series**  
> Document Code: `INT-PAYSLIP-001`  
> Version: `v1.0` | Published: October 2026  
> Authors: SuccessLabs Academy Advisory Board  
> Applies to: SAP SuccessFactors Employee Central + SAP S/4HANA Private Cloud Edition (RISE/PCE) — **No Employee Central Payroll (ECP)**

---

## Table of Contents

1. [Overview and Scope](#1-overview-and-scope)
2. [Architecture Explained](#2-architecture-explained)
3. [Prerequisites](#3-prerequisites)
4. [Add-On Selection: ECS4HCM vs PA_SE_IN](#4-add-on-selection-ecs4hcm-vs-pa_se_in)
5. [Step 1 — SAML2 Configuration on S/4HANA (KBA 2253359)](#5-step-1--saml2-configuration-on-s4hana-kba-2253359)
6. [Step 2 — SICF Service Activation on S/4HANA](#6-step-2--sicf-service-activation-on-s4hana)
7. [Step 3 — Network Exposure via Web Dispatcher / Cloud Connector (RISE-Specific)](#7-step-3--network-exposure-via-web-dispatcher--cloud-connector-rise-specific)
8. [Step 4 — Assertion Consumer Service (ACS) in SF Provisioning](#8-step-4--assertion-consumer-service-acs-in-sf-provisioning)
9. [Step 5 — Payroll System Configuration Object in EC Admin Center](#9-step-5--payroll-system-configuration-object-in-ec-admin-center)
10. [Step 6 — Role-Based Permissions in EC](#10-step-6--role-based-permissions-in-ec)
11. [Step 7 — People Profile and Homepage Configuration](#11-step-7--people-profile-and-homepage-configuration)
12. [Testing and Validation](#12-testing-and-validation)
13. [Common Errors and Resolutions](#13-common-errors-and-resolutions)
14. [Security Checklist](#14-security-checklist)
15. [Key SAP References](#15-key-sap-references)
16. [Glossary](#16-glossary)

---

## 1. Overview and Scope

### 1.1 What This Guide Covers

This guide describes how to surface **payslips from SAP S/4HANA Private Cloud Edition (RISE/PCE) directly inside SAP SuccessFactors Employee Central (EC)** — without using Employee Central Payroll (ECP) as an intermediary.

This is the **Core Hybrid** deployment model:

| Component | Deployment | Role |
|---|---|---|
| SAP SuccessFactors Employee Central | Public Cloud (SaaS) | Master HR data — source of truth for all employee records |
| SAP S/4HANA PCE | Private Cloud (RISE — managed by SAP) | Payroll engine — runs all pay calculations, stores payroll results (RT cluster) |
| Employee Central Payroll (ECP) | ❌ **Not in scope** | Not deployed in this scenario |

### 1.2 What This Guide Does NOT Cover

- EC + ECP payslip configuration (see SAP Help: *Implementing Employee Central Payroll based on SAP HCM for S/4HANA*)
- Third-party payroll (ADP, Ceridian, etc.) payslip embedding into EC
- SAP S/4HANA Public Cloud payslip
- Custom payslip development (HRFORMS, SmartForms)

### 1.3 Core Principle

> **The payslip is NOT replicated to EC as data.** It is rendered live from S/4HANA into EC via a **SAML2 SSO mashup**. When an employee clicks "View Pay Statement" in EC, EC acts as the Identity Provider (IdP) and authenticates the employee silently against S/4HANA, which then renders the payslip document inside the EC iFrame.

---

## 2. Architecture Explained

```
┌─────────────────────────────────────────────────────────────────────┐
│                   SF Employee Central (Public Cloud)                │
│                                                                     │
│  ┌──────────────┐   ┌────────────────────────────────────────────┐  │
│  │  EC People   │   │  Payroll Information Portlet               │  │
│  │  Profile     │──▶│  └─ "View Pay Statement" link              │  │
│  └──────────────┘   └────────────────┬───────────────────────────┘  │
│                                      │  SAML2 Assertion (IdP = EC)  │
│  ┌────────────────────────────────┐  │                              │
│  │  EC acts as SAML2 IdP          │──┘                              │
│  │  (issues signed SAML token)    │                                 │
│  └────────────────────────────────┘                                 │
└────────────────────────────────────┬────────────────────────────────┘
                                     │ HTTPS POST (SAML Assertion)
                                     ▼
                    ┌────────────────────────────┐
                    │   SAP Web Dispatcher        │
                    │   (Managed by SAP RISE ops) │
                    │   Exposes ACS URL publicly  │
                    └──────────────┬─────────────┘
                                   │ Routes to backend
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│              S/4HANA PCE — Private Cloud (RISE)                      │
│                                                                      │
│  ┌───────────────────┐    ┌─────────────────────────────────────┐   │
│  │  SAML2 Service    │    │  SICF Services (Active)             │   │
│  │  Provider Config  │    │  ├─ HRSFEC_PAY_OVERVIEW_SRV         │   │
│  │  (T-code: SAML2)  │    │  ├─ /sap/bc/webdynpro/.../hcmq_    │   │
│  │                   │    │  │   paylip                         │   │
│  │  Validates SAML   │    │  └─ /sap/public/bc/sec/saml2        │   │
│  │  assertion from EC│    └─────────────────────────────────────┘   │
│  └───────────────────┘                                              │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  HCM Payroll Engine (SAP HCM for S/4HANA — H4S4)             │  │
│  │  Payroll Results (RT Cluster) → rendered as payslip document  │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘

Master Data Flow (separate from payslip):
  EC → [ECS4HCM iFlows via SAP Integration Suite] → S/4HANA PCE
  (HR infotypes: IT0001, IT0002, IT0006, IT0007, IT0008, IT0009, etc.)
```

### 2.1 Data Flow Summary

| Flow | Direction | Mechanism |
|---|---|---|
| Employee master data replication | EC → S/4HANA PCE | ECS4HCM add-on iFlows (SAP Integration Suite) |
| Payslip display | S/4HANA PCE → EC (rendered live) | SAML2 SSO mashup via Web Dispatcher |
| Payroll confirmation | S/4HANA PCE → EC | ECS4HCM confirmation iFlow |
| Payroll results storage | Stays in S/4HANA PCE | NOT replicated back to EC |

---

## 3. Prerequisites

Before beginning configuration, verify the following are in place:

### 3.1 Technical Prerequisites

| # | Prerequisite | Check |
|---|---|---|
| 1 | S/4HANA PCE release **2022 or higher** | Mandatory for ECS4HCM add-on |
| 2 | **SAP HCM for S/4HANA** solution installed on PCE | Required — payroll must be H4S4, not legacy ECC HCM |
| 3 | **ECS4HCM add-on** (`SFSF EC S4 HCM INTEGRATION`) installed on S/4HANA PCE | Replaces old PA_SE_IN add-on |
| 4 | SAP Integration Suite (Cloud Integration) tenant provisioned | Required for master data replication iFlows |
| 5 | SF EC instance active (Dev / Staging / Prod) | Your IdP endpoint |
| 6 | SAP Web Dispatcher accessible (or Cloud Connector configured) | Required to expose S/4HANA SICF URLs to EC |
| 7 | RISE TSM / CDM contact identified | Needed to request Web Dispatcher configuration |
| 8 | Basis team access to S/4HANA PCE | For SAML2, SICF, and SICF transactions |
| 9 | SF EC Provisioning Admin access | For ACS URL maintenance |
| 10 | SF EC Admin Center access | For Payroll System Configuration |

### 3.2 Pay Statement Prerequisite (S/4HANA Side)

> **Important:** The payslip must already be configured inside S/4HANA before EC integration can display it. This means:
> - Payroll schema configured and validated
> - HRFORMS / SmartForms payslip form activated for the relevant country grouping (Molga)
> - At least one successful payroll run completed (data exists in RT cluster)
> - T-code `PC_PAYRESULT` verifiable for a test employee

---

## 4. Add-On Selection: ECS4HCM vs PA_SE_IN

If you are planning a fresh implementation or migrating, understand the two add-ons:

| Feature | PA_SE_IN (Old) | ECS4HCM (New — Use This) |
|---|---|---|
| Full product name | SFSF EC INTEGRATION | SFSF EC S4 HCM INTEGRATION |
| Middleware supported | Boomi (Dell) OR SAP HCI | **SAP Cloud Integration ONLY** |
| Min. SAP version | ECC 6.0 / ERP HCM | **S/4HANA 2022+** |
| Maintenance end | 2027 (SAP Business Suite mainstream) | **2040** (aligned with H4S4) |
| Implementation effort | Higher | **Lower — simpler to implement** |
| Payslip integration method | SAP Note 2253359 (SAML) | Same SAP Note 2253359 (SAML) |
| Recommended for RISE PCE | ❌ No | ✅ **Yes — mandatory** |

> **Migration note:** Customers currently using PA_SE_IN who want to move to ECS4HCM must **upgrade** — not install alongside. Engage SAP or your system integrator for the migration path.

---

## 5. Step 1 — SAML2 Configuration on S/4HANA (KBA 2253359)

This step configures S/4HANA PCE as a **SAML2 Service Provider (SP)** that trusts SF EC as the **Identity Provider (IdP)**.

### 5.1 Download EC IdP Metadata

In SF EC Provisioning:

1. Navigate to **Company Settings**
2. Locate **SAML Single Sign-On (SSO) Settings**
3. Download the **IdP Metadata XML** file for your EC instance
   - File typically named: `idp_metadata_{companyID}.xml`
   - Contains: EntityID, SSO endpoint URLs, signing certificate

> 💡 You need separate metadata files for each EC instance (Dev, Staging, Production). Never use Prod metadata in Dev/QA.

### 5.2 Run SAML2 Automated Configuration Report (Recommended)

On S/4HANA PCE, run report **`RP_HRSFEC_SAML_CONFIG`**:

```
T-code: SA38
Program: RP_HRSFEC_SAML_CONFIG
```

This report automates the SAML2 configuration for payslip. Provide:
- Your EC Company ID
- EC Data Center (e.g., `api4.successfactors.com`)
- Client number of the S/4HANA PCE system

### 5.3 Manual SAML2 Configuration (via T-code SAML2)

If running the report is not possible, configure manually:

**Step 5.3.1 — Configure Local Provider (S/4HANA as SP)**

```
T-code: SAML2
→ Tab: "Local Provider"
→ Provider Name: e.g., S4PCE_PROD_SP
→ Description: SuccessFactors Payslip SP - Production
```

**Step 5.3.2 — Import EC as Trusted IdP**

```
T-code: SAML2
→ Tab: "Trusted Providers"
→ Import: Upload the EC IdP metadata XML downloaded in Step 5.1
→ Assign: Default Name ID format → Email (or NameID as configured in EC)
```

**Step 5.3.3 — Configure Assertion Consumer Service (ACS)**

The ACS URL is the endpoint where EC sends the SAML assertion. After configuring the SP, export the **S/4HANA SP Metadata** — it contains:

```xml
<!-- Example ACS URL pattern in S/4HANA metadata -->
<AssertionConsumerService 
  Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
  Location="https://{web-dispatcher-host}/sap/saml2/sp/acs/{client}"
  index="1"/>

<!-- Example SLO URL -->
<SingleLogoutService 
  Binding="urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST"
  Location="https://{web-dispatcher-host}/sap/saml2/sp/slo/{client}"/>
```

Save this metadata XML — you need the **ACS URL** and **SLO URL** for Step 4 (SF Provisioning).

**Step 5.3.4 — Configure NameID Mapping**

Ensure the NameID sent by EC matches the user in S/4HANA:

```
T-code: SAML2
→ Identity Provider: [Your EC IdP entry]
→ NameID Format: Email Address (recommended)
→ User Mapping: Email → S/4HANA User (SY-UNAME)
```

> ⚠️ The email address in EC must exactly match the email-based user in S/4HANA PCE. Mismatches are the #1 cause of "user not found" errors during payslip access.

---

## 6. Step 2 — SICF Service Activation on S/4HANA

The payslip WebDynpro application is exposed via **SICF (Internet Communication Framework)** services. These must be activated before the payslip URL can be accessed.

### 6.1 Activate Pay Statement Services

```
T-code: SICF
```

Activate the following services (right-click → **Activate Service**):

| SICF Path | Service Name | Purpose |
|---|---|---|
| `/sap/bc/webdynpro/sap/hcmq_paylip` | HCMQ_PAYLIP | Standard payslip (arrow-based UI) |
| `/sap/bc/webdynpro/sap/hrsfec_pay_overview_srv` | HRSFEC_PAY_OVERVIEW_SRV | **Direct Pay Statement** (modern UI — recommended) |
| `/sap/public/bc/sec/saml2` | SAP SAML2 | SAML2 metadata endpoint |
| `/sap/bc/sec/saml2` | SAML2 SP ACS | SAML assertion consumer endpoint |

### 6.2 Configure SICF Logon Procedure

For each payslip service node:

```
T-code: SICF
→ Navigate to the payslip service node
→ Double-click → Logon Data tab
→ Logon Procedure: SAML2 (or "Anonymous with SAML2 Header")
```

> 🚨 **CRITICAL SECURITY RULE (KBA 2838571):**  
> **NEVER** enter a fixed username and password in the SICF Logon Data for the payslip service.  
> A fixed credential in SICF means any user who knows the URL can authenticate as that user — a serious security vulnerability.  
> **Use SAML2 authentication only.**

### 6.3 Verify Service Activation

Test that the SICF endpoints respond:

```bash
# From within the SAP network (before external exposure)
# Replace {host}, {port}, {client} with your values
curl -I "http://{s4-internal-host}:{port}/sap/bc/webdynpro/sap/hrsfec_pay_overview_srv?sap-client={client}"
# Expected: HTTP 302 redirect to SAML login (not 404 or 503)
```

---

## 7. Step 3 — Network Exposure via Web Dispatcher / Cloud Connector (RISE-Specific)

> **This step is unique to RISE Private Cloud and is the most operationally complex part.**  
> S/4HANA PCE sits inside SAP's managed private cloud — its services are NOT internet-accessible by default.  
> EC (running in SAP's public cloud) must be able to reach S/4HANA's SICF payslip URL over HTTPS.

### 7.1 Option A — SAP-Managed Web Dispatcher (Standard RISE Path)

SAP manages a Web Dispatcher for each RISE customer as part of the service. This is the **recommended and most straightforward path**.

**7.1.1 Raise a Service Request (SR)**

Contact your SAP RISE operations team:
- **RISE Technical Service Manager (TSM)** — primary contact
- **Cloud Delivery Manager (CDM)** — escalation contact
- Channel: SAP for Me → Service Request (component: `BC-OP-LNX` or RISE-specific component)

Request the following in the SR:

```
Subject: Expose SuccessFactors Payslip SICF URL via Web Dispatcher — [System ID]

We need the following S/4HANA PCE services exposed via the 
SAP-managed Web Dispatcher for SuccessFactors EC payslip integration:

System: [SID], Client: [100/200/etc.]
Environment: [Production / QA / Dev]

Services to expose:
1. /sap/bc/webdynpro/sap/hrsfec_pay_overview_srv
2. /sap/bc/webdynpro/sap/hcmq_paylip  
3. /sap/public/bc/sec/saml2
4. /sap/bc/sec/saml2

Protocol: HTTPS (TLS 1.2 or higher)
Expected caller: SAP SuccessFactors EC (SAP-managed public cloud IPs)

Reference: SAP KBA 2253359 (EC payslip integration for S/4HANA customers)
```

**7.1.2 Resulting Public URL Pattern**

After the SR is actioned, your Web Dispatcher URL will follow this pattern:

```
https://{your-s4-webdispatcher-hostname}.hanacloudservices.cloud.sap/sap/bc/webdynpro/sap/hrsfec_pay_overview_srv?sap-client={client}
```

Record this URL — it is used in:
- Step 4 (ACS URL in SF Provisioning)
- Step 5 (Payroll System Configuration in EC)

### 7.2 Option B — SAP Cloud Connector (Alternative)

For customers who prefer or already use SAP BTP for integration:

**7.2.1 Install SAP Cloud Connector**

```
Download: tools.hana.ondemand.com > Cloud Connector
Installation: On a host within the RISE-managed network (or on-premise near it)
```

**7.2.2 Connect to BTP Subaccount**

```
Cloud Connector Admin UI: https://localhost:8443
→ Add Subaccount
  Region: [Your BTP region]
  Subaccount ID: [BTP Subaccount ID]
  User / Password: [BTP S-User credentials]
```

**7.2.3 Map S/4HANA Backend Resource**

```
Cloud Connector Admin UI
→ Cloud to On-Premise
→ Add Backend System:
  Backend Type: SAP HANA System (or ABAP)
  Protocol: HTTPS
  Internal Host: {s4-internal-hostname}
  Internal Port: {443 or HTTPS port}
  Virtual Host: s4pce-payslip
  Virtual Port: 443

→ Resources Accessible:
  URL Path: /sap/bc/webdynpro/sap/hrsfec_pay_overview_srv
  URL Path: /sap/public/bc/sec/saml2
  URL Path: /sap/bc/sec/saml2
```

> 💡 SAP Router is offered as a managed service for S/4HANA systems in RISE. Contact your RISE TSM to set up SAP Router-based connectivity from ECP or other cloud services to S/4HANA RISE — both are ABAP systems technically and this connectivity pattern is supported.

### 7.3 TLS / Certificate Considerations

| Requirement | Detail |
|---|---|
| Protocol | HTTPS (TLS 1.2+) — never HTTP |
| S/4HANA server certificate | Must be from a trusted CA (SAP-issued or customer-provided) |
| Certificate validation by EC | EC validates the server certificate; self-signed certificates will be rejected |
| Client certificate (optional) | Not required for SAML2-based payslip; SAML assertion handles authentication |

---

## 8. Step 4 — Assertion Consumer Service (ACS) in SF Provisioning

Now that the Web Dispatcher URL is known, register S/4HANA as a **trusted Service Provider** in SF EC Provisioning.

### 8.1 Locate the S/4HANA SP Metadata

From Step 5.3.3, you exported the S/4HANA SP metadata XML. Extract the following URLs:

```
ACS URL:  https://{web-dispatcher-host}/sap/saml2/sp/acs/{client}
SLO URL:  https://{web-dispatcher-host}/sap/saml2/sp/slo/{client}
EntityID: https://{web-dispatcher-host}/sap/saml2/sp/metadata/{client}
```

Example for Production:
```
ACS URL:  https://s4pce-prod.hanacloudservices.cloud.sap/sap/saml2/sp/acs/100
SLO URL:  https://s4pce-prod.hanacloudservices.cloud.sap/sap/saml2/sp/slo/100
```

### 8.2 Configure in SF Provisioning

```
URL: https://{your-ec-instance}.successfactors.com/provisioning_login

→ Company Settings
→ SAML Single Sign-On Settings (or "Trusted Service Providers")
→ Click "Add another Service Provider ACS"

Add:
  Service Provider Name:   S4PCE_PROD  (or descriptive label)
  Assertion Consumer URL:  https://{web-dispatcher-host}/sap/saml2/sp/acs/{client}
  Logout URL:              https://{web-dispatcher-host}/sap/saml2/sp/slo/{client}
  Entity ID:               (from SP metadata)
  Enabled:                 ✅ Yes
```

> ⚠️ **Environment isolation (KBA 3061976):** Each S/4HANA system (Dev / QA / Prod) must have its **own ACS URL** registered in the corresponding EC instance. Mixing environments causes the "system redirects to wrong environment" error. Never register a QA S/4HANA ACS URL in the Production EC instance.

### 8.3 Verify the Configuration

After saving, download the updated **EC IdP metadata** and confirm the S/4HANA SP ACS URL appears in it.

---

## 9. Step 5 — Payroll System Configuration Object in EC Admin Center

This step configures **which payroll service EC presents** in the employee's Payroll Information portlet.

### 9.1 Import Payroll Configuration Picklists (if missing)

If the Pay Statement service does not appear in the portlet dropdown:

```
EC Admin Center
→ Manage Picklists  
→ Import from Success Store
→ Package: "Payroll Configuration Picklists"
→ Type: Payroll Configuration
→ Import
```

Verify: The picklist **`payroll_services`** exists with an entry for **Pay Statement** having external code = `1`.

### 9.2 Create / Update Payroll System Configuration

```
EC Admin Center
→ Manage Data
→ Create New → Payroll System Configuration
```

Fill in the following:

| Field | Value | Notes |
|---|---|---|
| **System Name** | `S4PCE_PROD` | Descriptive identifier |
| **Payroll Vendor** | SAP | |
| **System URL** | `https://{web-dispatcher-host}` | Base URL only — no path |
| **Payroll System URL** | `https://{web-dispatcher-host}` | Same as System URL |
| **Country** | Select applicable countries | Must match the Molga in S/4HANA |
| **Legal Entity** | Assign to relevant legal entities | |

### 9.3 Configure the Pay Statement Portlet

Under the **Portlets** section of the Payroll System Configuration:

```
→ Earnings and Deductions → Details
→ Service: "Pay Statement" (or "Direct Pay Statement" for modern UI)
→ External Code for Pay Statement: 1
→ Save
```

> 💡 **Pay Statement (Direct) vs Pay Statement (Arrow-based):**  
> Pay Statement Direct (`HRSFEC_PAY_OVERVIEW_SRV`) is the modern version — it displays pay date, gross, and net payout on the card view before the employee clicks in. It is more user-friendly and is the recommended choice for new implementations.

### 9.4 Configure Service URL

For Pay Statement Direct, additionally configure:

```
→ Service URL: https://{web-dispatcher-host}/sap/bc/webdynpro/sap/hrsfec_pay_overview_srv
→ Add System Alias: (if multiple S/4HANA clients)
```

---

## 10. Step 6 — Role-Based Permissions in EC

Grant employees and managers access to the payslip portlet.

### 10.1 Employee Self-Service Permission

```
EC Admin Center
→ Manage Permission Roles
→ Select Role: Employee Self-Service (or your equivalent ESS role)

→ Permissions tab → Employee Data
  └─ Payroll Information
      ├─ Pay Statement:              ✅ View
      └─ Payroll Information block:  ✅ View
```

### 10.2 Manager / HR View (Optional)

If managers should view direct report payslips (review client requirements carefully — many clients restrict this):

```
→ Select Role: Manager Self-Service
→ Permissions tab → User Data
  └─ Payroll Information
      └─ Pay Statement: ✅ View (if client requires)
```

> ⚠️ Most organizations grant **View Pay Statement** to employees only (self-service). Manager access to payslips is a sensitive HR policy decision — confirm with the client before enabling.

### 10.3 Save and Re-index Permissions

After saving permission changes:

```
EC Admin Center
→ Manage Permission Roles
→ "Refresh Permissions" or allow nightly permission re-indexing
```

---

## 11. Step 7 — People Profile and Homepage Configuration

Two surfaces in EC display the payslip link.

### 11.1 People Profile — Payroll Information Portlet

```
EC Admin Center
→ Configure People Profile
→ Add Block: Payroll Information
→ Position: In the employee's profile (typically under Employment Information or dedicated Payroll tab)
→ Portlet Name: "Pay Statement" (or "Payroll Information" — per client naming convention)
→ Save
```

### 11.2 Latest Homepage — Quick Action Card

The **"View Pay Statement"** quick action card on the EC Latest Homepage gives employees one-click access:

```
EC Admin Center
→ Configure Latest Home Page
→ Quick Actions
→ Enable: View Pay Statement
→ Save
```

> 💡 The quick action card shows the employee's **most recent payslip** with pay date, gross, and net payout. It is only available with **Pay Statement Direct** (`HRSFEC_PAY_OVERVIEW_SRV`) — not with the legacy arrow-based service.

---

## 12. Testing and Validation

### 12.1 Test Sequence

Perform tests in this order — Dev → Staging → Production:

| # | Test | Expected Result | T-code / Action |
|---|---|---|---|
| 1 | Verify payslip exists in S/4HANA for test employee | Pay result visible in RT cluster | `PC_PAYRESULT` |
| 2 | Confirm SICF services are active | HTTP 302 redirect on accessing URL | `SICF` |
| 3 | Confirm SAML2 SP configuration exists | SP entity shown | `SAML2` |
| 4 | Test SAML metadata endpoint is accessible externally | HTTP 200 with XML response | `GET {web-dispatcher-host}/sap/public/bc/sec/saml2/sp/metadata/{client}` |
| 5 | Open EC People Profile for test employee | Payroll Information portlet visible | EC Web UI |
| 6 | Click "View Pay Statement" | Payslip renders inside EC — no login prompt | EC People Profile |
| 7 | Navigate to previous payslip periods | Period navigation works | EC Payroll Information portlet |
| 8 | Test "Print" functionality | PDF rendered correctly | EC Payroll Information portlet |
| 9 | Test from employee self-service (non-admin) | Employee can only see own payslip | EC ESS |
| 10 | Test payslip quick action on Latest Homepage | Quick action card visible and functional | EC Latest Homepage |

### 12.2 "Proxy Now" Test in EC

EC provides a built-in test tool for payroll system connectivity:

```
EC Admin Center
→ Manage Data → Payroll System Configuration → [Your config]
→ Button: "Proxy Now"
```

This simulates the SAML2 redirect flow without requiring an actual employee click — useful for Basis teams to validate connectivity before user acceptance testing.

---

## 13. Common Errors and Resolutions

| Error Message | Cause | Resolution |
|---|---|---|
| Payslip URL not reachable (HTTP 404 / 502) | SICF service not exposed via Web Dispatcher | Raise SR with RISE TSM to expose the SICF service URL |
| "The requested operation is not available" | SAML IdP session / entity mismatch between environments | See KBA 3061976 — verify entity IDs match between EC instance and S/4HANA SP config |
| System redirects to wrong EC environment (e.g., test payslip opens prod EC) | Multiple EC instances sharing same SAML config | KBA 3061976 — each environment needs its own ACS URL in the corresponding EC Provisioning |
| "You cannot execute the Payslip Service for Security reasons. Please contact your admin." | Fixed username/password stored in SICF payslip node | KBA 2838571 — remove fixed credentials from SICF, configure SAML only |
| "User not found" or "No authorization" in S/4HANA | NameID mismatch — email in EC does not match S/4HANA user | Verify email-to-user mapping in SAML2 T-code; ensure users are provisioned in S/4HANA with matching email |
| Payslip link not visible in employee profile | Missing picklist value OR portlet not configured | Import "Payroll Configuration Picklists" from Success Store; verify Payroll System Configuration portlet is set up |
| ACS URL not accepted in SF Provisioning | URL formatted incorrectly (http vs https, wrong client number) | Use exactly the ACS URL from S/4HANA SAML2 SP metadata XML |
| SAML certificate error | S/4HANA signing certificate expired or not trusted by EC | Renew S/4HANA SAML2 certificate; re-import updated SP metadata in SF Provisioning |
| Payslip renders blank / no data | Pay results do not exist for this employee in S/4HANA | Run `PC_PAYRESULT` in S/4HANA to verify results exist for the employee and pay period |
| Mashup opens in a new browser window instead of EC iFrame | CSP/X-Frame-Options header set on S/4HANA | SAP Basis to check Web Dispatcher response headers — remove X-Frame-Options: DENY or SAMEORIGIN restriction for payslip path |

---

## 14. Security Checklist

Before going live, validate the following security controls:

- [ ] **No fixed users in SICF** — payslip SICF node uses SAML2 authentication only (KBA 2838571)
- [ ] **TLS 1.2+ enforced** — Web Dispatcher configured for TLS 1.2 minimum; TLS 1.0 and 1.1 disabled
- [ ] **S/4HANA SAML certificate validity** — SAML signing certificate validity checked; expiry calendar entry created
- [ ] **Environment isolation** — Dev/QA/Prod S/4HANA ACS URLs are in their respective EC instances only — never cross-registered
- [ ] **NameID format locked** — SAML NameID format (email) matches consistently across all S/4HANA systems
- [ ] **RBP scoped to self** — Employee permission for Pay Statement scoped to "self" only using the standard EC RBP self-service filter
- [ ] **No HTTP allowed** — Web Dispatcher URL for payslip responds to HTTP requests with a redirect to HTTPS (not serving over plain HTTP)
- [ ] **Payslip path is allowlisted** — Only payslip-related SICF paths are exposed via Web Dispatcher; no broader `/sap/` exposure
- [ ] **Audit log** — SAP Security Audit Log (T-code `SM19`) includes payslip SICF service access in audit scope
- [ ] **Network penetration** — Web Dispatcher payslip URL tested for unauthorized access (unauthenticated request should receive SAML redirect, not payslip data)

---

## 15. Key SAP References

### 15.1 SAP KBAs and Notes (SAP for Me Login Required)

| Reference | Title | Relevance |
|---|---|---|
| **KBA 2253359** | How to enable Pay Statement/Mashup services in Employee Central System for SAP S/4HANA and ERP customers | **Primary reference** — full SAML + SICF setup steps for non-ECP scenario |
| **KBA 3061976** | When opening Pay Statement/mashup in EC, system redirects to a different environment or error "The requested operation is not available" | Environment redirect error — ACS URL and entity ID fixes |
| **KBA 2838571** | Unable to access payslip from EC: "You cannot execute the Payslip Service for Security reasons" | SICF fixed-user security error — SAML-only configuration |
| **SAP Note 2253359** | SAML2 configuration for S/4HANA / ERP EC integration | Same as KBA 2253359 — use for manual SAML steps |

> Access all KBAs at: [SAP for Me — Knowledge Base](https://me.sap.com/knowledgebasearticles)

### 15.2 SAP Help Portal Documentation

| Document | URL |
|---|---|
| EC Integration with SAP HCM for S/4HANA (ECS4HCM) | [help.sap.com — SAP SuccessFactors Employee Central Integration with SAP HCM for S/4HANA](https://help.sap.com/docs/SAP_SUCCESSFACTORS_EMPLOYEE_CENTRAL/58e40e8fdde64ca5b44d92b65e9f2ade/7c60b3cc0f6b45a780c8c0db9a24fef3.html) |
| ECS4HCM Add-On Overview | [help.sap.com — SFSF EC S4 HCM INTEGRATION](https://help.sap.com/docs/SAP_S4HANA_ON-PREMISE/c0e06e8dd1c3469a8dcef2a5a93b6df0/7bb6ee8c05a14b0cad3c5ac21a35a0b9.html) |
| SAP Cloud Integration — EC iFlows | [help.sap.com — Integration Content for Employee Central](https://help.sap.com/docs/SAP_INTEGRATION_SUITE/368c481cd6954bdfa5d0435479fd4eaf/7c60b3cc0f6b45a780c8c0db9a24fef3.html) |
| RISE with SAP S/4HANA PCE Service Description | [SAP Cloud Service Specifications](https://www.sap.com/about/agreements/policies/cloud-service-specifications.html) |

### 15.3 SAP Community Blogs

| Title | URL | Authors |
|---|---|---|
| How to Enable Pay Statement Mashup from SAP Payroll to SuccessFactors | [SAP Community Blog](https://blogs.sap.com/t5/human-capital-management-blogs-by-members/how-to-enable-pay-statement-mashup-from-sap-payroll-to-successfactors/ba-p/13346933) | Gobinder Sandhu, Jai Korpal |
| Enabling Country-Specific Payroll Tax Form Integration in SuccessFactors | [SAP Community Blog](https://blogs.sap.com/t5/human-capital-management-blogs-by-members/enabling-the-country-specific-payroll-tax-form-integration-in/ba-p/13339638) | Gobinder Sandhu, Jai Korpal |
| New Add-On for SF EC Integration with SAP HCM for S/4HANA (ECS4HCM) | [Rizing Blog](https://rizing.com/human-capital-management/h2-2022-sap-successfactors-release-analysis-technical) | Rizing HCM Team |
| Setting Up Payroll Processing with SF EC Payroll with S/4HANA | [SAP Community Blog](https://answers.sap.com/t5/technology-blogs-by-members/setting-up-payroll-processing-with-sap-successfactors-employee-central/ba-p/13556072) | SAP Community |

### 15.4 Related SuccessLabs Academy Resources

| Resource | Series Code | Description |
|---|---|---|
| SAP ECP International Payroll Implementation Guide | `INT-GLOBAL-001` | 59-country KDS framework; Molga 99 design; EC→ECP→S/4HANA integration |
| KDS Workbooks — APAC Series | AP01–AP12 | Country-specific payroll design sheets for APAC |
| KDS Workbooks — EU Series | EU01–EU27 | Country-specific payroll design sheets for Europe |
| KDS Workbooks — ME Series | ME01–ME08 | Country-specific payroll design sheets for Middle East |
| KDS Workbooks — LATAM Series | LA01–LA06 | Country-specific payroll design sheets for Latin America |
| SAP Cloud ALM Tier Comparison Framework | `ALM-TIER-001` | 4-tier model for ALM in SuccessFactors implementations |

---

## 16. Glossary

| Term | Definition |
|---|---|
| **ACS** | Assertion Consumer Service — the HTTPS endpoint on S/4HANA that receives the SAML2 assertion from EC (IdP) |
| **Core Hybrid** | SAP's term for the deployment where EC is the HR master in the cloud and on-premise/PCE S/4HANA runs payroll |
| **ECP** | Employee Central Payroll — SAP's cloud-hosted payroll engine (NOT in scope for this guide) |
| **ECS4HCM** | SFSF EC S4 HCM INTEGRATION — the SAP add-on for replicating EC data to S/4HANA (successor to PA_SE_IN) |
| **H4S4** | SAP HCM for S/4HANA — the payroll component embedded in S/4HANA (replaces ECC HCM on S/4HANA) |
| **IdP** | Identity Provider — in this scenario, SF EC issues SAML assertions identifying the employee |
| **Mashup** | SAP/SF term for embedding a remote application (S/4HANA payslip) inside the SF UI via SAML SSO |
| **Molga** | Country grouping identifier in SAP payroll schema/PCR logic (e.g., Molga 10 = Germany, Molga 40 = India) |
| **PCE** | Private Cloud Edition — SAP's managed cloud offering for S/4HANA; underlying platform for RISE |
| **RISE with SAP** | SAP's bundled offering for migrating to S/4HANA private cloud, including managed infrastructure, SAP BTP credits, and transformation services |
| **RT** | Payroll Results Table — the internal SAP payroll results cluster stored after each payroll run |
| **SAML2** | Security Assertion Markup Language 2.0 — the federation standard used for SSO between EC (IdP) and S/4HANA (SP) |
| **SICF** | SAP Internet Communication Framework — the S/4HANA service catalog that registers HTTP/HTTPS-accessible services |
| **SLO** | Single Logout — the SAML2 protocol flow for logging out of the SP (S/4HANA) when the user logs out of EC |
| **SP** | Service Provider — in this scenario, S/4HANA PCE consumes the SAML assertion issued by EC |
| **TSM** | Technical Service Manager — SAP's RISE customer-facing technical contact for infrastructure requests |
| **Web Dispatcher** | SAP's load-balancing and reverse-proxy product, used to expose S/4HANA SICF services to the internet |

---

## Appendix A — Configuration Checklist

Use this checklist to track configuration progress across environments:

| # | Task | Dev | QA/Staging | Prod |
|---|---|---|---|---|
| 1 | S/4HANA release verified ≥ 2022 | ☐ | ☐ | ☐ |
| 2 | ECS4HCM add-on installed | ☐ | ☐ | ☐ |
| 3 | Payslip form active in S/4HANA (HRFORMS) | ☐ | ☐ | ☐ |
| 4 | SAML2 config done (Report RP_HRSFEC_SAML_CONFIG or manual) | ☐ | ☐ | ☐ |
| 5 | SICF services activated | ☐ | ☐ | ☐ |
| 6 | Fixed credentials removed from SICF | ☐ | ☐ | ☐ |
| 7 | SR raised with RISE TSM for Web Dispatcher exposure | ☐ | ☐ | ☐ |
| 8 | Web Dispatcher URL confirmed and tested | ☐ | ☐ | ☐ |
| 9 | ACS URL registered in SF Provisioning | ☐ | ☐ | ☐ |
| 10 | Payroll System Configuration created in EC | ☐ | ☐ | ☐ |
| 11 | Pay Statement portlet configured | ☐ | ☐ | ☐ |
| 12 | Payroll services picklist verified (external code = 1) | ☐ | ☐ | ☐ |
| 13 | RBP granted for Pay Statement — employees | ☐ | ☐ | ☐ |
| 14 | People Profile portlet added | ☐ | ☐ | ☐ |
| 15 | Latest Homepage Quick Action enabled | ☐ | ☐ | ☐ |
| 16 | "Proxy Now" test passed in EC Admin Center | ☐ | ☐ | ☐ |
| 17 | Employee UAT — payslip renders correctly | ☐ | ☐ | ☐ |
| 18 | Security checklist completed | ☐ | ☐ | ☐ |

---

## Appendix B — T-code Quick Reference

| T-code | System | Purpose |
|---|---|---|
| `SAML2` | S/4HANA PCE | Configure SAML2 local provider and trusted IdPs |
| `SICF` | S/4HANA PCE | Activate and configure HTTP/HTTPS services |
| `SA38` | S/4HANA PCE | Run ABAP programs (e.g., `RP_HRSFEC_SAML_CONFIG`) |
| `PC_PAYRESULT` | S/4HANA PCE | View payroll results for a specific employee |
| `HRFORMS` | S/4HANA PCE | Manage HR forms (payslip form activation by Molga) |
| `SM19` | S/4HANA PCE | Security Audit Log configuration |
| `STRUST` | S/4HANA PCE | Trust Manager — manage certificates |
| `SU01` | S/4HANA PCE | User maintenance — verify employee user email mapping |

---

*This document is part of the SuccessLabs Academy Advisory Series.*  
*All configuration steps are based on SAP standard documentation as of October 2026.*  
*Configuration paths may vary slightly between S/4HANA releases — always verify against your system release notes.*

**Document Code:** `INT-PAYSLIP-001` | **Version:** `v1.0`  
*Prepared by: SuccessLabs Academy Advisory Board | All rights reserved.*
