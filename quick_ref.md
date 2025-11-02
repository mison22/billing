# Omada Interview Quick Reference Card

## Your Architecture in 60 Seconds

**Models:**
- `Account` → `Subscription` → `Invoice` → `InvoiceLineItem`
- `Plan` (has `widget_allocation`, `base_price`)
- `BillingFrequency` (Monthly/Annual)
- `WidgetUsageHistory` (tracks usage snapshots)

**Key Services:**
- `ActivateSubscriptionService` - First activation + invoice
- `RecurringBillingService` - Anniversary billing
- `CancelSubscriptionService` - Set end_date, status = cancelled
- `ReactivateSubscriptionService` - Resume cancelled subscription
- `ReconcilePaymentService` - Mark paid + reset widget_usage

**Status Flows:**
- Subscription: `pending → active → cancelled`
- Invoice: `pending → paid` OR `pending → overdue`

**Key Fields:**
- `subscriptions.last_billing_date` - Anniversary tracking
- `subscriptions.widget_usage` - Current cycle usage
- `invoices.billing_period_start/end` - What period invoice covers
- `plans.widget_allocation` - Max allowed per cycle

---

## Top 3 Most Likely Extensions

### 1. Overage Billing
**Add to RecurringBillingService:**
```ruby
def create_line_items(invoice)
  # Base fee
  invoice.line_items.create!(description: "...", amount: base_amount)
  
  # Overage if > allocation
  if @subscription.widget_usage > @subscription.plan.widget_allocation
    overage = @subscription.widget_usage - @subscription.plan.widget_allocation
    invoice.line_items.create!(
      description: "Overage - #{overage} widgets",
      amount: overage * @subscription.plan.overage_price
    )
  end
  
  invoice.update!(amount: invoice.line_items.sum(:amount))
end
```

### 2. Plan Change Proration
**New Service:**
```ruby
class ChangeSubscriptionPlanService
  def call
    credit = calc_credit_for_unused_time  # (days_left / days_in_period) * old_price
    charge = calc_charge_for_new_plan     # (days_left / days_in_period) * new_price
    net = charge - credit
    
    create_invoice(net) if net > 0
    @subscription.update!(plan: @new_plan)
  end
end
```

### 3. Grace Period for Overdue
**Add to Subscription:**
```ruby
add_column :subscriptions, :suspended, :boolean, default: false

validate :cannot_increase_usage_when_suspended

def cannot_increase_usage_when_suspended
  if suspended? && widget_usage_changed? && widget_usage > widget_usage_was
    errors.add(:widget_usage, "cannot increase while suspended")
  end
end
```

---

## Interview Flow (30 min)

**0-5 min: Clarify & Plan**
- "So I need to add X feature... let me confirm the requirements..."
- "I'll need to modify [Service], add [field], and test [scenario]"
- Draw quick diagram if helpful

**5-25 min: Code**
1. Migration (if needed) - 3 min
2. Model changes (validations) - 5 min
3. Service object logic - 12 min
4. Basic test - 5 min

**25-30 min: Test & Explain**
- Run the test
- Walk through the logic
- Mention edge cases you'd handle with more time

---

## Key Phrases to Use

**Leverage Your Code:**
- "I already have invoice_line_items set up for exactly this..."
- "Following the service pattern I established..."
- "This fits naturally with my widget_usage_histories table..."

**Show Expertise:**
- "In production, I'd also consider [X]..."
- "For this 30-minute scope, I'm focusing on the happy path..."
- "This transaction ensures atomicity when..."
- "The validation here prevents data corruption by..."

**Testing Mindset:**
- "Let me write a test for the core logic first..."
- "I'm testing both the calculation and the state change..."
- "This edge case would need handling: [X]..."

---

## Common Pitfalls to Avoid

❌ Don't overthink - 30 minutes is tight
❌ Don't forget the migration if you add columns
❌ Don't skip validations - they'll ask about edge cases
❌ Don't write tests after time expires - write as you go
✅ DO focus on business logic correctness
✅ DO explain your thought process out loud
✅ DO use your existing patterns consistently

---

## Date Math Quick Reference

**Proration (monthly):**
```ruby
days_used = (today - last_billing_date).to_i
days_in_period = (next_billing_date - last_billing_date).to_i
days_unused = days_in_period - days_used

credit = (old_price * days_unused / days_in_period).round(2)
```

**Anniversary Billing:**
```ruby
billing_period_start = last_billing_date + 1.day
billing_period_end = billing_period_start + 1.month - 1.day
```

**Annual:**
```ruby
billing_period_end = start_date + 1.year - 1.day
amount = base_price * 12 * 0.8  # 20% discount
```

---

## Your Database Schema (Quick Reference)

```
subscriptions:
  - plan_id, billing_frequency_id, account_id
  - status (pending/active/cancelled/expired)
  - start_date, end_date
  - last_billing_date  ← anniversary tracking
  - widget_usage       ← current cycle usage

invoices:
  - subscription_id
  - amount, status (pending/paid/overdue/cancelled)
  - due_date, paid_date
  - billing_period_start, billing_period_end

invoice_line_items:
  - invoice_id
  - description, amount

plans:
  - name, base_price
  - widget_allocation  ← max widgets per cycle

widget_usage_histories:
  - subscription_id
  - usage_date, widget_usage
  - billing_period_start, billing_period_end
```

---

## Test Patterns You Use

**Service Tests:**
```ruby
test "service does X when Y" do
  subscription = subscriptions(:monthly_active)
  subscription.update!(widget_usage: 150)  # Over limit
  
  service = MyService.new(subscription)
  result = service.call
  
  assert result[:success]
  assert_equal expected_value, actual_value
end
```

**Model Tests:**
```ruby
test "validation prevents X" do
  subscription = subscriptions(:basic)
  subscription.widget_usage = 999  # Exceeds allocation
  
  assert_not subscription.valid?
  assert_includes subscription.errors[:widget_usage], "cannot exceed"
end
```

---

## What Success Looks Like

✅ Feature works correctly for happy path
✅ At least one meaningful test passes
✅ Edge cases mentioned in discussion
✅ Code follows your existing patterns
✅ You can explain trade-offs made

Remember: They know you have limited time. They're testing how you think and code under pressure, not perfection.

---

## Pre-Interview Mental Checklist

- [ ] Can draw your data model from memory
- [ ] Know the flow: ActivateSubscriptionService → RecurringBillingService → ReconcilePaymentService
- [ ] Remember: widget_usage resets on PAYMENT not invoice generation
- [ ] Recall: last_billing_date tracks anniversary for monthly
- [ ] Know your fixtures cold (what test data exists)

**Breathe. You built this. You know it cold. Show them how you extend it.**
