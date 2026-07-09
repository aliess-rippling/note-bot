---
name: weekly-notes
description: >-
  Creates and maintains weekly work reports in Confluence (Atlassian) as a
  lightweight week hub plus one page per calendar day, with completed/outstanding
  task lists and child pages for meetings and tasks under each day, Google Calendar
  context for past and upcoming meetings, Rippling completed-task sync, Jira ticket
  sync, GitHub PR sync with linked work summaries and Jira ticket comments, Slack
  thread and channel message sync with summaries, brag-doc reminders on resolved
  initiatives, handwritten-note OCR from Gmail/Drive/local images, and Google Drive
  doc access. Use when updating weekly notes, syncing calendar meetings, creating
  meeting placeholders, syncing Rippling or Jira or GitHub or Slack activity, daily
  logs, meeting notes, or transcribing note photos.
---

# Weekly Notes (Confluence)

Maintain **ALiess weekly report** week hubs and **one Confluence page per calendar day** in the Rippling personal space. All Atlassian, GitHub, Gmail, Google Calendar, Google Drive, Rippling, and Slack access goes through MCP.

## Quick start

```
Task Progress:
- [ ] Read config.yaml (hub_page_id, days map, cloud_id)
- [ ] Resolve target day page for the date being edited (see below)
- [ ] Authenticate MCP servers if needed (mcp_auth)
- [ ] getConfluencePage → fetch that day page's HTML body + version only
- [ ] Plan edits (preserve data-local-id on existing nodes)
- [ ] updateConfluencePage (day page) or createConfluencePage (day or child)
- [ ] Verify with getConfluencePage or share webUrl
```

Load [config.yaml](config.yaml) before every session. For HTML patterns and MCP tool args, see [reference.md](reference.md).

**MCP payload limit**: never fetch or update the week hub with full daily content. One `getConfluencePage` / `updateConfluencePage` per **day page** (or child page). Hub edits are limited to the **Days** index list.

## Page hierarchy (one page per day)

```
Reports folder (reports_parent_folder_id)
└── ALiess weekly report 26/07/10          ← week hub (lightweight index)
    ├── 26/07/06 July 6, 2026              ← day page (Completed / Outstanding)
    ├── 26/07/07 July 7, 2026
    └── 26/07/08 July 8, 2026
        ├── 26/07/08 task meet the team    ← child (task group)
        │   └── Meet the team: Piotr …     ← child (1:1)
        └── Axl Daniyal gcp logging sync 26/07/08
```

| Page type | Parent | Body contains |
|-----------|--------|---------------|
| **Week hub** | reports folder | TOC, week metadata, **Days** links to day pages, optional Brag Doc link — **no** Completed/Outstanding |
| **Day page** | week hub | TOC, date heading, narrative, **Completed**, **Outstanding** |
| **Child page** | **day page** | meeting notes, task threads, sync write-ups |

Child pages for a given calendar day always use `parentId` = that day's `page_id`, not the hub.

## Resolve the active week and day

1. **Week hub**: `config.yaml` → `weekly_report.current.hub_page_id` (alias `page_id`).
2. If missing or user says "new week", run **New week workflow** below.
3. **Day page** for date `YYYY-MM-DD`:
   - `weekly_report.current.days["YYYY-MM-DD"].page_id` if present in config, else
   - `getConfluencePageDescendants` on the hub and match title prefix `{YY/MM/DD}`, or
   - CQL: `ancestor = {hub_page_id} AND title ~ "26/07/09"` (use `child_date_suffix` for that date).
4. If no day page exists, run **Add a new day** below.
5. If `legacy_monolithic: true` and the hub still holds inline `<h1>` day sections, run **Migrate legacy monolithic week** before routine edits.

Optionally confirm the hub with CQL: `title ~ "ALiess weekly report" AND space = "~7120204850b617efd944b9ae686dff14ee52b5" ORDER BY lastmodified DESC`.

## Daily page workflow

Each calendar day is its **own Confluence page** (title from `daily_page.title_template`, e.g. `26/07/09 July 9, 2026`).

Day page body structure:

```html
<div data-type="extension" data-extension-key="toc" ...></div>
<p><a href="{hub_web_url}">← Week of …</a></p>
<h1><time datetime="2026-07-09">July 9, 2026</time></h1>
<h2>Completed</h2>
<ul></ul>
<h2>Outstanding</h2>
<ul></ul>
```

