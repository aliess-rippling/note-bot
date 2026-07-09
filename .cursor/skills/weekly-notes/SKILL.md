---
name: weekly-notes
description: >-
  Creates and maintains weekly work reports in Confluence (Atlassian), including
  daily sections, completed/outstanding task lists, child pages for meetings and
  tasks, Google Calendar context for past and upcoming meetings, Rippling completed-task
  sync, Jira ticket sync, GitHub PR sync with linked work summaries and Jira ticket comments, brag-doc
  reminders on resolved initiatives, handwritten-note OCR from Gmail/Drive/local
  images, and Google Drive doc access. Use when updating weekly notes, syncing
  calendar meetings, creating meeting placeholders, syncing Rippling or Jira or
  GitHub activity, daily logs, meeting notes, or transcribing note photos.
---

# Weekly Notes (Confluence)

Maintain **ALiess weekly report** pages in the Rippling personal Confluence space. All Atlassian, GitHub, Gmail, Google Calendar, Google Drive, and Rippling access goes through MCP.

## Quick start

```
Task Progress:
- [ ] Read config.yaml (current page_id, cloud_id)
- [ ] Authenticate MCP servers if needed (mcp_auth)
- [ ] getConfluencePage → fetch full HTML body + version
- [ ] Plan edits (preserve data-local-id on existing nodes)
- [ ] updateConfluencePage or createConfluencePage
- [ ] Verify with getConfluencePage or share webUrl
```

Load [config.yaml](config.yaml) before every session. For HTML patterns and MCP tool args, see [reference.md](reference.md).

## Resolve the active weekly report

1. Use `config.yaml` → `weekly_report.current.page_id`.
2. If missing or user says "new week", run **New week workflow** below.
3. Optionally confirm with CQL: `title ~ "ALiess weekly report" AND space = "~7120204850b617efd944b9ae686dff14ee52b5" ORDER BY lastmodified DESC`.

## Daily section workflow

Each calendar day is an `<h1>` with a Confluence date:

```html
<h1><time datetime="2026-07-09">July 9, 2026</time></h1>
```

After the first onboarding-style days, prefer this structure under each day:

| Section | Heading | Content |
|---------|---------|---------|
| Narrative | (optional `<p>`) | Short context at top of day |
| Done | `<h2>Completed</h2>` | Bullet list; strike through with `<s>` when done |
| Open | `<h2>Outstanding</h2>` | Nested `<ul>` for subtasks |

### Add a new day (required)

When adding a new daily `<h1>`, **always roll forward Outstanding** from the **immediately previous day**:

1. `getConfluencePage` → find the prior day's `<h2>Outstanding</h2>` and its `<ul>` (including nested sub-items).
2. **Copy** that list into the new day's **Outstanding** — full tree, same wording and links.
3. **Omit** any line that is struck through with `<s>` (treat as done).
4. Leave the **previous day's** Outstanding unchanged (historical snapshot).
5. Append the new `<h1>` after the last day; do not reorder earlier days.
6. Add new tasks below the copied list; do not duplicate items already copied.

Skip roll-forward only when the user explicitly says prior Outstanding is cleared or not applicable.

**Update tasks**: edit in place on the correct day. Move items from Outstanding → Completed (with `<s>` on completed sub-items). Link Jira with inline cards: `<a href="https://rippling.atlassian.net/browse/KEY-123" data-card-appearance="inline">...</a>`. Optionally run **Rippling task sync**, **Jira ticket sync**, or **GitHub PR sync** to backfill Completed.

**Link child pages** from the daily list: `<a href="https://rippling.atlassian.net/wiki/spaces/.../pages/{id}">Title</a>`.

## Starred lines = tasks

A **star** (★, `*`, or a clear star doodle) beside a line marks an **open task** — in handwriting, on a meeting child page, or anywhere in notes.

| Where the star appears | What to do |
|------------------------|------------|
| Weekly report / daily scratch pad | Add line to that day's **Outstanding** |
| Meeting child page **Notes** | Keep starred line in **Notes** *and* add the task to the day's **Outstanding** on the weekly report |
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
| Multi-person task thread (e.g. meet-the-team) | Weekly report | `{YY/MM/DD} task {topic}` |
| Individual intro / 1:1 | Task group page | `Meet the team: {Name}` |
| Ad-hoc meeting or deep dive | Weekly report | `{Who} {topic} {YY/MM/DD}` |

