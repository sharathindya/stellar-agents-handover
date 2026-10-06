# Stellar Agents — AWS Handover for a Microsoft Foundry Azure-Management Agent

**Prepared:** 6 October 2026  
**Scope:** source code inspected locally. AWS runtime state has **not** been verified because the configured `startup` AWS session has expired.

## 1. Executive summary

Stellar Agents is a multi-tenant, AI customer-support product. A business registers, uploads documents, embeds a browser widget, and customers can chat with an AI support agent. The application is implemented for AWS, primarily using SAM/CloudFormation, Lambda, API Gateway, DynamoDB, S3, Amazon Bedrock, and SES.

This is **not an Azure-management agent** and there is no Azure or Microsoft Foundry deployment in this repository yet. A Foundry agent can use this document as context to help operate Azure resources, but it must not assume that it can manage the AWS resources below without separately configured AWS tools and credentials.

## 2. What has been built

| Area | Present in source | Notes |
|---|---|---|
| Chat API | Yes | `POST /chat`; retrieves tenant KB context, calls Bedrock Converse, and stores both turns. Includes a canned demo fallback when Bedrock access is unavailable. |
| Tenant and document API | Yes | `POST /ingest` supports `register`, `upload`, and `sync` actions. Uploads go to a tenant-specific S3 prefix. |
| Conversation history API | Yes | `GET /sessions/{sessionId}` returns tenant-filtered messages. |
| Human escalation API | Yes | `POST /escalate` emails a transcript through SES. |
| Embed widget | Yes | Plain JavaScript widget with tenant/API settings passed in the script tag. |
| Admin dashboard | Yes | Static dashboard files are included. |
| Marketing landing page | Yes | Static HTML/CSS/JS, including a waitlist form. Its API expectation needs completion; see Known gaps. |
| Infrastructure as code | Yes | AWS SAM template provisions the API, four functions, two DynamoDB tables, S3 buckets, IAM roles, and OpenSearch Serverless resources. |
| Automated tests / CI | Not found | No test suite or CI pipeline was present. |
| Azure / Foundry implementation | Not found | No `azure.yaml`, `.foundry/`, Bicep/Terraform, or Foundry agent source was present. |

## 3. AWS architecture

```text
Browser widget / dashboard / landing page
                 |
              API Gateway
     /chat  /ingest  /sessions  /escalate
        |       |         |          |
      Lambda  Lambda    Lambda     Lambda
        |       |         |          |
 Bedrock + DynamoDB   DynamoDB    SES
        |       |
 Bedrock KB  S3 tenant documents
```

The declared deployment region is `ap-southeast-2` and the deployment script targets the `startup` AWS profile. The stack name pattern is `stellar-agents-<dev|prod>`.

## 4. Source-of-truth locations

| Purpose | File |
|---|---|
| Architecture overview | `docs/architecture.md` |
| Infrastructure | `infra/template.yaml` |
| Deployment workflow | `deploy.sh` |
| Chat implementation | `backend/functions/chat/handler.py` |
| Tenant, upload, KB ingestion | `backend/functions/ingest/handler.py` |
| Session history | `backend/functions/sessions/handler.py` |
| Email escalation | `backend/functions/escalate/handler.py` |
| Browser widget | `frontend/widget/stellar-widget.js` |
| Landing page brief and target endpoint | `specs/landing-page.md` |
| Bedrock fallback brief | `specs/mock-agent.md` |

## 5. Deployed-resource contract

These are the resources declared in the SAM template. Actual creation and status must be checked after authenticating to AWS.

| Resource | Naming / purpose |
|---|---|
| HTTP API | Stage is `dev` or `prod`; routes are `/chat`, `/ingest`, `/sessions/{sessionId}`, `/escalate`. |
| Chat Lambda | `stellar-chat-<stage>`; Bedrock chat/retrieval plus DynamoDB session persistence. |
| Ingest Lambda | `stellar-ingest-<stage>`; tenant registration, document upload, and KB ingestion. |
| Sessions Lambda | `stellar-sessions-<stage>`; reads session history. |
| Escalate Lambda | `stellar-escalate-<stage>`; sends SES email to `support@stellaragents.in` by default. |
| Sessions table | `stellar-sessions-<stage>`; partition key `sessionId`, sort key `timestamp`, seven-day TTL. |
| Tenants table | `stellar-tenants-<stage>`; partition key `tenantId`. |
| Document bucket | `stellar-docs-<stage>-<account-id>`; private, encrypted, versioned. |
| Frontend bucket | `stellar-frontend-<stage>-<account-id>`; public S3 static-website hosting. |
| Bedrock KB role | `stellar-bedrock-kb-role-<stage>`. |
| OpenSearch Serverless collection | `stellar-vectors-<stage>`; provisioned in the template. |

