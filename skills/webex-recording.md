---
name: webex-recording
description: Find and extract Webex recordings. Discovery mode ("since 24h") lists new calls from both your own recordings and recordings shared with you by email; extraction pulls transcript, summary, chapters, and action items. Recordings you own need no browser; recordings shared with you need Chrome on --remote-debugging-port=9222.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Bash
  - Bash(curl *)
  - Bash(jq * ~/projects/webex-agent/.webex_token.json)
  - Read(~/projects/webex-agent/.webex_token.json)
  - Write(~/projects/webex-agent/.webex_token.json)
  - mcp__chrome-devtools__navigate_page
  - mcp__chrome-devtools__list_pages
  - mcp__chrome-devtools__new_page
  - mcp__chrome-devtools__take_snapshot
  - mcp__chrome-devtools__fill
  - mcp__chrome-devtools__click
  - mcp__chrome-devtools__press_key
  - mcp__chrome-devtools__wait_for
---

# /webex-recording — Find and Extract Webex Recording Content

Extract transcript, AI-generated summary, chapters, and action items from a Webex recording.
Discovery mode finds recordings you haven't processed yet across both sources.

## Dependencies

| Dependency | Purpose | Required? |
|-----------|---------|-----------|
| Webex REST API token with `spark:recordings_read` + `meeting:recordings_read` | List recordings you own and download their transcripts directly. Any stored OAuth token works; this skill reads one from `~/projects/webex-agent/.webex_token.json`. | For owned recordings and for discovery mode |
| [Chrome DevTools MCP server](https://github.com/anthropics/claude-code-chrome-devtools) | Browser automation to navigate Webex, enter passwords, and scrape content | Only for recordings **shared** with you |
| Google Chrome | Runs with `--remote-debugging-port=9222` for CDP access | Only for recordings **shared** with you |
| Email CLI or MCP tool that can search and read your mailbox | Find recordings other people hosted and shared. This skill uses a Microsoft Graph CLI; any equivalent works. | For shared recordings and for discovery mode |

## The two sources — read this before anything else

A recording reaches you one of two ways, and they do not overlap:

| Source | What it covers | How to reach it |
|--------|----------------|-----------------|
| **Owned** — calls you hosted | Anything recorded on your own account | Webex REST API. Returns a recording ID, duration, password, and a **direct transcript download**. No browser. |
| **Shared** — calls someone else hosted | Someone else's meeting, forwarded or auto-mailed to you | Email only. Requires the browser path — you don't own the recording, so the API cannot see it. |

**Checking one source misses roughly half your calls.** The API will never return a recording
from someone else's personal room; email will rarely carry a clean record of a call you hosted
yourself. Discovery mode must always query both.

## Prerequisites

**For owned recordings:** just the Webex API token described above.

**For shared recordings** (and only those), Chrome must be running with remote debugging:
```bash
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
  --remote-debugging-port=9222 \
  --user-data-dir=/tmp/chrome-debug \
  --remote-allow-origins='*'
```

The `chrome-devtools` MCP server must be configured in your Claude Code MCP settings (e.g.
`~/.mcp.json` or a project `.mcp.json`). Don't check for Chrome until you know the selected
recording actually needs it — most won't.

## Arguments

`$ARGUMENTS` can be:
- **A time window** → discovery mode. `since 24h`, `since 3d`, `since 2w`, or bare `24h` / `today`.
  Lists new calls from both sources and asks which to extract. **This is the common case.**
- A Webex recording URL (e.g., `https://cisco.webex.com/cisco/ldr.php?RCID=...`)
- An email search query to find one specific recording (e.g., `from:colleague weekly demo`)
- Empty — ask whether the user wants a window, a URL, or a search

## Step 0. Discovery mode — "what came in?"

Run this when `$ARGUMENTS` is a time window. Query both sources, merge, then let the user pick.

**Resolve the window** to an ISO-8601 UTC pair. Compute it — never hand-write a date:
```bash
FROM=$(python3 -c "import datetime;print((datetime.datetime.now(datetime.timezone.utc)-datetime.timedelta(hours=24)).strftime('%Y-%m-%dT%H:%M:%SZ'))")
TO=$(python3 -c "import datetime;print(datetime.datetime.now(datetime.timezone.utc).strftime('%Y-%m-%dT%H:%M:%SZ'))")
```

**a. Owned recordings** — the Webex API:
```bash
TOKEN=$(jq -r '.access_token' ~/projects/webex-agent/.webex_token.json)
curl -s "https://webexapis.com/v1/recordings?from=${FROM}&to=${TO}&max=50" \
  -H "Authorization: Bearer ${TOKEN}"
```
On **401**, refresh the token exactly as `/webex-search` does — same token file, same client
credentials — and retry once. Read `topic`, `createTime`, `durationSeconds`, `id`, `password`.

**b. Shared recordings** — email. The Graph CLI has no date flag, so the filter goes inside the
KQL query. `received>=` takes a plain `YYYY-MM-DD`:
```bash
~/.config/claude-graph/bin/msgraph email search 'webex recording received>=YYYY-MM-DD' --max 25
```
Widen the window by a day here — mail arrives after the call. Also try
`'"meeting content is available" received>=YYYY-MM-DD'`, which is the subject Webex itself
sends; the two queries catch different things, so run both and pool the results.

Recording passwords are often sitting in plain sight in the `body_preview` — grab them now
rather than re-reading the message later.

**c. Merge and dedupe.** Three passes, in order:

1. **Same call in both sources.** Match on topic similarity plus a date within ~1 day. Webex
   suffixes topics with `-YYYYMMDD HHMM-N`; strip that before comparing. Keep the **owned**
   record — it has the ID and the direct transcript.
2. **Already processed.** Glob `~/claude-memory/recording-*.md` and drop anything
   already captured. The filename's `-YYYY-MM-DD` and the note's `description` are enough to
   recognise a call. This is the dedupe ledger — there is no separate state file to maintain,
   and that is deliberate.
3. **Noise.** Drop calendar invitations, "here's the deck" follow-ups, and mail that merely
   mentions a recording without linking one.

**d. Present the list.** Newest first, numbered, with source marked:

```
Calls since <window> — <N> new, <M> already captured (hidden)

1. [owned]  2026-03-10  15m  Architecture sync
2. [shared] 2026-03-11  62m  Program weekly cadence        (from A. Colleague)
3. [shared] 2026-03-12  48m  Partner review                (from B. Colleague)
```

Then ask which to extract. **One at a time** — a full transcript is a lot of context, and
batching them produces worse notes on all of them. If the user names several, do them in
sequence and confirm each note before moving on.

State the hidden count but don't list the hidden calls. If the window turns up nothing, say so
plainly and give the date of the most recent recording so the user knows the query worked and
the window was simply empty.

## Steps

### 1. Get the recording — pick the path

**If it came from discovery mode as `[owned]`** — you already have the recording ID. Go
straight to **step 1a**. Skip the browser entirely.

**If it came from discovery mode as `[shared]`** — you have the email. Extract the URL and
password from it as below, then continue to step 2 (browser).

**If `$ARGUMENTS` is a URL:**
- Use it directly
- Ask the user for the password (or check if provided after the URL)
- If it's a `cisco.webex.com` link to a call the user hosted, it's worth one API lookup
  first — the direct transcript is faster and needs no browser

**If `$ARGUMENTS` is an email search query:**
- Search email via: `~/.config/claude-graph/bin/msgraph email search "<query>" --max 1`
- Get the message ID from the search result
- Read the full message: `~/.config/claude-graph/bin/msgraph email read <message-id>`
- Parse out `href` URLs matching `webex.com` patterns (ldr.php, recordingservice, etc.) from the HTML body
- Extract password from email body text (look for "Password" label followed by text)

**If `$ARGUMENTS` is empty:**
- Ask: "A time window (`since 24h`) to see what's new, a Webex recording URL, or an email search query?"

### 1a. Owned recording — pull the transcript directly (no browser)

Fetch the detail record. `temporaryDirectDownloadLinks` expires, so download in the same pass:
```bash
TOKEN=$(jq -r '.access_token' ~/projects/webex-agent/.webex_token.json)
curl -s "https://webexapis.com/v1/recordings/<RECORDING_ID>" -H "Authorization: Bearer ${TOKEN}"
```

This gives `topic`, `timeRecorded`, `durationSeconds`, `password`, and
`temporaryDirectDownloadLinks` with `transcriptDownloadLink`, `audioDownloadLink`, and
`recordingDownloadLink`.

Download the transcript to a real path — not `/tmp`, which gets swept:
```bash
mkdir -p ~/claude-memory/transcripts
curl -sL "<transcriptDownloadLink>" -o ~/claude-memory/transcripts/<slug>-YYYYMMDD.vtt
```

The result is WebVTT with **real speaker names** in each cue header:
```
1 "Einar Nilsen-Nygaard" (3539300608)
00:00:00.000 --> 00:00:17.813
Text of the first cue...
```
Parse cue headers for the speaker, the first timestamp for the offset, and merge consecutive
cues from the same speaker into one turn.

**If `transcriptDownloadLink` is absent**, transcription was off for that call. Say so and offer
the browser path — the Webex AI summary may still exist on the playback page even when no
transcript file does. Don't silently fall through.

Then **skip to step 6.** Chapters and the AI summary are browser-only; note their absence
rather than leaving blanks that read like the call had none.

⚠️ Two things the transcript will do that look like content but aren't:
- **Webex profanity-masks with `*******`.** Leave the mask; don't reconstruct the word.
- **Acronyms get auto-corrected into nonsense.** Confirmed corruptions: `ICC` → "International
  Criminal Court", `TEAP` → "teep", `ISE` → "ICE". Correct these silently in any summary you
  write. If a **codename** is garbled inconsistently across cues, flag it as unconfirmed and ask
  — don't pick whichever spelling appeared most. ("Gangplank" arrived as *gang blank*,
  *Ganglank*, and *gang Black* in one transcript.)

