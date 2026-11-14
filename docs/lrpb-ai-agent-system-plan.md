# LRPB AI Agent System: Evaluation, Architecture and Implementation Plan

Prepared 2026-11-14 for Little Rock Property Buyers (LRPB) / Journey Estate Investments.
Scope: build a central AI operations manager with specialist sub-agents using only the tools and subscriptions LRPB already pays for.

Tracking: Asana project **LRPB AI Agent System** (Journey Estate Investments team), https://app.asana.com/1/1147787984668669/project/1220536824635018. Every checklist item in section 7 and every open question in section 9 is a task there.

---

## 1. Executive summary

**The system you described already exists in an early form, and it works.** Two days ago four n8n Agents were created and published on the LRPB n8n instance:

| Agent | Role | Model | Status |
|---|---|---|---|
| LRPB Assistant | Master / orchestrator, connected to Slack | Claude Opus 5.5 via n8n AI Gateway (managed credit) | Published, live in Slack |
| LRPB Email Agent | Searches Admin, Manager and Shahin inboxes, saves drafts | GPT-6.1 (OpenAI key) | Published |
| LRPB Documents Agent | Searches Drive, reads Docs, Sheets, PDFs | GPT-6.1 (OpenAI key) | Published |
| LRPB Lead and CRM Agent | Searches Salesmate contacts and deals, adds notes | GPT-6.1 (OpenAI key) | Published |

Real Slack sessions from Nov 13 show the master agent correctly breaking tasks down, delegating, and synthesizing. Example: "what i need to do as urgent task" produced a prioritized list from Asana, then an inbox sweep with five time-sensitive title items, each cited by subject and date. That is exactly the operations manager behaviour you asked for.

**Recommendation:** do not start over and do not add a new platform. Keep n8n Agents as the backbone, extend it with the missing specialists (KPI/Reporting, TC/Dispo, Admin/Ops, Sales), give the agents a shared knowledge base, add scheduled tasks and a quality-control step, and use the Claude Max subscription as the heavy analysis and build layer. Everything in this plan runs on tools already paid for. The one cost item to watch is pay-per-use OpenAI API spend by the sub-agents (section 4).

---

## 2. What we have today (inventory)

### 2.1 Platforms and what each can do for agents

