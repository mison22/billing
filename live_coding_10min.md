# Live Coding - 10 Minute Challenges

## ⏱️ Context: Short Live Coding Session

**Time Available:** 10-20 minutes (likely closer to 10)  
**Format:** Small feature addition to your take-home  
**Goal:** See you code without AI, verify understanding of your codebase  
**Expectation:** Working code with basic test, not perfection

---

## 🎯 Top 10 Challenges (Ranked by Probability)

### Probability Scale:
- 🔥🔥🔥 = Very High (60-80%)
- 🔥🔥 = High (40-60%)
- 🔥 = Moderate (20-40%)

---

## 1. Add Validation to Prevent Invalid State 🔥🔥🔥

### 📋 Prompt
"Add validation to the Subscription model that prevents canceling a subscription if there are any unpaid invoices (pending or overdue)."

### ⏱️ Time: 8 minutes
- 3 min: Add validation
- 3 min: Write test
- 2 min: Run test & verify

### ✅ Solution

**Step 1: Add validation to Subscription model (3 min)**

```ruby
# app/models/subscription.rb
validate :cannot_cancel_with_unpaid_invoices, on: :update

private

def cannot_cancel_with_unpaid_invoices
  return unless status_changed? && status == "cancelled"
  
  if invoices.pending.exists? || invoices.overdue.exists?
    errors.add(:status, "cannot cancel subscription with unpaid invoices")
  end
end
```

**Step 2: Write test (3 min)**

```ruby
# test/models/subscription_test.rb
test "cannot cancel subscription with unpaid invoices" do
  subscription = subscriptions(:monthly_active)
  
  # Create unpaid invoice
  subscription.invoices.create!(
    amount: 99.99,
    status: "pending",
    due_date: Date.today + 14.days,
    billing_period_start: Date.today,
    billing_period_end: Date.today + 1.month
  )
  
  subscription.status = "cancelled"
  
  assert_not subscription.valid?
  assert_includes subscription.errors[:status], "cannot cancel subscription with unpaid invoices"
end

test "can cancel subscription with no invoices" do
  subscription = subscriptions(:monthly_active)
  subscription.status = "cancelled"
  subscription.end_date = Date.today
  
  assert subscription.valid?
end

test "can cancel subscription with all invoices paid" do
  subscription = subscriptions(:monthly_active_with_paid_invoice)
  subscription.status = "cancelled"
  subscription.end_date = Date.today
  
  assert subscription.valid?
end
```

**Step 3: Run test (2 min)**
```bash
rails test test/models/subscription_test.rb:15
```

### 💡 Why This Is #1
- Simple scope (one model, one validation)
- Tests your understanding of validations
- Tests your understanding of associations
- Realistic business requirement
- Easy to verify it works

---

## 2. Add Scope/Query Method 🔥🔥🔥

### 📋 Prompt
"Add a scope or method to the Subscription model that returns all subscriptions that are due for billing today (active status, last_billing_date is in the past, not ended)."

### ⏱️ Time: 7 minutes
- 3 min: Add scope
- 3 min: Write test
- 1 min: Verify

### ✅ Solution

**Step 1: Add scope (3 min)**

```ruby
# app/models/subscription.rb
scope :due_for_billing, -> {
  active
    .where("last_billing_date < ?", Date.today)
    .where("end_date IS NULL OR end_date > ?", Date.today)
}

# Alternative: class method
def self.due_for_billing
  active
    .where("last_billing_date < ?", Date.today)
    .where("end_date IS NULL OR end_date > ?", Date.today)
end
```

**Step 2: Write test (3 min)**

```ruby
# test/models/subscription_test.rb
test "due_for_billing includes subscriptions past billing date" do
  subscription = subscriptions(:monthly_active)
  subscription.update!(last_billing_date: 2.days.ago)
  
  assert_includes Subscription.due_for_billing, subscription
end

test "due_for_billing excludes subscriptions with future billing date" do
  subscription = subscriptions(:monthly_active)
  subscription.update!(last_billing_date: 2.days.from_now)
  
  assert_not_includes Subscription.due_for_billing, subscription
end

test "due_for_billing excludes ended subscriptions" do
  subscription = subscriptions(:monthly_active)
  subscription.update!(
    last_billing_date: 2.days.ago,
    end_date: 1.day.ago,
    status: "cancelled"
  )
  
  assert_not_includes Subscription.due_for_billing, subscription
end

test "due_for_billing excludes non-active subscriptions" do
  subscription = subscriptions(:pending_subscription)
  subscription.update!(last_billing_date: 2.days.ago)
  
  assert_not_includes Subscription.due_for_billing, subscription
end
```

