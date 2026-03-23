# VQMS — Agent Architecture Review and Final Recommended Flow

**Project:** Vendor Query Management System (VQMS)  
**Review Type:** Principal Architecture Review — Agent Simplification  
**Original Design:** 12-Agent Multi-Agent Orchestration  
**Recommended Design:** 3 Core Agents + 1 Optional Agent + 5 Services  
**Date:** March 2026

---

## Step 1. Project Understanding

### What This Project Is

VQMS automates how your company handles vendor emails. A vendor writes in about a late payment or a contract question. The system reads the email, figures out what they want, looks them up in Salesforce, opens a ServiceNow ticket, sends back an acknowledgment, routes the ticket to the right team, and tracks it until someone resolves it and closes it out.

That's the whole thing. Email in, work done, ticket closed, vendor happy.

### The Problem It Fixes

Right now, a human support person does all of this by hand. They read the email, guess what it's about, dig through Salesforce, type up a ticket, write a reply, and then try to remember to follow up before the SLA expires. Multiply that by hundreds of emails a week and the cracks show up fast.

Different people categorize the same email differently. Tickets get created with half the fields missing. Acknowledgments go out late or not at all. SLA deadlines get missed because nobody's watching the clock. When a vendor replies to a months-old thread, somebody creates a duplicate ticket because they didn't realize the original existed.

VQMS takes over the repetitive parts of this cycle and lets humans focus on the judgment calls — the ambiguous emails, the sensitive situations, the cases where the AI isn't sure what to do.

### What the System Does, End to End

Ten steps, start to finish:

1. Capture the vendor email (sender, subject, thread ID, attachments)
2. Read and understand the email (what do they want, who are they, how urgent is it)
3. Look up the vendor in Salesforce (match by email, then vendor ID, then name)
4. Create or update a ServiceNow ticket
5. Send an acknowledgment back to the vendor
6. Route the ticket to the right resolver team (Finance, Procurement, IT, Legal)
7. Package investigation context so the resolver doesn't start from scratch
8. Handle hold/resume when someone's waiting for information
9. Escalate when SLA deadlines approach (70%, 85%, 95%)
10. Resolve and close — and handle it cleanly when a vendor replies after closure

On top of that, the system watches for patterns. If seven vendors complain about payments on the same day, it flags a possible batch processing failure instead of treating each email as unrelated.

### Where You Actually Need AI Agents

Not everywhere. Only where the work requires real reasoning:

- **Reading and understanding the email.** You need an LLM to classify intent, pull out invoice numbers and vendor names, gauge urgency, and score its own confidence.
- **Deciding what to do and in what order.** An orchestrator agent that can build a plan, run things in parallel, and replan when something unexpected comes back.
- **Writing emails.** Drafting replies that sound right, reference the correct ticket and SLA, maintain thread continuity, and adapt tone to the vendor's situation. Templates alone can't do this well.
- **Helping resolvers investigate.** Finding similar past cases, pulling up knowledge base articles, suggesting resolutions. Useful copilot work.

### Where Rules and Services Are Enough

A lot of this system is plumbing, not intelligence:

- **Parsing emails** is string extraction, not reasoning. A service handles it.
- **Looking up a vendor in Salesforce** is a database query with a fuzzy matching fallback. Not an agent.
- **Creating and updating ServiceNow tickets** is CRUD. It does what the orchestrator tells it to do.
- **Routing** is a lookup table: this intent goes to that team, adjusted by vendor tier. You might need an LLM for odd edge cases, but not a whole agent.
- **SLA monitoring** is timers. 70% consumed? Fire an alert. That's a cron job, not an agent.
- **Escalation logic** is a rules check: SLA% + vendor tier + sentiment = escalation level. The orchestrator can handle this inline.
- **Memory** is a database. It stores things and retrieves them. Calling it an "agent" is like calling a filing cabinet a coworker.

### The Bottom Line

The original design calls everything an agent. Some of those things are agents — they reason, plan, or generate text. Most of them are services, connectors, or rules engines doing straightforward work. Giving each one the "agent" label doesn't make it smarter. It just makes the system harder to build, debug, and run.

---

## Step 2. Agent Architecture Review

### All 12 Proposed Agents — What's Worth Keeping

| # | Original Name | Recommended Name | Keep? | Why |
|---|---|---|---|---|
| 1 | Supervisor Agent | **Workflow Orchestration Agent** | **Yes** | The brain. Plans work, dispatches tasks, joins results, replans when needed. Can't remove this. |
| 2 | Email Intake Agent | *Email Ingestion Service* | **Demote to service** | Parses email headers and detects new vs. reply. No reasoning involved. |
| 3 | Understanding Agent | **Email Analysis Agent** | **Yes** | The most important LLM task in the system. Intent, entities, urgency, sentiment, confidence. |
| 4 | Vendor Resolution Agent | *Vendor Resolution Service* | **Demote to service** | Salesforce lookup with a 3-step matching cascade. A query, not a thinker. |
| 5 | Ticket Management Agent | *Ticket Operations Service* | **Demote to service** | ServiceNow CRUD. Follows instructions. |
| 6 | Routing Agent | *Fold into Orchestration* | **Remove** | A routing table with priority adjustments. The orchestrator does this in-line. |
| 7 | Communication Agent | **Communication Drafting Agent** | **Yes** | Writes all outbound vendor emails. Needs LLM for tone, context, personalization. |
| 8 | Investigation Support Agent | **Resolver Agent** | **Yes (optional)** | Builds context packages for human resolvers. High value, but not on the critical path for Phase 1. |
| 9 | Validation Agent | *Quality & Governance Gate* | **Demote to service** | Mostly rule-based checks (ticket number match, PII scan, template compliance). Uses an LLM call occasionally, but that doesn't make it an agent. |
| 10 | Escalation Agent | *Fold into Orchestration* | **Remove** | SLA% + tier + inactivity = escalation level. That's a formula, not an agent. |
| 11 | Memory Agent | *Memory & Context Service* | **Demote to service** | A data layer. Stores things, retrieves things, manages TTLs. |
| 12 | Monitoring Agent | *Monitoring & SLA Alerting Service* | **Demote to service** | Timer-based alerts and periodic pattern detection. A background job. |

