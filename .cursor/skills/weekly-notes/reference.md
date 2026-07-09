# Weekly Notes — Reference

## Atlassian constants

From [config.yaml](config.yaml):

| Key | Value |
|-----|-------|
| cloudId | `969226a5-2105-49eb-a9f7-e3852660973e` |
| site | `https://rippling.atlassian.net` |
| spaceId | `6406248027` |
| space key | `~7120204850b617efd944b9ae686dff14ee52b5` |
| reports folder (parent) | `6407226609` |
| current weekly report | page `6407750046` |

Pass `cloudId` on every Atlassian MCP call. Read tool schemas before invoking.

## MCP tool map

### Confluence (user-atlassian)

| Task | Tool | Notes |
|------|------|-------|
| Read page | `getConfluencePage` | `contentFormat: html`, include `pageId` |
| List children | `getConfluencePageDescendants` | `depth: 2` for task + meeting pages |
| Search | `searchConfluenceUsingCql` | See CQL below |
| Update report | `updateConfluencePage` | Full HTML body, required `cloudId`, `pageId`, `body` |
| New page | `createConfluencePage` | `spaceId`, `parentId`, `title`, `body` |
| Auth | `mcp_auth` | When STATUS.md requires it |

### Jira (user-atlassian)

| Task | Tool |
|------|------|
| Current user | `atlassianUserInfo` |
| Issue details + comments | `getJiraIssue` — `fields: ["summary","status","comment","updated","resolution","issuetype","assignee"]` |
| Search activity | `searchJiraIssuesUsingJql` |
| Comment | `addCommentToJiraIssue` |
| Create | `createJiraIssue` (only when asked) |

### Jira sync JQL (from config.yaml)

```
assignee = currentUser() AND updated >= "2026-07-06" ORDER BY updated DESC
reporter = currentUser() AND updated >= "2026-07-06" ORDER BY updated DESC
```

Replace date with `weekly_report.current.week_start` or the target day.

### Jira sync — completed line template

```html
<li><p>Submitted Android MDM risk for default permission groups — <a href="https://rippling.atlassian.net/browse/RISK-481" data-card-appearance="inline">https://rippling.atlassian.net/browse/RISK-481</a></p></li>
```

### Brag doc reminder template (agent reply, not Confluence)

```markdown
**Brag doc:** `RISK-481` — "Android MDM sandbox policy gap" is now **Resolved**.
Add an accomplishment entry to the [Brag Doc](https://docs.google.com/document/d/1Gz-XuBK6NWpus_mTUAdJS2GBgLdBqbjpnWKYPJAsUoA/edit).
```

Trigger when issuetype ∈ `{Epic, Initiative, Story, Task}` and status ∈ `{Done, Resolved, Closed}` and updated in sync window.

### Google Drive (user-google-drive)

| Task | Tool |
|------|------|
| Search files | `search_drive_files` — e.g. `name contains 'Brag'` |
| Read Google Doc | `get_drive_file_content` |
| Download / export | `get_drive_file_download_url` |
| Update doc content | `update_drive_file` — user confirmation required |
| Create file | `create_drive_file` |
| New folder | `create_drive_folder` |
| Auth | `mcp_auth` |

**Brag Doc** (config.yaml):

| Field | Value |
|-------|-------|
| file_id | `1Gz-XuBK6NWpus_mTUAdJS2GBgLdBqbjpnWKYPJAsUoA` |
| url | `https://docs.google.com/document/d/1Gz-XuBK6NWpus_mTUAdJS2GBgLdBqbjpnWKYPJAsUoA/edit` |

Drive search example: `name = 'Brag Doc' and mimeType = 'application/vnd.google-apps.document'`

### Google Calendar (user-google-calendar)

| Task | Tool |
|------|------|
| List events in window | `get_events` — `time_min`, `time_max`, `max_results`, `detailed: true` |
| Search by keyword | `get_events` with `query` param |
| List calendars | `list_calendars` |
| Auth | `mcp_auth` |

Read tool schemas under `mcps/user-google-calendar/tools/` before calling.

