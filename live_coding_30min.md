# Technical Interview Prep: Live Coding Enhancements

## Your Implementation Analysis

### Architecture Strengths
✅ **Service-oriented design** - Clean separation with ActivateSubscriptionService, RecurringBillingService, CancelSubscriptionService, ReactivateSubscriptionService, ReconcilePaymentService  
✅ **Normalized data model** - Separate `invoice_line_items` table  
✅ **Anniversary billing** - Tracks `last_billing_date` for precise monthly/annual cycles  
✅ **Widget tracking** - `widget_usage` on subscriptions, `widget_allocation` on plans, `widget_usage_histories` for historical data  
✅ **Status enums** - Subscription (pending/active/cancelled/expired), Invoice (pending/paid/overdue/cancelled)  
✅ **Comprehensive validations** - Model-level and custom validations  
✅ **Billing period tracking** - `billing_period_start` and `billing_period_end` on invoices

### Missing Features (Clues for Interview)
❌ **No proration logic** - Plan changes would bill full amounts  
❌ **No overage billing** - Usage beyond allocation isn't charged  
❌ **No trial periods** - Direct activation only  
❌ **No grace periods** - Overdue invoices don't suspend access  
❌ **No refund/credit system** - Cancellations don't generate credits  
❌ **No pause/resume** - Only cancel/reactivate (creates new invoice)  
❌ **No multi-currency** - Single currency only  
❌ **No discount/coupon system**

---

## Top 10 Live Coding Enhancements (30 minutes each)

Ranked by likelihood based on: (1) Natural extensions of your implementation, (2) Real-world SaaS billing needs, (3) Testable in 30 minutes

### 🥇 TIER 1: HIGHEST PROBABILITY

#### 1. **Usage-Based Overage Billing**
**Why #1:** Already hinted in PRD ("Usage Tracking and Overage Calculation"), you have all the infrastructure (widget_allocation, widget_usage, invoice_line_items).

**What they'll ask:**
> "Extend the system to charge $5 per widget when usage exceeds the plan allocation. Overages should appear as line items on the next invoice."

**Implementation (25-30 min):**

```ruby
# Add to Plan model
add_column :plans, :overage_price, :decimal, precision: 10, scale: 2, default: 0

# Modify RecurringBillingService
def create_line_items(invoice)
  # Base subscription fee
  invoice.line_items.create!(
    description: "#{@subscription.plan.name} - #{billing_frequency_name} Billing",
    amount: calculate_base_amount
  )
  
  # Overage charges if any
  if has_overage_usage?
    overage_amount = calculate_overage_amount
    invoice.line_items.create!(
      description: "Overage Usage - #{overage_widgets} widgets",
      amount: overage_amount
    )
  end
  
  # Update total
  invoice.update!(amount: invoice.line_items.sum(:amount))
end

private

def has_overage_usage?
  @subscription.widget_usage > @subscription.plan.widget_allocation
end

def overage_widgets
  @subscription.widget_usage - @subscription.plan.widget_allocation
end

def calculate_overage_amount
  overage_widgets * @subscription.plan.overage_price
end

# Add tests
# test/services/recurring_billing_service_test.rb
test "generates overage line item when usage exceeds allocation" do
  subscription = subscriptions(:monthly_active_over_limit)
  subscription.plan.update(overage_price: 5.00)
  subscription.update(widget_usage: 150) # allocation is 100
  
  service = RecurringBillingService.new(subscription)
  result = service.generate_invoice
  
  assert result[:success]
  invoice = result[:invoice]
  assert_equal 2, invoice.line_items.count
  
  overage_item = invoice.line_items.find_by("description LIKE '%Overage%'")
  assert_equal 250.00, overage_item.amount # 50 widgets * $5
  assert_equal 349.99 + 250.00, invoice.amount # base + overage
end
```

**Why they test this:**
- Tests your understanding of the existing codebase
- Line items architecture was clearly built for this
- Real SaaS billing pattern
- Straightforward but requires careful testing

---

#### 2. **Proration Logic for Mid-Cycle Plan Changes**
**Why #2:** Common SaaS feature, tests date math and money handling, natural next step after basic billing.

**What they'll ask:**
> "Add ability to upgrade from a $99 Basic plan to a $199 Pro plan mid-cycle. Calculate prorated credit for unused Basic time and prorated charge for Pro time until next anniversary."

