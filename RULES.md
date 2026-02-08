# Repository Rules & Guidelines

## 🎯 Purpose

This document establishes the rules, best practices, and operational guidelines for managing this shipping automation system repository.

## 🔐 Security Rules (MANDATORY)

### Rule #1: Never Commit Credentials
**NEVER commit to this repository:**
- API keys or tokens
- OAuth credentials
- Webhook URLs containing security tokens
- QuickBooks connection credentials
- Shipping carrier API credentials
- Monday.com board IDs that are paired with API keys
- Customer data or PII (Personally Identifiable Information)
- Order data with customer names, addresses, or payment info

### Rule #2: Blueprint Sanitization
Before committing Make.com blueprints:
1. Export blueprint from Make.com
2. Search JSON for any hardcoded credentials
3. Replace sensitive IDs with placeholders if needed
4. Verify no customer data leaked into test data
5. Commit only the sanitized version

### Rule #3: Access Control
- This is a **PRIVATE repository** - do not make public
- Only authorized team members should have access
- Review access permissions quarterly
- Remove access immediately when team members leave

## 📝 Commit Guidelines

### Commit Message Format
```
[Type] Brief description

Detailed explanation if needed
- What changed
- Why it changed
- Impact on other workflows
```

**Types:**
- `[FIX]` - Bug fix or error correction
- `[FEAT]` - New feature or workflow
- `[UPDATE]` - Modification to existing workflow
- `[DOC]` - Documentation only
- `[CONFIG]` - Configuration change
- `[REFACTOR]` - Code reorganization without functionality change

### Examples:
```
[FIX] BlindShipping label generation failing on PO Box addresses

Updated address validation module to properly handle PO Box formats.
Added fallback to carrier-specific PO Box routing.
Tested with 5 different PO Box formats.
```

```
[FEAT] Added FedEx as additional carrier option

- Added FedEx rate request module to rate comparison
- Updated carrier selection logic to include FedEx
- Added FedEx-specific label formatting
- Tested with ground and priority shipping
```

## 🔄 Workflow Change Process

### Before Making Changes:

1. **Understand Current State**
   - Read the entire workflow you're modifying
   - Trace data flow from trigger to completion
   - Identify dependent workflows

2. **Document Intent**
   - What problem are you solving?
   - What is the expected outcome?
   - What are potential side effects?

3. **Plan Testing**
   - How will you test the change?
   - What test scenarios cover edge cases?
   - How will you verify no regression?

### Making Changes:

1. **Work in Make.com**
   - Clone the scenario if possible
   - Test in cloned/dev version first
   - Use Make.com's testing tools

2. **Test Thoroughly**
   - Test happy path
   - Test error conditions
   - Test with real data volumes
   - Verify all branches of conditional logic

3. **Document Changes**
   - Add inline comments in Make.com
   - Update this repository's documentation
   - Note any configuration changes needed

### After Changes:

1. **Export Blueprint**
   - Export from Make.com as JSON
   - Sanitize per Security Rules
   - Replace old file in repository

2. **Update Documentation**
   - Update README.md if user-facing
   - Update CLAUDE_CONTEXT.md if architecture changed
   - Add notes to CHANGELOG.md

3. **Monitor**
   - Watch workflow executions for 24-48 hours
   - Check error logs
   - Verify expected outcomes
   - Be ready to rollback if needed

## 🚨 Emergency Procedures

### Critical Workflow Failure

**Immediate Actions:**
1. Disable failing workflow in Make.com (pause scenario)
2. Document the failure (symptoms, error messages)
3. Notify team via Monday.com or established channel
4. Implement manual workaround if needed

**Recovery:**
1. Identify root cause
2. If recent change caused it, rollback to previous version
3. Test fix thoroughly before re-enabling
4. Document incident and resolution

### Rollback Procedure

1. Locate previous working version in git history
2. Export that version's JSON
3. Import to Make.com, replacing failed scenario
4. Verify connections and settings
5. Test before re-enabling
6. Document what was rolled back and why

## 📊 Monitoring & Maintenance

### Daily Checks
- Review Make.com execution logs for errors
- Check Monday.com for stalled orders
- Verify invoices are being created

### Weekly Reviews
- Review error patterns
- Check for workflow timeouts
- Verify all webhooks are active
- Review shipping costs vs quotes (accuracy check)

