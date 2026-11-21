# AI PRD · Juno

> Module 3 · Write an AI PRD Engineers Can Build from. All six harness surfaces for Juno, Slido's Customer Voice agent.

## Problem & user

**User:** Me (Adam), the PM who runs Slido's customer voice program, and teammates who reply to NPS feedback in LaDesk.

**Problem:** Before an NPS comment can get a good reply, someone has to dig: open the event in Slido, check New Relic for errors, check Superset for usage. That takes time, so many comments get a late reply or none. The weekly summary is put together by hand, and patterns that build up over several weeks are easy to miss.

**What Juno changes:** Juno does the digging and the drafting. A human still decides and sends.

**Archetype:** Scheduled drafting agent with a human approving every action. Juno investigates and drafts; a person approves anything that leaves the building.
- Not a chat assistant: I'd still have to do the digging myself.
- Not a fully autonomous agent: a wrong reply to a customer can't be taken back.

**How to read the requirements:** **[Code]** = enforced by ordinary code, must hold 100% of the time. **[Model]** = what acceptable model output must satisfy, measured against a pass bar.

_____

## The harness

| Surface | Specification |
|---|---|
| 01 Context | NPS threads in LaDesk, the Slido event behind each one, New Relic errors, Superset usage, Jira status, Slido product knowledge and past replies for tone. No billing data. Personal data removed before the model sees anything. |
| 02 Tools | Read and draft tools only, plus posting to a private Slack review channel. No send, refund, create-ticket or account-change tools. |
| 03 Loop | Max 10 tool calls per feedback, 2 tries per source. Hands sensitive, unclear or contradictory items to a human. Groups outage spikes into one card. |
| 04 Memory | Team-level memory of past feedback, outcomes, decisions and patterns for 90 days. Patterns stay "possible" until a human confirms them. |
| 05 Permissions | read: auto · draft: auto · write: auto to the review channel only, everything else confirm · send: blocked for Juno, only a human click sends |
| 06 Verification | Code checks IDs, personal data, internal details, links and promises. Model output must meet the skill file and the Module 5 rubric. Failed checks are flagged to the reviewer, never dropped. |

## 01 Context · Data Requirements

- **Required sources:**
  - **LaDesk (LiveAgent) NPS tickets:** comment, rating, AI follow-up question and answer, our replies, customer replies.
  - **Slido event data:** integration, audience size, date and plan, looked up by event ID (internal endpoint, to be built with engineering).
  - **New Relic:** errors and logs for the event's time window, used to troubleshoot what might be behind an issue.
  - **Superset:** usage data for the event.
  - **Jira:** status of known issues, so Juno doesn't report something engineering already has.
  - **Slido product knowledge:** public help center, release notes and the team's confirmed "known behavior" notes, covering every integration (PowerPoint Mac and Windows, Google Slides, MS Teams, Zoom), host and participant experience, and plans.
  - **Skill file** (`01-prompting/skill-file.md`): role, rules and the Monday summary format.
  - **5–10 approved past replies:** examples of our tone.
- **Deliberate exclusions:**
  - Billing and payment data. Refunds are out of scope, and this data adds privacy risk.
  - Other customers' accounts. Not needed to answer one customer.
  - Internal Slack conversations. Too noisy and not a reliable source.
  - **[Code]** Personal data (emails, names, phone numbers) is removed from comments and logs before anything reaches the model.
- **SLA of truth:**
  - LaDesk, Jira and New Relic are read live at each run. Juno checks for new feedback every hour during working hours, and the daily batch runs at 17:30.
  - Known-behavior notes and confirmed patterns take effect on the next run after a human approves them.
  - Help center content is refreshed weekly.
- **When a source is unreachable:**
  - **LaDesk down:** fail loudly. No run, and a Slack alert to me. There's nothing to work on without it.
  - **Slido, New Relic, Superset or Jira down:** degrade loudly. Juno still drafts, and the card says "New Relic unavailable, evidence incomplete".
  - **[Code]** Juno never claims a check it didn't run.