**get_events example** (Pacific time, single work day):

```json
{
  "calendar_id": "primary",
  "time_min": "2026-07-08T00:00:00-07:00",
  "time_max": "2026-07-09T00:00:00-07:00",
  "max_results": 100,
  "detailed": true
}
```

**Work week** (Mon–Fri from `week_start`):

```json
{
  "calendar_id": "primary",
  "time_min": "2026-07-06T00:00:00-07:00",
  "time_max": "2026-07-11T00:00:00-07:00",
  "max_results": 100,
  "detailed": true
}
```

### Skip rules (no placeholders / no enrichment)

Skip when event `summary` matches (case-insensitive):

| Rule | Examples |
|------|----------|
| `skip_title_patterns` substring | focus time, lunch, ooo, **commuting**, **vet** |
| `skip_title_contains` | **dns** (personal do-not-schedule holds) |

### Rippling (user-rippling-mcp)

| Task | Tool |
|------|------|
| Execute sandbox JS | `code` — async arrow function body |
| Current worker profile | `codemode.lookup_me` inside `code` |
| Completed / open tasks | `codemode.ask_ai` inside `code` |
| Auth | `mcp_auth` |

Rippling exposes a single `code` tool. All Rippling APIs run inside the sandbox as `codemode.<name>(args)`.

**Completed tasks query** — use `rippling.completed_tasks_prompt` from config via `ask_ai`:

```javascript
async () => {
  let res = await codemode.ask_ai({
    operation: "start",
    message: "List all tasks I completed in Rippling between 2026-07-06 and 2026-07-09...",
    idempotency_key: "weekly-notes-" + Date.now(),
    wait_ms: 0,
    telemetry: { intent: "Query completed Rippling employee tasks for weekly notes sync." }
  });
  if (res.status === "running") {
    res = await codemode.ask_ai({
      operation: "poll",
      run_id: res.run_id,
      wait_ms: 25000,
      telemetry: { intent: "Poll Rippling AI for completed tasks." }
    });
  }
  return res;
}
```

Pass `extras: { telemetry: { intent: "..." } }` on the outer `code` call. **No PII in intent strings.**

### Rippling completed line template

```html
<li><p><s>Guardion Eoi form</s> <em>(Rippling HR)</em></p></li>
<li><p><s>AI@ Rippling training</s> — <a href="https://app.rippling.com/...">Rippling</a></p></li>
```

Match existing **Outstanding** items (e.g. "Set up 1passord") and strike through when Rippling reports complete.

### Event fields to capture

| Need | Typical event fields |
|------|---------------------|
| Title | `summary` |
| Start / end | `start.dateTime`, `end.dateTime` (or `date` for all-day) |
| Participants | `attendees[].displayName`, `attendees[].email` |
| Duration | computed from start/end |
| Zoom | `hangoutLink`, `conferenceData.entryPoints`, `description`, `location` (if contains `zoom.us`) |
| Physical room | `location` when not a video URL |
| Calendar link | `htmlLink` |

### Title fuzzy-match hints

Strip before comparing: `meet the team`, `intro`, `1:1`, `sync`, `call`, punctuation.

Examples:

| Note title | Calendar `summary` | Match? |
|------------|-------------------|--------|
| `Piotr intro` | `Axl / Piotr intro call` | yes |
| `gcp logging sync` | `Axl Daniyal - GCP logging` | yes |
| `team standup` | `SecEng standup` | maybe — confirm with user |

### Meeting details HTML (enriched past meeting)

```html
<h2>Meeting details</h2>
<p><strong>Title:</strong> Axl Daniyal gcp logging sync</p>
<p><strong>When:</strong> <time datetime="2026-07-08T15:00:00-07:00">July 8, 2026 3:00–3:30 PM</time> (30 min)</p>
<p><strong>Participants:</strong> Daniyal Ahmad, Axl Liess</p>
<p><strong>Location:</strong> <a href="https://rippling.zoom.us/j/123">Zoom</a></p>
<h2>Notes</h2>
<p>big query export discussion…</p>
```

