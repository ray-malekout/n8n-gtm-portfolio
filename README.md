# n8n GTM portfolio

GTM automation on a self-hosted n8n: UK grid data from Ember becomes market signals, then Slack briefings and HubSpot tasks for a cleantech sales team. Project rules are in `CLAUDE.md`.

## Workflows

| Workflow | Status | Export (from live n8n) |
|---|---|---|
| Ember UK – monthly sync | **Published** (active version = draft). Schedule day 10 of each month, 09:00, then runs the Slack briefing. Does **not** call the HubSpot workflow | `workflows/ember-uk-monthly-sync.json` |
| Ember UK – get month | **Published** (active version = draft), including the row-limit fix. Lookup tool for the "Ember Energy" agent | `workflows/ember-uk-get-month.json` |
| **Ember UK – Slack briefing** (phase 1) | **Published** (active version = draft). No trigger of its own: runs when the sync calls it; Manual Trigger for testing | `workflows/ember-uk-slack-briefing.json` |
| **Ember UK – HubSpot signal tasks** (phase 2) | **Tested and run on all 12 companies; not published** (left for the owner to publish). Two triggers: Manual (test mode on) and "When Executed by Another Workflow" (test mode off unless the caller passes it) | `workflows/ember-uk-hubspot-signal-tasks.json` |

**Draft vs active:** n8n runs the *active* (published) version for schedules and for the agent. Publish state was last checked through the n8n API on 2026-11-20 (see "Publish state" below).

All four exports were checked for secrets: only credential names and IDs appear (`Ember API Query Auth account`, `Anthropic account`, `Slack bot`, `HubSpot Sandbox`). No keys or tokens are stored in any node.

**Export rule:** exports are always taken from the live workflow (`get_workflow_details`), never reconstructed. See `CLAUDE.md`.

## Phase 1: Ember UK – Slack briefing

Data table: `ember_uk_monthly` (one row per month and series).

```
Manual Trigger
  -> Get all Ember rows     Data table, Return All (no default row limit)
  -> Pick signals           Code: latest month, Clean share + change, top 3 movers
  -> Write briefing         Anthropic, claude-sonnet-5-5
  -> Check briefing         Code: blocks the run if the text breaks the rules
  -> Post to #market-signals  Slack, channel ID C0CKMTH301X
```

Credentials (by name, none stored in nodes): `Anthropic account`, `Slack bot`.

### Design decisions

- **Code does the arithmetic, the AI only explains.** "Pick signals" finds the latest `YYYY-MM`, reads the Clean row, and ranks movers by absolute `share_change_pts`. The model is given the finished figures as text and told to quote them digit for digit.
- **Movers are individual generation sources only.** The table also holds roll-ups (Clean, Fossil, Renewables, Wind and solar, Hydro/bioenergy/other renewables), Demand, Total generation and Net imports. Ranking those would double-count (for 2026-10, Renewables -3.49 and Wind and solar -3.45 would top the list). Sources used: Bioenergy, Coal, Gas, Hydro, Nuclear, Other fossil, Solar, Wind.
- **Fail loudly, not silently.** "Pick signals" throws if there are no rows, no valid month, a missing or null Clean row, a source missing from the latest month, or a null value.
- **Slack by channel ID**, so renaming the channel does not break the workflow.

### The check node

"Check briefing" runs between the AI and Slack. It fails the whole run (nothing is posted) if:

1. there are more than 5 non-empty lines (exactly 5 passes),
2. the text contains any number that is not in the supplied figures (matched as exact strings, so 72.3 fails when the figure is 72.25; the year and month number are allowed so "October 2026" passes),
3. the last line does not start with "Sales angle:".

### Test results (2026-10)

Checked against the known-good values: Clean change -1.61 (match), Solar change -2.62 (match). 352 rows read, 16 in the month, movers Solar -2.62, Nuclear +1.88, Gas +1.80. Clean share for 2026-10 is 72.25. The 73.86 reference value is 2026-09 and was not tested here.

Posted briefing (execution 17):

> UK clean share of electricity generation was 72.25% in October 2026.
> That is down 1.61 percentage points on the previous month.
> Solar fell to a 5.38% share, down 2.62 percentage points.
> Nuclear rose to 13.72%, up 1.88 percentage points, and gas rose to 25.82%, up 1.8 percentage points.
> Sales angle: Use the fall in clean share and solar share to open conversations with prospects about on-site solar and storage.

### Run log