| Section | Heading | Content |
|---------|---------|---------|
| Narrative | (optional `<p>` after `<h1>`) | Short context for the day |
| Done | `<h2>Completed</h2>` | Bullet list; strike through with `<s>` when done |
| Open | `<h2>Outstanding</h2>` | Nested `<ul>` for subtasks |

### Add a new day (required)

When creating a day page, **always roll forward Outstanding** from the **immediately previous calendar day's page**:

1. Resolve the prior day page (sibling under the same hub).
2. `getConfluencePage` on the **prior day page only** → find `<h2>Outstanding</h2>` and its `<ul>` (including nested sub-items).
3. `createConfluencePage` with `parentId` = hub, title from `daily_page.title_template`, body = TOC + date `<h1>` + empty **Completed** + copied **Outstanding** (omit `<s>` items).
4. **Copy** the prior Outstanding list into the new day — full tree, same wording and links.
5. Leave the **previous day page** unchanged (historical snapshot).
6. Update the **hub** only: add a link in the **Days** list (small payload).
7. Record `weekly_report.current.days["YYYY-MM-DD"]` in `config.yaml` (`page_id`, `title`, `web_url`).

Skip roll-forward only when the user explicitly says prior Outstanding is cleared or not applicable.

**Update tasks**: `getConfluencePage` + `updateConfluencePage` on the **target day page** only. Move items from Outstanding → Completed (with `<s>` on completed sub-items). Link Jira with inline cards: `<a href="https://rippling.atlassian.net/browse/KEY-123" data-card-appearance="inline">...</a>`. Optionally run **Rippling task sync**, **Jira ticket sync**, **GitHub PR sync**, or **Slack thread sync** to backfill Completed.

**Link child pages** from the day's Completed or Outstanding: `<a href="https://rippling.atlassian.net/wiki/spaces/.../pages/{id}">Title</a>`.

## Starred lines = tasks

A **star** (★, `*`, or a clear star doodle) beside a line marks an **open task** — in handwriting, on a meeting child page, or anywhere in notes.

| Where the star appears | What to do |
|------------------------|------------|
| Day page scratch / narrative | Add line to that day's **Outstanding** |
| Meeting child page **Notes** | Keep starred line in **Notes** *and* add the task to the day's **Outstanding** on the day page |
| Handwritten page (OCR) | Same as above; route starred lines to **Outstanding** |

Rules:

- Do **not** put starred lines in **Completed** unless the user says the task is done.
- Preserve the star when transcribing, e.g. `★ follow up on GCP default permission groups`.
- Dedupe: if the task is already in **Outstanding**, do not add again.
- Optional context on Outstanding: prefix with meeting name or link to the child page, e.g. `★ (Piotr intro) send doc link`.
- Unstarred lines are narrative, decisions, or links — not tasks unless the user says otherwise.

## Child page workflows

### When to create a child page

| Situation | Parent | Title pattern |
|-----------|--------|---------------|
| Multi-person task thread (e.g. meet-the-team) | **Day page** for that date | `{YY/MM/DD} task {topic}` |
| Individual intro / 1:1 | Task group page | `Meet the team: {Name}` |
| Ad-hoc meeting or deep dive | **Day page** for that date | `{Who} {topic} {YY/MM/DD}` |

Use `getConfluencePageDescendants` on the **day page** (not the hub) to avoid duplicates. Link new pages from the relevant **Completed** or **Outstanding** line on that day page.

### Meet-the-team template

```html
<div data-type="extension" data-extension-key="toc" data-extension-type="com.atlassian.confluence.macro.core" data-parameters="{&quot;outline&quot;:true,&quot;maxLevel&quot;:3}"></div>
<p><strong>Role:</strong> …</p>
<p><strong>Can help with:</strong> …</p>
<p><a href="https://calendar.google.com/calendar/render?action=TEMPLATE&amp;text=...">📅 Schedule meeting with {Name}</a></p>
<h2>Notes</h2>
<p></p>
```

Fill **Notes** after the call. Keep prep info (role, calendar link) even when notes are empty. Apply **Starred lines = tasks** for any ★ in **Notes**.

When a calendar match exists, prepend **Meeting details** (see Calendar workflow) before **Notes**.

### Sync / working-session template

Short bullets, relevant doc links (Confluence cards, Google Docs, external URLs), optional screenshot. Example: `Axl Daniyal gcp logging sync 26/07/08`. Starred bullets in **Notes** → also add to daily **Outstanding**.

