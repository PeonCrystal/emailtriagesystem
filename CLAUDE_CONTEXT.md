# Claude Context - Shipping Automation System

This document provides context for AI assistants (particularly Claude) working with this shipping automation system. It explains the system architecture, business logic, and how to effectively help with maintenance, troubleshooting, and enhancements.

## 🎯 System Purpose

This is a **production e-commerce shipping automation system** serving:
- **dadsprinting.com** - established printing business
- **proteamjerseys.com** - growing Shopify-based team jersey/apparel business

The owner is an entrepreneur managing a small but growing business umbrella. These automations are **mission-critical** for daily operations.

## 🏗️ System Architecture

### Platform: Make.com (formerly Integromat)
Make.com is a visual automation platform that connects apps and services. The blueprints in this repo are JSON exports of visual workflow scenarios.

### Core Integration Triangle
```
Monday.com (Order Management)
     ↕️
Make.com (Automation Engine)
     ↕️
QuickBooks (Financial) ↔ Shipping APIs (Fulfillment)
```

### Two Parallel Systems

**1. Standard Shipping (Shipping_2.0_*.json)**
- Direct fulfillment from warehouse
- Customer sees actual shipper information
- Standard B2C e-commerce flow

**2. Blind Shipping (BlindShipping_2.0_*.json)**
- Dropshipping/3PL scenarios
- Supplier information hidden from end customer
- Return address shows retailer, not actual shipper
- More complex workflow with additional masking steps

## 📋 Workflow Components

### 1. Rate Request Workflows
- **Trigger**: New order or customer requests shipping quote
- **Process**: Query multiple carriers for rates
- **Output**: Display options to customer or auto-select best rate
- **Files**: `*Rate_Request*.json`, `*Label_Quote_Request*.json`

### 2. Label Booking Workflows
- **Trigger**: Shipping method selected/confirmed
- **Process**: 
  - Book chosen rate with carrier
  - Generate shipping label (PDF)
  - Store label URL and tracking number
- **Output**: Label ready for printing, tracking number captured
- **Files**: `*Book___Send_Label*.json`, `*Book___Print_Label*.json`

### 3. Invoice Creation Workflows
- **Trigger**: Label successfully booked
- **Process**:
  - Calculate actual shipping cost
  - Create invoice in QuickBooks
  - Link invoice to order
- **Output**: Invoice in QuickBooks, cost tracked
- **Files**: `*Create_Shipping_Invoice*.json`

### 4. Fulfillment Update Workflows
- **Trigger**: Package picked up by carrier / tracking event
- **Process**:
  - Update Monday.com order status
  - Mark as fulfilled
  - Archive or move to "shipped" group
- **Output**: Order status reflects shipment
- **Files**: `*Mark_shipments_as_fulfilled*.json`

### 5. Tracking Notification Workflows
- **Trigger**: Tracking number becomes active
- **Process**:
  - Format customer notification email
  - Send tracking link to customer
  - Update customer communication log
- **Output**: Customer receives tracking info
- **Files**: `*Send_Tracking*.json`

### 6. QuickBooks Payment Verification
- **Trigger**: Invoice payment status changes
- **Process**: 
  - Verify payment received
  - Update order financial status
  - Trigger post-payment workflows if needed
- **Output**: Synchronized payment status
- **Files**: `Invoice_Paid_-_Quickbooks_System_Check*.json`

## 🔍 Blueprint JSON Structure

Each blueprint file contains:
```json
{
  "name": "Workflow Name",
  "flow": [
    {
      "id": 1,
      "module": "service:action",
      "mapper": { /* field mappings */ },
      "metadata": { /* UI position, labels */ }
    }
  ],
  "metadata": { /* overall scenario config */ }
}
```

### Key Sections:
- **`name`**: Human-readable workflow name
- **`flow`**: Array of modules (steps) in execution order
- **`module`**: Service and action (e.g., "monday:watchBoardColumnValues")
- **`mapper`**: Data transformations and field mappings
- **`metadata.designer`**: Visual position in Make.com interface

## 💡 How to Assist Effectively

### When Asked About Workflows:

1. **Ask clarifying questions** about which workflow (standard vs blind shipping)
2. **Identify the trigger point** - what starts this automation?
3. **Trace the data flow** - what data comes in, how is it transformed?
4. **Check error handling** - what happens when something fails?
5. **Consider business impact** - failed labels = angry customers

### When Debugging Issues:

1. **Identify the module/step** having issues
2. **Check the mapper** - is data being transformed correctly?
3. **Verify connections** - are API credentials current?
4. **Look for filters** - is the workflow being skipped due to conditions?
5. **Check webhooks** - is Monday.com actually triggering the workflow?

### When Enhancing the System:

1. **Understand current flow** completely before suggesting changes
2. **Consider edge cases** - what if address is invalid? what if carrier API is down?
3. **Think about rollback** - how to undo if enhancement causes issues?
4. **Document changes** - update this file and README
5. **Suggest testing strategy** - how to verify without impacting production?

## 🎨 Business Logic Patterns

