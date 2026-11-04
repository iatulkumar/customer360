## Backend Implementation

Python FastAPI service (single deployable) that hosts the REST API and the LangGraph agent workflow in the same process. MongoDB Atlas is the only datastore. All endpoints are prefixed `/api/v1`.

### Project layout
- `app/main.py` - FastAPI app, CORS, router registration
- `app/demo/` - seeded persona endpoint and the `get_actor` dependency
- `app/routers/` - `customer`, `admin`
- `app/services/` - onboarding state machine, notifications, pricing, audit
- `app/agents/` - LangGraph workflow, Risk / Compliance / Document agents, tools, prompts, guardrail middleware
- `app/db/` - Mongo client, collection accessors, index setup
- `app/schemas/` - Pydantic models for requests, responses and agent structured outputs
- `scripts/` - seed data, regulation/procedure ingestion, vector index creation
- `tests/`

### Users and access (demo)
- No endpoint authentication, tokens or passwords. The API is for demo use only and must not be deployed with real data.
- `GET /demo/users` returns the seeded personas (customers and admins). The login portal lets the user pick one and the frontend redirects to the customer or admin site by the persona's role.
- Customer endpoints take the acting customer as a `customer_id` query parameter, and admin endpoints take an optional `admin_id` for audit entries. Both are trusted as given.
- Routes stay split under `/customer` and `/admin` so the structure matches what real role-based auth would protect later. A single FastAPI dependency (`get_actor`) reads these ids, so auth can be added in one place.
- Audit entries still record the actor id and role.

### Onboarding state machine
States: `APPLIED`, `VERIFYING`, `NEEDS_INFO`, `RISK_ASSESSMENT`, `COMPLIANCE_CHECK`, `DOCUMENT_GENERATION`, `ADMIN_REVIEW`, `AWAITING_SIGNATURE`, `ONBOARDED`, `PAUSED`, `CANCELLED`.

- Flow: `APPLIED` -> `VERIFYING` -> `RISK_ASSESSMENT` -> `COMPLIANCE_CHECK` -> `DOCUMENT_GENERATION` -> `ADMIN_REVIEW` -> `AWAITING_SIGNATURE` -> `ONBOARDED`.
- Human review gates (workflow pauses via LangGraph `interrupt`) after Risk + Compliance (also covers the recommended pricing) and after Document generation, before anything is shown to the customer. Admin decisions: `continue`, `pause`, `cancel`, or `request_info` (moves to `NEEDS_INFO`).
- `PAUSED` and `NEEDS_INFO` store the previous state so the run resumes where it stopped.
- Cancel is allowed from any non-terminal state, by the customer or an admin.
- Every transition writes an `audit_log` entry (actor, from, to, reason, timestamp) and a customer notification.
- Verification (KYC/KYB) is a mocked tool that checks the submitted data against seeded records.

### Customer endpoints
All customer endpoints take `customer_id` as a query parameter.

| Method | Path | Purpose |
|---|---|---|
| POST | `/customer/onboardings` | Start onboarding with the application data (business details, owners, products requested). Starts the workflow and returns the onboarding id and required documents |
| GET | `/customer/onboardings` | List the customer's onboardings |
| GET | `/customer/onboardings/{id}` | Progress: current step, steps completed / remaining, estimated completion, contact details for the team on the current step |
| POST | `/customer/onboardings/{id}/documents` | Upload a requested document or supply missing information (resumes from `NEEDS_INFO`) |
| GET | `/customer/onboardings/{id}/documents` | List generated and uploaded documents, with download |
| POST | `/customer/onboardings/{id}/sign` | Mocked e-sign: records signer, timestamp, and moves to `ONBOARDED` |
| POST | `/customer/onboardings/{id}/cancel` | Cancel the onboarding and create a follow-up task for the team |
| GET | `/customer/onboardings/{id}/summary` | Completed onboarding details: pricing, benefits, access |
| GET | `/customer/notifications` | In-app notifications (unread first) |
| POST | `/customer/notifications/{id}/read` | Mark as read |
| GET | `/customer/onboardings/{id}/events` | (Stretch) Server-sent events for live status |

### Admin endpoints
Admin endpoints take an optional `admin_id` query parameter for audit entries.

| Method | Path | Purpose |
|---|---|---|
| GET | `/admin/onboardings` | List and filter by status, risk level, date, sector |
| GET | `/admin/onboardings/{id}` | Full detail: application, state history, agent outputs, documents, audit trail |
| GET | `/admin/reviews` | Queue of onboardings waiting at a review gate |
| POST | `/admin/onboardings/{id}/review` | Submit decision: `continue`, `pause`, `cancel`, or `request_info`, with comments and optional edits. Resumes the paused workflow |
| GET | `/admin/onboardings/{id}/risk` | Risk assessment: overall score, per-category scores for the risk map, agent explanations with citations |
| GET | `/admin/onboardings/{id}/risk/prediction` | (Stretch) Predicted risk trend (up / down / stable) with rationale |
| GET | `/admin/risk-appetite` | Bank's risk appetite for a sector and customer type (query params) |
| GET | `/admin/onboardings/{id}/pricing` | Agent-recommended pricing, within the risk appetite |
| PUT | `/admin/onboardings/{id}/pricing` | Adjust pricing. Records original vs adjusted and the reason |
| POST | `/admin/onboardings/{id}/contact` | Send a message or information request to the customer (creates a notification, and an email if enabled) |
| POST | `/admin/onboardings/{id}/sign` | Mocked e-sign for the bank side, if needed |
| GET | `/admin/guardrails` | Active guardrails and version history |
| GET | `/admin/guardrail-proposals` | Proposed guardrail changes generated from admin overrides |
| POST | `/admin/guardrail-proposals/{id}/decision` | Approve or reject. Approval creates a new guardrail version |
| GET | `/admin/metrics` | Average time to onboard, count by state, agent override rate |