**Implementation (30 min):**

```ruby
# Create new service
# app/services/change_subscription_plan_service.rb
class ChangeSubscriptionPlanService
  def initialize(subscription, new_plan, effective_date: Date.today)
    @subscription = subscription
    @new_plan = new_plan
    @old_plan = subscription.plan
    @effective_date = effective_date
  end

  def call
    return failure("Cannot change plan") unless can_change_plan?

    ActiveRecord::Base.transaction do
      credit_amount = calculate_prorated_credit
      charge_amount = calculate_prorated_charge
      net_amount = charge_amount - credit_amount
      
      if net_amount > 0
        create_upgrade_invoice(net_amount, credit_amount, charge_amount)
      end
      
      @subscription.update!(plan: @new_plan)
      { success: true, subscription: @subscription }
    end
  rescue => e
    failure(e.message)
  end

  private

  def can_change_plan?
    @subscription.active? && @new_plan.id != @old_plan.id
  end

  def calculate_prorated_credit
    return 0 unless monthly_subscription?
    
    days_used = (@effective_date - @subscription.last_billing_date).to_i
    days_in_period = days_in_billing_period
    days_unused = days_in_period - days_used
    
    (@old_plan.base_price * days_unused / days_in_period).round(2)
  end

  def calculate_prorated_charge
    return 0 unless monthly_subscription?
    
    days_remaining = (@subscription.last_billing_date + 1.month - @effective_date).to_i
    days_in_period = days_in_billing_period
    
    (@new_plan.base_price * days_remaining / days_in_period).round(2)
  end

  def create_upgrade_invoice(net_amount, credit, charge)
    invoice = @subscription.invoices.create!(
      amount: net_amount,
      status: "pending",
      due_date: Date.today + 7,
      billing_period_start: @effective_date,
      billing_period_end: @subscription.last_billing_date + 1.month - 1.day
    )

    # Credit line item
    invoice.line_items.create!(
      description: "Credit for unused #{@old_plan.name} time",
      amount: -credit
    )

    # Charge line item
    invoice.line_items.create!(
      description: "Prorated #{@new_plan.name} charge",
      amount: charge
    )
  end

  def monthly_subscription?
    @subscription.billing_frequency.name == "Monthly"
  end

  def days_in_billing_period
    (@subscription.last_billing_date + 1.month - @subscription.last_billing_date).to_i
  end

  def failure(message)
    { success: false, error: message }
  end
end

# Migration
rails g migration AddProrationSupport
# Add any needed fields if necessary

# Test
test "prorates correctly when upgrading mid-cycle" do
  subscription = subscriptions(:monthly_active)
  subscription.update(last_billing_date: Date.new(2025, 1, 1))
  
  new_plan = plans(:premium)
  effective_date = Date.new(2025, 1, 16) # Halfway through month
  
  service = ChangeSubscriptionPlanService.new(subscription, new_plan, effective_date: effective_date)
  result = service.call
  
  assert result[:success]
  # Verify credit and charge calculations
end
```

**Why they test this:**
- Date math proficiency
- Money handling edge cases
- Understanding of service object patterns you established
- Real-world complexity

---

#### 3. **Grace Period for Overdue Invoices**
**Why #3:** You already have `overdue` status in Invoice enum, natural to implement behavior. Tests state management.

**What they'll ask:**
> "Add a 7-day grace period after invoice due date. If still unpaid after grace period, suspend widget access (don't allow usage increments beyond current amount)."

**Implementation (25 min):**

