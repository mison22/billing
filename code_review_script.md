# Code Review Presentation (5-10 Minutes)

## Opening (30 seconds)
"I'll walk you through my implementation at three levels: architecture, key design decisions, and how the billing flow works. I built a service-oriented billing system that handles subscription lifecycle, anniversary-based billing, and widget usage tracking."

---

## Part 1: Architecture Overview (2 minutes)

### Data Model (Show schema.rb or draw quick diagram)
**"Let me start with the data model:"**

```
Account
  └─→ Subscription (status: pending → active → cancelled)
       ├─→ Plan (base_price, widget_allocation)
       ├─→ BillingFrequency (Monthly/Annual)
       ├─→ Invoices (amount, status, billing_period_start/end)
       │    └─→ InvoiceLineItems (description, amount)
       └─→ WidgetUsageHistories (snapshot per billing cycle)
```

**Key Points (30 seconds each):**

1. **"Normalized invoice line items"** - Separate table instead of JSON gives us flexibility for multiple charges per invoice, supports overage billing in the future, and enables reporting.

2. **"Anniversary tracking with last_billing_date"** - This field ensures monthly subscriptions bill on the same day each month (e.g., 15th to 15th), critical for the PRD requirement.

3. **"Widget usage history"** - Captures snapshot before reset each cycle, enables trend analysis and audit trails.

---

## Part 2: Service Objects (3 minutes)

**"I extracted all business logic into five service objects following Rails best practices:"**

### 1. ActivateSubscriptionService (30 sec)
"Handles initial subscription activation. Creates first invoice based on billing frequency - monthly gets base price, annual gets 12 months with 20% discount. Sets the subscription to active and establishes the billing cycle start."

**Show code snippet:**
```ruby
# Key logic
def call
  invoice = create_invoice
  @subscription.update!(status: "active", start_date: Date.today)
  { success: true, invoice: invoice }
end
```

### 2. RecurringBillingService (60 sec) ⭐ MOST IMPORTANT
"This is the heart of the billing system. It runs daily to find subscriptions where `last_billing_date < today` and generates anniversary invoices."

**Show code snippet:**
```ruby
def generate_invoice
  # Creates invoice with proper billing period
  # Records usage history snapshot
  # Updates last_billing_date for next cycle
  # Usage reset happens in ReconcilePaymentService
end
```

**Key design choice:** 
"Notice I split monthly and annual into separate class methods - `generate_monthly_invoices` and `generate_annual_invoices`. This allows different scheduling in production with Sidekiq. You can run monthly checks daily and annual checks weekly, with separate monitoring."

**Anniversary calculation:**
```ruby
billing_period_start = last_billing_date + 1.day
billing_period_end = start + 1.month - 1.day
# Example: Jan 15 → Feb 14, then Feb 15 → Mar 14
```

### 3. ReconcilePaymentService (30 sec)
"Marks invoices as paid and resets widget usage. Important: usage reset happens here on payment, not at invoice generation. This ensures customers who haven't paid yet continue using their current cycle's allocation."

### 4 & 5. Cancel/Reactivate (30 sec)
"CancelSubscriptionService sets end_date and status. ReactivateSubscriptionService has smart logic - if reactivating an ended subscription, it creates a new paid invoice to restart the billing cycle. If reactivating a future-dated cancellation, it just clears the end_date."

---

## Part 3: Critical Design Decisions (2 minutes)

### 1. Billing Period Tracking
**"Every invoice has billing_period_start and billing_period_end."**

Why: PRD requires this, enables accurate proration in the future, supports reporting.

```ruby
validates :billing_period_start, presence: true
validates :billing_period_end, presence: true
```

### 2. Widget Usage Flow
**"Three-part system for widgets:"**

1. `Plan.widget_allocation` - defines the limit (e.g., 100 widgets)
2. `Subscription.widget_usage` - current cycle usage (external service updates this)
3. `WidgetUsageHistory` - snapshots captured at invoice generation

**Validation prevents exceeding allocation:**
```ruby
def widget_usage_within_allocation
  if widget_usage > plan.widget_allocation
    errors.add(:widget_usage, "cannot exceed allocation")
  end
end
```

**"This system doesn't track consumption - external service does that. We just store the total and enforce limits via validations."**

### 3. Money Handling
**"All amounts are decimal(10, 2) and I round to 2 places."**
```ruby
annual_price = (base_price * 12 * 0.8).round(2)
```

**Line items must sum to invoice amount:**
```ruby
def amount_matches_line_items
  if (amount - line_items.sum(:amount)).abs > 0.01
    errors.add(:amount, "does not match sum of line items")
  end
end
```

### 4. Service Response Pattern
**"All services return explicit success/failure hashes - no exceptions for business logic failures."**
```ruby
{ success: true, invoice: invoice }
# or
{ success: false, error: "descriptive message" }
```

**Why:** Makes error handling explicit in controllers, easier to test, prevents exception-driven control flow.

---

## Part 4: Testing Approach (1 minute)

**"I focused on three types of tests:"**

1. **Service tests** - Core business logic (invoice generation, calculations)
2. **Model tests** - Validations and associations
3. **Integration test** - End-to-end checkout flow

