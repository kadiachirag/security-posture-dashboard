# Security Posture Dashboard (Entra ID, Azure resources, Defender, AI & Agents)

A demo dashboard that shows security posture across four domains and maps every finding to Microsoft best practice, the CIS benchmarks and, for the AI domain, the OWASP AI threat taxonomies:

| Domain | What it covers | Frameworks mapped |
|---|---|---|
| Entra ID | MFA coverage, Conditional Access gaps, privileged roles, PIM, legacy auth, app consent, guest access, Identity Protection | MCSB, CIS Azure v3.0 section 2, CIS M365 v4 sections 1, 5, 6 |
| Azure resources | Defender for Cloud recommendations by resource type: storage, network, key vault, SQL, compute, logging, Defender plans | MCSB, CIS Azure v3.0 sections 3-9 |
| Defender | Defender Vulnerability Management CVEs, device health, endpoint protection, incidents, Secure Score, Defender for Office 365 | MCSB, CIS Azure v3.0 section 3, CIS M365 v4 sections 2, 3 |
| AI & Agents | Azure OpenAI / AI Foundry resource posture (network, key auth, logging, Prompt Shields), Defender for AI services plan and alerts, Purview DSPM for AI (sensitive prompts, Copilot oversharing), Copilot Studio agents, Entra Agent ID identities, MCP servers used by agents, shadow AI from Defender for Cloud Apps | MCSB, CIS M365 v4 section 3, OWASP Top 10 for LLM Applications 2025, OWASP Top 10 for Agentic Applications 2025 (coverage view), MITRE ATLAS technique references in the drawer |

CIS Azure Foundations v3.0 has no AI-services section, so AI findings carry no CIS Azure mapping.

**All data is sample data.** Nothing here connects to a tenant. The tenant name, counts, CVE exposure numbers and scores are illustrative.

Two deliverables share the same dataset:

| File | Purpose |
|---|---|
| `index.html` | Single-file dashboard. Open it in a browser; no server, build step or internet access needed. |
| `workbook.json` | Azure Monitor Workbook gallery template that mirrors the dashboard tiles. Every tile has a sample version (embedded JSON) and a live version (Azure Resource Graph or Log Analytics query) switched by a `DataMode` parameter. |
| `queries.kql` | Readable copy of every live query used in the workbook. |

## Documentation

| Document | Contents |
|---|---|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | How the two deliverables are built, render cycle, benchmarks vs taxonomies, workbook structure, live data sources, design decisions. Includes Mermaid diagrams. |
| [docs/architecture.svg](docs/architecture.svg) | One-page architecture diagram. Open it in a browser. |
| [docs/DATA-MODEL.md](docs/DATA-MODEL.md) | Finding schema, derivation rules, the full finding-to-control mapping for all 62 findings, per-framework control tables, supporting datasets, known issues. |

## Open the dashboard

```powershell
Start-Process .\FableTechDay\index.html
```

Refresh the browser after editing. Deep links work: `index.html#entra`, `#azure`, `#defender`, `#ai`, `#compliance`.

### Tour

- **Header**: the overall posture score is an equal-weight average of three numbers: the findings score, the Defender for Cloud secure score percentage and the Microsoft Secure Score percentage. The findings score is `100 - weighted open / weighted total`, with severity weights critical 4, high 3, medium 2, low 1, and `warn` findings counting half. The three components are printed under the score.
- **Filter row** scopes every tab: domain, severity chips, status (`Open` = fail + warn), framework + control ID prefix (for example `IM-6`, `2.2` or `LLM01`), and free-text search. On the Entra, Azure, Defender and AI & Agents tabs the domain filter is replaced by the tab's own domain.
- **Overview**: domain cards (click to jump to that tab), both secure scores, framework rollup, open findings by severity and domain, top open findings.
- **Domain tabs**: KPI tiles, open findings by category, coverage meters, and a sortable findings table. The Defender tab adds the CVE table with "Exploit available" and "CVSS >= 9" toggles.
- **AI & Agents tab**: KPI tiles (public network access, key auth, Defender for AI coverage, agents shared org-wide, ownerless agent identities, jailbreak attempts, sensitive prompts, unsanctioned gen-AI apps), open findings by category, agent inventory by platform and risk, Defender for AI services alerts by type, an OWASP coverage strip and a sortable agent inventory table whose finding chips open the related detail. Its findings table swaps the always-empty CIS Azure column for an "AI frameworks" column with OWASP chips.
- **Compliance mapping**: framework cards, plus a fourth card showing what Defender for Cloud's built-in CIS regulatory compliance standard reports, then the **AI threat coverage (OWASP)** card. Toggle "By finding" (cross-reference table; click a control chip to filter everything by that control) or "By control" (one row per control with state, worst severity and mapped findings; the OWASP lists are selectable here too).
- **Details drawer**: click any finding row for scope, description, remediation steps and full control titles. AI findings also list related MITRE ATLAS techniques. Esc closes it.

### How the framework rollup is computed