### What Gets Combined

| New Component | Absorbs | Reason |
|---|---|---|
| Email Ingestion Service | Email Intake Agent | Parsing, not thinking |
| Vendor Resolution Service | Vendor Resolution Agent | API lookups with a matching algorithm |
| Ticket Operations Service | Ticket Management Agent | ServiceNow API calls |
| Orchestration Agent | Routing Agent + Escalation Agent | Both are rules the orchestrator applies |
| Monitoring Service | Monitoring Agent | Timers and scheduled analysis |
| Memory & Context Service | Memory Agent | Database layer |
| Quality & Governance Gate | Validation Agent | Rule-based checks with occasional LLM help |

### Where Humans Stay in the Loop

| When | What Happens | Why Automation Can't Handle It |
|---|---|---|
| AI confidence is low | Human reviews the AI's best guess, corrects it, confirms the right category | The AI admitted it's unsure. Someone has to decide. |
| Vendor can't be matched | Human identifies the vendor manually | Guessing vendor identity has real consequences. |
| Email is high-risk | Human reviews the draft before it sends | Legal issues, angry vendors, large amounts — too risky for auto-send. |
| L2+ escalation | Human approves the escalation to senior management | Organizational authority, not AI authority. |
| Complex resolution | Human confirms the fix before the closure email goes out | Wrong resolution email to a vendor is hard to walk back. |

### Verdict

| | |
|---|---|
| Should you build all 12 agents? | **No.** |
| How many do you actually need? | **3 core + 1 optional = 4 agents max** |
| What happens if you keep 12? | More moving parts, more latency between agents, more things that break, harder to debug, higher cloud bill, and a lot of "agents" that are just doing API calls. |
| What happens with 4? | Faster processing, simpler graph, easier debugging, lower cost, and an honest line between "things that think" and "things that fetch." |

---

## Step 3. Final Recommended Agents

### The Three You Must Build

**1. Workflow Orchestration Agent**

This is the coordinator. Every inbound event lands here — new email, vendor reply, SLA alert, pattern anomaly. It looks at what came in, checks Memory for context, builds a plan, and starts dispatching work. If something comes back that changes the picture (low confidence, vendor not found, ticket already exists), it replans instead of blindly pushing forward.

It also handles routing logic and escalation rules directly. Those don't need separate agents — they're decision points the orchestrator evaluates using rules and context.

Runs on LangGraph with state management. AI-driven.

**2. Email Analysis Agent**

This one reads the vendor's email and answers the questions that everything else depends on: What does the vendor want? Who are they? How urgent is this? Are they angry? How confident is the AI in these answers? Are there multiple issues packed into one email?

If this agent gets it wrong, everything downstream goes wrong — wrong routing, wrong acknowledgment, wrong SLA. So it deserves its own focused agent with purpose-built prompts and potentially different models for classification vs. extraction.

LLM-powered. The most important AI component in the system.

**3. Communication Drafting Agent**

Every email the vendor sees comes from this agent. Acknowledgments, status updates, clarification requests, escalation notices, resolution confirmations, reopen acknowledgments. It has to get the tone right, reference the correct ticket and SLA details, maintain email thread continuity, and know whether to send automatically or hold for human review.

You can't do this with simple templates. The combinations of vendor context, ticket state, conversation history, and communication purpose are too varied. This needs LLM generation with template guardrails.

### One You Should Build (But Can Wait)

**4. Resolver Agent**

When a human resolver opens a ticket, this agent hands them a ready-made context package: vendor snapshot, ticket history, similar past cases pulled from vector search, relevant knowledge base articles, and draft response options they can use or ignore.

It also has a "research mode" for when the Monitoring Service spots a pattern (like seven vendors complaining about the same payment batch). In that mode, it investigates across tickets instead of supporting one specific case.

This is what makes the system more than a triage machine. But it's not on the critical path — the system works without it. Build it in Phase 2.

### Five Services (Not Agents)

These do defined jobs. They don't reason, plan, or make judgment calls.

**Email Ingestion Service** — Connects to the mailbox, parses headers, extracts attachments, detects new thread vs. reply, filters junk. Hands the orchestrator a clean structured payload.

**Vendor Resolution Service** — Takes the vendor identifiers the Email Analysis Agent extracted and runs them against Salesforce. Email match first, then vendor ID match, then fuzzy name match. Returns the vendor profile, tier, contracts, and any risk flags. Caches results for repeated lookups.

**Ticket Operations Service** — All the ServiceNow work: create tickets, update them, transition states, start/pause/resume SLA clocks, handle reopen logic, write audit entries. Idempotent, so retries can't create duplicates.

