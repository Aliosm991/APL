# End-to-End Architectural Analysis: SBX APL Approval Platform Solution

## 1. Executive Summary & Solution Overview

The **SBX APL Approval Platform** (`SBXAPLApprovalPlatformV7_8_0_0_6.zip`) is an enterprise-grade Microsoft Power Platform solution designed for automated processing, multi-tier routing, approval, document generation, and reconciliation of **Approved Price Lists (APL)** and **IT Requests**.

### Key Architectural Capabilities
* **Multi-App Architecture**: Contains 3 Canvas Apps representing the solution's evolution:
  * `sbxapl_sbxaplapprovalappv7` (Original V7 Canvas App utilizing AutoLayout flex containers).
  * `sbxapl_sbxaplapprovalappv8` (V8 Production Release featuring a deterministic ManualLayout home screen `scrHomeV8` with 163 fixed-canvas controls to eliminate layout shift and PA2108 property errors).
  * `sbxapl_sbxaplapprovalplatformv9_fcf0b` (V9 Shell/Prototype introducing Price Tier master data management).
* **Automated Multi-Tier Approval Engine**: 8 Power Automate cloud flows orchestrating request intake, tier matrix evaluation, parallel and sequential approver task assignment, decision processing, status reconciliation, and custom HTML-based email notifications.
* **Hybrid Data Layer**: Integrated with 16+ SharePoint Online lists (`Requests`, `APPROVAL`, `ApprovalTasks`, `ApprovalTiersMatrix`, etc.), an Excel Online Business dataset (`dim_supplier`), and a SharePoint Document Library (`APL LIBRARY`) for generated documents and attachments.
* **External Integration**: Automated daily background synchronization of FMC catalog articles (`SBXAPL_V7_SyncFMCArticles`).

---

## 2. Solution Package & Metadata Analysis

| Attribute | Value |
| :--- | :--- |
| **Solution Unique Name** | `SBXAPLApprovalPlatformV7` |
| **Display Name** | SBX APL Approval Platform V7 |
| **Solution Version** | `8.0.0.6` |
| **Publisher Unique Name** | `SBXAPLPlatform` (Prefix: `sbxapl`, Option Value Prefix: `92500`) |
| **Package Type** | Unmanaged (`<Managed>0</Managed>`) |
| **CRM Schema Version** | `9.2.26085.166` |

### Connection References
The solution standardizes all external connector instances across all 3 Canvas Apps and 8 Power Automate flows using 7 connection references defined in `customizations.xml`:
1. `sbxapl_V7FMCSyncSharePoint` (`/providers/Microsoft.PowerApps/apis/shared_sharepointonline`)
2. `sbxapl_V7SharePoint` (`/providers/Microsoft.PowerApps/apis/shared_sharepointonline`)
3. `sbxapl_V7OneDrive` (`/providers/Microsoft.PowerApps/apis/shared_onedriveforbusiness`)
4. `sbxapl_V7Outlook` (`/providers/Microsoft.PowerApps/apis/shared_office365`)
5. `sbxapl_V7PowerBI` (`/providers/Microsoft.PowerApps/apis/shared_powerbi`)
6. `sbxapl_V7PowerBIData` (`/providers/Microsoft.PowerApps/apis/shared_powerbi`)
7. `sbxapl_V7PowerBIData2` (`/providers/Microsoft.PowerApps/apis/shared_powerbi`)

---

## 3. End-to-End System Architecture