## 6. Important operational facts

- The chat model configured in the template is `apac.anthropic.claude-haiku-4-5-20251001-v1:0`; the configured retrieval model is `apac.anthropic.claude-sonnet-4-6`.
- Chat has a deliberate demo fallback for Bedrock access/model errors, so a successful chat response does not prove Bedrock is working.
- Documents are limited to 5 MB and the allowed types are PDF, TXT, Markdown, HTML, and DOCX.
- Tenant data is scoped in S3 as `tenants/<tenant-id>/`.
- API CORS currently allows every origin. The frontend bucket is also public for S3 website hosting.
- The template enables Lambda X-Ray tracing and JSON logging, but no alarms, dashboards, or retention policy are defined.

## 7. Known gaps and inconsistencies — resolve before production

1. **KB setup is incomplete.** `create_knowledge_base()` exists but is never called during tenant registration. New tenants therefore receive empty `knowledgeBaseId` and `dataSourceId`; `sync` will fail until the creation flow is wired in.
2. **Storage design conflicts.** The ingest code creates a Bedrock-managed vector store, while the template still provisions OpenSearch Serverless and passes its ARN to the ingest function. The collection appears unused by the runtime code.
3. **Landing-page waitlist is not implemented server-side.** The landing page sends `action: "waitlist"`; the ingest handler supports only `register`, `upload`, and `sync`, so this request currently returns HTTP 400.
4. **CloudFront is documented but not provisioned.** The architecture document mentions CloudFront/OAC/HTTPS, while the template serves the frontend through a public HTTP S3 website endpoint.
5. **CORS and public frontend access need hardening.** Replace `*` with approved origins and serve via CloudFront/HTTPS before handling real customer traffic.
6. **Deployment state is unknown.** The spec records a previous dev endpoint and bucket, but they are not current-state evidence. The AWS credential session expired during handover preparation.
7. **No automated tests or CI/CD were found.** Add unit tests, API smoke tests, deployment validation, and least-privilege policy checks.

## 8. Handover checklist for the next operator

1. Re-authenticate the `startup` AWS profile, then inspect `stellar-agents-dev` and any production stack.
2. Record stack outputs, function health, recent CloudWatch errors, SES production-access/verified-identity status, and Bedrock model access.
3. Decide on one vector-store approach: Bedrock-managed storage or OpenSearch Serverless. Remove unused infrastructure and permissions.
4. Wire KB creation into tenant onboarding and test register → upload → sync → chat with a real document.
5. Implement or remove the landing-page waitlist action.
6. Put the frontend behind HTTPS/CloudFront, restrict CORS, add authentication/rate limiting, and review IAM permissions.
7. Add observability, tests, and CI/CD before treating the product as production-ready.

## 9. Microsoft Foundry agent operating boundary

Give the Foundry agent only the Azure permissions it needs, ideally through a managed identity and Azure RBAC. It should:

- inspect and report on Azure resource health, deployments, costs, and logs;
- prepare a remediation plan before destructive or high-impact changes;
- require explicit confirmation for deletions, role changes, public-network exposure, quota changes, and production deployments;
- keep AWS and Azure inventories separate; AWS actions require their own approved tool connection and credential scope;
- never store cloud credentials, connection strings, or customer documents in prompts, source control, or this handover.

Recommended first tools for the Azure agent are read-only Azure Resource Graph/ARM inventory, Azure Monitor/Application Insights queries, Cost Management reads, and deployment-status reads. Add write tools only after an approval workflow and audit trail are in place.

## 10. Verification completed for this handover

- Python handlers compiled successfully with `python3 -m compileall`.
- Cloud state and SAM validation could not be completed: the local AWS session is expired and the configured shared credentials are incomplete.