**Quality & Governance Gate** — The last checkpoint before anything goes out the door. Does the ticket number in the email match a real ticket? Is the SLA wording correct? Does the email follow the approved template? Any PII leaking? Policy says this can auto-send, or does it need human eyes? Returns pass, fail, or needs-review.

**Memory & Context Service** — Stores and retrieves everything across workflows. Short-term context for active cases (in Redis or Valkey), long-term vendor patterns (in DynamoDB or PostgreSQL), past resolutions for vector search (in OpenSearch or Qdrant), and agent execution traces. Every agent and service reads from it and writes to it.

### Two Infrastructure Components

**Monitoring & SLA Alerting Service** — Watches every active ticket's SLA clock. Fires alerts at 70% (warning to resolver), 85% (L1 escalation trigger), 95% (L2 trigger). Also runs periodic batch analysis looking for cross-vendor patterns. When it finds something, it sends an alert event to the Orchestration Agent.

**LLM Gateway Service** — Routes LLM calls to the right provider. Claude via Bedrock for most things, smaller or cheaper models for simple classification, OpenAI as fallback. Manages prompt versions, token budgets, rate limits, and failover. Agents call the gateway; they never talk to model providers directly.

### Human Checkpoints

| Situation | What Gets Reviewed |
|---|---|
| AI confidence below threshold | Human corrects classification, confirms vendor, assigns category |
| Vendor not found in Salesforce | Human identifies the vendor manually |
| High-risk or sensitive email | Human reviews draft before it sends |
| L2+ escalation | Human approves the notification |
| Complex resolution | Human confirms the fix before closure email goes out |

### Count

| What | How Many |
|---|---|
| Core AI agents | 3 (Orchestration, Email Analysis, Communication Drafting) |
| Optional AI agent | 1 (Resolver Agent) |
| Services | 5 (Email Ingestion, Vendor Resolution, Ticket Ops, Quality Gate, Memory) |
| Infrastructure | 2 (Monitoring, LLM Gateway) |
| Human checkpoints | 5 |

---

## Step 4. Technology Stack — AWS and Open Source

For every layer, here's the AWS-managed option and the open-source alternative. Your choice depends on how much operational overhead your team wants to take on.

### Agent Orchestration

| What | AWS | Open Source |
|---|---|---|
| Agent orchestration | Step Functions + Bedrock Agents | **LangGraph** (MIT, by LangChain). Graph-based multi-agent orchestration with state management, human-in-the-loop, conditional branching, and checkpointing. Used in production at Klarna, Replit, Elastic. |
| Workflow durability | Step Functions (built-in) | **Temporal** (MIT). Durable execution that survives crashes and network failures. SDKs for Python, Go, Java, TypeScript. |
| Agent observability | X-Ray + CloudWatch | **Langfuse** (MIT, open-source LLM observability and prompt management) or **LangSmith** (by LangChain, paid but full-featured) |

### LLM Access

| What | AWS | Open Source |
|---|---|---|
| Primary LLM | Bedrock (Claude, Titan) | **Ollama** + **vLLM** for self-hosted open models (Llama 3, Mistral) |
| LLM gateway / router | Bedrock model routing | **LiteLLM** (MIT). Unified API across 100+ providers. Load balancing, fallback, token tracking. |
| Prompt management | Bedrock Prompt Management | **Langfuse** or **LangSmith Prompt Hub**. Version control, A/B testing. |
| Embeddings | Bedrock Titan Embeddings | **Sentence Transformers** (Apache 2.0, Hugging Face) |

### Data Storage

| What | AWS | Open Source |
|---|---|---|
| Cache / short-term state | ElastiCache (Redis) | **Valkey** (Linux Foundation, BSD, drop-in Redis replacement) or **Redis OSS** |
| Long-term memory | DynamoDB | **PostgreSQL** + **pgvector** (relational + vector search in one database) |
| Vector search / past resolutions | OpenSearch Service | **OpenSearch** (Apache 2.0, v3.0 with GPU-accelerated vector search) or **Qdrant** (Apache 2.0, built for RAG) or **Milvus** (Apache 2.0, scales to billions of vectors) |
| File storage | S3 | **MinIO** (AGPL-3.0, S3-compatible) |
| Reporting / analytics | Aurora PostgreSQL | **PostgreSQL** |

### Messaging

| What | AWS | Open Source |
|---|---|---|
| Message queue | SQS FIFO | **RabbitMQ** (MPL-2.0) or **Apache Kafka** (Apache 2.0, better for high throughput and event sourcing) |
| Event bus | EventBridge | **Kafka** with Kafka Connect, or **NATS** (Apache 2.0, lightweight) |

### Monitoring

| What | AWS | Open Source |
|---|---|---|
| Metrics | CloudWatch | **Prometheus** (Apache 2.0) |
| Dashboards | CloudWatch Dashboards | **Grafana** (AGPL-3.0) |
| Tracing | X-Ray | **OpenTelemetry** (Apache 2.0) + **Jaeger** (Apache 2.0) |
| Logs | CloudWatch Logs | **Loki** (AGPL-3.0, by Grafana) or **OpenSearch** |
| Alerting | SNS + CloudWatch Alarms | **Grafana Alerting** or **Alertmanager** (Prometheus ecosystem) |

### Security

| What | AWS | Open Source |
|---|---|---|
| Identity / auth | IAM + Cognito | **Keycloak** (Apache 2.0, SSO, RBAC, OIDC/SAML) |
| Secrets | Secrets Manager | **HashiCorp Vault** (BUSL-1.1) |
| Encryption | KMS | Vault Transit engine or PostgreSQL native encryption |
| PII detection | Comprehend | **Presidio** (MIT, by Microsoft) |
| API gateway | API Gateway | **Kong** (Apache 2.0) or **Traefik** (MIT) |
| WAF | AWS WAF + Shield | **ModSecurity** (Apache 2.0) |

