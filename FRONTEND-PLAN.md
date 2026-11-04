## Frontend Implementation

React + Vite + TypeScript + shadcn/ui + Tailwind CSS. React Router for navigation, a data-fetching library with polling (for example TanStack Query) for agent status, a component library for consistent UI, and a chart library for the risk map. The API base URL comes from an environment variable (`VITE_API_URL`). Features marked (Stretch) are built only if time allows. See PLAN.md for the MVP definition and BACKEND-PLAN.md for the endpoints.

### General behaviour
- Agent steps are asynchronous. Pages that show onboarding status poll the backend every few seconds, and show progress, loading and error states. Failed steps show a clear message and, for admins, a retry action.
- Keep the UI simple and accessible: labelled form fields, keyboard navigation, sufficient contrast, plain language for customers.
- Types for API responses are shared in one place, matching the backend schemas.

### Login Portal (persona picker, no authentication)
- The portal lists the seeded personas from `GET /demo/users` (customers and bank employees). Selecting one redirects to the customer site or admin site according to its role.
- The selected persona is kept in app state and sent as `customer_id` (customer calls) or `admin_id` (admin calls). There are no passwords or tokens.
- A persona switcher is always visible in the header so the demo can move quickly between customer and admin.

### Theme
The UI theme must be visually pleasing, and implement a cool truthworthy blue.
Palette: blue as the primary colour, a neutral grey or white background, and red, amber and green reserved for risk levels and status only. Don't rely on colour alone for risk. Add labels or icons so the risk map stays accessible, which the frontend plan already asks for.


### Customer facing site
Simple and accessible. Features:

**Feature #1 (MVP): Start onboarding.** A short multi-step form collects business details, owners, and products requested. On submit, the customer sees the onboarding id and the list of required documents returned by the backend.

**Feature #2 (MVP): Track onboarding.** Shows the current step, steps completed and remaining, estimated completion time, and the contact details of the team responsible for the current step. A progress stepper covers Apply, Verify, Risk, Compliance, Documents, Review, Sign, Onboard.

**Feature #3 (MVP): Provide information and documents.** When the status is "needs information" (or required documents are listed), the customer sees what is requested and can upload documents or answer questions. A documents list shows uploaded and generated documents with download.

**Feature #4 (MVP): E-sign.** When documents are ready, the customer sees them and a "Sign" button. This is a mock: it calls the sign endpoint and the onboarding completes. No real signature handling.

**Feature #5 (MVP): View completed onboarding details.** Summary of the finished onboarding: pricing, benefits, access, and documents.

**Feature #6 (MVP): Notifications.** In-app notification bell with unread count, a list of notifications, and mark-as-read. Email notifications are sent by the backend and need no frontend work (Stretch).

**Feature #7 (Stretch): Cancel onboarding.** A cancel button with a confirmation dialog, available at any non-final step. The customer is told a team member will reach out.

### Admin facing site
Employees maintain customer profiles and step in when needed. Features:

**Feature #1 (MVP): Onboarding list and detail.** A filterable list (status, risk level, date, sector) of in-progress and completed onboardings. The detail page shows the application, state history, documents, audit trail, and an agent trace panel (agent steps, sources and confidence), so reviewers can see why agents produced each result.

**Feature #2 (MVP): Review queue and decision.** A queue of onboardings waiting at a review gate. There are two gates: after Risk + Compliance (including recommended pricing, shown read-only), and after Document generation, before anything is shown to the customer. The reviewer can read the agent output with citations, add comments, edit where allowed, and choose Continue, Pause, Request information, or Cancel. Edits are recorded as overrides.

**Feature #3 (MVP): Risk map.** For each onboarding, a read-only view of the Risk agent's results: an overall score, a per-category breakdown (for example a radar chart or heatmap) highlighting high and low risk areas, the agent's explanation for each category with cited sources, and the bank's risk appetite for the customer's sector and type.
- (Stretch) Interactivity: click or hover a category to drill into its explanation and sources, and filter or compare areas.
- (Stretch) Risk prediction: a trend indicator (expected to increase, decrease or stay stable) with the rationale.

**Feature #4 (Stretch): Pricing adjustments.** The agent-recommended pricing, which sits within the risk appetite, is shown read-only in the review (MVP). In this feature the admin can adjust it and must give a reason. The original and adjusted values are shown.

**Feature #5 (Stretch): Customer contact.** From an onboarding, the admin can send a message or information request to the customer at any step.

**Feature #6 (Stretch): E-sign.** Mock "Sign" button for the bank side, if needed.

**Feature #7 (MVP): Guardrails.** A page with the active guardrails and version history, and a list of guardrail proposals generated from admin overrides. Each proposal shows the overrides that led to it, and the admin can approve or reject it. This demonstrates the system adapting its guardrails with human approval.

**Feature #8 (Stretch): Metrics.** A small dashboard: average time to onboard, count by state, and agent override rate. Use it to show the before and after improvement in the demo.


