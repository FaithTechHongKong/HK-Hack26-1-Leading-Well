# Product Requirements Document (PRD): Care Group Copilot

## 1. Executive Vision & Architectural Altitude

### 1.1 Strategic Outcome ("The Goal Beyond the Goal")

Small-group leaders routinely spend hours before each season guessing which topics, discussion formats, and spiritual themes fit their members, while members hesitate to share vulnerable life struggles or prayer needs if their names are attached to AI-processed records.

The objective of **Care Group Copilot** is not merely to build a digital survey form with an LLM wrapper. The system establishes a **Zero-Knowledge Pastoral Intelligence Engine**: it decouples member identity from member vulnerability at the ingestion boundary, transforms anonymized life-stage, faith-stage, preference, and prayer signals into contextual group intelligence, and generates ready-to-execute week-by-week seasonal plans that leaders review, refine, and approve in minutes.

### 1.2 Current State vs. Future State Operational Model

| **Dimension** | **Current State (Manual Care Group Prep)** | **Future State (Care Group Copilot MVP)** |
|---|---|---|
| **Member Context Gathering** | Informal check-ins or spreadsheets; members self-censor sensitive struggles due to privacy concerns. | Zero-account mobile web link/QR intake capturing demographics, life stages, faith stages, and anonymous prayers. |
| **Privacy & Data Handling** | Raw names tied to personal notes in chat logs or leader notebooks. | **Strictly unlinked anonymity**: zero member accounts, stripped network metadata, and automated PII redaction before AI egress. |
| **Prayer Visibility** | Scattered across long group chat threads; urgent themes get lost. | Dedicated **Group Prayer Board** displaying auto-summarized themes and redacted prayer cards so members can pray for one another. |
| **Seasonal Planning** | Leader spends 2 to 4 hours manually researching topics and writing weekly agendas. | AI synthesizes all anonymized cohort signals into a structured **week-by-week seasonal plan** (short- and long-term alignment) for leader approval. |

### 1.3 SMART Business Objectives & Success Metrics

| **Objective ID** | **Business Objective** | **Key Performance Indicator (KPI)** | **Target (90 Days Post-Launch)** | **Measurement Mechanism** |
|---|---|---|---|---|
| **OBJ-01** | Slash leader seasonal preparation workload | Median time from intake close to approved seasonal plan | **≤ 30 minutes** (≥ 75% reduction vs. 2-hr baseline) | Server telemetry on leader session timestamps |
| **OBJ-02** | Maximize member candor via psychological safety | Anonymous intake completion rate per active care group | **≥ 80%** of invited group members submit | Submitted count vs. leader-declared group headcount |
| **OBJ-03** | Guarantee zero-PII exposure to external LLMs | PII leakage rate into LLM prompts or persistent storage | **0.00%** (100% redaction of names, phones, emails, addresses) | Automated red-team regex/NER audit suite |
| **OBJ-04** | Deliver high-utility, context-tuned seasonal plans | Leader plan approval rate without full regeneration | **≥ 85%** approved with ≤ 3 inline week edits | Plan state transition & edit-diff telemetry |

### 1.4 Top 3 Foundational Architectural Decisions & Trade-Offs

1. **Strictly Unlinked Ephemeral Submission Model vs. Pseudonymous Longitudinal Tokens:**
   - **Decision:** Enforce strictly unlinked submissions with zero member accounts, zero persistent cookies or localStorage IDs, and no cross-season user linking.
   - **Trade-Off:** The system sacrifices tracking an individual member's personal spiritual progression across multiple seasons in exchange for mathematical immunity against identity correlation and member fears of being profiled by AI. Longitudinal intelligence is computed at the *group cohort level* across seasonal snapshots.
2. **Server-Side Local NER + Deterministic Regex Redaction Gate Prior to LLM Egress:**
   - **Decision:** Every free-text submission passes through an isolated, locally hosted PII scrubber (Named Entity Recognition + deterministic patterns) *before* writing to the database or calling any external LLM API. Raw unredacted text is never persisted to disk or application logs.
   - **Trade-Off:** Adds roughly 150 to 300 ms of server-side CPU latency per submission and requires container memory (~1.5 GB for NER models), but eliminates third-party PII exposure and ensures compliance with strict privacy expectations.
3. **Human-in-the-Loop Structured Plan Synthesis vs. Autonomous Member Chatbot:**
   - **Decision:** Restrict AI interaction strictly to backend aggregation and structured JSON plan generation reviewed by the human leader. No messaging bots (WhatsApp/Telegram) or direct member-to-AI chat in MVP.
   - **Trade-Off:** Requires leaders to copy/share the exported plan into their existing group chat manually, but eliminates bot API approval bottlenecks, conversational hallucination risks in pastoral care, and multi-platform webhook maintenance during MVP.

## 2. Scope Boundaries, Actors & System Flow

### 2.1 MVP Scope Definition

**In-Scope (v1.0 MVP):**