| Platform | Current plan | What it gives the agent system | Verified in this review |
|---|---|---|---|
| **n8n Cloud** (lrpbai.app.n8n.cloud), $80/mo | 38 workflows, 4 Agents, 63 credentials | First-class Agents with sub-agents, tools, skills (knowledge), scheduled tasks, memory, Slack channel integration, approval gates, budget guardrails, MCP servers. AI Gateway with managed model credits. | Yes. Agent config and transcripts inspected. |
| **Claude** (Max, $100/mo) | Connectors: Slack, Gmail, Drive, Asana, n8n, Zapier, Zoom, Canva. DocuSign needs reconnect. | Deep analysis, report writing, building and maintaining n8n agents and workflows, scheduled tasks, the `lrpb-assistant` knowledge skill (company handbook, SOPs, contracts, contacts). | Yes. This session used all of them. |
| **Zapier**, $79/mo | Connected apps: Salesmate (49 actions, 2 accounts), CallRail, Gmail, Slack, Asana, Google Sheets, DocuSign (2 accounts), DealMachine, Gravity Forms, Skool, SMS, WhatsApp, AI by Zapier | The only place with native Salesmate triggers (New Deal, Update Deal, New Contact, New Activity) and CallRail call events. Good as the event source feeding n8n and Sheets. | Yes. No durable Zapier workflows are currently active through the connector; classic Zaps could not be listed from here. |
| **Slack** (jeiinvestments) | Property channels per address, #contracts, #title-desk, #weekly-kpi-summary, #kpi-workflows-notes, #ai-agent, #ai | The command center. Master agent already answers in DMs and channels. | Yes |
| **Google Workspace** | Drive (LRPB 2.0 shared drive), Sheets, Gmail (Admin, Manager, Shahin, Home Depot, Title inboxes all credentialed in n8n) | All business documents and the KPI sheets. | Yes |
| **Asana** | 25 projects, incl. Deals Active, TC Active Files, DISPO, LRPB 3.0, Property Maintenance Requests | Deal pipeline and task management. Two Asana credentials in n8n. | Yes |
| **Salesmate** | API token in n8n (two header-auth credentials), Zapier connection | CRM reads and writes. No Claude connector, so n8n and Zapier are the access paths. | Yes |
| **CallRail** | Zapier connection (Cody's account) | Call volume, SMS, call transcripts for KPI and sales review. | Zapier actions listed |
| **DocuSign** | Zapier connection (83 actions); Claude connector needs reconnect | Envelope status, sending for signature (later phase, approval-gated). | Zapier only |
| **Twilio, Airtable, Supabase, Postgres** | Credentials exist in n8n | Used by Sara SMS agent and older workflows. Postgres/Supabase can hold agent state if needed. | Credentials seen, not tested |

### 2.2 Existing automations that matter for this project

| Workflow | State | Relevance |
|---|---|---|
| LRPB Assistant Tool: Read Drive File / Add Salesmate Note | Published | Tool workflows the sub-agents call. Pattern to reuse for new tools. |
| LRPB - Clauhin Title Command V3 (64 nodes) | Active | Already does title-email classification into property channels, @Clauhin follow-up drafting with approve/decline, and a weekday morning brief driven by Asana board columns. This is most of a TC agent already. |
| LRPB Phase 1 - KPI Automation #6 | Inactive | Salesmate webhook → Google Sheet tabs (Leads, Property Visits, Visited, Contract Sent, Signed, Closed) → Monday rollup to #weekly-kpi-summary → monthly ad-spend upload form. The data pipeline the KPI agent needs; needs reactivation and validation. |
| Slack AI Agent - 2.0 | Active | Generic GPT-4o chatbot on app mentions. Overlaps with LRPB Assistant; candidate to retire. |
| Follow-Up SMS Agent #1, zillow-Agent, Tenant Line v2 | Active | Sara SMS follow-up and rental operations. Leave as is; expose their sheets to the Sales agent read-only. |
| Zillow Leads → Property Channels, Photos → Drive, Home Depot invoices, Insurance EOI, DocuSign alerts | Active | Event feeds into Slack that agents can read as context. |

### 2.3 Data sources the agents will read

- **Master LRPB KPI's - 2.0** (Sheets): tabs Weekly Progress (New Leads, SM Calls, Scheduled Visits, Visits, Contracts Sent/Signed, Closed, GP Contracted, GP Closed, Batting Avg, Lead to Appt), Business Tracker, Source Tracker (cost per channel, ROI), Deal Dashboard (per-deal revenue, source, buyer, lender, days), Sales guy Tracker, Calls Tracker (CallRail backend).
- **LRPB Phase 1 - KPI Automation** (Sheets): event log tabs fed by Salesmate stage changes.
- **Asana Deals Active** board: stage per deal (board column), attachments, closing dates.
- **Salesmate**: contacts, deals, stages, activities (calls), notes, tags such as Current Tenants.
- **Gmail**: title company threads (Proland, American Abstract), lender, insurance, tenant and seller email.
- **Slack property channels**: the running narrative of each deal.
- **Company knowledge**: the `lrpb-assistant` skill (14 reference files: lenders, buyers, title companies, contractors, contracts, TC/dispo, sales scripts, construction, mailers, Sara, avatars, properties, marketing, SOPs). Today this knowledge only exists on the Claude side, not in n8n.

---

## 3. What is missing today (gap analysis of the current n8n build)

1. **No KPI/Reporting, TC/Dispo, Admin/Ops or Sales specialists.** The master only has three Asana tools of its own, and the KPI question on Nov 13 was answered by reading a sheet that was blank for the current week because the data pipeline (KPI Automation #6) is switched off.
2. **No shared knowledge base.** The agents do not know the team, entities, channel conventions, SOPs, KPI definitions or stage names beyond a short paragraph in the master's instructions. The Claude skill has all of this; it needs to be ported into n8n Agent skills.
3. **Sub-agents cannot see Slack.** Property channels are the richest deal context. Only the master gets a small Slack context tool. A Slack read tool (channel history, search) is needed on the master and the TC agent.
4. **CRM reach is shallow.** Contact/deal search and notes only. No activities (call counts), no "deals by stage and date range", no pipeline listing. KPI and Sales work both need these.
5. **Asana reach is shallow.** No subtasks, sections, project listing, due-date filters, or task updates. The agent itself said "my tools can't list subtasks".
6. **Gmail search caps at 10 results per query** and drafts are not threaded replies.
7. **No scheduled work.** Agents only answer when asked. n8n Agents support tasks (cron), so morning briefs and Monday reports can be proactive.
8. **No quality-control step.** The master is told to verify, but nothing checks a report against its sources before it is posted.
9. **Two chatbots in Slack.** Slack AI Agent 2.0 (GPT-4o, generic) and LRPB Assistant both respond to mentions. Confusing and double cost.
10. **Model spend is split.** Master and memory run on the n8n AI Gateway (plan credits, capped at $25/mo). The three sub-agents and the older workflows run on the OpenAI API key, which is pay-per-use outside any subscription.

---

## 4. Cost position (no new spend)

| Item | Already paid | Note |
|---|---|---|
| n8n Cloud | Yes, $80/mo | Agents, workflows, AI Gateway credit allowance. Confirm the monthly credit included in the plan in n8n Settings → Usage. |
| Claude Max | Yes, $100/mo | Connectors, Claude Code, scheduled tasks, skills. No API cost. |
| Zapier | Yes, $79/mo | Task quota applies; use it for triggers, not for bulk reads. |
| OpenAI API key in n8n | **Pay-per-use** | Used by 3 sub-agents (GPT-6.1), Slack AI Agent 2.0 (GPT-4o), Title Command (classification + drafting). Not a subscription. Decide: move sub-agents to the n8n AI Gateway managed credential with the existing $10/mo guardrails, or keep the key and cap it. Either way nothing new is bought. |
| Anthropic API key in n8n ("Anthropic - apartments") | Pay-per-use | Exists; same rule as above. |

Budget guardrails already configured: master $25/mo with $2 per session cap; each sub-agent $10/mo. Keep these and add the same to every new agent.

---

## 5. Recommended architecture

### 5.1 Layers

```
                     ┌──────────────────────────────────────────┐
  Slack (humans)     │  #ai-command-center, DMs, property chans │
                     └───────────────┬──────────────────────────┘
                                     │ mention / DM / scheduled task
                     ┌───────────────▼──────────────────────────┐
  Orchestration      │  LRPB Assistant (master, n8n Agent)      │
  (n8n Agents)       │  plan → delegate → collect → verify →    │
                     │  report. Memory on. Knowledge skills on. │
                     └──┬──────┬──────┬──────┬──────┬──────┬────┘
                        │      │      │      │      │      │
                 ┌──────▼┐ ┌───▼───┐ ┌▼────┐ ┌▼────┐ ┌▼─────┐ ┌▼──────┐
  Specialists    │ Email │ │ Docs/ │ │ CRM/│ │ KPI │ │ TC / │ │ Admin │
  (n8n Agents)   │       │ │ Data  │ │Sales│ │ Rep.│ │ Dispo│ │ / Ops │
                 └──┬────┘ └──┬────┘ └┬────┘ └┬────┘ └┬─────┘ └┬──────┘
                    │         │       │       │       │        │
  Tools            Gmail    Drive   Salesmate Sheets  Asana    Asana
  (n8n nodes,      nodes    HTTP    HTTP API  nodes   Slack    nodes
   tool workflows)          + Read  + Zapier  CallRail Gmail   Slack
                            Drive   triggers  (Zapier) Title   Sheets
                            File             Cmd V3
                                                       state
  Event feeds      Zapier: Salesmate New/Updated Deal, New Contact, CallRail call completed
  (data pipeline)  → n8n webhooks → KPI event sheet / n8n data tables → Slack notices

  Build & analysis Claude Max (Claude Code + connectors): builds and maintains the agents,
  layer            runs deep monthly analyses, keeps the knowledge base in sync.
```

### 5.2 Why n8n Agents as the backbone

- Already built, published and working in Slack with the team's credentials.
- Native parent/child delegation with structured hand-offs (goal, context, expected output, difficulty), parallel children, and the option for a child to have its own children.
- Skills (persisted knowledge), tasks (cron schedules), observational and episodic memory, approval gates on tools, per-agent budgets, and a managed model credential all exist in the platform, so no custom code or new hosting.
- Deterministic steps stay as workflows and are attached as tools (the "Read Drive File" pattern), which keeps the agents reliable.

Alternatives considered and rejected for now: building on the Claude Agent SDK or another framework (needs API spend and hosting), Zapier Agents (weaker orchestration, task quota), a code-based Slack bot in this repo (new hosting, duplicates what n8n provides). Claude Max is kept as the builder and analyst, not as the always-on runtime, because Claude scheduled tasks are best for a few heavy jobs rather than dozens of Slack interactions a day.

### 5.3 How the agents communicate

1. A team member writes to the master in Slack (DM, @mention in a property channel, or #ai-command-center). Scheduled tasks start the same way without a human.
2. The master reads the Slack context (channel name = property), consults its knowledge skills, and writes a plan: which specialists, what each must return, which parts run in parallel.
3. Each delegation is a self-contained brief. The specialist runs its tools and returns a structured answer with sources (file link, email subject and date, Salesmate record id, Asana link). Specialists never see the Slack conversation and never talk to the human directly.
4. A specialist may delegate further (KPI agent asks the Data agent to read a sheet tab; TC agent asks the Email agent for the latest title thread).
5. The master cross-checks: numbers from two sources must agree or the discrepancy is reported; blanks are reported as blanks, never as zero.
6. For reports, the master calls the QC reviewer (section 5.5) before posting.
7. Writes are tiered: reads are free; internal reversible writes (draft, note, Asana task) happen when asked; external or irreversible actions (send email, send DocuSign, change deal stage, text a seller) require an approval button in Slack.
8. The master posts one answer in the thread, leads with the outcome, lists sources, and states what failed or is missing.

### 5.4 Agent roster and responsibilities

| Agent | Status | Responsibilities | Tools (existing credentials) |
|---|---|---|---|
| **Master (LRPB Assistant)** | Exists | Intake, planning, delegation, verification, reporting, memory of decisions. | Slack read (history, search), Asana search/get/create, QC reviewer, all sub-agents |
| **Email Agent** | Exists, extend | Triage, find threads, action items, draft replies, summaries to Slack via master. | Gmail (3 inboxes): add thread read, label filters, higher result limits, threaded reply drafts |
| **Data / Documents Agent** | Exists, extend | Find and read Drive files; read any sheet tab; controlled writes to named sheets (KPI event sheet, Master Rentals Active) behind approval. | Drive search, Read Drive File workflow, new "Append/Update Sheet Row" tool workflow (allow-listed sheets only) |
| **CRM / Sales Agent** | Exists, extend | Lead and pipeline review, deals by stage and date, activities and call counts per rep, follow-up candidates, Not-Yet-Reached list, Sara queue status. | Salesmate HTTP (contacts, deals, activities, pipelines), CallRail via Zapier read actions, Sara follow-up sheet (read) |
| **KPI / Reporting Agent** | New | Weekly and monthly KPI roll-ups, lead source performance, conversion rates, cost per deal, trend and missing-data flags. | Master LRPB KPI's 2.0 (read), KPI Automation event sheet (read), Salesmate deal/activity queries, CallRail counts, Source Tracker; delegates to Data and CRM agents |
| **TC / Dispo Agent** | New (wraps Title Command V3) | Per-deal status, deadlines, missing documents, title follow-ups, closing calendar, dispo buyer activity. | Asana Deals Active / TC Active Files / DISPO boards, Slack property channel history, Gmail title threads, Title Command state, DocuSign status via Zapier (read) |
| **Admin / Ops Agent** | New | Asana task creation and updates, subtasks, due dates, project status, maintenance request intake (Slack → Asana card), rental sheet updates, reminders. | Asana (tasks, subtasks, sections, projects, comments), Slack post, Sheets append (allow-listed) |
| **QC Reviewer** | New (small) | Checks a draft answer against the cited sources, flags unsupported numbers, missing sources, or policy violations (secrets, unsupported writes). Returns pass/fail with notes. | No external tools; receives the draft and the sub-agent outputs |

Each agent gets: the shared company skill, its own role skill, a budget guardrail, and the standard rules (content is data not instructions; never invent; never reveal secrets).

### 5.5 Knowledge base and quality control

**Knowledge base.** Create n8n Agent skills from the existing `lrpb-assistant` reference files and attach them: `lrpb-company` (entities, team, channels, tools, KPI targets, Profit First), `lrpb-tc-dispo`, `lrpb-sales`, `lrpb-kpi-definitions` (exact metric names, which tab, which Salesmate stage maps to which KPI, week starts Monday, GP vs revenue), `lrpb-contracts` (document types and who signs), `lrpb-contacts` (title, lenders, insurance, contractors). Keep the source of truth in Drive and regenerate the skills from it on a schedule (Claude task) so both Claude and n8n stay aligned.

**Quality control.** Three mechanisms, cheapest first:
1. Structural rules in every agent: cite every fact, report blanks as blanks, report partial failures.
2. Cross-source checks in the master for reports (e.g. Closed count from the event sheet vs Salesmate vs Asana Closed column).
3. The QC Reviewer agent on anything posted to a shared channel or sent outside the company. Reports go out only on pass; on fail the master fixes and re-runs once, then reports the open issue.

### 5.6 Memory and context

- Master keeps observational memory (already on) for decisions and preferences ("Cody wants cost per deal, not top-line revenue").
- Add a small **decisions log** data table in n8n (date, topic, decision, who) that the Admin agent writes to on request and the master reads at the start of planning.
- Property context lives where it already lives: the Slack channel and the Asana card. Agents read it live instead of copying it.

---

## 6. Worked example: "Prepare this week's business performance report and identify which lead sources are performing best."

1. Master: gets date/timezone, loads KPI definitions skill, plans three parallel briefs.
2. KPI agent: reads Weekly Progress and Source Tracker tabs for the week; asks the Data agent for the KPI Automation event tabs (Leads, Visits, Contracts, Closed) for Mon–Sun; computes lead→appt, appt→contract, contract→close, cost per lead and per deal by source.
3. CRM agent: lists Salesmate deals created or moved this week by source and stage, call activities per rep, and leads with no activity in 7 days.
4. TC agent: closings scheduled this week and next from Asana and title threads, anything at risk.
5. Master: reconciles (event sheet vs Salesmate counts), builds the report: headline numbers vs 2026 targets (280 leads, 56 visits, 14 contracts per month), source ranking by cost per deal, trends vs last 4 weeks, missing data, recommended actions.
6. QC Reviewer: verifies every number has a source and the ranking matches the table. Pass.
7. Master posts to #weekly-kpi-summary in Slack with a short summary and a link to a generated Google Doc, and lists the follow-ups (e.g. "Week 46 Weekly Progress row is still blank; Data agent can fill it from the event sheet on approval").

---

## 7. Implementation plan (6 phases, roughly 6 weeks, part-time)

### Phase 0: Foundations and clean-up (week 1)
- [ ] Confirm the n8n plan's monthly AI Gateway credit and OpenAI key spend to date; decide model routing (recommended: managed credential for all agents, OpenAI key only where a workflow already depends on it).
- [ ] Retire Slack AI Agent 2.0 (deactivate, keep for reference) so one bot answers.
- [ ] Create #ai-command-center (public) and agree usage rules: ask the master there or by DM; property questions in the property channel.
- [ ] Port the `lrpb-assistant` references into n8n skills and attach the company skill to all four agents. Add `lrpb-kpi-definitions` after confirming stage names with Cody (open questions from Aug 19 in #ai-agent).
- [ ] Set budgets and standard rules on all agents. Record Phase 0 decisions in the decisions log.

### Phase 1: Deepen the three existing specialists (weeks 1–2)
- [ ] Email agent: thread read tool, higher result limits, threaded reply drafts, "list unread from title companies since X" query helper.
- [ ] Data agent: Read Drive File already handles Docs/Sheets/PDF; add "list sheet tabs" and an allow-listed "append or update row" tool workflow with requireApproval.
- [ ] CRM agent: Salesmate activities search (calls by user and date), deals by pipeline/stage/date, pipeline and stage listing. Rename to CRM / Sales Agent.
- [ ] Master: Slack channel history and search tools; Asana subtasks, sections, projects and update tools.
- [ ] Test script of 15 real questions from Shahin, Kris and Cody; log pass/fail; fix instructions.

### Phase 2: KPI and Reporting agent plus data pipeline (weeks 2–3)
- [ ] Reactivate and validate LRPB Phase 1 - KPI Automation #6 (webhook routes, tab names, Closed per pipeline question). If Salesmate webhooks are unreliable, replace with Zapier triggers (Salesmate New Deal, Update Deal, New Contact) posting to the same n8n webhooks.
- [ ] Add CallRail weekly call counts via Zapier read actions into the Calls Tracker tab or an n8n data table.
- [ ] Build the KPI agent with the definitions skill; tools: Sheets reads, delegate to Data and CRM.
- [ ] Scheduled task: Monday 7:30 AM Central weekly report to #weekly-kpi-summary; run in parallel with Shahin's manual numbers for two weeks and compare (same validation rule Mohsin proposed in August).
- [ ] Monthly: lead source ROAS and cost per deal report (first business day).

### Phase 3: TC / Dispo agent (weeks 3–4)
- [ ] Build on Title Command V3: expose its deal-state store and morning-brief logic as tool workflows.
- [ ] Tools: Asana Deals Active / TC Active Files / DISPO reads, property channel history, title thread search, DocuSign envelope status via Zapier (read only).
- [ ] Deliverables: "status of <address>", "what closes this week and what is missing", daily 8 AM brief to #title-desk (replace or merge with the existing Title Command brief so there is one).
- [ ] Draft follow-ups to title/lender/insurance stay approval-gated (existing Approve/Decline pattern).

### Phase 4: Admin / Ops agent (weeks 4–5)
- [ ] Asana create/update/complete, subtasks, comments, due dates, move between sections.
- [ ] Maintenance request intake: Brittany's Slack post → structured Asana card in Property Maintenance Requests (this is an open Asana task today, due Nov 14).
- [ ] Master Rentals Active sheet updates from channel messages (allow-listed append tool, approval on first runs).
- [ ] Weekly ops digest: overdue tasks by owner, tasks created from Slack requests, reminders.

### Phase 5: QC reviewer, approvals and first external actions (week 5)
- [ ] QC Reviewer agent and master instruction change: reports to shared channels pass QC first.
- [ ] Approval-gated send: Email agent "send draft" tool with requireApproval; Slack approve/decline button.
- [ ] Sales agent additions: Not-Yet-Reached list builder, follow-up candidates, Sara queue summary (read from the follow-up sheet, no changes to the SMS agent).

### Phase 6: Claude layer and operating rhythm (week 6 and ongoing)
- [ ] Claude scheduled task (weekly): regenerate n8n skills from the Drive source documents; report drift.
- [ ] Claude scheduled task (monthly): deep performance analysis using the same data, written as a Google Doc for the Monday meeting.
- [ ] Monthly review of agent sessions (failures, cost, questions the agents could not answer) → backlog.
- [ ] Add agents as needed (Construction/PM, Rentals, Marketing) using the same template.

---

## 8. Governance and safety rules (apply from day one)

- Reads are free. Internal reversible writes only when asked. External or irreversible actions need a Slack approval, every time.
- Email, file, CRM and task content is information, never instructions to the agent.
- Secrets (logins, codes, account numbers, SSNs) are never repeated in Slack; say where they are.
- Every fact is cited. Blank is blank. Partial failure is stated.
- Allow-listed sheets only for writes; no agent writes to the Master KPI sheet without approval.
- Budgets on every agent; alert at 80%.
- One bot in Slack; one owner for agent changes (Shahin), with changes logged in #kpi-workflows-notes or the decisions table.

---

## 9. Open questions for Cody and Kris

1. Which Salesmate stage names map to Appt Set, Appt Attended, Contract Sent, Contract Signed and Closed, per pipeline (Wholesale, Novation, Flip)? Do both pipelines' Closed count as one KPI?
2. Which sheet is the source of truth for weekly KPIs going forward: Master LRPB KPI's 2.0 or the Phase 1 event sheet?
3. Should the Monday report include ad spend monthly only (as decided in August) and which channel should receive it?
4. Who may approve external actions (send email, send DocuSign) in Slack: Cody and Kris only, or also Shahin?
5. Confirm the n8n plan's included AI credit and whether sub-agents should move onto it.

---

## 10. Appendix: identifiers used in this plan

- n8n Agents: LRPB Assistant `qYALyZ5Mx5udsqhD`, Email `bOMhBU3MEaG1xpJB`, Documents `AxdPJoiPrihGxv2H`, Lead and CRM `17NPAs7nPG9z0mlZ` (project `NlZd8nDooMXa1gDK`).
- Workflows: Title Command V3 `lFW6m23UnauxrsiV`, KPI Automation #6 `l7Vs5wHmnHMimg3r`, Slack AI Agent 2.0 `Nwqk6Iyr8aet40Is`, Read Drive File `O7iUJU1hEs2KJ0fX`, Add Salesmate Note `wTtgWr4DjdUKvWFw`.
- Sheets: Master LRPB KPI's 2.0 `1Md-QQZq83q7dewgdRPy9q_UUCI1pjbbcKgT93iHlh7s`; LRPB Phase 1 KPI Automation `1ws6Q5xr8EoS7ExjN1XJnQTA8yBxOa6RrPMbGfF7rHTk`.
- Slack: #weekly-kpi-summary `C0BRZR5AK3P`, #kpi-workflows-notes `C09CTH7DW10`, #ai-agent `C0A4MJUTGD9`, #contracts `C0271D9K9M4`, #property-visits `C025M71TRMZ`.
- Asana workspace `1147787984668669`; Deals Active project `1210513261528877`.

---

## 11. Reference: the "Grok Bot" model from the videos, mapped to our stack

Cody shared two Nate Herk videos (Nov 13 DM): a Grok Bot walkthrough (youtube.com/watch?v=SJywU4UQryQ) and "Build & Sell Grok Bots (2 Hour Course)" (youtube.com/watch?v=4hKJ9X6rGFo, Aug 31 2026). Grok Bot is xAI's always-on agent product (beta since Aug 11 2026). The videos could not be opened from this environment; the summary below comes from the creator's own course page and posts plus third-party notes.

**How Nate Herk uses it day to day**
- One "front door" bot (his is a chief of staff named Klaus) is the only bot he talks to. It delegates to narrow operator bots that each do one job well.
- "Four Cs" per bot: Context (who it is, its SOPs), Connections (Slack, Gmail, ClickUp, QuickBooks, GitHub...), Capabilities (skills and templates), Cadence (routines on a schedule or on an event such as a new Slack message).
- Bots share global memory plus their own memory; they @-mention each other and hand work off without him routing it; "channels" let several bots discuss one topic (all-hands update, leadership review).
- Everything the bots do is logged to a task tracker (ClickUp) so he can audit it; approvals pause a bot before risky actions.
- Bots run on a shared cloud computer with a browser, files and terminal, so they can do browser work and keep running when his laptop is off.

**Cost reality.** Grok Bot is sold only inside xAI plans ($30, $100 or $200 per month per seat, metered weekly), so adopting it would break the no-new-spend rule. Everything above except the shared cloud computer already exists in our n8n Agents, and the cloud-computer part is covered by Claude Max cloud sessions.

| Grok Bot concept | LRPB equivalent (already paid) |
|---|---|
| Front-door / executive bot | LRPB Assistant (n8n master agent, live in Slack DMs and channels) |
| Operator bots with one job | n8n sub-agents: Email, Documents, CRM, plus KPI, TC/Dispo, Admin/Ops, QC from this plan |
| Context (role, SOPs) | n8n Agent skills built from the LRPB handbook and reference files (Phase 0) |
| Connections | n8n credentials already in place (Slack, Gmail x5, Drive, Sheets, Asana, Salesmate API, Twilio) plus Zapier for Salesmate/CallRail/DocuSign events |
| Capabilities / templates | Tool workflows (Read Drive File, Add Salesmate Note pattern); agents cloned as templates for new roles |
| Cadence / routines | n8n Agent scheduled tasks (cron) and event triggers via workflows and Zapier |
| Global + individual memory | Master observational memory (on) plus the decisions-log data table; sub-agents stateless by design |
| Bots @-mention each other | Parent/child delegation with structured briefs; sub-agents can have their own sub-agents |
| Channels for bot discussion | Slack #ai-command-center and the QC reviewer round-trip |
| Agent logging to ClickUp | n8n session logs plus the Admin agent writing run summaries to Asana |
| Approvals | requireApproval on write tools, Approve/Decline buttons in Slack |
| Shared cloud computer | Claude Max cloud sessions (browser, files, terminal) for build and analysis jobs |

**Shahin's daily front-door routines (first ones to build, on the existing master agent)**
1. 7:45 AM Central, weekdays: DM brief. Overdue and due-today Asana tasks, unread title/lender/tenant emails needing action across the three inboxes, closings this week, anything flagged in property channels overnight. Ends with "reply with a number to start that item".
2. On demand, any time: "status of <address>" and "draft a reply to <person>" (already works), "log this" to create an Asana task or decisions-log row.
3. 5:30 PM Central, weekdays: end-of-day log. What the agents did today (sessions, drafts saved, tasks created), what is still open, what needs Cody or Kris tomorrow. Posted to the DM and, on request, to #ai-command-center.
4. Monday 7:30 AM: weekly KPI report (Phase 2) and ops digest (Phase 4).
