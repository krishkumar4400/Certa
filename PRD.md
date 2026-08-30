# ComplyGPT — V1 Build Plan

## Step-by-Step, In Order

---

## Monorepo Structure

```text
complygpt/
├── apps/
│   ├── web/          # Next.js 14 frontend
│   ├── api/          # Node.js Express API  
│   └── ai/           # Python FastAPI AI service
├── packages/
│   ├── types/        # Shared TypeScript types (used by web + api both)
│   ├── ui/           # Shared shadcn components
│   └── validators/   # Zod schemas (shared validation)
├── docker-compose.yml
└── .github/workflows/
```

Tool: **Turborepo** — manages the monorepo, handles shared packages.

---

## Phase 0 — Project Setup

**Week 1–2**

### Step 1: Repo + Tooling

```bash
npx create-turbo@latest complygpt
cd complygpt

# Web app
cd apps/web
npx create-next-app@latest . --typescript --tailwind --app

# Node API
cd apps/api
npm init -y && npm install express typescript ts-node @types/express

# Python AI service
cd apps/ai
python -m venv venv
pip install fastapi uvicorn langchain anthropic pinecone-client openai pdfplumber
```

Setup:

- ESLint + Prettier (same config across all apps)
- Husky pre-commit hooks (lint + type-check before every commit)
- GitHub repository with branch protection on `main`
- GitHub Actions: CI runs on every PR (lint → typecheck → build)

### Step 2: Infrastructure

Do all of this on Day 1 — don't build without infra ready.

1. **Supabase** → Create project → note down `DATABASE_URL`, `ANON_KEY`, `SERVICE_KEY`
2. **AWS** → Create account → Create S3 bucket in `ap-south-1` → enable AES-256 encryption → enable versioning
3. **Upstash** → Create Redis instance (free tier) → note connection string
4. **Pinecone** → Create serverless index → dimension: 1536, metric: cosine, cloud: AWS, region: ap-south-1
5. **Railway** → Create project → add two services (Node API + Python AI) → connect to GitHub repo
6. **Vercel** → Connect GitHub repo → point to `apps/web` → add env variables
7. **Razorpay** → Create account → create 3 plans (Starter ₹2,999/mo, Growth ₹7,999/mo, Enterprise custom) → note API keys
8. **Interakt** → Create WhatsApp Business account → verify business
9. **Resend** → Create account → verify domain → note API key
10. **Sentry** → Create project → note DSN
11. **Posthog** → Create project → note API key

All API keys go into:

- `.env.local` for local development
- Railway environment variables for production API
- Vercel environment variables for production frontend
- Never committed to Git — add all to `.gitignore`

### Step 3: Database Schema

Run these migrations on Supabase:

```sql
-- Core multi-tenant tables
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name TEXT NOT NULL,
  cin TEXT,                          -- Company Identification Number (from MCA)
  industry TEXT,
  employee_count INTEGER,
  states TEXT[],                     -- Array of states they operate in
  plan_tier TEXT DEFAULT 'starter',  -- starter | growth | enterprise
  razorpay_subscription_id TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE users (
  id UUID PRIMARY KEY REFERENCES auth.users,
  org_id UUID REFERENCES organizations(id),
  full_name TEXT,
  role TEXT DEFAULT 'admin',         -- owner | admin | legal | hr | ciso | auditor
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- DPDP Module
CREATE TABLE dpdp_assessments (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID REFERENCES organizations(id),
  score INTEGER,                     -- 0-100
  answers JSONB,                     -- all 20 question answers
  gaps JSONB,                        -- list of identified gaps
  completed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE data_inventory_items (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID REFERENCES organizations(id),
  data_category TEXT,                -- customer_email | payment_data | health_data etc
  purpose TEXT,
  storage_location TEXT,
  retention_days INTEGER,
  third_parties TEXT[],
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE generated_documents (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID REFERENCES organizations(id),
  doc_type TEXT,                     -- privacy_policy | dpa | ropa | salary_structure | register
  regulation TEXT,                   -- dpdp | labour_code
  content_en TEXT,                   -- generated document content
  pdf_url TEXT,                      -- S3 URL
  docx_url TEXT,                     -- S3 URL
  version INTEGER DEFAULT 1,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Labour Code Module  
CREATE TABLE salary_analyses (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID REFERENCES organizations(id),
  input_structure JSONB,
  compliant_structure JSONB,
  gaps JSONB,
  state TEXT,
  employee_category TEXT,
  annual_impact_rs INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Action Queue (the to-do list)
CREATE TABLE compliance_actions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  org_id UUID REFERENCES organizations(id),
  regulation TEXT,                   -- dpdp | labour_code
  action_type TEXT,                  -- generate_doc | complete_assessment | review_vendor etc
  title TEXT,
  description TEXT,
  priority TEXT,                     -- critical | high | medium | low
  deadline DATE,
  status TEXT DEFAULT 'pending',     -- pending | in_progress | done | dismissed
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- RLS Policies (multi-tenancy security)
ALTER TABLE organizations ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE dpdp_assessments ENABLE ROW LEVEL SECURITY;
ALTER TABLE data_inventory_items ENABLE ROW LEVEL SECURITY;
ALTER TABLE generated_documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE salary_analyses ENABLE ROW LEVEL SECURITY;
ALTER TABLE compliance_actions ENABLE ROW LEVEL SECURITY;

-- Policy: users can only see their own org's data
CREATE POLICY "org_isolation" ON organizations
  USING (id = (SELECT org_id FROM users WHERE id = auth.uid()));

-- Same policy template applied to all tables
-- (write similar policies for each table — org_id = user's org_id)
```

