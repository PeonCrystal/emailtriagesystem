# Known Failure Modes & Mitigations

## Critical Failures (System-Breaking)

### 1. Gmail API Rate Limit Exceeded

**Symptom:** n8n workflows fail with "429 Too Many Requests"
**Cause:** Exceeded 250 quota units/second/user
**Impact:** Emails not processed, labels not applied, system stalled

**Mitigation:**
- Monitor API usage in n8n logs
- Add rate-limiting logic (max X requests/minute)
- Implement exponential backoff (retry after 30s, 60s, 120s...)
- Alert Adam immediately if rate limit hit

**Rollback:**
- Temporarily disable high-frequency workflows (Label Logic Controller)
- Process emails manually until quota resets (resets every minute)

**Prevention:**
- Audit n8n workflows for unnecessary API calls
- Batch operations (fetch 50 emails at once vs 50 separate calls)
- Cache label lists (refresh every 5 min vs every workflow run)

---

### 2. OpenAI API Failure (Extended Outage)

**Symptom:** Classification requests timeout or return errors
**Cause:** OpenAI service degradation or API key issue
**Impact:** Emails not classified, no A-labels or B-labels applied

**Mitigation:**
- Fallback to rule-based classification (keyword matching)
- Apply `needs-review` label to all unclassified emails
- Queue emails for re-processing when API recovers
- Monitor OpenAI status page: https://status.openai.com

**Rollback:**
- Disable triage workflows, process manually
- Use Gmail native filters only (limited functionality)

**Prevention:**
- Implement API health check before each workflow run
- Set aggressive timeouts (10s max)
- Store last-known-good API response for pattern matching

---

### 3. n8n Workflow Infinite Loop

**Symptom:** Same email processed repeatedly, label changes back and forth
**Cause:** Workflow triggers itself (e.g., "on label change" triggers workflow that changes labels)
**Impact:** API quota exhausted, email thread corrupted, system paralyzed

**Mitigation:**
- "Already processed" marker (custom label or thread metadata)
- Idempotency checks (if email has `processed:timestamp` label, skip)
- Circuit breaker (if workflow runs >5 times for same email in 10 min, halt)

**Rollback:**
- Manually disable looping workflow in n8n
- Remove all B-labels from affected threads
- Re-run Label Logic Controller to clean up

**Prevention:**
- Never trigger workflows on label changes they themselves make
- Always check "already processed" before applying logic
- Unit test workflows with same email multiple times

---

## High-Impact Failures (Data Corruption)

### 4. B-Label Conflict Not Detected

**Symptom:** Thread has multiple B-labels (e.g., `needs-info` + `quote`)
**Cause:** Label Logic Controller didn't run or failed
**Impact:** Downstream workflows fire incorrectly (double-send emails, wrong state)

**Mitigation:**
- Alert if thread has >1 B-label (daily audit)
- Manual cleanup script (remove all but most recent)
- Label Logic Controller health check (logs last run time)

**Detection:**
```
Gmail search: label:needs-info label:quote
```

**Rollback:**
- Run cleanup script immediately
- Investigate why controller failed (check n8n logs)

**Prevention:**
- Redundant controller (backup n8n instance)
- Controller monitors itself (if no run in >5 min, alert)

---

### 5. A-Label Misclassification Cascade

**Symptom:** Sales email classified as CS, routed to wrong team
**Cause:** OpenAI confidence score too low, default logic applied
**Impact:** Delayed response, customer dissatisfaction, lost sale

**Mitigation:**
- Confidence threshold tuning (require >80% for A-label)
- Human review queue for <80% confidence
- Feedback loop (correct misclassifications, retrain prompts)

**Detection:**
- Monitor A-label confidence scores in audit sheet
- Flag emails with <70% confidence for review

**Rollback:**
- Manually re-classify email
- Update OpenAI prompt to avoid similar errors

**Prevention:**
- Regularly review classification accuracy (weekly audit)
- Maintain example set of correctly classified emails
- A/B test prompt variations

---

## Medium-Impact Failures (Workflow Disruption)

### 6. Pipedrive Sync Failure

**Symptom:** B-label changes in Gmail don't sync to Pipedrive CRM
**Cause:** Pipedrive API error, credential expiry, workflow bug
**Impact:** CRM data stale, sales team works with outdated info

**Mitigation:**
- Retry logic (3 attempts with backoff)
- Queue failed syncs for manual review
- Daily reconciliation (compare Gmail labels vs Pipedrive stages)

