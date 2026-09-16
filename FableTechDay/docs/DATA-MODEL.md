# Data model and control mapping

All data in the dashboard is sample data authored as constants at the top of the `<script>` in `index.html`. This document describes the schema, the derivation rules, and the full mapping of every finding to its controls, generated from those constants on 2026-09-16. When the constants change, regenerate the tables here (see the last section).

Related: [ARCHITECTURE.md](ARCHITECTURE.md) for how the data flows through the two deliverables, [../README.md](../README.md) for the user tour.

## 1. Constants

| Constant | Type | Rows | Purpose |
|---|---|---|---|
| `FINDINGS` | array of finding objects | 62 | The core dataset. Every tile, chart, table and rollup derives from it. |
| `CVES` | array | 15 | Defender Vulnerability Management table on the Defender tab. |
| `SECURE_SCORE` | object | 2 scores | Defender for Cloud secure score with 8 security controls; Microsoft Secure Score with 4 categories. |
| `DFC_CIS` | object | 16 controls | What Defender for Cloud's built-in CIS regulatory compliance standard reports. Shown separately from the mapping. |
| `STATS` | object | entra, defender, ai | Headline numbers for KPI tiles that are not derivable from findings. |
| `AI_ALERTS` | array | 7 | Defender for AI services alert types and counts, last 30 days. |
| `AGENT_PLATFORMS` | array | 3 | Agent identities by platform and risk. Totals must equal `STATS.ai.agentIdentities` (52). |
| `AGENTS` | array | 10 | Representative agent inventory rows; `findingIds` link each row to findings. |
| `CONTROL_NAMES` | object | 122 keys | Control ID to short title for every ID used in the mapping arrays. |
| `FRAMEWORKS` | object | 5 | Framework definitions. Entries with `taxonomy: true` carry their fixed list of 10 category IDs. |

Derived at load: `BENCHMARKS` (frameworks without `taxonomy`), `TAXONOMIES` (with `taxonomy`), `DOMAINS`, `SEVERITIES`, `SEV_RANK`, `SEV_WEIGHT`, `TABS`.

## 2. Finding schema

```js
{
  id: 'AI-01',                 // E-nn Entra ID, A-nn Azure, D-nn Defender, AI-nn AI & Agents
  domain: 'ai',                // 'entra' | 'azure' | 'defender' | 'ai'
  category: 'AI Resources',    // free text; drives the "open findings by category" chart per domain
  title: '...',
  severity: 'high',            // 'critical' | 'high' | 'medium' | 'low'
  status: 'fail',              // 'fail' | 'warn' (partially met / needs review) | 'pass'
  affected: 5,                 // count; 0 for pass
  affectedUnit: 'AI resources',// free text: users, resources, devices, subscriptions, policies, agents, ...
  scope: 'Subscriptions: ...', // tenant, subscription or resource-group text
  source: 'Defender for Cloud (AI SPM)',
  detected: '2026-09-14',      // ISO date
  description: '...',
  remediation: '...',          // sentences; the drawer splits them into numbered steps
  mcsb: ['NS-2'],              // Microsoft cloud security benchmark control IDs
  cisAzure: [],                // CIS Microsoft Azure Foundations Benchmark v3.0 recommendation numbers
  cisM365: [],                 // CIS Microsoft 365 Foundations Benchmark v4 recommendation numbers
  owaspLlm: ['LLM10'],         // OWASP Top 10 for LLM Applications 2025 (taxonomy)
  owaspAgentic: [],            // OWASP Top 10 for Agentic Applications 2025 (taxonomy)
  atlas: ['AML.T0040'],        // MITRE ATLAS technique IDs; drawer only, not a framework
}
```

Mapping arrays may be omitted in the source; a normalization loop after the array fills missing ones with `[]`.

### Enumerations

| Field | Values | Notes |
|---|---|---|
| `domain` | entra, azure, defender, ai | Labels: Entra ID, Azure resources, Defender, AI & Agents. Colours blue, orange, aqua, violet; fixed, never reassigned on filter. |
| `severity` | critical, high, medium, low | Rank 0-3 for sorting; weights 4, 3, 2, 1 for the findings score. |
| `status` | fail, warn, pass | `warn` counts as open in filters and half-weight in the score. |

### Domain totals

| Domain | Fail | Warn | Pass | Total |
|---|---|---|---|---|
| Entra ID | 11 | 1 | 4 | 16 |
| Azure resources | 14 | 1 | 4 | 19 |
| Defender | 8 | 1 | 3 | 12 |
| AI & Agents | 13 | 1 | 1 | 15 |
| **All** | **46** | **4** | **12** | **62** |

## 3. Derivation rules