### 2. Navigate to the recording

*(Shared recordings only — skip if step 1a already got the transcript.)*

1. Check `mcp__chrome-devtools__list_pages` to confirm browser is connected
2. Navigate to the recording URL via `mcp__chrome-devtools__navigate_page`
3. Take a snapshot to check current state

### 3. Enter password (if required)

1. If snapshot shows a password prompt:
   - Find the password textbox element
   - Fill it with `mcp__chrome-devtools__fill`
   - Click OK or press Enter
   - Wait for page to load (take snapshot to confirm)
2. If snapshot shows a login page (SSO):
   - Tell the user: "Please log in manually in the Chrome window, then say 'continue' when you're on the recording page."

### 4. Handle cookie consent

If a cookie banner appears, click "Accept" to dismiss it.

### 5. Extract content

*(Shared recordings only.)* Once on the recording playback page, extract in this order:

**a. Chapters** (from the chapters panel):
- Look for chapter list items in the snapshot
- Extract: chapter number, timestamp, title

**b. Summary** (from the Summary tab):
- Click the Summary tab if not already selected
- Extract: Notes section, Action items section

**c. Transcript** (from the Transcript tab):
- Click the Transcript tab
- The transcript will be in the snapshot as StaticText elements
- Parse into structured format: `[timestamp] Speaker: text`
- Group by speaker turns