**Step 3: Verify (1 min)**
```bash
rails test test/models/subscription_test.rb:25
```

### 💡 Why This Is #2
- Common query pattern
- Tests ActiveRecord knowledge
- Simple enough for 10 minutes
- Multiple edge cases to consider
- Directly related to your RecurringBillingService

---

## 3. Add Simple Calculation Method 🔥🔥

### 📋 Prompt
"Add a method to the Subscription model called `widgets_remaining` that returns how many widgets are left in the current cycle (allocation minus usage)."

### ⏱️ Time: 6 minutes
- 2 min: Add method
- 3 min: Write tests
- 1 min: Verify

### ✅ Solution

**Step 1: Add method (2 min)**

```ruby
# app/models/subscription.rb
def widgets_remaining
  return 0 if widget_usage >= plan.widget_allocation
  plan.widget_allocation - widget_usage
end

# Or with safeguards
def widgets_remaining
  allocation = plan&.widget_allocation || 0
  usage = widget_usage || 0
  remaining = allocation - usage
  [remaining, 0].max # Never return negative
end
```

**Step 2: Write tests (3 min)**

```ruby
# test/models/subscription_test.rb
test "widgets_remaining returns correct count" do
  subscription = subscriptions(:monthly_active)
  subscription.plan.update!(widget_allocation: 100)
  subscription.update!(widget_usage: 30)
  
  assert_equal 70, subscription.widgets_remaining
end

test "widgets_remaining returns 0 when at limit" do
  subscription = subscriptions(:monthly_active)
  subscription.plan.update!(widget_allocation: 100)
  subscription.update!(widget_usage: 100)
  
  assert_equal 0, subscription.widgets_remaining
end

test "widgets_remaining returns 0 when over limit" do
  subscription = subscriptions(:monthly_active)
  subscription.plan.update!(widget_allocation: 100)
  subscription.update!(widget_usage: 105)
  
  assert_equal 0, subscription.widgets_remaining
end
```

**Step 3: Verify (1 min)**
```bash
rails test test/models/subscription_test.rb:40
```

### 💡 Why This Is #3
- Very simple calculation
- Tests business logic understanding
- Safe for 10-minute window
- Shows you think about edge cases

---

## 4. Add Status Check Method 🔥🔥

### 📋 Prompt
"Add a method to the Subscription model called `at_risk?` that returns true if the subscription has any overdue invoices (not just pending, but actually overdue)."

### ⏱️ Time: 7 minutes
- 2 min: Add method
- 4 min: Write tests
- 1 min: Verify

### ✅ Solution

**Step 1: Add method (2 min)**

```ruby
# app/models/subscription.rb
def at_risk?
  invoices.overdue.exists?
end

# Or more sophisticated
def at_risk?
  invoices.where(status: "overdue")
          .where("due_date < ?", Date.today - 7.days) # 7 days overdue
          .exists?
end
```

**Step 2: Write tests (4 min)**

```ruby
# test/models/subscription_test.rb
test "at_risk? returns true when subscription has overdue invoice" do
  subscription = subscriptions(:monthly_active)
  subscription.invoices.create!(
    amount: 99.99,
    status: "overdue",
    due_date: 10.days.ago,
    billing_period_start: 1.month.ago,
    billing_period_end: 1.day.ago
  )
  
  assert subscription.at_risk?
end

test "at_risk? returns false when all invoices are paid" do
  subscription = subscriptions(:monthly_active_with_paid_invoice)
  
  assert_not subscription.at_risk?
end

test "at_risk? returns false when invoice is pending but not overdue" do
  subscription = subscriptions(:monthly_active)
  subscription.invoices.create!(
    amount: 99.99,
    status: "pending",
    due_date: 5.days.from_now,
    billing_period_start: Date.today,
    billing_period_end: 1.month.from_now
  )
  
  assert_not subscription.at_risk?
end
```

**Step 3: Verify (1 min)**
```bash
rails test test/models/subscription_test.rb:55
```