| Value | Rule |
|---|---|
| Findings score | `100 − Σ weight(open) / Σ weight(all)`, where weight is 4/3/2/1 by severity and a `warn` finding contributes half its weight. Currently 10. |
| Overall posture | Mean of the findings score, Defender for Cloud secure score % (34/58 = 59) and Microsoft Secure Score % (412/731 = 56). Currently 42. |
| Control state (benchmarks) | For each control ID referenced by at least one finding: `fail` if any mapped finding fails, else `warn` if any is warn, else `pass`. |
| Framework rollup % | Passing controls ÷ referenced controls. Controls no finding references are not counted, so this is not benchmark coverage. |
| Taxonomy coverage (OWASP) | For each of the 10 fixed category IDs: the control state above if referenced, otherwise `not assessed`. Reported as "x of 10 assessed, y with open fails". Never a percentage; excluded from the rollup. |
| Category chart value | Entra, Defender, AI: number of open findings per category. Azure: sum of `affected` per category (unhealthy resources). |
| Agent risk | Authored per row in `AGENTS`; the platform chart uses `AGENT_PLATFORMS`. |

### Current rollup results

| Framework | Kind | Passing | Partial | Failing | Referenced |
|---|---|---|---|---|---|
| Microsoft cloud security benchmark (MCSB) | benchmark | 3 | 3 | 29 | 35 |
| CIS Microsoft Azure Foundations v3.0 | benchmark | 5 | 1 | 31 | 37 |
| CIS Microsoft 365 Foundations v4 | benchmark | 5 | 1 | 19 | 25 |
| OWASP Top 10 for LLM Applications 2025 | taxonomy | 0 | 1 | 6 | 7 of 10 assessed |
| OWASP Top 10 for Agentic Applications 2025 | taxonomy | 0 | 0 | 7 | 7 of 10 assessed |

## 4. Findings and their mappings