For each framework, the dashboard collects every control ID referenced by at least one finding. A control **fails** if any mapped finding fails, is **partial** if any mapped finding is `warn` and none fail, and **passes** otherwise. The percentage is passing controls over referenced controls. This is not full benchmark coverage; controls no finding references are not counted.

The Defender for Cloud CIS card is separate. It shows the standard as Defender reports it (currently CIS Azure Foundations v2.0.0 numbering), which differs from the v3.0 numbers used in the mapping fields, so the two are never merged.

### How the OWASP coverage is computed

The OWASP Top 10 for LLM Applications and the OWASP Top 10 for Agentic Applications are threat taxonomies, not control benchmarks, so they are never shown as a pass percentage and are excluded from the framework rollup. For each of the 10 categories in a list the dashboard shows one cell: **fail**, **warn** or **pass** using the same rule as the rollup when at least one finding maps to it, and **not assessed** when none does. The headline reads "x of 10 risk categories assessed, y with open fails". A passing cell means the mapped controls pass, not that the risk is absent. With the sample data LLM04, LLM05, LLM09, ASI05, ASI06 and ASI08 are not assessed, which is a useful "what would you add next" talking point.

## Import the workbook

1. Azure portal > **Monitor** > **Workbooks** > **+ New**.
2. Open the **Advanced Editor** (`</>` in the toolbar), choose **Gallery Template**, replace the content with `workbook.json`, click **Apply**.
3. **Save**: choose a title, subscription, resource group and region. Saving requires Workbook Contributor (or Contributor) on the resource group.
4. Leave `DataMode = Sample data` to show the embedded demo data with no data access at all.

### Go live

1. Set `Subscription` (one or more) and `Workspace` (the Log Analytics workspace that receives Entra and Defender logs), then switch `DataMode` to **Live tenant**.
2. **Defender for Cloud tiles** (assessments, secure score, plan coverage, alerts) use Azure Resource Graph and light up immediately with Reader + Security Reader.
3. **Regulatory compliance tiles** need the CIS standard assigned: Defender for Cloud > Environment settings > subscription > Security policies > enable the CIS Microsoft Azure Foundations Benchmark standard. Data appears after the next assessment cycle.
4. **Entra tiles** need diagnostic settings on the Entra tenant sending `SignInLogs`, `AuditLogs` and `RiskyUsers` to the workspace.
5. **Vulnerability tiles** (`DeviceTvm*` tables) need the Microsoft Defender XDR data connector in Microsoft Sentinel on that workspace. Without Sentinel, run the same queries in Defender XDR > Advanced hunting (see `queries.kql`; remove the `{TimeRange}` token).

#### AI & Agents tiles

6. **AI resource inventory, deployments, plan coverage, alerts and findings** use Azure Resource Graph and light up with Reader + Security Reader. Enable **Defender CSPM** (it includes AI security posture management) and the **Defender for AI services** plan on every subscription that hosts Azure OpenAI or AI Services accounts; turn on *user prompt evidence* so alerts carry the suspicious prompt segments.
7. **Azure OpenAI usage and content-filter blocks** need a diagnostic setting on each account sending the `RequestResponse`, `Audit`, `Trace` and `AllMetrics` categories to the workspace (`AzureDiagnostics`).
8. **Copilot interactions** (`CloudAppEvents`) and **shadow AI** (`McasShadowItReporting`) need the Defender XDR and Defender for Cloud Apps connectors in Sentinel. Without Sentinel, run the `CloudAppEvents` query in Advanced hunting and read app discovery in the Defender portal.
9. **Agent identity sign-ins** need the `ServicePrincipalSignInLogs` (and `ManagedIdentitySignInLogs`) categories in the Entra diagnostic setting.
10. **Copilot Studio activity** (`PowerPlatformAdminActivity`) needs Power Platform admin activity export to Log Analytics; create a Power Platform data policy for the default environment and require authentication for published agents while you are there.
11. **Purview DSPM for AI**: enable the one-click policies (audit Copilot interactions, DLP for AI apps, browser extension). Its reports stay portal-only; the workbook shows a note in Live mode.
12. **Agent inventory** (Copilot Studio agents, Entra Agent ID owners and permissions, MCP servers) is admin-center and Microsoft Graph data with no live tile. Export it on a schedule into a Log Analytics custom table if you want it in the workbook.

### Permissions

| Data | Role |
|---|---|
| Resource Graph: assessments, secure score, pricings, alerts, regulatory compliance | Reader on the subscription plus Security Reader |
| Log Analytics queries | Log Analytics Reader on the workspace |
| Entra sign-in and audit logs (diagnostic setting) | Global Administrator or Security Administrator to create the setting; Reports Reader to view |
| Defender XDR Advanced hunting | Security Reader in Defender XDR |
| Resource Graph: Cognitive Services accounts and deployments | Reader (or Cognitive Services Reader) on the AI subscriptions |
| Purview DSPM for AI reports (portal only) | Compliance Administrator or Information Protection Reader in Purview |
| Copilot Studio inventory and data policies (admin center only) | Power Platform Administrator |
| Entra Agent ID inventory (portal / Graph only) | Global Reader, or Application Administrator to fix owners and credentials |
| Saving the workbook | Workbook Contributor on the resource group |

