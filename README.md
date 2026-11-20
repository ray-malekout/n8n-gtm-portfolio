# n8n GTM portfolio

GTM automation on a self-hosted n8n: UK grid data from Ember becomes market signals, then Slack briefings (and later HubSpot tasks) for a cleantech sales team. Project rules are in `CLAUDE.md`.

## Workflows

| Workflow | Status | Export (from live n8n) |
|---|---|---|
| Ember UK – monthly sync | Published; schedule day 10 of each month, 09:00. **Draft now has the "Run Slack briefing" step; the active (scheduled) version does not yet** | `workflows/ember-uk-monthly-sync.json` (draft state) |
| Ember UK – get month | Published (agent lookup tool). **Draft has the row-limit fix; the active version the agent uses does not yet** | `workflows/ember-uk-get-month.json` (draft state) |
| **Ember UK – Slack briefing** (phase 1) | Tested, **unpublished**. Has two triggers: Manual and "When Executed by Another Workflow" | `workflows/ember-uk-slack-briefing.json` |

**Draft vs active:** n8n runs the *active* (published) version for schedules and for the agent. Edits are saved as drafts until published. Publishing needs to be done by the owner; see "Pending publishes" below. The export files show the draft, which is what manual runs use.

All three exports were checked for secrets: only credential names and IDs appear (`Ember API Query Auth account`, `Anthropic account`, `Slack bot`). No keys or tokens are stored in any node.

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

### Pending publishes (not done)

n8n schedules and the agent run the active version, and these three have changes only in draft:

1. **Ember UK – Slack briefing** must be published, otherwise a scheduled sync cannot call it.
2. **Ember UK – monthly sync** must be published, otherwise the scheduled run on the 10th still uses the old version without the briefing step.
3. **Ember UK – get month** must be published for the row-limit fix to reach the "Ember Energy" agent. Version history shows its draft differs from the active version only by this fix (no other node or connection differences).

Publishing was attempted for get month and denied by the permission system, so nothing has been published.

### Live export vs my earlier reconstructed export

The first export of this workflow was rebuilt from the code I used to create it. Compared with the live workflow, it differed only in metadata the n8n API adds: `id`, `active`, `versionId`, `activeVersionId`, the Slack node's `webhookId`, and `meta`. Nodes, parameters, code, credentials and connections were identical. The file is now the live export.

### Known limitations

- The Anthropic node (v1) has no option to disable extended thinking, so `maxTokens` stays at 2000 rather than a lower cap. If the node gains that option, 500 would be enough.
- Real Ember figures are posted to Slack (public grid data, no personal data). The fake-data rule applies to people, contacts and accounts.
- Not published and no schedule yet; it runs from the Manual Trigger only.

## Known issues in the existing system

- **"Ember UK – get month" returned only part of a month (fixed in draft, not yet published).**
  - *How it was found:* while exporting the workflow I noticed its "Get row(s)" node had no Return All set, so n8n's default limit of 50 rows applies to a 352-row table. I then read the output of its last run, execution 14 (month 2026-10): the result held only **2 of the 16 rows** (Bioenergy and Clean). Solar, Nuclear and Gas were missing, so the "Ember Energy" agent could not see them. The run still reported success, which is why nothing flagged it.
  - *Fix:* Return All switched on in "Get row(s)" (backup: `ember-uk-get-month.backup-before-row-limit-fix.json`).
  - *Verification:* test execution 18 for 2026-10 returned **16 rows**, including Solar -2.62, Nuclear 1.88, Gas 1.8 and Clean -1.61 (changes match the known-good values; Clean share for 2026-10 is 72.25).
  - *Still to do:* publish the workflow so the agent gets the fix (see Pending publishes).
  - *Lesson (already in `CLAUDE.md`):* default row limits fail silently, so check the row count of any data table read.
- The monthly sync shows 1 failed run out of 7 (execution 4) from before this work, not investigated.

## Files

- `workflows/ember-uk-slack-briefing.json`: current live export (import into n8n via Import from file)
- `workflows/ember-uk-monthly-sync.json`, `workflows/ember-uk-get-month.json`: live exports of the existing workflows
- Backups taken before edits:
  - `ember-uk-slack-briefing.backup-before-tweaks.json`: state before the "%" and channel-ID changes (hand-copied from the live output at the time, before the live-export rule)
  - `ember-uk-slack-briefing.backup-before-prompt-change.json`: live export before the sales-angle prompt change
  - `ember-uk-slack-briefing.backup-before-execute-trigger.json`: live export of the briefing before the second trigger was added
  - `ember-uk-monthly-sync.backup-before-briefing-step.json`: live export of the sync before the briefing step was added
  - `ember-uk-get-month.backup-before-row-limit-fix.json`: live export of get month before Return All was turned on
- `CLAUDE.md`: project rules
