# COMMS- System Change Log

## Format
```
### [Date] - [Change Type]
**Component:** [Workflow/Rule/Label]
**Description:** [What changed]
**Reason:** [Why changed]
**Impact:** [Affected workflows/emails]
**Rollback:** [How to undo]
**Status:** [Deployed/Testing/Rolled Back]
```

---

## 2026

### [2026-02-08] - TAG-SYS Label Cleanup Workflow (n8n)
**Component:** New n8n workflow — `TAG-SYS Label Cleanup (Daily).json`
**Description:** Scheduled daily cleanup (2 AM) that strips TAG-SYS labels from threads no longer in the Inbox (archived > 2 days). Reduces label clutter (e.g., OPS had 86 tagged threads, Personal 31, most not in Inbox). Also strips Notifications label from archived threads. Receipts 2026 is handled separately — marked as read after 2 days but labels are preserved for tax records.
**Reason:** Label counts were inflated by old archived threads, making label sidebar unreliable as a work queue
**Impact:** All TAG-SYS A-labels, B-labels, and Notifications cleaned from archived threads; Receipts 2026 marked as read only
**Rollback:** Disable the n8n workflow
**Status:** ⏳ Ready to import — replace `__NOTIFICATIONS_LABEL_ID__` with actual Gmail label ID

**Routes:**
- Sales: strips A-label + NEEDS-INFO, QUOTE, INVOICE, LOST
- CS: strips A-label + ROUTE-CS, INVOLVED, DELEGATED
- Personal, OPS, GOV, Payable, Receivable, Unsure: strips A-label
- Notifications: strips label
- Receipts 2026: mark as read only (labels preserved)

---

### [2026-02-08] - Workflow Conflict Prevention & Repo Cleanup
**Component:** Contact Us Form Autoresponder, Following Up Reply Automator, Main Email Triage System, Needs Info to Quote Agent, repo structure
**Description:** Fixed three categories of workflow conflicts:
1. **Orphaned B-labels:** Contact Us Form Autoresponder and Following Up Reply Automator were applying B-labels (NEEDS-INFO, ROUTE-CS) without parent A-labels. Label Logic Controller was silently stripping them. Fixed by adding A-label nodes (TAG-SYS/Sales, TAG-SYS/CS) before B-label application in both workflows.
2. **Sales/CS cross-contamination:** When a thread transitions from Sales to CS (or vice versa), the old category's labels were left behind, allowing multiple downstream workflows to fire on the same thread. Fixed by adding inline label cleanup at the point of B-label application — applying a CS B-label now strips all Sales labels first, and vice versa. This is done upstream (in the Main Triage and Contact Form Autoresponder) so downstream agents never see conflicting labels.
3. **Repo cleanup:** Converted RTF docs to Markdown, Excel to CSV, added .gitignore, removed duplicate workflow files.

**Reason:** Prevent silent routing failures and duplicate auto-replies to customers
**Impact:** All contact form submissions and follow-up replies now route correctly; no more cross-category label conflicts
**Rollback:** Remove the new A-label and label-cleanup nodes from affected workflows
**Status:** ✅ Deployed

**Workflows Modified:**
- Contact Us Form Autoresponder: Added "Add Sales A-Label" and "Add CS A-Label" nodes; added cross-category label removal on each branch
- Following Up Reply Automator: Added "Add CS A-Label" node on unhappy path
- Main Email Triage System: Added "Remove CS Labels" before Sales B-label application; added "Remove Sales Labels" before CS B-label application
- Needs Info to Quote Agent: Added "Remove CS Labels" before draft creation

---

## 2025

### [2025-02-05] - Cleanup Logic Deployment
**Component:** Label Logic Controller workflow
**Description:** Deployed centralized B-label mutual exclusivity enforcement
**Reason:** Eliminate B-label conflicts (multiple action states on same thread)
**Impact:** All email threads processed every 3 minutes, conflicts auto-resolved
**Rollback:** Disable Label Logic Controller workflow in n8n
**Status:** ✅  Deployed, monitoring for 7 days

**Technical Details:**
- Runs every 3 minutes
- Checks all threads with >1 B-label
- Keeps most recent B-label, removes others
- Logs all changes to audit sheet

**Known Issues:**
- Retroactive cleanup not yet implemented (pre-deployment threads may have conflicts)
- Controller doesn't yet optimize for thread last-modified (may process stale threads)

---