| ID | Domain | Category | Title | Severity | Status | Affected | MCSB | CIS Azure v3.0 | CIS M365 v4 | OWASP LLM | OWASP Agentic |
|---|---|---|---|---|---|---|---|---|---|---|---|
| E-01 | Entra ID | MFA | MFA not enforced for all users | critical | fail | 568 users | IM-6 | 2.2.5, 2.1.3 | 5.2.2.2, 5.2.3.4 | - | - |
| E-02 | Entra ID | MFA | Administrators not required to use phishing-resistant MFA | high | fail | 7 users | IM-6, PA-1 | 2.2.4 | 5.2.2.1, 5.2.2.5 | - | - |
| E-03 | Entra ID | Legacy Auth | Legacy authentication not blocked | high | fail | 41 users | IM-6 | - | 5.2.2.3, 6.5.4 | - | - |
| E-04 | Entra ID | Privileged Roles | Too many Global Administrators | high | fail | 9 users | PA-1 | 2.26 | 1.1.3 | - | - |
| E-05 | Entra ID | PIM | Permanent active assignments to privileged roles | high | fail | 14 users | PA-2 | - | 5.3.1 | - | - |
| E-06 | Entra ID | PIM | No approval required to activate Global Administrator | medium | fail | 1 policies | PA-2 | - | 5.3.4 | - | - |
| E-07 | Entra ID | Conditional Access | Emergency access accounts not defined or not excluded from Conditional Access | medium | warn | - | PA-5 | - | 1.1.2 | - | - |
| E-08 | Entra ID | App Consent | Users can consent to apps accessing company data | medium | fail | 1 policies | IM-3 | 2.12 | 5.1.5.1, 5.1.5.2 | - | - |
| E-09 | Entra ID | Guest Access | Guest invite restrictions allow any member or guest to invite | medium | fail | 1 policies | PA-3 | 2.16 | 5.1.6.3 | - | - |
| E-10 | Entra ID | Guest Access | Inactive guest accounts with no access reviews | medium | fail | 61 users | PA-4 | 2.4 | 5.3.2 | - | - |
| E-11 | Entra ID | Identity Protection | Risk-based Conditional Access policies not enforced | high | fail | 2 policies | IM-7 | 2.2.6 | 5.2.2.6, 5.2.2.7 | - | - |
| E-12 | Entra ID | Conditional Access | No MFA requirement for Windows Azure Service Management API | high | fail | 1 policies | IM-6 | 2.2.7 | - | - | - |
| E-13 | Entra ID | Tenant Settings | Non-admin users restricted from creating tenants | low | pass | - | PA-7 | 2.3 | 5.1.2.3 | - | - |
| E-14 | Entra ID | Tenant Settings | Security Defaults disabled because Conditional Access is in use | low | pass | - | IM-6 | 2.1.1 | 5.1.1.1 | - | - |
| E-15 | Entra ID | Tenant Settings | Password hash synchronization enabled | low | pass | - | IM-1 | - | 5.1.8.1 | - | - |
| E-16 | Entra ID | Conditional Access | Sign-in frequency and persistent browser session controlled for admins | low | pass | - | IM-7 | - | 5.2.2.9 | - | - |
| A-01 | Azure resources | Storage | Secure transfer (HTTPS) not required on storage accounts | high | fail | 4 resources | DP-3 | 4.1 | - | - | - |
| A-02 | Azure resources | Storage | Anonymous blob access allowed on storage accounts | critical | fail | 2 resources | NS-2 | 4.17 | - | - | - |
| A-03 | Azure resources | Storage | Storage account default network rule is Allow | medium | fail | 11 resources | NS-2 | 4.7 | - | - | - |
| A-04 | Azure resources | Network | NSG allows RDP (3389) from the Internet | critical | fail | 6 resources | NS-1, NS-3 | 7.1 | - | - | - |
| A-05 | Azure resources | Network | NSG allows SSH (22) from the Internet | high | fail | 4 resources | NS-1 | 7.2 | - | - | - |
| A-06 | Azure resources | Key Vault | Purge protection not enabled on key vaults | high | fail | 3 resources | DP-8 | 9.5 | - | - | - |
| A-07 | Azure resources | Key Vault | Secrets and keys without expiration dates | medium | fail | 49 resources | DP-6 | 9.1, 9.3 | - | - | - |
| A-08 | Azure resources | SQL | Auditing disabled on SQL servers | high | fail | 3 resources | LT-3 | 5.1.1 | - | - | - |
| A-09 | Azure resources | SQL | SQL server firewall allows all Azure services or 0.0.0.0 range | critical | fail | 2 resources | NS-2 | 5.1.2, 5.1.7 | - | - | - |
| A-10 | Azure resources | Compute | Virtual machines missing system updates | high | fail | 27 resources | PV-6 | 3.1.1.1 | - | - | - |
| A-11 | Azure resources | Compute | Virtual machines without endpoint protection | high | fail | 8 resources | ES-1, ES-2 | 8.8, 3.1.3.3 | - | - | - |
| A-12 | Azure resources | Logging & Monitoring | Activity log not exported to Log Analytics | high | fail | 2 subscriptions | LT-3, LT-5 | 6.1.1 | - | - | - |
| A-13 | Azure resources | Logging & Monitoring | Activity log alerts for critical operations missing | medium | fail | 5 subscriptions | LT-3, IR-2 | 6.2.1, 6.2.3, 6.2.7 | - | - | - |
| A-14 | Azure resources | Defender Plans | Defender for Cloud plans not enabled on all subscriptions | high | fail | 5 subscriptions | LT-1 | 3.1.3.1, 3.1.5.1, 3.1.8.1 | - | - | - |
| A-15 | Azure resources | Storage | Blob soft delete enabled on all storage accounts | low | pass | - | BR-1 | 4.10 | - | - | - |
| A-16 | Azure resources | Compute | Trusted Launch not enabled on eligible VMs | low | warn | 61 resources | PV-3 | 8.11 | - | - | - |
| A-17 | Azure resources | Storage | Storage accounts enforce TLS 1.2 minimum | low | pass | - | DP-3 | 4.15 | - | - | - |
| A-18 | Azure resources | SQL | Transparent Data Encryption enabled on all SQL databases | low | pass | - | DP-4 | - | - | - | - |
| A-19 | Azure resources | Network | Network Watcher enabled in all deployed regions | low | pass | - | LT-4 | 7.6 | - | - | - |
| D-01 | Defender | Vulnerabilities | Devices exposed to critical CVEs with public exploits | critical | fail | 214 devices | PV-5, PV-6 | 3.1.3.2 | - | - | - |
| D-02 | Defender | Vulnerabilities | Devices missing security updates for more than 30 days | high | fail | 96 devices | PV-6 | 3.1.1.1 | - | - | - |
| D-03 | Defender | Device Health | Devices running end-of-support operating systems | high | fail | 11 devices | PV-6 | - | - | - | - |
| D-04 | Defender | Endpoint Protection | Servers not onboarded to Defender for Endpoint | high | fail | 18 devices | ES-1 | 3.1.3.1 | - | - | - |
| D-05 | Defender | Endpoint Protection | Tamper protection disabled on devices | medium | fail | 22 devices | ES-2 | - | - | - | - |
| D-06 | Defender | Endpoint Protection | Attack surface reduction rules in audit mode only | medium | warn | 340 devices | ES-2, PV-1 | - | - | - | - |
| D-07 | Defender | Incidents | High-severity incidents open more than 7 days without assignment | high | fail | 3 policies | IR-3, IR-4 | - | - | - | - |
| D-08 | Defender | Email & Collaboration | Safe Links and Safe Attachments not applied to all users | medium | fail | 1 policies | - | - | 2.1.1, 2.1.4 | - | - |
| D-09 | Defender | Email & Collaboration | Priority account protection not enabled | low | fail | 1 policies | - | - | 2.4.1 | - | - |
| D-10 | Defender | Secure Score | Microsoft 365 unified audit log enabled | low | pass | - | LT-3 | - | 3.1.1 | - | - |
| D-11 | Defender | Endpoint Protection | Workstations onboarded to Defender for Endpoint | low | pass | - | ES-1 | - | - | - | - |
| D-12 | Defender | Endpoint Protection | Cloud-delivered protection and automatic sample submission enabled | low | pass | - | ES-2 | - | - | - | - |
| AI-01 | AI & Agents | AI Resources | Public network access enabled on Azure OpenAI / AI Services accounts | high | fail | 5 AI resources | NS-2 | - | - | LLM10 | - |
| AI-02 | AI & Agents | AI Resources | Key-based authentication enabled on AI accounts; keys found in app settings | high | fail | 7 AI resources | IM-1, IM-3, IM-8 | - | - | - | ASI03 |
| AI-03 | AI & Agents | Content Safety | Prompt Shields disabled on model deployments through a custom content filter | high | fail | 4 deployments | - | - | - | LLM01 | ASI01 |
| AI-04 | AI & Agents | AI Resources | Diagnostic logs not collected for AI accounts | high | fail | 5 AI resources | LT-3 | - | - | - | - |
| AI-05 | AI & Agents | Defender for AI | Defender for AI services plan not enabled on all subscriptions | high | fail | 3 subscriptions | LT-1 | - | - | LLM01, LLM02 | - |
| AI-06 | AI & Agents | Content Safety | Jailbreak attempts detected on production deployments | medium | warn | 2 deployments | LT-1 | - | - | LLM01, LLM07 | ASI01 |
| AI-07 | AI & Agents | Purview DSPM for AI | Sensitive data in prompts to Copilot and third-party AI apps without a DLP policy | high | fail | 1,284 prompts | DP-1, DP-2 | - | 3.2.1 | LLM02 | - |
| AI-08 | AI & Agents | Purview DSPM for AI | Confidential SharePoint sites discoverable by Microsoft 365 Copilot org-wide | high | fail | 23 sites | DP-1, PA-7 | - | - | LLM02, LLM08 | - |
| AI-09 | AI & Agents | Copilot Studio Agents | Copilot Studio agents shared with the whole organization or published without authentication | high | fail | 9 agents | IM-1, PA-7 | - | - | LLM06 | ASI03, ASI09 |
| AI-10 | AI & Agents | Copilot Studio Agents | Agents use premium connectors in the default environment without a DLP policy | high | fail | 9 agents | PA-7, DP-2 | - | - | LLM06 | ASI02, ASI04 |
| AI-11 | AI & Agents | Agent Identities | Agent identities without an owner or sponsor | medium | fail | 17 agent identities | IM-3, PA-4 | - | - | - | ASI03, ASI10 |
| AI-12 | AI & Agents | Agent Identities | Over-permissioned agent identities using long-lived client secrets | critical | fail | 12 agent identities | IM-3, IM-8, PA-7 | - | - | LLM06 | ASI03 |
| AI-13 | AI & Agents | MCP & Tools | Agents connect to MCP servers outside the approved catalog | medium | fail | 2 MCP servers | AM-2, NS-2 | - | - | LLM03 | ASI02, ASI04, ASI07 |
| AI-14 | AI & Agents | Shadow AI | Unsanctioned generative AI apps in use | medium | fail | 28 apps | AM-2, DP-2 | - | - | LLM02 | - |
| AI-15 | AI & Agents | Model Deployments | All model deployments on supported model versions | low | pass | - | - | - | - | LLM03 | - |