**Detection:**
- Pipedrive webhook failure notifications
- Daily audit: count emails with `quote` label vs Pipedrive deals in "Quote Sent" stage

**Rollback:**
- Manually update Pipedrive from Gmail labels
- Disable sync temporarily if causing errors

**Prevention:**
- Monitor Pipedrive API health
- Test credentials weekly (automated health check)
- Log all sync attempts with status

---

### 7. Email Thread Fragmentation

**Symptom:** Replies in same conversation treated as new threads
**Cause:** Gmail doesn't recognize reply (missing `In-Reply-To` header)
**Impact:** B-label applied to new thread, original thread orphaned

**Mitigation:**
- Thread ID matching (group by subject + participants)
- Merge logic (if subject matches 90% + same sender, merge)
- Manual review for orphaned threads

**Detection:**
```
Gmail search: subject:[partial subject] label:needs-info
```
(Should return only one thread, if multiple = fragmentation)

**Rollback:**
- Manually merge threads in Gmail (move messages)
- Re-apply correct B-label

**Prevention:**
- Educate customers: always reply to email (don't compose new)
- Implement fuzzy thread matching in n8n

---

## Low-Impact Failures (Minor Issues)

### 8. Delayed Processing (>1 Hour)

**Symptom:** Email arrives but not classified for 1+ hours
**Cause:** n8n polling interval, workflow backlog, API slowness
**Impact:** Delayed customer response, but no data loss

**Mitigation:**
- Reduce polling interval (3 min →  1 min for critical workflows)
- Parallel processing (run multiple instances)
- Priority queue (urgent emails processed first)

**Detection:**
- Monitor average processing time (should be <5 min)
- Alert if >20 unprocessed emails in queue

**Rollback:**
- Manually process urgent emails
- Temporarily increase n8n resources (CPU/memory)

**Prevention:**
- Regular performance audits
- Optimize workflows (remove unnecessary steps)
- Upgrade n8n hosting (more resources)

---

### 9. False Positive "Urgent" Flags

**Symptom:** Non-urgent emails tagged as urgent
**Cause:** OpenAI over-sensitive to keywords ("ASAP", "urgent")
**Impact:** CS team distracted by false alarms, real urgency diluted

**Mitigation:**
- Tune OpenAI prompt (define "urgent" more precisely)
- Human review for first 50 "urgent" classifications
- Feedback loop (mark false positives)

**Detection:**
- Manual review of `urgent` labeled emails (daily sample)
- Track CS team complaints about false urgency

**Rollback:**
- Temporarily disable `urgent` classification
- Fall back to manual urgency determination

**Prevention:**
- Maintain strict urgency criteria in OpenAI prompt
- Periodic prompt review (quarterly)

---

### 10. Duplicate Email Processing

**Symptom:** Same email processed twice, labels applied twice
**Cause:** n8n workflow re-triggered, no idempotency check
**Impact:** Wasteful API calls, confusion in audit logs, but no functional issue

**Mitigation:**
- Idempotency key (email message ID + timestamp)
- Skip if `processed:[timestamp]` label exists
- Dedupe logic (if processed in last 10 min, skip)

**Detection:**
- Audit log shows same email processed multiple times
- n8n execution history shows duplicate runs

**Rollback:**
- No rollback needed (duplicate processing is benign)
- Clean up duplicate audit log entries

**Prevention:**
- Always check "already processed" at start of workflow
- Use unique execution IDs to track processing

---

## Recovery Procedures

### General Recovery Checklist

1. **Identify failure mode** (match symptom to list above)
2. **Assess impact** (Critical / High / Medium / Low)
3. **Execute mitigation** (follow steps for that failure mode)
4. **Verify recovery** (test with sample email)
5. **Document incident** (add to CHANGE_LOG.md)
6. **Implement prevention** (update workflows to avoid recurrence)

### Emergency Contacts

- **n8n instance:** [URL or access info]
- **OpenAI API dashboard:** https://platform.openai.com/usage
- **Gmail API console:** https://console.cloud.google.com/apis/dashboard
- **Pipedrive admin:** [URL]
- **Audit logs:** [Google Sheets URL]

### Health Check Commands
```bash
# Check n8n workflow status
curl -X GET https://[n8n-instance]/workflows

# Check Gmail API quota
# (requires OAuth, manual check in Google Cloud Console)

# Check OpenAI API status
curl https://status.openai.com/api/v2/status.json

# Check last Label Logic Controller run
# (inspect audit log timestamp, should be <3 min old)
```

---

**Last Updated:** [Today's Date]
**Incident Log:** See CHANGE_LOG.md for historical failures
