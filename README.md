# 📅 Calendar Copilot

**A conversational calendar assistant that lives in Discord** — built with n8n, Claude, Google Calendar, and Google Sheets.

Talk to your calendar in plain English. Get a morning briefing every day. And when an event ends, the bot actually checks in and asks: *did you do it?*

This was my first n8n project — five workflows that work together as one stateful assistant.

![Architecture](images/architecture.svg)

## ✨ What it does

- 🌅 **Morning briefing** — every day at 8 AM, the bot posts a friendly summary of today's calendar events to Discord, formatted by Claude.
- ✅ **Accountability check-ins** — when a calendar event ends, the bot notices and asks whether you completed it. Reply `done`, `postpone`, or `cancel`. If you ignore it, it reminds you (and keeps count).
- 💬 **Natural language calendar control** — just type what you want:
  - *"add study tomorrow 3pm for 2 hours"*
  - *"move study to 4pm"*
  - *"delete study"*
  - *"what do I have on Friday?"*
- ⏩ **Smart rescheduling** — postpone an event and tell it *"tomorrow 3pm"*; Claude parses the date and a new calendar event is created automatically.
- 🔗 **URL saver** — drop any link in the channel and it gets captured to a spreadsheet, with a daily 7 PM digest of everything you saved.

## 🧠 How it works

The system is five n8n workflows sharing state through Google Sheets:

### Flow 1 — Morning Briefing
Schedule trigger (8 AM) → fetch today's Google Calendar events → Claude formats them into a friendly briefing → post to Discord.

![Flow 1](images/flow1-morning-briefing.png)

### Flow 2 — Event Check-in Poller
Every 30 minutes (waking hours only, 7 AM–11 PM): finds calendar events that just **ended**, dedupes them against an `event_tracker` sheet, and sends a Discord check-in. Events that are still unanswered get escalating reminders with a reminder counter.

![Flow 2](images/flow2-checkin-poller.png)

### Flow 3 — Discord Reply Handler
Every 2 minutes: reads recent Discord messages, and when you reply to a check-in, Claude classifies the intent (`done` / `postpone` / `cancel`). The bot then annotates the original calendar event (✅ / ⏩ / ❌), updates the tracker sheet, and — if you postponed — starts a conversation to reschedule.

![Flow 3](images/flow3-reply-handler.png)