**Create**: `createConfluencePage` with `parentId` = **day page** (or task-group page), `spaceId` from config, `contentFormat: html`. Start the body with a **table of contents** (see Page layout below).

## Google Calendar workflow

MCP server: `user-google-calendar`. Authenticate with `mcp_auth` if needed. Read tool schemas before calling (primary tool: `get_events` with `detailed: true`).

Use calendar to (a) enrich past meeting notes with metadata, and (b) create **placeholder child pages** for upcoming meetings in the current week.

### When to consult calendar

| Trigger | Action |
|---------|--------|
| Handwritten note has a **date + title** | `get_events` that day → fuzzy-match title → enrich note |
| Meeting child page exists, call already happened | Match by title + date → add participants, duration, location |
| User asks to prep upcoming meetings | `get_events` for the work week → create placeholder pages |
| New day page / weekly setup | Optionally seed **Outstanding** with linked upcoming meetings |

### Fetch events

Window defaults from `config.yaml`:

- **Single day**: `time_min` / `time_max` = start/end of day in `google_calendar.timezone`
- **Current work week**: `weekly_report.current.week_start` through that week's Friday (23:59:59)

```
get_events(calendar_id, time_min, time_max, max_results, detailed: true)
```

Filter to events where `google_calendar.user_email` is in attendees or the user is organizer.

**Skip events** (do not enrich or create placeholders) when the event `summary` matches any of:

- `skip_title_patterns` — case-insensitive substring (includes `commuting`, `vet`, focus blocks, lunch, OOO)
- `skip_title_contains` — case-insensitive contains (includes `dns` for personal do-not-schedule holds)

### Match events to notes

Given a candidate **date** and **title** (from handwriting OCR, child page title, or user input):

1. Normalize titles: lowercase, strip punctuation, remove prefixes like `meet the team:`, `intro`, `1:1`, `sync`.
2. `get_events` for that calendar day.
3. Score each event: title token overlap + attendee name overlap with note text.
4. Pick the best match above a reasonable threshold; if ambiguous, ask the user.
5. If no match, note `[no calendar match]` and continue without inventing metadata.

### Extract meeting metadata

From the matched event record:

| Field | Source | Notes |
|-------|--------|-------|
| **Participants** | `attendees` | Names or emails; exclude the user's own email |
| **Duration** | `start` + `end` | e.g. "30 min" or "2:00–2:30 PM" |
| **Zoom / video** | `hangoutLink`, `conferenceData`, `description`, `location` | URLs matching `zoom_url_patterns` in config |
| **Physical room** | `location` | When not a Zoom URL — office, room name, address |

Prefer explicit `location` for physical rooms. When both exist (hybrid), show both.

### Meeting details block (HTML)

Insert at top of meeting child page or under the daily narrative:

```html
<h2>Meeting details</h2>
<p><strong>Title:</strong> Axl / Piotr intro call</p>
<p><strong>When:</strong> <time datetime="2026-07-08T14:00:00-07:00">July 8, 2026 2:00–2:30 PM</time> (30 min)</p>
<p><strong>Participants:</strong> Piotr Szwajkowski, Axl Liess</p>
<p><strong>Location:</strong> <a href="https://rippling.zoom.us/j/...">Zoom</a></p>
<p><strong>Location:</strong> Seattle office — 3W-401</p>
```

Use one or two **Location** lines (video + physical) as applicable. Omit empty fields.

### Upcoming meeting placeholders

For each **future** event in the current work week that lacks a child page:

1. Resolve that event's **day page**; create the day page first if missing.
2. `getConfluencePageDescendants` on the **day page** — dedupe by similar title + date.
3. `createConfluencePage` with `parentId` = day page, placeholder body (no notes yet).
4. Link from that day's **Outstanding** on the **day page**.

**Title** — pick the best fit:

| Event type | Title pattern |
|------------|---------------|
| Intro / meet-the-team | `Meet the team: {Person}` (extract person from event title or attendees) |
| Named sync | `{participants or topic} {YY/MM/DD}` per `sync_meeting_title_template` |
| Generic | `{event_title} {YY/MM/DD}` |

**Placeholder body**:

