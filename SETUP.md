# Setup Guide

This guide walks you through setting up the shipping automation system from scratch or recovering from a catastrophic failure.

## 🎯 Prerequisites

Before beginning setup, ensure you have:

### Accounts & Access
- [ ] Make.com account (Team plan or higher recommended)
- [ ] Monday.com workspace with admin access
- [ ] QuickBooks Online account
- [ ] Shipping carrier account(s) with API access
- [ ] GitHub account (for this repository)

### Credentials Needed
- [ ] Monday.com API token
- [ ] QuickBooks OAuth credentials
- [ ] Shipping carrier API keys
- [ ] Email SMTP credentials (for notifications)

### Knowledge Requirements
- [ ] Basic understanding of Make.com workflows
- [ ] Familiarity with Monday.com boards
- [ ] QuickBooks invoice creation process
- [ ] Understanding of shipping workflows

## 📋 Setup Steps

### Phase 1: Repository Setup

#### 1.1 Clone Repository
```bash
git clone [your-repository-url]
cd shipping-automation-system
```

#### 1.2 Review Documentation
Read in this order:
1. README.md (system overview)
2. CLAUDE_CONTEXT.md (detailed architecture)
3. RULES.md (governance)
4. This file (SETUP.md)

### Phase 2: Monday.com Configuration

#### 2.1 Create/Verify Boards

You'll need boards for:
- **Orders** - Main order management
- **Shipments** - Shipping tracking
- **Returns** (optional) - RMA management

#### 2.2 Required Columns

**Orders Board** should have:
- Order ID (text)
- Customer Name (text)
- Shipping Address (location or text)
- Shipping Method (dropdown: Ground, Priority, Overnight, etc.)
- Booking Status (status: Pending, Booked, Shipped, Delivered)
- Tracking Number (text)
- Shipping Cost (numbers)
- Invoice ID (text)

**Shipments Board** should have:
- Shipment ID (text)
- Order Reference (text)
- Carrier (dropdown)
- Service Level (dropdown)
- Label URL (link)
- Tracking Number (text)
- Status (status: Label Created, Picked Up, In Transit, Delivered)

#### 2.3 Setup Webhooks

For each workflow trigger:
1. Go to board settings
2. Navigate to Integrations
3. Add webhook for column value changes
4. Note webhook IDs for Make.com configuration

#### 2.4 Get API Token

1. Click your profile picture → Admin
2. Go to API section
3. Generate API token
4. **Save securely** - you'll need this for Make.com

### Phase 3: Make.com Setup

#### 3.1 Create Account/Organization

If not already done:
1. Sign up at make.com
2. Create organization for your business
3. Invite team members

#### 3.2 Add Connections

**Monday.com Connection:**
1. In Make.com, go to Connections
2. Click "Add Connection" → Monday
3. Enter API token from Phase 2.4
4. Test connection

**QuickBooks Connection:**
1. Add Connection → QuickBooks Online
2. Follow OAuth flow
3. Grant necessary permissions
4. Test connection

**Shipping Carrier Connection(s):**

*For EasyPost:*
1. Add Connection → HTTP (custom)
2. Configure for EasyPost API
3. Add API key
4. Test with sample request

*For ShipEngine:*
1. Add Connection → ShipEngine
2. Enter API key
3. Test connection

*For Direct Carrier APIs:*
- Follow carrier-specific setup
- Configure authentication
- Test connection

#### 3.3 Import Workflows

For each blueprint JSON file:

1. **Import Blueprint**
   - In Make.com, go to Scenarios
   - Click "..." menu → Import Blueprint
   - Upload JSON file
   - Make.com will create the scenario

2. **Configure Connections**
   - Each module will need connections assigned
   - Select the connections created in 3.2
   - Verify all modules have valid connections

3. **Update Webhooks**
   - Replace placeholder webhook IDs with real ones
   - Get webhook IDs from Monday.com (Phase 2.3)
   - Update in the trigger module

4. **Verify Settings**
   - Check all dropdown selections
   - Verify field mappings match your Monday boards
   - Confirm error handlers are configured

5. **Save as Draft**
   - Don't enable yet
   - Save as draft for testing

#### 3.4 Workflow Import Order

Import in this sequence (dependencies):

1. **First: Rate/Quote Requests**
   - `Shipping_2.0_-_Rate_Request_Trigger_blueprint.json`
   - `Blindshipping_2.0_-_Label_Quote_Request_blueprint.json`

2. **Second: Label Booking**
   - `Shipping_2.0_-_Book___Print_Label_blueprint.json`
   - `BlindShipping_2.0_-_Book___Send_Label_blueprint.json`

3. **Third: Invoice Creation**
   - `Shipping_2.0_Create_Shipping_Invoice_based_off_cost_blueprint.json`
   - `BlindShipping_2_0_Create_Shipping_Invoice_based_off_cost_blueprint.json`

4. **Fourth: Status Updates**
   - `Shipping_2.0_Mark_shipments_as_fulfilled_when_picked_up_blueprint.json`
   - `BlindShipping_2_0_Mark_shipments_as_fulfilled_when_picked_up_blueprint.json`