### 💡 Why This Is #4
- Simple boolean check
- Uses existing associations
- Common business logic pattern
- Good for 10 minutes

---

## 5. Add Before/After Callback 🔥🔥

### 📋 Prompt
"Add a callback to the Invoice model that automatically sets the `paid_date` to today when the status changes to 'paid'."

### ⏱️ Time: 8 minutes
- 3 min: Add callback
- 4 min: Write tests
- 1 min: Verify

### ✅ Solution

**Step 1: Add callback (3 min)**

```ruby
# app/models/invoice.rb
before_save :set_paid_date, if: :status_changed_to_paid?

private

def status_changed_to_paid?
  status_changed? && status == "paid"
end

def set_paid_date
  self.paid_date = Date.today if paid_date.blank?
end

# Alternative: after_commit
after_commit :set_paid_date, on: :update, if: :status_changed_to_paid?

private

def set_paid_date
  update_column(:paid_date, Date.today) if paid_date.blank?
end
```

**Step 2: Write tests (4 min)**

```ruby
# test/models/invoice_test.rb
test "automatically sets paid_date when status changes to paid" do
  invoice = invoices(:pending_invoice)
  assert_nil invoice.paid_date
  
  invoice.update!(status: "paid")
  
  assert_equal Date.today, invoice.reload.paid_date
end

test "does not override existing paid_date" do
  invoice = invoices(:pending_invoice)
  past_date = 5.days.ago.to_date
  invoice.update!(status: "paid", paid_date: past_date)
  
  assert_equal past_date, invoice.reload.paid_date
end

test "does not set paid_date when status is not paid" do
  invoice = invoices(:pending_invoice)
  invoice.update!(status: "overdue")
  
  assert_nil invoice.paid_date
end
```

**Step 3: Verify (1 min)**
```bash
rails test test/models/invoice_test.rb:10
```

### 💡 Why This Is #5
- Tests callback knowledge
- Common Rails pattern
- Straightforward implementation
- Good edge cases to consider

---

## 6. Add Custom Validation Logic 🔥

### 📋 Prompt
"Add validation to the Subscription model that ensures `widget_usage` cannot be increased if the subscription status is 'cancelled'."

### ⏱️ Time: 9 minutes
- 3 min: Add validation
- 5 min: Write tests
- 1 min: Verify

### ✅ Solution

**Step 1: Add validation (3 min)**

```ruby
# app/models/subscription.rb
validate :usage_cannot_increase_when_cancelled

private

def usage_cannot_increase_when_cancelled
  return unless cancelled?
  return unless widget_usage_changed?
  
  if widget_usage > widget_usage_was
    errors.add(:widget_usage, "cannot increase while subscription is cancelled")
  end
end
```

**Step 2: Write tests (5 min)**

```ruby
# test/models/subscription_test.rb
test "cannot increase widget_usage when cancelled" do
  subscription = subscriptions(:cancelled_subscription)
  subscription.update(widget_usage: 50)
  
  subscription.widget_usage = 60
  
  assert_not subscription.valid?
  assert_includes subscription.errors[:widget_usage], 
                  "cannot increase while subscription is cancelled"
end

test "can decrease widget_usage when cancelled" do
  subscription = subscriptions(:cancelled_subscription)
  subscription.update(widget_usage: 50)
  
  subscription.widget_usage = 40
  
  assert subscription.valid?
end

test "can increase widget_usage when active" do
  subscription = subscriptions(:monthly_active)
  subscription.update(widget_usage: 50)
  
  subscription.widget_usage = 60
  
  assert subscription.valid?
end
```

**Step 3: Verify (1 min)**
```bash
rails test test/models/subscription_test.rb:70
```

### 💡 Why This Is #6
- Tests conditional validation
- Uses `attribute_changed?` pattern
- Business logic enforcement
- Good test coverage needed

---

## 7. Add Association Counter Cache 🔥

### 📋 Prompt
"Add a counter cache to track the number of invoices on each subscription. Add the column and configure the association."

### ⏱️ Time: 10 minutes
- 3 min: Migration
- 2 min: Update association
- 4 min: Write tests
- 1 min: Verify

### ✅ Solution

**Step 1: Migration (3 min)**

```bash
rails g migration AddInvoicesCountToSubscriptions invoices_count:integer
```