## 02 Tools · System Capabilities

- **The verb list:**

| Tool | What it does | Side-effect class |
|---|---|---|
| `read_feedback` | Reads NPS threads from LaDesk | read |
| `get_event` | Gets Slido event details by event ID | read |
| `query_errors` | New Relic errors for the event's time window | read |
| `query_usage` | Superset usage for the event | read |
| `get_jira_status` | Checks whether an issue is already known | read |
| `search_product_knowledge` | Searches help center, release notes and known-behavior notes | read |
| `search_memory` | Finds similar past feedback and patterns | read |
| `draft_reply` | Drafts a reply in the customer's language, with an English translation | draft |
| `draft_jira_ticket` | Drafts an engineering ticket with the evidence attached | draft |
| `draft_weekly_summary` | Drafts the Monday summary in the skill file's format | draft |
| `post_review_card` | Posts a review card to the private Slack review channel | write |
| `save_memory` | Saves a short summary of the item and any "possible" pattern | write |

- **Human-only actions (not Juno tools):** sending a reply, creating a Jira ticket and logging the decision. These run in separate code, with separate keys, only after an approver clicks a button in Slack.
- **Deliberate omissions:**
  - `send_reply`: a sent reply can't be taken back.
  - `refund` / `offer_credit`: money decisions stay with people.
  - `create_jira_ticket`: engineering's backlog should only get tickets a human has checked.
  - `close_ticket`, `delete_ticket`, `retag_ticket`: would hide feedback from the team.
  - `change_account` / `change_event`: no reason for Juno to touch customer settings.
  - Any channel outside LaDesk (email, Slack DMs to customers).
- **Access enforcement [Code]:**
  - Juno's LaDesk API key has only `ticket.read` and `ticket.read_all`.
  - The Slack bot can only post in the review channel.
  - The New Relic key is read-only.

## 03 Loop · AI Costs & Latency

- **Turn ceiling [Code]:**
  - Max 10 tool calls per feedback item and max 2 tries per source. After that, "no signal found" and Juno moves on.
  - Monday summary: one draft plus one retry if verification fails.
  - Why 10: one call per source plus a retry and a memory lookup is about 8. More calls rarely add evidence and only add cost.
- **Escalation triggers** (Juno stops and hands the item to me):
  - Legal, security, privacy, accessibility or abuse: no draft at all.
  - Refund or discount request: flag it, and the draft makes no promises.
  - Missing event ID, sources that contradict each other, or turn ceiling reached: mark "Unclear", don't guess.
  - Live-event failure or large audience: posted within the hour at the top of the queue, not held for 17:30.
- **Outage spikes [Code]:** 5 or more similar feedbacks within 24 hours become one grouped card with one shared draft and the list of UUIDs, instead of separate cards.
- **Latency and cost target:**
  - It's a batch job, not a chat, so latency matters less than cost. Target under 2 minutes per feedback item, all cards in Slack by 17:30, Monday summary by 09:00.
  - Cost: under $0.15 per feedback item and under $3 per Monday summary (placeholders until I know daily volume).
  - **Trade-off:** a cheaper model sorts and flags the hourly feedback, and a stronger model writes replies and the Monday summary. The hourly urgent scan costs more than a once-a-day run, but live-event failures can't wait until evening.

## 04 Memory · Data Requirements

- **What persists, at what scope:**

| Fact | Scope | Expiry |
|---|---|---|
| Evidence gathered for one item | Session (one run) | Deleted after the run |
| Final reply and the human decision (approved / edited / sent to engineering / skipped, who, when) | Team | 90 days |
| Who has already been replied to | Team | 90 days |
| My edits to drafts (used as tone examples) | Team | 90 days |
| What went to engineering, with the Jira link | Team | Until the Jira ticket is closed |
| Short summary of each past feedback (UUID, integration, issue, outcome) | Team | 90 days, rolling |
| Patterns ("possible" or "confirmed") with their UUIDs | Team | Possible: 90 days rolling. Confirmed: until the Jira ticket is closed |
| Known-behavior notes | Team | Until a human removes them |

