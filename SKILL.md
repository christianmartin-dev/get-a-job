---
name: gaj
description: |
  Run your entire job search from the terminal. Investigates every recruiter and company
  before you respond, deduces unnamed clients from staffing firms, drafts calibrated
  responses, generates cover letters, and negotiates salary with market data. Triggers
  on /gaj, job pipeline, job search, add a job, respond to recruiter, cover letter,
  negotiate salary, triage inbox, pipeline status, or any job search management request.
---

When this skill loads, display this banner (output as text, do not run as bash):

```
   ██████╗  █████╗      ██╗
  ██╔════╝ ██╔══██╗     ██║
  ██║  ███╗███████║     ██║
  ██║   ██║██╔══██║██   ██║
  ╚██████╔╝██║  ██║╚█████╔╝
   ╚═════╝ ╚═╝  ╚═╝ ╚════╝

  Get A Job  v1.0.0
```

# GAJ - Job Search Pipeline

Manage your entire job search from Claude Code. Track opportunities, update statuses, search, and get stats.

## Setup check

Before any operation, verify the database exists:

```bash
ls ~/gaj/gaj.db
```

If the database does not exist, run the first-run setup:

1. Create the GAJ directory structure:
   ```bash
   mkdir -p ~/gaj/context
   ```

2. Set up the database:
   ```bash
   npx tsx scripts/setup-db.ts
   ```

3. Create `~/gaj/config.yaml` with default content:
   ```yaml
   # GAJ Configuration
   # Uncomment and fill in to enable Google Sheets sync:
   # sheets_id: "your-google-sheet-id"
   # credentials_path: "~/gaj/credentials.json"
   ```

4. Create `~/gaj/context/about-me.md` with this template:
   ```markdown
   # About Me

   ## Target roles

   ## Employment type preferences

   ## Compensation floors

   ## Preferred tech stack

   ## Evaluation criteria

   ## Location preferences

   ## Company size preference

   ## Key achievements

   ## Resume variants

   ## Contact
   ```

5. Begin guided profile creation:

   Present:
   > Your profile helps GAJ personalize everything: which jobs to pursue, what to say to recruiters, and how to negotiate.
   >
   > Got a resume handy? Drop it here (PDF, markdown, or paste text). Or type "skip" and I'll ask a few questions instead.

   **If resume provided:**
   - Extract: target roles, tech stack, key achievements
   - Show extracted profile: "Here's what I found. I'll fill this in and ask about the rest."
   - Write extracted fields to the about-me.md draft

   **If skipped:**
   - Proceed directly to the questions below

6. Ask what the resume can't tell us (one at a time, short answers):

   1. "What roles are you targeting?" (skip if extracted from resume)
   2. "Open to W-2 salaried, contract-to-hire, 1099, or all three?"
   3. "What's your salary floor?" (prompt per type based on answer to #2)
   4. "Remote only, hybrid, on-site, or flexible?"
   5. "Any company size or stage preference?"
   6. "What matters most to you beyond compensation? Pick your top 2-3: team quality, tech stack, product domain, growth path, mission, work-life balance, company reputation"
   7. "Do you use different resume versions for different roles? If so, where are they?" (only ask if resume was provided)
   8. "What email should outbound messages sign off with?"

7. Show the completed profile. Ask: "Anything to change? Otherwise I'll save this."
   Write to `~/gaj/context/about-me.md`.

8. Deploy command stubs for autocomplete:
   ```bash
   mkdir -p ~/.claude/commands/gaj && cp commands/*.md ~/.claude/commands/gaj/
   ```

9. Show the onboarding guide:

   ```
   Profile saved. Here's what you can do:

   Pipeline commands:
     /gaj:add              Add a job to the pipeline
     /gaj:list             List/filter pipeline items
     /gaj:status           Update a job's status
     /gaj:search           Search by company, title, or keyword
     /gaj:stats            Pipeline statistics

   Generation commands:
     /gaj:cover-letter     Generate a Hook/Proof/Close cover letter
     /gaj:respond          Assess interest and draft a recruiter response
     /gaj:negotiate        Ackerman-based salary negotiation guidance

   Workflow commands:
     /gaj:triage           Batch-filter a job alert digest
     /gaj:sync             Export pipeline to Google Sheets
     /gaj:profile          View or update your search profile

   Start here:
     1. /gaj:add to add your first job
     2. /gaj:help any time to see this reference again
   ```

Then stop. Do not attempt other pipeline operations until setup completes.

## Routing

When the user gives a natural language instruction, route to the correct sub-command:

| User says | Route |
|-----------|-------|
| "Add [company] [role]" or provides job details | `gaj:add` |
| "What's in my pipeline?" or "Show my jobs" | `gaj:list` |
| "Show [status] jobs" or "List pending jobs" | `gaj:list` |
| "Update [company] to [status]" or "Mark as [status]" | `gaj:status` |
| "Mark [company] as rejected/applied/approved" | `gaj:status` |
| "Find [company]" or "Search for [keyword]" | `gaj:search` |
| "How many jobs?" or "Pipeline stats" | `gaj:stats` |
| "Write a cover letter for [company]" or "Draft a letter" | `gaj:cover-letter` |
| "Reply to recruiter" or "Respond to [name]" | `gaj:respond` |
| "Negotiate salary" or "Counter offer for [company]" | `gaj:negotiate` |
| "Sync to sheets" or "Export pipeline" | `gaj:sync` |
| "Triage these jobs" or "Filter this digest" or pastes multiple jobs | `gaj:triage` |
| "Here's a LinkedIn alert" or "Batch these" or "Inbox buildup" | `gaj:triage` |
| "Update my profile" or "Change my targets" or "Edit about-me" | `gaj:profile` |

If the user invokes `/gaj` without a specific task, run `gaj:stats` and present a pipeline dashboard. After the dashboard, add: "Run /gaj:help for the full command reference."

## CLI tool

All database operations go through the pipeline CLI. Run from the GAJ repo directory:

```bash
npx tsx scripts/pipeline-cli.ts <command> [args]
```

The CLI outputs JSON. Never show raw JSON to the user. Always parse it and present as clean markdown tables.

## Job statuses

| Status | Meaning |
|--------|---------|
| `pending-review` | New, awaiting evaluation |
| `approved` | User approved, ready for cover letter |
| `cover-letter-ready` | Cover letter generated |
| `applied` | Application submitted |
| `interview` | Interview stage |
| `offer` | Offer received |
| `rejected` | Rejected by user or company |
| `expired` | Job listing no longer active |
| `filtered` | Filtered out during qualification |

## Output rules

- Present pipeline data as markdown tables with columns: ID, Company, Role, Status, Source, Added
- After mutations (add, update, status change), confirm what changed in one sentence
- When listing jobs, sort by most recent first
- For stats, present totals and breakdowns in a clean summary, not a table dump

## Voice rules

Apply these to all responses when this skill is active:

- No em dashes. Use commas or periods.
- No exclamation marks.
- No "I hope this helps" or "please don't hesitate" or "happy to help."
- No "excited," "thrilled," "delighted," or similar.
- Be direct. Name the company, the role, the number.
- Short responses. State what happened, offer the next logical action, stop.
