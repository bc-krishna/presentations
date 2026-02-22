# Verification & Indemnity Architecture

**Status:** Draft - Architecture Design
**Created:** February 22, 2026
**Authors:** BC, Claude

---

## Executive Summary

Transform supplier verification from a cost center into a revenue-generating indemnity product while improving risk management for buyers. Current market: most buyers do NOTHING and bear all supplier fraud risk themselves.

**Key Insight:** Don't just identify risk - transfer it.

---

## Identity Verification by Entity Type

### Overview

Exchange provides differentiated verification flows based on entity type and jurisdiction, leveraging free government APIs where available to maximize verification strength while minimizing cost.

**Coverage Summary:**
- **US (Business + Consumer):** ~100% via TINCheck + Melissa
- **EU (27 countries):** ~100% via VIES VAT validation
- **UK:** ~100% via Companies House API
- **Other International:** Address validation + AML only

**Combined Coverage:** ~70-80% of international suppliers can receive strong identity verification via free government APIs.

---

### US Business Verification

**Flow:**
1. **Address Validation** (Smarty) - Already implemented
   - Validates and standardizes business address
   - Confirms address exists and is deliverable

2. **TIN Validation** (TINCheck)
   - Supplier enters EIN (text entry, not W-9 upload)
   - Validate EIN + business name against IRS database
   - Handle fuzzy matching (DBA, abbreviations, legal name variations)

3. **Name Mismatch Handling**
   - If name doesn't match: Show IRS registered name
   - Options: [Try Again] or [Accept As Is]
   - If accepting: Capture reason (DBA, misspelling, division, legal vs trade name)
   - Convey mismatch to buyer with explanation

4. **AML/PEP Screening** (Didit)
   - Screen against global watchlists, sanctions, PEP databases
   - Generate AI summary if issues found

**Result:** "Verified US Business" with optional name variation notes
**Trust Score:** 30-35 points (Identity component)
**Cost:** ~$1.50-2.00 (TINCheck + Smarty + AML)

---

### US Consumer Verification

**Flow:**
1. **SSN4 + Address Validation** (Melissa Personator) - Already implemented
   - Supplier enters last 4 digits of SSN
   - Validate SSN4 + full name + address
   - Confirms identity match

2. **AML/PEP Screening** (Didit)
   - Screen individual against PEP databases, sanctions lists
   - Check for adverse media, watchlist matches

**Result:** "Verified US Consumer"
**Trust Score:** 28-32 points (Identity component)
**Cost:** ~$1.50-2.00 (Melissa + AML)

---

### EU Business Verification (27 Countries) ⭐

**Countries Covered:**
Austria, Belgium, Bulgaria, Croatia, Cyprus, Czech Republic, Denmark, Estonia, Finland, France, Germany, Greece, Hungary, Ireland, Italy, Latvia, Lithuania, Luxembourg, Malta, Netherlands, Poland, Portugal, Romania, Slovakia, Slovenia, Spain, Sweden

**Flow:**
1. **Address Validation** (Smarty International)
   - Validates international address format
   - Standardizes address where possible

2. **VAT Number Validation** (VIES - VAT Information Exchange System)
   - **API:** Free, official EU government service
   - **Endpoint:** `https://ec.europa.eu/taxation_customs/vies/`
   - Supplier enters VAT number (country code + number)
   - System validates against VIES database
   - Returns: Valid/invalid, registered company name, registered address

3. **Name Matching**
   - Compare VIES registered name with buyer-provided name
   - Handle fuzzy matching (abbreviations, legal entity suffixes like "SA", "GmbH", "Ltd")
   - If mismatch: Same flow as US TIN (try again or accept with reason)

4. **AML/PEP Screening** (Didit)
   - Screen against global databases
   - EU entities may have more comprehensive data

**Example:**
```
Supplier: Melia Hotels International (Spain)
VAT Input: ES + B57254090
VIES Response:
  Valid: true
  Name: MELIA HOTELS INTERNATIONAL SA
  Address: CALLE GREMIO DE TONELEROS 24, PALMA, BALEARES 07009

Result: ✓ Verified (name variation: legal entity suffix)
```

**Result:** "Verified EU Business"
**Trust Score:** 35 points (same as US business - full verification)
**Cost:** ~$1.00-1.50 (Smarty + VIES free + AML)

**Implementation Notes:**
- VIES API is SOAP-based but simple
- Can use REST wrapper libraries (`vat-validator` npm package)
- Rate limits: ~100 requests/minute (reasonable for our use)
- Reliability: High (official EU infrastructure)
- No API key required

---

### UK Business Verification ⭐

**Flow:**
1. **Address Validation** (Smarty International)
   - Validates UK address (Royal Mail database)
   - Standardizes to UK postal format

2. **Company Number Validation** (Companies House API)
   - **API:** Free with API key (registration required)
   - **Endpoint:** `https://api.company-information.service.gov.uk/`
   - Supplier enters Company Number (8 digits or 2 letters + 6 digits)
   - System validates against Companies House registry
   - Returns: Company name, status (active/dissolved), registered address, formation date, company type

3. **Enhanced Data Available:**
   - Company status (active, dissolved, liquidation, etc.)
   - Company type (private limited, PLC, LLP, etc.)
   - Formation date
   - Previous company names
   - Directors (optional - can retrieve if needed)
   - SIC codes (business activities)

4. **Name Matching**
   - Compare Companies House name with buyer-provided name
   - Handle UK-specific variations ("Limited", "Ltd", "LLP", etc.)
   - If mismatch: Same accept/retry flow

5. **AML/PEP Screening** (Didit)
   - Screen UK entity against global databases

**Example:**
```
Supplier: Melia Hotels International Limited (UK)
Company Number: 03977902
Companies House Response:
  company_name: "MELIA HOTELS INTERNATIONAL LIMITED"
  company_status: "active"
  company_number: "03977902"
  registered_office_address: {...}
  date_of_creation: "2000-04-28"

Result: ✓ Verified
```

**Result:** "Verified UK Business"
**Trust Score:** 35 points (comprehensive verification)
**Cost:** ~$1.00-1.50 (Smarty + Companies House free + AML)

**Implementation Notes:**
- REST API (easier than VIES SOAP)
- Requires free API key (register at developer.company-information.service.gov.uk)
- Rate limit: 600 requests per 5 minutes
- Reliability: Excellent (official UK government service)
- Well-documented API with detailed responses

---

### Other International (No Tax ID API)

**Applies to:** Canada, Australia, Mexico, Asia, Latin America, Africa (any country without free tax ID API)

**Flow:**
1. **Address Validation** (Smarty International)
   - Validates address format and existence
   - Weaker than government registry confirmation

2. **AML/PEP Screening** (Didit)
   - Primary verification mechanism
   - Screen against global databases
   - Particularly important given lack of government registry check

3. **Bank Account Verification** (Next step)
   - Lyons/Finicity bank account ownership verification
   - Proves supplier owns the payment account
   - Adds significant verification layer

**Result:** "Verified International Entity" (with acknowledged gaps)
**Trust Score:** 22-28 points (lower due to verification gaps)
**Cost:** ~$1.00 (Smarty + AML)

**Gap Mitigation:**
- Bank account verification (next step) is strong
- AML screening is comprehensive
- Combined: Address + AML + Bank = "good enough" for most suppliers

**Future Enhancement Options:**
- Australia ABN Lookup (free but requires GUID registration)
- OpenCorporates (paid international business registry)
- Country-specific paid services for high-value suppliers

---

### Verification Coverage Summary

| Entity Type | Address | Tax ID | Tax ID Service | AML | Trust Score | Cost |
|-------------|---------|--------|----------------|-----|-------------|------|
| **US Business** | ✓ Smarty | ✓ EIN | TINCheck (paid) | ✓ Didit | 30-35 pts | ~$2.00 |
| **US Consumer** | ✓ Buyer data | ✓ SSN4 | Melissa (paid) | ✓ Didit | 28-32 pts | ~$2.00 |
| **EU Business** | ✓ Smarty Intl | ✓ VAT | VIES (FREE) | ✓ Didit | 35 pts | ~$1.50 |
| **UK Business** | ✓ Smarty Intl | ✓ Company# | Companies House (FREE) | ✓ Didit | 35 pts | ~$1.50 |
| **Other Intl** | ✓ Smarty Intl | ✗ None | N/A | ✓ Didit | 22-28 pts | ~$1.00 |