```ruby
# db/migrate/[timestamp]_add_invoices_count_to_subscriptions.rb
class AddInvoicesCountToSubscriptions < ActiveRecord::Migration[8.0]
  def change
    add_column :subscriptions, :invoices_count, :integer, default: 0, null: false
    
    # Backfill existing data
    reversible do |dir|
      dir.up do
        Subscription.find_each do |subscription|
          Subscription.reset_counters(subscription.id, :invoices)
        end
      end
    end
  end
end
```

**Step 2: Update association (2 min)**

```ruby
# app/models/invoice.rb
belongs_to :subscription, counter_cache: true

# app/models/subscription.rb
# (association already exists, counter_cache works automatically)
```

**Step 3: Write tests (4 min)**

```ruby
# test/models/subscription_test.rb
test "invoices_count updates when invoice created" do
  subscription = subscriptions(:monthly_active)
  initial_count = subscription.invoices_count
  
  subscription.invoices.create!(
    amount: 99.99,
    status: "pending",
    due_date: Date.today + 14.days,
    billing_period_start: Date.today,
    billing_period_end: Date.today + 1.month
  )
  
  assert_equal initial_count + 1, subscription.reload.invoices_count
end

test "invoices_count updates when invoice destroyed" do
  subscription = subscriptions(:monthly_active_with_paid_invoice)
  initial_count = subscription.invoices_count
  
  subscription.invoices.first.destroy!
  
  assert_equal initial_count - 1, subscription.reload.invoices_count
end
```

**Step 4: Verify (1 min)**
```bash
rails db:migrate
rails test test/models/subscription_test.rb:85
```

### 💡 Why This Is #7
- Tests migration skills
- Common performance pattern
- Requires both migration and model change
- Good for 10 minutes if you know it

---

## 8. Add Enum Status 🔥

### 📋 Prompt
"Add a new status 'suspended' to the Subscription enum for subscriptions that are temporarily suspended due to non-payment."

### ⏱️ Time: 9 minutes
- 3 min: Update enum
- 2 min: Migration (if needed for existing data)
- 3 min: Write tests
- 1 min: Verify

### ✅ Solution

**Step 1: Update enum (3 min)**

```ruby
# app/models/subscription.rb
enum :status, {
  pending: "pending",
  active: "active",
  suspended: "suspended",  # NEW
  cancelled: "cancelled",
  expired: "expired"
}
```

**Step 2: Migration (2 min)**

```bash
# Only needed if you want to add check constraint
rails g migration AddSuspendedStatusToSubscriptions
```

```ruby
class AddSuspendedStatusToSubscriptions < ActiveRecord::Migration[8.0]
  def up
    # No schema change needed - enum values are stored as strings
    # This migration is optional, just for documentation
    
    # Could add check constraint if desired
    execute <<-SQL
      ALTER TABLE subscriptions
      ADD CONSTRAINT check_subscription_status
      CHECK (status IN ('pending', 'active', 'suspended', 'cancelled', 'expired'))
    SQL
  end
  
  def down
    execute "ALTER TABLE subscriptions DROP CONSTRAINT check_subscription_status"
  end
end
```

**Step 3: Write tests (3 min)**

```ruby
# test/models/subscription_test.rb
test "can set status to suspended" do
  subscription = subscriptions(:monthly_active)
  subscription.status = "suspended"
  
  assert subscription.valid?
  assert subscription.suspended?
end

test "suspended scope works" do
  subscription = subscriptions(:monthly_active)
  subscription.update!(status: "suspended")
  
  assert_includes Subscription.suspended, subscription
end

test "suspended? helper method works" do
  subscription = subscriptions(:monthly_active)
  subscription.update!(status: "suspended")
  
  assert subscription.suspended?
  assert_not subscription.active?
end
```

**Step 4: Verify (1 min)**
```bash
rails test test/models/subscription_test.rb:100
```

### 💡 Why This Is #8
- Simple enum addition
- Tests Rails enum knowledge
- Quick to implement
- Good test coverage possible

---

## 9. Add Class Method for Reporting 🔥

### 📋 Prompt
"Add a class method to the Invoice model called `total_revenue_for_month(year, month)` that returns the sum of all paid invoices for that month."

