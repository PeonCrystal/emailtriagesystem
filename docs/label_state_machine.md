# Label State Machine Reference

## B-Label State Machines

### Sales States (Mutually Exclusive)
```
┌──────────────┐
│   needs-info  │Initial state when customer inquiry lacks details
└──────┬───────┘
       │
       │  TRIGGER: Customer provides required info
       │  ACTION: Auto-respond with quote or flag for manual quote
       │
       ▼
┌──────────────┐
│     quote     │Quote sent, awaiting customer decision
└──────┬───────┘
       │
       │  TRIGGER: Customer accepts quote
       │  ACTION: Generate invoice, send to customer
       │
       ▼
┌──────────────┐
│    invoice    │Invoice sent, awaiting payment
└──────┬───────┘
       │
       │  TRIGGER: Payment received (manual mark or Pipedrive sync)
       │  ACTION: Remove B-label, archive thread
       │
       ▼
┌──────────────┐
│   [RESOLVED]  │└──────────────┘

REVERSE TRANSITIONS (Edge Cases):
- quote →  needs-info: Customer asks for modifications
- invoice →  quote: Invoice disputed, need to re-quote
```

### Customer Service States (Mutually Exclusive)
```
┌──────────────┐
│    route-cs   │Needs CS team assignment
└──────┬───────┘
       │
       │  TRIGGER: CS team member claims ticket
       │  ACTION: Assign in Pipedrive, notify team member
       │
       ▼
┌──────────────┐
│  cs-involved  │CS team actively working on issue
└──────┬───────┘
       │
       │  TRIGGER: Requires specialist or external help
       │  ACTION: Delegate to specific person/vendor
       │
       ▼
┌──────────────┐
│  cs-delegated │Delegated to specific team member
└──────┬───────┘
       │
       │  TRIGGER: Issue resolved
       │  ACTION: Remove B-label, send satisfaction survey
       │
       ▼
┌──────────────┐
│   [RESOLVED]  │└──────────────┘

ESCALATION PATH:
- route-cs →  cs-delegated (skip cs-involved if urgent)
```

---

## OPS Sub-Label Handling (New — Automation Lanes)

OPS emails previously had no B-labels — they were classification-only. Now, OPS has two sub-labels that act as **handling lanes**, driving specific automation behavior without touching the core Sales/CS state machines.

### Why Sub-Labels (Not Sub-Scenarios)

The alternative was to trigger different n8n scenarios when `TAG-SYS/OPS` gets applied, using code nodes to inspect the sender/subject and branch. The problem: that logic lives buried in workflow JSON, is invisible from Gmail, and scales poorly (every new OPS email type = another code branch).

**Sub-labels keep it clean because:**
1. **Visible in Gmail** — you can see `OPS/ALERT` and `OPS/DIGEST` in the sidebar, filter by them, and know exactly what handling an email is getting
2. **Gmail filters do the routing** — a new sender that fits the ALERT pattern just needs one Gmail filter rule, no n8n changes
3. **n8n workflows stay generic** — one workflow handles ALL `OPS/ALERT` emails the same way (forward + delayed archive), another handles ALL `OPS/DIGEST` emails (24h retention + 6 AM archive)
4. **Won't get messy** — there are only 2-3 handling patterns for OPS (alert, digest, and plain/unhandled). New senders map to existing patterns. You're not creating a sub-label per sender — you're creating a sub-label per **behavior**.

### Keeping It From Getting Messy — The Rule

> **Sub-labels represent handling patterns, not senders.**

- `TAG-SYS/OPS/ALERT` = "forward somewhere + archive after delay" (n8n errors, future: server alerts, monitoring)
- `TAG-SYS/OPS/DIGEST` = "keep latest in inbox, archive at 6 AM" (daily breakdowns, future: weekly reports)
- `TAG-SYS/OPS` (no sub-label) = "classify only, no automation" (vendor chatter, staff emails, general ops)

If a new email type doesn't fit ALERT or DIGEST, it stays plain OPS. Only add a new sub-label when you have a genuinely **new handling pattern** — not a new sender.

