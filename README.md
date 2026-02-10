## 🤖  Claude Code Integration

This repository is optimized for use with [Claude Code](https://www.anthropic.com/claude/code), an AI coding assistant.

### Quick Start with Claude Code

1. **Clone this repo** and navigate to the directory:
```bash
   git clone [repo-url]
   cd comms-email-triage
```

2. **Start Claude Code:**
```bash
   cline
```

3. **Context auto-loads** from `.clinerules` - Claude Code will understand the system architecture, constraints, and operating principles.

### Common Commands
```bash
# Review a workflow for race conditions
"Analyze workflows/label-logic-controller.json for potential race conditions"

# Validate rules against B-label mutual exclusivity
"Check rules/email_rules_master_clean.csv for conflicts with B-label states"

# Propose a new feature
"Suggest adding a 'high-priority' A-label with full change protocol"

# Debug a workflow issue
"Investigate why workflow X is failing - see logs in [location]"
```

### Documentation Structure

- **`clinerules.md`** - Auto-loaded context (core principles, constraints, checklist)
- **`Claude_context.md`** - Detailed system reference
- **`docs/system_architecture.md`** - Component diagrams and data flow
- **`docs/label_state_machine.md`** - A/B label rules and transitions
- **`docs/failure_modes.md`** - Known edge cases and mitigations
- **`docs/change_log.md`** - Historical changes and incidents

### Maintenance

- **Update `clinerules.md`** after major system changes
- **Update `docs/change_log.md`** after every deployment
- **Review `docs/failure_modes.md`** quarterly
- **Audit documentation** monthly for staleness

### Getting Help

If Claude Code seems unfamiliar with system details:
1. Check if `clinerules.md` exists and is up-to-date
2. Reference specific docs: `"See docs/label_state_machine.md for B-label rules"`
3. Provide current context: `"We're debugging the Label Logic Controller deployed 2025-02-05"`
