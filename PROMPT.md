**MINIONS HIRING — HIRING PIPELINE**

You are tasked with building **minions-hiring**, a structured recruiting and hiring pipeline system built on the Minions SDK. This is a structured approach to candidate tracking, interview management, and offer workflow designed for both recruiting teams and AI agents.

---

**PROJECT OVERVIEW**

**minions-hiring** provides structured candidate tracking, scorecard-based interviews, bias detection, and offer comparison. Unlike traditional ATS (Applicant Tracking Systems), every component of the hiring process (candidates, interviews, scorecards, offers) is a structured minion that can be queried, related, versioned, and manipulated by agents.

**Core value proposition:** Agents can autonomously manage hiring workflows, detect bias in evaluations, analyze pipeline velocity, and draft offers based on structured criteria.

**Positioning:** Structured recruiting meets data-driven hiring decisions.

---

**MINIONS SDK REFERENCE — REQUIRED DEPENDENCY**

This project depends on `minions-sdk`, a published package that provides the foundational primitives. The GH Agent building this project MUST install it from the public registries and use the APIs documented below — do NOT reimplement minions primitives from scratch.

**Installation:**
```bash
# TypeScript (npm)
npm install minions-sdk
# or: pnpm add minions-sdk

# Python (PyPI) — package name is minions-sdk, but you import as "minions"
pip install minions-sdk
```

**TypeScript SDK — Core Imports:**
```typescript
import {
  // Core types
  type Minion, type MinionType, type Relation,
  type FieldDefinition, type FieldValidation, type FieldType,
  type CreateMinionInput, type UpdateMinionInput, type CreateRelationInput,
  type MinionStatus, type MinionPriority, type RelationType,
  type ExecutionResult, type Executable,
  type ValidationError, type ValidationResult,

  // Validation
  validateField, validateFields,

  // Built-in Schemas (10 MinionType instances — reuse where applicable)
  noteType, linkType, fileType, contactType,
  agentType, teamType, thoughtType, promptTemplateType, testCaseType, taskType,
  builtinTypes,

  // Registry — stores and retrieves MinionTypes by id or slug
  TypeRegistry,

  // Relations — in-memory directed graph with traversal utilities
  RelationGraph,

  // Lifecycle — CRUD operations with validation
  createMinion, updateMinion, softDelete, hardDelete, restoreMinion,

  // Evolution — migrate minions when schemas change (preserves removed fields in _legacy)
  migrateMinion,

  // Utilities
  generateId, now, SPEC_VERSION,
} from 'minions-sdk';
```

**Python SDK — Core Imports:**
```python
from minions import (
    # Types
    Minion, MinionType, Relation, FieldDefinition, FieldValidation,
    CreateMinionInput, UpdateMinionInput, CreateRelationInput,
    ExecutionResult, Executable, ValidationError, ValidationResult,
    # Validation
    validate_field, validate_fields,
    # Built-in Schemas (10 types)
    note_type, link_type, file_type, contact_type,
    agent_type, team_type, thought_type, prompt_template_type,
    test_case_type, task_type, builtin_types,
    # Registry
    TypeRegistry,
    # Relations
    RelationGraph,
    # Lifecycle
    create_minion, update_minion, soft_delete, hard_delete, restore_minion,
    # Evolution
    migrate_minion,
    # Utilities
    generate_id, now, SPEC_VERSION,
)
```

**Key Concepts:**
- A `MinionType` defines a schema (list of `FieldDefinition`s) — each field has `name`, `type`, `label`, `required`, `defaultValue`, `options`, `validation`
- A `Minion` is an instance with `id`, `title`, `minionTypeId`, `fields` (dict), `status`, `tags`, timestamps
- A `Relation` is a typed directional link (12 types: `parent_of`, `depends_on`, `implements`, `relates_to`, `inspired_by`, `triggers`, `references`, `blocks`, `alternative_to`, `part_of`, `follows`, `integration_link`)
- Field types: `string`, `number`, `boolean`, `date`, `select`, `multi-select`, `url`, `email`, `textarea`, `tags`, `json`, `array`
- `TypeRegistry` auto-loads 10 built-in types; register custom types with `registry.register(myType)`
- `createMinion(input, type)` validates fields against the schema and returns `{ minion, validation }` (TS) or `(minion, validation)` tuple (Python)
- Both SDKs serialize to identical camelCase JSON; Python provides `to_dict()` / `from_dict()` for conversion