### OPS/ALERT State Machine
```
┌──────────────────┐
│   [Email Arrives]  │
└────────┬─────────┘
         │
         │  Gmail filter applies TAG-SYS/OPS/ALERT
         │  Gmail filter forwards to designated recipient
         │
         ▼
┌──────────────────┐
│   IN INBOX        │  Stays visible for 15 minutes minimum
│   (forwarded)     │
└────────┬─────────┘
         │
         │  n8n: OPS Alert Handler (runs every 5 min)
         │  Checks: is email >15 min old AND has OPS/ALERT label?
         │
         ▼
┌──────────────────┐
│   [ARCHIVED]      │  Removed from inbox, label preserved
└──────────────────┘
```

**Current ALERT senders:**
- n8n error notifications → forward to nsmajumder@gmail.com

### OPS/DIGEST State Machine
```
┌──────────────────┐
│   [Email Arrives]  │  e.g., daily breakdown
└────────┬─────────┘
         │
         │  Gmail filter applies TAG-SYS/OPS/DIGEST
         │  Email stays UNREAD and UNTOUCHED
         │
         ▼
┌──────────────────┐
│   IN INBOX        │  Visible for ~24 hours
│   (unread)        │
└────────┬─────────┘
         │
         │  TAG-SYS Label Cleanup (Daily at 6 AM)
         │  Archives all OPS/DIGEST emails
         │  (by then, next day's digest has arrived or is coming)
         │
         ▼
┌──────────────────┐
│   [ARCHIVED]      │  Only current day's digest remains in inbox
└──────────────────┘

Result: At most ONE daily breakdown visible at any time.
```

**Current DIGEST senders:**
- Daily breakdown emails

---

## A-Label Combinations (Can Co-Exist)

### Common Patterns

| A-Labels | Typical B-Label | Scenario |
|----------|----------------|----------|
| `sales-inquiry` + `quote-request` | `needs-info` | "I need jerseys, how much?" |
| `sales-inquiry` + `bulk-order` | `quote` | "Quote for 1000 units attached" |
| `customer-service` + `product-defect` | `route-cs` | "My order arrived damaged" |
| `invoice-related` + `payment-issue` | `route-cs` | "I was charged twice" |
| `quote-request` + `urgent` | `needs-info` | "Need quote ASAP for event tomorrow" |

### Invalid Combinations (System Should Flag)

| Invalid Combo | Why | Corrective Action |
|---------------|-----|-------------------|
| `needs-info` + `quote` | Mutually exclusive states | Keep most recent B-label |
| `quote` + `invoice` | Can't be both | Investigate: likely thread transition |
| `route-cs` + `cs-involved` | Can't be both | Keep most recent B-label |
| Any Sales B-label + Any CS B-label | Cross-category conflict | See Sales/CS Mutual Exclusivity below |

### Sales/CS Mutual Exclusivity (Inline Enforcement)

Sales and CS labels are mutually exclusive at the category level. A thread cannot be both a Sales matter and a CS matter simultaneously. This is enforced **inline at the point of B-label application**, not just by the Label Logic Controller.

**Rule:** When applying a Sales B-label, first remove all CS labels. When applying a CS B-label, first remove all Sales labels.

**Sales labels (stripped when applying CS):**
- `TAG-SYS/Sales`
- `TAG-SYS/Sales/NEEDS-INFO`
- `TAG-SYS/Sales/QUOTE`
- `TAG-SYS/Sales/INVOICE`
- `TAG-SYS/Sales/LOST`

**CS labels (stripped when applying Sales):**
- `TAG-SYS/CS`
- `TAG-SYS/CS/ROUTE-CS`
- `TAG-SYS/CS/INVOLVED`
- `TAG-SYS/CS/DELEGATED`

**Where enforced:**
- Main Email Triage System — before any B-label application
- Contact Us Form Autoresponder — on each branch of the Switch node
- Needs Info to Quote Agent — before draft creation

**Why inline, not just the controller:** The Label Logic Controller runs every 10 minutes. Without inline enforcement, there is a window where both Sales and CS agents could see the same thread and fire conflicting auto-replies.

---

## Transition Rules

### Automatic Transitions (System-Driven)
```yaml
needs-info →  quote:
  trigger: Customer email contains required fields (size, quantity, etc.)
  confidence_required: >85%
  action: Generate quote, apply 'quote' label, remove 'needs-info'

quote →  invoice:
  trigger: Customer email contains acceptance language ("yes", "approved", "go ahead")
  confidence_required: >90%
  action: Generate invoice, apply 'invoice' label, remove 'quote'

route-cs →  cs-involved:
  trigger: CS team member replies to thread
  confidence_required: 100% (deterministic)
  action: Apply 'cs-involved', remove 'route-cs'
```