### Upcoming placeholder HTML

```html
<p><strong>Scheduled:</strong> <time datetime="2026-07-10T14:00:00-07:00">July 10, 2026 2:00–2:30 PM</time> (30 min)</p>
<p><strong>Participants:</strong> Eric Ellett, Axl Liess</p>
<p><strong>Location:</strong> <a href="https://rippling.zoom.us/j/456">Zoom</a></p>
<p><a href="https://calendar.google.com/calendar/event?eid=...">Open in Google Calendar</a></p>
<h2>Notes</h2>
<p></p>
```

### Outstanding line for upcoming meeting

```html
<li><p>Intro call with <a href="https://rippling.atlassian.net/wiki/.../pages/{id}">Eric Ellett</a> — Jul 10 2:00 PM (30 min, Zoom)</p></li>
```

### GitHub (user-github)

| Task | Tool |
|------|------|
| Current user login | `get_me` |
| Search your PRs | `search_pull_requests` — use `sort: updated`, `order: desc` |
| PR details | `pull_request_read` method `get` |
| Commits | `pull_request_read` method `get_commits` |
| Files changed | `pull_request_read` method `get_files` |
| Auth | `mcp_auth` |

**Search examples** (substitute author and date from config):

```
author:YOUR_LOGIN updated:>=2026-07-06
author:YOUR_LOGIN org:Rippling updated:>=2026-07-06
author:YOUR_LOGIN repo:Rippling/rippling-main updated:>=2026-07-06
```

Use `search_pull_requests`, not `list_pull_requests`, when filtering by author.

### GitHub PR sync — completed line template

```html
<li><p>Added BQ scheduled export for GCP logging sync (<code>rippling-main</code> <a href="https://github.com/Rippling/rippling-main/pull/12345">#12345</a>) — <a href="https://rippling.atlassian.net/browse/CORPSE-97" data-card-appearance="inline">https://rippling.atlassian.net/browse/CORPSE-97</a></p></li>
```

### Jira comment from PR sync

```markdown
**PR sync** (2026-07-09): [rippling-main#12345](https://github.com/Rippling/rippling-main/pull/12345) — merged
Added Terraform for BQ export tables and GCS buckets.
```

Post via `addCommentToJiraIssue` with `contentFormat: markdown`. Skip if the PR URL already appears in recent comments.

### Jira key extraction

Scan PR title, body, head branch, and commit messages for keys matching:

```
[A-Z][A-Z0-9]+-\d+
```

Examples: `CORPSE-97`, `RISK-481`, `SECENG-1234`

### Gmail (user-gmail)

| Task | Tool |
|------|------|
| Find messages | `search_gmail_messages` |
| Read message | `get_gmail_message_content` |
| Download attachment | `get_gmail_attachment_content` |

Attachment workflow: search → identify `message_id` and `attachment_id` from message metadata → download → Read image → transcribe.

## CQL examples

```
title ~ "ALiess weekly report" AND space = "~7120204850b617efd944b9ae686dff14ee52b5" ORDER BY lastmodified DESC

title ~ "Meet the team" AND ancestor = 6407750046

title ~ "26/07/08" AND space = "~7120204850b617efd944b9ae686dff14ee52b5"
```

## HTML patterns (from live report)

### Day header

```html
<h1><time datetime="2026-07-08">July 8, 2026</time></h1>
```

### Completed item with strike-through

```html
<ul>
  <li><p><s>schedule time w/ sam</s></p></li>
  <li><p>set up codex mcp</p></li>
</ul>
```

### Nested outstanding tasks

```html
<h2>Outstanding</h2>
<ul>
  <li><p>Set up cursor w/ mcp</p></li>
  <li><p>review on boarding doc</p>
    <ul>
      <li><p>start planning/documenting project work</p></li>
    </ul>
  </li>
</ul>
```

### Jira inline card

```html
<p><a href="https://rippling.atlassian.net/browse/RISK-481" data-card-appearance="inline">https://rippling.atlassian.net/browse/RISK-481</a></p>
```

### Link to child Confluence page

