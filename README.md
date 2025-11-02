# Omada Health Interview Preparation Guide

## 📚 Overview

This repository contains comprehensive guides to prepare you for the Omada Health Software Engineer II interview. The interview has three parts:

1. **System Design (45-60 minutes)** - Design an integration service on a whiteboard
2. **Code Review (5-10 minutes)** - Present your take-home implementation
3. **Live Coding (10-30 minutes)** - Small feature additions to your codebase without AI assistance

These guides will help you master all three parts efficiently.

---

## 🗂️ Files in This Repository

### 🎯 System Design (Part 1 - 45-60 min)

| File | Purpose | When to Use |
|------|---------|-------------|
| **[omada_system_design_scope.md](omada_system_design_scope.md)** (15KB) | System design scope analysis and framework | Read 2-3 days before for context and patterns |
| **[omada_practice_problems.md](omada_practice_problems.md)** (45KB) | 3 detailed system design practice problems with solutions | Practice 1-2 days before, work through problems |

### 📝 Code Review (Part 2 - 5-10 min)

| File | Purpose | When to Use |
|------|---------|-------------|
| **[omada_code_review_script.md](omada_code_review_script.md)** (12KB) | Step-by-step presentation script for code review | Study 2-3 days before, review morning of interview |

### 💻 Live Coding (Part 3 - 10-30 min)

| File | Purpose | When to Use |
|------|---------|-------------|
| **[omada_live_coding_30min.md](omada_live_coding_30min.md)** (26KB) | Deep analysis of 10 likely 30-minute coding challenges (overage billing, proration, etc.) | Read 3-5 days before for context |
| **[omada_live_coding_10min.md](omada_live_coding_10min.md)** (33KB) | 10 quick coding challenges for 10-minute sessions (validations, scopes, simple methods) | Review 1-2 days before, practice top 3 patterns |
| **[omada_code_templates.md](omada_code_templates.md)** (13KB) | Copy-paste code templates for rapid implementation | Reference during live coding |

### ⚡ Quick Reference (Day Of)

| File | Purpose | When to Use |
|------|---------|-------------|
| **[omada_quick_ref.md](omada_quick_ref.md)** (6.5KB) | Quick reference card with key patterns | Keep open during interview |

**Total Content:** ~150KB of targeted preparation material

---

## 📖 Recommended Learning Path

### 🎯 Phase 0: System Design Prep (Days -5 to -3)

**Goal:** Understand what to expect and practice integration patterns

#### Step 0: Understand the System Design Exercise
1. **Read [`omada_system_design_scope.md`](omada_system_design_scope.md)** (30-45 minutes)
   - Understand the scope and focus (integration patterns, not billing domain)
   - Learn the universal framework (clarify → design → deep dive → trade-offs)
   - Review integration patterns you'll need (webhooks, polling, queues)
   - Note reliability patterns (retry, circuit breakers, idempotency)

**What to do:**
- [ ] Internalize the system design framework
- [ ] Review integration patterns section
- [ ] Connect patterns to your take-home code (Sidekiq, async processing)

#### Step 0.5: Practice System Design Problems
2. **Practice with [`omada_practice_problems.md`](omada_practice_problems.md)** (2-3 hours, optional but recommended)
   - **Problem 1: Health Data Integration** (40 min practice + review) ⭐ MOST LIKELY
     - Most relevant to Omada's domain
     - Practice drawing the architecture
     - Study the clarifying questions
     - Review the deep dive answers
   
   - **Problem 2: Partner Integration** (30 min practice + review)
     - Shows batch processing patterns
     - Good for SFTP/file handling scenarios
   
   - **Problem 3: Notification Service** (30 min practice + review)
     - Multi-channel patterns
     - Priority queuing concepts

**What to do:**
- [ ] Pick Problem 1, set timer for 30-40 min
- [ ] Work through it without looking at solution
- [ ] Compare your approach with the solution
- [ ] Practice explaining your design out loud

**Output:** You can confidently work through a system design problem, ask good clarifying questions, and explain integration patterns

---

## 📖 Recommended Learning Path (Code Review + Live Coding)