**Show one test example:**
```ruby
test "generates monthly invoice with correct billing period" do
  subscription = subscriptions(:monthly_active)
  subscription.update(last_billing_date: Date.new(2025, 1, 15))
  
  service = RecurringBillingService.new(subscription)
  result = service.generate_invoice
  
  assert result[:success]
  invoice = result[:invoice]
  assert_equal Date.new(2025, 1, 16), invoice.billing_period_start
  assert_equal Date.new(2025, 2, 14), invoice.billing_period_end
end
```

---

## Closing (30 seconds)

**"Three key takeaways from my implementation:"**

1. **Service-oriented design** keeps controllers thin and business logic testable
2. **Anniversary billing** via `last_billing_date` ensures correct monthly cycles
3. **Extensible architecture** - normalized line items and widget history support future features like overage billing and proration

**"The system handles the complete subscription lifecycle: create → activate → recurring billing → payment → cancel → reactivate. What questions do you have?"**

---

## Visual Aids to Prepare

### Option 1: Draw on screen share while talking
```
┌─────────────────────────────────────────────────┐
│ SUBSCRIPTION LIFECYCLE                           │
├─────────────────────────────────────────────────┤
│                                                  │
│  pending  ──activate──→  active  ──cancel──→  cancelled │
│              │             │  ↑                  │
│              │             │  │                  │
│           Invoice      Recurring     Reactivate │
│           Created       Billing                  │
│                           │                      │
│                      Update Usage                │
│                      Generate Invoice            │
│                                                  │
└─────────────────────────────────────────────────┘
```

### Option 2: Key Files to Reference
Open these in tabs for easy reference:
1. `app/models/subscription.rb` - Status enum, validations
2. `app/services/recurring_billing_service.rb` - Core billing logic
3. `db/schema.rb` - Data model
4. `test/services/recurring_billing_service_test.rb` - Testing approach

---

## Pro Tips for Delivery

### DO:
✅ **Start high-level, drill down** - Architecture → Services → Specific logic
✅ **Highlight decisions, not just code** - "I chose X because Y"
✅ **Show, don't tell** - Open files, point to specific lines
✅ **Connect to PRD requirements** - "This satisfies the anniversary billing requirement..."
✅ **Mention production considerations** - "In production, this would run via Sidekiq..."
✅ **Be confident about your choices** - You made good decisions

### DON'T:
❌ **Don't scroll through entire files** - Jump to key methods
❌ **Don't apologize for code** - It's solid
❌ **Don't get lost in details** - Stay at the right altitude
❌ **Don't rush** - 5-10 minutes is plenty
❌ **Don't hide trade-offs** - "I focused on X, could add Y later"

---

## Anticipated Questions & Answers

### "Why separate table for invoice line items instead of JSON?"
"Flexibility and queryability. We can filter by line item type, sum amounts, and easily add overage charges later. JSON would require parsing and doesn't support native Rails associations."

### "How do you handle February 29th or month-end billing?"
"Ruby's Date class handles this naturally. `Date.new(2024, 1, 31) + 1.month` returns Feb 29 (leap year) or Feb 28. The `last_billing_date` field preserves the original day, and I calculate billing periods relative to that."

### "What if widget usage increases while invoice is unpaid?"
"That's allowed currently - usage only resets when invoice is paid via ReconcilePaymentService. The external service that tracks consumption would need to check if usage is at the limit before allowing more consumption. A future enhancement would be suspending access for overdue accounts."

### "How would you add overage billing?"
"Perfect fit for my architecture. Add `overage_price` to Plan model. In RecurringBillingService, check if `widget_usage > widget_allocation`, calculate overage amount, and create a second line item. The line items table is designed for exactly this."

### "What about plan upgrades mid-cycle?"
"Would need proration. Calculate credit for unused time on old plan, charge for new plan prorated to next anniversary. Create invoice with two line items - credit and charge. This leverages the flexible line items structure I built."

### "Why service objects instead of model callbacks?"
"Testability and clarity. Services make the flow explicit - you can see exactly when invoices are created. Callbacks are implicit and can create hard-to-debug chains. Service objects also make it easy to orchestrate transactions and return explicit success/failure."

---

## Time Management

- **0:00-0:30** - Opening
- **0:30-2:30** - Architecture overview
- **2:30-5:30** - Service objects walkthrough
- **5:30-7:30** - Critical design decisions
- **7:30-8:30** - Testing approach
- **8:30-9:00** - Closing + invite questions

**If running short on time:** Skip detailed testing section, focus on architecture + RecurringBillingService

**If they're engaged:** Let them guide with questions, be ready to show any file

---

## Checklist Before Review

- [ ] Have code editor open with key files in tabs
- [ ] Can draw architecture diagram quickly if needed
- [ ] Rehearsed the flow once (don't memorize, just know the beats)
- [ ] Prepared 1-2 "trade-off" statements for major decisions
- [ ] Ready to answer "how would you add X feature"
- [ ] Confident body language - this is YOUR code, you know it

**Remember:** You built a solid, well-architected system. Show them the thoughtful decisions you made, not just the syntax. They're evaluating your judgment, not looking for perfection.

Go get it! 🚀