```html
<p>Intro call with <a href="https://rippling.atlassian.net/wiki/spaces/~7120204850b617efd944b9ae686dff14ee52b5/pages/6415877325">Piotr Szwajkowski</a> scheduled</p>
```

### Existing embedded image (preserve IDs when editing)

```html
<figure data-type="media-single" data-layout="center">
  <div data-type="media" data-media-type="file"
       data-id="15224c31-3e27-4cff-b9e6-ce506c4ef258"
       data-collection="contentId-6407750046"></div>
</figure>
```

Do not copy `data-id` to new images; only preserve when editing fetched content.

## Page hierarchy (example week)

```
ALiess weekly report 26/07/10 (6407750046)
├── 26/07/08 task meet the team (6416041103)
│   ├── Meet the team: Piotr Szwajkowski (6415877325)
│   ├── Meet the team: Eric Ellett (6416204782)
│   └── …
└── Axl Daniyal gcp logging sync 26/07/08 (6419218547)
```

## Gmail search operators

```
has:attachment (filename:jpg OR filename:jpeg OR filename:png) newer_than:7d
subject:notes has:attachment
from:me has:attachment newer_than:3d
```

## Calendar enrich checklist

- [ ] `get_events` for the target day or work week (`detailed: true`)
- [ ] Filtered to user's email; skipped commuting/vet/DNS/focus/lunch/OOO patterns
- [ ] Fuzzy-matched event title to note/page title (or asked user if ambiguous)
- [ ] Added participants, duration, Zoom link, and/or physical room
- [ ] Did not invent metadata when no calendar match

## Upcoming placeholder checklist

- [ ] `get_events` for current work week (future events only)
- [ ] `getConfluencePageDescendants` — no duplicate child page
- [ ] Created placeholder with **Scheduled**, **Participants**, **Location**, empty **Notes**
- [ ] Linked from correct day's **Outstanding**
- [ ] Did not overwrite pages that already have note content

## Rippling task sync checklist

- [ ] `code` → `lookup_me` to confirm signed-in worker
- [ ] `code` → `ask_ai` with `completed_tasks_prompt` for sync window
- [ ] Polled until `status: completed` (extra `code` calls if needed)
- [ ] Parsed tasks with completion dates and categories
- [ ] Moved matching Outstanding items to Completed with `<s>`
- [ ] Deduped task titles already on the report
- [ ] No PII in telemetry intent strings

## GitHub PR sync checklist

- [ ] `get_me` → substituted `{author}` in search query
- [ ] `search_pull_requests` since week_start (or target day)
- [ ] Per PR: `get` + `get_commits` (+ `get_files` if needed)
- [ ] Extracted Jira keys; deduped PR lines on weekly report
- [ ] Added Completed lines with PR + Jira inline links
- [ ] `addCommentToJiraIssue` on linked tickets (deduped by PR URL in comments)
- [ ] Did not transition/close tickets without explicit user request
- [ ] Brag doc reminder for merged PRs tied to initiative tickets

## Jira sync checklist

- [ ] Queried assignee + reporter issues since week_start (or target day)
- [ ] Extracted user comments in window as work-done summaries
- [ ] Deduped issue keys already on the weekly report
- [ ] Added Completed lines with inline Jira card links
- [ ] Brag doc reminders issued for Done/Resolved initiatives in window
- [ ] Did not edit Brag Doc without explicit user request

## OCR quality checklist

- [ ] Transcription added to Confluence (not image-only)
- [ ] Uncertain words marked or noted
- [ ] Section placement matches user intent (daily vs child page)
- [ ] Links to Jira/Confluence/Docs/Calendar added where references exist
- [ ] Child page created and linked when content exceeds ~2 short paragraphs

## updateConfluencePage call shape

```json
{
  "cloudId": "969226a5-2105-49eb-a9f7-e3852660973e",
  "pageId": "6407750046",
  "contentFormat": "html",
  "body": "<full html from getConfluencePage with edits>",
  "versionMessage": "Add July 9 daily section"
}
```

Fetch → edit → write the entire body in one call.