- **Expiry:** as in the table. The 90 days are a placeholder, long enough to spot a pattern building over a quarter.
- **Write rules:**
  - Juno can only save summaries and "possible" patterns.
  - Only a human can confirm a pattern, mark something as known behavior, or merge or split patterns.
  - A human decision always beats the model, and Juno never overwrites an entry a human wrote.
  - **[Code]** No personal data in memory, only summaries with links back to LaDesk, which stays the source of truth.

## 05 Permissions · AI Risks & Mitigations

| Side-effect class | Tier |
|---|---|
| read | auto |
| draft | auto |
| write | auto for the private review channel and "possible" memory entries; confirm for anything else |
| send | blocked for Juno; only a human click triggers separate sending code |

**Justification:**
- **write → review channel is auto:** only my team sees it, and a card is easy to delete. Requiring confirmation here would make Juno useless.
- **Jira tickets → confirm:** a bad ticket wastes engineering time and erodes trust in Juno's escalations. It's easy to undo, but costly in attention.
- **send → blocked:** a reply reaches a customer and can't be taken back. The blast radius is our relationship with that customer.
- **[Code]** Only people on the approver list can click Approve or Send to engineering. A named backup approver covers when I'm away.
- **[Code]** Drafts older than 2 days are flagged "stale, check before sending", because the event context may have changed.
- **[Code]** One setting switches Juno off.

## 06 Verification · AI Testing & Measurement

- **The check before output ships:**
  - **[Code] checks, must pass 100%:**
    - UUID, rating and integration match the LaDesk record.
    - No personal data in the reply beyond the customer's first name.
    - No internal details in the reply: no New Relic links, Jira keys or internal tool names.
    - Links only go to the public Slido help center.
    - No refund, discount, credit or date promises (keyword check).
    - Monday summary counts come from the decision log, not the model.
    - Every pattern lists at least 3 UUIDs that exist in LaDesk.
  - **[Model] acceptable output must satisfy:**
    - The reply addresses the customer's actual issue, after reading the whole thread.
    - It's in the customer's language, with an English translation for the reviewer.
    - Any root cause is marked "Unconfirmed" unless Jira shows engineering confirmed it.
    - Tone matches the approved examples, with no generic support phrases.
    - The Monday summary follows the skill file's format, quotes comments in full and contains no product recommendations.
    - Bar: at least 80% of drafts approved without heavy edits during the pilot (placeholder; Module 5 defines the rubric).
- **Failure behaviour:**
  - **Code check fails on a reply:** the card still goes to Slack with "Needs a closer look: <failed check>", and Approve stays disabled until the draft is edited.
  - **Model check fails:** the card goes to Slack with a warning saying what looks off.
  - **Monday summary fails:** retry once, then send it with the failed checks listed at the top.
  - Nothing is silently dropped.

## Eval plan

Stub. Module 5 fills this in with the human eval rubric in `05-evals/human-rubric.md`. Starting metric for the pilot: share of drafts approved without heavy edits, compared against replies we actually sent last week.

## Out of scope

- Sending replies, refunds, discounts or credits, creating Jira tickets, closing or re-tagging tickets, changing customer accounts or events, and contacting customers outside LaDesk. None of these are in the verb list, and send is tiered blocked.
- Product recommendations or solutions in any output. That's the product team's job when writing specs.
- Real-time chat with customers.

**Open questions:**
- Can I get a read-only LaDesk key from our admin?
- Does every feedback carry a Slido event ID?
- Who builds the Slido event lookup?
- Who is my backup approver?
- How many feedbacks do we get per day? (needed to set the cost target)
- Will security approve sending feedback to an AI model?