| Execution | Result | Notes |
|---|---|---|
| 15 | **Failed at "Check briefing", nothing posted** | See below |
| 16 | Success | Fixes applied; first message posted (with "72.25 %" spacing) |
| 17 | Success | "72.25%" spacing and channel ID; final version of phase 1 |
| 18 | Success | Test of "get month" for 2026-10 after the Return All fix: 16 rows (was 2) |
| 19 | Success | Manual run of the monthly sync with the new briefing step: 352 rows upserted, one briefing posted, new sales-angle prompt |

### Execution 15: the check node doing its job

The first run failed at "Check briefing" with:

`Briefing check failed: last line does not start with "Sales angle:". Text was: Cl`

What happened: the Anthropic model uses extended thinking, and thinking tokens count against `maxTokens`. With the cap at 400 the model used all 400 tokens thinking and returned only the two characters "Cl". The workflow would otherwise have posted "Cl" to #market-signals. The check blocked it, so nothing reached Slack.

A second bug showed up in the same run: the check read `mergedResponse`, but the node returns the merged text as `merged_response`. The check passed the empty-text guard only because the text blocks held "Cl".

Fixes (execution 16 onward): `maxTokens` raised to 2000, and the check now reads `merged_response`, falling back to `type: "text"` content blocks only (never the thinking block).

### Prompt change: sales angle must follow from the signal

After execution 17 the angle for a fall in solar was "on-site solar and storage". The system prompt now says the angle must follow logically from what the movement means for a UK business buyer, and not recommend more of the thing that fell (a fall in solar's share points to storage, flexibility and demand response; a rise in gas's share points to price-swing exposure, efficiency and long-term clean supply). The figures rules and the check node are unchanged. First run with the new prompt: execution 19 (below). It passed the check; the model chose the gas signal for the angle ("offer a long-term clean supply contract to steady their costs") rather than solar-to-storage. That follows the new rule but is one of several valid angles.

## Phase 2: the sync runs the briefing

"Ember UK – monthly sync" now ends with a **Run Slack briefing** node (Execute Workflow) after "Upsert row(s)":

```
Schedule / Manual -> HTTP Request -> Code -> Upsert row(s) -> Run Slack briefing
```

- **Execute once**: the upsert outputs one item per row (352). Without this the briefing would run, and post to Slack, 352 times.
- **Wait for sub-workflow**: on, so the briefing only reads the table after the upsert has finished.
- **Failure stays visible**: if the briefing fails (including the check node blocking a bad briefing), the sync run is marked failed. The upsert has already happened, so no data is lost.
- The briefing gained a second trigger, "When Executed by Another Workflow" (pass-through input), feeding "Get all Ember rows". The Manual Trigger is kept for testing.

Test, execution 19 (manual run of the sync, draft version): success. 352 items from the Code node, 352 upserted, 0 nulls, months 2025-01 to 2026-10, briefing ran once and posted one message. Known-good values from the sync's own output: 2026-09 Clean share 73.86, 2026-10 Clean change -1.61, 2026-10 Solar change -2.62, all matching.

Posted briefing:

> UK clean share of generation was 72.25% in October 2026, down 1.61 percentage points on the previous month.
> Solar fell to a 5.38% share, down 2.62 percentage points.
> Nuclear rose to a 13.72% share, up 1.88 percentage points.
> Gas rose to a 25.82% share, up 1.8 percentage points.
> Sales angle: Ask business buyers how exposed they are to gas-driven price swings and offer a long-term clean supply contract to steady their costs.

## Phase 2: Ember UK – HubSpot signal tasks

Turns the latest month's grid data into one talking point per customer segment and creates one follow-up task per company in the HubSpot sandbox (fake companies only).

```
Manual Trigger ------------> Mode: manual run (test_mode = true) ---------\
                                                                          +-> Run settings
When Executed by Another --> Mode: called by another workflow ------------/   (test_mode = false unless passed)
  -> Get all Ember rows            Data table, Return All
  -> Pick signals                  Code: latest month; Clean, Solar, Wind, Gas share + share_change_pts (individual sources only)
  -> Get Segment property definition   HTTP: reads the dropdown's labels and internal values
  -> Get companies                 HubSpot node, Return All, properties name + segment
  -> Select companies              Code: maps segment values to labels; test mode = 1 company from each of 2 segments
  -> Segment briefs                Code: one prompt per segment needed (2 in test mode, 4 in a full run)
  -> Write talking point           Anthropic, claude-sonnet-5-5, one call per segment
  -> Check talking points          Code: fails on >2 sentences or any number not in the data
  -> Attach talking points         Code: title "Market signal: <Month Year>", body = talking point + source line, due in 3 days
  -> List tasks on companies       HTTP: v4 associations batch read (tasks already on each company)
  -> Any existing tasks? -> Read task subjects   HTTP: v3 tasks batch read (subjects)
  -> Decide who needs a task       Code: skip a company if a task with the same title is already on it
  -> Needs a task? -> Create HubSpot task        HTTP: POST /crm/v3/objects/tasks with the company association (type 192)
  -> Created and skipped -> Run summary
```