```ruby
# Migration
add_column :subscriptions, :suspended, :boolean, default: false

# Invoice model
def grace_period_expired?
  overdue? && due_date < Date.today - 7.days
end

# Subscription model
def suspend_for_non_payment!
  update!(suspended: true)
end

def restore_access!
  update!(suspended: false)
end

def can_consume_widgets?
  active? && !suspended?
end

# New job or rake task
class CheckOverdueInvoicesJob
  def perform
    Invoice.overdue.find_each do |invoice|
      if invoice.grace_period_expired?
        subscription = invoice.subscription
        subscription.suspend_for_non_payment! unless subscription.suspended?
      end
    end
  end
end

# Validation on Subscription
validate :cannot_exceed_allocation_when_suspended

def cannot_exceed_allocation_when_suspended
  return unless suspended?
  if widget_usage_changed? && widget_usage > widget_usage_was
    errors.add(:widget_usage, "cannot increase while subscription is suspended")
  end
end

# Tests
test "suspends subscription after grace period expires" do
  invoice = invoices(:overdue_past_grace_period)
  subscription = invoice.subscription
  
  CheckOverdueInvoicesJob.new.perform
  
  assert subscription.reload.suspended?
end

test "cannot increase widget usage when suspended" do
  subscription = subscriptions(:suspended_subscription)
  subscription.widget_usage = subscription.widget_usage + 10
  
  assert_not subscription.valid?
  assert_includes subscription.errors[:widget_usage], "cannot increase while subscription is suspended"
end

test "restores access when overdue invoice is paid" do
  subscription = subscriptions(:suspended_subscription)
  invoice = subscription.invoices.overdue.first
  
  ReconcilePaymentService.new(invoice).mark_as_paid
  
  assert_not subscription.reload.suspended?
end
```

**Why they test this:**
- State management (suspended flag)
- Business logic around dates
- Validation edge cases
- Background job patterns (you mentioned Sidekiq in design doc)

---

### 🥈 TIER 2: MODERATE PROBABILITY

#### 4. **Trial Period Implementation**
**Why #4:** Common SaaS pattern, tests subscription lifecycle, no billing during trial but limits still apply.

**Implementation (25-30 min):**

```ruby
# Migration
add_column :subscriptions, :trial_end_date, :date

# Subscription model
def in_trial?
  trial_end_date.present? && Date.today <= trial_end_date
end

def trial_expired?
  trial_end_date.present? && Date.today > trial_end_date
end

# Modify ActivateSubscriptionService
def call
  return failure("Subscription must be pending") unless @subscription.pending?

  ActiveRecord::Base.transaction do
    if in_trial_period?
      activate_trial
    else
      activate_with_invoice
    end
  end
end

def in_trial_period?
  @subscription.trial_end_date.present? && @subscription.trial_end_date >= Date.today
end

def activate_trial
  @subscription.update!(
    status: "active",
    start_date: Date.today,
    last_billing_date: @subscription.trial_end_date # Billing starts after trial
  )
  { success: true, subscription: @subscription, trial: true }
end

def activate_with_invoice
  # ... existing logic
end

# RecurringBillingService - skip trial subscriptions
def self.generate_invoices_for_due_subscriptions
  subscriptions_due = Subscription.active
    .where("last_billing_date < ?", Date.today)
    .where("trial_end_date IS NULL OR trial_end_date < ?", Date.today) # Skip trials
    .includes(:plan, :billing_frequency, :account)
  # ...
end

# Test
test "activates subscription without invoice during trial" do
  subscription = subscriptions(:pending_with_trial)
  subscription.trial_end_date = 14.days.from_now.to_date
  
  service = ActivateSubscriptionService.new(subscription)
  result = service.call
  
  assert result[:success]
  assert result[:trial]
  assert subscription.reload.active?
  assert_equal 0, subscription.invoices.count
end

test "first invoice generated after trial ends" do
  subscription = subscriptions(:trial_expiring_today)
  subscription.update(trial_end_date: Date.today, last_billing_date: Date.today)
  
  service = RecurringBillingService.new(subscription)
  result = service.generate_invoice
  
  assert result[:success]
  assert_not_nil result[:invoice]
end
```

**Why they test this:**
- Conditional billing logic
- Date-based business rules
- Integration with existing service pattern

---

#### 5. **Subscription Pause/Resume**
**Why #5:** You have cancel/reactivate, but pause is different (billing stops, anniversary adjusts). Tests date manipulation.

**Implementation (30 min):**

