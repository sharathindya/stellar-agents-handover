# Stellar Agents — Azure and Microsoft Foundry Handover

**Prepared:** 6 October 2026  
**Scope:** target Azure operating model and handover baseline for a Microsoft Foundry agent. No Azure resources, Foundry project, agent source, or Azure infrastructure were found in this workspace.

## 1. Current position

The product currently exists as an AWS implementation named **Stellar Agents**. It is a multi-tenant AI customer-support platform with a browser widget, customer-document ingestion, RAG-style retrieval, chat history, and human escalation.

There is **no Azure deployment to take over yet**. In particular, the workspace contains no `azure.yaml`, Foundry `.foundry/` metadata, Bicep, Terraform, Azure Functions, Azure AI Search, Application Insights configuration, or Foundry agent project. Treat this document as the Azure handover baseline, not evidence of deployed Azure resources.

For the corresponding AWS inventory and known product gaps, see `docs/aws-handover-for-foundry.md`.

## 2. Intended role of the Foundry agent

The Foundry agent is intended to manage and report on Azure—not to run the customer-support workload itself unless that scope is explicitly added later.

Its initial responsibilities should be:

- inventory Azure resources and identify configuration drift;
- report health, deployment status, errors, costs, quotas, and security findings;
- investigate incidents from Azure Monitor/Application Insights evidence;
- draft remediation plans and deployment changes;
- execute only pre-approved, narrowly scoped operations with an audit trail.

It should not independently delete resources, change access roles, expose public network paths, rotate secrets, modify budgets, or deploy production changes.

## 3. Recommended Azure target architecture

For an Azure version of Stellar Agents, keep the application workload and the Azure-management agent separate.

```text
Management plane
Microsoft Foundry agent
  └─ Managed identity + Azure RBAC
       ├─ Azure Resource Graph / ARM inventory
       ├─ Azure Monitor + Application Insights
       ├─ Cost Management
       └─ Deployment and policy status

Application plane (future Stellar Agents migration)
Static web hosting + CDN/WAF
  └─ API service / Functions
       ├─ Azure AI Foundry model endpoint
       ├─ Azure AI Search (tenant-filtered retrieval)
       ├─ Blob Storage (tenant documents)
       ├─ Cosmos DB or Table Storage (tenants and sessions)
       ├─ Communication Services or Logic Apps (human handoff)
       └─ Key Vault, App Insights, managed identities
```

This separation prevents the management agent from becoming a customer-facing data path and gives it a smaller, auditable permission footprint.

## 4. Azure services and recommended permissions

| Capability | Azure service / interface | Initial access |
|---|---|---|
| Resource inventory | Azure Resource Graph, Azure Resource Manager | Read-only at subscription or scoped resource groups |
| Health and diagnostics | Azure Monitor, Log Analytics, Application Insights | Read/query only |
| Cost reporting | Cost Management | Read-only |
| Governance posture | Azure Policy, Defender for Cloud | Read-only |
| Deployment status | ARM deployment history / deployment center | Read-only |
| Secret references | Key Vault metadata only | List/read metadata; no secret-value access by default |
| Remediation | Azure Resource Manager | No write access until a human-approved workflow exists |

Use a managed identity for the Foundry agent. Do not give it Owner, User Access Administrator, or broad subscription-wide write permissions. Assign roles at the smallest suitable scope, preferably dedicated resource groups.

## 5. Environment and resource layout

Create distinct environments and resource groups; do not mix operational data.

| Environment | Purpose | Suggested resource-group pattern |
|---|---|---|
| `dev` | Agent and application development | `rg-stellar-dev-<region>` |
| `test` | Integration and security testing | `rg-stellar-test-<region>` |
| `prod` | Customer-facing workload | `rg-stellar-prod-<region>` |
| `platform` | Central monitoring, policy, and shared agent components | `rg-stellar-platform-<region>` |

Use tags such as `application=stellar-agents`, `environment`, `owner`, `costCenter`, `dataClassification`, and `managedBy`. The agent should use these tags to scope reports and avoid ambiguous changes.

## 6. Minimum security baseline

- Use Microsoft Entra ID and managed identities; avoid client secrets where possible.
- Put secrets and connection strings in Key Vault; use secret references rather than copying values into agent configuration or prompts.
- Enable diagnostic settings for Foundry, AI Search, Storage, API hosting, Key Vault, and network resources; centralize logs in Log Analytics.
- Use private endpoints and private DNS for production data services where network requirements allow.
- Restrict public ingress through a gateway/WAF, enforce HTTPS, and define a small CORS allow-list.
- Apply tenant isolation to documents, search filters, conversations, and authorization checks; never rely only on a client-provided tenant ID.
- Configure budgets, cost alerts, resource locks where appropriate, backup/retention policies, and Azure Policy assignments.

## 7. Foundry agent guardrails

The Foundry agent’s system instructions and tools should enforce these rules:

1. Classify operations as **read**, **plan**, or **change**.
2. Execute read operations automatically only within its granted scope.
3. For every change, present the exact target, expected impact, rollback method, and required approval.
4. Require explicit approval for production changes, deletes, role assignments, network exposure, budget changes, quota requests, and secret access.
5. Keep a concise action log with timestamp, principal, resource ID, requested action, approval reference, outcome, and rollback status.
6. Redact secrets, connection strings, access tokens, customer data, and personally identifiable information from responses and logs.
7. Never claim a resource was changed or verified unless the tool result confirms it.

## 8. Build sequence

1. Create the Azure subscription/resource-group layout and apply tags, budgets, and policy.
2. Create a Microsoft Foundry project in the selected region and deploy a supported model after confirming capacity and quota.
3. Create the management agent with a managed identity and read-only resource, monitoring, cost, and policy tool connections.
4. Add Application Insights and an evaluation suite for accuracy, safe refusal, approval handling, secret redaction, and correct resource scoping.
5. Test against `dev` with read-only inventory and incident-summary scenarios.
6. Add a human approval mechanism for any write operation; start with one low-risk remediation workflow.
7. Only then consider migrating the Stellar Agents customer-support application from AWS to Azure, as a separate project.

## 9. Information still needed before deployment

- Azure subscription and tenant IDs, target region, and data-residency requirements.
- Named owners for platform, security, billing, and production approvals.
- The Foundry project’s desired network model: public access, managed VNet, or bring-your-own VNet.
- Which Azure resource groups the management agent may read and which, if any, it may later change.
- The required audit-log destination and retention period.
- Model choice, expected volume, budget, and quota/capacity constraints.
- Whether this agent is management-only or will eventually also power the customer-support application.

## 10. Handover status

| Item | Status |
|---|---|
| Azure infrastructure | Not present in workspace |
| Foundry project / deployed model | Not verified / not configured locally |
| Foundry agent source and metadata | Not present |
| Azure Developer CLI | Not available in the local environment during handover preparation |
| Azure credentials / subscription access | Not supplied or verified |
| AWS application reference | Present; see AWS handover |

## 11. First-run acceptance criteria

The Azure handover is complete only after the following are evidenced:

- A Foundry project and model deployment exist in the approved region.
- The agent authenticates with a managed identity, not embedded credentials.
- The agent can list only in-scope resources and summarize Azure Monitor health and costs.
- The agent refuses or requests confirmation for out-of-scope and write operations.
- Logs and traces capture actions without exposing secrets or customer data.
- An evaluation suite demonstrates correct scoping, approval behavior, and safe error handling.