MITRE ATLAS technique IDs (`atlas`) appear on AI findings only and are shown in the detail drawer: AML.T0040 AI Model Inference API Access, AML.T0051 LLM Prompt Injection, AML.T0053 LLM Plugin Compromise, AML.T0054 LLM Jailbreak, AML.T0055 Unsecured Credentials, AML.T0057 LLM Data Leakage.

## 5. Controls by framework (as rolled up)

### 5.1 Microsoft cloud security benchmark (MCSB): 3 passing, 3 partial, 29 failing of 35 referenced

| Control | Title | State | Findings |
|---|---|---|---|
| AM-2 | Use only approved services | fail | AI-13, AI-14 |
| BR-1 | Ensure regular automated backups | pass | A-15 |
| DP-1 | Discover, classify, and label sensitive data | fail | AI-07, AI-08 |
| DP-2 | Monitor anomalies and threats targeting sensitive data | fail | AI-07, AI-10, AI-14 |
| DP-3 | Encrypt sensitive data in transit | fail | A-01, A-17 |
| DP-4 | Enable data at rest encryption by default | pass | A-18 |
| DP-6 | Use a secure key management process | fail | A-07 |
| DP-8 | Ensure security of key and certificate repository | fail | A-06 |
| ES-1 | Use Endpoint Detection and Response (EDR) | fail | A-11, D-04, D-11 |
| ES-2 | Use modern anti-malware software | fail | A-11, D-05, D-06, D-12 |
| IM-1 | Use centralized identity and authentication system | fail | E-15, AI-02, AI-09 |
| IM-3 | Manage application identities securely and automatically | fail | E-08, AI-02, AI-11, AI-12 |
| IM-6 | Use strong authentication controls | fail | E-01, E-02, E-03, E-12, E-14 |
| IM-7 | Restrict resource access based on conditions | fail | E-11, E-16 |
| IM-8 | Restrict the exposure of credentials and secrets | fail | AI-02, AI-12 |
| IR-2 | Preparation: set up incident notification | fail | A-13 |
| IR-3 | Detection and analysis: create incidents based on high-quality alerts | fail | D-07 |
| IR-4 | Detection and analysis: investigate an incident | fail | D-07 |
| LT-1 | Enable threat detection capabilities | fail | A-14, AI-05, AI-06 |
| LT-3 | Enable logging for security investigation | fail | A-08, A-12, A-13, D-10, AI-04 |
| LT-4 | Enable network logging for security investigation | pass | A-19 |
| LT-5 | Centralize security log management and analysis | fail | A-12 |
| NS-1 | Establish network segmentation boundaries | fail | A-04, A-05 |
| NS-2 | Secure cloud services with network controls | fail | A-02, A-03, A-09, AI-01, AI-13 |
| NS-3 | Deploy firewall at the edge of enterprise network | fail | A-04 |
| PA-1 | Separate and limit highly privileged/administrative users | fail | E-02, E-04 |
| PA-2 | Avoid standing access for user accounts and permissions | fail | E-05, E-06 |
| PA-3 | Manage lifecycle of identities and entitlements | fail | E-09 |
| PA-4 | Review and reconcile user access regularly | fail | E-10, AI-11 |
| PA-5 | Set up emergency access | warn | E-07 |
| PA-7 | Follow just enough administration (least privilege) principle | fail | E-13, AI-08, AI-09, AI-10, AI-12 |
| PV-1 | Define and establish secure configurations | warn | D-06 |
| PV-3 | Define and establish secure configurations for compute resources | warn | A-16 |
| PV-5 | Perform vulnerability assessments | fail | D-01 |
| PV-6 | Rapidly and automatically remediate vulnerabilities | fail | A-10, D-01, D-02, D-03 |

