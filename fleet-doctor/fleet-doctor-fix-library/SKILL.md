---
name: fleet-doctor-fix-library
description: >-
  Fleet Doctor bot only. Use to map audit findings to waste codes W1-W10, pick
  the lead safe, reversible fix F1-F7 with paste-ready wording, and when the
  user approves ("yes 1", "yes 1b", "yes 1c"), reverts ("revert C12") or
  changes Fleet Doctor's own routine.
---
# Fleet Doctor fix library

**Crisis first**: if a message signals crisis or risk of harm, follow fleet-doctor-audit's crisis rule before anything here; write nothing that turn.

## Principles (what a cheap routine looks like)
- Every scheduled wake costs usage. Pick the coarsest cadence that still lands before the user reads the result.
- Keep the covered hours. A coarser cadence runs less often over the SAME hours and days. Narrow them (weekdays, waking hours) only after asking once, "Does anyone need <bot>'s results overnight or on weekends? If yes or not sure, I'll keep the same hours.", and only if the user explicitly says no one needs those hours. A yes to F2 alone is not that. Never silently drop nights or weekends.
- Use an event trigger instead of a timer when the event exists. Avoid polling every 15 minutes.
- Stay silent when nothing changed. "No updates" filler is still a wake.
- A long chat makes every routine run reread the thread. Recurring jobs belong on a bot with a short chat.

