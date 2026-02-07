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

### Enforcement Logic (Runs Every 3 Minutes)
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

**Last Updated:** [Today's Date]
**See Also:** SYSTEM_ARCHITECTURE.md, FAILURE_MODES.md
