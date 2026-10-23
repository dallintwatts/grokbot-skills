---
name: fleet-doctor-activity-share
description: >-
  Fleet Doctor bot only. Use when the user asks which bots used the most, wants
  each bot's share of activity (including when they say "usage"), asks for a
  per-bot breakdown or says "activity share". Asks consent, then ranks bots by
  their share of counted steps from their own chat history, an estimate that is
  never billed usage.
---
# Fleet Doctor activity share

The usage meter is account-wide and shows no per-bot split. This skill gives an honest **activity** ranking instead: each bot's share of the model steps it took in a window, read from its own chat history. It is an estimate of relative activity. It is never billed usage.

**When it runs:** only after the user asks for an activity ranking (which bots used the most, a per-bot breakdown, "activity share") and says yes to the consent question below. Never from the audit, a routine or a hand-off on its own.

**The one exception:** fleet-doctor-audit says never split the account total per bot, and Fleet Doctor never reads other bots' files. This skill is the one sanctioned exception to both, and only in this form: it splits activity, never the meter, and it reads chat history only through the transcript tool, read-only, after the user's yes. All other audit rules (honesty, banned phrases, settings and files) still apply.

**Copy rules:** every line you write for this skill uses no semicolons, no em or en dashes and no exclamation marks.

## 1. Consent and scope
- Reading other bots' chat history is reading their conversations. "Which bot uses the most?" is not yet consent. Before reading anything, ask once: "Want me to rank your bots by activity? I'll read each bot's chat history for the window (read-only, nothing is sent or changed)." Skip the question only if the user already said yes to it, or said "activity share", in this conversation.
- If the user says no or doesn't answer: read nothing, don't ask again in this conversation, and offer what needs no reading (the account-wide meter, each bot's routines and settings).
- Window: the user's, else a rolling 7 x 24 hours ending now, in the user's time zone. Never widen it to calendar midnights. Print the exact start, end and zone on every answer.
- Roster: every agent in your teammates or agent directory, plus yourself. The user's pasted list wins if it differs. With no directory, use the user's list and say so.
- Read-only throughout: never message, wake, pause or restart any bot while doing this.
- Bots named "Retired:", group rooms, duplicates and temporary chats stay in the counts, are labelled, and are never shown as a top bot.
- Orchestrators (a bot that relays or coordinates other bots) carry relayed messages in their own history. A bot is an orchestrator only if its name says so, or the user or the roster says so. Never decide it from reading its conversations.

## 2. Collect (per bot)
Use ReadTranscript with `in=<agent id>`, newest first, `limit` 200, paging back with `before` (the first position of the previous page) until the oldest message read is earlier than the window start or the transcript ends. Write each page to a file (write_to_file) instead of loading it into context.
- A page holds at most about 47 KB, so `limit` 200 usually returns 30 to 110 messages. Positions include entries that are never printed, so never map lines to positions. Chain pages only by `before`. A busy bot can need 100+ pages for 30 days, so before a window over 7 days, give the rough page count and offer 7 days first.
- Check every page header. It must read `Transcript of agent "<name>" (<id>)` for the bot you asked for. Anything else ("Transcript of this conversation", another id) is not that bot: discard it.
- Yourself: read your own history with no `in`, and only from your main chat. A helper running this skill gets its own run back instead, so it records you as `no data self`.
- "No transcript recorded yet" means the bot never ran: record `never ran` (zero steps, listed, outside the shares).
- Any other failure (error, "No conversation, agent, or subagent you can read", not a member): record `no data` with the reason. Never count it as zero. Readability depends on who is reading, so re-check rather than copying an old run's `no data`.
- If reading stops before the window start, record `partial (read back to <time>)`. Finish it, or show it as "at least N" outside the ranking. Never rank a partial row as complete.
- The saved pages hold other bots' conversations and land in a folder every agent on this computer can read. Once the counts are saved, delete the pages this run wrote. Keep only the counts.

## 3. Count (script it, never eyeball)
Parse the saved pages with a short script. Each line is one JSON message.
- Read each `<timestamp>` with its UTC offset and convert it to the user's zone. Assign every message to the most recent user-message timestamp at or before it. Messages before the first timestamp in the window are excluded.
- Per bot, inside the window: **steps** = assistant messages (one model response each). Steps are the only ranked metric.
- Total = sum of steps across bots with complete data. Share = bot steps / total, to one decimal. If the total is 0, the answer is "No counted activity in <window>" with no percentages.
- Save `activity/<date>.csv` (bot, steps, share, status, window start, window end, status one of "", orchestrator, retired, no data self, no data, never ran, partial) and the counting script beside it in `settings.folder`. Only counts are saved.

## 4. Build the answer (chat only)
Work these out from the saved counts. Never hardcode a number.
- **Ranked rows:** bots with complete data and steps above 0, not retired, highest steps first (ties keep roster order). Show the top 8. Fold every other counted bot (retired ones and zero-step ones included) into one row "Other <k> bots".
- **Headline:** let n = counted bots with steps above 0.
  - n above 5: "Top 5 bots logged <P>% of counted steps."
  - n is 4 or 5: "Top <n minus 1> bots logged <P>% of counted steps."
  - n is 2 or 3: "<top bot> logged <P>% of counted steps."
  - n is 1: "<bot> logged 100% of counted steps."
  - P = the combined steps of those top bots / total, as a whole percent, held between 1 and 99 unless it covers every counted step. A named bot that is an orchestrator gets "(orchestrator)" after its name.
- **Orchestrator rule:** mark each orchestrator row with † after its name and add the † line. Leave the line out when no orchestrator row is shown.
- **Missing line:** M = never ran + unreadable (no data self + no data), T = every roster bot, N = never ran, K = unreadable.
  - "Missing: <M> of <T> bots. <N> never ran (zero steps, no history at all). <unreadable sentence>, so the counted bots' shares may run high."
  - Unreadable sentence: "The counting bot could not read its own history" when only you are unreadable, "The counting bot could not read its own history, and <K minus 1> other bots' histories were unreadable" when you and others are, "<K> bots' histories were unreadable" when only others are. Use singular forms for 1 ("1 bot never ran", "1 bot's history was unreadable").
  - Drop the never-ran sentence when N is 0 and the unreadable sentence when K is 0. When M is 0, write "Missing: none."
  - "may run high" always attaches to the unreadable bots, never to bots that never ran, since those add zero steps.
- **Partial rows:** list them after the Missing line as "Partial: <bot> at least <steps> steps (read back to <time>)."

Send exactly this shape:
```
<headline>
Step counts. Share of activity, not billed usage.
<N days>, <start date, year, time> to <end date, year, time> <zone>
<total> steps · <counted> of <T> bots counted

1. <bot>< †> · <p>% · <steps> steps
... up to 8 rows
Other <k> bots · <p>% · <steps> steps

Not cost: a step is one model response. Long chats and bigger models cost more per step.
† Orchestrators overstated: messages they relay count as their steps.
Not counted: work a bot hands to helpers or outside tools, so the bots that hand off are understated.
Missing: <built as above>
Source: each bot's own chat history, counted by Fleet Doctor. Built by Dallin. One person's fleet, <one week | N days>.
```
- With no counted activity, send the headline "No counted activity in this window", the label, window and totals lines, then only the Not cost, Missing and Source lines.
- Name missing bots by status only. In the user's own chat, use real bot names. If the user wants to share it publicly, offer generic labels (Bot A, Bot B) unless they name the bots to show.
- Next, one line: say "audit now" for fixes.

When the user asks "what % of my usage is X" (or cost, credits, spend), answer in this shape and no other:
"I can't split billed usage by bot, because the meter is account-wide. By activity, X took <N> of <M> counted steps (<P>%) from <start> to <end> <zone>. That's an estimate of activity, not billed usage<, and it leaves out ...>."

## 5. Hand-off to the audit
- The ranking feeds fleet-doctor-audit only as an input to the **burn band**: a bot in the top quarter of activity share with a Low band gets "<bot>: high activity, check its routines" in Unknowns. Never overwrite a band from shares, and never show a share on the audit card.
- If the user wants this every week, offer to add it to the weekly checkup rather than adding a new routine.

## Never
- Never write "<bot> used X% of your usage" or any phrasing that equates activity with billed usage, cost, credits or savings.
- Never invent a count or fill a `no data` or `partial` row with a guess, or with another run's number.
- Never send, quote or summarize one bot's conversation content to another bot or anyone else. Only counts leave this skill.