### 🎯 Phase 1: Understanding (Days -5 to -3)

**Goal:** Build mental models of likely challenges and your codebase architecture

#### Step 1: Read [`omada_live_coding_30min.md`](omada_live_coding_30min.md) (60-90 minutes)
**Note:** This file focuses on 30-minute live coding challenges (larger feature extensions). For 10-minute quick additions, see [`omada_live_coding_10min.md`](omada_live_coding_10min.md).

**Focus areas:**
1. **"Your Implementation Analysis"** section (10 min)
   - Validates your architecture strengths
   - Identifies gaps that might be tested
   
2. **"Top 10 Live Coding Enhancements (30 minutes each)"** (45 min)
   - Read Tier 1 (most likely) thoroughly - overage billing, proration, grace period
   - Skim Tier 2 and 3
   - Understand WHY each is ranked where it is
   - These are larger features requiring migrations, services, and tests

3. **"Interview Strategy"** section (10 min)
   - Internalize the 5-25 minute breakdown (for 30-minute sessions)
   - Note the "Key Talking Points" phrases

**What to do:**
- [ ] Create mental map: "If they ask about overages, I modify RecurringBillingService + add line items"
- [ ] Highlight your top 3 predictions: Overage Billing, Proration, Grace Period
- [ ] Note production considerations you can mention

**Output:** You should be able to answer "What are the 3 most likely 30-minute extensions?" without looking (overage billing, proration, grace period)

---

#### Step 2: Study [`omada_code_templates.md`](omada_code_templates.md) (45 minutes)
**Also review:** [`omada_live_coding_10min.md`](omada_live_coding_10min.md) top 3 challenges (30 min) - for quick 10-minute additions
**Focus areas:**
1. **Migration Template** (5 min)
   - Memorize the basic structure
   - Note: `precision: 10, scale: 2` for money

2. **Service Object Template** (10 min)
   - Internalize your pattern: `initialize` → `call` → `private methods`
   - Return format: `{ success: bool, error: string }`

3. **Quick Implementations** section (30 min)
   - **Overage Billing** - Study this DEEPLY (most likely)
   - **Proration Logic** - Understand the date math
   - **Grace Period** - Note the validation pattern

**What to do:**
- [ ] Copy the overage billing implementation to a scratch file
- [ ] Walk through it line by line - could you explain each part?
- [ ] Practice the proration calculation on paper

**Output:** When you see "add overage billing", you know exactly what to type

---

### 🎯 Phase 2: Code Review Prep (Days -2 to -1)

**Goal:** Nail the 5-10 minute presentation of YOUR code

#### Step 3: Master [`omada_code_review_script.md`](omada_code_review_script.md) (2 hours total, spread over 2 days)

**Day -2 (1 hour):**
1. **Read entire script once** (20 min)
   - Understand the 4-part structure
   - Note time allocations per section

2. **Practice Part 2: Service Objects** (30 min)
   - This is the most important section
   - Open your actual code files
   - Practice explaining RecurringBillingService while pointing at code
   - Time yourself - should be ~3 minutes

3. **Review "Anticipated Questions"** (10 min)
   - Read each Q&A pair
   - Practice saying answers out loud
   - Note which questions you feel less confident about

**Day -1 (1 hour):**
1. **Full run-through with timer** (9 min)
   - Set timer, go through entire presentation
   - Don't memorize - just hit the key beats
   - Adjust if you're running long

2. **Practice drawing architecture diagram** (10 min)
   - Can you sketch the data model from memory?
   - Account → Subscription → Invoice → LineItems

3. **Review "Pro Tips" section** (5 min)
   - Focus on DO's and DON'T's
   - Internalize: "Start high-level, drill down"

4. **Final checklist review** (5 min)
   - Which files will you have open in tabs?
   - Are you ready for the "how would you add X" questions?

