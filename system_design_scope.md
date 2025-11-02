# System Design Exercise - Scope Analysis

## 🔍 What They Said

> "an in-person system design whiteboard exercise where you'll design a service that integrates with various internal and external applications"

## 🎯 Key Phrase Analysis

### "design a service"
- Singular service, not entire system
- Focused scope, not broad architecture

### "integrates with various internal and external applications"
- **Internal:** Other Omada services/systems
- **External:** Third-party APIs, webhooks, SaaS tools
- Focus on **integration patterns**, not the domain itself

### "There's nothing additional to prepare beyond your submitted code"
- The design exercise is separate from billing domain
- Your take-home shows you can code
- The design tests integration/architecture thinking

## 🤔 Is It Related to Billing or Different?

### Most Likely: **DIFFERENT DOMAIN**

**Evidence:**
1. They said "design **a service**" (new, not extend yours)
2. Emphasis on "integrates with various applications" (integration focus)
3. "Nothing additional to prepare" (not billing-specific prep needed)
4. Omada is a health tech company - design likely health-tech related

### Possible: **Related to Billing Conceptually**

**Could be:**
- "Design a payment gateway integration service"
- "Design a service that syncs subscription data with CRM"
- Uses similar patterns (webhooks, APIs) but different domain

## 🏥 Likely System Design Topics (Omada Context)

Omada Health is a **digital health platform** for chronic disease management. Possible designs:

### Option 1: Healthcare Data Integration Service
"Design a service that pulls patient health data from various EHR systems (Epic, Cerner) and wearable devices (Fitbit, Apple Health)."

**Focus:**
- Multiple external integrations
- Data normalization
- HIPAA compliance
- Real-time vs batch processing
- Error handling for flaky APIs

### Option 2: Coaching/Care Team Integration
"Design a service that coordinates between care coaches, patients, and clinical systems."

**Focus:**
- Real-time messaging
- Task assignment and routing
- Notification systems
- Availability scheduling
- Multi-channel (SMS, email, in-app)

### Option 3: Partner Integration Platform
"Design a service that syncs Omada member data with employer HR systems and health plan databases."

**Focus:**
- Bidirectional sync
- Data consistency
- Batch file processing (SFTP, S3)
- Webhook handling
- Retry/reconciliation

### Option 4: Device/App Data Aggregation
"Design a service that aggregates health metrics from multiple sources (glucose meters, scales, fitness trackers)."

**Focus:**
- Multiple API integrations
- Rate limiting
- Data validation
- Time-series storage
- Anomaly detection

## 🎯 Common Patterns (Regardless of Domain)

The design will test your knowledge of:

### 1. Integration Patterns
- **Webhooks** (receiving real-time updates)
- **Polling** (checking for updates periodically)
- **Batch processing** (SFTP, S3 file drops)
- **REST APIs** (calling external services)
- **Message queues** (async processing)

### 2. Data Flow
- **Ingestion** → Validation → Transformation → Storage
- **Event-driven** vs request/response
- **Idempotency** (handling retries/duplicates)

### 3. Reliability
- **Retry logic** with exponential backoff
- **Circuit breakers** for flaky services
- **Dead letter queues** for failed processing
- **Monitoring and alerting**