### 6. Assign projects

Before saving, decide which of the user's workboard projects this call is about. Read
`~/claude-memory/workboard.md` and use the project names exactly as they appear
there.

This is a judgment call, not a keyword match. You've just read the transcript — you know what
the call was actually about. Project names share generic tokens (*identity, fabric, CUI, data,
access, Duo*), so matching on words mis-files almost everything.

- **A call can serve several projects.** List every one it genuinely advances or blocks. In
  Hanuman Hub the call appears under each, so a missing one hides it from that project's view.
- **Only projects the call actually moved.** A passing mention in someone's status round-up is
  not an association. If it wasn't discussed or decided, leave it off.
- **If nothing fits, write no projects.** An empty list is honest; a wrong one is worse than
  none because it puts the call in a view where it doesn't belong.
- Names must match the workboard verbatim. The Hub flags names it can't resolve with a dashed
  chip, but a typo means the call never shows under the real project.

**A placeholder invite title is not a record of who attended.** Read the speaker names in the
transcript before describing a call — a meeting titled for three people may have had two, and
the difference often decides whether a follow-up is still open.

Show the user the assignments before writing them — one line, e.g. "Projects: Alpha, Beta."
They may add or remove one.

### 7. Return structured output

Return the content in this format:

```markdown
# Webex Recording: <title>
**Date**: <extracted from title or URL>
**Duration**: <if available>
**Recording URL**: <url>

## Chapters
1. [0:00] Chapter title
2. [MM:SS] Chapter title
...

## AI Summary
<notes from Webex AI>

## Action Items
<action items from Webex AI>

## Transcript
[0:00] Speaker Name: text
[0:15] Speaker Name: text
...
```