**What to do:**
- [ ] Record yourself doing the presentation (phone video is fine)
- [ ] Watch it back - are you confident? Too fast? Too detailed?
- [ ] Practice 2-3 times total (not more - you'll sound robotic)

**Output:** You can confidently walk through your code in 8-9 minutes

---

### 🎯 Phase 3: Day-Of Preparation (Interview Day)

**Goal:** Have quick references ready, be in the zone

#### Step 4: Morning Review (30 minutes before interview)

**15 minutes: [`omada_quick_ref.md`](omada_quick_ref.md)**
This is your cheat sheet. Read it fully, focusing on:

1. **"Your Architecture in 60 Seconds"** (5 min)
   - Refresh the key models and fields
   - `last_billing_date` → anniversary tracking
   - `widget_usage` → current cycle, resets on payment

2. **"Top 3 Most Likely Extensions"** (5 min)
   - Review the quick implementations
   - Overage: Check has_overage?, create line item
   - Proration: Credit + charge calculation
   - Grace period: Add suspended flag, validate

3. **"Interview Flow"** timing (2 min)
   - 0-5: Clarify
   - 5-25: Code
   - 25-30: Test & explain

4. **"Key Phrases to Use"** (3 min)
   - "I already have invoice_line_items set up for this..."
   - "Following the service pattern I established..."
   - "In production, I'd also consider..."

**What to do:**
- [ ] Print or have on second screen during interview
- [ ] Highlight the Top 3 implementations
- [ ] Have your data model mental image fresh

---

**15 minutes: Setup Check**
1. Open code editor with tabs:
   - `app/services/recurring_billing_service.rb`
   - `app/models/subscription.rb`
   - `app/models/invoice.rb`
   - `db/schema.rb`
   - `test/services/recurring_billing_service_test.rb`

2. Have ready in browser:
   - [`omada_quick_ref.md`](omada_quick_ref.md) (visible, don't hide it)
   - [`omada_code_templates.md`](omada_code_templates.md) (easy to search/copy)

3. Terminal ready:
   - `cd` to project directory
   - Test you can run: `rails test test/services/recurring_billing_service_test.rb`

---

## 🎓 How to Use During Interview

### Part 1: System Design (45-60 min)

**Reference:** [`omada_system_design_scope.md`](omada_system_design_scope.md) + [`omada_practice_problems.md`](omada_practice_problems.md)

**Framework (follow this):**
1. **Clarify Requirements (5-7 min)**
   - Ask about scale (users, requests, data volume)
   - Ask about external systems (APIs, rate limits, formats)
   - Ask about SLAs (latency, availability)
   - Ask "Is this real-time or batch?"

2. **High-Level Design (10-15 min)**
   - Draw the flow: External Systems → API Gateway → Queue → Workers → Database
   - Explain each component's purpose
   - Show data flow with arrows

3. **Deep Dive (20-25 min)**
   - They'll ask: "How do you handle failures?" "What if API is down?" "How prevent duplicates?"
   - Be ready for: Error handling, retry logic, idempotency, circuit breakers

4. **Trade-offs (10 min)**
   - Proactively discuss alternatives
   - "For v1 I'd do X, at scale we'd need Y"

**Quick mental review right before:**
- Integration patterns (webhooks, polling, batch)
- Reliability patterns (retry, circuit breaker, idempotency)
- Your framework (don't forget to ask questions first!)

---

### Part 2: Code Review (5-10 min)

**Reference:** [`omada_code_review_script.md`](omada_code_review_script.md)

**Quick review right before:**
- Opening statement (30 sec)
- Part 2: Services emphasis on RecurringBillingService
- Closing: Three takeaways

**During presentation:**
- Don't read the script
- Use it as guardrails for structure
- If you blank, glance at section headers to remember next beat

---

### Part 3: Live Coding (10-30 min)

**Reference:** [`omada_quick_ref.md`](omada_quick_ref.md) + [`omada_code_templates.md`](omada_code_templates.md) + [`omada_live_coding_10min.md`](omada_live_coding_10min.md)

**If 10-15 minutes (most likely for small additions):**
- Review [`omada_live_coding_10min.md`](omada_live_coding_10min.md) top 3 challenges before interview
- Focus on: Validations, Scopes, Simple calculation methods
- Strategy: Clarify (1-2 min) → Locate file (30 sec) → Implement (3-4 min) → Test (3-4 min) → Verify (1 min)
- Keep it simple - working code with one test is enough

**If 30 minutes (larger feature extension):**
- Reference [`omada_code_templates.md`](omada_code_templates.md) for complex implementations
- Follow the full pattern: Migration → Model → Service → Test
- See details below

**Minute 0-2: Clarify & Plan**
1. Listen to the requirement carefully
2. Ask: "Should I write a test?" "Any edge cases to consider?"
3. Identify which model/file needs changes
4. If it's a quick challenge, use patterns from [`omada_live_coding_10min.md`](omada_live_coding_10min.md)

**Minute 2-20: Implementation**
1. **For simple additions** (validations, scopes, methods):
   - Open the model file
   - Follow existing patterns
   - Keep it simple and correct

2. **For larger features** (migrations, services):
   - Migration first if adding columns
   - Search "Migration Template" in [`omada_code_templates.md`](omada_code_templates.md)
   - Model/Service next using templates
   - Adapt to specific requirement

3. **Test as you go**
   - Write test alongside code
   - Happy path + 1-2 edge cases
   - Use existing test patterns

**Minute 20-30: Demonstrate & Discuss**
1. Run test, show it passing
2. Walk through key logic
3. Mention edge cases you'd handle with more time

---

## 🧠 Mental Models to Internalize

### Your Architecture Pattern
```
Question Asked
    ↓
Which Service/Model affected?
    ↓
Need new column? → Migration
    ↓
Update Model → Validations
    ↓
Update/Create Service → Transaction + business logic
    ↓
Write Test → Happy path + edge case
```

### Billing Flow Pattern
```
Subscription Created (pending)
    ↓
ActivateSubscriptionService → First invoice created
    ↓
RecurringBillingService → Anniversary invoices (last_billing_date + 1.month)
    ↓
ReconcilePaymentService → Mark paid, reset widget_usage
```

### Extension Pattern
Most extensions follow:
1. Add field to Plan or Subscription
2. Modify calculation in RecurringBillingService
3. Add line item creation
4. Update total amount
5. Test calculation

---

## 📊 Study Time Estimates

**Minimum (6-7 hours):**
- System Design: 2 hours ([`omada_system_design_scope.md`](omada_system_design_scope.md) + practice Problem 1)
- 30-min Live Coding: [`omada_live_coding_30min.md`](omada_live_coding_30min.md) - 60 min (focus on Tier 1)
- 10-min Live Coding: [`omada_live_coding_10min.md`](omada_live_coding_10min.md) - 30 min (review top 3 challenges)
- [`omada_code_templates.md`](omada_code_templates.md): 30 min (focus on top 3)
- [`omada_code_review_script.md`](omada_code_review_script.md): 90 min (practice presentation)
- [`omada_quick_ref.md`](omada_quick_ref.md): 30 min (day-of review)

**Recommended (9-10 hours):**
- Day -5 to -3: 5 hours (system design scope + practice Problem 1: 2.5 hours, live_coding_30min + code_templates + 10min challenges: 2.5 hours)
- Day -2 to -1: 2 hours (code review practice, twice through)
- Day of: 30 min (quick_ref refresh + system design framework reminder)
- Live practice: 2 hours (implement top 3 coding scenarios yourself for real)

**Thorough (12+ hours):**
- All of the above
- Plus: Practice all 3 system design problems in [`omada_practice_problems.md`](omada_practice_problems.md)
- Plus: Actually implement each Tier 1 feature in your codebase
- Plus: Record yourself doing code review 5 times
- Plus: Pair program practice session with someone

---

## ✅ Pre-Interview Checklist

### 5 Days Before
- [ ] Read [`omada_system_design_scope.md`](omada_system_design_scope.md) fully
- [ ] Practice Problem 1 from [`omada_practice_problems.md`](omada_practice_problems.md) (Health Data Integration)
- [ ] Can you explain integration patterns (webhooks, polling, queues)?

### 3 Days Before
- [ ] Read [`omada_live_coding_30min.md`](omada_live_coding_30min.md) fully (30-minute challenges)
- [ ] Study [`omada_code_templates.md`](omada_code_templates.md) top 3 implementations
- [ ] Review [`omada_live_coding_10min.md`](omada_live_coding_10min.md) top 3 challenges (10-minute quick additions: validations, scopes, methods)
- [ ] Can you explain why overage billing is #1 most likely for 30-minute challenges?

### 1 Day Before
- [ ] Practice code review presentation 2-3 times
- [ ] Watch yourself once (video)
- [ ] Can you draw your data model from memory?
- [ ] Review anticipated questions in [`omada_code_review_script.md`](omada_code_review_script.md)
- [ ] Quick review: System design framework (clarify → design → deep dive → trade-offs)

### Morning Of
- [ ] Read [`omada_quick_ref.md`](omada_quick_ref.md) completely
- [ ] Have code editor open with key files in tabs
- [ ] Print or have quick_ref visible during interview
- [ ] Can run `rails test` in your project

### Right Before Interview (5 min)
- [ ] Deep breath
- [ ] Review your opening statement
- [ ] Glance at Top 3 predictions
- [ ] Remember: You built this. You know it.

---

## 🎯 Success Criteria

**You're ready when you can:**

1. **System Design Part:**
   - [ ] Can ask 5-7 good clarifying questions
   - [ ] Can draw a clear architecture diagram in 10 minutes
   - [ ] Can explain integration patterns (webhooks, polling, queues)
   - [ ] Can discuss failure handling (retry, circuit breakers)
   - [ ] Can explain trade-offs clearly

2. **Code Review Part:**
   - [ ] Explain your architecture in 2 minutes without looking
   - [ ] Walk through RecurringBillingService confidently
   - [ ] Answer "why service objects?" without hesitation
   - [ ] Respond to "how would you add overage billing?" immediately

3. **Live Coding Part:**
   - [ ] Identify which files need changes in < 1 minute
   - [ ] For 10-min: Can add a validation, scope, or simple method in 5-7 minutes
   - [ ] For 30-min: Know the service object template by heart (migrations, services, tests)
   - [ ] Can write a basic test without reference
   - [ ] Recognize the top 3 quick challenges (validation, scope, calculation method)
   - [ ] Recognize the top 3 larger challenges (overage billing, proration, grace period)

4. **General Confidence:**
   - [ ] You can explain any file in your codebase
   - [ ] You know the trade-offs you made
   - [ ] You're comfortable saying "in production I'd also..."
   - [ ] You see the extensions as natural next steps, not tricks

---

## 🔥 Power Tips

### Time Management
- **Don't** read all files linearly (too much content)
- **Do** use the learning path phases
- **Don't** try to memorize code
- **Do** internalize patterns and structure

### Day Of
- **Don't** cram new material morning of
- **Do** refresh with [`omada_quick_ref.md`](omada_quick_ref.md) only
- **Don't** hide your reference materials during live coding
- **Do** search [`omada_code_templates.md`](omada_code_templates.md) quickly when needed

### During Interview
- **Don't** panic if question isn't in top 10
- **Do** follow the same pattern: migration → model → service → test
- **Don't** code silently
- **Do** narrate your thought process

### Mindset
- **Don't** aim for perfect code
- **Do** aim for correct business logic and good patterns
- **Don't** apologize for your implementation
- **Do** confidently explain your choices

---

## 🆘 If Things Go Wrong

### Code Review: Running Long
- Skip detailed testing section
- Focus on RecurringBillingService only
- Jump to closing with 3 takeaways

### Code Review: They Look Confused
- Stop and ask: "Would you like me to drill into any specific area?"
- Show code instead of explaining
- Draw the data model

### Live Coding: Don't Recognize the Challenge
- Ask clarifying questions to understand requirement
- Identify which service/model is affected
- Check if it's a quick validation/scope/method (see [`omada_live_coding_10min.md`](omada_live_coding_10min.md))
- If larger feature, use the generic service template from [`omada_code_templates.md`](omada_code_templates.md)

### Live Coding: Code Isn't Working
- Narrate what you're debugging
- Explain what the code SHOULD do
- Mention how you'd fix it with more time

### Live Coding: Running Out of Time (10-minute session)
- Focus on working code over comprehensive tests
- One passing test is enough
- Explain what you'd add with more time
- Show you understand the pattern

### Live Coding: Running Out of Time (30-minute session)
- Focus on business logic over tests
- Explain what's missing: "I'd add validation for X"
- Show you know the pattern even if incomplete

### System Design: Drawing Mistakes
- It's okay to erase and redraw
- Start simple, add complexity
- Label everything clearly
- Ask "should I add more detail here?" if unsure

### System Design: Don't Know an Answer
- Say "I'm not familiar with that specific technology, but here's how I'd think about it..."
- Reference similar patterns from your experience
- Ask "what would you recommend?" (shows collaboration)

---

## 📝 Final Notes

### What Makes Your Implementation Strong
1. Service-oriented design (clean separation)
2. Anniversary billing with `last_billing_date`
3. Extensible line items table
4. Widget usage history tracking
5. Proper money handling and validations

### What They're Really Testing
1. Can you navigate and extend your own code?
2. Do you follow Rails conventions?
3. Can you handle date math and money calculations?
4. Do you think about edge cases?
5. Can you work under time pressure without AI?

### Remember
- You built a solid system
- The extensions are natural next steps
- They want to see how you think, not perfection
- You know this codebase better than anyone in that room

---

## 🚀 You've Got This!

**Your preparation:**
- ✅ Understood system design scope and patterns
- ✅ Practiced integration service designs
- ✅ Analyzed your implementation deeply
- ✅ Identified top 10 likely extensions (30-min challenges)
- ✅ Reviewed top 3 quick challenges (10-min validations/scopes/methods)
- ✅ Practiced your code review presentation
- ✅ Have templates ready for rapid coding

**Your strengths:**
- ✅ Clean service-oriented architecture
- ✅ Thoughtful design decisions
- ✅ Production-ready patterns
- ✅ You actually built this

**Bottom line:** Follow the learning path, practice the code review twice, refresh with quick_ref day-of, and trust your preparation. You're ready.

Now go nail this interview! 💪

---

## 📞 Quick Reference Summary

**Before Interview:**
1. Read [`omada_system_design_scope.md`](omada_system_design_scope.md) for context
2. Practice Problem 1 from [`omada_practice_problems.md`](omada_practice_problems.md)
3. Study [`omada_live_coding_30min.md`](omada_live_coding_30min.md) for 30-minute coding challenges
4. Review [`omada_live_coding_10min.md`](omada_live_coding_10min.md) for 10-minute quick additions
5. Master [`omada_code_templates.md`](omada_code_templates.md) top 3
6. Practice [`omada_code_review_script.md`](omada_code_review_script.md) 2-3 times

**Day Of:**
1. Review [`omada_quick_ref.md`](omada_quick_ref.md) only (30 min)
2. Quick mental review: System design framework
3. Have code editor open with key files
4. Keep quick_ref visible during interview

**During System Design:**
1. Ask clarifying questions first (5-7 min)
2. Draw high-level architecture (10-15 min)
3. Deep dive on 2-3 areas (20-25 min)
4. Discuss trade-offs (10 min)

**During Code Review:**
1. Architecture → Services → Design decisions
2. Focus on RecurringBillingService
3. Close with 3 takeaways

**During Live Coding:**
- **If 10-15 min:** Follow [`omada_live_coding_10min.md`](omada_live_coding_10min.md) strategy (validation/scope/method pattern)
- **If 30 min:** Full feature extension
  1. Clarify (0-5 min)
  2. Code (5-25 min): Migration → Model → Service → Test
  3. Demonstrate (25-30 min)

**If stuck:**
- System Design: Review integration patterns from [`omada_system_design_scope.md`](omada_system_design_scope.md)
- Live Coding (quick): Check [`omada_live_coding_10min.md`](omada_live_coding_10min.md) for similar patterns
- Live Coding (larger): Search [`omada_code_templates.md`](omada_code_templates.md)
- Follow your service object pattern
- Narrate your thinking

**You know this. Show them.**