### 5.2 CIS Microsoft Azure Foundations Benchmark v3.0: 5 passing, 1 partial, 31 failing of 37 referenced

| Control | Title | State | Findings |
|---|---|---|---|
| 2.1.1 | Ensure Security Defaults is enabled (if Conditional Access is not used) — see known issue below | pass | E-14 |
| 2.1.3 | Ensure MFA is enabled for all users | fail | E-01 |
| 2.2.4 | Ensure a Conditional Access policy requires MFA for administrators | fail | E-02 |
| 2.2.5 | Ensure a Conditional Access policy requires MFA for all users | fail | E-01 |
| 2.2.6 | Ensure a Conditional Access policy requires MFA for risky sign-ins | fail | E-11 |
| 2.2.7 | Ensure a Conditional Access policy requires MFA for Azure management | fail | E-12 |
| 2.3 | Ensure non-admin users are restricted from creating tenants | pass | E-13 |
| 2.4 | Ensure guest users are reviewed on a regular basis | fail | E-10 |
| 2.12 | Ensure user consent for applications is restricted | fail | E-08 |
| 2.16 | Ensure guest invite restrictions are set to admins or Guest Inviter role | fail | E-09 |
| 2.26 | Ensure fewer than 5 users have Global Administrator assigned | fail | E-04 |
| 3.1.1.1 | Ensure Auto provisioning / Machine configuration remediates missing system updates | fail | A-10, D-02 |
| 3.1.3.1 | Ensure Microsoft Defender for Servers is set to On | fail | A-14, D-04 |
| 3.1.3.2 | Ensure Defender Vulnerability Management is enabled | fail | D-01 |
| 3.1.3.3 | Ensure Endpoint Protection is installed on all VMs | fail | A-11 |
| 3.1.5.1 | Ensure Microsoft Defender for Storage is set to On | fail | A-14 |
| 3.1.8.1 | Ensure Microsoft Defender for Key Vault is set to On | fail | A-14 |
| 4.1 | Ensure Secure transfer required is set to Enabled | fail | A-01 |
| 4.7 | Ensure default network access rule for storage accounts is set to Deny | fail | A-03 |
| 4.10 | Ensure soft delete is enabled for Azure Blob storage | pass | A-15 |
| 4.15 | Ensure the Minimum TLS version for storage accounts is set to 1.2 | pass | A-17 |
| 4.17 | Ensure Allow Blob anonymous access is set to Disabled | fail | A-02 |
| 5.1.1 | Ensure Auditing is set to On for SQL servers | fail | A-08 |
| 5.1.2 | Ensure no SQL server firewall rule allows 0.0.0.0/0 | fail | A-09 |
| 5.1.7 | Ensure public network access on SQL servers is disabled | fail | A-09 |
| 6.1.1 | Ensure a Diagnostic Setting exists for subscription Activity logs | fail | A-12 |
| 6.2.1 | Ensure an Activity Log alert exists for Create/Update Policy Assignment | fail | A-13 |
| 6.2.3 | Ensure an Activity Log alert exists for Create/Update Network Security Group | fail | A-13 |
| 6.2.7 | Ensure an Activity Log alert exists for Create/Update SQL Server Firewall Rule | fail | A-13 |
| 7.1 | Ensure RDP access from the Internet is restricted | fail | A-04 |
| 7.2 | Ensure SSH access from the Internet is restricted | fail | A-05 |
| 7.6 | Ensure Network Watcher is enabled for all regions | pass | A-19 |
| 8.8 | Ensure Endpoint Protection for all Virtual Machines is installed | fail | A-11 |
| 8.11 | Ensure Trusted Launch is enabled on Virtual Machines | warn | A-16 |
| 9.1 | Ensure expiration date is set for all keys in RBAC key vaults | fail | A-07 |
| 9.3 | Ensure expiration date is set for all secrets in RBAC key vaults | fail | A-07 |
| 9.5 | Ensure the key vault is recoverable (purge protection) | fail | A-06 |