**International Supplier Coverage Estimate:**
```
EU (27 countries):        40-50%  → Strong verification via VIES
UK:                       10-15%  → Strong verification via Companies House
Other (strong):            0-5%   → Could add Australia ABN (free)
───────────────────────────────────────────────────────
Can strongly verify:      50-70%  → Same level as US businesses

Other (gaps):             30-50%  → Address + AML + Bank verification
```

**Result:** 50-70% of international suppliers receive government-backed identity verification at no incremental API cost (VIES and Companies House are free).

---

### Implementation Priority

**Phase 1 (Immediate):**
1. US Business - TINCheck ✓ (Already scoped)
2. US Consumer - SSN4 + Melissa ✓ (Already built)
3. Basic AML for all ✓ (Just implemented)

**Phase 2 (Next Sprint):**
4. EU VAT validation via VIES
   - 27 countries with single API
   - Biggest international coverage gain
   - Free API = no marginal cost
   - Estimated: 2-3 days development

**Phase 3 (Following Sprint):**
5. UK Companies House validation
   - Strong verification for UK suppliers
   - Also free API
   - Estimated: 1-2 days development

**Phase 4 (Future):**
6. Australia ABN (optional - small coverage gain)
7. Other country-specific APIs as needed

---

## Business Model

### Three-Tier Approach

#### 🆓 Tier 1: BASIC (Free - Included for All Suppliers)

**What Runs:**
- TIN Validation (W-9 EIN check for US, W-8BEN-E for international)
- Basic AML/PEP Screening (Didit API)

**What Suppliers See:**
- Badge: "✓ Verified" (green) if clean
- Badge: "⚠️ Review Required" (yellow) if issues found

**What Buyers See - Clean Result:**
- Simple status: "Screened - No Flags"
- Green verification badge
- Trust score reflects basic verification (25-28 points)

**What Buyers See - Issues Found:**
```
⚠️ VERIFICATION ISSUES FOUND

AI Summary (Free):
"This supplier has 3 watchlist matches related to historical
business dealings with Iran (2016-2019). Project was cancelled
before sanctions took effect. Current risk level: Medium.

Match Details:
• Iran UANI Business Registry - Hotel development project
• No active sanctions matches
• No PEP matches

Recommendation: Review business relationship context before
proceeding with high-value transactions."

[See Full Detailed Report - $25]
[Get Indemnity Protection - Starting at $75/month]
```

**Cost to Centime:** ~$0.50-1.00 per supplier (TIN validation + basic AML API call)

**Trust Score Impact:**
- Clean result: Identity component gets 25-28 points
- Issues found: Identity component gets 15-22 points (risk-adjusted)

---

#### 📄 Tier 2: DETAILED REPORT (One-time purchase: $25-50)

**What's Included:**
- Everything in Basic tier
- **Middesk Full Business Verification** (US entities only)
  - Legal entity confirmation
  - Business registration status
  - Registered address validation
  - Formation date, entity type, registered agent
  - Active/dissolved status
- **Detailed AML/PEP Analysis**
  - All hits with match scores and confidence levels
  - Complete adverse media summaries with sources
  - Linked entities and relationships
  - Risk breakdown by category (crimes, countries, sanctions, PEP)
  - Historical screening data
- **AI-Generated Executive Summary**
  - Claude analyzes all data (Middesk + AML)
  - 2-3 paragraph plain-English summary
  - Risk factors identified and explained
  - Clear recommendation (Approve / Review / Decline)
  - Action items for buyer
- **Formatted PDF Report**
  - Professional layout
  - Exportable/shareable
  - Timestamped and versioned

**Validity:** 90 days (can re-purchase if stale)

**Cost to Centime:** ~$3-5 per report (Middesk API + detailed AML + Claude API)
**Price to Buyer:** $25-50 per report
**Margin:** ~$20-45 per report

**Trust Score Impact:**
- Full detailed report: Identity component gets full 35 points
- Demonstrates comprehensive due diligence

**Reality Check:** Users likely won't buy this often - it's positioning for Tier 3.

---

#### 💎 Tier 3: INDEMNITY PROTECTION (Subscription: $75/month)

**The Core Product - Where Real Value Lives**

**What's Included:**

1. **Fraud Indemnity Coverage**
   - Protection against supplier fraud losses
   - Coverage: Up to $X per incident (TBD: $25K? $50K? $100K?)
   - Annual aggregate: $Y (TBD: $250K? $500K?)
   - Deductible: $Z (TBD: $0? $1K? $2.5K?)
   - Covered events:
     - Payment fraud (fake bank accounts, wire fraud)
     - Identity fraud (impersonation, fake businesses)
     - Invoice fraud (duplicate payments, altered invoices)
   - Exclusions:
     - Product quality disputes
     - Delivery/performance issues
     - Contract disputes (non-fraud)
     - Willful negligence by buyer

2. **50 Detailed Verification Reports/Year**
   - Full Middesk + AML reports
   - Value: $1,250-2,500/year if purchased individually
   - Unused reports don't roll over

3. **Ongoing Monitoring & Alerts**
   - Quarterly AML re-screening of all suppliers
   - Real-time alerts if supplier status changes
   - Watchlist additions/changes
   - Adverse media updates
   - Monitoring doesn't count against 50 report limit

4. **AI Risk Summaries for All Suppliers**
   - Free AI-generated summaries (like Tier 1) for entire supplier base
   - Risk dashboard showing portfolio overview
   - Trend analysis and risk scoring

5. **Priority Support**
   - Dedicated fraud prevention assistance
   - 4-hour response time for fraud alerts
   - Monthly risk review calls
   - Best practices guidance

6. **Centime Has Skin in the Game**
   - We verify suppliers on your behalf
   - We indemnify fraud losses
   - Our incentives are aligned with buyer protection

**Pricing:**
- **$75/month** ($900/year)
- OR **$750/year** (save $150 - 2 months free)
- Enterprise pricing for high-volume buyers (TBD)

**Cost to Centime:**
- Verification costs: ~$50-150/year (depends on supplier count & report usage)
- Monitoring: ~$25-50/year (quarterly AML rescreening)
- Insurance reserve/risk pool: TBD (actuarial analysis needed)
- Support overhead: ~$100-200/year per subscriber

**Target Customers:**
- Companies with 10+ suppliers
- Processing $500K+ annually through supplier network
- High-risk industries (construction, professional services, manufacturing)
- Companies that have experienced supplier fraud before

**Value Proposition:**
> "Most payment platforms leave you exposed to supplier fraud. We don't just identify risk - we transfer it. Get comprehensive verification, ongoing monitoring, and fraud protection for less than $3/day."

---

## Technical Architecture

### Verification Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Supplier Identity Verification Flow                         │
└─────────────────────────────────────────────────────────────┘

1. Supplier clicks "Verify Identity" in Profile
   │
   ├─→ Detect entity type (us_business, international_business, etc.)
   │
   ├─→ Check if TIN document exists (W-9 or W-8BEN-E)
   │   ├─→ If not: Prompt to upload first
   │   └─→ If yes: Extract TIN/EIN
   │
   ├─→ Run TIN Validation
   │   ├─→ US: IRS TIN Match API (via existing W-9 validation)
   │   └─→ International: W-8BEN-E format validation
   │
   ├─→ Run Basic AML/PEP Screening (Didit)
   │   ├─→ POST /api/didit/aml-screening
   │   ├─→ entity_type: 'company' or 'person'
   │   ├─→ full_name: Business/Person name
   │   ├─→ nationality: Country code
   │   └─→ Returns: status, risk_score, total_hits, hits[]
   │
   ├─→ Save results to identity_verification table
   │   ├─→ verification_method: 'tin_aml_basic'
   │   ├─→ verification_data: { tin_valid, aml_status, risk_score }
   │   └─→ verified_at: timestamp
   │
   ├─→ Generate AI Summary (if issues found)
   │   ├─→ Pass AML hits to Claude API
   │   ├─→ Prompt: "Analyze these AML screening results and provide a plain-English summary for a business user..."
   │   └─→ Returns: 2-3 paragraph summary + recommendation
   │
   └─→ Update Trust Score
       └─→ Calculate identity component based on verification tier

2. Buyer views supplier in Client Portal
   │
   ├─→ If clean: Show green "✓ Verified" badge
   │
   └─→ If issues: Show yellow "⚠️ Issues Found" badge
       │
       ├─→ Click to expand: AI Summary (free)
       │
       ├─→ Button: "Get Full Report - $25"
       │   └─→ Triggers Detailed Verification Flow
       │
       └─→ Button: "Subscribe to Indemnity - $75/mo"
           └─→ Links to subscription signup