### Compute

| What | AWS | Open Source |
|---|---|---|
| Containers | ECS Fargate | **Kubernetes** (Apache 2.0) |
| Runtime | ECS | **Docker** / **containerd** |
| Infrastructure as code | CDK | **OpenTofu** (MPL-2.0, open fork of Terraform) or **Pulumi** (Apache 2.0) |
| CI/CD | CodePipeline | **GitLab CI** (MIT) or **GitHub Actions** or **Jenkins** |

### Email

| What | AWS | Open Source |
|---|---|---|
| Send / receive | SES + WorkMail | Direct integration with **Exchange** (Graph API) or **Google Workspace** (Gmail API) |
| Parsing | Lambda custom code | **mail-parser** (npm, MIT) or Python's built-in **email** library |

### Frontend

| What | AWS | Open Source |
|---|---|---|
| Framework | Amplify + React | **React** (MIT) or **Next.js** (MIT) |
| Components | — | **shadcn/ui** (MIT) or **Ant Design** (MIT) |
| Hosting | CloudFront + S3 | **Nginx** (BSD) or **Caddy** (Apache 2.0) |

### Two Ready-Made Stacks

**If you want full control and minimal vendor lock-in:**

LangGraph + Temporal for orchestration. LiteLLM gateway pointing at Claude API (primary) and self-hosted Ollama/vLLM (for cheaper tasks). PostgreSQL + pgvector for data and vectors in one place. Valkey for cache. MinIO for files. OpenSearch for logs and hybrid search. RabbitMQ or Kafka for messaging. Prometheus + Grafana + Loki + Jaeger + Langfuse for observability. Keycloak + Vault + Presidio + Kong for security. Kubernetes + Docker + OpenTofu for compute.

**If you want to ship fast on AWS:**

LangGraph on ECS Fargate + Step Functions. Bedrock for all LLM calls. DynamoDB + Aurora PostgreSQL + OpenSearch Service + ElastiCache + S3. SQS FIFO + EventBridge. CloudWatch + X-Ray. IAM + Cognito + Secrets Manager + KMS + Comprehend + API Gateway + WAF. ECS Fargate + CDK.

---

## Step 5. Vendor Reopen Ticket Flow

This is one of the trickier parts of the system to get right. A vendor replies to an old email after the ticket has been closed. Could be hours later, could be weeks. They might be saying "this still isn't fixed," or asking something new, or just saying thanks.

The system has to figure out which of those it is, and handle each one differently without creating duplicate tickets.

### How It Works

**Step 1.** The Email Ingestion Service picks up the reply. It finds a thread ID in the email headers that maps to an existing ticket. Flags the email as a reply with the ticket reference.

**Step 2.** The Orchestration Agent looks up that ticket and discovers it's closed. Now it knows this is a post-closure reply, not a normal conversation update.

**Step 3.** The Email Analysis Agent reads the new email. It's trying to answer one question: is this about the same problem, or a different one?

**Step 4.** The Orchestration Agent makes the call:

- **Same issue, not actually resolved?** Reopen the original ticket. Move it from "Closed" to "Reopened." Keep all the original history, resolver assignment, and vendor context. Start a fresh SLA clock.

- **Different issue?** Create a new ticket, but link it to the old one. The new ticket gets its own SLA and routing, but the resolver can see the related history.

- **Can't tell?** Route to human triage with both options laid out. Let a person decide.

**Step 5.** The Ticket Operations Service does whatever the Orchestration Agent decided — reopen or create-and-link. SLA tracking starts fresh either way.

**Step 6.** The Communication Drafting Agent writes an acknowledgment referencing the ticket number(s) and setting expectations.