### Monthly Audits
- Full workflow testing
- Documentation accuracy review
- Credential rotation if needed
- Performance optimization review

### Quarterly Reviews
- Architecture review - are workflows still optimal?
- Integration health - are APIs up to date?
- Team training - does everyone understand the system?
- Disaster recovery test

## 🤝 Collaboration Rules

### Code Review (When Applicable)
For significant changes:
1. Create feature branch
2. Make changes and commit
3. Have another team member review
4. Merge to main after approval

### Documentation Standards
- Keep README.md updated with any new workflows
- Update CLAUDE_CONTEXT.md when architecture changes
- Maintain CHANGELOG.md with all significant changes
- Comment complex logic in Make.com scenarios

### Communication
- Announce significant changes to team before deployment
- Document rationale for major decisions
- Share lessons learned from troubleshooting
- Maintain institutional knowledge

## 🎓 Training Requirements

### New Team Members Must:
1. Read all documentation in this repository
2. Review each workflow in Make.com interface
3. Understand the trigger → process → outcome for each
4. Shadow experienced team member for 1 week
5. Successfully complete test scenarios before working independently

### Ongoing Education:
- Stay updated on Make.com platform changes
- Monitor shipping carrier API updates
- Review Monday.com and QuickBooks new features
- Share knowledge with team regularly

## 📈 Performance Standards

### Workflow Execution
- 95%+ success rate for each workflow
- < 5 minute execution time for label generation
- < 1 hour for invoice creation
- Same-day tracking notification

### System Reliability
- 99%+ uptime during business hours
- < 1 hour mean time to recovery for critical failures
- Documented procedures for all common failures

### Data Accuracy
- 100% accuracy for customer addresses
- 100% accuracy for shipping costs in invoices
- 100% accuracy for tracking number capture

## 🔧 Troubleshooting Hierarchy

### Level 1: Basic Checks
- Is the workflow enabled?
- Are webhooks active?
- Are API connections valid?
- Check recent error logs

### Level 2: Data Flow Analysis
- Trace data through each module
- Verify mappings are correct
- Check for conditional filters
- Review transformation logic

### Level 3: Deep Debugging
- Export execution bundles
- Analyze API responses
- Check for timing issues
- Review Make.com platform status

### Level 4: Escalation
- Contact Make.com support
- Contact carrier/Monday/QuickBooks support
- Consult with external experts
- Consider architectural changes

## 🌟 Best Practices

### Error Handling
- Always include error handlers
- Log errors with context
- Notify team of critical errors
- Include retry logic where appropriate

### Data Validation
- Validate addresses before label generation
- Verify costs before invoice creation
- Check tracking numbers before notifications
- Sanitize user inputs

### Efficiency
- Minimize API calls
- Use caching where possible
- Batch operations when feasible
- Monitor execution time

### Maintainability
- Use consistent naming conventions
- Comment complex logic
- Keep workflows focused (single responsibility)
- Avoid unnecessary complexity

## 📞 Support Contacts

### Internal
- System Owner: [Contact via Monday.com]
- Technical Lead: [Contact via Monday.com]
- Backup Admin: [Contact via Monday.com]

### External
- Make.com Support: support.make.com
- Monday.com Support: support.monday.com
- QuickBooks Support: [Account-specific]
- Shipping Carrier Support: [Carrier-specific]

## 📜 Version Control

### Branching Strategy
- `main` - Production-ready code
- `development` - Testing and development
- `feature/*` - Specific features or fixes
- `hotfix/*` - Emergency production fixes

### Version Numbering
Format: `MAJOR.MINOR.PATCH`
- MAJOR: Breaking changes or complete rewrites
- MINOR: New features, non-breaking changes
- PATCH: Bug fixes, small improvements

Current Version: 2.0 (as indicated by workflow names)

## ⚖️ Compliance

### Data Privacy
- Comply with GDPR, CCPA, and other applicable regulations
- Do not store customer data unnecessarily
- Implement data retention policies
- Provide data deletion capabilities

### Business Compliance
- Maintain audit trails for financial transactions
- Keep accurate shipping records
- Comply with carrier terms of service
- Follow e-commerce regulations

---

**Acknowledgment:** By working with this repository, you agree to follow these rules and guidelines. Violations may result in access removal and could impact business operations.

**Last Updated:** 2026-02-08  
**Rules Version:** 1.0