### Limits

Azure Monitor Workbooks cannot query Microsoft Graph. MFA registration details, Conditional Access policy definitions, PIM assignments, consent and guest settings, and the Microsoft 365 Secure Score are Graph-only, so those tiles stay in sample mode. Options: use the built-in Entra workbooks (Conditional Access gap analyzer, Sign-ins using legacy authentication, Multifactor authentication gaps) and Identity Protection reports, or export the Graph data into a Log Analytics custom table and point the live tiles at it.

For the AI & Agents tab the same applies to Purview DSPM for AI reports, Copilot Studio agent inventory (sharing, authentication mode, connectors), Entra Agent ID inventory (owners, permissions, credentials), Prompt Shields configuration per deployment, model retirement dates and the MCP server catalog: all sample-only, with a text note in Live mode. Every AI live query is marked in `queries.kql` with what to verify against a real tenant, because table, property and plan names in this area change frequently.

## Editing the sample data

All data lives at the top of the `<script>` in `index.html`:

| Constant | Contents |
|---|---|
| `FINDINGS` | One object per finding. Fields: `id`, `domain` (`entra`/`azure`/`defender`/`ai`), `category`, `title`, `severity` (`critical`/`high`/`medium`/`low`), `status` (`fail`/`warn`/`pass`), `affected`, `affectedUnit`, `scope`, `source`, `detected`, `description`, `remediation`, and the mapping arrays `mcsb`, `cisAzure`, `cisM365`, `owaspLlm`, `owaspAgentic`, `atlas`. Mapping arrays may be omitted; a normalization loop after the array fills missing ones with `[]`. |
| `CVES` | Rows for the Defender Vulnerability Management table. |
| `SECURE_SCORE` | Defender for Cloud score with its security controls, and Microsoft Secure Score with its categories. |
| `DFC_CIS` | Defender for Cloud's built-in CIS standard summary and control rows. |
| `STATS` | Headline numbers for KPI tiles that are not derivable from findings (`STATS.ai` holds the AI inventory counts). |
| `AI_ALERTS` | Defender for AI services alert types and counts for the last 30 days. |
| `AGENT_PLATFORMS` | Agent counts by platform and risk level; totals must equal `STATS.ai.agentIdentities`. |
| `AGENTS` | Sample rows for the agent inventory table, each with `findingIds` linking to findings. |
| `CONTROL_NAMES` | Control ID to title, used in the drawer, chips and the by-control view. Includes OWASP LLM01-LLM10, ASI01-ASI10 and the MITRE ATLAS technique IDs. |
| `FRAMEWORKS` | Framework definitions. Entries with `taxonomy: true` (the OWASP lists) carry their 10 category IDs and are rendered as coverage, not rollup. |

To add a finding, append an object to `FINDINGS` and add any new control ID to `CONTROL_NAMES`. Everything else derives.

**Keep `workbook.json` in sync.** Its sample tiles embed a JSON-encoded copy of the same rows. After changing `FINDINGS`, regenerate the affected sample tiles. Each sample tile's `query` is a JSON string of the form `{"version":"1.0.0","content":"<JSON-encoded row array>","transformers":null}`. To check the file parses correctly:

```powershell
$wb = Get-Content .\FableTechDay\workbook.json -Raw | ConvertFrom-Json
$wb.version   # Notebook/1.0
function Get-Items($items) { foreach ($i in $items) { $i; if ($i.type -eq 12) { Get-Items $i.content.items } } }
Get-Items $wb.items | Where-Object { $_.content.queryType -eq 8 } | ForEach-Object {
  $rows = ($_.content.query | ConvertFrom-Json).content | ConvertFrom-Json
  "{0}: {1} rows" -f $_.name, @($rows).Count
}
```

## Framework references

- Microsoft cloud security benchmark (MCSB) v1: control IDs such as IM-6, PA-2, NS-1, LT-3, PV-6, ES-1; for AI also IM-8, DP-1, DP-2, AM-2.
- CIS Microsoft Azure Foundations Benchmark v3.0.0 (no AI-services section).
- CIS Microsoft 365 Foundations Benchmark v4.0.0.
- OWASP Top 10 for LLM Applications 2025 (LLM01-LLM10) and OWASP Top 10 for Agentic Applications 2025 (ASI01-ASI10), genai.owasp.org.
- MITRE ATLAS (atlas.mitre.org) technique IDs, drawer only.
- Microsoft Learn: Defender for Cloud AI security posture management, Defender for AI services alerts reference, Microsoft Purview DSPM for AI, Copilot Studio security and governance, Microsoft Entra Agent ID.

Control numbers in the sample data were transcribed by hand from the benchmark structure and abbreviated in `CONTROL_NAMES`; the OWASP Agentic titles and ATLAS IDs were transcribed the same way. Verify them against the published sources before using this mapping for anything beyond a demo.