- **Leader Authentication & Group Management:** Passwordless magic-link login for leaders; creation of Care Groups and Seasonal Intake Cycles; generation of shareable mobile URLs and QR codes.
- **Strictly Unlinked Member Intake Flow (Mobile Web):** Three-part mobile-first intake covering (1) Onboarding (demographics, life stage, faith stage), (2) Seasonal Group Preferences (topics, formats, cadence, social vs. study balance), and (3) Anonymous Prayer Requests (short-term urgent needs and ongoing life/spiritual burdens).
- **Automated PII-Redaction Pipeline:** Real-time client-side soft nudges plus mandatory server-side NER/regex scrubbing before storage and AI processing.
- **Group Prayer Board:** Read-only mobile web view accessible via the group link displaying AI-clustered prayer themes and redacted brief details of each prayer request.
- **AI Week-by-Week Seasonal Plan Engine:** Context-aware generation of a structured 4-to-12-week seasonal plan calibrated to cohort demographics, life/faith stages, seasonal preferences, and prayer burdens.
- **Leader Review, Inline Editing & Export:** Leader interface to inspect aggregated cohort insights, edit weekly themes/questions/activities inline, approve the final plan, and export as clean text/Markdown/PDF for sharing.

**Explicitly Out-of-Scope (Deferred to Post-MVP):**

- Messaging platform bot integrations (WhatsApp, Telegram, LINE, Discord, Slack).
- Member user accounts, logins, password recovery, or personal member profiles.
- Cross-season tracking of individual members.
- Direct conversational AI chat for group members.
- Automated SMS/email reminder blasts or calendar invites.

### 2.2 Actors & Access Control Matrix

| **Actor Type** | **Role Name** | **Auth Mechanism** | **Permitted Capabilities** | **Restricted Boundaries** |
|---|---|---|---|---|
| **Primary Human Actor** | **Care Group Leader** | Email Magic Link / OAuth (JWT Session) | Create groups/seasons; set headcount; close intake; view aggregated insights & prayer board; trigger/edit/approve/export plans. | Cannot view raw unredacted submissions; cannot view IP addresses, exact timestamps, or individual submission linkages. |
| **Primary Human Actor** | **Care Group Member** | Zero-Auth Capability Token (intake_token via URL/QR) | Submit anonymous onboarding, seasonal preferences, and prayer requests; view Group Prayer Board. | Cannot access leader dashboard, cohort analytics, or unapproved AI plan drafts. |
| **Internal System Actor** | **PII Redaction Gate** | Internal Service Mesh / Local Process | Inspect free-text inputs, redact entities ([PERSON], [PHONE], [EMAIL], [ADDRESS]), emit sanitized text. | Stateless; prohibited from writing raw input payloads to disk, telemetry, or error logs. |
| **External System Actor** | **LLM Synthesis Engine** | API Key (Zero-Data-Retention Endpoint) | Receive aggregated cohort stats + redacted prayer corpus; return schema-validated week-by-week JSON plan. | Zero direct access to database or raw user requests. |

### 2.3 End-to-End System Architecture & Data Flow

```text
+------------------+       +-----------------------+       +------------------------+
| Care Group Member|       |  API Gateway / Edge   |       |  PII Redaction Gate    |
| (Mobile Browser) |       |  (Strip IP/UserAgent) |       |  (Local NER + Regex)   |
+--------+---------+       +-----------+-----------+       +-----------+------------+
         |                             |                               |
         | 1. POST /intake (Raw JSON)  |                               |
         +---------------------------->| 2. Forward stripped payload   |
         |                             +------------------------------>|
         |                             |                               | 3. Scrub PII entities
         |                             |                               |    (Never log raw text)
         |                             |       4. Persist Sanitized    |
         |                             |          Record + Jitter Time |
         |                             |<------------------------------+
         | 5. 201 Created (No Cookie)  |                               |
         |<----------------------------+                               |
         |                                                             |
+--------+---------+       +-----------------------+       +-----------+------------+
| Care Group Leader|       |   PostgreSQL (ZDR)    |       |  LLM Synthesis Engine  |
| (Mobile/Desktop) |       |   (Unlinked Tables)   |       |  (Structured JSON Out) |
+--------+---------+       +-----------+-----------+       +-----------+------------+
         |                             |                               |
         | 6. POST /seasons/:id/generate                               |
         +---------------------------->| 7. Fetch Cohort Stats +       |
         |                             |    Redacted Prayers (N >= 3)  |
         |                             +------------------------------>|
         |                             |                               | 8. Synthesize Weekly
         |                             |       9. Save Draft Plan      |    Plan (Short & Long
         |                             |<------------------------------+    Term Alignment)
         | 10. Render Draft Plan &     |                               |
         |     Prayer Themes for Edit  |                               |
         |<----------------------------+                               |
         | 11. Inline Edit & Approve   |                               |
         +---------------------------->| 12. Lock Plan & Enable Export |
```

## 3. Business Requirements (BR-01 through BR-07)