Credentials (by name, none stored in nodes): `HubSpot Sandbox` (App Token), `Anthropic account`.

### Design decisions

- **Why HTTP for tasks.** The built-in HubSpot node has no Task resource; its only route is the deprecated v1 engagements API, which cannot search. Tasks use the CRM v3 objects API through the same `HubSpot Sandbox` credential (n8n's predefined credential type, no token in the node).
- **Duplicate guard by association, not search.** HubSpot's search index lags by seconds, so a quick re-run could miss a task it had just created. Reading the company's associated tasks has no such lag. A task counts as a duplicate if it is on the same company and has the same title. Any unreadable associated task fails the run rather than risk a duplicate.
- **Test mode defaults.** Manual Trigger sets `test_mode = true`. The "When Executed by Another Workflow" path sets it only if the caller passes boolean `true` or the string `"true"`; a missing or any other value gives `false`, so a scheduled run can never silently process just 2 companies.
- **One talking point per segment**, reused for every company in that segment. Test mode only generates the segments it needs.
- **Segment safety.** Companies with an empty Segment (the default "HubSpot" sample company) are skipped and listed in the run summary. A non-empty value that is not one of the four labels fails the run. The four labels and internal values are identical in this portal.
- **Tasks are unassigned** (HubSpot accepts this).

### Test results (execution 23 and 24, test mode, 2026-10)

Signals match the known-good values: Clean 72.25 (change -1.61), Solar 5.38 (change -2.62), Wind 34.91 (-0.85), Gas 25.82 (+1.8). 352 rows read. 13 companies fetched, 12 in scope, 1 skipped (blank Segment).

- Execution 23: 2 tasks created (Corvid Data Centres, Battery storage prospect; Meadowbank Farms, Solar customer), due 3 days out, associated with their companies.
- Execution 24 (immediate re-run): 0 created, both skipped as existing. The association read also confirmed both tasks are linked to their companies.
- The service key's scopes were sufficient for company read, property read, task read, association read and task create.

### Prompt change: rep-facing notes

The first talking points mixed voices ("your customer" / "their") and the Solar one implied a customer's sunny hours were shrinking because the national solar share fell. The system prompt and segment guidance now say: the note is for the rep, sentence 1 is the market fact using only the given figures, sentence 2 tells the rep what to raise with "these customers" and why it follows, and a grid share must never be presented as a change to a customer's own assets, output or bills. The figures rules and the check node are unchanged. Backup of the previous version: the committed export in `29fa23a`.

Test, execution 28 (test mode, new wording), passed the check node:

> **Battery storage prospect:** In October 2026 gas supplied 25.82% of UK electricity generation, up 1.8 percentage points on the previous month, while the Clean share fell 1.61 percentage points to 72.25%. Lead with storage as a way for these customers to gain flexibility and arbitrage against grid price swings, since a rising gas share points to greater exposure to gas-driven pricing.
>
> **Solar customer:** Solar's share of UK electricity generation in October 2026 was 5.38%, down 2.62 percentage points on the previous month, while gas rose 1.8 percentage points to a 25.82% share. Raise battery storage with these customers as a way to shift and store the power they generate, so they are less tied to the times when the grid leans more on other sources such as gas.

### Stubbed test of the "When Executed by Another Workflow" path

Run with `test_workflow` and pin data: every HubSpot, Anthropic and task-create node was stubbed, so nothing was written. The stub had 3 in-scope companies (2 Battery, 1 Solar) plus 1 with a blank Segment, so the two modes give different counts.

| Execution | Input from caller | `test_mode` | Companies processed |
|---|---|---|---|
| 25 | `{}` (not passed) | `false` | 3 of 3 in scope |
| 26 | `{ "test_mode": true }` | `true` | 2 (one from each of 2 segments) |

### Test-data cleanup

Test tasks were removed with a throwaway workflow that first read each task and refused to delete anything whose title did not start with "Market signal: ". Deleted: 550375678184 and 550374717672 (execution 27, before the prompt change), then 550382279882 and 550382203123 (execution 29, the wording check). That cleanup workflow is archived, as is the read-only verifier used for the full run (below).

### Full run on all 12 companies (execution 30, 2026-11-20)

Manual path temporarily set to `test_mode = false` for the run, then set back to `true` and confirmed by reading the live workflow (version `b18a93c9…`). It was set to `false` once more for the duplicate-guard re-run (execution 32), set back to `true`, and confirmed again by reading the live workflow (final version `8c68e6b5…`). The committed export has `test_mode = true` on the manual path.

Run summary: `test_mode` false, 13 companies fetched, **12 in scope**, 1 skipped for a blank Segment (the HubSpot sample company), 12 processed, **12 tasks created**, 0 skipped. Title "Market signal: October 2026", due 2026-11-23, no owner. Segment split: Solar customer 4, Battery storage prospect 3, Energy-intensive manufacturer 3, EV fleet operator 2.

| Company | Segment | Task id |
|---|---|---|
| Corvid Data Centres | Battery storage prospect | 550315856078 |
| Ashcombe Retail Parks | Battery storage prospect | 550382293201 |
| Pennine Steel Fabrication | Battery storage prospect | 549955471585 |
| Meadowbank Farms | Solar customer | 550382293200 |
| Northgate Cold Storage | Solar customer | 550383018199 |
| Clydeside Brewing | Solar customer | 550369257668 |
| Fellside Hotels | Solar customer | 550382370001 |
| Ridgeway Couriers | EV fleet operator | 550359177431 |
| Harbourline Logistics | EV fleet operator | 550365808846 |
| Tamar Glassworks | Energy-intensive manufacturer | 550337278163 |
| Brightwell Foods | Energy-intensive manufacturer | 550001279208 |
| Kestrel Plastics | Energy-intensive manufacturer | 549943069931 |

Bodies as stored in HubSpot (the talking point is generated once per segment and reused):

> **Battery storage prospect:** In October 2026 gas supplied 25.82% of UK electricity generation, up 1.8 percentage points on the previous month, while the Clean share fell 1.61 percentage points to 72.25%. Lead with storage as a way for these customers to gain flexibility and arbitrage, because a grid leaning more on gas leaves them more exposed to price swings that a battery can help them manage.
>
> **Solar customer:** In October 2026, solar's share of UK electricity generation was 5.38%, down 2.62 percentage points on the previous month, while gas rose 1.8 percentage points to 25.82%. Raise battery storage with these customers as a way to shift and store the power they generate, so they are less tied to the times when the grid is leaning on other sources such as gas.
>
> **EV fleet operator:** Clean share of UK electricity generation was 72.25% in October 2026, down 1.61 percentage points on the previous month, while Gas rose 1.8 percentage points to a 25.82% share. Raise smart and off-peak charging with these customers, because a grid leaning a little more on gas makes the timing of charging matter more for both carbon and cost, and scheduling it into cleaner, cheaper periods is how they can manage that exposure.
>
> **Energy-intensive manufacturer:** In October 2026 gas supplied 25.82% of UK electricity generation, up 1.8 percentage points on the previous month, while the clean share fell 1.61 percentage points to 72.25%. Raise long-term clean supply contracts and demand flexibility with these customers, because a larger gas share on the grid means more exposure to gas-linked price swings and these options reduce that exposure.

Each body ends with a blank line and "Source: Ember monthly UK generation data".

**Verified in HubSpot (execution 31, read-only):** a search for tasks titled "Market signal: October 2026" returned exactly 12; each is linked to exactly one company (association type 192 on the task side, 191 on the company side); each of the 12 companies has exactly one; the sample "HubSpot" company has none; every body has the source line.

**Duplicate guard in full mode (execution 32):** immediate re-run with `test_mode = false`: **0 created, 12 skipped as existing**, each skipped company pointing at its existing task id. The re-run generated different wording (the AI runs every time) but, correctly, nothing was written; the tasks in HubSpot keep the execution-30 wording.

### Run log (phase 2)

| Execution | What | Result |
|---|---|---|
| 22 | Read-only stage 1 (signals, Segment property, companies) in a scratch workflow, since archived | OK. Found 13 companies, one with a blank Segment |
| 23 | Test mode, 2 companies | 2 tasks created (first wording) |
| 24 | Immediate re-run | 0 created, 2 skipped |
| 25, 26 | Stubbed `test_workflow` of the "When Executed by Another Workflow" path | `test_mode` false when not passed, true when passed |
| 27 | Cleanup of the first 2 tasks | 2 deleted |
| 28 | Test mode with the rep-facing prompt | 2 tasks created, check passed |
| 29 | Cleanup of those 2 | 2 deleted |
| 30 | **Full run, `test_mode = false`** | **12 created** |
| 31 | Read-only verification in HubSpot | 12 tasks, one per company, all linked |
| 32 | Re-run, `test_mode = false` | 0 created, 12 skipped |

### Publish state (checked via the API, 2026-11-20)

| Workflow | Active | Draft vs active version |
|---|---|---|
| Ember UK – monthly sync | yes (1 schedule trigger) | identical (`versionId` = `activeVersionId` = `db71e06d…`) |
| Ember UK – Slack briefing | yes | identical (`4abb33b4…`) |
| Ember UK – get month | yes | identical (`04921143…`) |
| Ember UK – HubSpot signal tasks | **no** | never published; draft `b18a93c9…` |

The earlier "pending publishes" list is obsolete: the sync, the briefing and get month were published by the owner, so the scheduled sync on day 10 now runs the Slack briefing and the agent uses the row-limit fix. The HubSpot workflow is intentionally unpublished for the owner to publish. It is **not wired into the monthly sync**: even published, the scheduled sync will not create HubSpot tasks until an Execute Workflow step is added (with "Execute once" on, like the briefing step).

### Live export vs my earlier reconstructed export

The first export of this workflow was rebuilt from the code I used to create it. Compared with the live workflow, it differed only in metadata the n8n API adds: `id`, `active`, `versionId`, `activeVersionId`, the Slack node's `webhookId`, and `meta`. Nodes, parameters, code, credentials and connections were identical. The file is now the live export.

### Known limitations

- The Anthropic node (v1) has no option to disable extended thinking, so `maxTokens` stays at 2000 rather than a lower cap. If the node gains that option, 500 would be enough.
- Real Ember figures are posted to Slack (public grid data, no personal data). The fake-data rule applies to people, contacts and accounts.
- Published, but it has no trigger of its own: it runs when the monthly sync calls it (or from the Manual Trigger).

## Known issues in the existing system

- **"Ember UK – get month" returned only part of a month (fixed and published).**
  - *How it was found:* while exporting the workflow I noticed its "Get row(s)" node had no Return All set, so n8n's default limit of 50 rows applies to a 352-row table. I then read the output of its last run, execution 14 (month 2026-10): the result held only **2 of the 16 rows** (Bioenergy and Clean). Solar, Nuclear and Gas were missing, so the "Ember Energy" agent could not see them. The run still reported success, which is why nothing flagged it.
  - *Fix:* Return All switched on in "Get row(s)" (backup: `ember-uk-get-month.backup-before-row-limit-fix.json`).
  - *Verification:* test execution 18 for 2026-10 returned **16 rows**, including Solar -2.62, Nuclear 1.88, Gas 1.8 and Clean -1.61 (changes match the known-good values; Clean share for 2026-10 is 72.25).
  - *Published:* the fix is in the active version (checked via the API, see Publish state). A run of the agent against the published version has not been repeated here.
  - *Lesson (already in `CLAUDE.md`):* default row limits fail silently, so check the row count of any data table read.
- The monthly sync shows 1 failed run out of 7 (execution 4) from before this work, not investigated.

## Files

- `workflows/ember-uk-slack-briefing.json`: current live export (import into n8n via Import from file)
- `workflows/ember-uk-hubspot-signal-tasks.json`: phase 2 export, hand-copied from the live `get_workflow_details` output; checked against the source submitted to n8n (7 of 8 Code nodes identical; the two nodes changed by the prompt change were copied from the live output)
- `workflows/ember-uk-monthly-sync.json`, `workflows/ember-uk-get-month.json`: live exports of the existing workflows, re-exported after they were published so `active` and `activeVersionId` are current. Compared with the previous exports, the only differences were those publish fields (sync and get-month: `activeVersionId`; briefing: `active` false to true and `activeVersionId`); nodes, code, credentials and connections were unchanged
- Backups taken before edits:
  - `ember-uk-slack-briefing.backup-before-tweaks.json`: state before the "%" and channel-ID changes (hand-copied from the live output at the time, before the live-export rule)
  - `ember-uk-slack-briefing.backup-before-prompt-change.json`: live export before the sales-angle prompt change
  - `ember-uk-slack-briefing.backup-before-execute-trigger.json`: live export of the briefing before the second trigger was added
  - `ember-uk-monthly-sync.backup-before-briefing-step.json`: live export of the sync before the briefing step was added
  - `ember-uk-get-month.backup-before-row-limit-fix.json`: live export of get month before Return All was turned on
- `CLAUDE.md`: project rules