```
+---------------------------------------------------------------------------------------------------+
|                                      POWER CANVAS APPLICATIONS                                    |
|                                                                                                   |
|   +----------------------------+   +----------------------------+   +-------------------------+   |
|   | SBX APL Approval App V7    |   | SBX APL Approval App V8    |   | SBX APL Platform V9     |   |
|   | (Legacy Flex Container UI) |   | (Production ManualLayout)  |   | (Price Tier Admin Shell)|   |
|   +----------------------------+   +----------------------------+   +-------------------------+   |
+----------------------------------------------+----------------------------------------------------+
                                               |
                                        PowerApp V2 Trigger
                                               |
+----------------------------------------------v----------------------------------------------------+
|                                      POWER AUTOMATE FLOW ENGINE                                   |
|                                                                                                   |
|  [SubmitRequest] --> Creates Parent Request, Approval Header, Approval Tasks & Uploads Attachments|
|  [ProcessApprovalDecision] --> Validates Task, Updates Task Decision, Advances Tier / Approves / Rejects
|  [SendApprovalNotification] --> Fetches Email Template, Contextual Markets, Formats Attachments & Sends
|  [ReconcileApprovalStatus] / [ReconcileAllOpenRequests] --> Automated Batch & On-Demand Status Audit
|  [GenerateRequestDocument] --> Compiles Request Metadata into HTML/PDF Document stored in Library  |
|  [SyncFMCArticles] --> Incremental Daily Sync of FMC Articles Catalog                            |
+----------------------------------------------+----------------------------------------------------+
                                               |
                                     SharePoint / Excel / Outlook
                                               |
+----------------------------------------------v----------------------------------------------------+
|                                            DATA LAYER                                             |
|                                                                                                   |
|  SharePoint Site: https://alshayacom.sharepoint.com/sites/sbxapl2                                 |
|  * Requests (Central Request Register)           * APL_User_Credentials (RBAC Permissions)         |
|  * APPROVAL (Header Approval State)               * APL_EXCHANGE_RATE (Currency Conversion)         |
|  * ApprovalTasks (Sequential/Parallel Line Tasks) * SAVED_APL / SAVED_IT_REQUEST (Draft Form State) |
|  * ApprovalTiersMatrix (Tier & Routing Matrix)   * APL LIBRARY (Document Library)                  |
|  * FMC Articles (Product Catalog)                * Price Tier Master Data & Description (V9 Schema)|
|                                                                                                   |
|  Excel Online Business:                                                                           |
|  * dim_supplier ({73F30C9D-8E2D-4375-857C-37547DFAE605} - Supplier Master Dimension)                |
+---------------------------------------------------------------------------------------------------+
```

---

## 4. Data Architecture & Entity Relationship Mapping

### Primary Datasets & Table Roles

```
[Requests] (1) <=====> (1) [APPROVAL] (1) <=====> (N) [ApprovalTasks]
    ||                                                    ||
    || (1)                                                || (Evaluated against)
    ||                                                    \/
    ||<=====> (N) [APL LIBRARY]                  [ApprovalTiersMatrix]
    ||
    +=====> [SAVED_APL] / [SAVED_IT_REQUEST] (Drafts before submission)
```

1. **`Requests` (SharePoint List)**
   * **Role**: Primary transaction registry for all submitted APL and IT requests.
   * **Key Attributes**: `Title` (Request Code e.g. `APL-2026-001`), `RequestType` (`APL Request` vs `IT Request`), `RequesterEmail`, `RequesterName`, `Department`, `Status` (`In Progress`, `Approved`, `Rejected`), `SubmittedDate`, `TotalAmount`.
2. **`APPROVAL` (SharePoint List)**
   * **Role**: Parent state tracking record for approval workflow orchestration.
   * **Key Attributes**: `Title` (Request ID link), `Status` (`In Progress`, `Approved`, `Rejected`), `CurrentOrder` (Active approval tier integer), `ResponseTableHTML` (Cached HTML response matrix table for rendering), `SubmittedDate`.
3. **`ApprovalTasks` (SharePoint List)**
   * **Role**: Granular task line items generated for each required approver at each tier level.
   * **Key Attributes**: `Title` (Request ID), `Order0` (Tier level number), `ApproverEmail`, `ApproverName`, `TaskStatus` (`Pending`, `Approved`, `Rejected`), `DecisionDate`, `Comments`.
4. **`ApprovalTiersMatrix` (SharePoint List)**
   * **Role**: Configuration matrix determining approval routing rules based on request category, threshold amount, and department.
   * **Key Attributes**: `RequestType`, `TierOrder`, `ApproverRole`, `ApproverEmail`, `MinAmount`, `MaxAmount`.
5. **`APL_User_Credentials` (SharePoint List)**
   * **Role**: Custom Role-Based Access Control (RBAC) credential store.
   * **Key Attributes**: `UserEmail`, `UserRole` (`Requester`, `Approver`, `Finance`, `IT Reviewer`, `Admin`), `Department`, `IsActive`.
6. **`APL_EXCHANGE_RATE` & `dim_supplier`**
   * **Role**: Master currency lookup and Excel-based supplier master dimension table used in financial cost calculations.

---

## 5. Power Automate Cloud Flows (Deep Dive)

