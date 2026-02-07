
```bash\
   git clone [repo-url]\
   cd comms-email-triage\
```\
\
2. **Start Claude Code:**\
```bash\
   cline\
```\
\
3. **Context auto-loads** from `.clinerules` - Claude Code will understand the system architecture, constraints, and operating principles.\
\
### Common Commands\
```bash\
# Review a workflow for race conditions\
"Analyze workflows/label-logic-controller.json for potential race conditions"\
\
# Validate rules against B-label mutual exclusivity\
"Check rules/email_rules_master_clean.csv for conflicts with B-label states"\
\
# Propose a new feature\
"Suggest adding a 'high-priority' A-label with full change protocol"\
\
# Debug a workflow issue\
"Investigate why workflow X is failing - see logs in [location]"\
```\
\
### Documentation Structure\
\
- **`.clinerules`** - Auto-loaded context (core principles, constraints, checklist)\
- **`CLAUDE_CONTEXT.md`** - Detailed system reference\
- **`docs/SYSTEM_ARCHITECTURE.md`** - Component diagrams and data flow\
- **`docs/LABEL_STATE_MACHINE.md`** - A/B label rules and transitions\
- **`docs/FAILURE_MODES.md`** - Known edge cases and mitigations\
- **`docs/CHANGE_LOG.md`** - Historical changes and incidents\
\
### Maintenance\
\
- **Update `.clinerules`** after major system changes\
- **Update `CHANGE_LOG.md`** after every deployment\
- **Review `FAILURE_MODES.md`** quarterly\
- **Audit documentation** monthly for staleness\
\
### Getting Help\
\
If Claude Code seems unfamiliar with system details:\
1. Check if `.clinerules` exists and is up-to-date\
2. Reference specific docs: `"See docs/LABEL_STATE_MACHINE.md for B-label rules"`\
3. Provide current context: `"We're debugging the Label Logic Controller deployed 2025-02-05"`}
