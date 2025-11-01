# Omada Health Interview Preparation Guide

## 📚 Overview

This repository contains four comprehensive guides to prepare you for the Omada Health Software Engineer II interview. The interview has two parts:

1. **Code Review (5-10 minutes)** - Present your take-home implementation
2. **Live Coding (30 minutes)** - Extend your code without AI assistance

These guides will help you master both parts efficiently.

---

## 🗂️ Files in This Repository

| File | Purpose | When to Use |
|------|---------|-------------|
| **[omada_code_review_script.md](omada_code_review_script.md)** (12KB) | Step-by-step presentation script for code review | Study 2-3 days before, review morning of interview |
| **[omada_interview_prep.md](omada_interview_prep.md)** (26KB) | Deep analysis of 10 likely coding challenges | Read 3-5 days before for context |
| **[omada_quick_ref.md](omada_quick_ref.md)** (6.5KB) | Quick reference card with key patterns | Keep open during interview |
| **[omada_code_templates.md](omada_code_templates.md)** (13KB) | Copy-paste code templates for rapid implementation | Reference during live coding |

**Total Content:** ~57KB of targeted preparation material

---

## 📖 Recommended Learning Path

### 🎯 Phase 1: Understanding (Days -5 to -3)

**Goal:** Build mental models of likely challenges and your codebase architecture

#### Step 1: Read [`omada_interview_prep.md`](omada_interview_prep.md) (60-90 minutes)
**Focus areas:**
1. **"Your Implementation Analysis"** section (10 min)
   - Validates your architecture strengths
   - Identifies gaps that might be tested
   
2. **"Top 10 Live Coding Enhancements"** (45 min)
   - Read Tier 1 (most likely) thoroughly
   - Skim Tier 2 and 3
   - Understand WHY each is ranked where it is

3. **"Interview Strategy"** section (10 min)
   - Internalize the 5-25 minute breakdown
   - Note the "Key Talking Points" phrases

**What to do:**
- [ ] Create mental map: "If they ask about overages, I modify RecurringBillingService + add line items"
- [ ] Highlight your top 3 predictions: Overage Billing, Proration, Grace Period
- [ ] Note production considerations you can mention

**Output:** You should be able to answer "What are the 3 most likely extensions?" without looking

---

#### Step 2: Study [`omada_code_templates.md`](omada_code_templates.md) (45 minutes)
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

### Part 1: Code Review (5-10 min)

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

### Part 2: Live Coding (30 min)

**Reference:** [`omada_quick_ref.md`](omada_quick_ref.md) (visible) + [`omada_code_templates.md`](omada_code_templates.md) (searchable)

**Minute 0-5: Clarify & Plan**
1. Listen to the requirement
2. Glance at "Top 3" in quick_ref - is it there?
3. If yes: You know the pattern, outline approach
4. If no: Identify which service/model needs changes

**Minute 5-25: Implementation**
1. **Migration first** if adding columns
   - Search "Migration Template" in [`omada_code_templates.md`](omada_code_templates.md)
   - Copy structure, adapt

2. **Model/Service next**
   - Search for relevant feature in [`omada_code_templates.md`](omada_code_templates.md)
   - Copy relevant portions
   - Adapt to specific requirement

3. **Test as you go**
   - Copy test template
   - Write one happy path test
   - Run it periodically

**Minute 25-30: Demonstrate & Discuss**
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

**Minimum (4 hours):**
- [`omada_interview_prep.md`](omada_interview_prep.md): 60 min (focus on Tier 1)
- [`omada_code_templates.md`](omada_code_templates.md): 30 min (focus on top 3)
- [`omada_code_review_script.md`](omada_code_review_script.md): 90 min (practice presentation)
- [`omada_quick_ref.md`](omada_quick_ref.md): 30 min (day-of review)

**Recommended (6-7 hours):**
- Day -5 to -3: 2.5 hours (interview_prep + code_templates deep study)
- Day -2 to -1: 2 hours (code review practice, twice through)
- Day of: 30 min (quick_ref refresh)
- Live practice: 2 hours (implement top 3 scenarios yourself for real)

**Thorough (10+ hours):**
- All of the above
- Plus: Actually implement each Tier 1 feature in your codebase
- Plus: Record yourself doing code review 5 times
- Plus: Pair program practice session with someone

---

## ✅ Pre-Interview Checklist

### 3 Days Before
- [ ] Read [`omada_interview_prep.md`](omada_interview_prep.md) fully
- [ ] Study [`omada_code_templates.md`](omada_code_templates.md) top 3 implementations
- [ ] Can you explain why overage billing is #1 most likely?

### 1 Day Before
- [ ] Practice code review presentation 2-3 times
- [ ] Watch yourself once (video)
- [ ] Can you draw your data model from memory?
- [ ] Review anticipated questions in [`omada_code_review_script.md`](omada_code_review_script.md)

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

1. **Code Review Part:**
   - [ ] Explain your architecture in 2 minutes without looking
   - [ ] Walk through RecurringBillingService confidently
   - [ ] Answer "why service objects?" without hesitation
   - [ ] Respond to "how would you add overage billing?" immediately

2. **Live Coding Part:**
   - [ ] Identify which files need changes in < 1 minute
   - [ ] Know the service object template by heart
   - [ ] Can write a basic test without reference
   - [ ] Recognize the top 3 challenges instantly if asked

3. **General Confidence:**
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
- Use the generic service template from [`omada_code_templates.md`](omada_code_templates.md)

### Live Coding: Code Isn't Working
- Narrate what you're debugging
- Explain what the code SHOULD do
- Mention how you'd fix it with more time

### Live Coding: Running Out of Time
- Focus on business logic over tests
- Explain what's missing: "I'd add validation for X"
- Show you know the pattern even if incomplete

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
- ✅ Analyzed your implementation deeply
- ✅ Identified top 10 likely extensions
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
1. Study [`omada_interview_prep.md`](omada_interview_prep.md) for context
2. Master [`omada_code_templates.md`](omada_code_templates.md) top 3
3. Practice [`omada_code_review_script.md`](omada_code_review_script.md) 2-3 times

**Day Of:**
1. Review [`omada_quick_ref.md`](omada_quick_ref.md) only (30 min)
2. Have code editor open with key files
3. Keep quick_ref visible during interview

**During Code Review:**
1. Architecture → Services → Design decisions
2. Focus on RecurringBillingService
3. Close with 3 takeaways

**During Live Coding:**
1. Clarify (0-5 min)
2. Code (5-25 min): Migration → Model → Service → Test
3. Demonstrate (25-30 min)

**If stuck:**
- Search [`omada_code_templates.md`](omada_code_templates.md)
- Follow your service object pattern
- Narrate your thinking

**You know this. Show them.**