### 1. `SBXAPL_V7_SubmitRequest`
* **Trigger**: `PowerAppV2` (`Request ID`, `Requester Email`, `Requester Name`, `Request Type`, `Title`, `Product_Info`, `UploadedBy`, `AttachmentFile`).
* **Execution Logic**:
  1. Checks if `Requests` record exists; inserts new record if missing.
  2. Queries `APPROVAL` parent list; creates header record if missing or updates payload metadata.
  3. Queries `ApprovalTiersMatrix` matching request type and creates sequential task records in `ApprovalTasks` for each required tier.
  4. Generates initial HTML summary table of approval tasks and updates `APPROVAL.ResponseTableHTML`.
  5. Uploads attached document file to `APL LIBRARY` document library and stamps metadata fields.
  6. Returns HTTP 200 JSON success response to Power Apps.

### 2. `SBXAPL_V7_ProcessApprovalDecision`
* **Trigger**: `PowerAppV2` (`Task ID`, `Request ID`, `Approver Email`, `Decision`, `Comments`).
* **Validation Guard**: Verifies task exists, parent approval exists, task is `Pending`, caller email matches assigned `ApproverEmail`, and task `Order0` equals parent `CurrentOrder`.
* **Decision Branching**:
  * **REJECTED Branch**: Sets current task `TaskStatus = Rejected`, immediately sets parent `APPROVAL.Status = Rejected` and `Requests.Status = Rejected`.
  * **APPROVED Branch**:
    1. Sets current task `TaskStatus = Approved`.
    2. Queries remaining `Pending` tasks in current tier.
    3. If current tier has remaining pending tasks, waits for parallel approvers.
    4. If current tier is complete (0 pending tasks remaining), queries if higher tier tasks exist in `ApprovalTasks`.
    5. If higher tier exists: advances parent `APPROVAL.CurrentOrder = CurrentOrder + 1`.
    6. If no higher tier exists: updates parent `APPROVAL.Status = Approved` and `Requests.Status = Approved`.
  * Re-computes cached HTML approval matrix table and returns updated state to Power Apps.

