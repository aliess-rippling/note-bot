---
name: weekly-notes
description: >-
  Creates and maintains weekly work reports in Confluence (Atlassian), including
  daily sections, completed/outstanding task lists, child pages for meetings and
  tasks, Jira ticket sync, GitHub PR sync with linked work summaries and Jira
  ticket comments, brag-doc reminders on resolved initiatives, handwritten-note
  OCR from Gmail/Drive/local images, and Google Drive doc access. Use when
  updating weekly notes, syncing Jira or GitHub activity, daily logs, meeting
  notes, or transcribing note photos.
---

# Weekly Notes (Confluence)

Maintain **ALiess weekly report** pages in the Rippling personal Confluence space. All Atlassian, GitHub, Gmail, and Google Drive access goes through MCP.

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

**Add a new day**: append after the last day's content (do not reorder past days). Copy **Outstanding** items from the previous day into the new day's **Outstanding** unless the user says they are done.

**Update tasks**: edit in place on the correct day. Move items from Outstanding → Completed (with `<s>` on completed sub-items). Link Jira with inline cards: `<a href="https://rippling.atlassian.net/browse/KEY-123" data-card-appearance="inline">...</a>`. Optionally run **Jira ticket sync** or **GitHub PR sync** to backfill Completed.

**Link child pages** from the daily list: `<a href="https://rippling.atlassian.net/wiki/spaces/.../pages/{id}">Title</a>`.

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
<p><strong>Role:</strong> …</p>
<p><strong>Can help with:</strong> …</p>
<p><a href="https://calendar.google.com/calendar/render?action=TEMPLATE&amp;text=...">📅 Schedule meeting with {Name}</a></p>
<h2>Notes</h2>
<p></p>
```

Fill **Notes** after the call. Keep prep info (role, calendar link) even when notes are empty.

### Sync / working-session template

Short bullets, relevant doc links (Confluence cards, Google Docs, external URLs), optional screenshot. Example: `Axl Daniyal gcp logging sync 26/07/08`.

**Create**: `createConfluencePage` with `parentId` = weekly report or task-group page, `spaceId` from config, `contentFormat: html`.

## Handwritten notes / image OCR workflow

Sources (in order of user hint):

1. **Gmail** — `search_gmail_messages` → `get_gmail_message_content` → `get_gmail_attachment_content` (use `return_base64: true` if sandboxed).
2. **Local directory** — scan paths in `config.yaml` → `handwritten_notes.local_image_dirs`; user may pass an explicit path.
3. **Google Drive** — `search_drive_files` or `config.yaml` file_id → `get_drive_file_content` (text/docs) or `get_drive_file_download_url` (images).

**Transcription** (required):

1. Open the image with the Read tool (vision) or decode base64 to a temp file then Read.
2. Transcribe faithfully; mark illegible spans as `[illegible]`.
3. Structure as headings/bullets matching the handwriting.

**Publish**:

| User request | Action |
|--------------|--------|
| Short note | Add transcribed text under today's `<h1>` on the weekly report |
| Long or multi-topic | Create child page; link from daily section |
| User wants image preserved | Add transcription under a `## Transcription` or `## Notes` heading; embed image if possible (see below) |

**Image embed limitation**: Atlassian MCP has no attachment-upload tool. Prefer full transcription in the body. If the user requires the image on the page, either (a) they upload via Confluence UI while you add the text, or (b) store in Google Drive via MCP and link the Doc/Drive URL in the page. Existing pages use `<figure data-type="media-single">` with server-assigned `data-id` — do not invent media IDs.

## New week workflow

1. Compute `week_label_date` as `YY/MM/DD` of that week's Friday (see config).
2. `createConfluencePage`:
   - `title`: `ALiess weekly report {week_label_date}`
   - `parentId`: `reports_parent_folder_id` from config
   - `spaceId`: from config
   - `body`: first day `<h1>` + empty Completed/Outstanding or user's opening notes
3. Update `config.yaml` → `weekly_report.current` with new `page_id`, `title`, dates, `web_url`.
4. Leave prior week's page unchanged (historical record).

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
