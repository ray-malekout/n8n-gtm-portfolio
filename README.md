# UK Grid Market Signals → Sales Actions
Turns monthly UK electricity data into segment-specific sales
signals for a cleantech sales team — posted to Slack and created as HubSpot tasks.

## The problem
Sales teams selling storage/flexibility need timely, relevant reasons to reach out.
Market data exists but nobody turns it into "what to say to which customer".

## What it does
- Monthly: fetches Ember UK generation data, calculates changes
- Posts a 5-line market briefing to Slack
- Creates a tailored follow-up task on every target account in HubSpot
- An AI agent answers ad-hoc questions about the data

## Key design decisions
- Code does the maths, AI writes the words (after the AI misattributed a change)
- Validation step blocks any AI output with a number not in the data
- One AI call per segment, not per account (cost + consistency)
- Duplicate and "same month" guards so re-runs never double-post

## Problems found and fixed
- Lookup silently returned 2 of 16 rows → found by testing against known values
- Agent quoted the wrong row from 150 near-identical ones → restructured its input
- A step ran 123,904 times (352²) → diagnosed from item counts

## Stack
n8n · Claude (Anthropic API) · Claude Code + MCP · Ember API · Slack API ·
HubSpot API · GitHub

## My role
Designed the system, defined requirements and test criteria, and directed and
verified the build using AI-assisted development (Claude Code).

## What I'd do next
Assign tasks to account owners · always-on hosting · accuracy tracking over time

## Build log
Full technical log, test runs and decisions: docs/BUILD_LOG.md
