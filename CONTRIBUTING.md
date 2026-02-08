# Contributing Guidelines

Thank you for contributing to our shipping automation system! This document provides guidelines for internal team members working on this project.

## 🎯 Project Goals

This system serves the operational needs of:
- **dadsprinting.com** - Established printing business
- **proteamjerseys.com** - Growing team apparel business

Our automation must be:
- **Reliable**: 99%+ uptime during business hours
- **Accurate**: Zero tolerance for shipping/billing errors
- **Fast**: Minimize customer wait time
- **Maintainable**: Clear, documented, understandable

## 🚀 Getting Started

### 1. Access Requirements

Before contributing, ensure you have:
- [ ] Make.com account access
- [ ] Monday.com workspace access
- [ ] QuickBooks credentials (if working on financial workflows)
- [ ] GitHub repository access
- [ ] Completed initial training (see RULES.md)

### 2. Development Environment

**Tools Needed:**
- Make.com account
- JSON editor (VS Code recommended)
- Git client
- Access to staging Monday.com board (if available)

**Recommended Setup:**
```bash
# Clone the repository
git clone [repository-url]
cd shipping-automation-system

# Create your working branch
git checkout -b feature/your-feature-name
```

## 📋 Contribution Workflow

### Step 1: Identify the Need

Before making changes:
1. Document the problem or enhancement need
2. Discuss with team (via Monday.com or team channel)
3. Get approval for significant changes
4. Check if similar work is already in progress

### Step 2: Plan Your Changes

Create a brief plan including:
- Which workflow(s) will be modified
- Expected behavior changes
- Potential risks or side effects
- Testing strategy
- Rollback plan

### Step 3: Make Changes in Make.com

**Best Practices:**
1. **Clone scenarios** when possible for testing
2. **Add comments** to complex modules
3. **Use consistent naming** for variables
4. **Test incrementally** - don't make all changes at once
5. **Document as you go** - note what each change does

**Testing Checklist:**
- [ ] Test with realistic data
- [ ] Test error conditions
- [ ] Test edge cases (empty fields, special characters, etc.)
- [ ] Verify no impact on related workflows
- [ ] Check execution time (should be reasonable)
- [ ] Confirm all paths through conditional logic

### Step 4: Export and Document

Once tested in Make.com:

1. **Export the blueprint**
   - Go to scenario settings in Make.com
   - Export as JSON
   - Save with descriptive filename

2. **Sanitize the export**
   - Remove any API keys or credentials
   - Replace sensitive IDs with placeholders if needed
   - Verify no customer data in test samples

3. **Update documentation**
   - Update README.md if functionality changed
   - Update CLAUDE_CONTEXT.md if architecture changed
   - Add entry to CHANGELOG.md
   - Add inline comments explaining complex logic

### Step 5: Commit Changes

Use clear, descriptive commit messages:

```bash
git add [modified-files]
git commit -m "[TYPE] Brief description

Detailed explanation:
- What changed
- Why it changed
- How it was tested
- Any special considerations"
```

**Example:**
```bash
git commit -m "[FEAT] Added address validation to blind shipping

- Added address validation module before label booking
- Catches PO Box vs residential mismatches
- Prevents 90% of address-related failures
- Tested with 50+ address variations
- Falls back gracefully if validation API down"
```

### Step 6: Create Pull Request (if using branches)

1. Push your branch to GitHub
2. Create pull request with description
3. Request review from team member
4. Address any feedback
5. Merge after approval

### Step 7: Deploy and Monitor

After merging:

1. **Import to Production Make.com**
   - Import the updated JSON
   - Verify all connections
   - Confirm settings match documentation

2. **Enable and Monitor**
   - Enable the scenario
   - Watch first few executions closely
   - Monitor error logs for 24-48 hours
   - Be available to rollback if needed

3. **Communicate**
   - Notify team of deployment
   - Document any new procedures
   - Update training materials if needed

## 🐛 Bug Fixes

### Reporting Bugs

When reporting a bug:
1. **Workflow affected**: Which scenario?
2. **Symptom**: What's happening vs what should happen?
3. **Frequency**: Every time, intermittent, specific conditions?
4. **Impact**: How many orders/customers affected?
5. **Error messages**: Copy exact error text
6. **Recent changes**: Any changes before bug appeared?

### Fixing Bugs

Priority levels:
- **P0 - Critical**: Orders can't ship → Fix immediately
- **P1 - High**: Manual workaround needed → Fix within 24h
- **P2 - Medium**: Minor impact → Fix within week
- **P3 - Low**: Cosmetic or nice-to-have → Schedule appropriately

