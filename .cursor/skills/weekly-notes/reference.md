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

Drive search example: `name = 'Brag Doc' and mimeType = 'application/vnd.google-apps.document'`

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
- [ ] Links to Jira/Confluence/Docs added where handwriting references them
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
