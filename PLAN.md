# Customer 360

Detailed plans: [FRONTEND-PLAN.md](FRONTEND-PLAN.md) and [BACKEND-PLAN.md](BACKEND-PLAN.md).

## Hackathon Requirements

**Event:** MongoDB x LangChain x Google Cloud Hackathon, Toronto, Saturday November 14 (single day). Teams of up to 4.

**Goal:** Build an agentic system that holds context across sessions, adapts its guardrails, and reliably turns information into action. It should be durable infrastructure, not a disconnected demo.

### Mandatory technology
| # | Requirement | How Customer 360 meets it |
|---|-------------|---------------------------|
| 1 | MongoDB Atlas as the data and context layer | Customer profiles, onboardings, documents, regulations, procedures, guardrails, and audit log are stored in Atlas |
| 2 | Atlas Vector Search | Retrieval of regulations, bank procedures, past onboardings, and document content |
| 3 | Agentic Memory | Short-term run state (checkpointer) and long-term memory (store) for customer facts, past decisions, and admin overrides |
| 4 | LangChain for agent experiences and workflows | LangChain and LangGraph for the onboarding workflow and specialist agents |
| 5 | Google Cloud as the cloud foundation | Gemini on Vertex AI, hosting on Cloud Run |

### Mandatory system behaviours
| # | Requirement | How Customer 360 meets it |
|---|-------------|---------------------------|
| 6 | Must be an agentic workflow, not just AI-assisted | A LangGraph workflow coordinates agents (Risk, Compliance, Document), which call tools and take actions |
| 7 | Holds context across sessions | A customer or admin can leave and return days later, and the workflow resumes from persisted state and memory |
| 8 | Adapts its guardrails | Guardrails (risk thresholds, required documents, escalation rules) are stored in MongoDB and updated, with human approval, from admin overrides. Changes are versioned and audited |
| 9 | Reliably turns information into action | Agents use structured outputs and tools to create records, generate documents, update state, and send notifications, with retries and a human review gate |

### Judging considerations
- Durable and impactful: real persistence, audit trail, and a path beyond the demo.
- Solves a real problem: customer onboarding is slow, and the demo measures days versus minutes.
- Working demo within the single day, with clear scope and a defined MVP.

## Project Idea
An agentic web-based application to aid in the process of customer onboarding. This can be applied to any business, but this demo is focused on banking. It is a single workflow that takes a business through the application process, risk and compliance review, document execution, and final onboarding. The name of the application is Customer 360.

The project is built with MongoDB Atlas, LangChain and Google Cloud.

The application contains several AI agents, each with a specific task:
- Risk assessment
- Document generation
- Legal and local legislative compliance
- Future risk prediction (stretch)

Agentic context is stored in MongoDB Atlas and accessed through vector search. Relevant context: customer profiles, local and federal rules and laws, company risk and onboarding procedures, and documents and metadata.

The customer journey:
Apply -> Verify -> Assess Risk -> Check Compliance -> Generate Documents -> Admin Review -> Sign -> Onboard

The customer can track progress at any point in the journey.

## MVP
Must be complete for the demo:
- Journey: Apply -> Verify (mocked) -> Risk -> Compliance -> Generate Documents -> Admin Review -> Sign (mock e-sign) -> Onboard.
- Customer: start onboarding, track progress, provide missing information and documents, mock e-sign, view completed details, in-app notifications.
- Admin: onboarding list and detail with agent trace, review queue and decision, read-only risk map with explanations and pricing, guardrails page (proposals and approval).
- Agentic memory: state persisted across sessions, and long-term memory of customer facts and admin overrides.

Stretch, if time allows:
- Future risk prediction, and risk map interactivity
- Pricing adjustments by the admin
- Customer cancel
- Email notifications and reminders
- Customer contact from the admin site
- Metrics dashboard
- Bank-side e-sign (can be mocked)
- Field-level encryption (can be mocked)

## Architecture

Customer 360 is a React frontend and a Python FastAPI backend that runs a LangGraph workflow. MongoDB Atlas holds all data, vector search indexes and agent memory. Gemini on Vertex AI is the LLM, and everything runs on Google Cloud.

| Layer | Technology | Role |
|---|---|---|
| Frontend | React + Vite + TypeScript | Customer site and admin site, with persona-based login redirect |
| API | FastAPI | Customer and admin endpoints (demo personas, no authentication), onboarding state, human review actions |
| Orchestration | LangGraph | The onboarding workflow: deterministic steps (state machine, review gates) combined with agentic steps. Supports durable execution, pause/resume and human-in-the-loop |
| Agents | LangChain `create_agent` | Risk, Compliance and Document agents, each with its own tools and prompt. Middleware is used for guardrails and retries |
| Data, retrieval and memory | MongoDB Atlas with `langchain-mongodb` | `MongoDBAtlasVectorSearch` and the hybrid (vector + full-text) retriever for regulations, procedures, past onboardings and documents. Run state and long-term memory are also stored in Atlas |
| LLM and embeddings | Gemini via `langchain[google-genai]` | Reasoning, extraction, document drafting |
| Observability | LangSmith | Tracing, debugging and evaluation of agent runs |
| Hosting | Google Cloud Run | Frontend and API containers |

**Design choices**
- LangChain agents are built on LangGraph, so persistence and human-in-the-loop come from the same layer.
- Deep Agents and LangSmith Fleet (no-code agents) are not used, because the workflow has fixed steps and needs fine-grained control.
- Guardrails are stored as versioned data in MongoDB and are updated only after human approval.
- Spring Boot is not used, to keep the build to one backend language and one service.
- The MongoDB checkpointer and long-term store must be confirmed in a short spike before the event. If unavailable, long-term memory falls back to a plain MongoDB collection.

## Repo and Development setup
- `frontend/` - React app. `backend/` - FastAPI app, agents and scripts (layout in BACKEND-PLAN.md). `docker-compose` or Cloud Run configs as needed.
- Configuration through environment variables (Mongo connection string, Google Cloud project, Gemini model names, LangSmith keys, `VITE_API_URL`). No secrets in the repository.
- Scripts: seed data, ingest regulations and procedures, create vector and full-text indexes.

## MongoDB Atlas
- Collections and indexes are listed in BACKEND-PLAN.md.
- Vector Search indexes: `regulations`, `procedures`, `document_chunks`, and optionally past `onboardings`, with metadata pre-filters (jurisdiction, product, sector).
- Atlas free tier limits the number of search indexes, so keep it to those above.

## Security and compliance
- Use synthetic data only, with no real customer information.
- There is no authentication in the demo. Endpoints are open, so the app must never be deployed with real data.
- Plan PII handling, with field-level encryption as a stretch goal.
- Keep an immutable audit log of agent actions and human decisions.
- Add prompt-injection guardrails for uploaded documents.
- Require explainability and citations, and make the agent recommend only, with the human deciding.
- Name the Canadian regulations in the knowledge base: FINTRAC, PCMLTFA, PIPEDA, and OSFI guidance.

## Demo and seed data
Include a seed script, a demo script, and 3-4 scripted scenarios:
- Low-risk customer
- High-risk customer
- High-risk customer whose predicted future risk is low (needs risk prediction, which is stretch)
- Customer with missing documents (triggers the information request)

The demo should show two moments: a customer returning later and the workflow resuming with memory, and an admin override that leads to an approved guardrail change affecting the next similar case.