### 5.3 CIS Microsoft 365 Foundations Benchmark v4: 5 passing, 1 partial, 19 failing of 25 referenced

| Control | Title | State | Findings |
|---|---|---|---|
| 1.1.2 | Ensure two emergency access accounts have been defined | warn | E-07 |
| 1.1.3 | Ensure that between two and four global admins are designated | fail | E-04 |
| 2.1.1 | Ensure Safe Links for Office applications is enabled | fail | D-08 |
| 2.1.4 | Ensure Safe Attachments policy is enabled | fail | D-08 |
| 2.4.1 | Ensure Priority account protection is enabled | fail | D-09 |
| 3.1.1 | Ensure Microsoft 365 audit log search is enabled | pass | D-10 |
| 3.2.1 | Ensure DLP policies are enabled | fail | AI-07 |
| 5.1.1.1 | Ensure Security Defaults is disabled when Conditional Access is used | pass | E-14 |
| 5.1.2.3 | Ensure Restrict non-admin users from creating tenants is Yes | pass | E-13 |
| 5.1.5.1 | Ensure user consent to apps accessing company data is not allowed | fail | E-08 |
| 5.1.5.2 | Ensure the admin consent workflow is enabled | fail | E-08 |
| 5.1.6.3 | Ensure guest invite restrictions are configured | fail | E-09 |
| 5.1.8.1 | Ensure password hash sync is enabled for hybrid deployments | pass | E-15 |
| 5.2.2.1 | Ensure MFA is enabled for administrative roles | fail | E-02 |
| 5.2.2.2 | Ensure MFA is enabled for all users | fail | E-01 |
| 5.2.2.3 | Enable Conditional Access policies to block legacy authentication | fail | E-03 |
| 5.2.2.5 | Ensure phishing-resistant MFA strength is required for administrators | fail | E-02 |
| 5.2.2.6 | Enable Identity Protection user risk policies | fail | E-11 |
| 5.2.2.7 | Enable Identity Protection sign-in risk policies | fail | E-11 |
| 5.2.2.9 | Ensure sign-in frequency is enabled and browser sessions are not persistent for administrators | pass | E-16 |
| 5.2.3.4 | Ensure all member users are MFA capable | fail | E-01 |
| 5.3.1 | Ensure Privileged Identity Management is used to manage roles | fail | E-05 |
| 5.3.2 | Ensure access reviews for guest users are configured | fail | E-10 |
| 5.3.4 | Ensure approval is required for Global Administrator role activation | fail | E-06 |
| 6.5.4 | Ensure SMTP AUTH is disabled | fail | E-03 |

### 5.4 OWASP Top 10 for LLM Applications 2025 (taxonomy): 7 of 10 assessed, 6 with open fails

| Category | Title | State | Findings |
|---|---|---|---|
| LLM01 | Prompt Injection | fail | AI-03, AI-05, AI-06 |
| LLM02 | Sensitive Information Disclosure | fail | AI-05, AI-07, AI-08, AI-14 |
| LLM03 | Supply Chain | fail | AI-13, AI-15 |
| LLM04 | Data and Model Poisoning | not assessed | - |
| LLM05 | Improper Output Handling | not assessed | - |
| LLM06 | Excessive Agency | fail | AI-09, AI-10, AI-12 |
| LLM07 | System Prompt Leakage | warn | AI-06 |
| LLM08 | Vector and Embedding Weaknesses | fail | AI-08 |
| LLM09 | Misinformation | not assessed | - |
| LLM10 | Unbounded Consumption | fail | AI-01 |

### 5.5 OWASP Top 10 for Agentic Applications 2025 (taxonomy): 7 of 10 assessed, 7 with open fails

| Category | Title | State | Findings |
|---|---|---|---|
| ASI01 | Agent Goal Hijack | fail | AI-03, AI-06 |
| ASI02 | Tool Misuse and Exploitation | fail | AI-10, AI-13 |
| ASI03 | Identity and Privilege Abuse | fail | AI-02, AI-09, AI-11, AI-12 |
| ASI04 | Agentic Supply Chain Vulnerabilities | fail | AI-10, AI-13 |
| ASI05 | Unexpected Code Execution | not assessed | - |
| ASI06 | Memory and Context Poisoning | not assessed | - |
| ASI07 | Insecure Inter-Agent Communication | fail | AI-13 |
| ASI08 | Cascading Failures | not assessed | - |
| ASI09 | Human-Agent Trust Exploitation | fail | AI-09 |
| ASI10 | Rogue Agents | fail | AI-11 |

