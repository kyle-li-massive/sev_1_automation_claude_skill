---
name: sev1-automation
description: Automates the SEV-1 incident flow — checks Linear for a same-day duplicate first, then files a Support Engineering (SPT) ticket with the Sev-1 label and waits for the user's review (single confirmation gate), then creates a dedicated `#sev-1-spt-xxxx` Slack channel, posts the ticket there, and starts a thread in `#support-sev-1`, all without asking again. Then prints a recommended status-page fill-in (deltas only) and, if the user opts in, polls the incident channel read-only every ~2 min and prints status-page suggestions in terminal — never posts/writes to Slack, Linear, or the status page itself. Triggers: "file a SEV-1 ticket", "file a SEV-1 SPT ticket", "declare a sev-1", "this is a sev-1".
---

# SEV-1 Incident Automation

Source of truth: [SEV-1 Claude automation](https://app.notion.com/p/mod-labs/SEV-1-Claude-automation-3cff21ab91ff8071b0f1d143b81899d3) (Notion). Reference precedent: ticket [SPT-2029](https://linear.app/massive-com/issue/SPT-2029/benzinga-cursor-pagination-500s-on-page-2-earnings-ratings-analyst), channel [#sev-1-spt-2029](https://polygonio.slack.com/archives/C0BUVGBNF8C), and its [#support-sev-1 thread](https://polygonio.slack.com/archives/C04M9V73W0L/p1788442157104459).

## Trigger

User asks to file/declare a SEV-1 (SPT) ticket, or clear equivalents ("this is a sev-1", "sev-1 this issue").

## Flow overview

1. Check Linear for a same-day Sev-1 duplicate.
2. File the SPT ticket, hand the link back, and stop for review.
3. On go-ahead: create the incident channel, post the ticket there, start the `#support-sev-1` thread.
4. Remind the user to handle PagerDuty and the status page themselves.
5. Print a recommended status-page fill-in — deltas only, never filed automatically.
6. Offer opt-in, read-only Slack monitoring that only ever prints suggestions.

Only steps 3 and the read in step 6 touch Slack; step 2 is the only write to Linear. Nothing in steps 4–6 writes to Slack, Linear, or the status page.

## Step 1 — Check for a same-day duplicate first

- `list_issues` with `team: SPT`, `label: Sev-1`, `orderBy: createdAt`, `fields: ["title","createdAt","url","status"]`.
- Compare each result's `createdAt` against today's date. If one was already filed **today**, stop — don't file a new ticket.
  - Tell the user which ticket it is (title + URL) and that someone may already be mid-flow setting up the channel/thread for it.
  - Point them at that ticket instead of proceeding. Only continue past this if the user explicitly confirms it's a distinct, unrelated incident.
- No same-day match → continue to Step 2.

## Step 2 — File the Linear ticket, then STOP

- Use the Linear MCP. Team is always **Support Engineering (SPT)** per CLAUDE.md.
- Follow `/linear-ticket-format` for description structure, evidence, and writing style — **except** the trailing Impact box: skip it. `/linear-ticket-format`'s template normally appends the empty `Impact to Users` / `Impact to Massive` scoring table, but a Sev-1 is impact by definition, so leave that table off the description entirely for this flow.
- Attach the **Sev-1** label — look it up with `list_issue_labels` (team: SPT); don't guess the label ID.
- Save the issue, then paste the ticket URL back to the user and ask them to review / give feedback.
- **Hard stop here.** Do not touch Slack until the user explicitly says to proceed. If they ask for changes, edit the ticket and stay at this gate.

## Step 3 — Once the user says proceed

This is the only confirmation gate in this flow. The user's go-ahead at Step 2 covers all of Step 3 — run the three sub-steps below in order without asking the user to re-confirm partway through.

1. **Create the incident channel**: name it `sev-1-spt-xxxx`, where `xxxx` is the Linear ticket number, e.g. ticket `SPT-2029` → channel `sev-1-spt-2029` (lowercase, matches Linear's own casing convention).
2. **Post in that new channel**: `SPT-XXXX` as hyperlinked text (markdown `[SPT-XXXX](ticket url)`) — not the bare URL — cc'ing `@support-team`. Set `unfurl_app_links: true` so the Linear preview still renders. Slack's unfurl triggers off the URL itself, not the link's display text, so the hyperlinked short form still shows the same rich preview while keeping the message readable.
3. **Start a new thread in `#support-sev-1`** with exactly this structure (blank lines are literal, not decorative):
   ```
   SPT-XXXX          <- hyperlinked text (markdown [SPT-XXXX](ticket url)), not the bare URL — unfurl_app_links: true
                      <- blank line
   Please join #sev-1-spt-xxxx     <- use <#CHANNEL_ID> so it renders as a clickable channel link
                      <- blank line
   cc @product-eng-support         <- use the proper mention syntax so it actually tags
   ```

## Step 4 — Once done, remind the user

Tell the user — don't do these yourself, they're not automatable from here:
- Trigger the PagerDuty alert, if needed
- Update the status page, if needed

## Step 5 — Print a status-page recommendation (never file it)

Right after Step 4's reminders, print a recommended fill-in for the status page. This is terminal output only, for the human to act on — no status-page tool exists in this flow, and none should be invented; the recommendation is the deliverable.

Fields:
- **Incident Name** — short, Title Case (every word capitalized), no technical detail, illustrative of user-facing impact (e.g. "Futures Aggregates Pagination Not Accessible").
- **Message** — recommend leaving blank by default. Only suggest 1–2 sentences if there's real scope context worth adding beyond the name.
- **Components affected** — **deltas only**: list just the asset-class × source combinations that are actually impacted, each with its status (Degraded performance / Partial outage / Major outage). Asset classes: Stocks, Options, Indices, Forex, Crypto, Futures. Sources within each: Market Data REST Endpoints, Reference Data REST Endpoints, Market Data WebSocket Channels, Market Data Flat Files. Don't enumerate unaffected combinations — they stay Operational by default and adding them is just noise.

Base this on what's actually confirmed in the ticket (the "What we confirmed" / scope section) — never guess impact that wasn't verified.

## Step 6 — Offer read-only Slack follow-up (opt-in)

After Step 5, ask the user (AskUserQuestion, don't assume): **"Do you want me to keep following up on the status in Slack?"**

If no — stop here.

If yes — poll for updates, but this is strictly observe-and-suggest:
- Use `ScheduleWakeup` at ~120s (2 min) cadence to resume.
- Each wake-up: read `#sev-1-spt-xxxx` (the incident channel from Step 3) **read-only** — `slack_read_channel`/`slack_read_thread` only — for messages since the last check. Never post, react, or otherwise write to Slack, Linear, or the status page during this loop; the only output is what you print to the user in terminal.
- Match new messages against these patterns and print a recommendation (nothing else):
  1. **Partial fix, improving but not fully resolved** → suggest setting the status page to "Monitoring" (default assumed "Investigating").
  2. **Fix applied, issue likely fully resolved immediately** → suggest closing the incident on the status page.
  3. **Fix applied, resolved but a data backfill is still pending** → suggest closing the incident, but note the backfill lag in the suggested close message.
  4. **No fix in the last 30+ min** → draft a 1–3 sentence status-update message the user could post.
- Recommending case 2 or 3 (closing the incident) is the natural stop condition — after printing that recommendation, call `ScheduleWakeup` with `stop: true` instead of scheduling another wake-up; the incident is presumed resolved from a status-page standpoint. Otherwise keep rescheduling until the user says to stop.

## Rules

- **Never skip the Step 2 gate.** Channel creation and posting to `#support-sev-1` are visible, hard-to-reverse, org-wide actions — always wait for explicit confirmation after the ticket is filed.
- **Never skip the Step 1 duplicate check.** Filing a second ticket for an incident already in flight creates confusion and duplicate channels/threads mid-incident.
- **Match the template text exactly** — line breaks and tag placement in the `#support-sev-1` thread matter; people skim it fast during an incident.
- **Don't invent the label or user-group IDs.** Resolve `Sev-1`, `@support-team`, and `@product-eng-support` via the Linear/Slack MCPs each time rather than hardcoding — they can be renamed.
- **Steps 5 and 6 never write to Slack, Linear, or the status page.** They only read (Step 6) and print recommendations. Any actual status-page fill-in, Slack post, or Linear update stays a human action — the skill's job stops at suggesting it.