| **BRD ID** | **Business Requirement Statement** | **Rationale & Source Alignment** | **Priority** | **Acceptance Benchmark** |
|---|---|---|---|---|
| **BR-01** | The system shall enable members to submit onboarding profiles, seasonal preferences, and prayer requests without creating accounts or attaching names. | Eliminates fear of personal data being fed into AI models, increasing honesty and participation. | Must-Have (P0) | Zero PII or session identifiers stored with member payloads. |
| **BR-02** | The system shall capture structured member onboarding data across demographics, life stages, and faith stages. | Provides foundational cohort context so seasonal curriculum matches members' actual maturity and life phase. | Must-Have (P0) | All 3 onboarding dimensions captured and aggregated per season. |
| **BR-03** | The system shall capture member preferences for upcoming care group seasons (topics, study vs. fellowship ratio, activities). | Aligns group direction with member desires rather than top-down leader guesswork. | Must-Have (P0) | Structured preference distribution available to AI engine. |
| **BR-04** | The system shall collect anonymous free-text prayer requests and strip all PII before AI ingestion or storage. | Prayer requests reveal immediate life and spiritual challenges, giving AI deep context while protecting member privacy. | Must-Have (P0) | 100% redaction of test PII corpus before DB persistence. |
| **BR-05** | The system shall display a shared Group Prayer Board presenting summarized themes and brief redacted details of each prayer request. | Allows group members to pray meaningfully for one another and keeps leaders aware of group burdens. | Must-Have (P0) | Prayer board renders themes + redacted cards without leaking identity. |
| **BR-06** | The system shall synthesize all anonymized onboarding, preference, and prayer data into a structured week-by-week seasonal plan. | Directly reduces leader prep burden while fine-tuning short-term and long-term sessions to member needs. | Must-Have (P0) | Generates complete W-week JSON agenda within 15 seconds. |
| **BR-07** | The system shall require explicit leader review, inline editing capability, and approval before finalizing and exporting a seasonal plan. | Preserves pastoral oversight, theological alignment, and human accountability over AI-generated content. | Must-Have (P0) | Plans remain in DRAFT until leader triggers APPROVE. |

## 4. Detailed Functional Requirements (FR-01 through FR-08)

### Module A: Leader Group & Season Setup

**FR-01: Care Group & Seasonal Intake Creation**

- **Parent BRD:** BR-01, BR-07
- **Primary Actor:** Care Group Leader
- **Pre-conditions:** Leader is authenticated via valid magic-link session.
- **Inputs:** group_name (string), expected_headcount (integer, 3–50), season_title (string), duration_weeks (integer, 4–12), meeting_cadence (enum: WEEKLY, BIWEEKLY), intake_deadline (ISO-8601 date).
- **Processing Logic:**
   1. Validate input constraints and generate a cryptographically random, unguessable intake_token (128-bit URL-safe Base64 string).
   2. Create care_group and season records in state INTAKE_OPEN.
   3. Generate a mobile-optimized share path (/g/{intake_token}) and an SVG QR code.
- **Outputs:**
   - *Success:* HTTP 201 returning Season Dashboard with one-tap "Copy Invite Link" and downloadable QR code.
   - *Error:* HTTP 400 with field-level validation errors if duration_weeks is outside [4, 12] or expected_headcount < 3.

### Module B: Strictly Unlinked Member Intake (Mobile-First Web)

**FR-02: Zero-Auth Member Onboarding Capture**

- **Parent BRD:** BR-01, BR-02
- **Primary Actor:** Care Group Member
- **Pre-conditions:** Member opens valid /g/{intake_token} route while season status is INTAKE_OPEN.
- **Inputs:**
   - age_bracket: Enum (18-24, 25-34, 35-44, 45-54, 55+, PREFER_NOT_TO_SAY)
   - life_stages: Multi-select Enum (SINGLE, DATING_ENGAGED, NEWLY_MARRIED, PARENTING_YOUNG_KIDS, PARENTING_TEENS, EMPTY_NESTER, CAREGIVING_PARENTS, CAREER_TRANSITION, STUDENT, RETIRED)
   - faith_stage: Single-select Enum (EXPLORING_SEEKING, NEW_BELIEVER, GROWING_CONSISTENT, DEEPENING_MATURE, FEELING_DRY_BURNT_OUT, RETURNING_RECONNECTING)
- **Processing Logic:**
   1. Render a mobile-first stepper UI (Step 1 of 3) requiring zero login or personal identifiers.
   2. Hold Step 1 selections strictly in browser memory (React state / sessionStorage cleared on submit) until final submission of the 3-step intake wizard so each member's onboarding context stays bound to their seasonal preference and prayer request within a single atomic payload—without ever identifying *who* the member is.
- **Outputs:** Advances UI to Step 2 (Seasonal Preferences).

**FR-03: Seasonal Group Preferences Intake**

- **Parent BRD:** BR-03
- **Primary Actor:** Care Group Member
- **Pre-conditions:** Step 1 completed in active wizard session.
- **Inputs:**
   - desired_topics: Multi-select (up to 3) from curated tags (BOOK_OF_BIBLE_STUDY, PRAYER_AND_SPIRITUAL_DISCIPLINES, WORK_AND_FAITH, RELATIONSHIPS_AND_FAMILY, ANXIETY_AND_MENTAL_RESILIENCE, APOLOGETICS_AND_DOUBT, SERVING_AND_OUTREACH, TOPICAL_SERIES) plus optional custom topic tag (max 60 chars, PII-scrubbed).
   - session_format_ratio: Slider (1 = Deep Bible Study to 5 = Relational Sharing & Fellowship).
   - social_gathering_interest: Enum (NONE, ONCE_PER_SEASON, MONTHLY).
- **Processing Logic:** Validate selections and advance wizard to Step 3 (Anonymous Prayer Requests).
- **Outputs:** Advances UI to Step 3.

**FR-04: Anonymous Prayer Request Submission & Real-Time Privacy Nudges**