```html
<div data-type="extension" data-extension-key="toc" data-extension-type="com.atlassian.confluence.macro.core" data-parameters="{&quot;outline&quot;:true,&quot;maxLevel&quot;:3}"></div>
<p><strong>Scheduled:</strong> <time datetime="...">...</time> ({duration})</p>
<p><strong>Participants:</strong> ...</p>
<p><strong>Location:</strong> Zoom / physical room (as applicable)</p>
<p><a href="{htmlLink or hangoutLink}">Open in Google Calendar</a></p>
<h2>Notes</h2>
<p></p>
```

Do not overwrite child pages that already have **Notes** content. Update metadata only when the page is still a placeholder.

### Handwritten notes + calendar

After OCR, if the transcription includes a date and meeting title:

1. Run calendar match for that date/title.
2. If matched → create or update child page with **Meeting details** + transcribed **Notes**.
3. If unmatched → add transcription to the **day page** or a child page without fabricated metadata.

## Handwritten notes / image OCR workflow

Sources (in order of user hint):

1. **Gmail** — `search_gmail_messages` → `get_gmail_message_content` → `get_gmail_attachment_content` (use `return_base64: true` if sandboxed).
2. **Local directory** — scan paths in `config.yaml` → `handwritten_notes.local_image_dirs`; user may pass an explicit path.
3. **Google Drive** — `search_drive_files` or `config.yaml` file_id → `get_drive_file_content` (text/docs) or `get_drive_file_download_url` (images).

**Transcription** (required):

1. Open the image with the Read tool (vision) or decode base64 to a temp file then Read.
2. Transcribe faithfully; mark illegible spans as `[illegible]`.
3. Structure as headings/bullets matching the handwriting.
4. Apply **Starred lines = tasks** for any starred lines.

**Publish**:

| User request | Action |
|--------------|--------|
| Short note | Add transcribed text to today's **day page** (narrative or Completed) |
| Starred task line | Add to **Outstanding** on that day's page (see **Starred lines = tasks**) |
| Long or multi-topic | Create child page under the day page; link from Completed/Outstanding |
| User wants image preserved | Add transcription under a `## Transcription` or `## Notes` heading; embed image if possible (see below) |

**Image embed limitation**: Atlassian MCP has no attachment-upload tool. Prefer full transcription in the body. If the user requires the image on the page, either (a) they upload via Confluence UI while you add the text, or (b) store in Google Drive via MCP and link the Doc/Drive URL in the page. Existing pages use `<figure data-type="media-single">` with server-assigned `data-id` — do not invent media IDs.

## New week workflow

1. Compute `week_label_date` as `YY/MM/DD` of that week's Friday (see config).
2. `createConfluencePage` — **week hub**:
   - `title`: `ALiess weekly report {week_label_date}`
   - `parentId`: `reports_parent_folder_id` from config
   - `spaceId`: from config
   - `body`: TOC macro + week heading + empty **Days** list + optional Brag Doc link
