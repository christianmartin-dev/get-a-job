---
name: gaj:sherlock
description: |
  Deep investigative research on a company, recruiter, or job listing. Triggers on
  /gaj:sherlock, "research this company", "investigate this recruiter", "who is this
  firm", "what do we know about [company]", or any standalone research request before
  deciding to respond or apply.
---

# gaj:sherlock - Deep Investigation

Run parallel investigative research on a company, recruiter, staffing firm, or job
listing. Returns a structured detective report for internal use only.

## When to use

- Before responding to a recruiter you haven't researched
- When a staffing firm won't name their client — infer them
- When a company name appears in the digest and you want intel before committing time
- When a job listing has red flags worth investigating
- As a standalone step before running /gaj:respond

## Process

### Step 1: Parse the target

The user will provide one of:
- A company name ("Fitch Ratings")
- A recruiter or firm name ("Burtch Works")
- A job ID from the pipeline (e.g. `fitch-ratings-1775584530503`)
- A job title + company combo ("Senior AI Engineer at Accenture")
- A raw job listing pasted into the prompt

If a job ID is provided, look it up first:

```bash
npx tsx scripts/pipeline-cli.ts search '<company>' 2>&1
```

Extract: company name, job title, source, any existing research_notes.

### Step 2: Determine investigation type

| Target type | Research threads to run |
|-------------|------------------------|
| Named company (direct listing) | Company, role, compensation |
| Staffing firm (recruiter) | Firm reputation, client inference, comp range |
| Individual recruiter | Person's tenure/credibility, their firm, clients |
| Unnamed client via staffing firm | JD forensics to identify likely client |

### Step 3: Launch parallel research threads

Read `~/gaj/context/about-me.md` for the user's salary floor, tech stack, and location
preferences before assessing fit.

**Always run these three threads in parallel via subagents:**

#### Thread 1: The source (company or recruiter firm)

If a named company:
- Ownership, size, funding stage, recent news
- AI/tech investment signals: partnerships, product launches, job posting trends
- Engineering culture: Glassdoor signal, tech stack evidence, remote policy
- Office locations vs. user's location preferences
- Any red flags: layoffs, PE squeeze, revolving door leadership

If a staffing firm or recruiter:
- Is this a body shop, boutique, or executive search?
- Specialization: do they actually place AI/ML talent or are they generalists?
- Reputation: Glassdoor (internal), candidate testimonials
- Geographic footprint and known client industries
- Check against blacklist in `~/gaj/gaj.json`

#### Thread 2: The role and likely client

If the company is named:
- Find the specific job posting if publicly available
- Extract tech stack, team structure, reporting line, product vs. internal tooling
- Is the "AI" work real (RAG, agents, LLM pipelines) or theater (prompt wrapping)?
- Seniority and scope match vs. user's target level

If the client is unnamed (staffing firm):
- Cross-reference JD clues: industry keywords, stack, location, company size descriptors,
  compliance/regulatory language, domain vocabulary
- Generate a ranked list of 3-5 likely clients with supporting evidence
- State confidence level for each candidate

#### Thread 3: Compensation and market fit

- Stated comp vs. user's floor (read from `~/gaj/gaj.json` compensation.w2_salaried.filter_floor; currently $120K/yr entry-level, absolute floor $100K, W-2 salaried)
- Market benchmarks: Glassdoor, Levels.fyi, H1B LCA filings, Robert Half, Burtch Works
  annual report data if applicable
- For staffing firm roles: confirm perm vs. contract; if contract, note W-2 vs. 1099
  implications (never conflate rates)
- Stack alignment: does the tech stack match the user's TypeScript/React/Node/AI profile?

### Step 4: Present the detective report

Format the report as:

```
## Detective Report: [Target] — [Role or context]

### 1. [Company/Firm] overview
[Ownership, size, stage, red flags]

### 2. AI/tech investment signals
[Evidence of real work vs. theater]

### 3. Role assessment
[What the job actually is, tech stack confirmed, product vs. internal]

### 4. Client inference (if unnamed)
[Ranked candidates with evidence and confidence]

### 5. Compensation analysis
| Source | Role | Range |
...
Does this likely clear the W-2 filter_floor in gaj.json ($120K entry-level)? Y/N + reasoning.

### 6. Remote/location viability
[Remote, hybrid, on-site — vs. user's preferences]

### 7. Verdict
PURSUE / INVESTIGATE / PASS — one line rationale
[3-5 bullet points: what you'd want confirmed before proceeding]
```

**CRITICAL: The detective report is internal intel only.** Never surface research
findings in any outbound communication. The edge comes from knowing more than the
recruiter expects. If asked to draft a follow-up response after Sherlock, apply the
intelligence asymmetry: ask open questions that extract what you already suspect,
don't signal what you know.

### Step 5: Save research to pipeline

If a matching pipeline entry exists, store the report summary:

```bash
npx tsx scripts/pipeline-cli.ts update '<job-id>' research_notes '<summary>' 2>&1
```

Keep the `research_notes` field under 500 characters (CLI limit). Store the verdict,
key flags, and 2-3 qualifiers needed.

If no pipeline entry exists, offer to create one:

```bash
npx tsx scripts/pipeline-cli.ts add '{"company_name":"<company>","job_title":"<role>","status":"pending-review","source":"sherlock"}' 2>&1
```

### Step 6: Recommend next action

After the report, suggest one next step:

- If PURSUE: "Run /gaj:respond to draft an outreach message"
- If INVESTIGATE: list the 2-3 qualifying questions to ask before committing time
- If PASS: offer to mark the pipeline entry as `filtered` with a reason

## CLI reference

```bash
npx tsx scripts/pipeline-cli.ts search '<company>'
npx tsx scripts/pipeline-cli.ts update '<job-id>' research_notes '<summary>'
npx tsx scripts/pipeline-cli.ts add '<json>'
npx tsx scripts/pipeline-cli.ts status '<job-id>' filtered
```

## Blacklist check

Before running any research, scan the company or firm name against the blacklist in
`~/gaj/gaj.json` (under `blacklist.auto_skip`, `blacklist.deprioritize`,
`blacklist.companies`). If matched, report the blacklist reason immediately and skip
research — do not invest compute on a known skip.