- **Parent BRD:** BR-01, BR-04
- **Primary Actor:** Care Group Member
- **Pre-conditions:** Step 2 completed in active wizard session.
- **Inputs:** prayer_items: Array of 0 to 3 prayer objects, each containing:
   - category: Enum (HEALTH_HEALING, WORK_FINANCES, FAMILY_RELATIONSHIPS, SPIRITUAL_GROWTH, ANXIETY_PEACE, GRATITUDE_PRAISE, OTHER)
   - time_horizon: Enum (IMMEDIATE_SHORT_TERM, ONGOING_LONG_TERM)
   - raw_text: Free-text string (max 500 characters per prayer)
   - share_on_board: Boolean (default true; allows member to choose whether the redacted prayer appears on the shared Group Prayer Board or is used *only* as private context for AI seasonal planning).
- **Processing Logic:**
   1. **Client-Side Soft Nudge:** As the user types in raw_text, run a lightweight local regex check for capitalized two-word name patterns, phone numbers, or emails; display a non-blocking helper pill: *"Privacy tip: Avoid using real names—our filter will automatically mask names and contact details before saving."*
   2. **Atomic Submission (POST /api/v1/intake/{intake_token}):** Edge proxy strips X-Forwarded-For, CF-Connecting-IP, User-Agent, and Referer headers before forwarding payload to the backend.
   3. **Server-Side PII Redaction:** Backend passes custom_topic and every raw_text through the PII Redaction Gate (FR-05).
   4. **Timestamp Jittering:** Apply a random temporal jitter (±12 hours within the intake window) to created_at so leaders cannot correlate submission timestamps with when a member was seen on their phone.
   5. **Persistence:** Write sanitized anonymous_submission and redacted_prayer_items in a single database transaction. Discard raw payload from memory immediately.
- **Outputs:**
   - *Success:* HTTP 201 returning a confirmation screen with a direct link to the Group Prayer Board.
   - *Error:* HTTP 422 if payload exceeds character limits or 403 if season status is no longer INTAKE_OPEN.

**FR-05: Automated PII-Redaction Pipeline**

- **Parent BRD:** BR-04
- **Primary Actor:** PII Redaction Gate (System)
- **Pre-conditions:** Incoming free-text string from FR-03 or FR-04.
- **Inputs:** raw_text (string).
- **Processing Logic:**
   1. Execute deterministic regex passes to replace emails ([EMAIL]), phone numbers ([PHONE]), URLs ([LINK]), and street/unit numbers ([ADDRESS]).
   2. Execute local NER model (Microsoft Presidio + spaCy en_core_web_trf / multilingual NER) to detect PERSON, ORG, GPE (specific neighborhoods/workplaces), and DATE_OF_BIRTH.
   3. Replace detected person names with consistent context-preserving tokens within the single submission (e.g., *"Pray for Sarah and Sarah's mom"* → *"Pray for [Person A] and [Person A]'s mom"*).
   4. Generate a brief_board_summary (max 140 chars, stripped of specific situational identifiers while preserving prayer burden) for the public Group Prayer Board.
- **Outputs:** sanitized_text (for AI planning context), brief_board_summary (for Prayer Board), and redaction_count (integer metadata only).

### Module C: Group Prayer Board & Thematic Aggregation

**FR-06: Group Prayer Board Rendering**

- **Parent BRD:** BR-05
- **Primary Actor:** Care Group Member, Care Group Leader
- **Pre-conditions:** Season has ≥ 3 total submissions (k-anonymity threshold met).
- **Inputs:** intake_token (or authenticated season_id for Leader).
- **Processing Logic:**
   1. If total submissions < 3, display a privacy-protection placeholder: *"Prayer Board unlocks once at least 3 members have submitted to protect everyone's anonymity."*
   2. When submissions ≥ 3, query all redacted_prayer_items where share_on_board = true.
   3. Cluster items by category and LLM-generated **Summarized Theme Headers** (e.g., *"Navigating Career Transitions & Financial Peace (4 requests)"*, *"Family Health & Caregiving Strength (3 requests)"*).
   4. Render each theme block followed by brief, redacted prayer cards (brief_board_summary + time_horizon badge) with an interactive *"I Prayed"* counter (stored as an aggregate integer increment with zero user tracking).
- **Outputs:** Mobile-first Group Prayer Board view showing thematic clusters and redacted prayer cards.

### Module D: AI Seasonal Plan Synthesis & Leader Approval

**FR-07: Context-Aware Week-by-Week Seasonal Plan Generation**

- **Parent BRD:** BR-06
- **Primary Actor:** Care Group Leader (Trigger) → LLM Synthesis Engine
- **Pre-conditions:** Leader is authenticated; season has ≥ 3 submissions; leader clicks **"Generate Seasonal Plan"** (or **"Regenerate with Guidance"**).
- **Inputs:**
   - Aggregated cohort distribution (age brackets, life stages, faith stages, format slider average, top desired topics).
   - Full corpus of sanitized_text prayer requests (both IMMEDIATE_SHORT_TERM and ONGOING_LONG_TERM horizons, including items marked share_on_board = false).
   - duration_weeks (W ∈ [4, 12]) and optional leader_steering_note (e.g., *"Focus on Book of James while addressing anxiety and parenting burnout"*).