### ⏱️ Time: 9 minutes
- 3 min: Add method
- 5 min: Write tests
- 1 min: Verify

### ✅ Solution

**Step 1: Add method (3 min)**

```ruby
# app/models/invoice.rb
def self.total_revenue_for_month(year, month)
  start_date = Date.new(year, month, 1)
  end_date = start_date.end_of_month
  
  paid
    .where(paid_date: start_date..end_date)
    .sum(:amount)
end

# Alternative with more specificity
def self.total_revenue_for_month(year, month)
  paid.where(
    "EXTRACT(YEAR FROM paid_date) = ? AND EXTRACT(MONTH FROM paid_date) = ?",
    year, month
  ).sum(:amount)
end
```

**Step 2: Write tests (5 min)**

```ruby
# test/models/invoice_test.rb
test "total_revenue_for_month calculates correctly" do
  # Create paid invoices in November 2024
  invoice1 = invoices(:paid_invoice_1)
  invoice1.update!(amount: 100.00, paid_date: Date.new(2024, 11, 5))
  
  invoice2 = invoices(:paid_invoice_2)
  invoice2.update!(amount: 150.00, paid_date: Date.new(2024, 11, 20))
  
  # Create invoice in different month
  invoice3 = invoices(:paid_invoice_3)
  invoice3.update!(amount: 200.00, paid_date: Date.new(2024, 10, 15))
  
  revenue = Invoice.total_revenue_for_month(2024, 11)
  
  assert_equal 250.00, revenue
end

test "total_revenue_for_month excludes pending invoices" do
  invoice = invoices(:pending_invoice)
  invoice.update!(amount: 100.00, due_date: Date.new(2024, 11, 5))
  
  revenue = Invoice.total_revenue_for_month(2024, 11)
  
  assert_equal 0, revenue # pending invoices not counted
end

test "total_revenue_for_month returns zero when no invoices" do
  revenue = Invoice.total_revenue_for_month(2099, 1)
  
  assert_equal 0, revenue
end
```

**Step 3: Verify (1 min)**
```bash
rails test test/models/invoice_test.rb:15
```

### 💡 Why This Is #9
- Tests query/aggregation knowledge
- Common reporting pattern
- Date handling
- Good test scenarios

---

## 10. Add Index to Database 🔥

### 📋 Prompt
"Add a database index to improve the performance of queries that filter subscriptions by `last_billing_date`. Write a migration and explain why this helps."

### ⏱️ Time: 8 minutes
- 3 min: Write migration
- 2 min: Explain
- 2 min: Write test (verify index exists)
- 1 min: Run migration

### ✅ Solution

**Step 1: Write migration (3 min)**

```bash
rails g migration AddIndexToSubscriptionsLastBillingDate
```

```ruby
# db/migrate/[timestamp]_add_index_to_subscriptions_last_billing_date.rb
class AddIndexToSubscriptionsLastBillingDate < ActiveRecord::Migration[8.0]
  def change
    add_index :subscriptions, :last_billing_date, 
              name: "index_subscriptions_on_last_billing_date"
  end
end
```

**Step 2: Explain (2 min)**

Write in comments or be ready to explain:
```ruby
# Why this index helps:
#
# Before: Full table scan when running:
#   Subscription.where("last_billing_date < ?", Date.today)
#
# With index: Database can use B-tree to quickly find matching rows
#
# Common queries that benefit:
#   - RecurringBillingService.generate_invoices_for_due_subscriptions
#     Uses: WHERE last_billing_date < ?
#   - Finding subscriptions that need billing (daily job)
#   - Reporting on billing cycles
#
# Trade-off: 
#   - Faster reads
#   - Slightly slower writes (index must be updated)
#   - Small storage overhead
#
# For our use case (read-heavy billing queries), this is worth it.
```

**Step 3: Test index exists (2 min)**

```ruby
# test/models/subscription_test.rb
require 'test_helper'

class SubscriptionIndexTest < ActiveSupport::TestCase
  test "last_billing_date index exists" do
    indexes = ActiveRecord::Base.connection.indexes(:subscriptions)
    index = indexes.find { |i| i.columns == ["last_billing_date"] }
    
    assert index.present?, "Index on last_billing_date should exist"
  end
end
```

**Step 4: Run migration (1 min)**
```bash
rails db:migrate
rails test test/models/subscription_test.rb:115
```