### Step 4: Regulatory RAG Corpus Setup

This is the AI's "brain" — what it reads to give accurate legal advice.

```python
# apps/ai/scripts/seed_corpus.py
# Run this once to populate Pinecone with regulatory knowledge

documents_to_index = [
    # DPDP
    "data/dpdp_act_2023.pdf",
    "data/dpdp_rules_2025.pdf", 
    "data/dpbi_notifications/*.pdf",
    
    # Labour Codes
    "data/code_on_wages_2019.pdf",
    "data/industrial_relations_code_2020.pdf",
    "data/social_security_code_2020.pdf",
    "data/occupational_safety_code_2020.pdf",
    
    # State notifications (10 states)
    "data/state_rules/maharashtra/*.pdf",
    "data/state_rules/karnataka/*.pdf",
    # ... 8 more states
]

# Chunk → Embed → Store in Pinecone
for doc_path in documents_to_index:
    text = extract_pdf_text(doc_path)
    chunks = chunk_text(text, chunk_size=512, overlap=50)
    embeddings = openai.embed(chunks)
    pinecone_index.upsert(zip(chunk_ids, embeddings, chunk_metadata))
```

All these PDFs are freely available from mca.gov.in, labour.gov.in, meity.gov.in.

**Deliverable end of Week 2:**
Local `docker-compose up` runs all 3 apps. DB is live on Supabase with RLS. All infra services created. CI/CD deploys to staging on push.

---

## Phase 1 — Auth + Onboarding + Billing

**Week 2–5**

### Step 5: Authentication

Supabase Auth in Next.js App Router:

```typescript
// apps/web/middleware.ts
import { createMiddlewareClient } from '@supabase/auth-helpers-nextjs'

export async function middleware(req: NextRequest) {
  const res = NextResponse.next()
  const supabase = createMiddlewareClient({ req, res })
  const { data: { session } } = await supabase.auth.getSession()
  
  // Redirect unauthenticated users away from dashboard
  if (!session && req.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', req.url))
  }
  
  return res
}
```

Three login methods:

- Email + Password (primary — B2B users have work emails)
- Google OAuth (for startup founders — one click)
- Magic Link (backup — no password needed)

No phone OTP for ComplyGPT — this is B2B enterprise, not MSME. Email is fine.

### Step 6: Company Onboarding Flow

