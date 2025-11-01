# Omada System Design - Practice Problems

## 🎯 How to Use This Document

For each problem:
1. **Read the prompt** - Don't look at solution yet
2. **Set timer for 30-40 min** - Practice like real interview
3. **Work through it:** Clarify → Design → Deep Dive → Trade-offs
4. **Draw on paper/whiteboard** - Practice visual communication
5. **Compare with solution** - See what you missed
6. **Practice explaining out loud** - Simulate interview

---

## Problem 1: Health Data Integration Service ⭐ MOST LIKELY

### 📋 Prompt

**Interviewer says:**
"Omada needs to integrate with multiple health data sources to track our members' health metrics. Design a service that pulls data from wearable devices like Fitbit, Apple Health, and Garmin. The service should collect step counts, weight, blood glucose, and activity data. We have about 50,000 active members, and we need to keep their data reasonably up-to-date throughout the day."

**Follow-up info they'll give if you ask:**
- Members can connect multiple devices
- Fitbit has webhooks, Apple Health requires polling (via HealthKit on member's device)
- Garmin has both webhooks and API
- Rate limits: Fitbit 150 req/hour per user, Garmin 5000 req/day
- Data should be available within 1 hour of activity
- Need to handle members disconnecting/reconnecting devices
- Historical data import when first connecting

---

### 🤔 Step 1: Questions You Should Ask

**Before designing, clarify:**

1. **Scale questions:**
   - "How many device connections total?" → ~100K (members have 2 devices on average)
   - "How often do we check for new data?" → Continuous for webhooks, every 15-30 min for polling
   - "How much historical data?" → Up to 3 months on initial connect

2. **Priority questions:**
   - "What's more important: real-time data or data completeness?" → Completeness (okay if 30-60 min delayed)
   - "Do we need to deduplicate data from multiple sources?" → Yes, same step count shouldn't double-count
   - "What happens if a device API is down?" → Fall back to polling, don't lose data

3. **Technical questions:**
   - "What data format do these APIs return?" → Mostly JSON, some XML
   - "Do we need to normalize data across sources?" → Yes, unified format internally
   - "Where does the data go after ingestion?" → Database for querying, possibly data warehouse

4. **Compliance questions:**
   - "Any HIPAA concerns?" → Yes, health data needs encryption, audit logs

---

### 🎨 Step 2: High-Level Design

**Draw this:**

```
┌─────────────────────────────────────────────────────────────┐
│                    External Device APIs                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐             │
│  │  Fitbit  │    │  Garmin  │    │  Apple   │             │
│  │ (webhook)│    │(webhook) │    │(polling) │             │
│  └─────┬────┘    └─────┬────┘    └────┬─────┘             │
└────────┼───────────────┼──────────────┼────────────────────┘
         │               │              │
         ↓               ↓              ↓
    ┌────────────────────────────────────────┐
    │      API Gateway / Load Balancer       │
    │   (Auth, Rate Limiting, Validation)    │
    └───────────────┬────────────────────────┘
                    │
         ┌──────────┴──────────┐
         ↓                     ↓
┌─────────────────┐   ┌─────────────────┐
│Webhook Receiver │   │  Polling Jobs   │
│   (Web App)     │   │   (Sidekiq)     │
└────────┬────────┘   └────────┬────────┘
         │                     │
         └──────────┬──────────┘
                    ↓
            ┌───────────────┐
            │     Queue     │
            │   (Sidekiq)   │
            └───────┬───────┘
                    ↓
         ┌──────────────────────┐
         │  Processing Workers  │
         │  (Parallel, Async)   │
         └──────────┬───────────┘
                    │
         ┌──────────┴──────────┐
         ↓                     ↓
    ┌─────────┐         ┌──────────┐
    │   DB    │         │  Cache   │
    │(Postgres│         │ (Redis)  │
    │+ TimeSca│         │          │
    │  leDB)  │         └──────────┘
    └─────────┘
         │
         ↓
    ┌─────────┐
    │  Data   │
    │Warehouse│
    │(Analytics)
    └─────────┘
```

---

### 💬 Step 3: Explain Each Component

**As you draw, say:**

**1. API Gateway / Load Balancer**
"The API Gateway handles all incoming requests - both webhooks from Fitbit/Garmin and health checks. It does authentication (verifying webhook signatures), rate limiting (so we don't exceed our own quotas), and basic validation before passing to backend services."

**2. Webhook Receiver (Web App)**
"For Fitbit and Garmin webhooks, we have a lightweight web app that receives the notification, validates it, and immediately queues the work. The endpoint returns 200 OK quickly - we don't process data synchronously. This ensures we don't miss webhooks if processing is slow."

**3. Polling Jobs (Sidekiq)**
"For Apple Health and as backup for webhooks, we have scheduled Sidekiq jobs that poll for new data. These run every 15-30 minutes per user. We batch users to respect rate limits. If a webhook hasn't fired in X hours, polling picks it up."

**4. Queue (Sidekiq/Redis)**
"The queue decouples receiving data from processing it. This lets us handle spikes (e.g., morning when everyone syncs their overnight data). We can scale workers independently. The queue provides retry logic automatically."

**5. Processing Workers**
"Workers pull jobs from queue and do the heavy lifting:
- Call device API to fetch detailed data (webhook just says 'new data available')
- Validate and normalize data (different APIs have different formats)
- Deduplicate (check if we already have this data point)
- Transform to our internal schema
- Store in database"

**6. Database (Postgres + TimescaleDB)**
"Primary storage. Using Postgres for relational data (user-device connections) and TimescaleDB extension for time-series health metrics (efficient for date-range queries). Partitioned by user_id and date for performance."

**7. Cache (Redis)**
"Stores:
- API quota usage per device (respect rate limits)
- Recent data points (for deduplication checks)
- User session tokens for device APIs"

**8. Data Warehouse**
"CDC pipeline moves data to warehouse for analytics, ML models, reporting. Not in real-time path."

---

### 🔍 Step 4: Deep Dive on Key Areas

**They'll ask detailed questions. Be ready:**

#### Q: "How do you handle Fitbit webhooks specifically?"

**Your answer:**
"Fitbit sends a webhook notification when new data is available, but doesn't include the actual data. Our flow:

1. **Receive webhook** at `/webhooks/fitbit` endpoint
2. **Verify signature** using HMAC with Fitbit's secret key
3. **Parse notification** - contains user_id, data_type (steps/weight), date
4. **Queue job** with these params, return 200 OK immediately (under 3 seconds)
5. **Worker fetches data** by calling Fitbit API with OAuth token
6. **Store data** after deduplication check

**Retry logic:** If webhook times out, Fitbit retries 3 times with exponential backoff. We track processed webhooks by ID to handle duplicates."

#### Q: "How do you handle rate limits?"

**Your answer:**
"Multi-layered approach:

**Per-user rate limiting:**
- Store counter in Redis: `fitbit:user_123:api_calls:2024-11-01:15` (hour bucket)
- Before making API call, check: `INCR key` and `TTL 3600`
- If count > 150, delay the job for next hour

**Global rate limiting:**
- For Garmin's daily limit, track: `garmin:api_calls:2024-11-01`
- If approaching limit (e.g., 4800/5000), start queueing low-priority jobs for tomorrow

**Backoff on 429:**
- If we get rate limit response, parse `Retry-After` header
- Re-queue job with delay
- Update our quota counter

**Prioritization:**
- Recent data (today) = high priority queue
- Historical backfill = low priority queue
- Ensures recent data isn't blocked by backfill jobs"

#### Q: "What if Fitbit API is down?"

**Your answer:**
"Circuit breaker pattern:

**Detection:**
- Track failure rate in Redis: `fitbit:failures:count`
- If 10 consecutive failures OR 50% failure rate over 5 min, open circuit

**Open state (API down):**
- Stop making API calls to Fitbit
- Queue jobs for later retry
- Send alert to on-call
- Return cached data if available
- Show users 'Syncing...' status

**Half-open (testing recovery):**
- After 5 minutes, try one request
- If succeeds, close circuit and resume
- If fails, back to open for another 5 min

**Monitoring:**
- Alert: 'Fitbit integration down'
- Dashboard: Show success/failure rates per provider
- Log: Structured logs with provider, error type, user_id (hashed)"

#### Q: "How do you handle duplicate data?"

**Your answer:**
"Multiple strategies depending on data type:

**For discrete events (steps, activities):**
- Unique constraint: `(user_id, device_id, data_type, timestamp)`
- Before insert, check: `SELECT EXISTS WHERE ...`
- Use upsert if data might be corrected (e.g., device uploads revised step count)

**For continuous metrics (weight, glucose):**
- Accept most recent value for a given timestamp
- Track `last_updated_at` from device
- Only store if newer than what we have

**Cross-device deduplication:**
- If user has Fitbit AND Apple Health, both might report steps
- Use heuristic: Keep data from source user set as 'primary' device
- Or: Aggregate intelligently (pick higher value, assume non-overlapping)

**Idempotency:**
- Webhook processing: Store webhook_id, skip if already processed
- Job processing: Check if data point exists before calling API"

#### Q: "How do you handle initial sync of historical data?"

**Your answer:**
"Separate workflow:

**On device connection:**
1. User authorizes device (OAuth flow)
2. Queue `InitialSyncJob` with low priority
3. Job requests data in chunks (e.g., 7 days at a time, working backwards)
4. Rate-limited to not block real-time syncs
5. Progress tracked: `user_device_connections.sync_status = 'in_progress'`
6. On completion: `sync_status = 'completed'`, `last_synced_at = NOW()`

**Implementation:**
```ruby
class InitialSyncJob < ApplicationJob
  queue_as :low_priority
  
  def perform(user_device_connection_id, start_date, end_date)
    connection = UserDeviceConnection.find(user_device_connection_id)
    
    # Rate limit check
    return reschedule_later if rate_limited?(connection)
    
    # Fetch data for date range
    data = fetch_from_api(connection, start_date, end_date)
    
    # Store
    store_health_data(data)
    
    # Queue next chunk if more history
    if start_date > 3.months.ago
      InitialSyncJob.perform_later(
        user_device_connection_id,
        start_date - 7.days,
        start_date - 1.day
      )
    else
      mark_sync_complete(connection)
    end
  end
end
```

**Why this approach:**
- Chunked to avoid memory issues
- Low priority queue doesn't block real-time data
- Resumable if interrupted (tracks progress)
- Rate-limited to stay under API quotas"

---

### ⚖️ Step 5: Trade-offs & Alternatives

**Proactively discuss:**

**Webhook vs Polling trade-off:**
"I used a hybrid approach - webhooks as primary, polling as backup. 

**Why not pure webhook?**
- Not all providers support webhooks (Apple Health)
- Webhooks can be missed (network issues, our downtime)
- Member might disconnect/reconnect device

**Why not pure polling?**
- Uses more API quota
- Higher latency (15-30 min vs real-time)
- More load on our system

**Hybrid gives us reliability with efficiency."**

**TimescaleDB vs Plain Postgres:**
"Using TimescaleDB extension for time-series data.

**Benefit:**
- Automatic partitioning by time
- Optimized queries for date ranges
- Compression for old data
- Retention policies (auto-delete old data)

**Cost:**
- Slightly more complex
- Need to learn TimescaleDB specifics

**Alternative:** Could use plain Postgres with manual partitioning, but TimescaleDB does it better."

**Synchronous vs Asynchronous:**
"All processing is async through queues.

**Why not synchronous?**
- API calls can take 5-30 seconds
- Would tie up web workers
- Poor user experience
- Can't handle load spikes

**Why async works:**
- Fast webhook response (<200ms)
- Can scale workers independently
- Natural retry mechanism
- Better user experience (show 'syncing' status)"

---

### 📊 Step 6: Monitoring & Operations

**If they ask about observability:**

**Metrics to track:**
- Webhook success rate per provider
- API call latency (p50, p95, p99)
- Queue depth (jobs waiting)
- Worker utilization
- Data freshness (last sync time per user)
- API quota usage per provider

**Alerts:**
- Critical: Circuit breaker opened (provider down)
- High: Queue depth > 10,000 (processing falling behind)
- Medium: Sync delayed > 2 hours for >100 users
- Info: Approaching API rate limit (80% of quota)

**Logging:**
```ruby
logger.info({
  event: 'health_data_fetched',
  provider: 'fitbit',
  user_id: hashed_user_id, # HIPAA: don't log PII
  data_type: 'steps',
  records_count: 24,
  api_latency_ms: 234,
  success: true
})
```

**Dashboard:**
- Provider health (green/yellow/red per provider)
- Data pipeline flow (webhooks received → jobs queued → processed → stored)
- Per-user sync status
- API quota usage graphs

---

### ✅ Complete Solution Summary

**System handles:**
- ✅ Multiple providers (Fitbit, Garmin, Apple Health)
- ✅ Mixed patterns (webhooks + polling)
- ✅ Rate limit management
- ✅ Historical data backfill
- ✅ Deduplication across sources
- ✅ Failure handling and retries
- ✅ HIPAA compliance (encryption, audit logs)
- ✅ Scalability (50K users, 100K devices)
- ✅ Monitoring and alerting

**Key design decisions:**
1. Hybrid webhook/polling for reliability
2. Async processing via queues
3. Circuit breaker for provider failures
4. TimescaleDB for time-series data
5. Redis for rate limiting and caching
6. Separate low-priority queue for backfill

---

## Problem 2: Partner/Employer Integration Platform

### 📋 Prompt

**Interviewer says:**
"Omada partners with large employers and health plans. We need to sync member enrollment data bidirectionally with their HR systems. Design a service that:
- Receives daily SFTP file drops with new enrollments (CSV, 1000-50,000 records)
- Sends daily reports back to partners (enrollment status, program progress)
- Handles real-time eligibility checks via REST API
- Supports 200+ partners with different data formats

Some partners are very technical (REST APIs), others are not (SFTP files). Data formats vary: some use SSN, others employee IDs. File formats differ (CSV vs Excel vs fixed-width)."

---

### 🤔 Step 1: Clarifying Questions

**Ask these:**

1. **Scale:**
   - "How many total members?" → 500K active
   - "How many file uploads per day?" → 50-100 partners upload daily
   - "Largest file size?" → Up to 100MB (50K records)

2. **Requirements:**
   - "How fast must we process uploads?" → Within 4 hours
   - "Real-time API latency requirements?" → <500ms
   - "What if partner sends bad data?" → Validate, report errors, don't fail entire file

3. **Data:**
   - "What data do we receive?" → Name, DOB, employee ID, eligibility dates, health plan info
   - "What data do we send back?" → Enrollment status, completion milestones, health outcomes
   - "PII/PHI handling?" → Yes, HIPAA compliance required

4. **Integrations:**
   - "Do partners use same SFTP server or their own?" → Mix - some use our SFTP, some we connect to theirs
   - "API authentication?" → OAuth2 for modern partners, API keys for others

---

### 🎨 Step 2: High-Level Design

**Draw this:**

```
┌──────────────────────────────────────────────────────────────┐
│                     Partner Systems                           │
│                                                               │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │ SFTP 1  │  │ SFTP 2  │  │ REST API│  │ S3 Drop │        │
│  │(Push)   │  │(Pull)   │  │ Partner │  │(Event)  │        │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘        │
└───────┼────────────┼────────────┼─────────────┼─────────────┘
        │            │            │             │
        ↓            ↓            ↓             ↓
   ┌────────────────────────────────────────────────┐
   │           Ingestion Layer                       │
   │  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
   │  │SFTP Watch│  │SFTP Poller│ │API Gateway│    │
   │  │(Listener)│  │(Scheduled)│  │           │    │
   │  └─────┬────┘  └─────┬────┘  └─────┬────┘     │
   └────────┼─────────────┼─────────────┼───────────┘
            │             │             │
            └─────────────┴─────────────┘
                          ↓
                  ┌───────────────┐
                  │  File Store   │
                  │  (S3 Bucket)  │
                  └───────┬───────┘
                          ↓
                  ┌───────────────┐
                  │     Queue     │
                  │ (File Process)│
                  └───────┬───────┘
                          ↓
              ┌───────────────────────┐
              │  Processing Workers    │
              │                        │
              │ ┌────────────────────┐ │
              │ │ 1. Parse/Validate  │ │
              │ │ 2. Transform       │ │
              │ │ 3. Enrich          │ │
              │ │ 4. Load (Upsert)   │ │
              │ └────────────────────┘ │
              └───────┬───────────────┘
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
    ┌──────────┐          ┌──────────┐
    │ Database │          │ Partner  │
    │(Member   │          │ Config   │
    │ Data)    │          │  Store   │
    └─────┬────┘          └──────────┘
          │
          ↓
    ┌──────────┐
    │  Event   │
    │  Stream  │◄────── Triggers outbound sync
    └──────────┘
          │
          ↓
    ┌──────────────────┐
    │ Outbound Workers │
    │ (Generate Reports)│
    └────────┬─────────┘
             │
       ┌─────┴─────┐
       ↓           ↓
  ┌────────┐  ┌────────┐
  │SFTP Send│ │API POST│
  │         │  │        │
  └────────┘  └────────┘
```

---

### 💬 Step 3: Explain Components

**Ingestion Layer:**

**SFTP Watch (Listener):**
"For partners that push files to our SFTP server, we have a listener service (using inotify or similar) that detects new files immediately. When a file arrives, we move it to S3 and queue processing job."

**SFTP Poller (Scheduled):**
"For partners where we pull from their SFTP, we have scheduled jobs (cron/Sidekiq) that check for new files every hour or daily (partner preference). We track last processed file by timestamp/checksum to avoid reprocessing."

**API Gateway:**
"For real-time eligibility checks, we expose REST endpoints. Partners call `/api/v1/eligibility/check` with member info. We look up in our DB and return status synchronously. Authenticated via OAuth2 or API keys per partner."

**File Store (S3):**
"All files stored in S3 for:
- Durability (don't lose files)
- Audit trail (keep original files)
- Replay capability (reprocess if needed)
- Cost-effective storage

Organized: `s3://partner-files/{partner_id}/{year}/{month}/{day}/{filename}`"

**Processing Workers:**
"Four-stage pipeline:

1. **Parse:** Detect format (CSV, Excel, fixed-width), parse into records
2. **Validate:** Check required fields, data types, business rules
3. **Transform:** Map partner format to our schema using config
4. **Load:** Upsert members (update if exists, insert if new)

Each stage logs progress. If validation fails, we don't stop - we process valid records and generate error report."

**Partner Config Store:**
"Database table with partner-specific configurations:
- File format (CSV, Excel, fixed-width)
- Column mappings (their 'EMP_ID' → our 'employee_id')
- Required fields
- Business rules (e.g., only enroll if eligible_date <= today)
- Credentials (SFTP host, API keys)
- Schedule (daily, weekly, monthly)

This allows adding new partners without code changes."

**Event Stream:**
"When member data changes (new enrollment, status update), we publish event. Triggers outbound sync to send updates back to partner. Decouples inbound from outbound processing."

**Outbound Workers:**
"Generate reports for partners:
- Query members for this partner
- Format per their spec (CSV vs API payload)
- Encrypt if required
- Upload to their SFTP or POST to their API
- Mark as sent, track delivery"

---

### 🔍 Step 4: Deep Dive Questions

#### Q: "How do you handle different file formats from 200+ partners?"

**Your answer:**
"Configuration-driven approach to avoid custom code per partner:

**Partner Configuration:**
```ruby
{
  partner_id: 'acme_corp',
  inbound: {
    format: 'csv',
    delimiter: ',',
    encoding: 'UTF-8',
    has_header: true,
    column_mappings: {
      'Employee ID': 'employee_id',
      'First Name': 'first_name',
      'Last Name': 'last_name',
      'DOB': 'date_of_birth',
      'Elig Date': 'eligibility_start_date'
    },
    date_format: '%m/%d/%Y',
    required_fields: ['employee_id', 'first_name', 'last_name', 'date_of_birth'],
    validations: {
      'date_of_birth': { type: 'date', max: 'today' },
      'employee_id': { type: 'string', pattern: '^[A-Z0-9]{6,10}$' }
    }
  },
  outbound: {
    format: 'csv',
    columns: ['employee_id', 'enrollment_status', 'completion_date'],
    schedule: 'daily',
    delivery: {
      method: 'sftp',
      host: 'ftp.acmecorp.com',
      path: '/inbound/omada/'
    }
  }
}
```

**Generic Parser:**
```ruby
class FileProcessor
  def process(file, partner_config)
    # Detect and parse based on config
    records = parse_file(file, partner_config[:inbound][:format])
    
    # Transform using column mappings
    records.map do |record|
      transform_record(record, partner_config[:inbound][:column_mappings])
    end
  end
  
  def parse_file(file, format)
    case format
    when 'csv'
      CSV.parse(file, headers: true)
    when 'excel'
      Roo::Excelx.new(file).parse
    when 'fixed_width'
      parse_fixed_width(file, partner_config[:inbound][:field_widths])
    end
  end
end
```

**Why this works:**
- Add new partner by creating config, no code deploy
- Standardized validation logic
- Easy to debug (see exact config used)
- Testable (mock different configs)"

#### Q: "What if a file has 10,000 records and 5,000 are invalid?"

**Your answer:**
"Partial success strategy:

**Processing approach:**
1. Parse all records
2. Validate each record independently
3. Collect errors per record
4. Process valid records (upsert to DB)
5. Generate error report with invalid records

**Implementation:**
```ruby
class FileImportJob
  def perform(file_id, partner_id)
    file = File.find(file_id)
    partner_config = PartnerConfig.find(partner_id)
    
    results = {
      total: 0,
      success: 0,
      errors: []
    }
    
    records = parse_file(file.path, partner_config)
    
    records.each_with_index do |record, index|
      results[:total] += 1
      
      # Validate
      validation_result = validate_record(record, partner_config)
      
      if validation_result.valid?
        # Process valid record
        Member.upsert(transform_record(record, partner_config))
        results[:success] += 1
      else
        # Collect error
        results[:errors] << {
          line: index + 2, # +2 for header row and 0-index
          record: record,
          errors: validation_result.errors
        }
      end
    end
    
    # Store results
    FileImport.create!(
      file: file,
      partner: partner,
      processed_at: Time.current,
      total_records: results[:total],
      success_count: results[:success],
      error_count: results[:errors].size,
      errors_json: results[:errors].to_json
    )
    
    # Generate error report if any errors
    if results[:errors].any?
      generate_error_report(file, results[:errors])
      notify_partner_of_errors(partner, file)
    end
    
    # Notify success
    notify_import_complete(partner, file, results)
  end
end
```

**Error Report (CSV):**
```csv
Line,Employee_ID,First_Name,Last_Name,Errors
5,ABC123,,Smith,"First name is required"
10,XYZ789,John,Doe,"Invalid date of birth: 'not-a-date'"
```

**Why this approach:**
- Don't fail entire file for some bad records
- Partner gets actionable error report
- Valid records still processed (no wasted effort)
- Maintains audit trail
- Can resubmit just errors after fixing"

#### Q: "How do you ensure idempotency if a partner resubmits the same file?"

**Your answer:**
"Multi-layer idempotency:

**File-level:**
- Track processed files by MD5 checksum
- Before processing: `SELECT * FROM file_imports WHERE checksum = ? AND partner_id = ?`
- If exists and succeeded: Skip processing, return success
- If exists and failed: Allow reprocessing

**Record-level:**
- Use upsert (UPDATE if exists, INSERT if new)
- Primary key: `(partner_id, employee_id)` or similar partner-specific identifier
- Track `last_updated_at` and only update if newer data

**Implementation:**
```ruby
def process_file(file)
  checksum = Digest::MD5.hexdigest(file.read)
  
  # Check if already processed
  existing = FileImport.find_by(
    partner_id: partner_id,
    checksum: checksum,
    status: 'completed'
  )
  
  return existing if existing.present?
  
  # Process file...
  
  # Upsert records
  members_data.each do |member|
    Member.upsert(
      {
        partner_id: partner_id,
        employee_id: member[:employee_id],
        first_name: member[:first_name],
        # ... other fields
        updated_at: Time.current
      },
      unique_by: [:partner_id, :employee_id]
    )
  end
end
```

**Why this works:**
- Safe to reprocess (no duplicates)
- Handles network failures (can retry)
- Handles partner mistakes (resubmit same file)
- Audit trail shows all submissions"

#### Q: "How do you handle the real-time API for eligibility checks?"

**Your answer:**
"Synchronous API with caching:

**API Design:**
```ruby
# POST /api/v1/eligibility/check
# Request:
{
  "partner_id": "acme_corp",
  "employee_id": "EMP123456",
  "date_of_birth": "1990-01-15"
}

# Response (200 OK):
{
  "eligible": true,
  "program": "diabetes_prevention",
  "eligibility_start_date": "2024-01-01",
  "eligibility_end_date": "2024-12-31"
}

# Response (404 Not Found):
{
  "eligible": false,
  "reason": "Employee not found in eligibility data"
}
```

**Implementation:**
```ruby
class Api::V1::EligibilityController < ApiController
  before_action :authenticate_partner
  before_action :rate_limit_check
  
  def check
    # Input validation
    params_valid = validate_params(params)
    return render_error(422, params_valid.errors) unless params_valid.success?
    
    # Check cache first
    cache_key = cache_key_for(params)
    cached = Rails.cache.read(cache_key)
    return render json: cached if cached.present?
    
    # Lookup in database
    member = Member.find_by(
      partner_id: @current_partner.id,
      employee_id: params[:employee_id]
    )
    
    # Verify DOB matches (security check)
    if member.blank? || member.date_of_birth != Date.parse(params[:date_of_birth])
      response = { eligible: false, reason: 'Member not found or DOB mismatch' }
      return render json: response, status: 404
    end
    
    # Check eligibility
    result = check_eligibility(member)
    
    # Cache result (TTL 1 hour)
    Rails.cache.write(cache_key, result, expires_in: 1.hour)
    
    render json: result
  end
  
  private
  
  def check_eligibility(member)
    eligible = member.eligibility_start_date <= Date.today &&
               member.eligibility_end_date >= Date.today &&
               member.enrollment_status != 'terminated'
    
    {
      eligible: eligible,
      program: member.program_type,
      eligibility_start_date: member.eligibility_start_date,
      eligibility_end_date: member.eligibility_end_date
    }
  end
  
  def cache_key_for(params)
    "eligibility:#{@current_partner.id}:#{params[:employee_id]}"
  end
end
```

**Performance optimizations:**
- Cache results (1 hour TTL)
- Database indexes on lookup fields
- Connection pooling
- Read replica for queries

**Rate limiting:**
- Per partner: 1000 requests/minute
- Tracked in Redis: `partner:acme_corp:api_calls:minute:2024-11-01-15-30`

**Monitoring:**
- P95 latency < 200ms (alert if > 500ms)
- Success rate > 99%
- Cache hit rate > 80%"

---

### ⚖️ Step 5: Trade-offs

**File-based vs API integration:**
"Hybrid approach supports both.

**File benefits:**
- Partner doesn't need technical integration
- Batch processing is cost-effective
- Natural for daily updates

**API benefits:**
- Real-time data
- Better error handling (immediate feedback)
- More flexible

**Why support both:** Partners have different capabilities. Large employers have APIs, small ones use files."

**Sync vs Async processing:**
"File processing is async (background jobs).

**Why not sync?**
- Files can be large (100MB, 50K records)
- Processing takes 5-30 minutes
- Would timeout HTTP request

**API is sync:**
- Need real-time response
- Simple lookup (fast)
- Cached for performance"

---

### ✅ Complete Solution Summary

**System handles:**
- ✅ Multiple ingestion methods (SFTP push/pull, API, S3)
- ✅ 200+ partners with different formats
- ✅ Partial success (process valid, report invalid)
- ✅ Idempotency (safe to reprocess)
- ✅ Real-time eligibility API
- ✅ Bidirectional sync
- ✅ HIPAA compliance
- ✅ Scale (500K members, 100 files/day)

---

## Problem 3: Multi-Channel Notification Service

### 📋 Prompt

**Interviewer says:**
"Design a notification service that sends messages to Omada members via email, SMS, and in-app notifications. The service needs to:
- Send appointment reminders, health tips, coach messages
- Support personalization (member name, coach name, upcoming appointments)
- Handle delivery failures and retries
- Support priority levels (urgent vs informational)
- Track delivery status and read receipts
- Scale to 500K users, millions of notifications per day"

---

### 🤔 Clarifying Questions

**Ask:**
1. "What triggers notifications?" → Multiple: scheduled (reminders), event-driven (coach sends message), system-generated (health tips)
2. "Latency requirements?" → Urgent: <1 minute, Normal: <5 minutes, Low priority: <1 hour
3. "What providers?" → SendGrid (email), Twilio (SMS), Firebase (push), WebSocket (in-app)
4. "User preferences?" → Yes, users can opt out of certain types, choose channels
5. "Rate limits?" → SendGrid 100K/day, Twilio varies by plan

---

### 🎨 High-Level Design

```
┌───────────────────────────────────────────────────────────┐
│                  Trigger Sources                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │Scheduled │  │  Event   │  │  Coach   │              │
│  │  (Cron)  │  │(Webhooks)│  │Dashboard │              │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘              │
└───────┼─────────────┼──────────────┼────────────────────┘
        │             │              │
        └─────────────┴──────────────┘
                      ↓
            ┌─────────────────┐
            │ Notification API │
            │  (Create Notif)  │
            └────────┬────────┘
                     ↓
            ┌─────────────────┐
            │   Preferences   │
            │     Check       │
            │ (Can we send?)  │
            └────────┬────────┘
                     ↓
            ┌─────────────────┐
            │  Template       │
            │  Rendering      │
            │ (Personalize)   │
            └────────┬────────┘
                     ↓
            ┌─────────────────┐
            │  Priority Queue │
            │  (Redis/SQS)    │
            │  High/Med/Low   │
            └────────┬────────┘
                     ↓
        ┌────────────┴────────────┐
        ↓                         ↓
┌───────────────┐         ┌───────────────┐
│ Urgent Workers│         │Normal Workers │
│  (High CPU)   │         │  (Standard)   │
└───────┬───────┘         └───────┬───────┘
        │                         │
        └────────────┬────────────┘
                     ↓
        ┌────────────────────────┐
        │   Channel Dispatcher    │
        │  (Route by channel)     │
        └─────┬──────────┬────────┘
              │          │         
     ┌────────┘          └────────┐
     ↓                            ↓
┌─────────┐  ┌─────────┐  ┌─────────┐
│  Email  │  │   SMS   │  │  Push   │
│SendGrid │  │ Twilio  │  │Firebase │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┴────────────┘
                  ↓
        ┌───────────────────┐
        │ Delivery Tracking │
        │    (Status DB)    │
        └───────────────────┘
```

---

### 💬 Key Components Explained

**Template System:**
```ruby
# Template with placeholders
subject: "Appointment Reminder"
body: "Hi {{member_name}}, your appointment with {{coach_name}} is on {{appointment_date}} at {{appointment_time}}."

# Rendered
subject: "Appointment Reminder"
body: "Hi Sarah, your appointment with Coach Mike is on Nov 5th at 2:00 PM."
```

**Priority Queue:**
- Urgent queue: Appointment changes, critical health alerts
- Normal queue: Daily tips, coach messages
- Low priority: Marketing, newsletters

**Channel Dispatcher:**
```ruby
def dispatch(notification)
  channels = determine_channels(notification.user, notification.type)
  
  channels.each do |channel|
    case channel
    when 'email'
      EmailWorker.perform_async(notification.id)
    when 'sms'
      SmsWorker.perform_async(notification.id)
    when 'push'
      PushWorker.perform_async(notification.id)
    end
  end
end
```

**Delivery Tracking:**
```ruby
create_table :notification_deliveries do |t|
  t.references :notification
  t.string :channel # email, sms, push
  t.string :provider_message_id
  t.string :status # pending, sent, delivered, failed
  t.json :provider_response
  t.datetime :sent_at
  t.datetime :delivered_at
  t.datetime :failed_at
  t.integer :retry_count, default: 0
  t.timestamps
end
```

---

### 🔍 Deep Dive Questions

#### Q: "How do you handle SMS rate limits and costs?"

**Your answer:**
"Multi-layer strategy:

**Rate limiting:**
- Track sends per minute in Redis: `sms:sent:minute:2024-11-01-15:30`
- Before sending, check count < limit
- If at limit, delay job for next minute

**Cost optimization:**
- SMS is expensive ($0.01-0.05 per message)
- Only send SMS for urgent notifications
- Prefer email for non-urgent
- Batch SMS sends (Twilio allows batching)
- User preference: 'SMS only for urgent'

**Implementation:**
```ruby
class SmsWorker
  def perform(notification_id)
    notification = Notification.find(notification_id)
    
    # Check if we can send (rate limit)
    unless can_send_sms?
      # Retry in 1 minute
      self.class.perform_in(1.minute, notification_id)
      return
    end
    
    # Send via Twilio
    response = TwilioClient.send_sms(
      to: notification.user.phone_number,
      body: notification.body
    )
    
    # Track delivery
    NotificationDelivery.create!(
      notification: notification,
      channel: 'sms',
      provider_message_id: response.sid,
      status: 'sent',
      sent_at: Time.current
    )
    
    # Increment counter
    increment_sms_count
  end
  
  private
  
  def can_send_sms?
    key = "sms:sent:minute:#{Time.current.strftime('%Y-%m-%d-%H:%M')}"
    count = Redis.current.incr(key)
    Redis.current.expire(key, 90) # 90 seconds
    count <= SMS_RATE_LIMIT_PER_MINUTE
  end
end
```

**Why this works:**
- Respects rate limits
- Controls costs
- Graceful degradation (delay, don't fail)"

#### Q: "How do you handle delivery failures and retries?"

**Your answer:**
"Exponential backoff with max attempts:

**Retry strategy:**
- Attempt 1: Immediate
- Attempt 2: 5 minutes later
- Attempt 3: 30 minutes later  
- Attempt 4: 2 hours later
- Give up after 4 attempts (or 24 hours for non-urgent)

**Error handling:**
```ruby
class EmailWorker
  retry_in_exponential_backoff # Sidekiq feature
  
  def perform(notification_id, attempt = 0)
    notification = Notification.find(notification_id)
    delivery = find_or_create_delivery(notification, 'email')
    
    begin
      response = SendGridClient.send_email(
        to: notification.user.email,
        subject: notification.subject,
        body: notification.body
      )
      
      delivery.update!(
        status: 'sent',
        provider_message_id: response.message_id,
        sent_at: Time.current
      )
      
    rescue SendGrid::RateLimitError => e
      # Retry after rate limit window
      delivery.update!(status: 'rate_limited')
      raise # Let Sidekiq handle retry
      
    rescue SendGrid::InvalidEmailError => e
      # Don't retry - bad email address
      delivery.update!(
        status: 'failed',
        error_message: e.message,
        failed_at: Time.current
      )
      notify_admin_of_invalid_email(notification.user)
      
    rescue => e
      # General error - retry
      delivery.increment!(:retry_count)
      
      if delivery.retry_count >= MAX_RETRIES
        delivery.update!(
          status: 'failed',
          error_message: e.message,
          failed_at: Time.current
        )
        notify_admin_of_failed_delivery(notification)
      else
        raise # Let Sidekiq retry
      end
    end
  end
end
```

**Dead letter queue:**
- After max retries, move to DLQ
- Manual review and reprocess
- Alert on-call if DLQ size > threshold"

---

### ⚖️ Trade-offs

**Push vs Pull for in-app:**
"Using WebSocket for real-time in-app.

**Push (WebSocket):**
- Real-time delivery
- Complex infrastructure
- Connection management

**Pull (Polling):**
- Simple
- Higher latency
- More load on API

**Choice: Push for logged-in users, poll for offline.**"

---

### ✅ Solution Summary

Handles:
- ✅ Multi-channel (email, SMS, push)
- ✅ Priority queuing
- ✅ Rate limiting per channel
- ✅ Retry with exponential backoff
- ✅ Delivery tracking
- ✅ User preferences
- ✅ Template personalization
- ✅ Scale (millions/day)

---

## 🎯 Practice Tips

### For Each Problem:

1. **Timebox: 30-40 minutes**
   - 5 min: Clarifying questions
   - 10 min: Draw high-level design
   - 15 min: Deep dive on 2-3 areas
   - 5 min: Discuss trade-offs

2. **Use Real Whiteboard/Paper**
   - Practice drawing clearly
   - Label components
   - Show data flow with arrows

3. **Talk Out Loud**
   - Explain as you draw
   - "I'm choosing X because..."
   - Ask yourself questions

4. **Practice Iteration**
   - Start simple
   - Add complexity based on "questions"
   - Show you can scale up

5. **Record Yourself (Optional)**
   - Video of whiteboard session
   - Watch back - are you clear?
   - Time yourself

### What Good Looks Like:

✅ Clear diagram with labeled components  
✅ Explained reasoning for each decision  
✅ Discussed at least 2 failure scenarios  
✅ Mentioned monitoring/observability  
✅ Considered trade-offs  
✅ Stayed within time  
✅ Responded well to "what if" questions

---

## 📚 Additional Practice Problems (Quick)

### Problem 4: Audit Logging Service
"Design a system to log all user actions (views, edits, deletes) for HIPAA audit compliance. Must retain logs for 7 years, support fast queries by user/date, handle 10M events/day."

### Problem 5: File Upload Service
"Design a service to handle large file uploads (health records, PDFs up to 100MB). Support resumable uploads, virus scanning, OCR for text extraction, secure storage."

### Problem 6: Real-time Chat Between Coaches and Members
"Design a messaging system for coaches to chat with members. Support text, images, read receipts. Must work on web and mobile. ~1000 concurrent chats."

---

## ✅ You're Ready When:

- [ ] Can draw clean architecture diagram in 5 minutes
- [ ] Can explain each component's purpose clearly
- [ ] Can discuss 3 failure scenarios
- [ ] Can propose 2 alternative approaches
- [ ] Can answer "how do you scale this?" confidently
- [ ] Can work through problem in 30-40 minutes

**Now go practice! Pick one problem, set timer, and try it. Then compare with solution.**

Good luck! 🚀