## 6. Supporting datasets

### 6.1 Secure scores (`SECURE_SCORE`)

| Score | Current | Max | % |
|---|---|---|---|
| Defender for Cloud secure score | 34 | 58 | 59 |
| Microsoft Secure Score | 412 | 731 | 56 |

Defender for Cloud controls: Enable MFA 4/10 (7 unhealthy), Remediate vulnerabilities 2/6 (214), Apply system updates 2/6 (27), Restrict unauthorized network access 1/4 (23), Manage access and permissions 2/4 (9), Enable encryption at rest 3/4 (3), Enable endpoint protection 1/2 (8), Enable auditing and logging 0/1 (38).

Microsoft Secure Score categories: Identity 41/82, Devices 210/371, Apps 88/168, Data 73/110.

### 6.2 Defender for Cloud built-in CIS standard (`DFC_CIS`)

Standard: CIS Microsoft Azure Foundations Benchmark v2.0.0. Summary: 41 passed, 23 failed, 6 skipped, 12 unsupported. Numbering follows the version Defender ships, so these rows are never merged with the v3.0 mapping above.

| Control | Title | State | Passed | Failed |
|---|---|---|---|---|
| 1.1.1 | Ensure Security Defaults is enabled on Azure Active Directory | Skipped | 0 | 0 |
| 1.1.2 | Ensure that multi-factor authentication is enabled for all privileged users | Failed | 16 | 7 |
| 1.1.3 | Ensure that multi-factor authentication is enabled for all non-privileged users | Failed | 1842 | 568 |
| 1.5 | Ensure guest users are reviewed on a regular basis | Failed | 126 | 61 |
| 2.1.1 | Ensure that Microsoft Defender for Servers is set to On | Failed | 4 | 1 |
| 2.1.4 | Ensure that Microsoft Defender for Storage is set to On | Failed | 2 | 3 |
| 3.1 | Ensure that Secure transfer required is set to Enabled | Failed | 33 | 4 |
| 3.7 | Ensure that Public access level is disabled for storage accounts with blob containers | Failed | 35 | 2 |
| 3.8 | Ensure default network access rule for storage accounts is set to deny | Failed | 26 | 11 |
| 3.9 | Ensure soft delete is enabled for Azure Blob storage | Passed | 37 | 0 |
| 4.1.1 | Ensure that Auditing is set to On for SQL servers | Failed | 6 | 3 |
| 5.1.1 | Ensure that a Diagnostic Setting exists for the Activity log | Failed | 3 | 2 |
| 6.1 | Ensure that RDP access from the Internet is evaluated and restricted | Failed | 31 | 6 |
| 6.2 | Ensure that SSH access from the Internet is evaluated and restricted | Failed | 33 | 4 |
| 7.6 | Ensure that Endpoint Protection for all Virtual Machines is installed | Failed | 134 | 8 |
| 8.5 | Ensure the key vault is recoverable (purge protection) | Failed | 5 | 3 |

### 6.3 CVEs (`CVES`)

| CVE | CVSS | Severity | Exploit | Exposed devices | Software | Vendor | Published |
|---|---|---|---|---|---|---|---|
| CVE-2024-38063 | 9.8 | critical | yes | 87 | Windows Server 2019/2022 (TCP/IP) | Microsoft | 2024-08-13 |
| CVE-2024-38077 | 9.8 | critical | yes | 41 | Remote Desktop Licensing Service | Microsoft | 2024-07-09 |
| CVE-2025-21298 | 9.8 | critical | yes | 63 | Windows OLE | Microsoft | 2025-01-14 |
| CVE-2024-43491 | 9.8 | critical | no | 12 | Windows 10 Servicing Stack | Microsoft | 2024-09-10 |
| CVE-2024-4577 | 9.8 | critical | yes | 23 | PHP 8.x (CGI on Windows) | PHP Group | 2024-06-09 |
| CVE-2024-7971 | 8.8 | high | yes | 152 | Microsoft Edge (Chromium V8) | Microsoft | 2024-08-21 |
| CVE-2023-4863 | 8.8 | high | yes | 38 | libwebp (Edge, Chrome, Teams) | Google | 2023-09-12 |
| CVE-2024-6387 | 8.1 | high | yes | 19 | OpenSSH 8.5-9.7 (Linux) | OpenBSD | 2024-07-01 |
| CVE-2024-21412 | 8.1 | high | yes | 71 | Windows SmartScreen | Microsoft | 2024-02-13 |
| CVE-2024-49138 | 7.8 | high | yes | 96 | Windows CLFS driver | Microsoft | 2024-12-10 |
| CVE-2024-38193 | 7.8 | high | yes | 58 | Windows AFD.sys | Microsoft | 2024-08-13 |
| CVE-2024-30051 | 7.8 | high | yes | 44 | Desktop Window Manager | Microsoft | 2024-05-14 |
| CVE-2024-38112 | 7.5 | high | yes | 29 | Windows MSHTML | Microsoft | 2024-07-09 |
| CVE-2024-30088 | 7.0 | medium | yes | 33 | Windows kernel | Microsoft | 2024-06-11 |
| CVE-2024-38213 | 6.5 | medium | yes | 117 | Windows Mark of the Web | Microsoft | 2024-08-13 |