```ruby
# Add new status to subscription enum
enum :status, {
  pending: "pending",
  active: "active",
  paused: "paused",    # NEW
  cancelled: "cancelled",
  expired: "expired"
}

# Migration
add_column :subscriptions, :paused_at, :date
add_column :subscriptions, :pause_duration_days, :integer, default: 0

# New service
class PauseSubscriptionService
  def initialize(subscription)
    @subscription = subscription
  end

  def call
    return failure("Can only pause active subscriptions") unless @subscription.active?

    ActiveRecord::Base.transaction do
      @subscription.update!(
        status: "paused",
        paused_at: Date.today
      )
    end

    { success: true, subscription: @subscription }
  rescue => e
    failure(e.message)
  end

  private
  def failure(msg); { success: false, error: msg }; end
end

class ResumeSubscriptionService
  def initialize(subscription)
    @subscription = subscription
  end

  def call
    return failure("Can only resume paused subscriptions") unless @subscription.paused?

    ActiveRecord::Base.transaction do
      pause_duration = (Date.today - @subscription.paused_at).to_i
      
      # Adjust anniversary date by pause duration
      new_last_billing_date = @subscription.last_billing_date + pause_duration.days
      
      @subscription.update!(
        status: "active",
        pause_duration_days: @subscription.pause_duration_days + pause_duration,
        last_billing_date: new_last_billing_date,
        paused_at: nil
      )
    end

    { success: true, subscription: @subscription }
  rescue => e
    failure(e.message)
  end

  private
  def failure(msg); { success: false, error: msg }; end
end

# RecurringBillingService - exclude paused
def self.generate_invoices_for_due_subscriptions
  subscriptions_due = Subscription.active  # Only active, not paused
    .where("last_billing_date < ?", Date.today)
    # ...
end

# Test
test "pausing and resuming adjusts anniversary date" do
  subscription = subscriptions(:monthly_active)
  original_billing_date = subscription.last_billing_date
  
  # Pause
  PauseSubscriptionService.new(subscription).call
  assert subscription.reload.paused?
  
  # Simulate 10 days passing
  travel 10.days do
    # Resume
    ResumeSubscriptionService.new(subscription).call
    
    assert subscription.reload.active?
    assert_equal original_billing_date + 10.days, subscription.last_billing_date
  end
end
```

**Why they test this:**
- Date arithmetic
- State transitions
- Business logic around billing cycles

---

#### 6. **Cancellation with Prorated Refund**
**Why #6:** You have cancel service, natural to add refund calculation.

**Implementation (25-30 min):**

```ruby
# Modify CancelSubscriptionService
def call
  return failure("Subscription cannot be cancelled") unless @subscription.can_be_cancelled?

  ActiveRecord::Base.transaction do
    cancel_subscription
    
    if monthly_subscription? && should_issue_refund?
      create_refund_credit
    end
  end

  { success: true, subscription: @subscription, refund: @refund_amount }
rescue => e
  failure(e.message)
end

private

def should_issue_refund?
  # Only refund if last invoice was paid and we're mid-cycle
  last_invoice = @subscription.invoices.paid.order(created_at: :desc).first
  last_invoice.present? && Date.today < next_billing_date
end

def create_refund_credit
  days_in_period = days_in_billing_period
  days_unused = (next_billing_date - Date.today).to_i
  
  @refund_amount = (@subscription.plan.base_price * days_unused / days_in_period).round(2)
  
  # Create credit line item on a refund invoice
  refund_invoice = @subscription.invoices.create!(
    amount: -@refund_amount,  # Negative = credit
    status: "paid",
    due_date: Date.today,
    billing_period_start: Date.today,
    billing_period_end: next_billing_date - 1.day
  )
  
  refund_invoice.line_items.create!(
    description: "Prorated refund for early cancellation",
    amount: -@refund_amount
  )
end

def next_billing_date
  @subscription.last_billing_date + 1.month
end

def days_in_billing_period
  (next_billing_date - @subscription.last_billing_date).to_i
end

def monthly_subscription?
  @subscription.billing_frequency.name == "Monthly"
end

# Test
test "cancellation mid-cycle generates prorated refund" do
  subscription = subscriptions(:monthly_active)
  subscription.update(last_billing_date: Date.new(2025, 1, 1))
  
  # Cancel halfway through
  travel_to Date.new(2025, 1, 16) do
    service = CancelSubscriptionService.new(subscription)
    result = service.call
    
    assert result[:success]
    assert result[:refund].present?
    
    refund_invoice = subscription.invoices.order(created_at: :desc).first
    assert refund_invoice.amount < 0  # Credit
  end
end
```

---

### 🥉 TIER 3: POSSIBLE BUT LESS LIKELY

#### 7. **Multi-Currency Support**
**Why #7:** Real feature but requires more setup (currency exchange rates, etc). Good if they want to test Rails conventions.

