{\rtf1\ansi\ansicpg1252\cocoartf2818
\cocoatextscaling0\cocoaplatform0{\fonttbl\f0\fswiss\fcharset0 Helvetica;}
{\colortbl;\red255\green255\blue255;}
{\*\expandedcolortbl;;}
\margl1440\margr1440\vieww11520\viewh8400\viewkind0
\pard\tx566\tx1133\tx1700\tx2267\tx2834\tx3401\tx3968\tx4535\tx5102\tx5669\tx6236\tx6803\pardirnatural\partightenfactor0

\f0\fs24 \cf0 # COMMS- Email Triage System - Complete Context\
\
## Table of Contents\
1. [System Overview](#system-overview)\
2. [Architecture](#architecture)\
3. [Label System](#label-system)\
4. [Workflows](#workflows)\
5. [Known Issues](#known-issues)\
6. [Debugging Patterns](#debugging-patterns)\
\
---\
\
## System Overview\
\
### Purpose\
Automated Gmail email triage for Dads Printing and Pro Team Jerseys to:\
- Classify incoming emails (sales vs support)\
- Track conversation state (quotes, invoices, CS routing)\
- Eliminate manual email sorting overhead\
\
### Success Metrics\
- Accurate email classification (A-labels)\
- Proper state transitions (B-labels)\
- Zero manual email sorting\
- No double-sends or mis-routes\
\
### Business Context\
- **Dads Printing:** dadsprinting.com\
- **Pro Team Jerseys:** proteamjerseys.com (Shopify)\
- Owner: Adam (entrepreneur managing multiple business units)\
\
---\
\
## Architecture\
\
### Technology Stack\
```\
Gmail (Email Source)\
  \uc0\u8595 \
Gmail Native Filters (initial tagging)\
  \uc0\u8595 \
n8n Workflows (classification + routing)\
  \uc0\u8595 \
OpenAI API (email content analysis)\
  \uc0\u8595 \
Gmail API (label management)\
  \uc0\u8595 \
Pipedrive (CRM for sales states)\
```\
\
### Data Flow\
1. Email arrives \uc0\u8594  Gmail native filters apply initial labels\
2. n8n "Email Triage System" workflow triggers (30s delay to avoid race conditions)\
3. OpenAI classifies email \uc0\u8594  confidence scores for A-labels and B-labels\
4. B-label logic applies (mutually exclusive, thread-level)\
5. Label Logic Controller enforces consistency every 3 minutes\
\
### Race Condition Mitigation\
**Problem:** n8n and Gmail native filters can process simultaneously, causing:\
- Double-tagging\
- State conflicts\
- Classification errors\
\
**Solution:** \
- 30-second delay in n8n workflows\
- Eventual consistency via Label Logic Controller\
- Conservative classification (err on side of no action)\
\
---\
\
## Label System\
\
### A-Labels (Categorization - Can Co-Exist)\
- `sales-inquiry` - New sales opportunities\
- `customer-service` - Support requests\
- `invoice-related` - Billing inquiries\
- `quote-request` - Pricing requests\
- [Additional A-labels defined in Email_Tags_W_Associated_Actions.xlsx]\
\
### B-Labels (Action States - Mutually Exclusive)\
\
#### Sales Thread States\
```\
needs-info \uc0\u8594  quote \u8594  invoice\
```\
- **needs-info:** Waiting for customer details to provide quote\
- **quote:** Quote sent, awaiting customer decision\
- **invoice:** Invoice sent, awaiting payment\
\
#### Customer Service Thread States\
```\
route-cs \uc0\u8594  cs-involved \u8594  cs-delegated\
```\
- **route-cs:** Needs CS team assignment\
- **cs-involved:** CS team actively working\
- **cs-delegated:** Delegated to specific team member\
\
### B-Label Mutual Exclusivity Rules\
- A thread can have ONLY ONE B-label at a time\
- Transitions are explicit (tracked in workflows)\
- Label Logic Controller removes conflicts every 3 minutes\
- Historical threads may have multiple B-labels (pre-cleanup deployment)\
\
### Confidence Scoring\
- **A-label confidence:** 0-100, determines which A-labels to apply\
- **B-label confidence:** 0-100, separate calculation, determines which B-label (if any)\
- Scores are NOT interdependent (fixing previous bug)\
- Default behavior when uncertain: Apply no B-label\
\
---\
\
## Workflows\
\
### Key n8n Workflows\
1. **Email Triage System** - Main classification workflow\
2. **Label Logic Controller** - Centralized cleanup (every 3 min)\
3. **Sales Quote Automator** - Handles `quote` state transitions\
4. **CS Router** - Handles `route-cs` assignments\
5. **Invoice Tracker** - Monitors `invoice` state\
\
### Common n8n Patterns\
```javascript\
// Traditional JavaScript syntax (required for n8n)\
function cleanLabels(labels) \{\
  var cleaned = [];\
  for (var i = 0; i < labels.length; i++) \{\
    if (labels[i].indexOf('COMMS-') === 0) \{\
      cleaned.push(labels[i]);\
    \}\
  \}\
  return cleaned;\
\}\
```\
\
### Node Naming Convention\
- Use descriptive names: "Get Many Messages", "Filter by Label", "Apply B-Label"\
- Avoid duplicates: n8n appends "1" which breaks references ("Get Many Messages1")\
- Case-sensitive: "filter" \uc0\u8800  "Filter"\
\
---\
\
## Known Issues\
\
### 1. Retroactive B-Label Cleanup\
**Status:** Consideration phase  \
**Issue:** Emails processed before cleanup logic have multiple B-labels  \
**Impact:** Historical data inconsistency  \
**Options:**\
- One-time retroactive cleanup script\
- Leave historical, enforce going forward\
- Gradual cleanup as threads receive new messages\
\
### 2. Node Reference Errors\
**Symptom:** Workflow fails with "node not found"  \
**Cause:** Node renamed/duplicated, references not updated  \
**Fix Pattern:**\
1. Open workflow JSON\
2. Search for node references (by name)\
3. Verify exact naming match (case-sensitive)\
4. Update all references\
\
### 3. Confidence Score Thresholds\
**Status:** Monitoring  \
**Issue:** Optimal thresholds for A-label vs B-label unclear  \
**Current:** Conservative approach (higher threshold = fewer labels)  \
**Next Step:** Audit spreadsheet tracking to tune thresholds\
\
---\
\
## Debugging Patterns\
\
### n8n Workflow Debugging\
1. **Check execution history** (last 50 runs)\
2. **Inspect node outputs** (JSON view)\
3. **Verify node connections** (no orphaned nodes)\
4. **Test with sample data** (manual trigger)\
5. **Check error handling** (what happens on API failure?)\
\
### Gmail Label Debugging\
```\
# Find threads with multiple B-labels\
label:needs-info label:quote\
\
# Find emails processed recently\
newer_than:1d label:COMMS-\
\
# Find errors from automation\
subject:"COMMS- Error" OR subject:"n8n workflow failed"\
```\
\
### OpenAI Classification Debugging\
- Check prompt consistency (stored in n8n workflow)\
- Verify API key validity\
- Monitor token usage (rate limits)\
- Review classification examples in responses\
\
---\
\
## Change Management\
\
### Staging New Rules\
1. **Document proposed change** in CHANGE_LOG.md\
2. **Identify affected workflows** (search for label references)\
3. **Create backup** of workflow JSON\
4. **Test in n8n test environment** (if available)\
5. **Deploy with monitoring** (first 24 hours closely watched)\
6. **Rollback plan ready** (previous JSON on standby)\
\
### Git Workflow\
```bash\
# Before changes\
git checkout -b comms-[feature-name]\
git commit -m "backup: pre-[change description]"\
\
# After changes\
git add workflows/[affected].json\
git commit -m "feat/fix: [description] - see CHANGE_LOG.md"\
git push origin comms-[feature-name]\
```\
\
---\
\
## References\
\
### Key Files\
- `rules/email_rules_master_clean.csv` - 40 Gmail filter rules\
- `rules/Email_Tags_W_Associated_Actions.xlsx` - Tag definitions and actions\
- `workflows/*.json` - n8n workflow exports\
- `docs/FAILURE_MODES.md` - Known edge cases\
- `docs/CHANGE_LOG.md` - Recent system changes\
\
### External Documentation\
- n8n docs: https://docs.n8n.io\
- Gmail API: https://developers.google.com/gmail/api\
- OpenAI API: https://platform.openai.com/docs\
- Pipedrive API: https://developers.pipedrive.com\
\
### Notion Pages\
- Cleanup matrices\
- Operational procedures\
- Label transition documentation\
\
---\
\
**Last Updated:** [Today's Date]  \
**Maintained By:** Adam + Claude Code  \
**Review Cycle:** Weekly (active issues), Monthly (full context)}