10 questions. Multi-step wizard. Each step saves to DB immediately (don't lose progress if user closes tab).

```
Screen 1: "What's your company name and CIN?"
          → On CIN enter: auto-fetch from MCA API (free public API)
          → Auto-fills: company name, registered address, incorporation date, director names
          
Screen 2: "How many employees do you have?"
          → Options: 1-10 / 11-50 / 51-200 / 201-500 / 500+
          
Screen 3: "Which states do you operate in?"
          → Multi-select checkboxes (show 10 major states first)
          
Screen 4: "What kind of data do you collect from users?"
          → Multi-select: Customer emails / Payment card data / Health/medical data / 
                          Location data / Biometric data / Children's data / None
                          
Screen 5: "Are you a listed company (on NSE/BSE)?"
          → Yes / No (determines BRSR applicability — flagged for V2)
          
Screen 6: "What's your primary concern right now?"
          → Multi-select: DPDP compliance / Labour Code / Investment due diligence / 
                          Board reporting / All of the above
                          
Screen 7: "Who will use this platform?"
          → Their role: Founder/CEO / CTO/CISO / Legal Counsel / HR Head / Finance / Consultant
          
Screen 8: Invite team members (optional, skip for now)

Screen 9: Choose plan
          → Starter ₹2,999/month (DPDP only, 1 user)
          → Growth ₹7,999/month (DPDP + Labour, 5 users)
          → Enterprise: "Talk to us" (all modules, unlimited users, white-glove)

Screen 10: Razorpay checkout → payment → redirect to dashboard
```

After onboarding completes:

- Auto-generate initial compliance action queue based on answers
- Trigger: "Your initial compliance assessment is ready. 12 actions found." email via Resend

### Step 7: Razorpay Billing Integration

```typescript
// apps/api/src/routes/billing.ts

// Create subscription
router.post('/subscriptions/create', async (req, res) => {
  const { plan_id, org_id } = req.body
  
  const subscription = await razorpay.subscriptions.create({
    plan_id,                          // Razorpay plan ID
    total_count: 12,                  // 12 billing cycles (1 year)
    quantity: 1,
    notify_info: { notify_email: req.user.email }
  })
  
  await db.organizations.update({
    where: { id: org_id },
    data: { razorpay_subscription_id: subscription.id }
  })
  
  res.json({ subscription_id: subscription.id, short_url: subscription.short_url })
})

// Webhook handler (Razorpay calls this on payment events)
router.post('/webhooks/razorpay', async (req, res) => {
  const event = req.body.event
  
  if (event === 'subscription.activated') {
    await activateSubscription(req.body.payload.subscription.entity)
  }
  
  if (event === 'subscription.halted') {
    await downgradeToFree(req.body.payload.subscription.entity)
  }
  
  res.json({ status: 'ok' })
})
```

Feature gating: simple check before every module renders

```typescript
// packages/types/src/plans.ts
const PLAN_FEATURES = {
  starter:    ['dpdp_assessment', 'privacy_policy_gen', 'action_queue'],
  growth:     ['dpdp_assessment', 'privacy_policy_gen', 'action_queue', 
               'labour_code', 'salary_checker', 'contract_analyzer', 'board_report'],
  enterprise: ['*']  // everything
}
```

**Deliverable end of Week 5:**
User can sign up, complete onboarding, pay via Razorpay, land on dashboard with their company name and initial action queue. Feature gating works. Google OAuth works.

---

## Phase 2 — Dashboard Shell + Compliance Score

**Week 4–6 (parallel with Phase 1)**

### Step 8: Main Dashboard Layout

```
┌─────────────────────────────────────────────────────────┐
│  ComplyGPT          Acme Pvt Ltd          🔔  Priya ▼   │
├──────────┬──────────────────────────────────────────────┤
│          │                                              │
│  📊      │   Compliance Score        67 / 100           │
│  Dashboard│   ████████████░░░░░░░  AMBER                │
│          │                                              │
│  🔐      │   ┌────────────┐  ┌─────────────┐           │
│  DPDP    │   │ DPDP       │  │ Labour Code │           │
│          │   │ 54/100 🔴  │  │ 81/100 🟢   │           │
│  👷      │   │ 6 gaps     │  │ 2 gaps      │           │
│  Labour  │   └────────────┘  └─────────────┘           │
│          │                                              │
│  📄      │   Action Queue (8 pending)                   │
│  Reports │   ┌──────────────────────────────────────┐   │
│          │   │ 🔴 Generate DPDP Privacy Policy       │   │
│  ⚙️      │   │    Fine risk: ₹250Cr  Due: Oct 2026  │   │
│  Settings│   │                          [Fix it →]   │   │
│          │   ├──────────────────────────────────────┤   │
│          │   │ 🔴 Complete Data Inventory Mapping    │   │
│          │   │    Required for ROPA                 │   │
│          │   │                          [Fix it →]   │   │
│          │   ├──────────────────────────────────────┤   │
│          │   │ 🟡 Review Vendor DPAs (3 vendors)    │   │
│          │   │    Razorpay, AWS, Segment            │   │
│          │   │                          [Fix it →]   │   │
│          │   └──────────────────────────────────────┘   │
└──────────┴──────────────────────────────────────────────┘
```

### Step 9: Compliance Score Calculation

```typescript
// apps/api/src/services/score.ts

async function calculateComplianceScore(org_id: string): Promise<Score> {
  const weights = {
    dpdp: 0.60,        // 60% weight — highest urgency
    labour: 0.40,      // 40% weight
  }
  
  const dpdpScore = await calculateDPDPScore(org_id)
  const labourScore = await calculateLabourScore(org_id)
  
  const overall = Math.round(
    dpdpScore * weights.dpdp + 
    labourScore * weights.labour
  )
  
  return {
    overall,
    dpdp: dpdpScore,
    labour: labourScore,
    color: overall >= 71 ? 'green' : overall >= 41 ? 'amber' : 'red'
  }
}

// DPDP score based on:
// - Assessment completed? (20 pts)
// - Privacy Policy generated? (20 pts)  
// - Data Inventory done? (20 pts)
// - Vendor DPAs completed? (20 pts)
// - Consent notices setup? (10 pts)
// - Breach playbook reviewed? (10 pts)
function calculateDPDPScore(org_id): number { ... }

// Labour score based on:
// - Salary structure analyzed? (40 pts)
// - Salary compliant? (30 pts)
// - Registers generated? (20 pts)
// - Contracts analyzed? (10 pts)
function calculateLabourScore(org_id): number { ... }
```

Score updates in real-time as user completes actions — satisfying progress loop.

**Deliverable end of Week 6:**
Dashboard loads with real compliance score. Action queue shows specific gaps from onboarding answers. Score updates when an action is marked complete.

---

## Phase 3 — DPDP Module

**Week 6–10**

This is the primary value proposition of ComplyGPT. Build this module with the most care.

### Step 10: Public DPDP Assessment (Lead Magnet)

No login needed. Lives at `/dpdp-check` — this page must rank on Google.

```typescript
// 20 questions across 6 categories
const assessmentQuestions = [
  // Category 1: What data you collect
  { id: 'q1', category: 'collection', 
    question: 'Do you collect personal data from Indian users (name, email, phone, etc.)?',
    options: ['Yes, we collect basic contact info', 'Yes, including sensitive data (health, financial, children\'s)', 'No, B2B only'],
    weight: 10 },
    
  { id: 'q2', category: 'collection',
    question: 'Do you collect data from children under 18?',
    options: ['Yes', 'No', 'Not sure'],
    weight: 8 },
    
  // Category 2: Consent
  { id: 'q3', category: 'consent',
    question: 'How do you get consent from users before collecting their data?',
    options: ['Clear checkbox at signup', 'Pre-ticked checkbox', 'Just a privacy policy link', 'No explicit consent'],
    weight: 12 },
    
  // ... 17 more questions across consent, storage, vendors, rights, breach
]

// Score calculation
function calculatePublicScore(answers: Answer[]): AssessmentResult {
  let score = 0
  const gaps: Gap[] = []
  
  for (const answer of answers) {
    const question = questions.find(q => q.id === answer.questionId)
    const selectedOption = question.options[answer.selectedIndex]
    
    if (selectedOption.isCompliant) {
      score += question.weight
    } else {
      gaps.push({
        regulation: 'DPDP',
        issue: question.gapDescription,
        risk: question.penaltyRisk,
        fix: question.fixDescription
      })
    }
  }
  
  return { score, gaps, freeReport: score < 60 }
  // If score < 60: "You're at risk. Get your full compliance plan →" (drives signup)
}
```

### Step 11: DPDP AI Analysis Pipeline (Python)

```python
# apps/ai/src/services/dpdp_analyzer.py

async def analyze_dpdp_gap(gap: dict, org_profile: dict) -> DPDPAnalysis:
    """
    For each gap identified in the assessment, get specific actionable advice.
    Uses RAG to ground advice in actual DPDP Act provisions.
    """
    
    # 1. Retrieve relevant DPDP Act sections from Pinecone
    relevant_sections = await pinecone_query(
        query=gap['issue'],
        filter={'regulation': 'dpdp'},
        top_k=5
    )
    
    # 2. Generate specific advice with Claude
    response = await claude_client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1000,
        system="""You are an expert DPDP Act compliance advisor. 
        Give specific, actionable advice based on the actual Act provisions provided.
        Always cite the specific Section number from DPDP Act 2023.
        Be practical — this is a real company that needs to fix a real problem.""",
        
        messages=[{
            "role": "user",
            "content": f"""
            Company profile: {json.dumps(org_profile)}
            Compliance gap: {gap['issue']}
            
            Relevant DPDP Act provisions:
            {format_sections(relevant_sections)}
            
            Provide:
            1. Why this is a problem (which section they're violating)
            2. The exact penalty risk
            3. Step-by-step how to fix it
            4. How long it will take to fix
            """
        }]
    )
    
    return parse_analysis(response.content[0].text)
```

### Step 12: Document Generators (The Core Value)

Each generator follows the same pattern:
`User answers questions → AI generates complete document → Save to S3 → Serve download link`

```python
# apps/ai/src/generators/privacy_policy.py

async def generate_privacy_policy(org_profile: dict, data_inventory: dict) -> str:
    """
    Generates a complete DPDP-compliant Privacy Policy.
    This replaces a ₹15,000–50,000 lawyer engagement.
    """
    
    prompt = f"""
    Generate a complete, legally valid Privacy Policy for an Indian company.
    
    Company: {org_profile['name']}
    Industry: {org_profile['industry']}  
    Data collected: {json.dumps(data_inventory['categories'])}
    Third-party processors: {json.dumps(data_inventory['vendors'])}
    
    Requirements:
    - Must comply with DPDP Act 2023 and DPDP Rules 2025
    - Must include all 8 mandatory disclosures under Section 5
    - Must include user rights under Section 11-14 (access, correction, erasure, grievance)
    - Must name a Data Protection Officer or point of contact
    - Must specify data retention periods for each category
    - Language: Plain English, avoid legalese, maximum 1500 words
    - Include "Last updated: {today}" at top
    
    Return only the complete Privacy Policy document, no explanation.
    """
    
    response = await claude_client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=3000,
        messages=[{"role": "user", "content": prompt}]
    )
    
    document_text = response.content[0].text
    
    # Convert to PDF and upload to S3
    pdf_url = await generate_pdf(document_text, org_profile['name'])
    docx_url = await generate_docx(document_text, org_profile['name'])
    
    # Save to DB
    await save_generated_document(
        org_id=org_profile['id'],
        doc_type='privacy_policy',
        regulation='dpdp',
        content_en=document_text,
        pdf_url=pdf_url,
        docx_url=docx_url
    )
    
    return { 'pdf_url': pdf_url, 'docx_url': docx_url }
```

Same generator pattern for:

- Consent notice text
- DPA (Data Processing Agreement with vendors)
- ROPA (Records of Processing Activities — the full data map document)
- Breach response playbook (structured Markdown doc)

### Step 13: Data Inventory Wizard

Guided form that builds the ROPA. This is the most time-intensive feature to design but highest value.

```
Step 1: "Pick all data categories you collect"
        → Customer name, email, phone
        → Payment card / bank details  
        → Location / GPS data
        → Health / medical records
        → Biometric (fingerprint, face)
        → Children's data (under 18)

Step 2 (for each selected category): 
        "Why do you collect [Customer email]?"
        → Account creation / login
        → Marketing emails
        → Order confirmation
        → Customer support
        → Analytics

Step 3: "Where is [Customer email] stored?"
        → Our own servers in India
        → AWS (India region)
        → AWS (outside India) ← ⚠️ flags cross-border transfer issue
        → Google Cloud
        → Third-party SaaS: [text field]

Step 4: "How long do you keep [Customer email]?"
        → Until user deletes account
        → 1 year after last activity  
        → 3 years (for tax compliance)
        → Indefinitely ← ⚠️ flags retention policy issue

Step 5: "Who can access [Customer email]?"
        → Only internal team
        → Shared with: [select from detected vendors]

→ Repeat Steps 2-5 for each data category selected in Step 1

Final Output: Complete ROPA document auto-generated
```

**Deliverable end of Week 10:**
Public DPDP assessment live and SEO-indexed. Authenticated users can complete full assessment, do data inventory wizard, and download Privacy Policy + ROPA + DPA documents in under 30 minutes. Vendor DPA checker works.

---

## Phase 4 — Labour Code Module

**Week 9–13 (parallel with Phase 3)**

### Step 14: Salary Structure Checker

This is the most surprising feature for founders. Most funded Indian startups fail this check.

The rule: Basic + DA must be ≥ 50% of total Cost to Company (CTC).

Most startups do: Basic = 30%, HRA = 20%, Special Allowance = 50% → to minimize PF contribution.

Under the new Code on Wages: that structure is non-compliant. PF must be deducted on the higher base.

```typescript
// Input form
interface SalaryInput {
  employee_tier: string        // 'engineer' | 'manager' | 'vp' etc
  ctc_annual: number           // e.g. 1200000 (₹12 lakh)
  current_components: {
    basic: number              // e.g. 360000
    hra: number                // e.g. 240000  
    special_allowance: number  // e.g. 600000
    lta: number                // e.g. 0
    medical: number            // e.g. 0
    other: number[]            // any other components
  }
  state: string                // 'Maharashtra'
}
```

```python
# AI Analysis
async def check_salary_compliance(salary_input: dict, state: str) -> SalaryAnalysis:
    
    # First: rule-based checks (fast, no AI needed for basic math)
    basic_percentage = salary_input['basic'] / salary_input['ctc_annual']
    pf_base_current = salary_input['basic'] + salary_input['hra']
    
    # Code on Wages check
    basic_compliant = basic_percentage >= 0.50
    
    # Get state minimum wage from DB (no AI needed — seeded data)
    state_min_wage = await get_min_wage(state, salary_input['employee_tier'])
    min_wage_compliant = salary_input['basic'] >= state_min_wage
    
    # For complex analysis: use Claude
    response = await claude_client.messages.create(
        model="claude-haiku-20241022",  # Haiku — simpler task, cheaper
        messages=[{
            "role": "user",
            "content": f"""
            Analyze this salary structure for Labour Code compliance in {state}:
            {json.dumps(salary_input)}
            
            Checks needed:
            1. Code on Wages: Is basic >= 50% of CTC? (Currently {basic_percentage:.0%})
            2. PF Act: Is PF correctly calculated on basic+DA?
            3. Gratuity: Correct formula?
            4. Overtime: If any component is for overtime, is rate correct?
            
            Return JSON: gaps[], compliant_structure{}, additional_pf_cost_annual, risk_level
            """
        }]
    )
    
    return parse_analysis(response.content[0].text)
```

Output UI:

```
Current Structure    vs    Compliant Structure
─────────────────         ─────────────────────
Basic:     ₹30,000        Basic:     ₹50,000 ↑
HRA:       ₹20,000        HRA:       ₹20,000
Spec.All:  ₹50,000        Spec.All:  ₹30,000 ↓
─────────────────         ─────────────────────
CTC:      ₹1,00,000       CTC:      ₹1,00,000 (same)

Additional PF liability: ₹2,400/month per employee
For 50 employees: ₹14.4 lakh/year additional PF cost

Risk if not fixed: PF authorities can demand 3 years back dues + 12% interest + penalty
```

### Step 15: Digital Register Generator

State-specific statutory registers. Pure document generation — no AI needed here.

```python
async def generate_attendance_register(org_id: str, month: str, state: str) -> str:
    """
    Generates Form B Attendance Register in the exact format required 
    by the state Labour Department.
    Each state has slightly different column requirements — handle 10 states.
    """
    
    template = await get_register_template('attendance', state)
    # Template is an Excel template stored in S3, pre-formatted with all headers
    
    # Fill in company header information
    wb = openpyxl.load_workbook(template)
    ws = wb.active
    ws['B2'] = org_profile['name']
    ws['B3'] = org_profile['address']
    ws['B4'] = month
    # ... fill all headers
    
    # Save and upload
    output_path = f'/tmp/{org_id}_attendance_{month}.xlsx'
    wb.save(output_path)
    return await upload_to_s3(output_path)
```

Registers to generate: Attendance (Form B), Wages (Form D), Overtime (Form C), Leave (Form F)

### Step 16: Employment Contract Analyzer

```python
async def analyze_contract(contract_pdf_url: str, state: str) -> ContractAnalysis:
    
    # Extract text from uploaded PDF
    contract_text = await extract_pdf_text(contract_pdf_url)
    
    # Retrieve Labour Code sections from Pinecone
    relevant_law = await pinecone_query(
        query="employment contract clauses Labour Code requirements",
        filter={'regulation': 'labour_code', 'state': state}
    )
    
    response = await claude_client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2000,
        system="""You are a senior Indian employment lawyer. 
        Review employment contracts against the 4 Labour Codes (2020).
        Identify specific non-compliant clauses. Be precise — cite section numbers.""",
        
        messages=[{
            "role": "user",
            "content": f"""
            Review this employment contract for Labour Code compliance in {state}:
            
            CONTRACT:
            {contract_text}
            
            APPLICABLE LAW:
            {format_sections(relevant_law)}
            
            For each issue found, return:
            - clause_text: the problematic text from the contract
            - issue: what's wrong
            - law_reference: which Section/Rule is violated
            - suggested_replacement: compliant replacement clause
            - severity: critical | major | minor
            """
        }]
    )
    
    return parse_contract_issues(response.content[0].text)
```

**Deliverable end of Week 13:**
Salary structure checker works with side-by-side comparison and financial impact calculation. Digital registers downloadable in statutory format for all 10 states. Contract analyzer returns specific redlined issues with section citations.

---

## Phase 5 — Reports

**Week 12–14 (parallel with Phase 4)**

### Step 17: Board Report Generator

One-click PDF. This is what closes enterprise deals — the CFO or CISO can show the board they're taking compliance seriously.

```python
async def generate_board_report(org_id: str) -> str:
    
    # Gather all data
    score = await get_compliance_score(org_id)
    completed_actions = await get_completed_actions(org_id)
    pending_actions = await get_pending_actions(org_id, priority='critical')
    generated_docs = await get_generated_documents(org_id)
    upcoming_deadlines = await get_upcoming_deadlines(org_id, days=90)
    
    # Generate executive summary with Claude
    summary = await claude_client.messages.create(
        model="claude-haiku-20241022",
        messages=[{
            "role": "user",
            "content": f"""
            Write a 150-word executive summary for a board compliance report.
            Tone: professional, factual, no jargon.
            
            Company: {org['name']}
            Overall score: {score['overall']}/100 ({score['color']})
            DPDP score: {score['dpdp']}/100
            Labour score: {score['labour']}/100
            Actions completed this quarter: {len(completed_actions)}
            Critical pending actions: {len(pending_actions)}
            Next major deadline: {upcoming_deadlines[0]['date']} - {upcoming_deadlines[0]['name']}
            """
        }]
    )
    
    # Generate PDF using WeasyPrint (Python library)
    html = render_report_template(
        summary=summary.content[0].text,
        score=score,
        completed=completed_actions,
        pending=pending_actions,
        documents=generated_docs,
        deadlines=upcoming_deadlines
    )
    
    pdf_bytes = weasyprint.HTML(string=html).write_pdf()
    return await upload_to_s3(pdf_bytes, f"board_report_{org_id}_{today}.pdf")
```

Report sections:

1. Executive Summary (AI-written, 150 words)
2. Compliance Score (visual gauge)
3. DPDP Status (what's done, what's pending)
4. Labour Code Status
5. Documents Generated (list with download links)
6. Upcoming Deadlines (next 90 days)
7. Risk Summary (what penalties they're now protected from)

### Step 18: Investor Due Diligence Export

Formatted for Series A/B data rooms (Notion, Google Drive, Confluence format):

```
📁 Compliance/
   ├── Compliance_Score_[Company]_[Date].pdf
   ├── DPDP/
   │   ├── DPDP_Readiness_Assessment.pdf
   │   ├── Privacy_Policy_v1.pdf
   │   ├── Records_of_Processing_Activities.pdf
   │   ├── Vendor_DPA_Tracker.xlsx
   │   └── Breach_Response_Playbook.pdf
   └── Labour/
       ├── Salary_Compliance_Analysis.pdf
       └── Employment_Contract_Review_Summary.pdf
```

Download as a single ZIP file. This is a huge value-add during fundraising.

**Deliverable end of Week 14:**
Board report generates in < 30 seconds. Investor due diligence ZIP works. Both look professional enough to share externally.

---

## Phase 6 — Testing + Security + Launch

**Week 14–17**

### Step 19: Security Hardening

```typescript
// Rate limiting — prevent AI endpoint abuse
import rateLimit from 'express-rate-limit'

const aiEndpointLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,   // 1 hour
  max: 20,                      // 20 AI calls per hour per IP
  message: 'Too many requests'
})

app.use('/api/generate', aiEndpointLimiter)
app.use('/api/analyze', aiEndpointLimiter)

// Input sanitization — prevent prompt injection
function sanitizeForClaude(userInput: string): string {
  // Remove any instruction-like patterns before passing to Claude
  return userInput
    .replace(/ignore previous instructions/gi, '')
    .replace(/system prompt/gi, '')
    .replace(/\[INST\]/gi, '')
    .slice(0, 10000)  // Hard limit on input size
}

// CORS — restrict to your domains only
app.use(cors({
  origin: ['https://complygpt.in', 'https://www.complygpt.in'],
  credentials: true
}))
```

OWASP checklist:

- ✅ SQL injection: Supabase parameterized queries only
- ✅ XSS: React escapes by default, DOMPurify on any innerHTML
- ✅ CSRF: SameSite=Strict cookies
- ✅ Broken Auth: Supabase session management, 24hr token expiry
- ✅ Sensitive data exposure: All PII encrypted at rest (S3 AES-256)
- ✅ Rate limiting: On all AI endpoints

### Step 20: Testing

```
Unit Tests (Jest):
- Score calculation logic (salary compliance math)
- Compliance score calculator
- Action queue prioritization

Integration Tests (Supertest):
- Full flow: onboarding → assessment → document generation → download
- Billing: subscription creation → feature unlock
- Auth: login → session → protected route access

Load Test (k6):
- 50 concurrent users each uploading a PDF
- Verify queue handles backpressure (doesn't crash, jobs complete)
- Verify AI rate limits don't cause user-facing errors

Manual Test:
- All 10 states Labour Code matrix
- All document generators output valid, readable PDFs
- Mobile responsiveness (at least Safari iOS + Chrome Android)
- Score updates in real-time after action completion
```

### Step 21: Monitoring Setup

```typescript
// Sentry — error tracking
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  tracesSampleRate: 0.1,   // 10% of transactions
  environment: process.env.NODE_ENV
})

// Posthog — product analytics
posthog.capture('assessment_completed', {
  org_id: org.id,
  score: result.score,
  gaps_found: result.gaps.length,
  plan_tier: org.plan_tier
})

posthog.capture('document_generated', {
  doc_type: 'privacy_policy',
  generation_time_ms: endTime - startTime,
  org_id: org.id
})

// Track funnel: 
// signup → onboarding_complete → first_assessment → first_document → paid
```

Key alerts to set up in Sentry:

- Any 500 error → immediate Slack notification
- Claude API error rate > 5% → page on-call
- PDF generation failure → retry + alert

---

## Phase 7 — Beta Launch

**Week 17–18**

### Step 22: Soft Launch Checklist

- [ ] Free DPDP Assessment live at `/dpdp-check` — no login, shareable link
- [ ] SEO: Title tag "Free DPDP Compliance Check for Indian Startups", meta description, sitemap
- [ ] Resend email sequences: welcome (Day 0), first tip (Day 2), assessment reminder (Day 5), case study (Day 10)
- [ ] Privacy Policy and Terms of Service live (ironic if a compliance tool doesn't have these)
- [ ] Support: Interakt WhatsApp widget on site
- [ ] Status page: status.complygpt.in (Better Uptime)

### Step 23: Beta GTM — First 20 Customers

Target: 20 paying companies in first 8 weeks. Here's exactly where to find them.

**Channel 1: VC Portfolio (Fastest)**

- Email 5 Indian VCs (Blume, Peak XV, Elevation, Lightspeed, 3one4)
- Subject: "Free DPDP compliance audit for your portfolio — no catch"
- Offer: Free 3-month trial for 5 portfolio companies
- 1 VC with 100 portfolio companies = 100 warm leads

**Channel 2: Your Personal Network**

- Any founder you know, any CTO, any CISO
- Ask for 20-minute call: "Can I show you something I built? Takes 5 minutes to get your DPDP score"
- Don't pitch. Run the assessment. The score does the selling.

**Channel 3: LinkedIn Content**

- Post weekly: "DPDP Deadline: X days left. Here's what Indian startups need to do."
- Share the free assessment tool in every post
- Tag DPDP + compliance + startup keywords
- Consistency beats virality for B2B

**Channel 4: Startup Communities**

- Product Hunt (Monday launch, 12:01 AM EST)
- iSPIRT community Slack
- NASSCOM DeepTech club
- YourStory + Inc42 press outreach

---

## Timeline Summary

| Phase | Weeks | Deliverable |
| --- | --- | --- |
| 0: Setup | 1–2 | Monorepo, infra, DB, CI/CD live |
| 1: Auth + Billing | 2–5 | Signup, onboarding, Razorpay working |
| 2: Dashboard | 4–6 | Compliance score, action queue |
| 3: DPDP Module | 6–10 | Assessment, all 4 document generators |
| 4: Labour Module | 9–13 | Salary checker, registers, contract analyzer |
| 5: Reports | 12–14 | Board report, DD export |
| 6: Testing + Security | 14–17 | Hardened, load-tested, monitored |
| 7: Launch | 17–18 | First 20 paying customers |

---

## Cost to Build V1 (Lean — Solo Founder + 1 Hire)

| Item | Monthly | 5-Month Total |
| --- | --- | --- |
| Claude API (Sonnet + Haiku mix) | ₹20,000 | ₹1,00,000 |
| OpenAI Embeddings (one-time seed) | ₹3,000 | ₹3,000 |
| Pinecone Serverless | ₹4,000 | ₹20,000 |
| Supabase Pro | ₹2,000 | ₹10,000 |
| AWS S3 | ₹2,000 | ₹10,000 |
| Railway (2 services) | ₹3,500 | ₹17,500 |
| Vercel Pro | ₹1,500 | ₹7,500 |
| Upstash Redis | ₹800 | ₹4,000 |
| Interakt WhatsApp | ₹3,000 | ₹15,000 |
| Resend + misc tools | ₹1,500 | ₹7,500 |
| **Infra Total** | **₹41,300** | **₹2,06,500** |
| Part-time Legal Advisor | ₹20,000 | ₹1,00,000 |
| Contract Frontend Dev (if needed) | ₹50,000 | ₹2,50,000 |
| **Total Burn** | **~₹1.1L/month** | **~₹5.5L** |

Realistically: ₹6–8 lakh total to get to first paying customer. Fundable from personal savings or 1 angel check.

---

## V1 Success Metrics

| Metric | Target |
| --- | --- |
| Free DPDP assessments completed | 200+ by Month 2 |
| Assessment → signup conversion | > 20% |
| Paying customers by Week 8 | 20 |
| MRR by Month 3 | ₹1,50,000 (50 customers at avg ₹3,000) |
| Document generation success rate | > 95% |
| Score update latency | < 500ms |
| AI document generation time | < 90 seconds |
| User NPS | > 40 |

---

## What Comes After V1

**V1.5 (Month 5–8):**

- DPDP Consent SDK (embeddable React component for their website)
- Automated regulatory monitoring (scrape MCA/MEITY/Labour.gov.in for new notifications)
- DigiLocker eSign for generated documents
- All 28 states Labour Code matrix

**V2 (Month 9–15):**

- BRSR / ESG module (listed companies — high ACV)
- Multi-company dashboard (for consultants managing multiple clients)
- Breach Response Command Center (automated DPBI notification workflow)
- Big 4 white-label conversations

**V3 (Month 18+):**

- LexBharat (MSME version) — now with the tech stack proven and team in place
- Regulatory Intelligence API
- International expansion (Singapore PDPA, Thailand PDPA)