### 3. `SBXAPL_V7_SendApprovalNotification`
* **Trigger**: `PowerAppV2` (`Request ID`, `Notification Type`, `Recipient Emails`, `Current Tier`, `Requester Email`, `Comments`, `Request Link`).
* **Execution Logic**:
  1. Retrieves email template from SharePoint master library.
  2. Routes context by request type (IT vs APL) to parse dynamic market arrays without duplicate entries.
  3. Queries attachments in `APL LIBRARY` for request, reads binary streams, and builds attachment array avoiding duplicated file names.
  4. Formats email body using standard brand palette (#123D35 primary, #D9B56D accent).
  5. Dispatches email via Office 365 Outlook connector (`Send_exact_formatted_email`).

### 4. `SBXAPL_V7_ReconcileApprovalStatus` & `SBXAPL_V7_ReconcileAllOpenRequests`
* **Execution Logic**:
  * On-demand (`ReconcileApprovalStatus`) and daily automated batch (`ReconcileAllOpenRequests` scheduled at 01:00 UTC) reconciliation routines.
  * Audits open requests (`Status eq 'In Progress'`). If any child task is `Rejected`, forces parent state to `Rejected`. If all child tasks are `Approved`, forces parent state to `Approved`. If pending tasks remain, aligns parent `CurrentOrder` to minimum pending tier.

### 5. `SBXAPL_V7_SyncFMCArticles`
* **Trigger**: Recurrence (Daily scheduled job at 07:00 UTC).
* **Execution Logic**: Incremental synchronization job querying highest `ArticleID` in `FMC Articles` SharePoint list, executing incremental SQL batch query against FMC catalog database, filtering valid new records, and writing new article catalog items to SharePoint.

---

## 6. Canvas Applications Comparison & UX Analysis

### App Version Comparison Matrix

| Feature / Aspect | SBX APL Approval App V7 | SBX APL Approval App V8 | SBX APL Platform V9 |
| :--- | :--- | :--- | :--- |
| **App Schema Logical Name** | `sbxapl_sbxaplapprovalappv7` | `sbxapl_sbxaplapprovalappv8` | `sbxapl_sbxaplapprovalplatformv9_fcf0b` |
| **Primary Home Screen** | `scrHome` / `scrHome_new` | `scrHomeV8` | `scrHomeV9` |
| **Layout Strategy** | AutoLayout / Flex Containers | Deterministic Fixed-Canvas ManualLayout (Width=1600) | Simplified Shell Layout |
| **Controls Count on Home** | Dynamic flex controls | 163 explicitly computed controls | 28 controls |
| **Navigation Mapping** | Points to `scrHome` | Updated 12 Navigate() calls across 8 screens to `scrHomeV8` | Independent prototype |
| **AppChecker Issues** | 969 total issues | 569 total issues | 99 total issues |
| **Target Role** | Legacy Production App | Current Production App | Price Tier Admin Module |

### Architectural Significance of V8 (`scrHomeV8`)
In canvas app development, container-based flex layouts (`AutoLayout`) can experience PA2108 property resolution errors and unexpected element reflows when complex formulas evaluate asynchronously during startup. V8 solved this by engineering `scrHomeV8` using fixed-canvas pixel absolute positioning (`ManualLayout`). Every one of the 163 controls on `scrHomeV8` uses explicit X/Y/Width/Height values, completely eliminating startup visual reflows while preserving full formula functionality.

---

## 7. Security, Access Control & Credential Management

* **Authentication & Authorization**: Built on Microsoft 365 Azure AD / Entra ID identity.
* **Custom Role-Based Access Control (RBAC)**: App startup (`App.OnStart`) fetches permissions from `APL_User_Credentials`:
  ```powerapps
  ClearCollect(USERS, APL_User_Credentials);
  ```
  App controls condition visibility and edit capabilities based on user role lookups against `USERS` collection:
  * **Requesters**: Can create/edit drafts (`SAVED_APL`, `SAVED_IT_REQUEST`) and submit requests.
  * **Approvers**: Access restricted to `scrApprovals` for tasks where `ApproverEmail` matches `User().Email`.
  * **Admins**: Granted administrative access to master data screens and manual reconciliation triggers.

---

## 8. Identified Gaps, Technical Debt & Risks

1. **Custom RBAC Storage in SharePoint List (`APL_User_Credentials`)**
   * *Risk*: Users with read access to the SharePoint site can view or bypass frontend visibility restrictions if direct SharePoint access is not locked down via list permissions.
2. **App.OnStart Heavy Loading & Delegation Warnings**
   * *Risk*: `App.OnStart` executes synchronous `ClearCollect()` calls on full tables. `AppChecker` flagged 33 delegation warnings (`app-SuggestRemoteExecutionHint`) where `Distinct()` or `Filter()` operations risk exceeding the 500/2000 record delegation threshold.
3. **Accessibility Compliance Debt**
   * *Risk*: High count of accessibility warnings (over 200+ missing AccessibleLabels and TabIndices across screens), impacting screen reader usability and M365 compliance.
4. **Hardcoded App URL References**
   * *Risk*: `App.OnStart` sets `varAppShareBaseUrl` to a hardcoded tenant app play URL (`https://apps.powerapps.com/play/e/default-8ce6dfd4-5866-4fdb-90d0-fb4a65d44617...`), which breaks when migrating across environments (Dev -> Test -> Prod).

---

## 9. Strategic Recommendations & Optimization Roadmap

### Phase 1: Immediate Maintenance & Hardening (1 - 2 Weeks)
1. **Environment Variable Strategy**: Replace hardcoded URL string (`varAppShareBaseUrl`) in `App.OnStart` with a Power Platform Environment Variable.
2. **Delegation Fixes**: Replace non-delegatable Power Fx queries in `scrAPLRequest` and `scrITRequest` with delegatable SharePoint queries or indexed view collections.
3. **Flow Error Handling**: Add `Configure Run After` blocks in `SBXAPL_V7_ProcessApprovalDecision` and `SBXAPL_V7_SubmitRequest` to catch flow failures and return structured error messages back to Power Apps.

### Phase 2: Security & Architecture Modernization (1 - 2 Months)
1. **Migration to Dataverse or Enhanced Security**: Transition sensitive tables (`Requests`, `APPROVAL`, `APL_User_Credentials`) to Microsoft Dataverse or enforce SharePoint Role-Based Item-Level Security to prevent unauthorized direct list edits.
2. **Accessibility Remediation**: Batch-update missing `AccessibleLabel` and `TabIndex` properties on interactive controls in `scrHomeV8`, `scrAPLRequest`, and `scrApprovals`.

### Phase 3: Solution Consolidation (3 Months)
1. **App Package Cleanup**: Remove deprecated V7 app (`sbxapl_sbxaplapprovalappv7`) from the solution manifest once V8 adoption is fully validated, reducing solution zip size by ~50%.
2. **Modularized Component Library**: Extract `cmpAttachmentManager` and `cmpSavedRequestDialog` into a shared Power Platform Component Library for enterprise reuse.

---
*Report generated automatically following end-to-end extraction, schema inspection, and workflow logic analysis of SBX APL Approval Platform V7/V8/V9.*
