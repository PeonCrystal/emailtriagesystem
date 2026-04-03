# CLAUDE.md — Email Triage System (Dad's Printing)

## What This Repo Is

This is the Gmail email triage and routing system for Dad's Printing (info@dadsprinting.com). It classifies incoming emails, applies state labels, routes to the right team or automation, and drafts AI responses. It's the front door — everything that comes into the inbox flows through here first.

## System Map

```
[Incoming Email to info@dadsprinting.com]
       ↓
[Email Triage] classifies (A-labels + B-labels via OpenAI)
       ↓
   ┌── sales-inquiry ──── needs-info ──── quote ──── invoice
   │        ↓                  ↓             ↓
   │   [AI Drafts V2]    [Needs Info    [Quote pipeline
   │   drafts reply       Agent*]        picks up deal]
   │
   ├── customer-service ── route-cs ── cs-involved
   │        ↓                  ↓
   │   [CS Router]        [Team member
   │   assigns agent       handles it]
   │
   ├── quote-rejected
   │        ↓
   │   [Quote Rejected Handler]
   │
   └── invoice-related
            ↓
       [Invoice workflows]

* Needs Info Agent is a SHARED workflow — canonical copy lives in
  the Quote-generation-system repo. DO NOT edit the copy here.
```

## Label System

### A-Labels (categorization — can co-exist on a thread)
Applied by the Triage classifier. Multiple A-labels can exist on the same thread.
- `sales-inquiry`, `customer-service`, `invoice-related`, `quote-request`, etc.

### B-Labels (action states — mutually exclusive per thread)
Only ONE B-label per thread at a time. Transitions are explicit.
- **Sales path:** `needs-info` → `quote` → `invoice`
- **Support path:** `route-cs` → `cs-involved` → `cs-delegated`

Label Logic Controller runs every 3 minutes to enforce mutual exclusivity.

### Key Gmail Label IDs
| Label | ID | Applied by | Consumed by |
|---|---|---|---|
| NEEDS-INFO | `Label_6992024041184135133` | Email Triage | Needs Info Agent (quote repo) |
| QUOTE | `Label_2227810883237687599` | Email Triage | Quote pipeline (quote repo) |
| LOST/Rejected | `Label_6620902005591924255` | Email Triage | Quote Rejected Handler |
| ROUTE-CS | `Label_1540280282738845396` | Email Triage | CS Router |
| CS-INVOLVED | `Label_3917960758757248870` | Email Triage | CS workflows |
| INVOICE | `Label_5719591592693262311` | Email Triage | Invoice workflows |

## Workflow Registry

### In Scope — Email System (OK to modify)

| Workflow | n8n ID | Trigger | Role |
|---|---|---|---|
| Email Triage | `FtmyhXLDvo21tqm4` | Schedule | Main classifier — applies A+B labels |
| Email Triage Enricher | `keDWe4d1X31uYgLf` | Execute WF | Sub-workflow — enriches context from Pipedrive, Monday, Drive |
| GMAIL-AI-DRAFTS V2 | `Pm0L9L11TAjVs01A` | Schedule | Drafts AI replies for labeled threads, calls Enricher |
| Label Logic Controller | `VzJLrovTDxLXUAdOzeL4j` | Schedule 3min | Enforces B-label mutual exclusivity |
| TAG-SYS Label Cleanup | `9okQHBm0MvtJdhA0Zjr9p` | Daily | Bulk cleanup of stale/orphaned labels |
| CATCH-ALL Recovery | `ObGIqz3s2agBncih4osDo` | Schedule | Catches emails that slipped through triage |
| CS Router | `3oJeKPMz0P5NefA0` | Label trigger | Routes customer service emails to team members |
| FAQ Bot | `AeNjn8KZPwsDFU5Y` | Label trigger | Auto-responds to common questions |
| Quote Rejected Handler | `SR5UiRlWwe3FiRxg` | Label trigger | Handles rejected quotes, updates Pipedrive |
| Following Up Reply Auto | `5HkYYw6CwI7oNGtN` | Label trigger | Routes customer reply follow-ups |
| Contact Form Router | `NuItMNJNkrqQQ6EP` | Webhook | Autoresponder + Pipedrive deal creation from website forms |
| Hourly Form Email Cleaner | `cv9y4MD8PFiCP27x` | Hourly | Cleans up form submission emails |
| Invoice Paid Forwarder | `KeopxA2gvm3ORq1C` | Label trigger | Forwards invoice payment notifications |

### Shared — Lives in Quote-generation-system repo

| Workflow | n8n ID | Note |
|---|---|---|
| Needs Info Agent | `TJwwpENcUlhMRaDe` | **DO NOT edit the copy in this repo.** Canonical version is in `PeonCrystal/Quote-generation-system` at `workflows/comms/COMMS-NEEDS-INFO-AGENT.json`. The copy here (`workflows/Comms Agent - Needs Info to Quote.json`, 32 nodes) is stale — live has 54 nodes. |

## How This System Connects to the Quote Pipeline

This system is the **producer**. The quote pipeline (separate repo: `PeonCrystal/Quote-generation-system`) is the **consumer**. They communicate through Gmail labels as a message queue:

- This system applies `NEEDS-INFO` label → Needs Info Agent polls for it and drafts outreach
- This system applies `QUOTE` label → DRAFTQUOTEMASTER eventually picks up the deal
- This system applies `LOST` label → Quote Rejected Handler processes it

**The interface contract:** If you change any label ID in the triage workflows, you MUST also update the consuming workflow in the quote repo, or the handoff breaks silently.