## Waste codes (severity 1-3)
Codes stay in the findings file; the Waste cell on cards uses only the card words.
| Code | Pattern | Detect | Sev | Card words |
|---|---|---|---|---|
| W1 | Runs more often than once an hour; or overnight/weekends *only if the user said no one reads it then* | schedule (+ the user's own words for the second clause) | 3 | the cadence ("every 15 min"); "runs overnight/weekends nobody reads" |
| W2 | Posts even when nothing changed ("no change" filler) | routine lacks stop-if-unchanged, or self-report says no | 3 | posts even when nothing changed |
| W3 | Routine on a long-chat bot | chat length long + any routine | 2 | long chat |
| W4 | Two bots doing the same job | same source and job in inventory | 2 | duplicate job |
| W5 | Bots waking bots with status chatter | messages other bots every run | 2 | chatty hand-offs |
| W6 | Browser loops or retries | browser every run, or reported retries | 3 | browser loop |
| W7 | Overlapping personas | several general assistants | 1 | overlapping assistants |
| W8 | Stale but armed (owner silent 30+ days) | last-talked date | 2 | owner silent 30+ days |
| W9 | Event trigger on a busy channel | trigger type + volume | 3 | busy trigger |
| W10 | Cloud agent launches (may bill separately: see the usage screen or official docs; state as fact only with a link fetched this run) | self-report | 1 | cloud agent launches |

W1's overnight/weekends clause never comes from guessed reading habits. If a schedule runs overnight or on weekends and the user hasn't said, don't tag it; add "<bot>: needed overnight/weekends?" to Unknowns.

## Fixes: the lead fix comes from this table
Only safe, reversible fixes lead. Use this table so every run leads the same way:

| Bot's codes | Lead ("yes N") | Gentler ("yes Nb") |
|---|---|---|
| W1 | F2 | F1, if W2 is also present |
| W9 | F5 | F1, if W2 is also present |
| W2 | F1 | – |
| W5 | F4 | – |
| W3 | F3 | – |
| W6 | F3 (script the site, cap retries) | – |
| W4, W7 or W8 | F6 or F7 (proposal only) | – |
| only W10 | none (information only) | – |

With several codes, the lead is the first matching row in the lead table, read top-down. Risk order (F1 < F4 < F2 < F5 < F3) breaks ties only when no row applies. The scheduled-run delta is for the effect line, never for choosing the lead. Offer F1 as "Nb" when W2 is also present and F1 isn't the lead. Example: a bot running every 15 minutes, 24/7, that posts even when nothing changed (W1+W2) leads with F2 hourly over the same hours (672 → 168 scheduled runs/wk), and "yes 1b" = F1 first (zero risk). F6 and F7 are last-resort proposals (pause, never delete). Never raise or change a limit. Never propose deleting anything.

**Three options per bot, at most** (the only ones shown in chat; NEXT ACTION shows only N and Nb, the table's Fix cell may add Nc):
- **"yes N"** = the lead fix.
- **"yes Nb"** = the gentler version of the lead fix (an alternative to N, from the Gentler column), if any. Card wording: "gentler: <fix>".
- **"yes Nc"** = the fix for a SECOND severity-3 code (W1, W2, W6, W9) on the same bot, when that code's own lead fix (from the table) differs from both N and Nb. It is an addition, not an alternative: the user can approve N (or Nb) and Nc together. Card wording: "also: <fix>". Examples: W1+W6: "yes N" = F2 coarser cadence, "yes Nc" = F3 script the site, cap retries. W2+W6: "yes N" = F1 stay-quiet line (W2's row comes first), "yes Nc" = F3. W1+W2+W6: N = F2, Nb = F1, Nc = F3.
- Any further fixes (a third severity-3 code, or severity 1-2 codes with a different fix) go to the findings file only.

Pastes are plain language, never raw cron: the target bot knows its own routine and picks the minute. Name the routine as the user did; if its name is unknown, write "your routine that checks <what it checks>" in place of "your <routine> routine".
- **F1 Stay-quiet line** (W2). Card: "stay-quiet line". Paste: "Update your <routine> routine: if nothing changed since your last run, end silently without posting. Change nothing else." Risk: none. Revert: remove that line.
- **F4 Quiet hand-offs** (W5). Card: "quiet hand-offs". Paste: "Only message other bots when they need to act. No status updates. Change nothing else." Risk: less visibility. Revert: remove that line.
- **F2 Coarser cadence** (W1). Card: "hourly, same hours" (or the step chosen, e.g. "every 2 hours, same hours"). Paste: "Change your <routine> routine to run <new step> instead of <current step>, same hours and days. Change nothing else." Default: same hours and days, coarser step. If the schedule runs 24/7, the reply to the yes is the one-line echo ("yes 1 = <bot>: hourly, same hours.") plus the Principles question, no paste yet; then give one paste, opening with the answer instead of the echo ("Keeping the same hours. Send this in <bot>'s chat:"): same hours, or on an explicit "no one needs nights/weekends" the narrowed hours ("…to run <new step> on <kept hours> instead of <current step> around the clock. Change nothing else."). An unclear answer gets the same-hours paste. Never two pastes back to back. Risk: fewer updates; if narrowed, none overnight/weekends. Revert: the saved schedule, in the same plain style.
- **F5 Timed digest instead of a busy trigger** (W9). Card: "timed digest". Paste: "Stop running your <routine> routine on every <event>; instead run it <every 2 hours | at 9 AM and 5 PM> with a digest of everything since your last run. Change nothing else." Risk: delay. Revert: the saved trigger, in plain words.
- **F3 Fresh short chat / script it** (W3, W6). Card: "move to a fresh short chat" (W3) or "script the site" (W6). For W3 there is no paste; give the user these steps: "Create a new bot for <routine>, copy <bot>'s standing instructions into its profile, recreate the routine there, then pause the old routine (don't delete it)." Risk: loses thread context. Revert: resume the old routine and pause the new one. For W6 a fresh chat alone doesn't stop browser loops, so the paste is: "Instead of clicking through <site> each run, fetch it once with a script or its API, and stop after one failed retry. Change nothing else." Risk: a script can break if the site changes. Revert: remove that line.
- **F6 Merge, proposal only** (W4, W7). Card: "pause this one, keep <other>", on the bot that runs more (one number for the pair; the other row's Fix is "– (covered by "yes N")"; "Keep this one instead" swaps them). Paste (send in that bot's chat): "Pause your <routine> routine until I say resume. Don't delete it. Change nothing else." Risk: an edge case only the paused bot covered. Revert: "Resume your <routine> routine."
- **F7 Pause instead of delete, proposal only** (W8, or whenever the user asks to delete or drop a bot). Card: "pause until needed". Offered under its existing number if one was offered; otherwise the next unused number. Paste (send in <bot>'s chat): "Pause your <routine> routine until I say resume. Don't delete it. Change nothing else." The same reply gives the resume paste ("To resume later, send: "Resume your <routine> routine.""). **F7 relief** (fleet-doctor-audit F relief line): the reply to its yes is only "Does anyone need <bot>'s results before <date>?"; only on "no", give the pause paste plus the resume paste ("After <date>, send: "Resume your <routine> routine.""; reset unknown again: "Keep it paused until the reset is known again, then send: …", never an old date); yes or unsure: no pause, point to its regular fix. A relief pause is that reply's one change: no other fix's paste in it; the next checkup can offer the next fix. Never create a routine to resume it; on the first checkup after the reset, if no resume is recorded in `changes.jsonl`, ask once "Did you resume <bot>?" Risk: its results stop until resumed. Revert: "Resume your <routine> routine."
- **A1 Ask about hours** (hours not given and its 24/7 band would be High: fleet-doctor-audit D). Card: 'ask: needed overnight/weekends? ("yes N" asks it)'. The reply to its yes is only: "Does <bot> run overnight or on weekends, and does anyone need its results then? If yes or not sure, I'll keep the same hours." Then re-band with the answer; a fix that now applies (F2 same hours, or narrowed hours only on an explicit "no one needs them") is offered under the next unused number. If that fix needs hours still unknown, ask once on its yes, before any paste. Nothing is logged until a paste is given.
- **W10**: information only ("these launches may bill separately; see your usage screen or official docs"). State it as fact only with a link to official docs fetched this run.

Every proposal states the fix and, in one line, the effect (estimate band or scheduled-run delta) and the risk. State an unread-runs effect as a condition ("Then its overnight and weekend runs go unread"), never a counted fact.

## Approve
- **Numbers stay stable**: a number offered keeps its meaning until that fix is applied, declined or superseded; a new fix takes the next number never used in this install, and a number is never reused. Record them in settings `fix_numbers` (`{"1":{"bot":"...","fix":"...","state":"offered|applied|declined|superseded"},...}`; missing = `{}`; applied = a change logged and not `not sent`; superseded = a later report gives that bot a different fix, said once: ""yes 3" is now "yes 5""). A later report shows the same number for the same offer; a revert sets it back to offered. A yes to a closed number: say what happened to it and name the open one. When the user's yes could be read two ways, restate each option in full and ask.
- "yes N" approves fix N's lead fix. "yes Nb" approves its gentler fix (instead of N). "yes Nc" approves its "also" fix (in addition to N or Nb; "yes 1 and 1c" approves both, each logged as its own change). For another bot, a yes means "I'll give you the paste": nothing changes until the user sends it.
- **The reply to a yes**, nothing more: the echo ("yes 2 = Scout: stay-quiet line. Send this in Scout's chat:"; after the hours question, the answer replaces it), one paste, one line of effect and risk, the undo line (Apply 4), and at most one question (on the first paste, that can be the checkup offer, fleet-doctor-getting-started section 3). The self-report prompt only on request.
- "yes all", or a second yes in the same week: confirm the list once and offer pacing: "Apply 1, 2 and 4 now, or one per week, so a meter change can likely be tied to one fix?"
- Vague ("sure", "fix it"): ask which number.
- Refuse even with a yes: delete a bot; raise, change or remove any limit; remove a responsibility or permission; narrow covered hours unless the user explicitly said no one needs those hours (a yes to F2 alone is not that); weaken a deadline. Say: "I won't do that; it stays your call in the app." Then offer the safe alternative (under its existing number if it has one): F7 pause, or for covered hours, the Principles question.
- Change IDs (C12) appear only in undo and redo hints, never in other prose ("you reverted the hourly change").

## Apply (in this order)
1. **Snapshot first** to `backups/<date>/<change-id>-<bot>-<routine>.md`: prior schedule or trigger, routine text, description, exactly as known, plus `source:` (paste, self-report, own, user's description) and `completeness:` (full when the old value of what this change touches is known, e.g. the old schedule for F2; else partial).
2. **Fleet Doctor's own routine**: change it with your routine tool. The app may show a confirm card; never retry a declined change.
3. **Any other bot (v1)**: give the user the exact paste text and say "Send this in <bot>'s chat." No up-front warning about partial snapshots. Never message another bot to change it.
4. **Log** to `changes.jsonl` with the next change ID: `{"id":"C12","date":"...","bot":"...","finding":"W2","fix":"F1","before_ref":"backups/<date>/C12-...","completeness":"full","after":"...","approved_at":"...","applied_by":"user-paste|self","status":"sent?|not sent|confirmed (user)|confirmed (self-report, unverified)","revert":"<exact paste text or action>"}`. A paste the user says they never sent is exactly `not sent` (never "not sent (user)"): its number goes back to offered, the bot leaves "in flight", and a new yes re-gives the same paste under a new change id. A `user-paste` change stays `sent?` (card: "you said you'd paste it") until the user confirms it (`confirmed (user)`) or the next inventory's self-report shows it (`confirmed (self-report, unverified)`). Your own routine changes are `confirmed (user)` once the tool reports success. End the reply with the undo line: "To undo, say "revert C12" and I'll give you a paste that puts it back to <old value in plain words>." (your own routine: "…and I'll put it back to <old value>.").

## Verify (next audit)
"Since last change" keeps the status wording (fleet-doctor-audit F) with its undo hint. Once a comparable reading exists after a change landed, only the Measured account change line judges it (A.5): "likely from one change: <bot>'s <fix>" (exactly one change in between AND the user says their own use was similar AND the bot was Medium or High and the fix cut its scheduled runs: F2, F5 or F6; never a relief pause: "can't compare: relief pause in effect"), "can't tie it to one change" (+ " (that fix is too small to explain it)" when that last part fails), or "can't compare: <why>". Never say a fix caused a change.

## Revert ("revert C12")
Read change C12. Give the exact revert paste text in the same plain style ("Send this in <bot>'s chat."), or restore your own routine (the app may show a confirm card; never retry a declined change). If C12 is still `sent?`, lead with: "If you never sent the <hourly> paste, there's nothing to undo; if you did, send this:". Log a new line with its own ID and `"reverts":"C12"`; its hint reads 'To redo it, say "yes N"' (N = that fix's stable number). Only if the snapshot is partial, say what can't be restored; a schedule the user described counts as known. The user's "revert C<n>" is the yes for that revert.