### Agentic orchestration
The agents run in-process. They are invoked by the services above, not exposed as free-form chat endpoints.

- **Graph:** one LangGraph workflow per onboarding, `thread_id = onboarding_id`. State and checkpoints are persisted in MongoDB, so a run can resume after a pause, a review wait, a restart, or days later.
- **Nodes:** Verify (tool), Risk agent, Compliance agent, pricing step, Document agent, review gates (`interrupt`), Guardrail-learning step.
- **Routing:** deterministic edges for the fixed flow, with agent-decided branching only where needed (for example, Compliance finds missing information and routes to `NEEDS_INFO`).
- **Agents:** created with LangChain `create_agent`, with tools and a system prompt each:
  - Risk: retrieves procedures and past onboardings, reads the risk appetite, returns a structured score, category breakdown, explanation, and citations.
  - Compliance: retrieves regulations (FINTRAC, PCMLTFA, PIPEDA, OSFI) with hybrid search, returns pass / fail / needs-info per requirement with cited sources.
  - Document: generates the onboarding documents (PDF or markdown) from the profile, risk result, pricing and templates.
  - Future risk prediction (stretch): trend estimate from the profile and similar past onboardings.
- **Outputs:** every agent returns a Pydantic-validated structured result with `confidence` and `sources`. Invalid output triggers a retry, then escalates to an admin.
- **Agents recommend, humans decide:** no agent output becomes final without the review gate.
- **Endpoints for the workflow:**

| Method | Path | Purpose |
|---|---|---|
| GET | `/admin/agents/runs/{onboarding_id}` | Run status, current node, last checkpoint |
| POST | `/admin/agents/runs/{onboarding_id}/retry` | Retry a failed node |
| GET | `/admin/agents/runs/{onboarding_id}/trace` | Agent steps, tool calls and outputs for the admin detail view |

### Memory and adaptive guardrails
- **Short-term memory:** MongoDB-backed LangGraph checkpointer, keyed by onboarding.
- **Long-term memory:** customer facts, past decisions, and admin overrides are stored per customer and per sector, and retrieved by agents on later runs and later sessions.
- **Guardrails:** stored as versioned documents (risk thresholds, required documents per product and sector, escalation rules). Agents load the active version at the start of each run through middleware.
- **Learning loop:** when an admin changes an agent recommendation (risk level, pricing, compliance result), the override is stored with its reason. The Guardrail-learning step analyses recent overrides and creates a `guardrail_proposal`. An admin must approve it before it becomes the new active version. All versions are kept.

### Data model (MongoDB collections)
| Collection | Contents |
|---|---|
| `users` | id, name, role, customer_id (seeded personas, no credentials) |
| `customers` | business profile, owners, sector, nested application data |
| `onboardings` | customer_id, state, state history, current step, ETA, pricing (recommended, adjusted), risk summary |
| `agent_checkpoints` / `agent_memory` | LangGraph checkpointer data, long-term memory entries |
| `risk_assessments` | scores, categories, explanation, sources, prediction |
| `documents` | metadata, type, status, storage location or content |
| `document_chunks` | chunked uploaded document text with embeddings (vector) |
| `regulations` | regulation text chunks, jurisdiction, source (vector) |
| `procedures` | bank risk and onboarding procedures, risk appetite by sector (vector) |
| `guardrails` | key, version, rule, status, created_by |
| `guardrail_proposals` | proposed change, supporting overrides, status |
| `overrides` | admin changes to agent outputs, with reasons |
| `notifications` | recipient, type, message, read flag, channel |
| `audit_log` | append-only record of actions by users and agents |

**Indexes:** Atlas Vector Search indexes on `regulations`, `procedures`, `document_chunks`, and optionally past `onboardings`. Use metadata pre-filters (jurisdiction, product, sector). Add full-text search indexes on `regulations` and `procedures` for hybrid search. Regular indexes on `onboardings.customer_id` and `onboardings.state`. Atlas free tier limits the number of search indexes, so keep it to the ones above.

### Estimated completion and contacts
- ETA = sum of remaining steps' average durations (configured per step, updated from completed onboardings) plus any time spent waiting at review gates.
- Contacts per step come from a static configuration (team name, email, phone) returned with the progress endpoint.

### Notifications
- State changes, review outcomes, admin messages and missing-information requests create `notifications` records.
- In-app is required. Email is a stretch and uses a sandbox SMTP provider.
- Reminders (stretch): a scheduled job finds customers with actions outstanding for longer than a configured time.

### Errors and reliability
- Consistent error response format, with a request id.
- LLM calls have a timeout and a limited number of retries. A failed node leaves the run in a recoverable failed state and notifies admins.
- Uploaded document text is treated as untrusted data: it is delimited in prompts and never allowed to change instructions or tool calls.

### Configuration
Environment variables: Mongo connection string, Google Cloud project and credentials, Gemini model names, LangSmith keys, SMTP settings, feature flags for stretch features. CORS allows the frontend origin only. No secrets in the repository.

### Testing
- Unit tests for the state machine.
- Integration tests for each endpoint using seeded data.
- Agent evaluation cases for the seeded demo scenarios, asserting expected risk level and compliance outcome.
- One end-to-end test of the MVP journey.