### 4. Scalability
- **Rate limiting** (respecting external API limits)
- **Batching** (grouping requests)
- **Async processing** (don't block the API)
- **Caching** (reducing external calls)

### 5. Security/Compliance
- **Authentication** (OAuth, API keys)
- **Data encryption** (in transit and at rest)
- **PII handling** (especially in healthcare)
- **Audit logging**

## 📚 How to Prepare

### Don't Over-Prepare
> "There's nothing additional to prepare"

They mean it. They want to see:
- How you think through problems
- How you ask clarifying questions
- How you communicate design decisions
- How you handle feedback and iterate

### Do Review These Concepts

**Integration Patterns (1 hour):**
- Webhook architecture
- API polling strategies
- Batch file processing
- Event-driven design

**Reliability Patterns (30 min):**
- Idempotency keys
- Retry with exponential backoff
- Circuit breakers
- Dead letter queues

**Your Take-Home (30 min):**
- You used similar patterns in RecurringBillingService
- Background jobs (Sidekiq)
- Async processing
- Retry logic
- These patterns apply to ANY integration service

## 🎨 System Design Framework (Universal)

Use this framework regardless of domain:

### Step 1: Clarify Requirements (5-7 min)

**Ask these questions:**
1. "What's the scale?" (users, requests/day, data volume)
2. "What are the external systems?" (APIs, data formats, rate limits)
3. "What are the SLAs?" (latency, availability)
4. "Is this real-time or batch?"
5. "What's the priority: consistency or availability?"
6. "What already exists in the system?"

### Step 2: High-Level Design (10-15 min)

**Draw the flow:**
```
External Systems → API Gateway → Queue → Workers → Database
                      ↓
                  Webhooks
                      ↓
                  Our Service
```

**Explain each component:**
- Why API Gateway? (authentication, rate limiting)
- Why Queue? (async processing, retry, scale)
- Why Workers? (parallel processing, isolation)
- Storage strategy (what data, where, why)

### Step 3: Deep Dive (20-25 min)

**They'll ask about:**
- "How do you handle failures?"
- "What if an external API is down?"
- "How do you prevent duplicates?"
- "How do you scale this?"
- "What monitoring do you add?"

**Be ready to discuss:**
- Error handling strategy
- Retry logic with backoff
- Idempotency (using request IDs)
- Circuit breaker pattern
- Metrics and alerts

### Step 4: Edge Cases & Trade-offs (10 min)

**Proactively mention:**
- "If [external service] is slow, we could..."
- "Trade-off: consistency vs. availability"
- "For v1, I'd start simple with [X], but at scale we'd need [Y]"
- "We need to handle: rate limits, timeouts, partial failures"

## 🎤 Example Walkthrough

### If They Ask: "Design a service that integrates with Fitbit API to pull step data"

**Clarify (you ask):**
- "How many users? How often do we pull data?"
- "Real-time or batch? Once daily or continuously?"
- "What's Fitbit's rate limit?"
- "Do we need historical data or just recent?"
- "What do we do if Fitbit is down?"

**Design (you draw):**
```
┌─────────────┐
│  Fitbit API │
└──────┬──────┘
       │
       ↓
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Webhook   │────→│    Queue    │────→│   Workers   │
│  Receiver   │     │  (Sidekiq)  │     │(Process Data)│
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
       ↑                                       ↓
       │                                 ┌─────────────┐
┌─────────────┐                         │  Database   │
│   Poller    │                         │(Steps Data) │
│(Backup/Init)│                         └─────────────┘
└─────────────┘
```

**Explain:**
1. **Webhook receiver**: Fitbit sends updates when new data available
2. **Queue**: Decouple receiving from processing
3. **Workers**: Process data asynchronously
4. **Poller**: Backup mechanism if webhook fails, also for initial data load
5. **Database**: Store processed step data

**Deep dive (they ask):**

Q: "What if webhook fails?"
A: "We have the poller as backup. Poll once daily for users who haven't had a webhook in 24 hours."

Q: "How do you handle rate limits?"
A: "Track API quota usage. Use Redis to count requests per hour. Implement exponential backoff on 429 responses. Queue requests during high traffic."

Q: "What if Fitbit is down?"
A: "Circuit breaker pattern. After N consecutive failures, stop trying for X minutes. Alert on-call. Queue requests for retry when service recovers."

Q: "How do you prevent duplicate data?"
A: "Use idempotency keys. Fitbit webhook includes event ID - store processed event IDs. Skip if already processed. Database unique constraint on (user_id, date, data_source)."

**Trade-offs:**
- "Started with webhook + poller hybrid for reliability"
- "Could do pure polling, but webhook gives real-time updates"
- "Could do pure webhook, but polling provides safety net"
- "Trade-off: more complex (two mechanisms) but more reliable"

## 🎯 Key Success Factors

### ✅ DO:

1. **Ask clarifying questions first**
   - Show you gather requirements before designing
   - Demonstrates thoughtfulness

2. **Start simple, add complexity**
   - "For v1, I'd do X..."
   - "At scale, we'd need Y..."

3. **Draw clear diagrams**
   - Boxes and arrows
   - Label components
   - Show data flow

4. **Explain your reasoning**
   - "I chose X because..."
   - "This optimizes for..."
   - "The trade-off is..."

5. **Discuss failure modes**
   - "If this fails, then..."
   - "We handle this by..."
   - Show you think about reliability

6. **Mention monitoring**
   - "We'd track these metrics..."
   - "Alert on this condition..."
   - Show production awareness

7. **Be collaborative**
   - Listen to their feedback
   - Iterate on your design
   - "Good point, we could also..."

### ❌ DON'T:

1. **Don't jump straight to code**
   - This is architecture, not implementation
   - They want to see system thinking

2. **Don't over-engineer v1**
   - Start simple, mention future complexity
   - "Could add Kafka later, but start with Sidekiq"

3. **Don't ignore constraints**
   - Rate limits, latency, scale
   - Ask about these if not mentioned

4. **Don't forget about errors**
   - Every external call can fail
   - Show you plan for failures

5. **Don't be rigid**
   - If they suggest alternative, consider it
   - "That's a good point, if we..."

## 🔗 Connection to Your Take-Home

**You already demonstrated these skills:**

From your RecurringBillingService:
- ✅ Background job processing (Sidekiq)
- ✅ Retry logic considerations
- ✅ Transaction handling
- ✅ Error handling patterns

From your design_decisions.md:
- ✅ Production thinking ("In production, this would use Sidekiq")
- ✅ Scalability awareness
- ✅ Monitoring considerations

**During the interview, reference your take-home:**
- "Similar to how I used Sidekiq for billing, here I'd..."
- "Like the idempotency in payment processing, we'd need..."
- "The pattern I used for invoice generation applies here..."

## 📋 Pre-Interview Prep Checklist

### Day Before (1-2 hours)

**Review Integration Patterns (30 min):**
- [ ] Webhooks (how they work, retry, verification)
- [ ] API polling (frequency, rate limits, backoff)
- [ ] Batch processing (file formats, schedules)
- [ ] Message queues (decoupling, scaling)

**Practice Drawing (30 min):**
- [ ] Draw 3 simple architectures on paper
- [ ] Practice explaining components
- [ ] Practice adding complexity incrementally

**Review Reliability Patterns (30 min):**
- [ ] Idempotency (how to implement)
- [ ] Retry logic (exponential backoff)
- [ ] Circuit breakers (when to use)
- [ ] Dead letter queues (failure handling)

**Review Your Take-Home (30 min):**
- [ ] How you used Sidekiq
- [ ] How you handled errors
- [ ] How you ensured data integrity
- [ ] Production considerations you mentioned

### Day Of (15 min)

**Quick Review:**
- [ ] System design framework (clarify → design → deep dive → trade-offs)
- [ ] Key phrases ("The trade-off is...", "At scale we'd need...")
- [ ] Remember to ask questions first
- [ ] Remember to start simple

**Mental Prep:**
- [ ] This is a conversation, not a test
- [ ] It's okay to say "I don't know, but here's how I'd think about it"
- [ ] Show your thought process
- [ ] Be collaborative

## 🎯 Bottom Line

### The System Design Will Likely Be:
- ✅ Different domain (not billing)
- ✅ Focus on integration patterns
- ✅ Universal architecture concepts
- ✅ Not specific to your take-home code

### But You Can Still Connect It:
- "Similar to my billing service, I'd use..."
- "The patterns I applied there work here..."
- Show consistency in your thinking

### What Matters Most:
1. **Asking good questions** (gathering requirements)
2. **Clear communication** (explaining your design)
3. **Practical thinking** (failure modes, monitoring)
4. **Collaboration** (listening, iterating)
5. **Production awareness** (scale, reliability, ops)

### Final Prep Priority:
1. **Integration patterns** (1 hour) - webhooks, APIs, queues
2. **Reliability patterns** (30 min) - retry, idempotency, circuit breaker
3. **Practice drawing** (30 min) - clear diagrams on paper
4. **Your take-home review** (30 min) - reference-able patterns

**Total prep: 2-3 hours beyond your take-home review**

The good news: You already think this way (your design_decisions.md proves it). Now practice explaining it clearly and drawing it visually.

You've got this! 🚀