Use `getConfluencePageDescendants` to avoid duplicates. Link new pages from the relevant **Completed** or **Outstanding** line on the daily section.

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

**Create**: `createConfluencePage` with `parentId` = weekly report or task-group page, `spaceId` from config, `contentFormat: html`. Start the body with a **table of contents** (see Page layout below).

## Google Calendar workflow

MCP server: `user-google-calendar`. Authenticate with `mcp_auth` if needed. Read tool schemas before calling (primary tool: `get_events` with `detailed: true`).

Use calendar to (a) enrich past meeting notes with metadata, and (b) create **placeholder child pages** for upcoming meetings in the current week.

### When to consult calendar

| Trigger | Action |
|---------|--------|
| Handwritten note has a **date + title** | `get_events` that day → fuzzy-match title → enrich note |
| Meeting child page exists, call already happened | Match by title + date → add participants, duration, location |
| User asks to prep upcoming meetings | `get_events` for the work week → create placeholder pages |
| New day section / weekly setup | Optionally seed **Outstanding** with linked upcoming meetings |

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

1. `getConfluencePageDescendants` on the weekly report — dedupe by similar title + date.
2. `createConfluencePage` with placeholder body (no notes yet).
3. Link from that day's **Outstanding** on the weekly report.

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
3. If unmatched → add transcription to daily section or child page without fabricated metadata.

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
| Short note | Add transcribed text under today's `<h1>` on the weekly report |
| Starred task line | Add to **Outstanding** for that day (see **Starred lines = tasks**) |
| Long or multi-topic | Create child page; link from daily section |
| User wants image preserved | Add transcription under a `## Transcription` or `## Notes` heading; embed image if possible (see below) |

**Image embed limitation**: Atlassian MCP has no attachment-upload tool. Prefer full transcription in the body. If the user requires the image on the page, either (a) they upload via Confluence UI while you add the text, or (b) store in Google Drive via MCP and link the Doc/Drive URL in the page. Existing pages use `<figure data-type="media-single">` with server-assigned `data-id` — do not invent media IDs.

## New week workflow

1. Compute `week_label_date` as `YY/MM/DD` of that week's Friday (see config).
2. `createConfluencePage`:
   - `title`: `ALiess weekly report {week_label_date}`
   - `parentId`: `reports_parent_folder_id` from config
   - `spaceId`: from config
   - `body`: **TOC macro** + first day `<h1>` + empty Completed/Outstanding or user's opening notes
3. Update `config.yaml` → `weekly_report.current` with new `page_id`, `title`, dates, `web_url`.
4. Leave prior week's page unchanged (historical record).

## Rippling task sync workflow

Pull **completed Rippling tasks** (onboarding checklist, IT provisioning, HR forms, trainings, action items) into the weekly report **Completed** section. MCP server: `user-rippling-mcp` via the `code` tool and `codemode.*` sandbox.

```
Task Progress:
- [ ] Read config.yaml (rippling.completed_tasks_prompt, week_start)
- [ ] code → codemode.lookup_me (confirm signed-in worker)
- [ ] code → codemode.ask_ai (completed tasks since week_start)
- [ ] Poll ask_ai across code calls if status is "running"
- [ ] Parse task table: title, completion date, category, rippling_url
- [ ] Match existing Outstanding lines → move to Completed with <s>
- [ ] Dedupe by task title already on report
- [ ] updateConfluencePage
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

- Place on the **completion date** from Rippling (correct daily `<h1>` section).
- If the task already appears in **Outstanding**, strike through and move to **Completed** instead of duplicating.
- Link `rippling_url` from the ask_ai response when provided; otherwise link `rippling.profile_url` or omit.

### Outstanding sync (optional)

When user asks, run a second `ask_ai` query for **incomplete** onboarding/IT/HR tasks and add missing items to **Outstanding** (do not mark complete).

## Jira ticket sync workflow

Pull the user's Jira activity for the current week (or a specific day) and add **work done** to the weekly report with **inline ticket links**. Run on request or at the end of a daily-notes update when the user wants Jira synced.

```
Task Progress:
- [ ] Read config.yaml (jira JQL, brag_doc_issue_types, week_start)
- [ ] atlassianUserInfo → confirm current user
- [ ] searchJiraIssuesUsingJql (assignee + reporter queries)
- [ ] getJiraIssue per candidate (fields: summary, status, comment, updated, resolution, issuetype)
- [ ] Extract user-authored comments in date range → work-done bullets
- [ ] Dedupe against issue keys already on the weekly report
- [ ] Append to today's Completed (or correct day) with Jira inline cards
- [ ] Flag newly Done/Resolved initiatives → brag-doc reminder (below)
- [ ] updateConfluencePage
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