### Flow 4 — Natural Language Handler
Every 2 minutes: any free-form message (that isn't a keyword reply or a URL) goes to a Claude-powered router that classifies it into `add_event`, `edit_event`, `delete_event`, `list_events`, `reschedule_date`, or `unclear` — extracting titles, dates, and durations — then performs the calendar operation and confirms in Discord. Processed message IDs are stored in an `app_state` sheet so nothing is handled twice.

![Flow 4](images/flow4-natural-language.png)

### Flow 5 — URL Saver
Daily at 7 PM: scans the day's messages for links, appends them to a sheet, and posts a digest of everything you saved.

![Flow 5](images/flow5-url-saver.png)

### Bonus — Conversational AI Agent
A later iteration that consolidates the message handling into a single agent workflow: one entry point reads Discord messages, routes through an AI agent with memory, and fans out to every action branch (create / update / delete / list events, save URLs, reschedule postponed events).

![Conversational AI Agent](images/conversational-ai-agent.png)

## 🗂 State machine

Each tracked event moves through a simple lifecycle stored in Google Sheets:

```
pending ──▶ awaiting ──▶ done
                │
                ├──▶ postpone_pending ──▶ postponed  (+ new calendar event)
                │
                └──▶ cancelled
```

Google Sheets acts as the database: an `event_tracker` tab for the check-in state machine, an `app_state` tab for deduplication, and a tab for saved URLs.

## 🏗 System design & decisions

### The shape of the system

Everything is **polling, not webhooks**. The n8n instance runs on localhost with no public URL, so Discord can't push events to it — instead, schedule triggers pull recent messages every 2 minutes (reply handling, natural language) or 30 minutes (event check-ins). The tradeoff is latency and API quota in exchange for zero exposed surface area and no tunnel to babysit. This one decision shaped almost every hard problem below: **a poller sees the same data over and over, so every flow has to answer "have I already handled this?"**

Google Sheets is the database. Not because it's the best database, but because state you can *see* is worth a lot when you're debugging a distributed set of workflows — you can watch a row flip from `pending` to `awaiting` to `done` in real time, and fix bad state with a click.

Claude is used narrowly: it classifies intent and parses dates, returning strict JSON. It never decides *what happens* — the switch nodes and code nodes do. Keeping the LLM at the edges (parse in, format out) means the state machine stays deterministic and testable.

### The duplicate problem

Polling creates duplicates by default. Discord returns the last N messages *every* poll, and the calendar returns the same ended events *every* poll. Left unhandled, the bot would re-ask about the same event every 30 minutes and re-execute the same command every 2 minutes, forever. Each flow needed its own dedup strategy:

**1. Calendar events → dedup by primary key (Flow 2).**
Every ended event gets a row in the `event_tracker` sheet keyed by Google Calendar's `event_id`. On each poll, the flow merges today's ended events with existing sheet rows and keeps only events whose `event_id` is not already present:

```js
const newEvents = calendarEvents.filter(e => !existingIds.includes(e.event_id));
```

An event gets exactly one row for life, and the check-in conversation runs off the row's `status` — so the calendar can be re-read a hundred times without a second check-in.

**2. Discord replies → dedup by time window + keyword whitelist (Flow 3).**
Replies like `done` have no natural key, so the flow only accepts messages that are (a) not from the bot itself, (b) an exact keyword match (`done` / `postpone` / `cancel`), and (c) **newer than 3 minutes** — just wider than the 2-minute poll interval, so a message is seen by roughly one poll. Even if a reply slipped through twice, the state machine absorbs it: once the row flips to `done`, there's no `awaiting` event left for a duplicate reply to act on. Idempotence as a backstop.

**3. Free-form messages → dedup by watermark (Flow 4).**
Natural language commands can't be keyword-filtered, so Flow 4 keeps a *last processed message ID* in an `app_state` sheet and skips anything already consumed (`m.id !== lastProcessedId`), writing the new ID back after each run. A stale-read window still exists between the read and the write — the 3-minute freshness filter covers most of it.

**4. URLs → dedup by content (Flow 5).**
Before appending a link, the flow checks whether that URL already exists in the sheet and branches to "already saved" instead of inserting a second row. Dedup by value rather than by message, since the same link can be posted twice.

**5. Flows competing for the same message → partition the message space.**
Three flows poll the same channel, so one message could trigger all of them. The filters are mutually exclusive by design: keyword replies belong to Flow 3, URLs are excluded from Flow 4's filter (`!m.content.match(/https?:\/\//)`) and belong to Flow 5, and everything else falls to Flow 4. Exactly one flow claims each message.

**6. The postpone trap → dedup by conversation timestamp.**
Subtle one: after you reply `postpone`, the bot asks "when?" — but your original `postpone` message is still in the recent-messages window and could be misread as the answer. The fix: when the bot asks, it stamps `messaged_at` on the row (status `postpone_pending`), and the date parser rejects any reply older than that stamp:

```js
if (replyTime <= askedAt) return [{ json: { error: 'no_new_reply' } }];
```

Only words spoken *after* the question count as the answer.

### Error handling

**Errors are data, not exceptions.** Code nodes never throw; they return sentinel objects like `{ error: 'no_awaiting_event' }` or `{ skipped: true }`, and an IF gate downstream routes error items to a dead end. The branch simply stops instead of failing the whole execution — important when one workflow run handles several independent things at once.

**Empty is a valid state, not a failure.** n8n silently halts a branch when a node outputs zero items, which can mask "no data" as "nothing happened." Nodes that read the calendar or sheets set `alwaysOutputData` so an empty read still flows downstream as an explicit empty list — that's how the morning briefing can say "no events today!" instead of never posting at all.

**Side effects are guarded against fan-out.** In n8n, a node runs once *per input item* — so a sheet-read node fed 3 items reads the sheet 3 times, and a Discord node fed 3 items posts 3 messages. Shared reads set `executeOnce` to collapse that fan-out, which is both a quota saver and a duplicate-message guard.

**LLM output is treated as untrusted input.** Prompts demand bare JSON with no markdown, but the parsing code still strips code fences before `JSON.parse` because models add them anyway. Every classifier has an `unclear` escape hatch that routes to a "didn't catch that, try rephrasing" reply — an unparseable human message is a normal case, not an error.

**Lookups fail loudly to the user, silently to the system.** If you say *"delete study"* and no event matches, the code returns `{ error: 'event_not_found', title }` and the flow replies "couldn't find that event" — no crash, no silent no-op.

**Time is pinned, not assumed.** The Docker container runs on UTC, so every "today", "just ended", and waking-hours check explicitly converts to `America/Los_Angeles`. Nothing trusts the server's local clock — the difference between a briefing at 8 AM and one at midnight.

### What I'd change now

- **Webhooks over polling** — a Discord gateway bot or n8n webhook endpoint would remove the entire class of "have I seen this?" problems and the 2-minute reply latency.
- **A real database** — Sheets served its purpose as a transparent learning tool, but row-number-based updates are fragile and there are no transactions; the watermark race window in Flow 4 is exactly the kind of thing a database `UPDATE ... WHERE` closes.
- **One workflow instead of five** — the Conversational AI Agent iteration above was the beginning of that consolidation: one entry point, one router, shared memory.

## 🛠 Stack

| Piece | Role |
|---|---|
| [n8n](https://n8n.io) (self-hosted, Docker) | Workflow orchestration & scheduling |
| Claude Haiku 4.5 (Anthropic API) | Intent classification, date parsing, message formatting |
| Discord bot | Chat interface |
| Google Calendar API | Event source & CRUD target |
| Google Sheets API | Lightweight state store |

## 📝 Notes

- All schedules respect waking hours (7 AM–11 PM PT) so the bot never pings at 3 AM.
- The workflow definitions aren't published here — this repo documents the design and behavior.
