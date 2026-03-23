# VQMS — Complete Architecture & End-to-End Flow Reference

**Project:** Vendor Query Management System (VQMS)  
**Architecture:** 3 Core AI Agents + 1 Optional Agent + 5 Services  
**Date:** March 2026

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture Components](#2-architecture-components)
3. [Component Interaction Map](#3-component-interaction-map)
4. [Confidence Scoring — How It Works](#4-confidence-scoring--how-it-works)
5. [Flow 1: New Vendor Email — Happy Path (High Confidence)](#5-flow-1-new-vendor-email--happy-path)
6. [Flow 2: Low Confidence — Human Triage Path](#6-flow-2-low-confidence--human-triage-path)
7. [Flow 3: Vendor Reply on Existing Thread](#7-flow-3-vendor-reply-on-existing-thread)
8. [Flow 4: SLA Breach Escalation (Proactive)](#8-flow-4-sla-breach-escalation)
9. [Flow 5: Vendor Reopens After Closure](#9-flow-5-vendor-reopens-after-closure)
10. [Flow 6: Resolution and Closure](#10-flow-6-resolution-and-closure)
11. [Human-in-the-Loop Checkpoints](#11-human-in-the-loop-checkpoints)
12. [Technology Stack](#12-technology-stack)

---

## 1. System Overview

### What VQMS Does

VQMS automates the entire lifecycle of vendor email handling — from the moment a vendor
sends an email (about a payment, contract, or issue) all the way through to resolution
and ticket closure. It replaces manual work done by human support executives with
AI-driven automation while keeping humans in control of high-risk decisions.

### The 10-Step Lifecycle

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    VQMS — 10-STEP LIFECYCLE                             │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   1. CAPTURE ──► 2. UNDERSTAND ──► 3. IDENTIFY VENDOR ──►               │
│                                                                          │
│   4. CREATE/UPDATE TICKET ──► 5. SEND ACKNOWLEDGMENT ──►                │
│                                                                          │
│   6. ROUTE TO RESOLVER GROUP ──► 7. PACKAGE INVESTIGATION CONTEXT ──►   │
│                                                                          │
│   8. MANAGE HOLD/RESUME ──► 9. ESCALATE ON SLA BREACH ──►              │
│                                                                          │
│   10. RESOLVE, CLOSE, HANDLE REOPENS                                    │
│                                                                          │
│   + PROACTIVE: SLA breach monitoring + Cross-vendor pattern detection   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Why 3 Agents Instead of 12

The original 12-agent design treated every function as a separate AI agent. But 8 of
those functions are really just integration services, rules engines, or data layers
that don't need autonomous AI reasoning.

```
┌─────────────────────────────────────────────────────────────────────┐
│              ORIGINAL 12 AGENTS → RECOMMENDED 3+1+5                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   KEPT AS AI AGENTS (need LLM reasoning):                          │
│   ┌─────────────────────────────────────────┐                      │
│   │  1. Workflow Orchestration Agent    [AI] │                      │
│   │  2. Email Analysis Agent           [AI] │                      │
│   │  3. Communication Drafting Agent   [AI] │                      │
│   │  4. Resolver Agent (optional)      [AI] │                      │
│   └─────────────────────────────────────────┘                      │
│                                                                     │
│   DOWNGRADED TO SERVICES (no AI reasoning needed):                 │
│   ┌─────────────────────────────────────────┐                      │
│   │  1. Email Ingestion Service       [SVC] │  ← was Email Intake  │
│   │  2. Vendor Resolution Service     [SVC] │  ← was Vendor Agent  │
│   │  3. Ticket Operations Service     [SVC] │  ← was Ticket Agent  │
│   │  4. Quality & Governance Gate     [SVC] │  ← was Validation    │
│   │  5. Memory & Context Service      [SVC] │  ← was Memory Agent  │
│   └─────────────────────────────────────────┘                      │
│                                                                     │
│   ABSORBED INTO ORCHESTRATION:                                     │
│   ┌─────────────────────────────────────────┐                      │
│   │  Routing Agent logic      → Orchestrator │                      │
│   │  Escalation Agent logic   → Orchestrator │                      │
│   └─────────────────────────────────────────┘                      │
│                                                                     │
│   INFRASTRUCTURE:                                                  │
│   ┌─────────────────────────────────────────┐                      │
│   │  Monitoring & SLA Alerting Service      │                      │
│   │  LLM Gateway Service                   │                      │
│   └─────────────────────────────────────────┘                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Architecture Components

### Core AI Agents

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        CORE AI AGENTS                                    │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  AGENT 1: WORKFLOW ORCHESTRATION AGENT                            │  │
│  │  ─────────────────────────────────────                            │  │
│  │  Purpose:  The coordinator. Receives every inbound event          │  │
│  │            (new email, reply, SLA alert, pattern anomaly).        │  │
│  │            Decides what needs to happen, builds execution plans,  │  │
│  │            dispatches work, joins parallel results, replans       │  │
│  │            when something changes mid-workflow.                   │  │
│  │            ALSO absorbs routing rules + escalation logic.         │  │
│  │                                                                    │  │
│  │  Inputs:   Structured email payload, Memory context,              │  │
│  │            Monitoring alerts, Human corrections                   │  │
│  │  Outputs:  Execution plans, dispatch instructions, routing        │  │
│  │            decisions, escalation triggers, completion signals     │  │
│  │  Type:     AI-driven (LLM-powered planning + LangGraph state)    │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  AGENT 2: EMAIL ANALYSIS AGENT                                    │  │
│  │  ─────────────────────────────────                                │  │
│  │  Purpose:  Reads vendor email. Classifies intent, extracts        │  │
│  │            entities, scores urgency, reads sentiment, assigns     │  │
│  │            calibrated confidence score. Detects multi-issue       │  │
│  │            emails. The MOST IMPORTANT LLM task in the system.     │  │
│  │                                                                    │  │
│  │  Inputs:   Structured email payload, sender history from Memory   │  │
│  │  Outputs:  Analysis Object — primary/secondary intent, entities,  │  │
│  │            urgency, sentiment, confidence, multi-issue flag,      │  │
│  │            explanation markers                                    │  │
│  │  Type:     AI-driven (LLM NLP — fast model for classification,   │  │
│  │            stronger model for extraction)                         │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  AGENT 3: COMMUNICATION DRAFTING AGENT                            │  │
│  │  ─────────────────────────────────────                            │  │
│  │  Purpose:  Writes EVERY outbound email — acknowledgments,         │  │
│  │            clarification requests, status updates, escalation     │  │
│  │            notices, resolution messages, reopen acks, batch       │  │
│  │            vendor updates. Applies templates, maintains thread    │  │
│  │            continuity, personalizes tone, decides auto-send       │  │
│  │            vs. staged for review.                                 │  │
│  │                                                                    │  │
│  │  Inputs:   Communication brief (purpose, ticket context, vendor   │  │
│  │            profile, SLA details, tone policy, thread history)     │  │
│  │  Outputs:  Draft email package — recipients, subject, body,       │  │
│  │            thread headers, send mode (auto/draft), template ref   │  │
│  │  Type:     AI-driven (LLM generation + template constraints)     │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  AGENT 4: RESOLVER AGENT  [OPTIONAL — Phase 2]                    │  │
│  │  ─────────────────────────────────────────────                    │  │
│  │  Purpose:  When a resolver opens a ticket, assembles full         │  │
│  │            context package — vendor snapshot, ticket history,     │  │
│  │            similar past cases (vector search), KB articles,       │  │
│  │            draft response options. Can also run in "research      │  │
│  │            mode" for proactive pattern investigation.             │  │
│  │                                                                    │  │
│  │  Inputs:   Ticket ID, vendor ID, analysis summary, mode flag     │  │
│  │  Outputs:  Context package — vendor profile, related tickets,    │  │
│  │            similar past resolutions, KB articles, suggestions    │  │
│  │  Type:     AI-driven (vector search + LLM summarization)        │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Services (No AI Reasoning)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    SERVICES (DETERMINISTIC)                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌───────────────────────────────┐  ┌───────────────────────────────┐   │
│  │  EMAIL INGESTION SERVICE      │  │  VENDOR RESOLUTION SERVICE    │   │
│  │  ─────────────────────────    │  │  ──────────────────────────   │   │
│  │  • Connects to mailbox        │  │  • 3-step Salesforce cascade: │   │
│  │    (Exchange/Google)          │  │    Step 1: Email match        │   │
│  │  • Parses headers, body,      │  │    Step 2: Vendor ID match    │   │
│  │    attachments                │  │    Step 3: Fuzzy name match   │   │
│  │  • Detects new vs. reply      │  │  • Returns: vendor profile,   │   │
│  │    (thread ID matching)       │  │    tier, contracts, SLA terms │   │
│  │  • Filters spam, OOO          │  │  • Uses caching for repeated  │   │
│  │  • Emits structured payload   │  │    lookups                    │   │
│  │  Type: Rules-driven           │  │  Type: Rules + fuzzy matching │   │
│  └───────────────────────────────┘  └───────────────────────────────┘   │
│                                                                          │
│  ┌───────────────────────────────┐  ┌───────────────────────────────┐   │
│  │  TICKET OPERATIONS SERVICE    │  │  QUALITY & GOVERNANCE GATE    │   │
│  │  ──────────────────────────   │  │  ──────────────────────────   │   │
│  │  • All ServiceNow CRUD        │  │  Pre-send validation:        │   │
│  │  • Create, update, append     │  │  • CHECK 1: Ticket # matches? │   │
│  │  • State transitions          │  │  • CHECK 2: SLA wording OK?   │   │
│  │  • SLA clock start/pause/     │  │  • CHECK 3: Template comply?  │   │
│  │    resume                     │  │  • CHECK 4: PII exposure?     │   │
│  │  • Detect reopen conditions   │  │  • CHECK 5: Governance allows  │   │
│  │  • Idempotent (retry-safe)    │  │    auto-send?                 │   │
│  │  Type: Rules-driven API       │  │  Returns: PASS / FAIL /       │   │
│  │                               │  │    REVIEW-REQUIRED            │   │
│  └───────────────────────────────┘  │  Type: Rules + selective LLM  │   │
│                                      └───────────────────────────────┘   │
│  ┌───────────────────────────────┐                                      │
│  │  MEMORY & CONTEXT SERVICE     │                                      │
│  │  ──────────────────────────   │                                      │
│  │  4 Memory Tiers:              │                                      │
│  │  • Short-term: active         │                                      │
│  │    workflow context            │                                      │
│  │  • Long-term: vendor          │                                      │
│  │    interaction patterns        │                                      │
│  │  • Episodic: past resolutions │                                      │
│  │    for vector search           │                                      │
│  │  • Agent state: execution     │                                      │
│  │    traces                      │                                      │
│  │  Handles TTL, retention,      │                                      │
│  │  retrieval ranking             │                                      │
│  │  Type: Data/infrastructure    │                                      │
│  └───────────────────────────────┘                                      │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Infrastructure Components

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      INFRASTRUCTURE                                     │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌───────────────────────────────┐  ┌───────────────────────────────┐   │
│  │  MONITORING & SLA ALERTING    │  │  LLM GATEWAY SERVICE          │   │
│  │  ──────────────────────────   │  │  ──────────────────────────   │   │
│  │  • Tracks SLA timers          │  │  • Routes LLM calls to right  │   │
│  │  • Fires threshold alerts:    │  │    provider (Claude via       │   │
│  │    70% → warning to resolver  │  │    Bedrock primary, OpenAI    │   │
│  │    85% → L1 escalation alert  │  │    fallback)                  │   │
│  │    95% → L2 escalation alert  │  │  • Model selection by task:   │   │
│  │  • Queue depth, agent health  │  │    fast model → classification│   │
│  │  • Periodic batch analysis    │  │    strong model → drafting    │   │
│  │    for cross-vendor patterns  │  │  • Prompt versioning, token   │   │
│  │  • Triggers Orchestration     │  │    budgets, rate limiting,    │   │
│  │    Agent on actionable alerts │  │    failover, safety           │   │
│  └───────────────────────────────┘  └───────────────────────────────┘   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Interaction Map

```
                           ┌─────────────────┐
                           │  VENDOR EMAIL    │
                           │  (External)      │
                           └────────┬─────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                        EMAIL INGESTION SERVICE  [SVC]                            │
│   Parse headers → Detect new/reply → Extract attachments → Filter spam → Emit   │
└──────────────────────────────┬───────────────────────────────────────────────────┘
                               │  Structured Email Payload
                               ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                   WORKFLOW ORCHESTRATION AGENT  [AI]                             │
│                                                                                  │
│   ┌──────────┐   ┌───────────┐   ┌───────────┐   ┌──────────┐   ┌───────────┐  │
│   │ PLAN     │──►│ DISPATCH  │──►│ JOIN      │──►│ ROUTE    │──►│ REPLAN    │  │
│   │          │   │           │   │ PARALLEL  │   │ + DECIDE │   │ IF NEEDED │  │
│   └──────────┘   └───────────┘   └───────────┘   └──────────┘   └───────────┘  │
│                                                                                  │
│   Queries:  Memory Service (sender history, ticket context)                     │
│   Absorbs:  Routing rules (intent→team mapping) + Escalation logic             │
│                                                                                  │
│   Dispatches to:                                                                │
│   ┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐ │
│   │ Email Analysis │  │ Vendor Resol.  │  │ Ticket Ops     │  │ Comm Draft   │ │
│   │ Agent          │  │ Service        │  │ Service        │  │ Agent        │ │
│   └────────┬───────┘  └────────┬───────┘  └────────┬───────┘  └──────┬───────┘ │
│            │                   │                    │                  │         │
└────────────┼───────────────────┼────────────────────┼──────────────────┼─────────┘
             │                   │                    │                  │
             ▼                   ▼                    ▼                  ▼
┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌────────────────────┐
│ EMAIL ANALYSIS │  │ VENDOR RESOL.  │  │ TICKET OPS     │  │ COMMUNICATION      │
│ AGENT    [AI]  │  │ SERVICE  [SVC] │  │ SERVICE  [SVC] │  │ DRAFTING AGENT[AI] │
│                │  │                │  │                │  │                    │
│ • Intent       │  │ • 3-step       │  │ • ServiceNow   │  │ • Ack emails       │
│ • Entities     │  │   Salesforce   │  │   CRUD         │  │ • Escalation       │
│ • Urgency      │  │   cascade      │  │ • State        │  │   notices          │
│ • Sentiment    │  │ • Profile,     │  │   transitions  │  │ • Resolution       │
│ • Confidence   │  │   tier, SLA    │  │ • SLA clocks   │  │   emails           │
│ • Multi-issue  │  │ • Caching      │  │ • Idempotent   │  │ • Thread-safe      │
└────────┬───────┘  └────────────────┘  └────────────────┘  └─────────┬──────────┘
         │                                                             │
         │  Analysis Object                              Draft Email  │
         │  (with confidence)                            Package      │
         ▼                                                             ▼
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    QUALITY & GOVERNANCE GATE  [SVC]                               │
│   Ticket # match? │ SLA wording? │ Template? │ PII scan? │ Auto-send policy?    │
│   Result: PASS / FAIL / REVIEW-REQUIRED                                         │
└──────────────────────────────────┬───────────────────────────────────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
              ┌──────────┐  ┌──────────┐  ┌──────────────┐
              │ AUTO-SEND│  │ STAGE FOR│  │ REJECT       │
              │          │  │ HUMAN    │  │ (fix needed) │
              └──────────┘  │ REVIEW   │  └──────────────┘
                            └──────────┘

                    BACKGROUND SERVICES (always running)
┌──────────────────────────────────────────────────────────────────────────────────┐
│  ┌───────────────────────┐    ┌──────────────────────┐    ┌──────────────────┐  │
│  │ MONITORING & SLA      │    │ MEMORY & CONTEXT     │    │ RESOLVER AGENT   │  │
│  │ ALERTING SERVICE      │    │ SERVICE              │    │ (optional)       │  │
│  │                       │    │                      │    │                  │  │
│  │ Tracks SLA timers     │    │ Stores workflow      │    │ Assembles context│  │
│  │ Fires 70/85/95% alerts│    │ traces, vendor       │    │ packages for     │  │
│  │ Pattern detection     │    │ patterns, episodic   │    │ human resolvers  │  │
│  │                       │    │ memory, agent state  │    │                  │  │
│  └───────────────────────┘    └──────────────────────┘    └──────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Confidence Scoring — How It Works

### What Confidence Represents

The Email Analysis Agent produces a calibrated confidence score between **0.0 and 1.0**.
This score reflects the AI's certainty about its *overall* analysis — not just intent,
but the combination of intent classification, entity extraction, and email clarity.

### Factors That Drive the Score

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    CONFIDENCE SCORE CALCULATION                                  │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  FACTOR 1: INTENT CLARITY                                                       │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  HIGH: Single unambiguous intent                                          │  │
│  │    "Where is my payment for INV-90210?"                                   │  │
│  │    → intent: "payment_status_inquiry" → HIGH clarity                      │  │
│  │                                                                            │  │
│  │  LOW: Multiple competing intents                                          │  │
│  │    Email mixes payment complaint + contract reference                     │  │
│  │    → "payment_dispute" 52%  vs  "contract_inquiry" 38%                   │  │
│  │    → LOW clarity (no dominant intent)                                     │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  FACTOR 2: ENTITY EXTRACTION COMPLETENESS                                       │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  HIGH: All key identifiers found clearly                                  │  │
│  │    vendor_name: "Acme Corp"          ← exact match                       │  │
│  │    invoice_number: "INV-90210"       ← exact match                       │  │
│  │    amount: "$45,000"                 ← exact match                       │  │
│  │    date: "2026-02-28"               ← exact match                       │  │
│  │                                                                            │  │
│  │  LOW: Missing or fuzzy identifiers                                        │  │
│  │    vendor_name: "possibly Acme"      ← fuzzy/uncertain                   │  │
│  │    invoice_number: null              ← missing                           │  │
│  │    contract_ref: "partial match"     ← incomplete                        │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  FACTOR 3: AMBIGUITY AND MIXED SIGNALS                                          │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  HIGH confidence when:                                                    │  │
│  │    • Email body is focused on one topic                                   │  │
│  │    • Language is clear and direct                                         │  │
│  │    • No contradictory signals                                             │  │
│  │                                                                            │  │
│  │  LOW confidence when:                                                     │  │
│  │    • Email body mixes unrelated topics                                    │  │
│  │    • Vague or ambiguous language                                          │  │
│  │    • Contradictory signals (e.g., references both payment AND contract)   │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  FACTOR 4: SENDER HISTORY (from Memory)                                         │
│  ┌────────────────────────────────────────────────────────────────────────────┐  │
│  │  Known sender with consistent patterns   → can reinforce confidence       │  │
│  │  Unknown sender or erratic history       → does not help / may lower     │  │
│  └────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### The Threshold Mechanism

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  Email Analysis Agent                                                           │
│         │                                                                        │
│         │  Returns: confidence = 0.XX                                           │
│         ▼                                                                        │
│  ┌─────────────────────────────────┐                                            │
│  │ ORCHESTRATION AGENT             │                                            │
│  │ checks confidence vs threshold  │                                            │
│  │                                 │                                            │
│  │  THRESHOLD = 0.75               │                                            │
│  └────────────┬────────────────────┘                                            │
│               │                                                                  │
│       ┌───────┴────────┐                                                        │
│       │                │                                                        │
│       ▼                ▼                                                        │
│  ≥ 0.75              < 0.75                                                     │
│  HIGH CONFIDENCE     LOW CONFIDENCE                                             │
│       │                │                                                        │
│       ▼                ▼                                                        │
│  ┌─────────────┐  ┌─────────────────────────────────────────┐                   │
│  │ FULL AUTO   │  │ HUMAN TRIAGE PATH                       │                   │
│  │             │  │                                         │                   │
│  │ • Vendor    │  │ • System does NOT stop                   │                   │
│  │   resolution│  │ • Still gathers evidence in parallel:    │                   │
│  │ • Ticket    │  │   - Best-effort vendor match             │                   │
│  │   creation  │  │   - Similar past triage cases from       │                   │
│  │ • Routing   │  │     Memory                               │                   │
│  │ • Draft ack │  │ • Creates preliminary ticket in          │                   │
│  │ • Validate  │  │   "Pending Triage" state                 │                   │
│  │ • Auto-send │  │ • Assembles TRIAGE PACKAGE:              │                   │
│  │             │  │   - Original email                       │                   │
│  │             │  │   - AI analysis + uncertainty markers    │                   │
│  │             │  │   - Best-guess vendor match              │                   │
│  │             │  │   - Similar past cases                   │                   │
│  │             │  │   - Ambiguity explanation                │                   │
│  │             │  │ • Routes to human reviewer               │                   │
│  │             │  │ • Human confirms → corrections saved     │                   │
│  │             │  │   to Memory → flow resumes               │                   │
│  └─────────────┘  └─────────────────────────────────────────┘                   │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Where Else Confidence Is Used

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                    CONFIDENCE IMPACTS ACROSS THE SYSTEM                          │
├──────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  1. SEND MODE DECISION                                                          │
│     High confidence + routine category ─────────────► AUTO-SEND                 │
│     Low confidence OR policy-sensitive ─────────────► STAGED FOR HUMAN REVIEW   │
│                                                                                  │
│  2. REOPEN vs NEW TICKET (Flow 5)                                               │
│     High confidence on reopen recommendation ───────► System decides             │
│     Low confidence ─────────────────────────────────► Human triage with          │
│                                                        both options presented    │
│                                                                                  │
│  3. QUALITY GATE CHECK #5                                                       │
│     Governance policy considers confidence when ────► Deciding if auto-send     │
│     deciding auto-send eligibility                    is allowed                 │
│                                                                                  │
│  4. FEEDBACK LOOP (learning over time)                                          │
│     Human corrects AI → corrections saved to Memory                             │
│     → Email Analysis Agent accesses similar past ───► Produces higher           │
│       triage cases for future emails                  confidence over time       │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### Confidence — Concrete Examples

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                                                                                  │
│  EXAMPLE A — HIGH CONFIDENCE (0.91)                                             │
│  ──────────────────────────────────                                              │
│  Email: "We have not received payment for INV-90210..."                         │
│  Analysis:                                                                       │
│    primary_intent: "payment_status_inquiry"      ← single, clear intent         │
│    entities: vendor="Acme Corp", invoice="INV-90210", amount="$45,000"          │
│              date="2026-02-28"                   ← all entities extracted        │
│    urgency: 3/5                                                                  │
│    sentiment: "negative"                                                         │
│    confidence: 0.91                              ← ABOVE 0.75 → AUTO            │
│    multi_issue: false                                                            │
│    explanation: "Vendor asking about overdue payment for specific invoice"       │
│                                                                                  │
│  ─────────────────────────────────────────────────────────────────────────────   │
│                                                                                  │
│  EXAMPLE B — LOW CONFIDENCE (0.52)                                              │
│  ──────────────────────────────────                                              │
│  Email: Ambiguous email mixing payment complaint with contract reference         │
│         and incomplete identifiers                                               │
│  Analysis:                                                                       │
│    primary_intent: "payment_dispute" (52%)       ← no dominant intent           │
│    secondary_intent: "contract_inquiry" (38%)    ← competing intent             │
│    entities: vendor="possibly Acme" (fuzzy),                                    │
│              invoice=null, contract_ref="partial match"                          │
│                                                  ← missing/fuzzy entities       │
│    urgency: 4/5                                                                  │
│    sentiment: "angry"                                                            │
│    confidence: 0.52                              ← BELOW 0.75 → TRIAGE         │
│    explanation: "Email mixes payment and contract topics,                        │
│                  vendor identity uncertain, key IDs missing"                     │
│                                                                                  │
│  ─────────────────────────────────────────────────────────────────────────────   │
│                                                                                  │
│  EXAMPLE C — REOPEN CONFIDENCE (0.89)                                           │
│  ──────────────────────────────────────                                          │
│  Email: "The payment still has not arrived. I thought this was resolved."        │
│  Analysis:                                                                       │
│    reopen_recommendation: "reopen_original"                                     │
│    same_intent: true                             ← same issue category          │
│    same_entities: true (same INV-90210)          ← same identifiers             │
│    confidence: 0.89                              ← ABOVE 0.75 → AUTO REOPEN    │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Flow 1: New Vendor Email — Happy Path

```
========================================================================================
  FLOW 1: NEW VENDOR EMAIL — HAPPY PATH (HIGH CONFIDENCE)
========================================================================================

                         ┌──────────────────────────────┐
                         │     VENDOR SENDS EMAIL        │
                         │     to support@company.com    │
                         │     "Payment delay INV-90210" │
                         └──────────────┬───────────────┘
                                        │
                                        ▼
STEP 1 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  EMAIL INGESTION SERVICE  [SVC]                                                        │
│                                                                                        │
│  INPUT:  Raw email from mail server (Exchange / Google Workspace)                      │
│          Headers, body, attachments, transport metadata                                │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Extracts: sender (vendor@acme.com), subject, timestamp                           │
│    • Extracts thread ID (Message-ID, In-Reply-To, References headers)                 │
│    • Detects: NEW THREAD (no prior thread ID found)                                   │
│    • Processes attachments (virus scan, file type validation)                          │
│    • Filters out spam, out-of-office, auto-replies                                    │
│    • Detects language (English)                                                        │
│                                                                                        │
│  OUTPUT: Structured Email Payload                                                      │
│    {                                                                                   │
│      sender: "vendor@acme.com",                                                        │
│      subject: "Payment delay - Invoice INV-90210",                                     │
│      timestamp: "2026-03-19T10:15:00Z",                                                │
│      thread_id: "msg-abc-123" (new),                                                   │
│      body: "We have not received payment for INV-90210...",                             │
│      attachments: [invoice_copy.pdf],                                                  │
│      is_reply: false,                                                                  │
│      language: "en"                                                                    │
│    }                                                                                   │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 2 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  WORKFLOW ORCHESTRATION AGENT  [AI]                                                    │
│                                                                                        │
│  INPUT:  Structured Email Payload from Step 1                                          │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Queries MEMORY SERVICE for sender history                                         │
│      Result: vendor@acme.com has 3 prior tickets, vendor ID V-4455                    │
│    • Recognizes: new thread (not a reply)                                              │
│    • Builds EXECUTION PLAN:                                                            │
│      ┌────────────────────────────────────────────────────────────────┐                │
│      │  Plan = [                                                      │                │
│      │    1. Send to Email Analysis Agent                             │                │
│      │    2. Based on confidence, proceed or triage                   │                │
│      │    3. PARALLEL: Vendor Resolution + Ticket Lookup              │                │
│      │    4. Apply routing rules                                      │                │
│      │    5. Create ticket                                            │                │
│      │    6. Draft acknowledgment                                     │                │
│      │    7. Validate                                                 │                │
│      │    8. Send or stage for review                                 │                │
│      │  ]                                                             │                │
│      └────────────────────────────────────────────────────────────────┘                │
│                                                                                        │
│  OUTPUT: Execution plan created. Dispatches payload to Email Analysis Agent.           │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 3 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  EMAIL ANALYSIS AGENT  [AI — LLM Inference]                                            │
│                                                                                        │
│  INPUT:  Structured Email Payload + sender history from Memory                         │
│                                                                                        │
│  WHAT HAPPENS (LLM-powered):                                                          │
│    • INTENT CLASSIFICATION: "payment_status_inquiry" (primary)                        │
│      No secondary intent detected.                                                     │
│    • ENTITY EXTRACTION:                                                                │
│        vendor_name: "Acme Corp"                                                        │
│        invoice_number: "INV-90210"                                                     │
│        amount_mentioned: "$45,000"                                                     │
│        date_referenced: "2026-02-28" (payment due date)                               │
│    • URGENCY SCORE: 3 out of 5 (moderate — payment is overdue)                        │
│    • SENTIMENT: Negative (frustrated but professional tone)                            │
│    • CONFIDENCE: 0.91 (high — clear intent, clear entities)                           │
│    • MULTI-ISSUE FLAG: false (single issue)                                            │
│                                                                                        │
│  OUTPUT: Analysis Object                                                               │
│    {                                                                                   │
│      primary_intent: "payment_status_inquiry",                                         │
│      entities: { vendor: "Acme Corp", invoice: "INV-90210",                           │
│                  amount: "$45,000", date: "2026-02-28" },                              │
│      urgency: 3,                                                                       │
│      sentiment: "negative",                                                            │
│      confidence: 0.91,                                                                 │
│      multi_issue: false,                                                               │
│      explanation: "Vendor asking about overdue payment for specific invoice"           │
│    }                                                                                   │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 4 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  ORCHESTRATION AGENT — CONFIDENCE EVALUATION  [AI]                                     │
│                                                                                        │
│  INPUT:  Analysis Object (confidence = 0.91)                                           │
│                                                                                        │
│  DECISION:                                                                             │
│    ┌────────────────────────────────────────────────────────────────────────┐           │
│    │  confidence (0.91) >= threshold (0.75)                                 │           │
│    │  ═══════════════════════════════                                       │           │
│    │  RESULT: HIGH CONFIDENCE → proceed with full automation               │           │
│    │                                                                        │           │
│    │  (If it were below 0.75 → would route to HUMAN TRIAGE — see Flow 2)  │           │
│    └────────────────────────────────────────────────────────────────────────┘           │
│                                                                                        │
│  ACTION: Launches PARALLEL execution:                                                  │
│    Branch A: Vendor Resolution Service                                                 │
│    Branch B: Ticket Operations Service (existing ticket lookup)                        │
│                                                                                        │
└──────────────────────┬─────────────────────────────────────┬──────────────────────────┘
                       │                                     │
            ┌──────────┘                                     └──────────┐
            ▼                                                           ▼
STEP 5A (PARALLEL) ──────────────────────   STEP 5B (PARALLEL) ──────────────────────
┌───────────────────────────────────────┐   ┌───────────────────────────────────────┐
│  VENDOR RESOLUTION SERVICE  [SVC]     │   │  TICKET OPERATIONS SERVICE  [SVC]     │
│                                       │   │                                       │
│  INPUT: sender email, extracted       │   │  INPUT: thread_id, sender email       │
│         vendor name, invoice #        │   │                                       │
│                                       │   │  WHAT HAPPENS:                        │
│  WHAT HAPPENS:                        │   │    Searches ServiceNow for existing   │
│    Step 1: EMAIL MATCH                │   │    ticket matching this thread ID     │
│      vendor@acme.com → Salesforce     │   │    Result: NO existing ticket         │
│      Result: MATCH → Vendor V-4455    │   │    (this is a new thread)             │
│                                       │   │                                       │
│    Step 2: FETCH FULL PROFILE         │   │  OUTPUT:                              │
│      Status: Active                   │   │    { existing_ticket: null }          │
│      Tier: 2 (Standard)              │   │                                       │
│      Contract: Active until 2027      │   └───────────────────────────────────────┘
│      SLA Category: Standard (48hr)    │
│      Payment Terms: Net 30            │
│      Risk Signals: NONE              │
│      Match confidence: 0.99           │
│                                       │
│  OUTPUT: Vendor Match Object          │
│    { vendor_id: "V-4455",            │
│      name: "Acme Corp",             │
│      tier: 2, status: "Active",     │
│      sla_category: "Standard",      │
│      risk_flags: [] }                │
└───────────────────┬───────────────────┘
                    │                                     │
            ┌───────┘                                     └───────┐
            │                                                     │
            ▼                                                     ▼
STEP 6 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  ORCHESTRATION AGENT — JOIN RESULTS + APPLY ROUTING  [AI]                              │
│                                                                                        │
│  INPUT:  Vendor Match Object + Ticket Lookup Result + Analysis Object                  │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Joins parallel results                                                            │
│    • Confirms: new ticket needed (no existing ticket found)                            │
│    • Applies ROUTING RULES:                                                            │
│        Intent "payment_status_inquiry" → Finance Accounts Payable team                │
│    • Adjusts PRIORITY:                                                                 │
│        Base priority: Medium (payment inquiry)                                         │
│        Vendor tier 2 (Standard) → no upgrade                                          │
│        Urgency 3 + negative sentiment → upgrade to Medium-High                        │
│    • No escalation flags at this stage                                                 │
│                                                                                        │
│  OUTPUT: Routing Decision                                                              │
│    { resolver_group: "Finance-AP",                                                     │
│      priority: "Medium-High",                                                          │
│      escalation_flag: false,                                                           │
│      routing_rationale: "Payment inquiry routed to Finance-AP..." }                   │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 7 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  TICKET OPERATIONS SERVICE — CREATE SERVICENOW TICKET  [SVC]                           │
│                                                                                        │
│  INPUT: Analysis Object + Vendor Profile + Routing Decision                            │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    Creates new ServiceNow ticket with ALL enriched fields:                             │
│    ┌──────────────────────────────────────────────────────────────────┐                │
│    │  Ticket ID:       TKT-80301                                     │                │
│    │  Category:        Payment Inquiry                               │                │
│    │  Vendor:          Acme Corp (V-4455)                            │                │
│    │  Priority:        Medium-High                                   │                │
│    │  Assigned Group:  Finance-AP                                    │                │
│    │  SLA Policy:      Standard 48hr response                       │                │
│    │  SLA Deadline:    2026-03-21T10:15:00Z                          │                │
│    │  Description:     AI-generated summary of vendor email          │                │
│    │  Original Email:  attached/linked                               │                │
│    └──────────────────────────────────────────────────────────────────┘                │
│    • Starts SLA clock                                                                  │
│    • Writes audit entry: "Ticket created by VQMS automation"                          │
│    • Operation is IDEMPOTENT (retry-safe — no duplicates)                              │
│                                                                                        │
│  OUTPUT: Ticket Object                                                                 │
│    { ticket_id: "TKT-80301", state: "New",                                            │
│      sla_deadline: "2026-03-21T10:15:00Z", assigned_group: "Finance-AP" }             │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 8 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  COMMUNICATION DRAFTING AGENT — WRITE ACKNOWLEDGMENT  [AI — LLM Generation]            │
│                                                                                        │
│  INPUT: Communication Brief                                                            │
│    { purpose: "acknowledgment",                                                        │
│      ticket: "TKT-80301",                                                              │
│      vendor: "Acme Corp",                                                              │
│      sla_deadline: "2026-03-21T10:15:00Z",                                             │
│      tone_policy: "professional, empathetic",                                          │
│      thread_id: "msg-abc-123",                                                         │
│      audience: "vendor" }                                                              │
│                                                                                        │
│  WHAT HAPPENS (LLM-powered):                                                          │
│    • Selects acknowledgment template                                                   │
│    • Generates personalized email body:                                                │
│      ┌────────────────────────────────────────────────────────────────────┐            │
│      │  "Dear Acme Corp team,                                            │            │
│      │                                                                    │            │
│      │   Thank you for reaching out regarding invoice INV-90210.         │            │
│      │   We have registered your inquiry as TKT-80301 and our Finance    │            │
│      │   team is reviewing the payment status. You can expect an         │            │
│      │   update within 48 hours..."                                      │            │
│      └────────────────────────────────────────────────────────────────────┘            │
│    • Sets thread headers (In-Reply-To, References) for continuity                     │
│    • Determines send mode: AUTO (high confidence, routine query)                      │
│                                                                                        │
│  OUTPUT: Draft Email Package                                                           │
│    { to: "vendor@acme.com",                                                            │
│      subject: "Re: Payment delay - Invoice INV-90210 [TKT-80301]",                    │
│      body: "Dear Acme Corp team...",                                                   │
│      thread_headers: { in_reply_to: "msg-abc-123" },                                  │
│      send_mode: "auto",                                                                │
│      template_used: "ack_payment_inquiry_v3" }                                         │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 9 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  QUALITY & GOVERNANCE GATE — VALIDATION  [SVC + selective LLM]                         │
│                                                                                        │
│  INPUT: Draft Email Package + Ticket Object + Vendor Profile                           │
│                                                                                        │
│  5-POINT VALIDATION:                                                                   │
│    CHECK 1: Ticket # in email body matches real ticket?                               │
│             "TKT-80301" exists in ServiceNow ──────────────────────────► PASS          │
│    CHECK 2: SLA wording matches ticket SLA policy?                                    │
│             "48 hours" matches Standard SLA ───────────────────────────► PASS          │
│    CHECK 3: Template compliance?                                                       │
│             Uses approved template ack_payment_inquiry_v3 ────────────► PASS          │
│    CHECK 4: PII scan?                                                                  │
│             No SSN, no credit card, no personal addresses ────────────► PASS          │
│    CHECK 5: Governance policy allows auto-send?                                       │
│             Confidence 0.91, routine category, no risk flags ─────────► PASS          │
│                                                                                        │
│    OVERALL RESULT: ✓ PASS — approved for auto-send                                    │
│                                                                                        │
│  OUTPUT: { status: "pass", issues: [], audit_entry: "All 5 checks passed" }           │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 10 ─────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  EMAIL SENT TO VENDOR                                                                  │
│                                                                                        │
│  • Email dispatched via Email Server (Exchange / Google)                               │
│  • Thread headers ensure vendor's email client shows this as a reply                  │
│  • Delivery confirmation logged                                                        │
│                                                                                        │
│  OUTPUT: Email delivered to vendor@acme.com                                            │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 11 ─────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  POST-SEND: MONITORING + MEMORY + RESOLVER                                             │
│                                                                                        │
│  MONITORING & SLA ALERTING SERVICE:                                                    │
│    • Picks up TKT-80301                                                                │
│    • Sets alert thresholds:                                                            │
│        70% SLA consumed → warning to resolver                                         │
│        85% SLA consumed → L1 escalation alert to Orchestration Agent                  │
│        95% SLA consumed → L2 escalation alert                                         │
│                                                                                        │
│  MEMORY & CONTEXT SERVICE:                                                             │
│    • Stores complete workflow trace:                                                    │
│        Email payload, analysis result, vendor match, ticket created,                  │
│        routing decision, draft sent, validation result                                │
│    • Indexes for future retrieval:                                                     │
│        By vendor (V-4455), by ticket (TKT-80301), by intent type                     │
│    • Updates sender profile: vendor@acme.com now has 4 tickets                        │
│                                                                                        │
│  RESOLVER AGENT (if active — optional Phase 2):                                        │
│    • Prepares context package for Finance-AP resolver:                                 │
│        Vendor snapshot, 3 prior tickets, similar payment inquiries,                   │
│        KB articles on payment processing delays                                       │
│                                                                                        │
│  OUTPUT: Ticket in resolver queue, SLA tracking active, context ready                 │
└────────────────────────────────────────────────────────────────────────────────────────┘

                           ════════════════════════
                            >>> FLOW 1 COMPLETE <<<
                           ════════════════════════
              Vendor has acknowledgment. Ticket is in resolver queue.
                  SLA clock is running. Context package is ready.
```

---

## 6. Flow 2: Low Confidence — Human Triage Path

```
========================================================================================
  FLOW 2: LOW CONFIDENCE — HUMAN TRIAGE PATH
========================================================================================

STEPS 1-2 — SAME AS FLOW 1
(Email Ingestion Service → Orchestration Agent plans and dispatches)
                                         │
                                         ▼
STEP 3 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  EMAIL ANALYSIS AGENT — RETURNS LOW CONFIDENCE  [AI]                                   │
│                                                                                        │
│  INPUT: Structured email payload (ambiguous email mixing payment complaint             │
│         with contract reference and incomplete identifiers)                             │
│                                                                                        │
│  OUTPUT: Analysis Object                                                               │
│    {                                                                                   │
│      primary_intent: "payment_dispute" (52% confidence)  ← no dominant intent         │
│      secondary_intent: "contract_inquiry" (38% confidence) ← competing intent         │
│      entities: {                                                                       │
│        vendor_name: "possibly Acme" (fuzzy),              ← uncertain                 │
│        invoice: null,                                      ← MISSING                  │
│        contract_ref: "partial match"                       ← incomplete               │
│      },                                                                                │
│      urgency: 4,                                                                       │
│      sentiment: "angry",                                                               │
│      confidence: 0.52,                                     ← BELOW 0.75 THRESHOLD     │
│      explanation: "Email mixes payment and contract topics,                            │
│                    vendor identity uncertain, key IDs missing"                         │
│    }                                                                                   │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 4 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  ORCHESTRATION AGENT — CONFIDENCE BELOW THRESHOLD  [AI]                                │
│                                                                                        │
│  INPUT: Analysis Object (confidence = 0.52, below 0.75 threshold)                      │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Detects low confidence → switches to TRIAGE PLAN                                  │
│    • Does NOT stop. Still gathers evidence in PARALLEL:                                │
│        Branch A: Vendor Resolution Service (best-effort match)                        │
│        Branch B: Memory Service (similar past triage cases)                           │
│    • Creates preliminary ticket in "Pending Triage" state                              │
│    • Assembles TRIAGE PACKAGE for human reviewer:                                      │
│                                                                                        │
│  OUTPUT: Triage Package                                                                │
│    ┌──────────────────────────────────────────────────────────────────────────┐        │
│    │  original_email: [full email]                                            │        │
│    │  ai_analysis: [analysis object with uncertainty markers]                 │        │
│    │  vendor_match: { best_guess: "Acme Corp", confidence: 0.61 }            │        │
│    │  similar_past_cases: [3 similar triage cases from Memory]               │        │
│    │  preliminary_ticket: "TKT-80302 (Pending Triage)"                       │        │
│    │  ambiguity_explanation: "AI could not determine if this is a payment    │        │
│    │    dispute or contract inquiry. Vendor ID uncertain."                    │        │
│    └──────────────────────────────────────────────────────────────────────────┘        │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 5 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  HUMAN TRIAGE REVIEWER  [HUMAN]                                                        │
│                                                                                        │
│  INPUT: Triage Package (displayed in Agent Workspace Portal)                           │
│                                                                                        │
│  WHAT THE HUMAN DOES:                                                                  │
│    • Reads the original email                                                          │
│    • Reviews AI's best guess and uncertainty markers                                   │
│    • Decides: This is a payment dispute, not a contract inquiry                        │
│    • Confirms vendor: Yes, this is Acme Corp (V-4455)                                 │
│    • Assigns category: Payment Dispute                                                 │
│    • Clicks "Confirm and Resume"                                                       │
│                                                                                        │
│  OUTPUT: Human Corrections                                                             │
│    { confirmed_intent: "payment_dispute",                                              │
│      confirmed_vendor: "V-4455",                                                       │
│      assigned_category: "Payment Dispute",                                             │
│      corrected_by: "agent.jane@company.com",                                           │
│      correction_timestamp: "2026-03-19T10:32:00Z" }                                   │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 6 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  CORRECTIONS SAVED + ORCHESTRATION RESUMES  [AI + SVC]                                 │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Human corrections written to MEMORY SERVICE                                       │
│      (indexed so Email Analysis Agent LEARNS from this for similar future emails       │
│       — reduces future low-confidence cases)                                           │
│    • Orchestration Agent RESUMES from ROUTING onward                                   │
│      using human-confirmed data instead of AI guesses                                  │
│    • Ticket TKT-80302 updated from "Pending Triage" → "New"                           │
│    • Flow continues: Routing → Ticket Update → Ack Draft →                             │
│      Validation → Send (same as Flow 1, Steps 6-11)                                   │
│                                                                                        │
└────────────────────────────────────────────────────────────────────────────────────────┘

                           ════════════════════════
                            >>> FLOW 2 COMPLETE <<<
                           ════════════════════════
                Human corrected the AI. System learned from it.
                   Normal flow resumed with confirmed data.
```

---

## 7. Flow 3: Vendor Reply on Existing Thread

```
========================================================================================
  FLOW 3: VENDOR REPLY ON EXISTING THREAD
========================================================================================

STEP 1 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  EMAIL INGESTION SERVICE — DETECTS REPLY  [SVC]                                        │
│                                                                                        │
│  INPUT: New email from vendor@acme.com                                                 │
│         In-Reply-To header references thread "msg-abc-123"                             │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Detects: is_reply = true                                                          │
│    • Thread ID "msg-abc-123" matches known ticket TKT-80301                           │
│                                                                                        │
│  OUTPUT: Structured payload with is_reply: true, thread_id, ticket_ref                │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 2 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  ORCHESTRATION AGENT — RECOGNIZES EXISTING CONTEXT  [AI]                               │
│                                                                                        │
│  INPUT: Reply payload with ticket reference TKT-80301                                  │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Queries Memory: TKT-80301 is in-progress, assigned to Finance-AP,                │
│      vendor is Acme Corp (V-4455), tier 2                                             │
│    • Recognizes: NO NEED to re-run vendor resolution or routing                       │
│      (those facts haven't changed)                                                     │
│    • Builds ABBREVIATED PLAN:                                                          │
│      ┌──────────────────────────────────────────────────────┐                          │
│      │  1. Analyze new email content ONLY                   │                          │
│      │  2. Append to existing ticket                        │                          │
│      │  3. Resume SLA clock if on hold                      │                          │
│      │  4. Draft brief acknowledgment                       │                          │
│      │  5. Validate and send                                │                          │
│      └──────────────────────────────────────────────────────┘                          │
│                                                                                        │
│  OUTPUT: Abbreviated execution plan                                                    │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 3 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  EMAIL ANALYSIS AGENT — NEW CONTENT ONLY  [AI]                                         │
│                                                                                        │
│  INPUT: New email body only (not full thread history — that's in Memory)               │
│                                                                                        │
│  OUTPUT: { intent: "information_provided",                                             │
│            entities: { attachment: "payment_receipt.pdf" },                             │
│            confidence: 0.94,                                                           │
│            explanation: "Vendor is providing requested documents" }                    │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 4 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  TICKET UPDATE + SLA RESUME  [SVC]                                                     │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Ticket Operations Service appends reply to TKT-80301                              │
│    • If ticket was on hold (waiting for info): RESUMES SLA clock                      │
│    • Ticket state: "On Hold" → "In Progress"                                          │
│    • Audit entry: "Vendor reply received, SLA resumed"                                │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEPS 5-7 ───────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  DRAFT ACK → VALIDATE → SEND                                                          │
│                                                                                        │
│  Communication Drafting Agent: short acknowledgment                                    │
│    "Thank you for providing the payment receipt. Our Finance team                     │
│     has been notified and will review shortly."                                        │
│  Quality Gate: PASS                                                                    │
│  Email sent to vendor.                                                                 │
│  Memory updated.                                                                       │
└────────────────────────────────────────────────────────────────────────────────────────┘

                           ════════════════════════
                            >>> FLOW 3 COMPLETE <<<
                           ════════════════════════
                Reply appended. SLA resumed. Resolver notified.
```

---

## 8. Flow 4: SLA Breach Escalation

```
========================================================================================
  FLOW 4: SLA BREACH ESCALATION — PROACTIVE (NO INBOUND EMAIL)
========================================================================================

STEP 1 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  MONITORING SERVICE — DETECTS SLA RISK  [INFRA]                                        │
│                                                                                        │
│  INPUT: Continuous SLA timer data for all active tickets                               │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • TKT-80301 has consumed 85% of its 48hr SLA                                       │
│    • No resolution activity logged in 12 hours                                         │
│    • Threshold rule fires: 85% + 12hr inactivity = ALERT                              │
│                                                                                        │
│  OUTPUT: SLA Alert Event                                                               │
│    { ticket: "TKT-80301", sla_consumed: 85%,                                          │
│      inactive_hours: 12, alert_level: "L1" }                                          │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 2 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  ORCHESTRATION AGENT — BUILDS ESCALATION PLAN  [AI]                                    │
│                                                                                        │
│  INPUT: SLA Alert Event                                                                │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Queries Memory: ticket context, vendor tier, communication history                │
│    • Evaluates ESCALATION RULES:                                                       │
│      ┌─────────────────────────────────────────────────────────────────────┐           │
│      │  85% SLA consumed                                                   │           │
│      │  + Tier 2 vendor                                                    │           │
│      │  + 12hr inactive                                                    │           │
│      │  + Negative sentiment                                               │           │
│      │  ═══════════════════════                                            │           │
│      │  RESULT: L1 escalation to team lead                                 │           │
│      │                                                                     │           │
│      │  (If 95% → L2 escalation to manager)                               │           │
│      └─────────────────────────────────────────────────────────────────────┘           │
│    • Plan: draft escalation notice → validate → send to team lead                     │
│                                                                                        │
│  OUTPUT: Escalation dispatch to Communication Drafting Agent                           │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 3 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  COMMUNICATION DRAFTING AGENT — ESCALATION NOTICE  [AI]                                │
│                                                                                        │
│  INPUT: Escalation brief (internal audience, team lead)                                │
│                                                                                        │
│  OUTPUT: Internal escalation email                                                     │
│    ┌──────────────────────────────────────────────────────────────────────────┐        │
│    │  "TKT-80301 (Acme Corp, payment inquiry) is at 85% SLA with no         │        │
│    │   activity for 12 hours. Vendor sentiment is negative.                  │        │
│    │   Recommended action: immediate assignment or vendor update."           │        │
│    └──────────────────────────────────────────────────────────────────────────┘        │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEPS 4-5 ───────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  VALIDATE → SEND → UPDATE TICKET                                                       │
│                                                                                        │
│  • Quality Gate: checks accuracy → PASS                                               │
│  • Escalation notice sent to team lead                                                │
│  • Ticket Operations Service: marks TKT-80301 as escalated + timestamp                │
│  • Monitoring continues: if 95% hit → L2 escalation to manager                       │
└────────────────────────────────────────────────────────────────────────────────────────┘

                           ════════════════════════
                            >>> FLOW 4 COMPLETE <<<
                           ════════════════════════
              Team lead notified. Ticket marked escalated.
                    Monitoring continues watching.
```

---

## 9. Flow 5: Vendor Reopens After Closure

```
========================================================================================
  FLOW 5: VENDOR REPLIES AFTER TICKET CLOSURE — REOPEN FLOW
========================================================================================

STEP 1 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  EMAIL INGESTION SERVICE — DETECTS POST-CLOSURE REPLY  [SVC]                           │
│                                                                                        │
│  INPUT: New email from vendor@acme.com                                                 │
│         Thread ID maps to ticket TKT-80301                                             │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Detects: is_reply = true                                                          │
│    • Thread ID "msg-abc-123" → associated ticket TKT-80301                            │
│                                                                                        │
│  OUTPUT: Structured payload with is_reply: true, ticket_ref: TKT-80301                │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 2 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  ORCHESTRATION AGENT — DISCOVERS TICKET IS CLOSED  [AI]                                │
│                                                                                        │
│  INPUT: Reply payload with ticket reference TKT-80301                                  │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Queries Ticket Operations Service: TKT-80301 status?                              │
│      Result: CLOSED (resolved 2026-03-20, closure confirmed)                          │
│    • Queries Memory: full ticket history, resolution details                           │
│    • Recognizes: POST-CLOSURE REPLY → needs reopen evaluation                         │
│    • Dispatches Email Analysis Agent to analyze the new email                          │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 3 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  EMAIL ANALYSIS AGENT — ANALYZES POST-CLOSURE EMAIL  [AI]                              │
│                                                                                        │
│  INPUT: New email body + original ticket context from Memory                           │
│                                                                                        │
│  WHAT HAPPENS (LLM-powered):                                                          │
│    Determines what the vendor is saying:                                               │
│    ┌──────────────────────────────────────────────────────────────────────┐            │
│    │  Option A: "This wasn't fixed" → same issue → REOPEN               │            │
│    │  Option B: New unrelated question → new linked ticket               │            │
│    │  Option C: "Thank you" → no action needed                          │            │
│    └──────────────────────────────────────────────────────────────────────┘            │
│                                                                                        │
│    In this case: Vendor says "The payment still has not arrived.                       │
│    I thought this was resolved but the amount is still outstanding."                   │
│    Result: SAME ISSUE, NOT RESOLVED → recommend REOPEN                                │
│                                                                                        │
│  OUTPUT: Reopen Analysis                                                               │
│    { reopen_recommendation: "reopen_original",                                         │
│      reason: "Vendor reports original issue persists",                                 │
│      same_intent: true,                                                                │
│      same_entities: true (same invoice INV-90210),                                     │
│      confidence: 0.89 }                                                                │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 4 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  ORCHESTRATION AGENT — DECIDES: REOPEN vs NEW LINKED TICKET  [AI]                      │
│                                                                                        │
│  INPUT: Reopen Analysis from Email Analysis Agent                                      │
│                                                                                        │
│  DECISION LOGIC:                                                                       │
│    ┌──────────────────────────────────────────────────────────────────────────┐        │
│    │                                                                          │        │
│    │  reopen_recommendation = "reopen_original"                              │        │
│    │  same_intent = true  +  same_entities = true  +  confidence = 0.89     │        │
│    │  ═══════════════════════════════════════════════                         │        │
│    │  DECISION: REOPEN TKT-80301                                             │        │
│    │                                                                          │        │
│    │  OTHER POSSIBLE OUTCOMES:                                               │        │
│    │  • "different issue" or "new entities"                                  │        │
│    │     → CREATE LINKED NEW TICKET with "related_to: TKT-80301"           │        │
│    │  • Low confidence on recommendation                                     │        │
│    │     → ROUTE TO HUMAN TRIAGE with both options presented                │        │
│    │                                                                          │        │
│    └──────────────────────────────────────────────────────────────────────────┘        │
│                                                                                        │
│  OUTPUT: Reopen instruction to Ticket Operations Service                               │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 5 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  TICKET OPERATIONS SERVICE — REOPENS THE TICKET  [SVC]                                 │
│                                                                                        │
│  INPUT: Reopen instruction for TKT-80301                                               │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Transitions TKT-80301: "Closed" → "Reopened"                                     │
│    • Appends new email content to ticket                                               │
│    • Starts FRESH SLA clock (reopen policy: new 48hr window)                          │
│    • New SLA deadline: 2026-03-21T14:00:00Z                                            │
│    • Preserves ALL original ticket history and audit trail                             │
│    • Audit entry: "Ticket reopened — vendor reports issue persists."                   │
│    • Reopen counter incremented (quality metric)                                       │
│                                                                                        │
│  OUTPUT: Updated Ticket Object                                                         │
│    { ticket_id: "TKT-80301", state: "Reopened",                                       │
│      sla_deadline: "2026-03-21T14:00:00Z",                                             │
│      reopen_count: 1,                                                                  │
│      assigned_group: "Finance-AP" (preserved from original) }                         │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 6 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  ORCHESTRATION AGENT — CHECKS ROUTING  [AI]                                            │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Checks: Is the original resolver group still appropriate?                         │
│      Same intent (payment inquiry) + same team exists → YES                           │
│    • Re-assigns to Finance-AP with updated context                                     │
│    • (If original resolver had left the team or issue shifted category,                │
│       it would re-route to the correct group)                                          │
│                                                                                        │
│  OUTPUT: Routing confirmed (Finance-AP)                                                │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 7 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  COMMUNICATION DRAFTING AGENT — REOPEN ACKNOWLEDGMENT  [AI]                            │
│                                                                                        │
│  OUTPUT: Reopen acknowledgment email                                                   │
│    ┌──────────────────────────────────────────────────────────────────────────┐        │
│    │  "Dear Acme Corp team,                                                  │        │
│    │                                                                          │        │
│    │   We have received your follow-up regarding invoice INV-90210.          │        │
│    │   Your ticket TKT-80301 has been reopened and reassigned to our         │        │
│    │   Finance team for immediate review. We apologize for the               │        │
│    │   inconvenience and will provide an update within 48 hours."            │        │
│    └──────────────────────────────────────────────────────────────────────────┘        │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEPS 8-9 ───────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  VALIDATE → SEND                                                                       │
│                                                                                        │
│  • Quality Gate: checks ticket number, SLA, template, PII → PASS                     │
│  • Email sent to vendor                                                                │
│  • Monitoring picks up reopened ticket with new SLA deadline                           │
│  • Memory records: reopen event, reason, vendor's follow-up email,                    │
│    analysis results, decision taken (reopen vs. new ticket)                            │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 10 ─────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  QUALITY SIGNAL (background)                                                           │
│                                                                                        │
│  • Reopen event logged as a quality metric                                             │
│  • High reopen rates on specific issue types or resolver groups                       │
│    indicate a resolution quality problem                                               │
│  • Monitoring Service can aggregate reopen patterns and alert                         │
│    managers if a trend develops                                                        │
│  • Memory stores the reopen context so the Resolver Agent can                         │
│    flag "this issue was reopened once before" when the resolver                        │
│    picks it up again                                                                   │
└────────────────────────────────────────────────────────────────────────────────────────┘

                    KEY DESIGN RULES FOR REOPEN FLOW:
                    ─────────────────────────────────
                    • Never create a duplicate ticket for the same thread
                    • Always preserve original ticket history/audit trail
                    • Reopened ticket gets FRESH SLA clock (not remainder)
                    • Vendor always gets acknowledgment within SLA window
                    • If system can't decide (low confidence) → human triage
                    • Reopen events tracked as quality metrics

                           ════════════════════════
                            >>> FLOW 5 COMPLETE <<<
                           ════════════════════════
              Ticket reopened. Fresh SLA. Vendor acknowledged.
              No duplicate ticket. Full history preserved.
```

---

## 10. Flow 6: Resolution and Closure

```
========================================================================================
  FLOW 6: RESOLUTION AND CLOSURE
========================================================================================

STEP 1 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  RESOLVER MARKS ISSUE AS RESOLVED  [HUMAN]                                             │
│                                                                                        │
│  INPUT: Resolver in Finance-AP opens TKT-80301 in portal                               │
│         Reviews context package from Resolver Agent                                    │
│         Investigates and resolves the payment issue                                    │
│         Adds resolution notes: "Payment re-processed, ETA 3 days"                     │
│         Clicks "Mark as Resolved"                                                      │
│                                                                                        │
│  OUTPUT: Ticket state change request: "In Progress" → "Resolved"                      │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 2 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  ORCHESTRATION AGENT — HANDLES CLOSURE FLOW  [AI]                                      │
│                                                                                        │
│  WHAT HAPPENS:                                                                         │
│    • Receives ticket state change event                                                │
│    • Dispatches Communication Drafting Agent to write resolution email                 │
│    • Dispatches Ticket Operations Service to update state                              │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 3 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  COMMUNICATION DRAFTING AGENT — RESOLUTION EMAIL  [AI]                                 │
│                                                                                        │
│  OUTPUT: Resolution email                                                              │
│    ┌──────────────────────────────────────────────────────────────────────────┐        │
│    │  "Dear Acme Corp team,                                                  │        │
│    │                                                                          │        │
│    │   We are pleased to confirm that the payment for invoice INV-90210      │        │
│    │   has been re-processed. You should receive the payment within 3        │        │
│    │   business days. If you have any further questions, please reply         │        │
│    │   to this email. Ticket: TKT-80301."                                   │        │
│    └──────────────────────────────────────────────────────────────────────────┘        │
└────────────────────────────────────────┬───────────────────────────────────────────────┘
                                         │
                                         ▼
STEP 4 ──────────────────────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────────────────────────────────────────────┐
│  VALIDATE → SEND → WAIT FOR CONFIRMATION                                               │
│                                                                                        │
│  Quality Gate: PASS                                                                    │
│  Email sent to vendor.                                                                 │
│                                                                                        │
│  THREE POSSIBLE PATHS:                                                                 │
│                                                                                        │
│    ┌──────────────┐   ┌──────────────────┐   ┌────────────────────────┐               │
│    │   PATH A     │   │     PATH B       │   │      PATH C            │               │
│    │              │   │                  │   │                        │               │
│    │ Vendor says: │   │ No vendor reply  │   │ Vendor says:           │               │
│    │ "Thank you,  │   │ within policy    │   │ "This is NOT           │               │
│    │  confirmed." │   │ window (5 days)  │   │  resolved"             │               │
│    │              │   │                  │   │                        │               │
│    │      │       │   │       │          │   │         │              │               │
│    │      ▼       │   │       ▼          │   │         ▼              │               │
│    │ "Resolved"   │   │ Auto-closure     │   │ Triggers FLOW 5       │               │
│    │  → "Closed"  │   │ triggered        │   │ (Reopen Flow)         │               │
│    │              │   │ "Resolved"       │   │                        │               │
│    │ Audit summary│   │  → "Closed"      │   │                        │               │
│    │ generated    │   │                  │   │                        │               │
│    │              │   │ Audit summary    │   │                        │               │
│    │              │   │ generated        │   │                        │               │
│    └──────────────┘   └──────────────────┘   └────────────────────────┘               │
│                                                                                        │
│  AUDIT SUMMARY includes:                                                               │
│    • Full timeline from intake to closure                                              │
│    • All agent actions and decisions                                                   │
│    • Human interventions (if any)                                                      │
│    • SLA compliance status                                                             │
│    • Resolution details                                                                │
└────────────────────────────────────────────────────────────────────────────────────────┘

                           ════════════════════════
                            >>> FLOW 6 COMPLETE <<<
                           ════════════════════════
```

---

## 11. Human-in-the-Loop Checkpoints

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                     HUMAN-IN-THE-LOOP CHECKPOINTS                                    │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  CHECKPOINT 1: LOW-CONFIDENCE TRIAGE                                                │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │  Trigger:   Email Analysis Agent confidence < 0.75                            │  │
│  │  What:      Human reviews AI analysis, corrects classification,               │  │
│  │             confirms vendor, assigns category                                 │  │
│  │  Feedback:  Corrections feed back to Memory for learning                      │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  CHECKPOINT 2: UNMATCHED VENDOR REVIEW                                              │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │  Trigger:   Vendor Resolution Service fails all 3 match strategies            │  │
│  │  What:      Human manually identifies vendor, links to Salesforce,            │  │
│  │             or creates new vendor entry                                       │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  CHECKPOINT 3: HIGH-RISK EMAIL APPROVAL                                             │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │  Trigger:   Quality Gate flags draft as review-required                       │  │
│  │             (policy-sensitive, legal, high-value, negative sentiment)         │  │
│  │  What:      Human reviews drafted email, edits if needed,                     │  │
│  │             approves or rejects sending                                       │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  CHECKPOINT 4: ESCALATION APPROVAL (L2+)                                            │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │  Trigger:   Orchestration Agent determines L2+ escalation needed              │  │
│  │  What:      Human reviews escalation package, approves notification           │  │
│  │             to senior management                                              │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  CHECKPOINT 5: RESOLUTION REVIEW                                                    │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │  Trigger:   Complex or disputed resolutions before closure email              │  │
│  │  What:      Human confirms the resolution is correct and the                  │  │
│  │             closure email is appropriate                                      │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  CHECKPOINT 6: CORRECTION FEEDBACK                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────────┐  │
│  │  Trigger:   Human corrects AI classification or routing at any point          │  │
│  │  What:      Corrections feed back into Memory for future learning             │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 12. Technology Stack

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                          TECHNOLOGY STACK OPTIONS                                    │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  AGENT ORCHESTRATION                                                                │
│  ┌─────────────────────────────┐  ┌────────────────────────────────────────────┐    │
│  │  AWS Option                 │  │  Open Source Option                        │    │
│  │  AWS Step Functions +       │  │  LangGraph (MIT) — graph-based stateful   │    │
│  │  Bedrock Agents             │  │  multi-agent orchestration                │    │
│  │                             │  │  Temporal (MIT) — durable execution       │    │
│  │  AWS X-Ray + CloudWatch     │  │  Langfuse (MIT) — LLM observability      │    │
│  └─────────────────────────────┘  └────────────────────────────────────────────┘    │
│                                                                                      │
│  LLM PROVIDERS                                                                      │
│  ┌─────────────────────────────┐  ┌────────────────────────────────────────────┐    │
│  │  AWS: Amazon Bedrock        │  │  Open Source: Ollama + vLLM               │    │
│  │  (Claude primary,           │  │  LiteLLM (MIT) — unified LLM gateway     │    │
│  │   Titan fallback)           │  │  Langfuse — prompt management             │    │
│  └─────────────────────────────┘  └────────────────────────────────────────────┘    │
│                                                                                      │
│  DATA STORAGE                                                                       │
│  ┌─────────────────────────────┐  ┌────────────────────────────────────────────┐    │
│  │  AWS: ElastiCache (Redis)   │  │  Open Source: Valkey (Redis fork)         │    │
│  │  DynamoDB, OpenSearch,      │  │  PostgreSQL + pgvector                    │    │
│  │  Aurora PostgreSQL, S3      │  │  OpenSearch / Qdrant / Milvus, MinIO     │    │
│  └─────────────────────────────┘  └────────────────────────────────────────────┘    │
│                                                                                      │
│  MESSAGING                                                                          │
│  ┌─────────────────────────────┐  ┌────────────────────────────────────────────┐    │
│  │  AWS: SQS FIFO +           │  │  Open Source: RabbitMQ or Kafka           │    │
│  │  EventBridge               │  │  NATS for lightweight messaging           │    │
│  └─────────────────────────────┘  └────────────────────────────────────────────┘    │
│                                                                                      │
│  OBSERVABILITY                                                                      │
│  ┌─────────────────────────────┐  ┌────────────────────────────────────────────┐    │
│  │  AWS: CloudWatch,           │  │  Open Source: Prometheus + Grafana        │    │
│  │  X-Ray, SNS                 │  │  OpenTelemetry + Jaeger, Loki            │    │
│  └─────────────────────────────┘  └────────────────────────────────────────────┘    │
│                                                                                      │
│  SECURITY                                                                           │
│  ┌─────────────────────────────┐  ┌────────────────────────────────────────────┐    │
│  │  AWS: IAM + Cognito,        │  │  Open Source: Keycloak (identity)         │    │
│  │  Secrets Manager, KMS,      │  │  HashiCorp Vault (secrets)               │    │
│  │  Comprehend (PII),          │  │  Presidio (PII by Microsoft)             │    │
│  │  API Gateway + WAF          │  │  Kong (API gateway), ModSecurity (WAF)   │    │
│  └─────────────────────────────┘  └────────────────────────────────────────────┘    │
│                                                                                      │
│  COMPUTE & DEPLOYMENT                                                               │
│  ┌─────────────────────────────┐  ┌────────────────────────────────────────────┐    │
│  │  AWS: ECS Fargate + CDK     │  │  Open Source: Kubernetes + Docker         │    │
│  │  CodePipeline               │  │  OpenTofu / Pulumi, GitLab CI            │    │
│  └─────────────────────────────┘  └────────────────────────────────────────────┘    │
│                                                                                      │
│  RECOMMENDED STACK (minimize vendor lock-in):                                       │
│  LangGraph + Temporal | LiteLLM → Claude API | PostgreSQL + pgvector + Valkey       │
│  + MinIO + OpenSearch | RabbitMQ/Kafka | Prometheus + Grafana + Loki + Langfuse     │
│  | Keycloak + Vault + Presidio + Kong | Kubernetes + Docker + OpenTofu              │
│                                                                                      │
│  RECOMMENDED STACK (fast AWS deployment):                                           │
│  LangGraph on ECS + Step Functions | Bedrock (Claude) | DynamoDB + Aurora +         │
│  OpenSearch + ElastiCache + S3 | SQS + EventBridge | CloudWatch + X-Ray +           │
│  Grafana | IAM + Cognito + Secrets Manager + KMS + Comprehend + API Gateway         │
│  | ECS Fargate + CDK                                                                │
│                                                                                      │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

*End of VQMS Complete Architecture & End-to-End Flow Reference.*  
*Original: 12 agents. Recommended: 3 core agents + 1 optional + 5 services.*  
*All 6 flows covered: New email, low confidence, reply, SLA escalation, reopen, and closure.*  
*Confidence scoring explained: factors, threshold mechanism, downstream impacts, feedback loop.*