**Implementation (30 min):**

```ruby
# Migration
add_column :plans, :currency, :string, default: "USD", null: false
add_column :invoices, :currency, :string, default: "USD", null: false

# Plan model
SUPPORTED_CURRENCIES = %w[USD EUR GBP CAD]
validates :currency, inclusion: { in: SUPPORTED_CURRENCIES }

# Invoice model
validates :currency, inclusion: { in: Plan::SUPPORTED_CURRENCIES }
validate :currency_matches_subscription

def currency_matches_subscription
  if currency != subscription.plan.currency
    errors.add(:currency, "must match subscription plan currency")
  end
end

# Modify RecurringBillingService
def create_invoice
  amount = calculate_amount
  
  @subscription.invoices.create!(
    amount: amount,
    currency: @subscription.plan.currency,  # NEW
    status: "pending",
    # ...
  )
end

# Views (if they want you to show it)
<%= number_to_currency(invoice.amount, unit: currency_symbol(invoice.currency)) %>

def currency_symbol(currency)
  { "USD" => "$", "EUR" => "€", "GBP" => "£", "CAD" => "CA$" }[currency]
end

# Test
test "invoice inherits currency from plan" do
  plan = plans(:basic)
  plan.update!(currency: "EUR", base_price: 99.00)
  subscription = create_subscription_with_plan(plan)
  
  service = ActivateSubscriptionService.new(subscription)
  result = service.call
  
  invoice = result[:invoice]
  assert_equal "EUR", invoice.currency
end
```

---

#### 8. **Invoice Reminder System**
**Why #8:** Service object pattern, query optimization. May be too trivial.

**Implementation (20 min):**

```ruby
# Service
class InvoiceReminderService
  def self.find_invoices_needing_reminder(days_before_due: 3)
    reminder_date = Date.today + days_before_due.days
    
    Invoice.pending
      .where(due_date: reminder_date)
      .includes(subscription: [:account, :plan])
      .order(:due_date)
  end

  def initialize(invoice)
    @invoice = invoice
  end

  def generate_reminder_data
    {
      account_name: @invoice.subscription.account.business_name,
      invoice_id: @invoice.id,
      amount: @invoice.amount,
      due_date: @invoice.due_date,
      plan_name: @invoice.subscription.plan.name,
      days_until_due: (@invoice.due_date - Date.today).to_i
    }
  end
end

# Test
test "finds invoices due in 3 days" do
  invoice_due_in_3_days = invoices(:pending_invoice)
  invoice_due_in_3_days.update(due_date: 3.days.from_now.to_date)
  
  reminders = InvoiceReminderService.find_invoices_needing_reminder(days_before_due: 3)
  
  assert_includes reminders, invoice_due_in_3_days
end
```

---

#### 9. **Account Credit System**
**Why #9:** Real feature but requires new model. Good test of associations.

**Implementation (30 min):**

```ruby
# Migration
rails g model AccountCredit account:references amount:decimal description:string applied:boolean applied_to_invoice:references

# Model
class AccountCredit < ApplicationRecord
  belongs_to :account
  belongs_to :applied_to_invoice, class_name: "Invoice", optional: true
  
  validates :amount, presence: true, numericality: { greater_than: 0 }
  validates :description, presence: true
  
  scope :unapplied, -> { where(applied: false) }
end

# Account model
has_many :account_credits, dependent: :destroy

def available_credit_balance
  account_credits.unapplied.sum(:amount)
end

# Modify RecurringBillingService
def create_invoice
  base_amount = calculate_amount
  available_credit = @subscription.account.available_credit_balance
  
  amount_after_credit = [base_amount - available_credit, 0].max
  
  invoice = @subscription.invoices.create!(
    amount: amount_after_credit,
    # ...
  )
  
  if available_credit > 0
    apply_credits_to_invoice(invoice, [available_credit, base_amount].min)
  end
  
  invoice
end

def apply_credits_to_invoice(invoice, credit_amount)
  credits = @subscription.account.account_credits.unapplied.order(created_at: :asc)
  remaining = credit_amount
  
  credits.each do |credit|
    break if remaining <= 0
    
    amount_to_apply = [credit.amount, remaining].min
    credit.update!(applied: true, applied_to_invoice: invoice)
    remaining -= amount_to_apply
  end
end
```

---