**Step 7.** Quality Gate checks it. Email sends (or gets staged for human review if it's a sensitive case).

**Step 8.** The Orchestration Agent checks whether the original resolver team is still the right fit. If yes, reassigns. If things have changed, re-routes.

**Step 9.** Monitoring picks up the ticket and starts the SLA clock. Memory records the entire reopen event for future learning.

### Rules That Matter

- Never create a duplicate. Either reopen the old ticket or link a new one.
- Reopened tickets get a fresh SLA clock, not the leftovers from the original.
- Vendors always get an acknowledgment, even for reopens.
- If the system can't tell whether to reopen or create new, a human decides.
- Track reopens as a quality metric. Lots of reopens on one issue type means the resolutions aren't sticking.

---

## Step 6. Detailed End-to-End Flows

### Flow 1: New Vendor Email — Happy Path

```
====================================================================================
  FLOW 1: NEW VENDOR EMAIL — HAPPY PATH (HIGH CONFIDENCE)
====================================================================================

STEP 1 — VENDOR SENDS EMAIL
+--------------------------------------------------------------------------+
| INPUT:  Vendor emails support@company.com about a delayed payment        |
|         for invoice INV-90210                                            |
| OUTPUT: Raw email lands on the company mail server                       |
| WHO:    Vendor (external)                                                |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 2 — EMAIL INGESTION SERVICE PICKS IT UP
+--------------------------------------------------------------------------+
| INPUT:  Raw email from Exchange or Google Workspace                      |
|                                                                          |
| WHAT HAPPENS:                                                            |
|   - Pulls out sender, subject, timestamp                                 |
|   - Grabs the thread ID from headers (Message-ID, In-Reply-To)          |
|   - Checks: new thread or reply? → NEW THREAD (no prior ID found)       |
|   - Scans attachments for viruses, validates file types                  |
|   - Filters out spam and auto-replies                                    |
|   - Detects language: English                                            |
|                                                                          |
| OUTPUT: Clean structured payload                                         |
|   {                                                                      |
|     sender: "vendor@acme.com"                                            |
|     subject: "Payment delay - Invoice INV-90210"                         |
|     timestamp: "2026-03-19T10:15:00Z"                                    |
|     thread_id: "msg-abc-123" (new)                                       |
|     body: "We have not received payment for INV-90210..."                |
|     attachments: [invoice_copy.pdf]                                      |
|     is_reply: false                                                      |
|     language: "en"                                                       |
|   }                                                                      |
| WHO: Email Ingestion Service (automated, no AI)                          |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 3 — ORCHESTRATION AGENT GETS THE PAYLOAD AND MAKES A PLAN
+--------------------------------------------------------------------------+
| INPUT:  Structured email payload                                         |
|                                                                          |
| WHAT HAPPENS:                                                            |
|   - Checks Memory: has vendor@acme.com written before?                   |
|     Yes — 3 prior tickets, vendor ID V-4455                              |
|   - This is a new thread, so build a full execution plan:                |
|     1. Analyze the email                                                 |
|     2. If confidence is high, proceed. If low, send to triage.           |
|     3. Look up vendor + check for existing ticket (in parallel)          |
|     4. Route it                                                          |
|     5. Create ticket                                                     |
|     6. Draft and send acknowledgment                                     |
|                                                                          |
| OUTPUT: Plan created. Sends payload to Email Analysis Agent.             |
| WHO: Workflow Orchestration Agent (LangGraph)                            |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 4 — EMAIL ANALYSIS AGENT READS THE EMAIL
+--------------------------------------------------------------------------+
| INPUT:  Email payload + sender history from Memory                       |
|                                                                          |
| WHAT HAPPENS (LLM does the work):                                        |
|   - INTENT: "payment_status_inquiry"                                     |
|   - ENTITIES FOUND:                                                      |
|     vendor_name: "Acme Corp"                                             |
|     invoice_number: "INV-90210"                                          |
|     amount: "$45,000"                                                    |
|     date_referenced: "2026-02-28" (due date)                             |
|   - URGENCY: 3/5 (overdue, not critical)                                |
|   - SENTIMENT: Negative (frustrated but professional)                    |
|   - CONFIDENCE: 0.91 (clear intent, clear entities)                      |
|   - MULTIPLE ISSUES: No, just one                                        |
|                                                                          |
| OUTPUT:                                                                  |
|   {                                                                      |
|     primary_intent: "payment_status_inquiry"                             |
|     entities: { vendor: "Acme Corp", invoice: "INV-90210", ... }         |
|     urgency: 3, sentiment: "negative", confidence: 0.91                  |
|     multi_issue: false                                                   |
|   }                                                                      |
| WHO: Email Analysis Agent (LLM inference)                                |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 5 — ORCHESTRATION AGENT CHECKS CONFIDENCE
+--------------------------------------------------------------------------+
| INPUT:  Analysis result (confidence = 0.91)                              |
|                                                                          |
| WHAT HAPPENS:                                                            |
|   - 0.91 is above the 0.75 threshold → proceed automatically            |
|   - (Below 0.75 would go to human triage — see Flow 2)                  |
|   - Kicks off TWO things at the same time:                               |
|     Branch A: Look up the vendor in Salesforce                           |
|     Branch B: Check if a ticket already exists for this thread           |
|                                                                          |
| OUTPUT: Two parallel dispatches                                          |
| WHO: Workflow Orchestration Agent                                        |
+-----------------------------------+--------------------------------------+
                                    |
                    +---------------+----------------+
                    |                                |
                    v                                v
STEP 6A — VENDOR LOOKUP              STEP 6B — TICKET CHECK
(runs in parallel)                    (runs in parallel)
+--------------------------------+   +--------------------------------+
| INPUT: sender email +          |   | INPUT: thread_id               |
| extracted name "Acme Corp"     |   |                                |
|                                |   | WHAT HAPPENS:                  |
| WHAT HAPPENS:                  |   |   Searches ServiceNow for a    |
|   Email match in Salesforce:   |   |   ticket matching this thread  |
|   vendor@acme.com → HIT        |   |   Result: nothing found        |
|   Vendor ID: V-4455            |   |   (new thread, no prior ticket)|
|                                |   |                                |
|   Fetches full profile:        |   | OUTPUT: { existing_ticket:     |
|   - Status: Active             |   |   null }                       |
|   - Tier: 2 (Standard)        |   |                                |
|   - Contract: Active til 2027  |   | WHO: Ticket Operations Service |
|   - SLA: Standard (48hr)      |   +--------------------------------+
|   - Payment Terms: Net 30      |
|   - Risk Flags: none           |
|                                |
| OUTPUT:                        |
|   { vendor_id: "V-4455",      |
|     name: "Acme Corp",        |
|     tier: 2, status: Active,  |
|     sla_category: "Standard", |
|     risk_flags: [] }           |
|                                |
| WHO: Vendor Resolution Service |
+--------------------------------+
                    |                                |
                    +---------------+----------------+
                                    |
                                    v
STEP 7 — RESULTS COME BACK, ORCHESTRATOR ROUTES IT
+--------------------------------------------------------------------------+
| INPUT:  Vendor profile + ticket check result + analysis                  |
|                                                                          |
| WHAT HAPPENS:                                                            |
|   - Both parallel branches finished. Joins the results.                  |
|   - No existing ticket found → need to create one                        |
|   - Routing rules:                                                       |
|     "payment_status_inquiry" → Finance Accounts Payable                  |
|   - Priority adjustment:                                                 |
|     Base: Medium (payment inquiry)                                       |
|     Tier 2 vendor → no change                                            |
|     Urgency 3 + negative sentiment → bump to Medium-High                 |
|                                                                          |
| OUTPUT:                                                                  |
|   { resolver_group: "Finance-AP", priority: "Medium-High" }             |
| WHO: Workflow Orchestration Agent (rules-based routing)                   |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 8 — TICKET GETS CREATED IN SERVICENOW
+--------------------------------------------------------------------------+
| INPUT:  Everything from the analysis, vendor profile, routing            |
|                                                                          |
| WHAT HAPPENS:                                                            |
|   Creates ticket TKT-80301 with:                                         |
|     Category: Payment Inquiry                                            |
|     Vendor: Acme Corp (V-4455)                                           |
|     Priority: Medium-High                                                |
|     Assigned to: Finance-AP                                              |
|     SLA: Standard 48hr, deadline 2026-03-21T10:15:00Z                    |
|     Description: AI summary of the vendor's email                        |
|   Starts the SLA clock.                                                  |
|   Writes audit entry.                                                    |
|   (Idempotent — safe to retry without creating duplicates)               |
|                                                                          |
| OUTPUT: { ticket_id: "TKT-80301", state: "New",                         |
|           sla_deadline: "2026-03-21T10:15:00Z" }                         |
| WHO: Ticket Operations Service                                           |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 9 — COMMUNICATION AGENT DRAFTS THE ACKNOWLEDGMENT
+--------------------------------------------------------------------------+
| INPUT:  Ticket details, vendor profile, SLA deadline, thread ID          |
|                                                                          |
| WHAT HAPPENS (LLM writes the email):                                     |
|   Picks the acknowledgment template.                                     |
|   Generates a personalized reply:                                        |
|                                                                          |
|   "Dear Acme Corp team,                                                  |
|    Thank you for reaching out about invoice INV-90210.                   |
|    We've registered your inquiry as TKT-80301 and our Finance            |
|    team is reviewing the payment status. Expect an update                 |
|    within 48 hours..."                                                   |
|                                                                          |
|   Sets thread headers so the vendor's email client shows it              |
|   as a reply in the same conversation.                                   |
|   Send mode: AUTO (high confidence + routine query)                      |
|                                                                          |
| OUTPUT: Draft email package ready for validation                         |
| WHO: Communication Drafting Agent (LLM generation)                       |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 10 — QUALITY GATE CHECKS THE DRAFT
+--------------------------------------------------------------------------+
| INPUT:  Draft email + ticket data + vendor profile                       |
|                                                                          |
| CHECKS:                                                                  |
|   1. Ticket TKT-80301 exists in ServiceNow? → YES                       |
|   2. "48 hours" matches the Standard SLA? → YES                         |
|   3. Uses an approved template? → YES                                    |
|   4. Any PII leaking (SSN, card numbers)? → NO                          |
|   5. Policy allows auto-send for this case? → YES                       |
|                                                                          |
|   All five pass. Approved for auto-send.                                 |
|                                                                          |
| OUTPUT: { status: "pass" }                                               |
| WHO: Quality & Governance Gate (rules + selective LLM check)             |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 11 — EMAIL GOES OUT
+--------------------------------------------------------------------------+
| Validated email dispatched through Exchange / Google.                     |
| Thread headers keep the conversation connected.                          |
| Delivery confirmation logged.                                            |
|                                                                          |
| The vendor gets their acknowledgment.                                    |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 12 — AFTER SENDING
+--------------------------------------------------------------------------+
| MONITORING SERVICE:                                                      |
|   Picks up TKT-80301. Sets SLA alert thresholds:                        |
|     70% → warning to resolver                                            |
|     85% → L1 escalation to Orchestration Agent                          |
|     95% → L2 escalation                                                  |
|                                                                          |
| MEMORY SERVICE:                                                          |
|   Stores the complete workflow: email, analysis, vendor match,           |
|   ticket, routing, draft, validation result.                             |
|   Indexes by vendor, ticket, and intent type.                            |
|                                                                          |
| RESOLVER AGENT (if active):                                              |
|   Builds a context package for the Finance-AP resolver:                  |
|   Acme Corp snapshot, 3 prior tickets, similar payment inquiries,        |
|   relevant KB articles.                                                  |
+--------------------------------------------------------------------------+

                     FLOW 1 DONE.
       Vendor has their acknowledgment. Ticket is queued.
       SLA clock is ticking. Resolver has context.
```


### Flow 2: Low Confidence — Human Has to Step In

```
====================================================================================
  FLOW 2: LOW CONFIDENCE — HUMAN TRIAGE
====================================================================================

STEPS 1-3: Same as Flow 1. Email comes in, gets parsed, hits the
Orchestration Agent, goes to the Email Analysis Agent.
                                    |
                                    v
STEP 4 — ANALYSIS COMES BACK UNCERTAIN
+--------------------------------------------------------------------------+
| The email is a mess — mixes a payment complaint with a contract          |
| reference and doesn't include a clear vendor ID or invoice number.       |
|                                                                          |
| OUTPUT:                                                                  |
|   primary_intent: "payment_dispute" (52% sure)                          |
|   secondary_intent: "contract_inquiry" (38% sure)                       |
|   entities: vendor name is fuzzy, no invoice found                       |
|   confidence: 0.52 — well below the 0.75 threshold                      |
|                                                                          |
| WHO: Email Analysis Agent                                                |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 5 — ORCHESTRATION AGENT SWITCHES TO TRIAGE MODE
+--------------------------------------------------------------------------+
| Confidence is too low for automation. But it doesn't just stop.          |
|                                                                          |
| It still runs vendor matching (best effort) and checks Memory            |
| for similar past triage cases — in parallel.                             |
|                                                                          |
| Then it assembles a TRIAGE PACKAGE:                                      |
|   - The original email                                                   |
|   - The AI's best guess with uncertainty markers                         |
|   - Best-effort vendor match (Acme Corp, 61% confidence)                |
|   - 3 similar triage cases from the past                                 |
|   - Clear explanation of what the AI couldn't figure out                 |
|                                                                          |
| Creates a preliminary ticket (TKT-80302, state: Pending Triage)          |
| so the case enters the queue even before the human looks at it.          |
|                                                                          |
| WHO: Workflow Orchestration Agent                                        |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 6 — HUMAN TRIAGE REVIEWER
+--------------------------------------------------------------------------+
| A support agent opens the triage package in their workspace.             |
|                                                                          |
| They read the original email, look at the AI's guesses, and decide:     |
|   - This is a payment dispute, not a contract question                   |
|   - Vendor is Acme Corp (V-4455)                                        |
|   - Category: Payment Dispute                                            |
|                                                                          |
| They click "Confirm and Resume."                                         |
|                                                                          |
| WHO: Support Agent (human)                                               |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 7 — CORRECTIONS SAVED, FLOW RESUMES
+--------------------------------------------------------------------------+
| The human's corrections go to Memory so the AI learns from them.         |
| Next time a similar email arrives, confidence should be higher.          |
|                                                                          |
| The Orchestration Agent picks up from routing onward, using the          |
| human-confirmed data. TKT-80302 goes from "Pending Triage" to "New."   |
| Rest of the flow is the same as Flow 1 (Steps 7-12).                    |
+--------------------------------------------------------------------------+

                     FLOW 2 DONE.
       Human fixed what the AI couldn't. System learned from it.
```


### Flow 3: Vendor Reply on Existing Thread

```
====================================================================================
  FLOW 3: VENDOR REPLY ON AN EXISTING THREAD
====================================================================================

STEP 1 — INGESTION SERVICE SPOTS A REPLY
+--------------------------------------------------------------------------+
| Email from vendor@acme.com. Thread ID matches ticket TKT-80301.         |
| Flagged as a reply.                                                      |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 2 — ORCHESTRATION AGENT RECOGNIZES THE CONTEXT
+--------------------------------------------------------------------------+
| Looks up TKT-80301: in progress, assigned to Finance-AP,                 |
| vendor is Acme Corp, tier 2. All known.                                  |
|                                                                          |
| Skips vendor resolution and routing — those facts haven't changed.       |
| Builds a short plan: analyze the new email, update the ticket,           |
| send a brief acknowledgment.                                             |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 3 — EMAIL ANALYSIS (NEW CONTENT ONLY)
+--------------------------------------------------------------------------+
| Reads just the new email, not the whole history.                         |
| Result: vendor is providing the documents they were asked for.           |
| Confidence: 0.94.                                                        |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 4 — TICKET UPDATE
+--------------------------------------------------------------------------+
| Appends the reply to TKT-80301.                                          |
| Ticket was on hold waiting for info → resumes SLA clock.                 |
| State: "On Hold" → "In Progress"                                        |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEPS 5-6 — DRAFT, VALIDATE, SEND
+--------------------------------------------------------------------------+
| Short acknowledgment: "Thanks for the receipt. Our Finance team          |
| has been notified."                                                      |
| Quality Gate: pass. Sent.                                                |
+--------------------------------------------------------------------------+

                     FLOW 3 DONE.
       Reply appended. SLA resumed. Resolver notified.
```


### Flow 4: SLA About to Breach — Escalation

```
====================================================================================
  FLOW 4: SLA ESCALATION (PROACTIVE — NO INBOUND EMAIL)
====================================================================================

STEP 1 — MONITORING SPOTS TROUBLE
+--------------------------------------------------------------------------+
| TKT-80301 has burned through 85% of its 48-hour SLA.                    |
| No one's touched it in 12 hours.                                         |
| Alert fires to the Orchestration Agent.                                  |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 2 — ORCHESTRATION AGENT EVALUATES
+--------------------------------------------------------------------------+
| Pulls ticket context, vendor tier, communication history from Memory.    |
| Applies escalation rules: 85% SLA + tier 2 + 12hr idle + negative       |
| sentiment → L1 escalation to team lead.                                  |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 3 — COMMUNICATION AGENT DRAFTS ESCALATION NOTICE
+--------------------------------------------------------------------------+
| Internal email to team lead:                                             |
| "TKT-80301 (Acme Corp, payment inquiry) is at 85% SLA with no           |
| activity for 12 hours. Vendor sentiment is negative. Please review."     |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEPS 4-5 — VALIDATE, SEND, UPDATE TICKET
+--------------------------------------------------------------------------+
| Quality Gate: pass. Notice sent to team lead.                            |
| Ticket marked as escalated with timestamp.                               |
| Monitoring keeps watching — 95% triggers L2 to manager.                  |
+--------------------------------------------------------------------------+

                     FLOW 4 DONE.
```


### Flow 5: Vendor Replies After Closure — Reopen

```
====================================================================================
  FLOW 5: POST-CLOSURE REPLY — REOPEN FLOW
====================================================================================

STEP 1 — INGESTION SERVICE CATCHES A POST-CLOSURE REPLY
+--------------------------------------------------------------------------+
| Vendor emails about the same thread. Thread ID maps to TKT-80301.        |
| But TKT-80301 was closed last week.                                      |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 2 — ORCHESTRATION AGENT FINDS A CLOSED TICKET
+--------------------------------------------------------------------------+
| Looks up TKT-80301: status is "Closed."                                  |
| This is a post-closure reply. Sends it to the Email Analysis Agent.      |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 3 — EMAIL ANALYSIS AGENT READS THE NEW EMAIL
+--------------------------------------------------------------------------+
| The vendor wrote: "The payment still hasn't arrived. I thought            |
| this was resolved but the amount is still outstanding."                   |
|                                                                          |
| Analysis: Same issue as before. Same invoice (INV-90210).                |
| Vendor says it's not fixed.                                              |
| Recommendation: REOPEN the original ticket.                              |
| Confidence: 0.89                                                         |
|                                                                          |
| (If the vendor had asked about something different — say, a new          |
| contract question — the recommendation would be: create a new            |
| linked ticket instead.)                                                  |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 4 — ORCHESTRATION AGENT DECIDES: REOPEN
+--------------------------------------------------------------------------+
| Same intent, same entities, high confidence → reopen TKT-80301.          |
|                                                                          |
| (Low confidence? Route to human triage with both options.)               |
| (Different issue? Create new ticket linked to TKT-80301.)               |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 5 — TICKET OPERATIONS SERVICE REOPENS THE TICKET
+--------------------------------------------------------------------------+
| TKT-80301: "Closed" → "Reopened"                                        |
| New email appended to ticket history.                                    |
| Fresh SLA clock starts (new 48hr window).                                |
| All original history preserved.                                          |
| Reopen counter: 1 (quality metric).                                      |
| Audit entry logged.                                                      |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 6 — ROUTING CHECK
+--------------------------------------------------------------------------+
| Original team was Finance-AP. Still the right fit for a payment issue.   |
| Reassigned to Finance-AP with updated context.                           |
|                                                                          |
| (If the team had changed or the issue shifted, it would re-route.)       |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 7 — REOPEN ACKNOWLEDGMENT
+--------------------------------------------------------------------------+
| Communication Drafting Agent writes:                                     |
| "We've received your follow-up about INV-90210. TKT-80301 has been      |
| reopened and reassigned to our Finance team. We apologize for the        |
| inconvenience. You'll hear from us within 48 hours."                     |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEPS 8-9 — VALIDATE, SEND, MONITOR
+--------------------------------------------------------------------------+
| Quality Gate: pass. Email sent.                                          |
| Monitoring picks up the reopened ticket with fresh SLA deadline.         |
| Memory stores the reopen event for future learning.                      |
|                                                                          |
| Background: reopen rates get tracked. If a particular issue type         |
| or resolver group keeps getting reopened, that's a quality signal        |
| for management.                                                          |
+--------------------------------------------------------------------------+

                     FLOW 5 DONE.
       Ticket reopened. Fresh SLA. No duplicate created.
       Full history preserved. Vendor acknowledged.
```


### Flow 6: Resolution and Closure

```
====================================================================================
  FLOW 6: RESOLUTION AND CLOSURE
====================================================================================

STEP 1 — RESOLVER FINISHES THE WORK
+--------------------------------------------------------------------------+
| Finance-AP resolver opens TKT-80301, reviews the Resolver Agent's        |
| context package, investigates, resolves the payment issue.               |
| Adds notes: "Payment re-processed, ETA 3 business days."                |
| Clicks "Mark as Resolved."                                               |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 2 — CLOSURE EMAIL
+--------------------------------------------------------------------------+
| Communication Drafting Agent writes:                                     |
| "Your payment for INV-90210 has been re-processed. You should            |
| receive it within 3 business days. If anything else comes up,            |
| reply to this email. Ticket: TKT-80301."                                 |
|                                                                          |
| Quality Gate: pass. Sent.                                                |
+-----------------------------------+--------------------------------------+
                                    |
                                    v
STEP 3 — WHAT HAPPENS NEXT (THREE PATHS)
+--------------------------------------------------------------------------+
|                                                                          |
| PATH A: Vendor replies "Got it, thank you."                              |
|   → Ticket goes from "Resolved" to "Closed."                            |
|   → Audit summary generated.                                             |
|                                                                          |
| PATH B: No response for 5 days (or whatever the policy says).           |
|   → Auto-closes. Audit summary generated.                                |
|                                                                          |
| PATH C: Vendor replies "Still not fixed."                                |
|   → Triggers FLOW 5 (Reopen).                                           |
|                                                                          |
+--------------------------------------------------------------------------+

                     FLOW 6 DONE.
```

---

*End of VQMS Agent Architecture Review.*  
*Original design: 12 agents. Recommended: 3 core agents + 1 optional + 5 services.*  
*Six flows covered: new email, low confidence, reply, SLA escalation, reopen, and closure.*
