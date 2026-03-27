# COMMS- System Architecture

## High-Level Diagram
```
┌─────────────────────────────────────────────────────────────────┐
│                          Gmail Inbox                              ││   (dadsprinting.com + proteamjerseys.com emails)                 │└────────────────┬────────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│               Gmail Native Filters (Initial Tagging)              ││    Apply workspace labels (dadsprinting/proteamjerseys)         ││    Apply initial categorization hints                            │└────────────────┬────────────────────────────────────────────────┘
                 │
                 │  (30 second delay to avoid race conditions)
                 │
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                     n8n: Email Triage System                      ││┌──────────────────────────────────────────────────────────┐│
││  1. Fetch new/unprocessed emails                          ││││  2. Send to OpenAI for classification                     ││││  3. Parse confidence scores (A-labels + B-labels)         ││││  4. Apply A-labels (categorization)                       ││││  5. Apply B-label (action state) - if confident           ││││  6. Mark as processed                                     │││└──────────────────────────────────────────────────────────┘│
└────────────────┬────────────────────────────────────────────────┘
                 │
                 ├─────────────────────────────────────────────────┐
                 ││
                 ▼▼
┌──────────────────────────────┐┌──────────────────────────────┐
│    Sales-Specific Workflows   ││Support-Specific Workflows  ││    Quote Generator            ││ CS Router                 ││    Invoice Tracker            ││ CS Delegation Logic       ││    Follow-up Automator        ││ Customer Response Bot     │└──────────────┬────────────────┘└────────────┬─────────────────┘
               ││
               └─────────────────┬───────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│             n8n: Label Logic Controller (Every 3 min)             ││    Enforce B-label mutual exclusivity                           ││    Remove conflicts (keep most recent)                          ││    Clean orphaned labels                                        ││    Log changes to audit sheet                                   │└─────────────────────────────────────────────────────────────────┘
```

## Component Details

### Gmail Native Filters
- **Purpose:** Fast initial categorization
- **Language:** Gmail search operators
- **Limitations:** No AI, pattern-matching only
- **Integration:** Runs independently, n8n reads results

### n8n Workflows
- **Deployment:** Self-hosted or cloud n8n instance
- **Trigger Types:**
- Polling (check every X minutes)
- Webhook (instant, if configured)
- Manual (testing)
- **Error Handling:** Retry logic + error notifications

### OpenAI Classification
- **Model:** GPT-4 (or specified model)
- **Prompt Structure:**
```
  Email Subject: [subject]
  Email Body: [body]
  Email Thread History: [previous messages]

  Classify this email:
  1. A-Labels (categorization): [options]
  2. B-Label (action state): [options]

  Return JSON with confidence scores.
```
- **Rate Limits:** Monitor token usage
- **Fallback:** If API fails, default to "needs-review" label

### Gmail API Integration
- **Authentication:** OAuth 2.0
- **Permissions Required:**
- Read email metadata
- Modify labels
- Send email (for autoresponders)
- **Rate Limits:** 250 quota units/second/user

### Pipedrive Integration
- **Purpose:** Sync sales states with CRM
- **Trigger:** B-label changes (quote, invoice)
- **Data Flow:**
- New `quote` label →  Create Pipedrive deal
- `invoice` label →  Update deal stage
- Deal closed →  Remove B-label

---

## State Machine Diagram

### Sales Thread Lifecycle
```
[New Email]
    │
    ▼
needs-info ────────────────┐
    ││
    │  (info received)      │(customer unresponsive)
    ▼│
  quote ───────────────────┤
    ││
    │  (quote accepted)     │▼│
 invoice ──────────────────┤
    ││
    │  (payment received)   │▼▼
 [Resolved]           [Archived]
```

### Customer Service Thread Lifecycle
```
[New CS Email]
    │
    ▼
 route-cs ─────────────────┐
    ││
    │  (assigned)           │(auto-resolved)
    ▼│
cs-involved ───────────────┤
    ││
    │  (delegated)          │▼│
cs-delegated ──────────────┤
    ││
    │  (resolved)           │▼▼
 [Resolved]           [Archived]
```

---

## Data Flow Examples

### Example 1: New Sales Inquiry

1. **Email arrives:** "Can I get a quote for 500 printed t-shirts?"
2. **Gmail filter:** Applies `workspace:dadsprinting`
3. **n8n (30s later):** Fetches email
4. **OpenAI:** Classifies as `A: sales-inquiry, quote-request` (95% confidence), `B: needs-info` (80% confidence)
5. **n8n:** Applies `sales-inquiry`, `quote-request`, `needs-info` labels
6. **Quote workflow:** Triggers, sends template "Need details: sizes, colors, design"
7. **Label Logic Controller:** Verifies no other B-labels present

### Example 2: B-Label Conflict

1. **Email thread:** Already has `needs-info` label
2. **New reply arrives:** "Here's my design file"
3. **n8n:** OpenAI suggests `quote` label (90% confidence)
4. **B-label logic:** Detects conflict (`needs-info` + `quote`)
5. **Resolution:** Removes `needs-info`, applies `quote`
6. **Audit log:** Records transition in tracking sheet

---

## Scaling Considerations

### Current Limitations
- **Volume:** Handles ~500 emails/day comfortably
- **API Costs:** OpenAI classification ~$0.02/email
- **n8n Performance:** Polling every 3 min (Label Logic Controller)

### Optimization Paths
1. **Reduce OpenAI calls:** Cache classifications for similar emails
2. **Faster processing:** Move from polling to webhooks
3. **Parallel workflows:** Process sales/support independently
4. **Smarter filtering:** Train model on historical classifications

### Breaking Points (When to Refactor)
- **>1000 emails/day:** Need dedicated classification service
- **>10 workflows:** Consolidate into modular sub-workflows
- **>50 labels:** Implement label hierarchy/namespacing
- **API failures >5%:** Add retry queues + fallback logic

---

**Last Updated:** [Today's Date]
