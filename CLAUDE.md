# n8n GTM portfolio — project rules

## Context
Self-hosted n8n on localhost:5678, accessed via MCP. Portfolio project showing GTM automation:
UK grid data (Ember) → market signals → Slack briefings and HubSpot tasks for a cleantech sales team.

## Existing system (do not break)
- "Ember UK – monthly sync": Schedule → HTTP (Ember API, Query Auth credential) → Code (calculates changes) → Upsert into data table `ember_uk_monthly`
- "Ember UK – get month": lookup tool for the "Ember Energy" agent; returns one month's rows
- Known-good values for testing: 2026-09 Clean share 73.86; 2026-10 Clean change −1.61; 2026-10 Solar change −2.62

## Rules
- Code does arithmetic; the AI only explains numbers it's given
- Never put API keys or tokens in nodes. Use existing n8n credentials by name; ask me to create new ones
- Before editing an existing workflow, export it to /workflows as a backup
- After building, run the workflow and check the output against the known-good values; report any mismatch
- Watch for silent failures: default row limits, empty results, nulls
- Keep changes small: build one workflow at a time, test it, then move on
- After each finished change, export the workflow JSON to /workflows and update README.md
- Always export from the live n8n workflow (fetch it with get_workflow_details at that moment). Never reconstruct an export from earlier code, memory or an older file. If the file has to be hand-copied from the live output, diff it against the live data and report any differences
- Use fake data only in HubSpot and Slack
- Commit after each finished, tested change, with a clear message that says what changed and why. Scan the files for API keys and tokens before every commit, and show me anything suspicious instead of committing it