### 💡 Why This Is #10
- Tests database knowledge
- Performance optimization
- Migration skills
- Good to explain trade-offs

---

## 🎯 Quick Reference: Time Management

### If You Get Each Challenge:

**2-3 minutes:** Clarify requirements
- "Should this prevent all cancellations or just for unpaid?"
- "What status counts as unpaid - pending, overdue, or both?"
- "Should I add a test?"

**3-5 minutes:** Write the code
- Model change
- Simple and correct
- Follow existing patterns

**3-5 minutes:** Write test
- Happy path
- 1-2 edge cases
- Existing test patterns

**1-2 minutes:** Run and verify
- `rails test path/to/test.rb:line`
- Fix if needed
- Explain what you did

---

## 📋 General Strategy for ANY 10-Minute Challenge

### Step 1: Clarify (1-2 min)
"Let me make sure I understand..."
- What exactly should happen?
- Any edge cases to consider?
- Should I write a test?

### Step 2: Locate (30 sec)
"I'll need to modify..."
- Which model/service?
- What file?
- (Open the file)

### Step 3: Implement (3-4 min)
"I'm going to..."
- Add validation/method/scope
- Follow existing code patterns
- Keep it simple

### Step 4: Test (3-4 min)
"Let me verify this works..."
- Write happy path test
- Add 1-2 edge case tests
- Use existing test structure

### Step 5: Verify (1 min)
"Let me run this..."
- Run the test
- Fix any issues
- Explain what you did

---

## 💡 Universal Patterns to Remember

### Validation Pattern:
```ruby
validate :method_name, if: :condition?
# or
validate :method_name, on: :update

private

def method_name
  if something_wrong
    errors.add(:field, "message")
  end
end
```

### Scope Pattern:
```ruby
scope :name, -> { where("condition") }
# or
scope :name, ->(param) { where("condition", param) }
```

### Method Pattern:
```ruby
def method_name
  # calculation or check
  result
end
```

### Test Pattern:
```ruby
test "describe what you're testing" do
  # Setup
  object = objects(:fixture)
  
  # Execute
  result = object.method_or_change
  
  # Assert
  assert_equal expected, result
  # or
  assert object.valid?
  assert_not object.valid?
  assert_includes collection, item
end
```

---

## ✅ Success Criteria for 10 Minutes

You succeed if you:
- [ ] Working code (doesn't have to be perfect)
- [ ] At least one test that passes
- [ ] Explained your thinking
- [ ] Stayed calm under pressure
- [ ] Followed existing code patterns

You DON'T need:
- ❌ Perfect edge case handling
- ❌ Comprehensive test coverage
- ❌ Production-ready code
- ❌ Optimized solution

---

## 🎯 Most Likely: Final Predictions

**Top 3 Predictions (75% confidence one of these):**
1. **Validation** - Prevent invalid state (most common)
2. **Scope/Query** - Find subscriptions due for billing
3. **Calculation Method** - widgets_remaining or similar

**Why These Are Most Likely:**
- ✅ Scoped to 10 minutes perfectly
- ✅ Test your understanding of the codebase
- ✅ Common Rails patterns
- ✅ Have clear success criteria
- ✅ Don't require new features, just small additions

**Dark Horse (15% confidence):**
- Status check method (`at_risk?`)
- Before/after callback

**Unlikely for 10 minutes:**
- Complex features (proration, overage billing)
- Multi-file changes
- Service object modifications
- Database schema changes (unless very simple)

---

## 📝 Prep Strategy

### Day Before:
- [ ] Review these top 3 challenges
- [ ] Practice one validation
- [ ] Practice one scope
- [ ] Practice one method with test

### Day Of:
- [ ] Have your test patterns fresh
- [ ] Know where your fixtures are
- [ ] Remember: simple > perfect

### During Interview:
- [ ] Listen carefully to requirements
- [ ] Ask clarifying questions
- [ ] Start with simplest solution
- [ ] Explain as you type
- [ ] Write test alongside code

---

## 🚀 You've Got This!

**Remember:**
- 10 minutes is SHORT - keep it simple
- They want to see you CODE, not perfect code
- Follow your existing patterns
- Test shows you care about correctness
- Explain your thinking out loud

**You already built this system. You know it better than anyone.**

Good luck! 💪