**IMPORTANT:** Do NOT recreate these primitives. Import them from `minions-sdk` (npm) / `minions` (PyPI). Build your domain-specific types and utilities ON TOP of the SDK.

---

**CORE PRIMITIVES**

The system is built from these minion types:

### `candidate`
A person applying for a role at the organization.

**Fields:**
- `name` (string, required) — candidate full name
- `email` (email, required) — primary contact email
- `phone` (string) — contact phone number
- `resumeUrl` (url) — link to resume/CV
- `linkedinUrl` (url) — LinkedIn profile
- `location` (string) — candidate location
- `yearsExperience` (number) — total years of professional experience
- `currentCompany` (string) — current employer
- `currentTitle` (string) — current job title
- `referredBy` (string) — referrer name or source
- `tags` (tags) — skills, technologies, attributes
- `notes` (textarea) — general notes about candidate
- `status` (select: sourced/screening/interviewing/offer/hired/rejected/withdrawn) — pipeline stage

**Relations:**
- `references` → `job-posting` (which role they're applying for)
- `parent_of` → `application` minions (applications to different roles)
- `parent_of` → `interview` minions (interview sessions)

---

### `application`
A candidate's application to a specific job posting.

**Fields:**
- `appliedAt` (date, required) — when they applied
- `source` (select: direct/referral/linkedin/indeed/agency/other) — application source
- `coverLetter` (textarea) — cover letter content
- `screeningScore` (number) — initial resume screening score (0-100)
- `status` (select: new/reviewing/passed-screen/rejected, required)
- `rejectionReason` (textarea) — if rejected, why

**Relations:**
- `part_of` → `candidate`
- `references` → `job-posting`

---

### `interview`
A scheduled or completed interview session.

**Fields:**
- `type` (select: phone-screen/technical/behavioral/panel/final/other, required) — interview type
- `scheduledAt` (date, required) — when the interview is/was scheduled
- `durationMinutes` (number) — planned or actual duration
- `interviewer` (string, required) — interviewer name or ID
- `location` (string) — physical location or video call link
- `status` (select: scheduled/completed/cancelled/no-show, required)
- `notes` (textarea) — interviewer notes
- `recordingUrl` (url) — link to recording if available
- `completedAt` (date) — when the interview was completed

**Relations:**
- `part_of` → `candidate`
- `references` → `job-posting`
- `parent_of` → `scorecard` minions (evaluation scorecards)

---

### `scorecard`
Structured evaluation criteria for an interview.

**Fields:**
- `dimensions` (json, required) — scoring dimensions with scores (e.g., `{ "technical": 8, "communication": 9, "culture-fit": 7 }`)
- `strengths` (textarea) — candidate strengths observed
- `concerns` (textarea) — areas of concern or weakness
- `recommendation` (select: strong-yes/yes/maybe/no/strong-no, required) — hire recommendation
- `confidence` (select: high/medium/low, required) — evaluator confidence in assessment
- `biasFlags` (tags) — detected potential biases (e.g., "affinity", "halo-effect")
- `submittedBy` (string, required) — who filled the scorecard
- `submittedAt` (date, required) — when submitted

**Relations:**
- `part_of` → `interview`
- `follows` → `scorecard-template` (which template was used)

---

### `scorecard-template`
Reusable evaluation framework for a specific role or interview type.

**Fields:**
- `title` (string, required) — template name (e.g., "Senior Engineer Technical Interview")
- `description` (textarea) — what this template evaluates
- `dimensions` (json, required) — evaluation dimensions with weights (e.g., `{ "technical": 40, "problem-solving": 30, "communication": 30 }`)
- `questionBankIds` (array) — references to question minions from `minions-skills`
- `passingScore` (number) — minimum score to pass (0-100)

**Relations:**
- `references` → `job-posting` (which role this template is for)

---

### `offer`
A formal job offer extended to a candidate.

**Fields:**
- `title` (string, required) — job title being offered
- `salary` (number, required) — base salary
- `currency` (string) — currency code (e.g., "USD")
- `equity` (string) — equity/options details
- `bonusStructure` (string) — bonus or commission structure
- `startDate` (date) — proposed start date
- `location` (string) — work location
- `remotePolicy` (select: on-site/hybrid/remote) — remote work policy
- `benefits` (textarea) — benefits summary
- `offerLetterUrl` (url) — link to formal offer letter
- `expiresAt` (date) — offer expiration date
- `status` (select: drafted/sent/accepted/rejected/expired, required)
- `sentAt` (date) — when offer was sent
- `respondedAt` (date) — when candidate responded

**Relations:**
- `references` → `candidate`
- `references` → `job-posting`

---

### `job-posting`
A role the organization is hiring for.

**Fields:**
- `title` (string, required) — job title
- `description` (textarea, required) — full job description
- `department` (string) — department or team
- `location` (string) — job location
- `remotePolicy` (select: on-site/hybrid/remote)
- `salaryMin` (number) — minimum salary range
- `salaryMax` (number) — maximum salary range
- `currency` (string) — currency code
- `requirements` (textarea) — required qualifications
- `niceToHave` (textarea) — preferred qualifications
- `status` (select: draft/open/paused/closed, required)
- `openedAt` (date) — when the role opened
- `closedAt` (date) — when the role closed
- `hiringManagerId` (string) — hiring manager reference

**Relations:**
- `parent_of` → `scorecard-template` minions (evaluation templates for this role)

---

**BEYOND THE STANDARD PATTERN**

### 1. Scorecard Templates & Question Banks

Unlike traditional ATS, interview evaluations are structured and reusable.

**Features:**
- Define scoring dimensions with weights (e.g., technical 40%, communication 30%)
- Link to question banks from `minions-skills` for consistency
- Calculate composite scores automatically
- Compare candidate scores across the same template
- Track which interviewers are consistently harsh or lenient

**Implementation:**
```typescript
class ScorecardAnalyzer {
  calculateCompositeScore(scorecard: Minion, template: Minion): number;
  compareScores(candidate1: Minion, candidate2: Minion, template: Minion): ComparisonResult;
  detectInterviewerBias(interviewerId: string): BiasReport;
}
```

---

### 2. Bias Detection

Automatically flag potential biases in scorecards.

**Detection heuristics:**
- **Affinity bias:** Similar backgrounds/schools to interviewer
- **Halo effect:** One strong dimension influences others
- **Recency bias:** Recent candidates scored higher
- **Contrast effect:** Scored relative to previous candidate

**Implementation:**
```typescript
class BiasDetector {
  analyzeScorecard(scorecard: Minion, candidate: Minion, interviewer: string): string[];
  flagPatterns(scorecards: Minion[]): BiasReport;
}
```

**CLI Integration:**
```bash
hiring bias-check <scorecard-id>
hiring bias-report --interviewer alice@company.com
hiring bias-report --role "Senior Engineer"
```

---

### 3. Offer Comparison

Compare multiple offers side-by-side or benchmark against market data.

**Features:**
- Total compensation calculation (salary + equity + bonus)
- Side-by-side comparison table
- Market data integration (if available)
- Equity value estimation

**Implementation:**
```typescript
class OfferComparator {
  compare(offers: Minion[]): ComparisonTable;
  calculateTotalComp(offer: Minion): number;
  benchmarkAgainstMarket(offer: Minion): MarketComparison;
}
```

**CLI Integration:**
```bash
hiring offer compare <offer-id-1> <offer-id-2> <offer-id-3>
hiring offer draft <candidate-id> --benchmark
```

---

### 4. Pipeline Velocity Metrics

Track how long candidates spend in each stage and where bottlenecks occur.

**Metrics:**
- Average time from application to offer
- Average time per pipeline stage
- Conversion rates between stages
- Bottleneck detection (where candidates stall)
- Interviewer response time

**Implementation:**
```typescript
class PipelineAnalyzer {
  calculateVelocity(candidates: Minion[]): VelocityReport;
  stageConversionRates(jobPostingId: string): ConversionReport;
  identifyBottlenecks(): Bottleneck[];
  interviewerResponseTimes(): Map<string, number>;
}
```

**CLI Integration:**
```bash
hiring pipeline
hiring pipeline velocity --role "Senior Engineer"
hiring pipeline bottlenecks
hiring funnel --role <job-id>
```

---

### 5. Interview Question Bank Integration

Link interviews to question sets from `minions-skills` for consistency and reusability.

**Features:**
- Reusable question libraries tagged by skill
- Track which questions have high signal (correlate with successful hires)
- Prevent question repetition across interview rounds
- Question effectiveness scoring

---

**DUAL SDK SUPPORT (TYPESCRIPT + PYTHON)**

Both SDKs provide identical functionality:

**TypeScript:**
```typescript
import { CandidateBuilder, ScorecardAnalyzer, OfferComparator } from 'minions-hiring';

const candidate = new CandidateBuilder()
  .withName('Alice Johnson')
  .withEmail('alice@example.com')
  .withStatus('interviewing')
  .build();

const analyzer = new ScorecardAnalyzer();
const score = analyzer.calculateCompositeScore(scorecard, template);
```

**Python:**
```python
from minions_hiring import CandidateBuilder, ScorecardAnalyzer, OfferComparator

candidate = (CandidateBuilder()
    .with_name('Alice Johnson')
    .with_email('alice@example.com')
    .with_status('interviewing')
    .build())

analyzer = ScorecardAnalyzer()
score = analyzer.calculate_composite_score(scorecard, template)
```

---

**CLI COMMANDS**

The `hiring` CLI extends the base `minions` CLI with recruiting-specific commands.

### Core Commands

```bash
# Candidate management
hiring candidate add "Alice Johnson" --email alice@example.com --role <job-id>
hiring candidate show <candidate-id>
hiring candidate update <candidate-id> --status interviewing

# Pipeline view
hiring pipeline
hiring pipeline --role "Senior Engineer"
hiring pipeline velocity --role <job-id>
hiring funnel --role <job-id>

# Interview scheduling
hiring interview schedule <candidate-id> --type technical --interviewer bob@company.com --date "2026-03-15 14:00"
hiring interview list --upcoming
hiring next-interviews

# Scorecard management
hiring scorecard fill <interview-id>
hiring scorecard show <scorecard-id>
hiring scorecard compare <candidate-id> --template <template-id>

# Bias detection
hiring bias-check <scorecard-id>
hiring bias-report --interviewer alice@company.com
hiring bias-report --role "Senior Engineer"

# Offer management
hiring offer draft <candidate-id>
hiring offer compare <offer-id-1> <offer-id-2>
hiring offer send <offer-id>
hiring offer status <offer-id>

# Job postings
hiring job create "Senior Engineer"
hiring job list
hiring job close <job-id>

# Reporting
hiring report --funnel
hiring report --velocity
hiring candidate-summary <candidate-id>
```

---

**DOCUMENTATION SITE**

Built with **Astro Starlight** with dual-language SDK examples.

**Site structure:**
```
docs/
├── index.md                   # Landing page
├── getting-started.md         # Quick start guide
├── core-concepts/
│   ├── candidates.md
│   ├── interviews.md
│   ├── scorecards.md
│   ├── offers.md
│   └── job-postings.md
├── guides/
│   ├── building-scorecards.md
│   ├── bias-detection.md
│   ├── pipeline-optimization.md
│   ├── offer-workflows.md
│   └── reporting.md
├── api-reference/
│   ├── typescript.md          # TypeScript SDK
│   └── python.md              # Python SDK
├── cli-reference.md
└── examples/
    ├── structured-interview.md
    ├── bias-analysis.md
    └── agent-recruiting.md
```

**Dual-language code tabs:**
Every code example includes both TypeScript and Python tabs.

---

**AGENT USE CASES**

**Autonomous Pipeline Management:**
An agent monitors the hiring pipeline, detects stalled candidates, and sends reminders to interviewers.

**Bias Analysis:**
An agent reviews all scorecards for a role, flags potential biases, and generates a fairness report.

**Offer Drafting:**
An agent generates offer letters based on scorecard results, internal compensation bands, and market benchmarks.

**Interview Scheduling:**
An agent coordinates calendars, schedules interviews, and sends calendar invites automatically.

**Pipeline Reporting:**
An agent generates weekly hiring metrics reports with velocity trends and bottleneck identification.

---

**PROJECT STRUCTURE**

```
minions-hiring/
├── packages/
│   ├── core/
│   │   ├── src/
│   │   │   ├── types/              # Candidate, Interview, Scorecard, etc.
│   │   │   ├── builders/           # CandidateBuilder, OfferBuilder
│   │   │   ├── analyzers/          # ScorecardAnalyzer, PipelineAnalyzer
│   │   │   ├── bias/               # BiasDetector
│   │   │   ├── comparators/        # OfferComparator
│   │   │   └── index.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── python/
│   │   ├── minions_hiring/
│   │   │   ├── __init__.py
│   │   │   ├── types.py
│   │   │   ├── builders.py
│   │   │   ├── analyzers.py
│   │   │   ├── bias.py
│   │   │   └── comparators.py
│   │   ├── tests/
│   │   └── pyproject.toml
│   └── cli/
│       ├── src/
│       │   ├── commands/
│       │   │   ├── candidate.ts
│       │   │   ├── pipeline.ts
│       │   │   ├── interview.ts
│       │   │   ├── scorecard.ts
│       │   │   ├── offer.ts
│       │   │   ├── bias.ts
│       │   │   └── report.ts
│       │   └── index.ts
│       └── package.json
├── apps/
│   ├── docs/                       # Astro Starlight
│   └── playground/                 # Optional web UI
├── spec/
│   └── v0.1.md
├── examples/
│   ├── structured-interview/
│   ├── bias-detection/
│   └── agent-recruiting/
├── README.md
├── CHANGELOG.md
└── package.json
```

---

**TONE & POSITIONING**

**minions-hiring** is a structured recruiting system designed for data-driven hiring decisions and agent-assisted workflows. The messaging should emphasize:

- **Structured evaluations:** Scorecards bring consistency and comparability to hiring decisions
- **Bias detection:** Built-in fairness analysis helps reduce unconscious bias
- **Pipeline visibility:** Real-time metrics show where candidates are stuck
- **Agent-native:** Agents can manage workflows, draft offers, and analyze patterns
- **Beyond ATS:** Not just tracking applicants, but optimizing hiring decisions

The docs should speak to hiring managers, recruiters, and HR teams building data-driven hiring processes.

---

**SUCCESS CRITERIA**

You will know this implementation is successful when:

1. A recruiter can track a candidate from application to offer using structured scorecards
2. An agent can detect bias patterns across 100+ interview scorecards and flag concerns
3. A hiring manager can compare offers side-by-side with total comp calculations
4. The documentation site clearly explains the scorecard template system with visual examples
5. Both TypeScript and Python SDKs provide identical functionality with idiomatic APIs
6. Pipeline velocity metrics surface bottlenecks and optimize hiring speed

---

**Start with the spec, then core types and builders, then analyzers and bias detection, then the CLI, then the docs, then the Python SDK, then the examples. Work systematically. Every file should be production quality — not stubs, not placeholders.**