```

### Detailed Verification Flow (Tier 2)

```
Buyer clicks "Get Full Report"
│
├─→ Check if buyer has Indemnity subscription
│   ├─→ If yes: Check report quota (50/year)
│   │   ├─→ If under quota: Generate report (free)
│   │   └─→ If over quota: Prompt to purchase ($25)
│   └─→ If no: Prompt to pay $25 or subscribe
│
├─→ Run Detailed Verification
│   │
│   ├─→ US Entity:
│   │   ├─→ Call Middesk API (POST /v1/businesses)
│   │   │   └─→ Returns: legal_name, entity_type, formation_date,
│   │   │       registered_address, status, registered_agent
│   │   └─→ Call Didit AML (detailed mode)
│   │       └─→ include_adverse_media: true
│   │       └─→ Returns: Full hit details, adverse media, linked entities
│   │
│   └─→ International Entity:
│       └─→ Call Didit AML (detailed mode)
│           └─→ Same as above
│
├─→ Generate AI Executive Summary
│   ├─→ Claude API with full context (Middesk + AML data)
│   ├─→ Prompt template:
│   │   """
│   │   You are analyzing supplier verification data for a business
│   │   buyer. Provide an executive summary including:
│   │
│   │   1. Business Overview (from Middesk)
│   │   2. Risk Assessment (from AML screening)
│   │   3. Key Findings (hits, flags, concerns)
│   │   4. Risk Level (Low/Medium/High)
│   │   5. Recommendation (Approve/Review/Decline)
│   │   6. Action Items for buyer
│   │
│   │   Data:
│   │   {middesk_data}
│   │   {aml_data}
│   │   """
│   └─→ Returns: Structured summary object
│
├─→ Generate PDF Report
│   ├─→ Professional template with:
│   │   ├─→ Cover page (supplier name, date, report ID)
│   │   ├─→ Executive Summary (AI-generated)
│   │   ├─→ Business Verification Section (Middesk data)
│   │   ├─→ AML/PEP Screening Section (detailed hits)
│   │   ├─→ Risk Score Breakdown
│   │   ├─→ Recommendations
│   │   └─→ Appendix (raw data, sources, methodology)
│   └─→ Store in documents table with category: 'verification_report'
│
├─→ Save to verification_reports table
│   ├─→ report_id, supplier_id, buyer_id, tier: 'detailed'
│   ├─→ middesk_data, aml_data, ai_summary
│   ├─→ pdf_url, generated_at, valid_until
│   └─→ If indemnity subscriber: decrement report quota
│
└─→ Return PDF URL + summary to buyer
```

### Indemnity Subscription Flow

```
Buyer clicks "Subscribe to Indemnity Protection"
│
├─→ Show subscription signup page
│   ├─→ Plan details and pricing
│   ├─→ Coverage limits
│   ├─→ Terms & conditions
│   └─→ Payment form (Stripe integration)
│
├─→ Buyer selects plan
│   ├─→ Monthly: $75/month
│   └─→ Annual: $750/year (save $150)
│
├─→ Payment processing (Stripe)
│   └─→ Create subscription in Stripe
│
├─→ Create indemnity_subscriptions record
│   ├─→ buyer_id (partner_id or client_id)
│   ├─→ plan: 'monthly' or 'annual'
│   ├─→ status: 'active'
│   ├─→ coverage_limit_per_incident: $X
│   ├─→ coverage_limit_annual: $Y
│   ├─→ deductible: $Z
│   ├─→ reports_quota: 50
│   ├─→ reports_used: 0
│   ├─→ started_at, renews_at
│   └─→ stripe_subscription_id
│
├─→ Enable ongoing monitoring
│   └─→ Create workflow: quarterly AML rescreening
│
└─→ Send confirmation email
    └─→ Welcome to Indemnity Protection
    └─→ Coverage summary
    └─→ Next steps
```

### Claims Flow

```
Buyer experiences supplier fraud
│
├─→ Files claim via Client Portal
│   ├─→ Claim form:
│   │   ├─→ Supplier name
│   │   ├─→ Fraud type (payment, identity, invoice)
│   │   ├─→ Loss amount
│   │   ├─→ Date of incident
│   │   ├─→ Description
│   │   └─→ Supporting documents (invoices, transfers, communications)
│   └─→ Creates indemnity_claims record
│
├─→ Centime reviews claim
│   ├─→ Verify subscription was active at time of incident
│   ├─→ Verify fraud type is covered
│   ├─→ Check coverage limits (per-incident, annual aggregate)
│   ├─→ Review verification history (was supplier screened?)
│   ├─→ Investigate fraud (request additional docs if needed)
│   └─→ Make decision: Approve / Deny / Request More Info
│
├─→ If approved:
│   ├─→ Calculate payout (loss amount - deductible, capped at coverage limit)
│   ├─→ Process payment to buyer
│   ├─→ Update claim status: 'paid'
│   ├─→ Update subscription: deduct from annual aggregate
│   └─→ Pursue recovery from fraudster (subrogation)
│
└─→ If denied:
    └─→ Send explanation + appeals process