5. **Fifth: Tracking Notifications**
   - `Shipping_2.0_Send_Tracking_blueprint.json`
   - `BlindShipping_2_0_Send_Tracking_blueprint.json`

6. **Finally: QuickBooks Sync**
   - `Invoice_Paid_-_Quickbooks_System_Check_p1_blueprint.json`
   - `Invoice_Paid_-_Quickbooks_System_Check_p2.json`

### Phase 4: Testing

#### 4.1 Create Test Data

In Monday.com:
1. Create test board or use test group
2. Add sample order with realistic data
3. Use test addresses (your office, etc.)
4. Mark clearly as TEST

#### 4.2 Test Each Workflow

**Testing Protocol:**

1. **Enable One Workflow** (start with rate request)
2. **Trigger It** (change column value in Monday)
3. **Watch Execution** in Make.com
4. **Verify Output**:
   - Correct data transformations
   - Expected API calls made
   - Proper data written back to Monday
   - Error handling works
5. **Check Logs** for any warnings
6. **Document Issues** found
7. **Fix and Retest** until perfect

#### 4.3 Integration Testing

After individual workflow tests:

1. **End-to-End Test**:
   - Create test order
   - Request rate quote
   - Book label
   - Verify invoice created
   - Simulate pickup
   - Verify tracking sent

2. **Error Scenario Testing**:
   - Invalid address
   - API timeout
   - Missing data
   - Duplicate request

3. **Volume Testing**:
   - Process 10 orders in sequence
   - Check for timeouts or bottlenecks
   - Verify all complete successfully

### Phase 5: Production Deployment

#### 5.1 Pre-Launch Checklist

- [ ] All workflows tested individually
- [ ] End-to-end test successful
- [ ] Error handling verified
- [ ] Team trained on system
- [ ] Documentation complete
- [ ] Rollback plan prepared
- [ ] Monitoring configured
- [ ] Test data cleaned up

#### 5.2 Gradual Rollout

**Week 1: Shadow Mode**
- Enable workflows
- Run alongside manual process
- Compare results
- Fix any discrepancies

**Week 2: Partial Automation**
- Automate rate requests only
- Manual label booking
- Monitor closely

**Week 3: Full Automation (non-peak)**
- Enable all workflows
- Start with low-volume days
- Have team ready to intervene

**Week 4: Full Production**
- Complete automation enabled
- Reduce manual oversight
- Establish normal monitoring

#### 5.3 Go-Live Day

On launch day:

1. **Morning**: Enable remaining workflows
2. **Monitor**: Watch every execution first few hours
3. **Communication**: Team aware and standing by
4. **Quick Response**: Fix issues immediately
5. **Document**: Note any unexpected behaviors
6. **Review**: End of day team review

### Phase 6: Ongoing Maintenance

#### 6.1 Daily Tasks
- Check error logs
- Verify all orders processing
- Monitor execution times
- Respond to alerts

#### 6.2 Weekly Tasks
- Review error patterns
- Check API rate limits
- Verify invoice accuracy
- Team sync meeting

#### 6.3 Monthly Tasks
- Full system audit
- Update documentation
- Review metrics
- Plan improvements

## 🚨 Troubleshooting Setup Issues

### Common Issues

**"Connection Failed" in Make.com**
- Verify API credentials are correct
- Check if API token has necessary permissions
- Confirm IP whitelist if applicable
- Test API directly with Postman/curl

**"Webhook Not Triggering"**
- Verify webhook URL is correct in Monday
- Check webhook is active
- Test webhook manually
- Check Make.com scenario is enabled

**"Data Not Mapping Correctly"**
- Verify Monday.com column names match blueprint
- Check field types match expectations
- Review mapper module configuration
- Test with sample data

**"Workflow Times Out"**
- Check API response times
- Look for inefficient loops
- Review error handler configurations
- Consider breaking into smaller workflows

### Getting Help

If stuck during setup:

1. **Check Documentation**: Review CLAUDE_CONTEXT.md
2. **Search Make.com Community**: Others may have solved it
3. **Contact Support**: Make.com, Monday, or carrier support
4. **Team Discussion**: Brainstorm with team
5. **Professional Help**: Consider hiring Make.com expert

## 📊 Success Criteria

Setup is complete when:

- [ ] All workflows imported and configured
- [ ] Connections tested and working
- [ ] Test orders process end-to-end
- [ ] Error handling verified
- [ ] Team trained
- [ ] Documentation updated
- [ ] Monitoring in place
- [ ] Production orders processing automatically

## 🎉 Post-Setup

After successful setup:

1. **Document**: Update CHANGELOG.md
2. **Celebrate**: Recognize team effort
3. **Monitor**: Watch closely for first week
4. **Optimize**: Identify improvement opportunities
5. **Share**: Document lessons learned

## 📞 Support Contacts

- Make.com Support: support.make.com
- Monday.com Support: support.monday.com
- QuickBooks Support: quickbooks.intuit.com/learn-support
- Repository Issues: [Your GitHub Issues URL]

---

**Estimated Setup Time:** 2-3 days for experienced team, 1-2 weeks for new team

**Last Updated:** 2026-02-08