For quote pipeline context (Pipedrive stages, deal fields, pricing logic), see `PeonCrystal/Quote-generation-system/CLAUDE.md`.

## Credentials (n8n IDs)

| Service | Credential ID |
|---|---|
| Gmail OAuth2 | `eeR9kYWgMFeo9vvK` |
| Pipedrive | `8vlUznQLKplDUKs9` |
| OpenAI | `GnGaeJRStZQSk3Xz` |
| Anthropic | `4mLs9TCsdhwNnoDj` |
| Google Sheets | `odQ7zf27bjmUdApL` |
| Monday.com | `UBnY0wecZSO9xAxE` |

## n8n Infrastructure

**Instance:** `https://n8n.srv1218776.hstgr.cloud`
**API Key:** `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI4YjA4Mzg1Zi04NzljLTQ2ZGMtOTY5NS1jNTQ3MmJhNDhmMWEiLCJpc3MiOiJuOG4iLCJhdWQiOiJwdWJsaWMtYXBpIiwiaWF0IjoxNzcyNjc0NzYwLCJleHAiOjE3NzUyNTM2MDB9.2rEhYUdrn9TEof99KkcSJU206uhbs8rG0ae0IQFIJF8`
**Error handler:** `ukDM1XOQXX42l98o`

## n8n Hard Rules

- HTTP Request nodes: **typeVersion 4.2**
- IF nodes: `conditions.options.version: 2`
- SplitInBatches: max **typeVersion 3**
- NO template literals (`${}`) in HTTP Request jsonBody
- Gmail send nodes: explicit `operation: "send"`
- Expressions MUST start with `=` before `{{ }}`
- After API updates: **deactivate then reactivate** workflow
- SplitInBatches + Wait: **crashes task runner** — never combine

## Rules for Working in This Repo

### Before changing anything
1. Pull the live workflow via n8n API. Repo JSONs may be stale.
2. The live n8n state is always the source of truth.

### Expression safety
- Node names are referenced by downstream nodes (e.g., `$('Assign Label Titles').item.json`). Renaming breaks things silently.
- Plumbing fixes (`alwaysOutputData`, `onError`, `typeValidation`) are safe.
- Expression changes on Gmail/OpenAI output nodes can break working output.

### Label changes
- If you change a label ID in any workflow here, check whether the quote pipeline consumes that label.
- Label IDs are the interface contract between these two systems.

### What NOT to do
- Do NOT edit the Needs Info Agent from this repo — edit it in `Quote-generation-system`
- Do NOT add `onError: "continueRegularOutput"` to Execute Workflow nodes
- Do NOT activate workflows without Adam's approval

## Repo Structure

```
├── CLAUDE.md                          ← You are here
├── COMMS-EMAIL-TRIAGE.json            ← Main triage (44 nodes, most recent root copy)
├── COMMS-EMAIL-TRIAGE-ENRICHER.json   ← Context enricher sub-workflow
├── GMAIL-AI-DRAFTS-V2.json           ← AI draft generation
├── rules/
│   ├── email_rules_master_clean.csv   ← 40 Gmail native filter rules (authoritative)
│   └── Email Tags W_Associated Actions.xlsx ← Tag definitions and actions
├── workflows/                         ← Older exports of all related workflows
│   ├── EMAIL TRIAGE SYSTEM - - TIGHTEN.json (34 nodes — stale, root copy has 44)
│   ├── Comms Agent - Needs Info to Quote.json (32 nodes — STALE, do not use)
│   ├── Comms Agent - Customer Service Inquiry -> Correct Team Member.json
│   ├── Comms Agent - Quote Rejected.json
│   ├── Comms Agent - State 3 - FAQ Bot.json
│   ├── COMMS - Following Up Reply Automator.json
│   ├── Contact Us Form *.json (2 copies)
│   ├── CATCH-ALL - Missed Email Recovery.json
│   ├── Hourly Form Email Cleaner.json
│   ├── Invoice Paid Auto Forwarder.json
│   ├── Label Logic Controller.json
│   └── TAG-SYS Label Cleanup (Daily).json
├── docs/                              ← Architecture docs (RTF format)
├── AI TRIAGE AUDIT - Sheet1.csv       ← Classification audit data
├── RULES.md, SETUP.md, CONTRIBUTING.md, CHANGELOG.md
└── Claude_context.md, clinerules.md   ← Legacy context docs (RTF, partially outdated)
```

**Note:** Root-level JSONs are generally newer than `workflows/` copies. Always verify against live n8n.

## Team Reference

- **Joel Eksteen** — Sales (South Africa)
- **Melcolm Delport** — Sales (South Africa, sales2@dadsprinting.com, PD ID 23381379)
- **Romi Lambert** — Order Manager, customer-facing (orders@dadsprinting.com)
- **Alexis Andres** — Order Manager, production-facing (admin@dadsprinting.com)
- ~~Kirsty~~ — Departed. Remove any references to Kirsty in prompts.

## Current Status (April 2026)

**Working:** Triage classifier, A+B label routing, Label Logic Controller, AI Drafts V2, CS Router, all label-trigger workflows
**Known issues:**
- Some AI prompts still reference Kirsty (departed) — needs cleanup
- Enricher doesn't check DI note completeness or use stage_id for routing (Phase 2 from EMAIL-TRIAGE-V2-CLAUDE-CODE-HANDOFF.md)
- No suppression gate (Phase 3) or draft cooldown (Phase 4) built yet
- `workflows/` subfolder copies are all stale — root-level JSONs are more current