- **Processing Logic:**
   1. Construct a structured system prompt instructing the LLM to act as an experienced small-group curriculum designer.
   2. Enforce **Dual-Horizon Calibration**:
      - **Weeks 1–3 (Short-Term Calibration):** Directly address acute IMMEDIATE_SHORT_TERM prayer burdens, build psychological safety, and match the cohort's session_format_ratio.
      - **Weeks 4–W (Long-Term Arc):** Progressively build spiritual depth tailored to the cohort's faith_stage distribution and ONGOING_LONG_TERM life-stage challenges.
   3. Enforce strict JSON schema output containing seasonal_arc_summary and weeks[] (exact length W), where each week object contains: week_number, title, core_theme, cohort_alignment_note, icebreaker, scripture_or_reading_anchor, discussion_questions (3–4 open-ended questions), group_prayer_focus, and suggested_activity_or_social.
   4. Validate output against JSON Schema; retry up to 2 times on schema mismatch before surfacing a graceful error.
- **Outputs:** Saved seasonal_plan in LEADER_REVIEW state; renders structured week-by-week timeline in Leader Workspace.

**FR-08: Leader Inline Plan Editor, Approval Lock & Export**

- **Parent BRD:** BR-07
- **Primary Actor:** Care Group Leader
- **Pre-conditions:** A generated seasonal_plan exists in LEADER_REVIEW state.
- **Inputs:** Leader inline edits to any week's title, icebreaker, discussion_questions, group_prayer_focus, week reordering, or single-week AI regeneration (POST /api/v1/plans/{plan_id}/weeks/{week_num}/regenerate).
- **Processing Logic:**
   1. Persist inline edits automatically via debounced PATCH requests.
   2. When leader clicks **"Approve Seasonal Plan"**, transition season state to APPROVED_ACTIVE, lock intake against new submissions, and snapshot the approved version (v1.0-approved).
   3. Provide one-tap export formats optimized for mobile sharing: Copy Full Plan (WhatsApp/Telegram Formatted Text), Copy Single Week Agenda, and Download Printable PDF.
- **Outputs:** Locked APPROVED_ACTIVE seasonal plan and ready-to-share formatted outputs.

## 5. Business Rules, Decision Logic & State Transitions

### 5.1 Business Rules & Decision Logic Matrix