### 8. Save options

After returning the content, ask:
> "Where do you want to save this? Options:
> 1. Recording note (`memory/recording-<slug>-YYYY-MM-DD.md`)
> 2. Recording note + update CII Product Pulse (if CII-related)
> 3. Just show me the content (don't save)
> 4. Custom location"

If the user specified a save location in arguments, use that directly.

**The recording note.** Save to
`~/claude-memory/recording-<slug>-YYYY-MM-DD.md`. The filename carries the date and
any index reads it from there, so the `-YYYY-MM-DD` suffix is not optional. `<slug>` is short
and kebab-case — the people or the topic, not the full title.

```markdown
---
name: recording-<slug>-YYYY-MM-DD
description: <one line: who, when, and the three or four things that actually came out of it>
projects:
  - <Workboard project name>
  - <Workboard project name>
metadata:
  type: reference
---

# Recording — <title>

**Duration:** <N> min · **Recording ID:** `<id>`
**Transcript:** `<path to the VTT or TXT on disk, if one was saved>`

<the content from step 7>
```

- `description` is what the Hub list shows under the title — it's the line the user reads to
  decide whether to open the call. Lead with the outcomes, not the agenda.
- Omit the `projects:` key entirely when step 6 found nothing. Don't write an empty list.
- Drop the `**Transcript:**` line if no transcript file was saved. The Hub marks the rows that
  have one, so a line pointing at a file that isn't there is a lie about what's on disk.
- Then add the one-line pointer to `MEMORY.md` under `## Recordings & Meeting Notes`.

## Error Handling

- **Webex API 401**: Refresh the token as `/webex-search` does, retry once. If refresh also
  fails, say the token needs manual re-auth — don't fall back to the browser silently,
  because that turns a 30-second fix into a scraping session.
- **Webex API 403**: The token lost the recordings scopes. Report the scope list rather than
  guessing at the cause.
- **Discovery returns nothing**: Say so, and give the date of the most recent owned recording
  as proof the query ran. An empty window and a broken query look identical otherwise.
- **`transcriptDownloadLink` absent**: Transcription was off for that call. Offer the browser
  path for the AI summary; don't report an empty transcript as an empty call.
- **`temporaryDirectDownloadLinks` expired**: Re-fetch the detail record. Never cache these.
- **Chrome not running**: Only relevant for shared recordings. Tell the user to launch Chrome
  with the debug port command above.
- **MCP not configured**: Tell the user to add `chrome-devtools` to `~/.mcp.json`
- **Password incorrect**: Show error and ask for correct password
- **SSO required**: Prompt user to log in manually
- **Recording not found**: If email search returns no results, ask for a direct URL

## Tips

- **Prefer the API path whenever the call is the user's own.** It needs no browser, no SSO, and
  no password entry, and it leaves a real `.vtt` on disk for the note's `**Transcript:**` line.
- For recurring recordings (like weekly demos), `since 7d` on a Friday is usually a better
  habit than a cron job — the pick step wants a human.
- Transcripts can be very large — for the structured output, consider
  summarizing rather than including the full transcript unless requested
- The Webex AI summary and chapters are usually good enough for a quick overview;
  the full transcript is useful for finding specific quotes or details
- Discovery mode is read-only until the user picks something. Running it to look around costs
  nothing and writes nothing.