CVE IDs are real; CVSS scores and exposure counts are illustrative.

### 6.4 AI datasets

`STATS.ai`: 9 AI accounts (6 Azure OpenAI, 3 AI Services), 3 hubs, 7 projects, 12 deployments; 5 accounts with public network access, 7 with key auth; 5 subscriptions, 2 with Defender for AI services; 38 Copilot Studio agents, 6 shared org-wide; 52 agent identities, 17 without owner; 53 prompt attacks in 30 days, 47 blocked; 1,284 sensitive prompts in 30 days; 41 generative AI apps discovered, 28 unsanctioned; 5 MCP servers.

`AI_ALERTS` (30 days, 81 total): Jailbreak blocked 47 (medium), Indirect prompt injection 12 (high), Sensitive data in response 9 (high), Jailbreak detected only 6 (high), Anomalous access from Tor 4 (medium), Credential theft attempt 2 (high), Wallet abuse 1 (low).

`AGENT_PLATFORMS` (high / medium / low risk): Copilot Studio 7 / 12 / 19, Foundry Agent Service 2 / 3 / 4, Custom (Entra Agent ID) 2 / 1 / 2. Total 52.

`AGENTS` (10 representative rows):

| Agent | Platform | Owner | Auth mode | Connectors | Sharing | Risk | Findings |
|---|---|---|---|---|---|---|---|
| Contoso Sales Copilot | Copilot Studio | Dana Whitfield | Manual (OAuth) | 7 | Everyone in org | high | AI-09, AI-10 |
| IT Helpdesk Bot | Copilot Studio | — | No authentication | 2 | Public website | high | AI-09, AI-11 |
| HR Benefits Assistant | Copilot Studio | Priya Natarajan | Microsoft Entra | 3 | Everyone in org | medium | AI-09 |
| Onboarding Q&A | Copilot Studio | Priya Natarajan | Microsoft Entra | 1 | Everyone in org | low | — |
| Expense Policy Helper | Copilot Studio | Finance Ops | Microsoft Entra | 2 | Everyone in org | low | — |
| Field Service Scheduler | Copilot Studio | Marcus Lee | Microsoft Entra | 4 | Specific users | low | AI-10 |
| Contract Review Agent | Foundry Agent Service | Legal AI team | Managed identity | 2 | Owner only | medium | AI-13 |
| Invoice Triage Agent | Foundry Agent Service | — | Client secret | 3 | Owner only | high | AI-11, AI-12, AI-13 |
| DevOps Release Agent | Custom (Entra Agent ID) | Platform Eng | Client secret | 5 | Specific users | high | AI-12 |
| Security Triage Agent | Custom (Entra Agent ID) | SecOps | Managed identity | 3 | Specific users | low | — |

## 7. Known issues and caveats

- **Shared control-ID namespace.** `CONTROL_NAMES` is one dictionary for all frameworks. CIS Azure v3.0 and CIS M365 v4 both use the key `2.1.1`, so the CIS Azure entry (Security Defaults) is overwritten by the CIS M365 entry (Safe Links) and the wrong title shows for CIS Azure 2.1.1 in the drawer, chips and by-control view. Rollup counts are unaffected because they are computed per framework. Fix by keying titles per framework (for example `cisAzure:2.1.1`) or by avoiding colliding IDs.
- **Hand-transcribed control numbers.** CIS, OWASP Agentic titles and ATLAS IDs were transcribed from the benchmark structure and abbreviated. Verify against the published sources before using the mapping beyond a demo.
- **Referenced, not total, controls.** Rollup percentages count only controls that at least one finding references.
- **Sample rows in the workbook that are not in `index.html`.** The workbook's sample tiles for legacy-auth breakdown, MFA sign-in split, role-change audit rows, risky users, missing updates, active alerts and Defender plan coverage were authored directly in `workbook.json` to fit their live query shapes. They are consistent with `STATS` and the findings but have no counterpart constant.

## 8. Regenerating the tables in this document

There is no build step. The tables above were produced by loading `index.html` in a browser and reading the constants from the page. Any of these works:

1. Open `index.html`, open DevTools, and run a snippet that iterates `FINDINGS` and `FRAMEWORKS` and prints Markdown rows with `console.log`.
2. Headless: load a small harness page that embeds `index.html` in an iframe (`--allow-file-access-from-files`), reads the constants via the iframe's window, writes Markdown into a `<pre>`, and dump the DOM with `msedge --headless=new --dump-dom`.

Also update the domain totals, rollup results and generation date at the top of this file.