### [2025-02-03] - Confidence Scoring Refactor
**Component:** OpenAI classification logic
**Description:** Separated A-label and B-label confidence scoring
**Reason:** Fix bug where high A-label confidence incorrectly triggered B-label assignment
**Impact:** More conservative B-label application (fewer false positives)
**Rollback:** Revert to previous OpenAI prompt (backup saved in `/prompts/openai_v1.txt`)
**Status:** ✅  Deployed, accuracy improved by ~15%

**Before:**
```javascript
if (confidence > 80) {
  applyALabel(email);
  applyBLabel(email); // Bug: uses same confidence
}
```

**After:**
```javascript
if (aLabelConfidence > 80) {
  applyALabel(email);
}
if (bLabelConfidence > 85) { // Separate threshold
  applyBLabel(email);
}
```

---

### [2025-01-28] - Race Condition Mitigation
**Component:** Email Triage System workflow
**Description:** Added 30-second delay before processing new emails
**Reason:** Gmail native filters and n8n were processing simultaneously, causing conflicts
**Impact:** Reduced classification errors by ~60%
**Rollback:** Remove delay node from workflow
**Status:** ✅  Deployed, stable for 8 days

**Lessons Learned:**
- Simple delays > complex sync logic
- 30s is sufficient (tested 10s, 15s, 30s, 60s)
- No noticeable impact on response time (customers don't expect instant replies)

---

### [2025-01-20] - B-Label System Redesign
**Component:** All workflows
**Description:** Converted B-labels from message-level to thread-level
**Reason:** Action states apply to conversations, not individual messages
**Impact:** Major refactor, all workflows updated
**Rollback:** Not feasible (fundamental architecture change)
**Status:** ✅  Deployed, major improvement in state consistency

**Migration:**
- Updated 12 workflows
- Re-trained OpenAI prompts
- Migrated ~2,000 existing threads (manual cleanup)
- Created Label Logic Controller to enforce new model

---

## 2024

### [2024-12-15] - Initial System Deployment
**Component:** Entire COMMS- system
**Description:** First production deployment of email triage automation
**Reason:** Manual email sorting unsustainable at current volume
**Impact:** Eliminated ~10 hours/week of manual work
**Rollback:** Disable all n8n workflows, revert to manual sorting
**Status:** ✅  Deployed, iterated over 7 weeks

**Initial Components:**
- Email Triage System (main workflow)
- Sales Quote Automator
- CS Router
- Gmail native filters (40 rules)
- OpenAI integration

---

## Upcoming Changes (Planned)

### [Planned] - Retroactive B-Label Cleanup
**Component:** One-time script
**Description:** Clean up threads with multiple B-labels from pre-cleanup-logic era
**Reason:** Historical data consistency
**Impact:** ~1,200 threads (estimated)
**Rollback:** Restore labels from backup (create before running)
**Status:** ⏳  Planning phase

**Considerations:**
- Risk of disrupting active threads
- May trigger downstream workflows unexpectedly
- Should run during low-traffic period (weekend)

---

### [Planned] - Confidence Threshold Tuning
**Component:** OpenAI classification
**Description:** Adjust A-label and B-label confidence thresholds based on audit data
**Reason:** Optimize precision/recall balance
**Impact:** TBD (requires analysis)
**Rollback:** Revert to current thresholds (A: 80%, B: 85%)
**Status:** ⏳  Data collection phase (2 more weeks)

---

### [Planned] - Workflow Consolidation
**Component:** Sales workflows
**Description:** Merge "Quote Automator" and "Follow-up Automator" into single workflow
**Reason:** Reduce complexity, easier maintenance
**Impact:** Fewer workflows to monitor
**Rollback:** Re-deploy separate workflows
**Status:** ⏳  Design phase

---

## Incident Log

### [2025-01-25] - OpenAI API Timeout
**Severity:** Medium
**Duration:** 45 minutes
**Impact:** 23 emails not classified
**Resolution:** Implemented retry logic + timeout increase (5s →  10s)
**Prevention:** Added OpenAI status monitoring

---

### [2025-01-18] - n8n Workflow Loop
**Severity:** High
**Duration:** 2 hours (overnight, undetected initially)
**Impact:** 147 emails processed repeatedly, API quota 80% consumed
**Resolution:** Disabled workflow, added idempotency check
**Prevention:** Circuit breaker logic (max 5 runs per email per hour)

---

### [2025-01-10] - Gmail API Rate Limit
**Severity:** Critical
**Duration:** 15 minutes
**Impact:** All workflows stalled
**Resolution:** Reduced Label Logic Controller frequency (1 min →  3 min)
**Prevention:** API usage monitoring + alerts

---

**Last Updated:** 2026-02-08
**Review Cycle:** Update after every change, full review monthly