### Manual Transitions (Human-Initiated)
```yaml
any_state →  needs-info:
  trigger: Human manually applies label (via Gmail or n8n override)
  action: Remove other B-labels, apply 'needs-info'

any_state →  [RESOLVED]:
  trigger: Human removes all B-labels
  action: Archive thread (optional), log in audit sheet
```

### Rollback Transitions (Error Recovery)
```yaml
quote →  needs-info:
  trigger: Customer requests changes ("Can you add X to the quote?")
  confidence_required: >75%
  action: Remove 'quote', apply 'needs-info', flag for manual review

invoice →  quote:
  trigger: Invoice disputed or requires modification
  confidence_required: >80%
  action: Remove 'invoice', apply 'quote', notify accounting
```

---

## Label Logic Controller Rules

### Enforcement Logic (Runs Every 10 Minutes)
```javascript
// Pseudo-code for Label Logic Controller

function enforceExclusivity(thread) {
  var bLabels = getBLabels(thread); // Extract all B-labels

  if (bLabels.length === 0) {
    return; // No B-labels, nothing to do
  }

  if (bLabels.length === 1) {
    return; // Exactly one B-label, valid state
  }

  // Multiple B-labels detected (conflict)
  var mostRecent = getMostRecentLabel(thread, bLabels);
  var toRemove = bLabels.filter(function(label) {
    return label !== mostRecent;
  });

  removeLabels(thread, toRemove);
  logConflictResolution(thread, toRemove, mostRecent);
}

function getMostRecentLabel(thread, bLabels) {
  // Check thread history for most recent B-label application
  // Fallback: Use predefined priority order
  var priority = ['invoice', 'quote', 'needs-info', 'cs-delegated', 'cs-involved', 'route-cs'];

  for (var i = 0; i < priority.length; i++) {
    if (bLabels.indexOf(priority[i]) !== -1) {
      return priority[i];
    }
  }

  return bLabels[0]; // Fallback: keep first found
}
```

### Audit Logging

Every conflict resolution logs:
- Thread ID
- Timestamp
- Labels removed
- Label kept
- Reason (conflict resolution / manual override / system error)

**Audit Sheet Location:** [Google Sheets URL or file path]

---

## Edge Cases & Resolutions

### Case 1: Email Arrives During Label Logic Controller Run

**Problem:** Race condition between n8n triage and controller
**Resolution:** Controller checks last-modified timestamp; if <5 min, skip thread
**Status:** Implemented ✅

### Case 2: Customer Sends Multiple Emails in Same Thread

**Problem:** OpenAI might classify each message differently
**Resolution:** Thread-level classification (analyze full thread, not just new message)
**Status:** Implemented ✅

### Case 3: Gmail Native Filter Applies Wrong Label

**Problem:** Filter rule outdated, applies incorrect A-label
**Resolution:** Label Logic Controller doesn't override A-labels (only B-labels)
**Fix:** Update Gmail filter rules manually
**Status:** Known issue ⚠️

### Case 4: API Failure During B-Label Application

**Problem:** n8n applies A-labels successfully, but B-label API call fails
**Resolution:** Retry logic (3 attempts), fallback to no B-label
**Monitoring:** Error notifications sent to Adam
**Status:** Implemented ✅

---

## Testing Scenarios

### Test 1: Normal Flow

1. Send test email: "I need 100 t-shirts, what's the price?"
2. **Expected:** `sales-inquiry` + `quote-request` A-labels, `needs-info` B-label
3. Reply: "We need your design file and sizes"
4. **Expected:** B-label stays `needs-info`
5. Reply from customer: "Here's the design and sizes: [details]"
6. **Expected:** B-label changes to `quote`

### Test 2: Conflict Resolution

1. Manually apply both `needs-info` and `quote` labels to thread
2. Wait 3 minutes (Label Logic Controller cycle)
3. **Expected:** Only `quote` label remains (higher priority)
4. Check audit log for conflict entry

### Test 3: CS Routing

1. Send test email: "My order #12345 is missing items"
2. **Expected:** `customer-service` + `order-issue` A-labels, `route-cs` B-label
3. CS team member replies: "Looking into this now"
4. **Expected:** B-label changes to `cs-involved`

---

**Last Updated:** 2026-02-08
**See Also:** system_architecture.md, failure_modes.md