#### 10. **Usage Analytics Query**
**Why #10:** Tests SQL/ActiveRecord. May be too straightforward.

**Implementation (25 min):**

```ruby
# Service
class UsageAnalyticsService
  def self.subscription_usage_stats(time_range: 3.months.ago..Date.today)
    Subscription.active
      .joins(:widget_usage_histories)
      .where(widget_usage_histories: { usage_date: time_range })
      .group("subscriptions.id")
      .select(
        "subscriptions.id",
        "subscriptions.account_id",
        "AVG(widget_usage_histories.widget_usage) as avg_usage",
        "MAX(widget_usage_histories.widget_usage) as peak_usage",
        "COUNT(widget_usage_histories.id) as billing_cycles"
      )
  end

  def self.top_consumers(limit: 10, time_range: 3.months.ago..Date.today)
    subscription_usage_stats(time_range: time_range)
      .order("avg_usage DESC")
      .limit(limit)
  end

  def self.usage_trend_by_month(subscription)
    subscription.widget_usage_histories
      .select(
        "DATE_TRUNC('month', usage_date) as month",
        "AVG(widget_usage) as avg_usage"
      )
      .group("DATE_TRUNC('month', usage_date)")
      .order("month DESC")
  end
end

# Test
test "calculates average usage across subscriptions" do
  subscription = subscriptions(:monthly_active)
  create_usage_history(subscription, usage: 50, date: 30.days.ago)
  create_usage_history(subscription, usage: 70, date: 60.days.ago)
  
  stats = UsageAnalyticsService.subscription_usage_stats
  subscription_stat = stats.find { |s| s.id == subscription.id }
  
  assert_equal 60, subscription_stat.avg_usage
  assert_equal 70, subscription_stat.peak_usage
end
```

---

## Interview Strategy

### What They're Testing
1. **Understanding of existing code** - Can you navigate and extend your own architecture?
2. **Rails conventions** - Migrations, validations, associations, service objects
3. **Business logic** - Billing math, date handling, state transitions
4. **Testing** - Writing meaningful tests for new features
5. **Time management** - Completing in 30 minutes with quality code

### Approach for Live Coding

**First 5 minutes:**
- Clarify requirements
- Identify which models/services need changes
- Outline the approach

**Next 20 minutes:**
- Write migration if needed
- Update model with validations
- Update/create service object
- Add basic test

**Last 5 minutes:**
- Run test
- Walk through your solution
- Discuss edge cases you'd handle with more time

### Key Talking Points During Implementation

**Your Strengths:**
- "I already have the infrastructure for this with the invoice_line_items table..."
- "Following the service object pattern I established, I'll create..."
- "This leverages the widget_usage_histories table I built for historical tracking..."
- "The enum status pattern makes this straightforward..."

**Show Trade-offs:**
- "In 30 minutes I'm focusing on X, but in production I'd also consider Y..."
- "This uses a simple calculation; for scale we might want to cache this..."
- "I'm storing this as a column, but we could denormalize if performance requires..."

**Testing Mindset:**
- "Let me write a test that covers the happy path and one edge case..."
- "This validation ensures data integrity at the model level..."
- "The transaction here is critical because..."

---

## Questions to Ask Them

1. "Should I prioritize completeness or demonstrate testing practices?"
2. "Are there any specific validation edge cases you'd like me to focus on?"
3. "Would you like me to implement the job/rake task, or focus on the service logic?"
4. "For the migration, should I show the up AND down methods, or is structure sufficient?"

---

## Final Preparation Checklist

- [ ] Review RecurringBillingService logic (most likely to be extended)
- [ ] Practice date arithmetic and proration formulas
- [ ] Know your fixtures/test data cold
- [ ] Rehearse service object pattern explanation
- [ ] Practice running tests in terminal
- [ ] Have a mental model of your DB schema
- [ ] Prepare "in production I would..." statements

---

## Most Likely: Top 3 Predictions

1. **Overage Billing** (65% confidence) - PRD hints, infrastructure ready
2. **Proration** (55% confidence) - Common SaaS, tests skills
3. **Grace Period** (45% confidence) - Natural extension of existing status

**Dark Horse:** Trial Periods (35%) - If they want to see conditional activation logic

Good luck! Your implementation is solid and you have all the building blocks they need to test extensions naturally.