3. `createConfluencePage` — **first day page** with `parentId` = new hub; TOC + first day `<h1>` + Completed/Outstanding (or user's opening notes).
4. Update hub **Days** list with a link to the first day page.
5. Update `config.yaml` → `weekly_report.current` with `hub_page_id`, `title`, dates, `web_url`, `legacy_monolithic: false`, and `days` map entry for the first day.
6. Leave prior week's hub and day pages unchanged (historical record).

**Hub body template**:

```html
<div data-type="extension" data-extension-key="toc" ...></div>
<h1>Week of July 6–10, 2026</h1>
<p><a href="{brag_doc_url}">Brag Doc</a></p>
<h2>Days</h2>
<ul>
  <li><p><a href="{day_page_url}">26/07/06 July 6, 2026</a></p></li>
</ul>
```

## Migrate legacy monolithic week

Use when `legacy_monolithic: true`, the hub body contains multiple `<h1>` day sections, or `updateConfluencePage` fails due to body size.

```
Task Progress:
- [ ] getConfluencePage (hub) — full HTML
- [ ] Split body at each <h1><time datetime="YYYY-MM-DD">...</time></h1>
- [ ] For each day: createConfluencePage (parentId = hub) with that section's content
- [ ] Record each day in config.yaml → weekly_report.current.days
- [ ] Replace hub body with lightweight index (TOC + Days links only)
- [ ] Set legacy_monolithic: false
- [ ] Link existing hub children from the matching day page (reparent in Confluence UI if needed)
```

1. For each `<h1>` day block on the monolithic hub, `createConfluencePage` with `parentId` = hub, title from `daily_page.title_template`, body = TOC + that day's HTML (Completed, Outstanding, narrative).
2. Replace the hub body with the **hub body template** (Days list links to new day pages). Do not leave inline day sections on the hub.
3. Populate `weekly_report.current.days` with each new `page_id`.
4. **Existing child pages** still parented to the hub: match by `{date_suffix}` in the title and add links from the correct day page. The Atlassian MCP has no reparent tool — drag pages under the day page in the Confluence UI when the user wants the tree cleaned up.
5. After migration, all routine edits target **day pages** only.

## Rippling task sync workflow

Pull **completed Rippling tasks** (onboarding checklist, IT provisioning, HR forms, trainings, action items) into the matching **day page** **Completed** section. MCP server: `user-rippling-mcp` via the `code` tool and `codemode.*` sandbox.

```
Task Progress:
- [ ] Read config.yaml (rippling.completed_tasks_prompt, week_start)
- [ ] Resolve day page(s) for the sync window
- [ ] code → codemode.lookup_me (confirm signed-in worker)
- [ ] code → codemode.ask_ai (completed tasks since week_start)
- [ ] Poll ask_ai across code calls if status is "running"
- [ ] Parse task table: title, completion date, category, rippling_url
- [ ] Match existing Outstanding lines → move to Completed with <s>
- [ ] Dedupe by task title already on the target day page
- [ ] updateConfluencePage (one day page per call)
```

### Query completed tasks

Use `rippling.completed_tasks_prompt` with `{since_date}` and `{until_date}` (default: `week_start` → today, or a single target day).

Invoke via `code` tool:

```javascript
async () => {
  let res = await codemode.ask_ai({
    operation: "start",
    message: "<prompt from config>",
    idempotency_key: "weekly-notes-" + Date.now(),
    wait_ms: 0,
    telemetry: { intent: "Query Rippling for employee tasks completed in the weekly notes sync window." }
  });
  if (res.status === "running") {
    res = await codemode.ask_ai({
      operation: "poll",
      run_id: res.run_id,
      wait_ms: 25000,
      telemetry: { intent: "Poll Rippling AI for completed task results." }
    });
  }
  return res;
}
```

**Polling limits**: at most one `start` + two `poll`s per `code` call (~60s sandbox cap). If still `running`, poll again in a **separate** `code` invocation with the same `run_id`.

**Telemetry**: `extras.telemetry.intent` on the `code` call and each `codemode.*` call must describe the goal **without PII** (no names, emails, or worker IDs in intent strings).

### What to capture

Include tasks from categories in `rippling.task_categories`: onboarding, IT setup (1Password, GitHub, MDM), HR forms, benefits, compliance trainings, reimbursements, and other employee action items visible in Rippling.

### Note format

```html
<li><p><s>Set up 1password</s> <em>(Rippling onboarding)</em></p></li>
<li><p><s>AI@ Rippling training</s> — <a href="https://app.rippling.com/...">Rippling</a></p></li>
```

- Place on the **completion date** from Rippling (correct **day page**).
- If the task already appears in **Outstanding**, strike through and move to **Completed** instead of duplicating.
- Link `rippling_url` from the ask_ai response when provided; otherwise link `rippling.profile_url` or omit.

### Outstanding sync (optional)

When user asks, run a second `ask_ai` query for **incomplete** onboarding/IT/HR tasks and add missing items to **Outstanding** (do not mark complete).

## Jira ticket sync workflow

Pull the user's Jira activity for the current week (or a specific day) and add **work done** to the matching **day page** with **inline ticket links**. Run on request or at the end of a daily-notes update when the user wants Jira synced.

```
Task Progress:
- [ ] Read config.yaml (jira JQL, brag_doc_issue_types, week_start)
- [ ] Resolve day page(s) for the sync window
- [ ] atlassianUserInfo → confirm current user
- [ ] searchJiraIssuesUsingJql (assignee + reporter queries)
- [ ] getJiraIssue per candidate (fields: summary, status, comment, updated, resolution, issuetype)
- [ ] Extract user-authored comments in date range → work-done bullets
- [ ] Dedupe against issue keys already on the target day page(s)
- [ ] Append to correct day Completed with Jira inline cards
- [ ] Flag newly Done/Resolved initiatives → brag-doc reminder (below)
- [ ] updateConfluencePage (one day page per call)
```

### JQL and scope

Use `since_date` from `weekly_report.current.week_start` unless syncing a single day (use that day's `YYYY-MM-DD`).

| Query | JQL template (config.yaml) |
|-------|---------------------------|
| Assigned & updated | `jql_my_updated_issues` |
| Reported & updated | `jql_my_reported_issues` |

Merge results; dedupe by issue key. Cap at `jira.max_results`.

### What counts as "work done"

For each issue, build a one-line summary from (in priority order):

1. **User's comments** in the sync window — paraphrase the substance; quote only short phrases.
2. **Status + summary** when the user moved it forward (e.g. transitioned to In Progress, submitted for review) — infer from latest comment or status name if no comment.
3. **Resolution** when Done/Resolved — e.g. "Resolved RISK-481: Android MDM sandbox policy gap".

Skip issues with no attributable user activity in the window.

### Note format

Add under **Completed** for the matching day:

```html
<li><p>{work summary} — <a href="https://rippling.atlassian.net/browse/{KEY}" data-card-appearance="inline">https://rippling.atlassian.net/browse/{KEY}</a></p></li>
```

If the issue key is already linked on that day, **update** the existing line instead of duplicating.

Long threads → child page titled `{KEY} {summary}` with comment excerpts; link from Completed.

### Brag doc reminder (required)

When an issue meets **all** of:

- `issuetype` in `jira.brag_doc_issue_types` (or user calls it an "initiative")
- `status.name` in `jira.brag_doc_status_names` (Done, Resolved, Closed)
- `updated` falls within the sync window (user likely just closed it)

**Do not** silently skip. In the agent reply, include a prominent reminder:

> **Brag doc:** `{KEY}` — "{summary}" is now **{status}**. Add an accomplishment entry to the [Brag Doc]({google_drive.brag_doc.url}).

Do **not** edit the Brag Doc unless the user explicitly asks. Offer to draft an entry if they want help.

Use `get_drive_file_content` on `google_drive.brag_doc.file_id` only when drafting or reviewing brag entries.

## GitHub PR sync workflow

Pull **your** pull requests updated in the sync window, summarize work completed, add lines to the matching **day page(s)**, and **comment on linked Jira tickets**. MCP server: `user-github`.

```
Task Progress:
- [ ] Read config.yaml (github search templates, week_start)
- [ ] Resolve day page(s) for the sync window
- [ ] get_me → GitHub login
- [ ] search_pull_requests (author + updated since)
- [ ] pull_request_read (get, get_commits, get_files) per PR
- [ ] Extract Jira keys from title, body, branch, commits
- [ ] Summarize work done → Completed on matching day page
- [ ] addCommentToJiraIssue on each linked ticket (dedupe)
- [ ] Merged PR + resolved Jira → brag-doc reminder if applicable
- [ ] updateConfluencePage (one day page per call)
```

### Which PRs to include

Use `since_date` from `weekly_report.current.week_start` (or target day). Query via `github.pr_search_query` with `{author}` from `get_me` and `{since_date}` substituted.

Default scope: PRs **authored by you** with activity in the window (`updated:>=since_date`). If the user says "review requests I sent", additionally search `review-requested-by:{author}` and note review activity separately.

Optionally scope to `org:{github.org}` or `repo:{org}/{repo}` from config when results are noisy.

### What counts as work done

For each PR, summarize from (priority order):

1. **Merged/closed state** — "Merged PR: {title}" or "Opened PR: {title} (in review)"
2. **PR body** — purpose and approach (1–2 sentences)
3. **Commits** in window — `pull_request_read` method `get_commits`
4. **Files changed** — highlight meaningful paths via `get_files` when summary is thin

Map each PR to the **day it was last updated** (or merged date) for the correct **day page**.

### Note format

```html
<li><p>{summary} (<code>{repo}</code> <a href="{pr_html_url}">#{number}</a>) — <a href="https://rippling.atlassian.net/browse/{KEY}" data-card-appearance="inline">https://rippling.atlassian.net/browse/{KEY}</a></p></li>
```

When no Jira key is found, omit the ticket link and keep the PR link. Dedupe by PR URL or `#number` already on the day page.

Long PRs → child page `{repo}#{number} {title}` under the matching day page with body excerpt + file list; link from Completed.

### Jira ticket updates

Extract issue keys matching `github.jira_key_pattern` (default `[A-Z][A-Z0-9]+-\d+`) from PR title, body, branch, and commit messages.

For each linked ticket:

1. `getJiraIssue` — confirm key exists and note current status.
2. `addCommentToJiraIssue` with `contentFormat: markdown`:

```markdown
**PR sync** ({date}): [{repo}#{number}]({pr_url}) — {state}
{one-line work summary}
```

**Dedupe**: skip if an identical PR URL already appears in recent ticket comments (last 7 days).

**Do not** transition or close tickets automatically unless the user explicitly asks. If a PR is **merged** and the ticket is still open, mention in the agent reply that the user may want to transition `{KEY}`.

### Brag doc (merged PRs)

When a PR is **merged** in the sync window and links to a Jira issue whose type is in `jira.brag_doc_issue_types`, include the standard brag-doc reminder (especially if the ticket also moved to Done/Resolved).

## Slack thread sync workflow

Pull **your** Slack messages and **threads you participated in** for the sync window, summarize substantive discussions, and add lines to the matching **day page(s)** **Completed** section (or child pages for long threads). MCP server: `user-slack`. Authenticate with `mcp_auth` if needed.

```
Task Progress:
- [ ] Read config.yaml (slack search templates, skip_channel_patterns, week_start)
- [ ] Resolve day page(s) for the sync window
- [ ] slack_read_user_profile → confirm current user / user_id
- [ ] Search your messages since since_date (public + private per config)
- [ ] Group hits by thread (thread_ts) or standalone message
- [ ] slack_read_thread for each thread needing full context
- [ ] Summarize decisions, asks, and outcomes (1–3 sentences)
- [ ] Dedupe against Slack permalinks already on target day page(s)
- [ ] Append Completed lines (or create child page for long threads)
- [ ] updateConfluencePage (one day page per call)
```

### Scope: what to include

Sync window: `since_date` from `weekly_report.current.week_start` unless syncing a single day (use that day's `YYYY-MM-DD`).

| Source | How to find |
|--------|-------------|
| **Messages you sent** to channels | `slack.my_messages_query` with `{since_date}` substituted |
| **Threads you participated in** | Same search — replies you sent appear as `from:me`; group by `thread_ts` |
| **Thread parents you started** | Optional second pass: `slack.my_thread_messages_query` |

**Search tool selection** (from `slack.include_private_channels`):

| `include_private_channels` | Tool | Notes |
|------------------------------|------|-------|
| `true` (default) | `slack_search_public_and_private` | DMs, private channels, MPIMs — **ask user consent** before first private search in a session |
| `false` | `slack_search_public` | Public channels only; no consent prompt |

Pass `sort: "timestamp"`, `sort_dir: "desc"`, `limit` ≤ `slack.max_search_results`, `include_context: false` on search calls to keep payloads small. Paginate with `cursor` when the window has many hits.

### Group and read threads

For each search result where you authored a message in the sync window:

1. **Thread** (`thread_ts` present and ≠ `ts`): use `(channel_id, thread_ts)` as the group key. Call `slack_read_thread` with `channel_id` and `message_ts` = parent `thread_ts` (or the parent `ts` from the hit).
2. **Standalone channel message** (no `thread_ts`): group by `(channel_id, ts)`.
3. **DM / group DM**: same grouping; channel_id may be a DM channel or user_id.

Skip when:

- Channel name matches any `slack.skip_channel_patterns` substring (case-insensitive).
- Your only contribution is trivial (shorter than `slack.min_message_length` chars, no replies, no decisions) — e.g. "thanks", "+1", emoji-only.
- The thread permalink or `(channel_id, thread_ts)` already appears on the target day page.

### What to summarize

For each included thread or substantive standalone message, write a **work-focused** summary from (priority order):

1. **Your messages** — what you asked, proposed, decided, or committed to.
2. **Thread context** — key replies from others that change the outcome (decisions, blockers, owners).
3. **Action items** — if a clear ★-worthy follow-up emerges, add to **Outstanding** on that day page (see **Starred lines = tasks**); do not duplicate if already listed.

Omit: banter, pure acknowledgments, bot notifications unless you materially responded.

Map each item to the **calendar day of your latest message** in that thread during the sync window (timezone: `google_calendar.timezone`).

### Note format

Short thread → **Completed** bullet on the matching day page:

```html
<li><p>Aligned with SecEng on GCP logging export scope for CORPSE-97 — <em>(Slack #corpsec-engineering)</em> <a href="https://rippling.slack.com/archives/C01234567/p1234567890123456?thread_ts=1234567890.123456&amp;cid=C01234567">thread</a></p></li>
```

Standalone channel message (no thread):

```html
<li><p>Posted BQ export design update to #data-platform — <em>(Slack)</em> <a href="https://rippling.slack.com/archives/C01234567/p1234567890123456">message</a></p></li>
```

**Permalink**: prefer `permalink` from search/thread results. If missing, build from `slack.workspace_url`:

```
{workspace_url}/archives/{channel_id}/p{ts_without_dot}
```

Append `?thread_ts={parent_ts}&cid={channel_id}` for threaded links.

Dedupe by permalink or `(channel_id, thread_ts)` already on the day page. If a line exists for the same thread, **update** the summary instead of duplicating.

### Long threads → child page

When `slack_read_thread` returns more than `slack.child_page_threshold_messages` messages, or the summary needs more than ~2 short paragraphs:

1. `createConfluencePage` under the **day page** — title: `{channel_name} slack {date_suffix}` or first line of parent message (truncated).
2. Body: TOC + **Context** (channel, participants, link to Slack thread) + **Summary** bullets + optional **Your messages** excerpt.
3. Link from **Completed**: `Discussed GCP logging sync in <a href="{child_page_url}">Slack thread</a> (<a href="{slack_permalink}">Slack</a>)`.

### Private content

Do not paste secrets, credentials, or PII into Confluence beyond what is already normal for your work notes. When a thread is sensitive (HR, personal), summarize at a high level or skip unless the user explicitly asks to include it.

## Jira linking (manual)

- Reference tickets inline on **day pages** or child pages.
- Use `getJiraIssue` / `searchJiraIssuesUsingJql` for summary and status when the user wants context.
- Create tickets only when explicitly asked (`createJiraIssue`).

## Confluence edit rules

- **Always** `getConfluencePage` with `contentFormat: html` before `updateConfluencePage`.
- Pass the **full** body back; partial patches are not supported.
- **One page per write**: fetch and update a single **day page** or **child page** — never the hub plus all days in one call.
- Preserve all `data-local-id` values from the fetched HTML.
- Set `versionMessage` to a short description of the edit.
- On validation errors, fix HTML nesting per tool error text and retry.
- If `updateConfluencePage` fails with a size/payload error, stop editing the hub — run **Migrate legacy monolithic week** or confirm you are on a day page.

### Page layout: table of contents

Every **new** Confluence page (`createConfluencePage`) must begin with a **table of contents** at the top, before other content.

**Week hub** — TOC indexes **Days** and other `<h2>` sections only (no inline daily Completed/Outstanding):

```html
<div data-type="extension" data-extension-key="toc" data-extension-type="com.atlassian.confluence.macro.core" data-parameters="{&quot;outline&quot;:true,&quot;maxLevel&quot;:3}"></div>
```

**Day pages** — TOC indexes the date `<h1>`, **Completed**, **Outstanding**, and any `<h2>` sections on that page.

**Child pages** (meetings, task groups, sync notes) — same TOC macro at top; it indexes **Meeting details**, **Notes**, and other `<h2>` sections.

If the TOC macro is rejected by Confluence HTML validation, use a manual fallback:

```html
<h2>Contents</h2>
<ul>
  <li>Completed</li>
  <li>Outstanding</li>
</ul>
```

On **existing** pages missing a TOC, add the macro (or manual list) at the top on the next edit.

## Google Drive & Brag Doc

MCP server: `user-google-drive`. Authenticate with `mcp_auth` if needed.

| Task | Tool |
|------|------|
| Find Brag Doc / files | `search_drive_files` |
| Read doc text | `get_drive_file_content` |
| Download image/PDF | `get_drive_file_download_url` |
| Draft brag entry (when asked) | `update_drive_file` with `content` — **only after user confirms** |

Brag Doc constants are in `config.yaml` → `google_drive.brag_doc`. Link it from the week hub or first day page; preserve existing links when editing.

**Brag doc workflow** (when user asks to add an entry):

1. `get_drive_file_content` → read current structure and tone.
2. Draft a bullet: impact, scope, outcome; link the Jira ticket.
3. Show draft to user; on approval, `update_drive_file` appending markdown content.

## Additional resources

- MCP tools, CQL examples, HTML snippets: [reference.md](reference.md)
- Routing and trigger phrases: [../manifest.yaml](../manifest.yaml)