| **Rule ID** | **Category** | **Rule Condition (IF)** | **System Action (THEN)** | **Rationale** |
|---|---|---|---|---|
| **BRULE-01** | Privacy / K-Anonymity | submitted_cohort_count < 3 | Block rendering of disaggregated onboarding charts, Prayer Board cards, and AI Plan Generation. | Prevents a leader from deducing who wrote a submission in a 1- or 2-response group. |
| **BRULE-02** | Privacy / Edge Stripping | Any HTTP request hits /api/v1/intake/* or /api/v1/board/* | Reverse proxy overwrites IP, User-Agent, and Referer headers with null values before app routing. | Guarantees zero network-level identity correlation. |
| **BRULE-03** | Safety / Crisis Guardrail | raw_text triggers high-confidence self-harm or imminent abuse classifier | Redact PII as normal; flag anonymized alert banner on Leader Dashboard with local crisis helpline resources. | Protects member safety without breaking anonymity. |
| **BRULE-04** | AI Generation Quota | Leader triggers full plan regeneration > 5 times per season | Disable full regeneration button; allow targeted single-week regeneration (up to 10/season). | Controls LLM API token spend while preserving leader flexibility. |
| **BRULE-05** | Intake Lifecycle | Season transitions to APPROVED_ACTIVE | Reject new member POSTs to /intake/{token} with 410 Gone; keep Prayer Board readable. | Freezes planning baseline once leader commits to the seasonal schedule. |

### 5.2 Season Lifecycle State Transition Matrix

| **Current State** | **Trigger Event** | **Guard Condition** | **Next State** | **Automated System Action** |
|---|---|---|---|---|
| DRAFT | Leader publishes season | Valid duration_weeks & title | INTAKE_OPEN | Activate intake_token URL & QR code. |
| INTAKE_OPEN | Member submits intake | Valid payload & PII scrubbed | INTAKE_OPEN | Increment submission_count; refresh Prayer Board themes if count ≥ 3. |
| INTAKE_OPEN | Leader clicks "Generate Plan" | submission_count ≥ 3 | LEADER_REVIEW | Execute FR-07 AI synthesis; persist draft week-by-week plan. |
| LEADER_REVIEW | Leader edits week inline | Plan not yet approved | LEADER_REVIEW | Auto-save field diffs; preserve edit history. |
| LEADER_REVIEW | Leader clicks "Approve Plan" | All W weeks non-empty | APPROVED_ACTIVE | Lock plan version; close intake URL; unlock WhatsApp/PDF exports. |
| APPROVED_ACTIVE | Season end date + 14 days | Cron sweep runs | ARCHIVED | Retain aggregated stats & approved plan; purge granular free-text prayer items per retention policy. |

## 6. Data Dictionary, Validation Rules & Privacy Schema

### 6.1 Core Relational Schema (Zero-PII Architecture)

```sql
-- 1. Authenticated Leaders Only (Zero Member Records Exist in DB)
CREATE TABLE leaders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email_hash TEXT UNIQUE NOT NULL,
    encrypted_email TEXT NOT NULL,
    display_name VARCHAR(80) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 2. Care Groups
CREATE TABLE care_groups (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    leader_id UUID NOT NULL REFERENCES leaders(id) ON DELETE CASCADE,
    group_name VARCHAR(100) NOT NULL,
    expected_headcount SMALLINT NOT NULL CHECK (expected_headcount BETWEEN 3 AND 50),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 3. Seasonal Cycles
CREATE TABLE seasons (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    group_id UUID NOT NULL REFERENCES care_groups(id) ON DELETE CASCADE,
    title VARCHAR(120) NOT NULL,
    intake_token VARCHAR(64) UNIQUE NOT NULL,
    duration_weeks SMALLINT NOT NULL CHECK (duration_weeks BETWEEN 4 AND 12),
    meeting_cadence VARCHAR(16) NOT NULL CHECK (meeting_cadence IN ('WEEKLY', 'BIWEEKLY')),
    status VARCHAR(24) NOT NULL DEFAULT 'INTAKE_OPEN',
    regeneration_count SMALLINT NOT NULL DEFAULT 0,
    intake_deadline TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 4. Strictly Unlinked Anonymous Submissions (NO user_id, NO ip_address, NO device_id)
CREATE TABLE anonymous_submissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id UUID NOT NULL REFERENCES seasons(id) ON DELETE CASCADE,
    age_bracket VARCHAR(24) NOT NULL,
    life_stages JSONB NOT NULL,          -- Array of validated life-stage enums
    faith_stage VARCHAR(32) NOT NULL,
    desired_topics JSONB NOT NULL,       -- Array of topic enums + PII-scrubbed custom topic
    session_format_ratio SMALLINT NOT NULL CHECK (session_format_ratio BETWEEN 1 AND 5),
    social_gathering_interest VARCHAR(24) NOT NULL,
    jittered_submitted_at TIMESTAMPTZ NOT NULL -- Randomized +/- 12h window
);

-- 5. Sanitized Prayer Requests (Post-PII Redaction Gate Only)
CREATE TABLE redacted_prayer_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    submission_id UUID NOT NULL REFERENCES anonymous_submissions(id) ON DELETE CASCADE,
    season_id UUID NOT NULL REFERENCES seasons(id) ON DELETE CASCADE,
    category VARCHAR(32) NOT NULL,
    theme_cluster VARCHAR(100),          -- AI-assigned theme label for Prayer Board
    time_horizon VARCHAR(24) NOT NULL CHECK (time_horizon IN ('IMMEDIATE_SHORT_TERM', 'ONGOING_LONG_TERM')),
    sanitized_text TEXT NOT NULL,        -- Used for AI seasonal plan fine-tuning
    brief_board_summary VARCHAR(160) NOT NULL, -- Displayed on shared Group Prayer Board
    share_on_board BOOLEAN NOT NULL DEFAULT TRUE,
    prayed_count INTEGER NOT NULL DEFAULT 0
);

-- 6. AI-Generated & Leader-Approved Seasonal Plans
CREATE TABLE seasonal_plans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id UUID UNIQUE NOT NULL REFERENCES seasons(id) ON DELETE CASCADE,
    seasonal_arc_summary TEXT NOT NULL,
    weeks_json JSONB NOT NULL,           -- Structured array of W week objects
    is_approved BOOLEAN NOT NULL DEFAULT FALSE,
    approved_at TIMESTAMPTZ,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 6.2 Field-Level Data Dictionary & Validation Rules

| **Field Name** | **Data Type** | **Mandatory** | **Validation Constraints** | **User-Facing Error Message** |
|---|---|---|---|---|
| group_name | VARCHAR(100) | Yes | min_len: 2, max_len: 100, trimmed | "Please enter a care group name between 2 and 100 characters." |
| duration_weeks | SMALLINT | Yes | Integer ∈ [4, 12] | "Seasonal plans must be between 4 and 12 weeks long." |
| life_stages | JSONB (Array) | Yes | 1 ≤ length ≤ 4, values ∈ LifeStageEnum | "Please select at least 1 and up to 4 current life stages." |
| faith_stage | VARCHAR(32) | Yes | Exact match ∈ FaithStageEnum | "Please select the option that best describes your current faith journey." |
| desired_topics | JSONB (Array) | Yes | 1 ≤ length ≤ 3 | "Select at least 1 topic you would like the group to explore this season." |
| prayer_items[].raw_text | TEXT | Conditional | Required if array length > 0; 10 ≤ length ≤ 500 | "Please keep each prayer request between 10 and 500 characters." |

## 7. Detailed Use Cases & BDD Acceptance Criteria

### UC-01: Member Submits Anonymous Onboarding, Preferences & Prayer Request

- **Actors:** Care Group Member (Primary), PII Redaction Gate (System)
- **Pre-conditions:** Leader has created Season S1 (status = INTAKE_OPEN) and shared link /g/{token}.
- **Main Flow:**
   1. Member taps /g/{token} on mobile phone; app loads in < 1.5s with zero login wall.
   2. Member completes Step 1 (Age bracket, Life stages, Faith stage), Step 2 (Desired topics, Study/Fellowship slider, Social preference), and Step 3 (2 prayer requests: one short-term work anxiety request mentioning a colleague's name *"David"*, one ongoing spiritual growth request).
   3. Member taps **"Submit Anonymously"**.
   4. Edge proxy strips IP/User-Agent headers; PII Redaction Gate masks *"David"* → *"[Person A]"* and generates brief_board_summary.
   5. Database saves sanitized record with jittered timestamp; member sees confirmation and link to Group Prayer Board.
- **Exception Flow (E1 - Season Closed):** If leader has already approved the seasonal plan (APPROVED_ACTIVE), member sees a friendly notice (*"This season's planning intake is closed, but you can still view and pray for your group's prayer board below"*) with a direct button to the Prayer Board.

### UC-02: Leader Generates, Fine-Tunes & Approves Week-by-Week Seasonal Plan

- **Actors:** Care Group Leader (Primary), LLM Synthesis Engine (External)
- **Pre-conditions:** Season S1 has 7 anonymous submissions (≥ 3 k-anonymity threshold).
- **Main Flow:**
   1. Leader opens Season Dashboard, reviews aggregated cohort breakdown (e.g., *57% Parenting Young Kids, 43% Feeling Dry/Burnt Out, top prayer themes: Work Stress & Family Health*).
   2. Leader clicks **"Generate 8-Week Seasonal Plan"**.
   3. System invokes LLM Synthesis Engine with cohort distributions and all sanitized_text prayer requests.
   4. Within 12 seconds, an 8-week structured plan renders: Weeks 1–2 focus on rest, burnout recovery, and bearing burdens (short-term relief); Weeks 3–8 walk through practical faith habits for busy parents (long-term growth).
   5. Leader edits Week 3's icebreaker inline and clicks **"Approve Seasonal Plan"**.
   6. Plan locks into APPROVED_ACTIVE; leader taps **"Copy for WhatsApp"** to share Week 1's agenda with the group.

### BDD / Gherkin Verification Scenarios

```gherkin
Feature: Privacy-First Intake & AI Seasonal Plan Synthesis

  Scenario: PII is stripped before database persistence and LLM prompt construction
    Given a Care Group season is in "INTAKE_OPEN" status
    When an unauthenticated member submits a prayer request containing "Pray for Michael Chen at 555-019-2834 regarding his surgery at Mercy Hospital"
    Then the HTTP response status is 201 Created with zero Set-Cookie headers
    And the persisted "sanitized_text" equals "Pray for [Person A] at [PHONE] regarding his surgery at [ORG]"
    And the raw string "Michael Chen" does not appear anywhere in the database or application logs

  Scenario: K-Anonymity gate prevents deanonymization in under-populated intakes
    Given a Care Group season has only 2 submitted anonymous responses
    When the Care Group Leader or a Member accesses the Group Prayer Board or Plan Generator
    Then the system displays the k-anonymity privacy lock message requiring at least 3 responses
    And the "Generate Seasonal Plan" action remains disabled

  Scenario: AI generates dual-horizon week-by-week plan calibrated to member context
    Given a Care Group season has 6 submissions with 4 "IMMEDIATE_SHORT_TERM" prayers about anxiety and "FEELING_DRY_BURNT_OUT" faith stages
    And the leader configured a 6-week season
    When the leader triggers seasonal plan generation
    Then the system produces a valid JSON plan with exactly 6 sequential week objects
    And Weeks 1 and 2 explicitly address rest and anxiety relief in their "cohort_alignment_note" and "group_prayer_focus"
    And the plan status is set to "LEADER_REVIEW" requiring explicit approval before export
```

## 8. Non-Functional Requirements (IEEE 830 / ISO 29148 Quality Targets)

| **Quality Attribute** | **Metric / Standard** | **Measurable Target** | **Verification Method** |
|---|---|---|---|
| **Mobile Performance** | Largest Contentful Paint (LCP) & First Input Delay | **LCP ≤ 1.8s** on simulated 4G mobile; UI interactions **≤ 100ms** | Lighthouse CI mobile audit & Playwright profiling |
| **AI Synthesis Latency** | End-to-end week-by-week plan generation time | **p95 ≤ 15.0s** with real-time step progress indicator | API load benchmark with 12-week, 25-member synthetic payload |
| **Zero-Knowledge Privacy** | Network & storage metadata hygiene | **0** IP addresses, cookies, browser fingerprints, or third-party analytics scripts on member routes | Automated OWASP ZAP / header inspection & DB schema audit |
| **PII Redaction Recall** | Named entity & contact pattern masking accuracy | **≥ 99.0% recall** on synthetic benchmark of 500 prayer requests containing names, phones, emails, and addresses | Automated adversarial red-team test suite in CI |
| **Accessibility** | WCAG 2.1 Level AA & Mobile Ergonomics | Contrast ratio **≥ 4.5:1**; all interactive controls **≥ 48 × 48px** touch targets | axe-core automated accessibility test suite |
| **Data Retention** | Automated ephemeral purge of sensitive prayer text | Granular redacted_prayer_items automatically purged **14 days post-season end** | Automated daily PostgreSQL cron job verification |

## 9. Recommended Technical Architecture, Stack & Cost Trade-Offs

### 9.1 Primary Stack Recommendation (Optimized for Rapid MVP & Hard Privacy Boundaries)

- **Frontend (Mobile-First Web App):** **Next.js (App Router) + TypeScript + Tailwind CSS + shadcn/ui** — Server-side rendering delivers instant mobile load times on weak church/cafe Wi-Fi; shared TypeScript types with Zod enforce strict client/server schema parity.
- **Backend API & PII Redaction Microservice:** **Python (FastAPI) + Microsoft Presidio (spaCy NER)** alongside **PostgreSQL (Supabase or Neon)** — Running Presidio locally inside the FastAPI container guarantees that PII scrubbing happens *in-memory inside your own infrastructure* before any database write or external API call, avoiding sending unredacted prayer text to a secondary PII-scrubbing SaaS.
- **LLM Synthesis Engine:** **OpenAI (gpt-4o / gpt-4.1 with Structured Outputs) or Anthropic (claude-sonnet-4-5) via Zero-Data-Retention (ZDR) API Tier** — Enforced JSON Schema guarantees the week-by-week plan never breaks UI rendering, while ZDR ensures prompts are never logged or used for model training.

### 9.2 Credible Alternative & Explicit Trade-Offs

- **Credible Alternative:** Pure TypeScript/Node.js serverless stack (Vercel Functions + regex-only PII scrubber + LLM-based PII scrubbing).
- **Why Rejected for Primary Recommendation:** Using an external LLM to scrub PII means *unredacted* member names and sensitive prayers must leave your server first—directly violating the core user trust promise. Local Python NER (Presidio) adds a lightweight Docker container (~25/mo on Railway/Fly.io/Render) but preserves an airtight privacy boundary.

### 9.3 Operating Cost Estimate & Reversal Conditions

- **Estimated Monthly Operating Cost (MVP Pilot — 100 Active Care Groups, ~1,200 Members):**
   - Managed PostgreSQL (Neon/Supabase Pro): **$25/mo**
   - Container Hosting (FastAPI + Presidio NER + Next.js on Fly.io/Railway): **45/mo**
   - LLM API Usage (~200 plan generations/mo @ ~4k input / 3k output tokens): **15/mo**
   - **Total Estimated Run Rate:** **~85 / month**
- **When This Architecture Becomes Wrong (Reversal Conditions):** If adoption scales beyond **2,500 concurrent seasonal intakes** or requires **multi-language NER** (e.g., Traditional Chinese, Spanish, Korean mixed in the same prayer text), switch the single-node spaCy NER model to a dedicated multi-lingual transformer inference endpoint (ONNX-quantized XLM-RoBERTa) and introduce a Redis job queue (BullMQ/Celery) for asynchronous plan generation.

## 10. Requirements Traceability Matrix & Team Build Split

### 10.1 End-to-End Requirements Traceability Matrix (RTM)

| **BRD ID** | **FRD ID** | **Module & Feature** | **Test Case Ref** | **MVP Priority** |
|---|---|---|---|---|
| BR-01, BR-07 | FR-01 | Leader Auth, Group & Season Creation, QR/Link Generator | TC-01, TC-02 | **P0 (Critical)** |
| BR-01, BR-02 | FR-02 | Zero-Auth Member Onboarding (Demographics, Life/Faith Stages) | TC-03 | **P0 (Critical)** |
| BR-03 | FR-03 | Seasonal Group Preferences & Format Ratio Intake | TC-04 | **P0 (Critical)** |
| BR-01, BR-04 | FR-04, FR-05 | Anonymous Prayer Intake + Edge Header Stripping + Local NER PII Scrubber | TC-05, TC-06 | **P0 (Critical)** |
| BR-06 | FR-07 | AI Week-by-Week Seasonal Plan Synthesis (Short & Long Term) | TC-07, TC-08 | **P0 (Critical)** |
| BR-07 | FR-08 | Leader Inline Plan Editor, Approval Lock & WhatsApp/Text Export | TC-09 | **P0 (Critical)** |
| BR-05 | FR-06 | Shared Group Prayer Board (Thematic Clusters + Redacted Cards) | TC-10, TC-11 | **P1 (MVP Complete)** |
| BR-04, BR-07 | FR-08 | Printable PDF Export & Single-Week AI Regeneration | TC-12 | **P2 (Fast-Follow)** |

### 10.2 Parallel Team Workstream Split (Ready for Developer Assignment)

1. **Workstream 1 — Privacy & Backend Core (FR-01, FR-04, FR-05):**
   - *Scope:* PostgreSQL zero-PII schema, reverse-proxy header stripping, FastAPI endpoints, Presidio NER + regex PII redaction pipeline, k-anonymity guardrails (N ≥ 3).
   - *Definition of Done:* Adversarial test suite with 100 planted names/phones/emails passes with 0 leaks in DB or logs.
2. **Workstream 2 — Member Mobile Intake & Prayer Board (FR-02, FR-03, FR-04, FR-06):**
   - *Scope:* Mobile-first 3-step intake wizard, client-side real-time privacy nudge pill, zero-cookie submission handling, and Group Prayer Board UI with theme clusters and *"I Prayed"* counters.
   - *Definition of Done:* Lighthouse mobile score ≥ 95; full 3-step submission completes in < 2 minutes on a 375px mobile viewport.
3. **Workstream 3 — AI Seasonal Plan Synthesis Engine (FR-06 clustering, FR-07):**
   - *Scope:* Cohort aggregation query builder, dual-horizon (short-term vs. long-term) prompt engineering, strict JSON Schema enforcement, and thematic prayer clustering.
   - *Definition of Done:* Generates schema-valid 4-to-12-week plans across 10 diverse synthetic group profiles in ≤ 15s p95.
4. **Workstream 4 — Leader Workspace, Inline Editor & Export (FR-01, FR-08):**
   - *Scope:* Magic-link leader auth, QR code generator, cohort insight cards, week-by-week inline plan editor, approval state lock, and one-tap WhatsApp/Markdown/PDF export.
   - *Definition of Done:* Leader can create a group, generate a plan, edit 2 weeks inline, approve, and paste formatted agenda into a messaging app seamlessly.