### Rate Shopping
The system likely compares multiple carrier options and selects based on:
- Cost (cheapest)
- Speed (fastest)
- Service level (ground vs priority)
- Business rules (preferred carriers)

### Address Masking (Blind Shipping)
For blind shipping, the system must:
1. Use actual recipient address for delivery
2. Use retailer address for "ship from" display
3. Ensure return labels go to retailer, not supplier
4. Handle carrier-specific blind shipping requirements

### Invoice Timing
Invoices are created **after** label booking because:
- Actual cost only known after booking
- Avoids invoicing for failed bookings
- Matches accounting to actual shipping expense

### Status Synchronization
Monday.com is the "source of truth" for order status:
- QuickBooks tracks financial status
- Monday.com tracks operational status
- Make.com keeps them synchronized

## 🚨 Critical Failure Points

### High-Priority Issues (Fix Immediately):
1. **Labels not generating** - orders can't ship
2. **Tracking not sending** - customer dissatisfaction
3. **Duplicate invoices** - financial reconciliation problems
4. **Wrong addresses** - packages to wrong customers

### Medium-Priority Issues (Fix Within 24h):
1. **Rate quotes not returning** - manual rate lookup needed
2. **Status not updating** - manual status changes needed
3. **Payment sync delays** - accounting impact

### Low-Priority Issues (Can Wait):
1. **Cosmetic issues** in notifications
2. **Non-critical data field updates**
3. **Enhancement requests**

## 🔧 Common Assistance Scenarios

### "Why isn't my workflow triggering?"
- Check Monday.com webhook configuration
- Verify the column being watched for changes
- Check if filters are excluding the order
- Confirm webhook hasn't been disabled

### "Labels are generating with wrong address"
- Check address field mapping in mapper
- Verify Monday.com column names match blueprint
- Check for address validation module
- Look for "ship to" vs "bill to" confusion

### "Invoices have wrong amounts"
- Check shipping cost calculation logic
- Verify cost source (actual charge vs quoted rate)
- Check for additional fees (fuel surcharge, residential, etc.)
- Confirm currency handling

### "How do I add a new carrier?"
- Identify carrier API integration in Make.com
- Add new API connection
- Clone existing rate request module
- Add to carrier comparison logic
- Test thoroughly before production

### "Can we automate [X]?"
- Understand the manual process first
- Identify trigger event
- Map required data transformations
- Consider error scenarios
- Estimate development time

## 📚 Useful Resources

### Make.com Documentation
- Module reference: https://www.make.com/en/help/modules
- Functions: https://www.make.com/en/help/functions
- Error handling: https://www.make.com/en/help/errors

### Monday.com API
- Webhooks: https://developer.monday.com/api-reference/docs/webhooks
- Column types: https://developer.monday.com/api-reference/docs/column-types

### QuickBooks API
- Invoices: https://developer.intuit.com/app/developer/qbo/docs/api/accounting/all-entities/invoice

### Shipping APIs (Common)
- EasyPost: https://www.easypost.com/docs/api
- ShipEngine: https://www.shipengine.com/docs/
- ShipStation: https://www.shipstation.com/docs/api/

## 🎓 Learning the System

### For New Team Members:
1. Start with `*Rate_Request*.json` - simplest workflow
2. Move to `*Book___Send_Label*.json` - core functionality
3. Study `*Create_Shipping_Invoice*.json` - QuickBooks integration
4. Understand difference between Standard and Blind shipping
5. Review error handling in each workflow

### For AI Assistants:
1. Always read the relevant blueprint JSON before answering questions
2. Consider the business context - this is production e-commerce
3. Be conservative with changes - suggest testing approaches
4. When uncertain, explain what you'd need to know to give better answer
5. Remember: downtime = lost revenue and angry customers

## 🔮 Future Development Considerations

### Likely Enhancement Requests:
- Multi-warehouse support (route to nearest warehouse)
- International shipping compliance
- Return merchandise authorization (RMA) automation
- Shipping analytics and reporting
- Batch processing for high-volume days
- Predictive shipping (pre-purchase labels)

### Technical Debt to Address:
- Duplicate workflows (e.g., multiple `Send_Tracking` versions)
- Hardcoded values that should be variables
- Error notifications (ensure team knows when automation fails)
- Backup/disaster recovery strategy
- Version control for blueprint changes

## 🤖 Working With This Repository

### When Making Changes:
1. Export updated blueprint from Make.com as JSON
2. Replace corresponding file in this repo
3. Update README.md if functionality changed
4. Update this CLAUDE_CONTEXT.md if architecture/logic changed
5. Document the change reason and date

### When Seeking Help:
1. Specify which workflow file
2. Describe expected vs actual behavior
3. Include error messages if any
4. Note what troubleshooting has been attempted
5. Indicate urgency level

### When Optimizing:
1. Identify bottleneck (slow module, expensive API calls)
2. Consider impact on other workflows
3. Test with production-like data volumes
4. Have rollback plan ready
5. Monitor closely after deployment

---

**Remember:** This system is not just code - it's the operational backbone of real businesses serving real customers. Treat it with the care and respect it deserves.

**Last Updated:** 2026-02-08  
**Document Version:** 1.0