```

---

## Data Model

### New Tables

#### `verification_reports`

Stores detailed verification reports (Tier 2 & 3).

```sql
CREATE TABLE verification_reports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- References
  supplier_id UUID NOT NULL REFERENCES imported_suppliers(id),
  buyer_id UUID NOT NULL, -- partner_id or client_id
  buyer_type TEXT NOT NULL, -- 'partner' or 'client'

  -- Report metadata
  report_id TEXT NOT NULL UNIQUE, -- Human-readable: VR-2026-001234
  tier TEXT NOT NULL, -- 'detailed' (Tier 2/3)
  generated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  valid_until TIMESTAMPTZ NOT NULL, -- generated_at + 90 days

  -- Verification data
  tin_validation JSONB, -- { valid: true, ein: '12-3456789', source: 'w9' }
  middesk_data JSONB, -- Full Middesk response (US entities only)
  aml_data JSONB, -- Full Didit AML response with hits
  ai_summary JSONB, -- { executive_summary, risk_level, recommendation, action_items }

  -- Report files
  pdf_url TEXT, -- S3 URL or similar
  pdf_generated_at TIMESTAMPTZ,

  -- Payment tracking
  purchased_via TEXT, -- 'one_time' or 'indemnity_subscription'
  indemnity_subscription_id UUID REFERENCES indemnity_subscriptions(id),
  amount_charged INTEGER, -- cents (0 if via subscription)
  stripe_charge_id TEXT,

  -- Audit
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_verification_reports_supplier ON verification_reports(supplier_id);
CREATE INDEX idx_verification_reports_buyer ON verification_reports(buyer_id, buyer_type);
CREATE INDEX idx_verification_reports_subscription ON verification_reports(indemnity_subscription_id);
```

#### `indemnity_subscriptions`

Stores indemnity subscription details.

```sql
CREATE TABLE indemnity_subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- Subscriber
  buyer_id UUID NOT NULL, -- partner_id or client_id
  buyer_type TEXT NOT NULL, -- 'partner' or 'client'

  -- Plan details
  plan TEXT NOT NULL, -- 'monthly' or 'annual'
  status TEXT NOT NULL, -- 'active', 'cancelled', 'expired', 'suspended'

  -- Coverage limits (in cents)
  coverage_limit_per_incident INTEGER NOT NULL, -- e.g., 2500000 = $25,000
  coverage_limit_annual INTEGER NOT NULL, -- e.g., 25000000 = $250,000
  coverage_used_current_period INTEGER NOT NULL DEFAULT 0, -- cents
  deductible INTEGER NOT NULL, -- e.g., 100000 = $1,000

  -- Reports quota
  reports_quota INTEGER NOT NULL DEFAULT 50, -- per year
  reports_used INTEGER NOT NULL DEFAULT 0,
  reports_reset_at TIMESTAMPTZ, -- annual reset date

  -- Subscription lifecycle
  started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  renews_at TIMESTAMPTZ NOT NULL,
  cancelled_at TIMESTAMPTZ,
  cancellation_reason TEXT,

  -- Payment integration
  stripe_subscription_id TEXT UNIQUE,
  stripe_customer_id TEXT,

  -- Audit
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_indemnity_subscriptions_buyer ON indemnity_subscriptions(buyer_id, buyer_type);
CREATE INDEX idx_indemnity_subscriptions_status ON indemnity_subscriptions(status);
CREATE INDEX idx_indemnity_subscriptions_stripe ON indemnity_subscriptions(stripe_subscription_id);
```

#### `indemnity_claims`

Stores fraud claims filed by indemnity subscribers.

```sql
CREATE TABLE indemnity_claims (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- References
  claim_id TEXT NOT NULL UNIQUE, -- Human-readable: CL-2026-001234
  indemnity_subscription_id UUID NOT NULL REFERENCES indemnity_subscriptions(id),
  supplier_id UUID NOT NULL REFERENCES imported_suppliers(id),
  buyer_id UUID NOT NULL,
  buyer_type TEXT NOT NULL,

  -- Claim details
  fraud_type TEXT NOT NULL, -- 'payment_fraud', 'identity_fraud', 'invoice_fraud'
  incident_date DATE NOT NULL,
  loss_amount INTEGER NOT NULL, -- cents
  description TEXT NOT NULL,

  -- Supporting documents
  supporting_documents JSONB, -- [{ document_id, type, url }]

  -- Review & decision
  status TEXT NOT NULL DEFAULT 'submitted',
  -- 'submitted', 'under_review', 'approved', 'denied', 'paid', 'closed'

  reviewed_by UUID, -- Centime admin user
  reviewed_at TIMESTAMPTZ,
  decision_reason TEXT,

  -- Payout
  payout_amount INTEGER, -- loss_amount - deductible, capped at coverage limit
  payout_date DATE,
  payout_method TEXT, -- 'ach', 'wire', 'check'
  payout_reference TEXT, -- transaction ID

  -- Subrogation (recovery from fraudster)
  subrogation_pursued BOOLEAN DEFAULT FALSE,
  subrogation_recovered INTEGER DEFAULT 0, -- cents

  -- Audit
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_indemnity_claims_subscription ON indemnity_claims(indemnity_subscription_id);
CREATE INDEX idx_indemnity_claims_supplier ON indemnity_claims(supplier_id);
CREATE INDEX idx_indemnity_claims_status ON indemnity_claims(status);
```

#### `monitoring_alerts`

Tracks ongoing monitoring alerts for indemnity subscribers.

```sql
CREATE TABLE monitoring_alerts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  -- References
  indemnity_subscription_id UUID NOT NULL REFERENCES indemnity_subscriptions(id),
  supplier_id UUID NOT NULL REFERENCES imported_suppliers(id),

  -- Alert details
  alert_type TEXT NOT NULL, -- 'aml_status_change', 'new_watchlist_hit', 'adverse_media', 'sanctions_added'
  severity TEXT NOT NULL, -- 'low', 'medium', 'high', 'critical'

  -- Alert content
  title TEXT NOT NULL,
  description TEXT NOT NULL,
  previous_state JSONB, -- Before change
  current_state JSONB, -- After change

  -- Status
  status TEXT NOT NULL DEFAULT 'unread', -- 'unread', 'read', 'acknowledged', 'dismissed'
  acknowledged_by UUID,
  acknowledged_at TIMESTAMPTZ,

  -- Audit
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_monitoring_alerts_subscription ON monitoring_alerts(indemnity_subscription_id);
CREATE INDEX idx_monitoring_alerts_supplier ON monitoring_alerts(supplier_id);
CREATE INDEX idx_monitoring_alerts_status ON monitoring_alerts(status);
```

### Updated Tables

#### `identity_verification`

Add fields for basic AML screening (Tier 1).

```sql
ALTER TABLE identity_verification ADD COLUMN IF NOT EXISTS tin_validation JSONB;
-- { valid: true/false, ein: '12-3456789', source: 'w9'/'w8ben_e' }

ALTER TABLE identity_verification ADD COLUMN IF NOT EXISTS aml_screening_basic JSONB;
-- { request_id, status, risk_score, total_hits, screened_at }

ALTER TABLE identity_verification ADD COLUMN IF NOT EXISTS ai_summary TEXT;
-- Free AI-generated summary if issues found
```

---

## API Endpoints

### Verification APIs

#### `POST /api/v1/verify/basic`
Run basic verification (Tier 1 - TIN + AML).

**Request:**
```json
{
  "supplier_id": "uuid",
  "entity_type": "us_business" | "international_business" | "us_consumer" | "international_consumer"
}
```

**Response:**
```json
{
  "success": true,
  "verification_id": "uuid",
  "status": "clean" | "issues_found",
  "tin_validation": {
    "valid": true,
    "ein": "12-3456789"
  },
  "aml_screening": {
    "status": "Approved" | "In Review" | "Declined",
    "risk_score": 57,
    "total_hits": 3
  },
  "ai_summary": "This supplier has 3 watchlist matches...", // Only if issues
  "trust_score_impact": {
    "previous": 45,
    "new": 73,
    "identity_points": 28
  }
}
```

#### `POST /api/v1/verify/detailed`
Generate detailed verification report (Tier 2/3).

**Request:**
```json
{
  "supplier_id": "uuid",
  "buyer_id": "uuid",
  "buyer_type": "partner" | "client"
}
```

**Response:**
```json
{
  "success": true,
  "report": {
    "report_id": "VR-2026-001234",
    "tier": "detailed",
    "generated_at": "2026-02-22T14:35:08Z",
    "valid_until": "2026-05-23T14:35:08Z",
    "pdf_url": "https://s3.../reports/VR-2026-001234.pdf",
    "summary": {
      "executive_summary": "Based on comprehensive screening...",
      "risk_level": "Medium",
      "recommendation": "Approve with monitoring",
      "action_items": [
        "Review historical Iran business context",
        "Establish payment verification procedures"
      ]
    }
  },
  "charged": true,
  "amount": 2500, // cents ($25)
  "via_subscription": false
}
```

#### `GET /api/v1/verify/reports/:reportId`
Retrieve existing verification report.

**Response:**
```json
{
  "report": {
    "report_id": "VR-2026-001234",
    "supplier_id": "uuid",
    "generated_at": "2026-02-22T14:35:08Z",
    "valid_until": "2026-05-23T14:35:08Z",
    "pdf_url": "https://...",
    "middesk_data": {...},
    "aml_data": {...},
    "ai_summary": {...}
  }
}
```

### Indemnity Subscription APIs

#### `POST /api/v1/indemnity/subscribe`
Create indemnity subscription.

**Request:**
```json
{
  "buyer_id": "uuid",
  "buyer_type": "partner" | "client",
  "plan": "monthly" | "annual",
  "payment_method_id": "pm_..." // Stripe payment method
}
```

**Response:**
```json
{
  "success": true,
  "subscription": {
    "id": "uuid",
    "status": "active",
    "plan": "monthly",
    "coverage_limit_per_incident": 2500000,
    "coverage_limit_annual": 25000000,
    "deductible": 100000,
    "reports_quota": 50,
    "reports_used": 0,
    "started_at": "2026-02-22T14:35:08Z",
    "renews_at": "2026-03-22T14:35:08Z"
  },
  "stripe_subscription_id": "sub_..."
}
```

#### `GET /api/v1/indemnity/subscription`
Get current indemnity subscription for buyer.

**Response:**
```json
{
  "subscription": {
    "id": "uuid",
    "status": "active",
    "plan": "annual",
    "coverage_limit_per_incident": 2500000,
    "coverage_limit_annual": 25000000,
    "coverage_used_current_period": 0,
    "deductible": 100000,
    "reports_quota": 50,
    "reports_used": 12,
    "reports_remaining": 38,
    "started_at": "2026-01-15T00:00:00Z",
    "renews_at": "2027-01-15T00:00:00Z"
  }
}
```

#### `POST /api/v1/indemnity/claims`
File fraud claim.

**Request:**
```json
{
  "supplier_id": "uuid",
  "fraud_type": "payment_fraud" | "identity_fraud" | "invoice_fraud",
  "incident_date": "2026-02-15",
  "loss_amount": 1500000, // cents ($15,000)
  "description": "Supplier provided fraudulent bank account information...",
  "supporting_documents": [
    { "document_id": "uuid", "type": "invoice" },
    { "document_id": "uuid", "type": "bank_transfer" }
  ]
}
```

**Response:**
```json
{
  "success": true,
  "claim": {
    "claim_id": "CL-2026-001234",
    "status": "submitted",
    "loss_amount": 1500000,
    "potential_payout": 1400000, // loss - deductible
    "created_at": "2026-02-22T14:35:08Z",
    "next_steps": "Our fraud team will review your claim within 2 business days..."
  }
}
```

#### `GET /api/v1/indemnity/claims`
List buyer's claims.

**Response:**
```json
{
  "claims": [
    {
      "claim_id": "CL-2026-001234",
      "supplier_name": "Acme Corp",
      "fraud_type": "payment_fraud",
      "loss_amount": 1500000,
      "status": "under_review",
      "filed_at": "2026-02-22T14:35:08Z"
    }
  ]
}
```

#### `GET /api/v1/indemnity/monitoring/alerts`
Get monitoring alerts for subscriber.

**Response:**
```json
{
  "alerts": [
    {
      "id": "uuid",
      "supplier_name": "Acme Corp",
      "alert_type": "new_watchlist_hit",
      "severity": "medium",
      "title": "New AML Watchlist Match",
      "description": "Supplier appeared on OFAC sanctions list",
      "status": "unread",
      "created_at": "2026-02-22T14:35:08Z"
    }
  ]
}
```

---

## Trust Score Integration

### Updated Business Trust Score Calculation

```javascript
// Business account scoring (from trustScore.js)
function calculateBusinessTrustScore(prefs, verification, docs, kyb, accountAge, clientCount) {
  const components = {
    verification: { earned: 0, max: 35, method: null },
    identity: { earned: 0, max: 35, method: null },
    compliance: { earned: 0, max: 15, details: [] },
    stability: { earned: 0, max: 10, details: null },
    history: { earned: 0, max: 5, details: null },
  }

  // Verification points (bank verification) - unchanged
  // ... existing logic ...

  // Identity points (business) — UPDATED
  const hasDetailedReport = verification?.detailed_report_id
  const hasMiddesk = verification?.middesk_data
  const hasAMLBasic = verification?.aml_screening_basic
  const hasAMLDetailed = verification?.aml_data

  if (hasDetailedReport && hasMiddesk && hasAMLDetailed) {
    // Tier 3: Full detailed report (Middesk + AML)
    components.identity.earned = 35
    components.identity.method = 'middesk_aml_detailed'
  } else if (hasMiddesk && hasAMLBasic) {
    // US Business: Middesk + Basic AML
    const basePoints = 25 // Middesk base
    const amlBonus = calculateAMLBonus(verification.aml_screening_basic)
    components.identity.earned = basePoints + amlBonus // 25-35 points
    components.identity.method = 'middesk_aml_basic'
  } else if (hasAMLDetailed) {
    // International Business: Detailed AML only
    components.identity.earned = calculateAMLPoints(verification.aml_data, 'detailed')
    components.identity.method = 'aml_detailed'
  } else if (hasAMLBasic) {
    // Basic AML only
    components.identity.earned = calculateAMLPoints(verification.aml_screening_basic, 'basic')
    components.identity.method = 'aml_basic'
  } else if (kyb?.legalName) {
    // Legacy: Middesk only (pre-AML)
    components.identity.earned = 25
    components.identity.method = 'middesk'
  }

  // ... rest of calculation ...
}

function calculateAMLBonus(amlData) {
  // For US businesses with Middesk + Basic AML
  // Returns: 0-10 bonus points
  const riskScore = amlData.risk_score || 0
  const status = amlData.status

  if (status === 'Declined') return 0
  if (status === 'In Review') return 3

  // Status is 'Approved'
  if (riskScore <= 40) return 10      // Excellent
  if (riskScore <= 60) return 8       // Good
  if (riskScore <= 80) return 5       // Fair
  return 3                             // Approved but higher risk
}

function calculateAMLPoints(amlData, tier) {
  // For international businesses (AML only) or detailed reports
  // Returns: 10-35 points based on risk score and status
  const riskScore = amlData.risk_score || 0
  const status = amlData.status

  if (status === 'Declined') return 10
  if (status === 'In Review') return 20

  // Status is 'Approved' - award points based on risk score
  if (tier === 'detailed') {
    // Detailed reports get better scoring
    if (riskScore <= 20) return 35      // Excellent
    if (riskScore <= 40) return 32      // Very Good
    if (riskScore <= 60) return 28      // Good
    if (riskScore <= 80) return 22      // Fair
    return 15                            // Poor but approved
  } else {
    // Basic screening gets conservative scoring
    if (riskScore <= 40) return 28      // Very Good
    if (riskScore <= 60) return 25      // Good
    if (riskScore <= 80) return 20      // Fair
    return 15                            // Poor but approved
  }
}
```

### Scoring Examples

**US Business:**
- Middesk + Clean AML (risk ≤ 40): 25 + 10 = **35 points** ⭐
- Middesk + Good AML (risk ≤ 60): 25 + 8 = **33 points**
- Middesk + Fair AML (risk ≤ 80): 25 + 5 = **30 points**
- Middesk only (no AML): **25 points**

**International Business (AML only):**
- Detailed report, risk ≤ 20: **35 points** ⭐
- Detailed report, risk ≤ 60: **28 points** (Melia example)
- Basic screening, risk ≤ 40: **28 points**
- Basic screening, risk ≤ 60: **25 points**

---

## UI/UX Design

### Supplier Portal (Profile Page)

#### Before Verification
```
┌─────────────────────────────────────────────────────┐
│ Identity Verification                               │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Verify your business identity to increase your     │
│ trust score and qualify for better payment terms.  │
│                                                     │
│ [ Verify Identity ]                                 │
│                                                     │
└─────────────────────────────────────────────────────┘
```

#### After Verification - Clean Result
```
┌─────────────────────────────────────────────────────┐
│ Identity Verification               ✓ Verified     │
├─────────────────────────────────────────────────────┤
│                                                     │
│ ✓ Tax ID Validated                                 │
│ ✓ AML/PEP Screening Passed                        │
│ ✓ No risk flags found                              │
│                                                     │
│ Verified on Feb 22, 2026                           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

#### After Verification - Issues Found
```
┌─────────────────────────────────────────────────────┐
│ Identity Verification          ⚠️ Issues Found     │
├─────────────────────────────────────────────────────┤
│                                                     │
│ ✓ Tax ID Validated                                 │
│ ⚠️ AML/PEP Screening: 3 matches found              │
│                                                     │
│ Your buyer may review these findings. This does    │
│ not necessarily indicate a problem - many          │
│ legitimate businesses have informational matches.  │
│                                                     │
│ Verified on Feb 22, 2026                           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Client Portal - Supplier Detail Panel

#### Clean Verification
```
┌─────────────────────────────────────────────────────┐
│ Melia Hotels International                          │
│ ✓ Verified                                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Identity Verification                               │
│ ✓ Tax ID Valid • ✓ AML Screening Passed           │
│ Verified Feb 22, 2026                              │
│                                                     │
│ Trust Score: 73/100                                 │
│ [ View Trust Score Breakdown ]                      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

#### Issues Found - Expanded View
```
┌─────────────────────────────────────────────────────┐
│ Melia Hotels International                          │
│ ⚠️ Issues Found                                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Identity Verification                               │
│ ✓ Tax ID Valid • ⚠️ 3 AML matches found            │
│                                                     │
│ ┌─────────────────────────────────────────────────┐ │
│ │ 🤖 AI Risk Summary (Free)                       │ │
│ ├─────────────────────────────────────────────────┤ │
│ │                                                 │ │
│ │ This supplier has 3 watchlist matches related  │ │
│ │ to historical business dealings with Iran       │ │
│ │ (2016-2019). The hotel development project was │ │
│ │ cancelled before sanctions took effect.         │ │
│ │                                                 │ │
│ │ Match Details:                                  │ │
│ │ • Iran UANI Business Registry - SIE             │ │
│ │ • No active sanctions matches                   │ │
│ │ • No PEP matches                                │ │
│ │                                                 │ │
│ │ Risk Level: Medium                              │ │
│ │ Risk Score: 57/100                              │ │
│ │                                                 │ │
│ │ Recommendation: Review business relationship   │ │
│ │ context before proceeding with high-value       │ │
│ │ transactions.                                   │ │
│ │                                                 │ │
│ └─────────────────────────────────────────────────┘ │
│                                                     │
│ [ Get Full Detailed Report - $25 ]                  │
│                                                     │
│ Or subscribe to Indemnity Protection:               │
│ • 50 detailed reports/year included                 │
│ • Fraud coverage up to $25K per incident            │
│ • Ongoing monitoring with alerts                    │
│ • Only $75/month                                    │
│                                                     │
│ [ Learn More About Indemnity ] [ Subscribe Now ]    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Detailed Report View (After Purchase)
```
┌─────────────────────────────────────────────────────┐
│ Verification Report: VR-2026-001234                 │
│ Melia Hotels International                          │
│ Generated Feb 22, 2026 • Valid until May 23, 2026  │
├─────────────────────────────────────────────────────┤
│                                                     │
│ 📄 Executive Summary                                │
│                                                     │
│ Based on comprehensive screening of Melia Hotels    │
│ International through Middesk business verification │
│ and global AML/PEP databases, we assess this       │
│ supplier as MEDIUM RISK.                            │
│                                                     │
│ Key Findings:                                       │
│ • Business is legally registered and active         │
│ • No sanctions or PEP matches identified            │
│ • Historical business activity with Iran flagged    │
│   (2016-2019, pre-sanctions)                        │
│ • Project was cancelled; no ongoing Iran ties       │
│                                                     │
│ Risk Level: MEDIUM                                  │
│ Overall Risk Score: 57/100                          │
│                                                     │
│ Recommendation: APPROVE WITH MONITORING             │
│                                                     │
│ Action Items:                                       │
│ ✓ Confirm no active business with sanctioned       │
│   entities                                          │
│ ✓ Establish payment verification procedures        │
│ ✓ Monitor for sanctions list changes               │
│                                                     │
│ ─────────────────────────────────────────────────── │
│                                                     │
│ 🏢 Business Verification (Middesk)                  │
│                                                     │
│ Legal Name: Meliá Hotels International, S.A.       │
│ Entity Type: Corporation                            │
│ Jurisdiction: Spain                                 │
│ Status: Active                                      │
│ Formation Date: January 15, 1956                   │
│ Registered Address: Calle Gremio De Toneleros 24,  │
│                     Palma, Illes Balears 07009, ES  │
│                                                     │
│ ─────────────────────────────────────────────────── │
│                                                     │
│ 🔍 AML/PEP Screening (Didit)                        │
│                                                     │
│ Status: Approved                                    │
│ Risk Score: 57/100                                  │
│ Total Matches: 3                                    │
│                                                     │
│ Match #1: Meliá Hotels International               │
│ Dataset: Iran UANI Business Registry (SIE)         │
│ Match Score: 97%                                    │
│ Risk Score: 57                                      │
│ Status: Unreviewed                                  │
│                                                     │
│ Details: Historical hotel development project in    │
│ Iran (2016-2019). Project cancelled before U.S.    │
│ sanctions took effect. No ongoing business ties.    │
│                                                     │
│ Sources:                                            │
│ • Wall Street Journal, May 2016                     │
│ • Vozpopuli, December 2019                          │
│                                                     │
│ [Show full details] [View source links]            │
│                                                     │
│ Match #2: Senator Hotels International Inc.        │
│ Dataset: ICIJ Offshore Leaks                        │
│ Match Score: 82%                                    │
│ Review Status: False Positive                       │
│                                                     │
│ Match #3: Sonesta International Hotels Corp.       │
│ Dataset: ICIJ Offshore Leaks                        │
│ Match Score: 85%                                    │
│ Review Status: False Positive                       │
│                                                     │
│ ─────────────────────────────────────────────────── │
│                                                     │
│ [ Download Full PDF Report ]                        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Indemnity Subscription Dashboard
```
┌─────────────────────────────────────────────────────┐
│ Indemnity Protection                                │
│ Active Subscription                                 │
├─────────────────────────────────────────────────────┤
│                                                     │
│ Plan: Annual • Renews Jan 15, 2027                 │
│ Coverage Status: Active                             │
│                                                     │
│ ┌─────────────────────────────────────────────────┐ │
│ │ Coverage Limits                                 │ │
│ ├─────────────────────────────────────────────────┤ │
│ │ Per Incident: $25,000                           │ │
│ │ Annual Aggregate: $250,000                      │ │
│ │ Used This Period: $0                            │ │
│ │ Remaining: $250,000                             │ │
│ │ Deductible: $1,000                              │ │
│ └─────────────────────────────────────────────────┘ │
│                                                     │
│ ┌─────────────────────────────────────────────────┐ │
│ │ Verification Reports                            │ │
│ ├─────────────────────────────────────────────────┤ │
│ │ Used: 12 of 50                                  │ │
│ │ Remaining: 38 reports                           │ │
│ │ Resets: Jan 15, 2027                            │ │
│ │                                                 │ │
│ │ ████████░░░░░░░░░░░░░  24%                      │ │
│ └─────────────────────────────────────────────────┘ │
│                                                     │
│ ┌─────────────────────────────────────────────────┐ │
│ │ Monitoring Alerts                      [ 3 New ] │ │
│ ├─────────────────────────────────────────────────┤ │
│ │ ⚠️ Medium • Acme Corp                           │ │
│ │ New AML watchlist match                         │ │
│ │ 2 hours ago                     [ View Details ] │ │
│ │                                                 │ │
│ │ 🟢 Low • Widget Inc                             │ │
│ │ Quarterly screening complete                    │ │
│ │ 1 day ago                      [ View Report ]  │ │
│ │                                                 │ │
│ │ [ View All Alerts (3) ]                         │ │
│ └─────────────────────────────────────────────────┘ │
│                                                     │
│ ┌─────────────────────────────────────────────────┐ │
│ │ Recent Reports                                  │ │
│ ├─────────────────────────────────────────────────┤ │
│ │ VR-2026-001234 • Melia Hotels • Feb 22, 2026   │ │
│ │ VR-2026-001233 • Acme Corp • Feb 20, 2026      │ │
│ │ VR-2026-001232 • Widget Inc • Feb 18, 2026     │ │
│ │                                                 │ │
│ │ [ View All Reports ]                            │ │
│ └─────────────────────────────────────────────────┘ │
│                                                     │
│ [ File a Fraud Claim ]                              │
│ [ Manage Subscription ]                             │
│ [ Contact Support ]                                 │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## Implementation Phases

### Phase 1: Basic Verification (Tier 1) - MVP
**Goal:** Get basic TIN + AML screening working for all entity types.

**Tasks:**
1. ✅ Create Didit AML endpoint (`/api/didit/aml-screening`) - DONE
2. ✅ Test with Melia Hotels - DONE
3. Update `identity_verification` table schema
4. Build basic verification flow in IdentityPanel
5. Integrate with trust score calculation
6. Add AI summary generation (Claude API) for flagged results
7. Update UI to show verification status badges
8. Test all four entity types (us_business, us_consumer, international_business, international_consumer)

**Success Criteria:**
- All suppliers can run basic verification
- Clean results show green badge
- Issues show yellow badge with free AI summary
- Trust score reflects verification status

**Timeline:** 1-2 weeks

---

### Phase 2: Detailed Reports (Tier 2) - Monetization Proof
**Goal:** Build paid detailed reports to validate buyer demand.

**Tasks:**
1. Create `verification_reports` table
2. Build Middesk integration for US entities
3. Enhance Didit AML to include detailed data (adverse media, etc.)
4. Build AI executive summary generation
5. Create PDF report template
6. Implement report generation endpoint (`POST /api/v1/verify/detailed`)
7. Add Stripe payment integration for one-time purchases
8. Build report viewer UI in Client Portal
9. Add "Get Full Report - $25" CTA on flagged suppliers

**Success Criteria:**
- Buyers can purchase detailed reports
- Reports include Middesk + AML + AI summary
- PDF generation works
- Payment flow functional

**Timeline:** 2-3 weeks

**Validation Question:** Do buyers actually purchase reports? (Hypothesis: No)

---

### Phase 3: Indemnity Subscription (Tier 3) - Core Product
**Goal:** Launch indemnity subscription as primary revenue driver.

**Tasks:**
1. Create `indemnity_subscriptions` table
2. Design coverage limits and pricing (requires actuarial input)
3. Build subscription signup flow
4. Integrate Stripe subscriptions
5. Build indemnity dashboard for subscribers
6. Implement report quota tracking (50/year)
7. Update verification flow to check subscription status
8. Build monitoring system (quarterly AML rescreening)
9. Create `monitoring_alerts` table and alert generation
10. Design and build indemnity marketing page

**Success Criteria:**
- Buyers can subscribe to indemnity
- Quota tracking works (reports included in subscription)
- Monitoring runs automatically
- Dashboard shows coverage and usage

**Timeline:** 3-4 weeks

---

### Phase 4: Claims Processing (Tier 3 - Full Product)
**Goal:** Enable fraud claims and payouts.

**Tasks:**
1. Create `indemnity_claims` table
2. Build claims filing flow in Client Portal
3. Build claims review workflow for Centime admin
4. Define claim approval/denial criteria
5. Implement payout processing
6. Build claims dashboard for buyers
7. Build claims admin panel for Centime
8. Create claims notification system
9. Legal: Draft indemnity terms & conditions
10. Legal: Review coverage exclusions and limits

**Success Criteria:**
- Buyers can file claims
- Centime can review and approve/deny
- Payouts process successfully
- Legal terms are sound

**Timeline:** 4-5 weeks

**Dependencies:** Legal review, actuarial analysis for reserves

---

### Phase 5: Ongoing Monitoring & Alerts
**Goal:** Deliver ongoing value to indemnity subscribers.

**Tasks:**
1. Build quarterly AML rescreening workflow (Inngest)
2. Implement change detection (compare current vs. previous screening)
3. Build alert generation logic
4. Create alert notification system (email + in-app)
5. Build alerts dashboard
6. Add alert acknowledgment/dismissal
7. Implement severity-based routing (critical alerts → immediate notification)

**Success Criteria:**
- Subscribers get quarterly rescreening
- Alerts generated when status changes
- Notifications sent reliably
- Dashboard shows alert history

**Timeline:** 2-3 weeks

---

## Open Questions & Decisions Needed

### Business Model

1. **Coverage Limits**
   - Per-incident: $10K? $25K? $50K? $100K?
   - Annual aggregate: $100K? $250K? $500K? $1M?
   - Deductible: $0? $500? $1K? $2.5K?
   - **Decision maker:** BC + Finance team
   - **Input needed:** Actuarial analysis, market research

2. **Pricing**
   - Detailed report: $25? $50? $75?
   - Indemnity monthly: $75? $99? $149?
   - Indemnity annual discount: How many months free?
   - Enterprise pricing tiers?
   - **Decision maker:** BC + Finance team
   - **Input needed:** Competitive analysis, cost structure

3. **Report Quota**
   - 50 reports/year? 25? 100? Unlimited?
   - Carryover unused reports?
   - **Decision maker:** BC
   - **Input needed:** Customer behavior data (Phase 2)

4. **What's Covered by Indemnity?**
   - Payment fraud: Yes (fake bank accounts, wire fraud)
   - Identity fraud: Yes (impersonation, fake businesses)
   - Invoice fraud: Yes? (duplicate payments, altered invoices)
   - Product quality: No
   - Delivery issues: No
   - Contract disputes: No
   - Willful negligence: No
   - **Decision maker:** BC + Legal
   - **Input needed:** Legal review, insurance expertise

5. **Underwriting Requirements**
   - Do we screen buyers before offering indemnity?
   - Eligibility criteria? (transaction volume, industry, credit check?)
   - Can we deny coverage to high-risk buyers?
   - **Decision maker:** BC + Risk team
   - **Input needed:** Risk assessment framework

### Technical

6. **AI Summary Prompt Engineering**
   - What context to provide Claude?
   - How much detail in free summary vs. paid report?
   - Tone and framing (alarming vs. reassuring)?
   - **Decision maker:** BC + Product team
   - **Input needed:** User testing, A/B testing

7. **Monitoring Frequency**
   - Quarterly rescreening? Monthly? On-demand only?
   - Cost vs. value trade-off
   - **Decision maker:** BC
   - **Input needed:** Cost analysis (Didit API fees)

8. **PDF Report Design**
   - Custom template? Use a service (DocRaptor, PDFKit)?
   - Branding and layout
   - **Decision maker:** BC + Design team

9. **International Entities - Middesk Alternative?**
   - Middesk is US-only
   - Do we need international business registry checks?
   - Or is AML-only sufficient for international?
   - **Decision maker:** BC
   - **Input needed:** Market research, customer feedback

10. **Consumer AML Screening**
    - Should US consumers also get AML screening?
    - Melissa + AML layered? Or skip for consumers?
    - **Decision maker:** BC
    - **Input needed:** Risk assessment, cost-benefit analysis

### Legal & Compliance

11. **Indemnity Terms & Conditions**
    - Exclusions and limitations
    - Claims process and requirements
    - Dispute resolution
    - Cancellation policy
    - **Decision maker:** Legal team
    - **Input needed:** Insurance law expertise

12. **Insurance vs. Indemnity**
    - Is this legally "insurance"? (may require licensing)
    - Or "indemnity/warranty" (different regulations)?
    - State-by-state variations
    - **Decision maker:** Legal team
    - **Input needed:** Insurance regulatory counsel

13. **Data Privacy - Sharing AML Results**
    - What can buyers legally see?
    - Do we need supplier consent to share AML data with buyers?
    - GDPR/privacy implications for adverse media
    - **Decision maker:** Legal + Privacy team
    - **Input needed:** Privacy counsel review

14. **Claims Reserve Requirements**
    - Do we need to maintain cash reserves for claims?
    - Accounting treatment
    - **Decision maker:** Finance + Legal
    - **Input needed:** Actuarial analysis, CFO input

### Product & UX

15. **What Triggers Basic Verification?**
    - Auto-run on supplier signup?
    - Require supplier to click "Verify"?
    - Buyer triggers when viewing supplier?
    - **Decision maker:** BC + Product team
    - **Input needed:** User research, funnel analysis

16. **Supplier Visibility**
    - Do suppliers know when buyers run detailed reports on them?
    - Do suppliers see their own AML results?
    - Privacy vs. transparency trade-off
    - **Decision maker:** BC + Product team
    - **Input needed:** User research, legal review

17. **Free AI Summary - How Much Detail?**
    - If we give away too much, nobody buys the report
    - If we give too little, buyers don't trust us
    - Sweet spot? (current approach: match count, datasets, risk level, recommendation)
    - **Decision maker:** BC + Product team
    - **Input needed:** User testing, conversion analysis

18. **Indemnity Upsell Positioning**
    - Where to promote indemnity subscription?
    - How aggressive with upsell messaging?
    - Fear-based ("protect yourself") vs. value-based ("get reports included")?
    - **Decision maker:** BC + Marketing team
    - **Input needed:** A/B testing, customer interviews

---

## Success Metrics

### Phase 1 (Basic Verification)
- **Adoption:** % of suppliers who complete verification
- **Clean rate:** % of suppliers with no issues
- **Flag rate:** % of suppliers with AML matches
- **Trust score impact:** Average score increase after verification

### Phase 2 (Detailed Reports)
- **Purchase rate:** % of buyers who purchase detailed reports
- **Revenue per report:** Average selling price achieved
- **Report view duration:** Time spent reviewing reports
- **Customer feedback:** Quality ratings on reports

### Phase 3 (Indemnity Subscription)
- **Subscription rate:** % of buyers who subscribe
- **MRR:** Monthly recurring revenue from subscriptions
- **Churn rate:** % of subscribers who cancel
- **Report utilization:** Average reports used per subscriber
- **LTV:** Lifetime value of indemnity subscribers

### Phase 4 (Claims Processing)
- **Claims rate:** Claims filed per 1000 subscribers per year
- **Approval rate:** % of claims approved
- **Average payout:** Mean claim payout amount
- **Loss ratio:** Claims paid / premiums collected (target: <60%)
- **Time to resolution:** Days from claim filed to payout

### Phase 5 (Monitoring)
- **Alert generation rate:** Alerts per subscriber per quarter
- **False positive rate:** % of alerts that are false alarms
- **Alert action rate:** % of alerts that lead to action
- **Subscriber satisfaction:** NPS for monitoring service

---

## Competitive Analysis

### What Competitors Offer

**Most payment platforms:** Nothing. Zero verification.

**Stripe Atlas (for sellers):**
- Business formation verification
- No supplier-side verification for buyers

**Bill.com:**
- Basic business verification
- No AML/PEP screening
- No indemnity coverage

**Ramp, Brex (corporate cards):**
- Merchant verification for card acceptance
- Fraud protection on transactions
- Not supplier-focused

**Trade credit insurance (Euler Hermes, Atradius):**
- Covers non-payment risk (buyer doesn't pay)
- Does NOT cover supplier fraud (fake invoices, fake bank accounts)
- Expensive (~1-3% of invoice value)
- Focus on large B2B transactions ($100K+)

### Our Differentiation

✅ **Only platform offering supplier fraud indemnity**
✅ **Comprehensive verification (TIN + Middesk + AML/PEP)**
✅ **Transparent pricing (subscription vs. % of transaction)**
✅ **AI-powered risk summaries**
✅ **Ongoing monitoring included**
✅ **Accessible to SMBs (not just enterprises)**

**Market position:** "Stripe for Supplier Payments with Built-in Fraud Protection"

---

## Revenue Model Projections

### Assumptions (Conservative)

- **Total buyers on platform:** 1,000 (Year 1)
- **Suppliers per buyer (avg):** 20
- **Total suppliers:** 20,000

**Tier 1 (Basic Verification) - Free:**
- Adoption rate: 60% of suppliers
- Verifications: 12,000
- Cost: $1 per verification = $12,000 expense
- Revenue: $0

**Tier 2 (Detailed Reports) - $25:**
- Purchase rate: 2% of suppliers (conservative)
- Reports sold: 400
- Revenue: $10,000
- Cost: $4 per report = $1,600
- Gross profit: $8,400

**Tier 3 (Indemnity Subscriptions) - $75/mo:**
- Subscription rate: 10% of buyers (100 subscribers)
- MRR: $7,500
- ARR: $90,000
- Verification cost per subscriber: ~$150/year
- Total verification costs: $15,000
- Monitoring costs: $50/subscriber/year = $5,000
- Support overhead: $100/subscriber/year = $10,000
- **Total costs:** $30,000
- **Gross profit:** $60,000 (67% margin)

**Year 1 Total Revenue:** $100,000
**Year 1 Total Costs:** $43,600
**Year 1 Gross Profit:** $56,400

### Scaling (Year 3)

- **Total buyers:** 10,000
- **Indemnity subscribers:** 1,500 (15% penetration)
- **ARR from subscriptions:** $1,350,000
- **Claims payouts (assuming 2% loss ratio):** $27,000
- **Gross profit (after claims):** ~$900,000

**Not including:**
- Enterprise pricing (likely higher)
- Upsells (monitoring for non-subscribers, one-off reports)
- White-label opportunities (offer to other platforms)

---

## Risk Mitigation

### Actuarial Risks

**Risk:** Claims exceed premiums (negative loss ratio)

**Mitigation:**
1. Conservative coverage limits initially ($25K per incident)
2. Deductibles to reduce small claims
3. Exclusions for non-fraud issues
4. Underwriting for high-risk buyers
5. Reinsurance for catastrophic losses (if we scale)
6. Annual aggregate caps per subscriber
7. Monitor loss ratios monthly, adjust pricing quarterly

### Technical Risks

**Risk:** API dependencies (Didit, Middesk) have outages or rate limits

**Mitigation:**
1. Graceful degradation (show cached results if APIs down)
2. Queue verification requests during outages
3. SLA monitoring and alerts
4. Fallback providers if available
5. Cache results for 90 days (verification report validity)

**Risk:** AI summaries are inaccurate or misleading

**Mitigation:**
1. Human review of AI prompts before launch
2. Disclaimer: "AI-generated summary, not legal advice"
3. A/B test different prompt approaches
4. User feedback mechanism ("Was this summary helpful?")
5. Regular spot-checks of AI output quality

### Legal Risks

**Risk:** Indemnity product is classified as insurance, requires licensing

**Mitigation:**
1. Legal review BEFORE launch (Phase 3 blocker)
2. Structure as "warranty" or "indemnity" if possible
3. State-by-state compliance if needed
4. Partner with licensed insurance carrier if required
5. Limit coverage amounts to stay under regulatory thresholds

**Risk:** Privacy lawsuits for sharing AML data without consent

**Mitigation:**
1. Privacy counsel review (Phase 1)
2. Supplier consent during verification flow
3. Terms of service updates
4. Transparency: show suppliers what buyers see
5. GDPR compliance for EU suppliers

### Business Model Risks

**Risk:** Nobody subscribes to indemnity (wrong product-market fit)

**Mitigation:**
1. Validate demand in Phase 2 (gauge interest, pre-sales)
2. Customer interviews before building Phase 3
3. Pilot program with select buyers
4. Flexible pricing (monthly option reduces commitment barrier)
5. Strong marketing positioning (fear + value)

**Risk:** Fraud is too rare, buyers don't see value

**Mitigation:**
1. Position monitoring and reports as primary value (not just claims)
2. Highlight "peace of mind" and "due diligence" benefits
3. Use case studies and fraud statistics in marketing
4. Offer detailed reports as included benefit (tangible value)
5. Bundle with other services if needed

---

## Next Steps

1. **Review this architecture** with BC and team
2. **Legal consultation** on indemnity structure and coverage limits
3. **Actuarial analysis** (or DIY risk modeling) for pricing and reserves
4. **Decide on Phase 1 scope** and begin implementation
5. **Design Phase 1 UI/UX** (basic verification flow + badges)
6. **Set up project tracking** (tasks, timeline, dependencies)

---

## Appendix: Example Scenarios

### Scenario 1: Clean US Business
**Supplier:** Acme Corp (US, established 2010)

**Verification Flow:**
1. Supplier clicks "Verify Identity"
2. System extracts EIN from uploaded W-9
3. TIN validation: ✅ Valid
4. Didit AML screening: ✅ No hits, Approved, Risk Score: 15
5. Result: Green "✓ Verified" badge
6. Trust score: +28 points (basic verification)

**Buyer View:**
- "✓ Verified" badge
- No issues flagged
- Can optionally purchase detailed report for Middesk data

---

### Scenario 2: Flagged International Business
**Supplier:** Melia Hotels International (Spain, historical Iran business)

**Verification Flow:**
1. Supplier clicks "Verify Identity"
2. System extracts info from W-8BEN-E
3. TIN validation: ✅ Valid (W-8BEN-E format check)
4. Didit AML screening: ⚠️ 3 hits, Approved, Risk Score: 57
5. AI summary generated: "Historical business dealings with Iran..."
6. Result: Yellow "⚠️ Issues Found" badge
7. Trust score: +25 points (flagged but approved)

**Buyer View (Free Tier):**
- "⚠️ Issues Found" badge
- Free AI summary shows:
  - 3 matches (Iran UANI registry)
  - Risk level: Medium
  - Recommendation: Review context before high-value transactions
- CTA: "Get Full Report - $25" or "Subscribe to Indemnity - $75/mo"

**Buyer View (After Purchasing Report):**
- Full Middesk + AML data
- AI executive summary with context
- Recommendation: Approve with monitoring
- Action items for buyer
- PDF download

**Buyer View (Indemnity Subscriber):**
- Same as detailed report (free, counts against quota)
- Supplier added to quarterly monitoring
- Alert if status changes

---

### Scenario 3: Declined Supplier
**Supplier:** Shady LLC (US, fake business)

**Verification Flow:**
1. Supplier clicks "Verify Identity"
2. System extracts EIN from W-9
3. TIN validation: ❌ Invalid (EIN doesn't exist in IRS database)
4. Cannot proceed to AML screening
5. Result: Red "❌ Verification Failed" badge

**Buyer View:**
- "❌ Verification Failed" badge
- Message: "Tax ID could not be validated. Proceed with caution."
- Buyer can still work with supplier (risky) or decline
- If indemnity subscriber: Supplier ineligible for coverage

**Alternative:** EIN valid, but Didit AML returns "Declined" status

1. TIN validation: ✅ Valid
2. Didit AML: ❌ Declined, Risk Score: 95, Active sanctions match
3. AI summary: "This entity appears on OFAC sanctions list..."
4. Result: Red "⚠️ High Risk" badge

**Buyer View:**
- Cannot proceed without manual review
- Free AI summary shows sanctions match
- Strong recommendation: Decline
- If indemnity subscriber: Centime advises against proceeding, no coverage if buyer proceeds anyway

---

### Scenario 4: Indemnity Claim - Payment Fraud
**Supplier:** Fake Supplier LLC
**Buyer:** Acme Manufacturing (indemnity subscriber)
**Scenario:** Supplier provided fraudulent bank account, diverted $15,000 payment

**Flow:**
1. Buyer realizes they were defrauded
2. Files claim in Client Portal:
   - Fraud type: Payment fraud
   - Loss amount: $15,000
   - Description: "Supplier provided fake bank account information, payment was diverted to fraudster"
   - Documents: Invoice, wire transfer confirmation, police report
3. Centime reviews claim:
   - Verify subscription was active ✅
   - Verify supplier was screened ✅ (basic verification, no flags)
   - Verify fraud type is covered ✅
   - Check coverage limit: Per-incident $25K ✅, Annual aggregate remaining: $250K ✅
   - Investigate: Contact buyer for more details, review verification records
   - Decision: **Approved**
4. Calculate payout:
   - Loss: $15,000
   - Deductible: $1,000
   - Payout: $14,000
5. Process payment to buyer
6. Update subscription: Coverage used = $14,000, remaining = $236,000
7. Pursue recovery from fraudster (subrogation) - work with law enforcement

**Result:**
- Buyer receives $14,000
- Claims experience validates indemnity value
- Buyer renews subscription
- Centime improves verification to catch similar fraud in future

---

**END OF DOCUMENT**