For P0/P1 bugs:
1. Disable failing workflow if causing harm
2. Implement immediate workaround
3. Fix thoroughly (don't hack)
4. Test extensively
5. Document root cause and fix

## ✨ Feature Requests

### Proposing Features

New feature proposal should include:
- **Business need**: What problem does it solve?
- **User story**: "As a [role], I want [feature] so that [benefit]"
- **Success criteria**: How do we know it's working?
- **Priority**: How important vs other work?
- **Effort estimate**: Hours/days/weeks?

### Implementing Features

For new features:
1. **Design first**: Plan the workflow before building
2. **Start simple**: MVP first, enhance later
3. **Consider scale**: Will it work with 10x current volume?
4. **Plan maintenance**: Who will maintain this?
5. **Document thoroughly**: Future you will thank you

## 📚 Documentation Standards

### Code Comments (in Make.com)

Add notes to modules explaining:
- **Purpose**: What does this module do?
- **Input**: What data does it expect?
- **Output**: What does it produce?
- **Edge cases**: What special conditions are handled?

### Markdown Documentation

Follow these conventions:
- Use clear headers (##, ###)
- Include code examples in code blocks
- Add links to related documentation
- Keep sentences concise
- Use bullet points for lists
- Add emojis sparingly for visual breaks (🎯, ✅, ⚠️)

### Examples Over Explanation

When possible, show rather than tell:
```json
// Good example
{
  "mapper": {
    "name": "{{customer.firstName}} {{customer.lastName}}",
    "email": "{{customer.email}}"
  }
}
```

## 🔍 Code Review Guidelines

### As a Reviewer

Look for:
- **Correctness**: Does it solve the problem?
- **Completeness**: Are edge cases handled?
- **Clarity**: Is it understandable?
- **Performance**: Will it be fast enough?
- **Security**: Any credentials exposed?
- **Documentation**: Is it documented?

Provide:
- **Constructive feedback**: Explain the "why"
- **Specific suggestions**: Offer alternatives
- **Positive reinforcement**: Note what's done well
- **Questions**: Ask if you don't understand

### As a Contributor

Expect:
- Questions about your approach
- Requests for clarification
- Suggestions for improvement
- Testing recommendations

Respond:
- Promptly (within 1 business day)
- Professionally
- With explanation of your reasoning
- With willingness to iterate

## ⚠️ What NOT to Do

### Never:
- ❌ Commit API keys or credentials
- ❌ Test in production without extensive staging tests
- ❌ Make changes without documentation
- ❌ Deploy on Friday afternoon (no weekend support)
- ❌ Delete workflows without backups
- ❌ Ignore errors or warnings
- ❌ Bypass the review process for "quick fixes"
- ❌ Make assumptions about data - validate everything

### Always:
- ✅ Test thoroughly before deploying
- ✅ Document your changes
- ✅ Communicate with the team
- ✅ Have a rollback plan
- ✅ Monitor after deployment
- ✅ Learn from mistakes
- ✅ Ask questions when unsure
- ✅ Prioritize business continuity

## 🆘 Getting Help

### When Stuck:

1. **Check documentation** (start here!)
2. **Review similar workflows** (how was it done before?)
3. **Search Make.com docs** (official documentation)
4. **Ask the team** (we're here to help)
5. **Contact support** (Make.com, Monday, QuickBooks, etc.)

### Questions Welcome

No question is too basic. It's better to ask than to:
- Break something
- Make incorrect assumptions
- Waste time troubleshooting alone

### Sharing Knowledge

When you learn something useful:
- Document it
- Share with the team
- Update relevant guides
- Consider adding to CLAUDE_CONTEXT.md

## 🎓 Learning Resources

### Make.com
- Official Docs: https://www.make.com/en/help
- University: https://www.make.com/en/academy
- Community: https://www.make.com/en/community

### Shipping APIs
- Usually have sandbox environments
- Test thoroughly before using production keys
- Read rate limit documentation
- Understand error codes

### Git/GitHub
- Basic tutorial: https://guides.github.com/
- Commit best practices
- Branching strategies

## 💡 Tips for Success

1. **Start small**: Make incremental changes
2. **Test often**: Catch issues early
3. **Document immediately**: Don't wait until later
4. **Ask questions**: Better to clarify than assume
5. **Learn continuously**: Technology changes
6. **Be patient**: Complex systems take time to understand
7. **Communicate proactively**: Keep team informed
8. **Think about future you**: Write code you'll understand in 6 months

## 🙏 Recognition

Great contributions include:
- Critical bug fixes
- Performance improvements
- Better documentation
- Helpful code reviews
- Knowledge sharing
- Process improvements

We appreciate all contributions that make our system better!

---

**Questions about these guidelines?** Ask in Monday.com or team channel.

**Last Updated:** 2026-02-08