Pull **your** pull requests updated in the sync window, summarize work completed, add lines to the weekly report, and **comment on linked Jira tickets**. MCP server: `user-github`.

```
Task Progress:
- [ ] Read config.yaml (github search templates, week_start)
- [ ] get_me → GitHub login
- [ ] search_pull_requests (author + updated since)
- [ ] pull_request_read (get, get_commits, get_files) per PR
- [ ] Extract Jira keys from title, body, branch, commits
- [ ] Summarize work done → Completed on matching day
- [ ] addCommentToJiraIssue on each linked ticket (dedupe)
- [ ] Merged PR + resolved Jira → brag-doc reminder if applicable
- [ ] updateConfluencePage
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

Map each PR to the **day it was last updated** (or merged date) for the correct daily section.

### Note format

```html
<li><p>{summary} (<code>{repo}</code> <a href="{pr_html_url}">#{number}</a>) — <a href="https://rippling.atlassian.net/browse/{KEY}" data-card-appearance="inline">https://rippling.atlassian.net/browse/{KEY}</a></p></li>
```

When no Jira key is found, omit the ticket link and keep the PR link. Dedupe by PR URL or `#number` already on the report.

Long PRs → child page `{repo}#{number} {title}` with body excerpt + file list; link from Completed.

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

## Jira linking (manual)

- Reference tickets inline on the weekly report or child pages.
- Use `getJiraIssue` / `searchJiraIssuesUsingJql` for summary and status when the user wants context.
- Create tickets only when explicitly asked (`createJiraIssue`).

## Confluence edit rules

- **Always** `getConfluencePage` with `contentFormat: html` before `updateConfluencePage`.
- Pass the **full** body back; partial patches are not supported.
- Preserve all `data-local-id` values from the fetched HTML.
- Set `versionMessage` to a short description of the edit.
- On validation errors, fix HTML nesting per tool error text and retry.

### Page layout: table of contents

Every **new** Confluence page (`createConfluencePage`) must begin with a **table of contents** at the top, before other content.

**Weekly report** — TOC lists day sections (`<h1>`) and updates as days are added:

```html
<div data-type="extension" data-extension-key="toc" data-extension-type="com.atlassian.confluence.macro.core" data-parameters="{&quot;outline&quot;:true,&quot;maxLevel&quot;:3}"></div>
```

**Child pages** (meetings, task groups, sync notes) — same TOC macro at top; it indexes **Meeting details**, **Notes**, and other `<h2>` sections.

If the TOC macro is rejected by Confluence HTML validation, use a manual fallback:

```html
<h2>Contents</h2>
<ul>
  <li>July 9, 2026</li>
</ul>
```

On **existing** pages missing a TOC, add the macro (or manual list) at the top on the next edit. When adding a new day to the weekly report, ensure the TOC block remains the first element in the body.

## Google Drive & Brag Doc

MCP server: `user-google-drive`. Authenticate with `mcp_auth` if needed.

| Task | Tool |
|------|------|
| Find Brag Doc / files | `search_drive_files` |
| Read doc text | `get_drive_file_content` |
| Download image/PDF | `get_drive_file_download_url` |
| Draft brag entry (when asked) | `update_drive_file` with `content` — **only after user confirms** |

Brag Doc constants are in `config.yaml` → `google_drive.brag_doc`. The weekly report already links it from day-one notes; preserve that link when editing early sections.

**Brag doc workflow** (when user asks to add an entry):

1. `get_drive_file_content` → read current structure and tone.
2. Draft a bullet: impact, scope, outcome; link the Jira ticket.
3. Show draft to user; on approval, `update_drive_file` appending markdown content.

## Additional resources

- MCP tools, CQL examples, HTML snippets: [reference.md](reference.md)
- Routing and trigger phrases: [../manifest.yaml](../manifest.yaml)
