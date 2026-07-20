# note-bot — agent harness

This repo orients agentic harnesses (Cursor, Flue, Pi, CLI agents) around **weekly work notes** stored in Confluence (Atlassian Cloud). Jira, Confluence, GitHub, Gmail, Google Calendar, Google Drive, Rippling, and Slack are accessed **only via MCP** — never via browser automation unless the user explicitly asks.

## Start here

1. Read [`.cursor/skills/manifest.yaml`](.cursor/skills/manifest.yaml) and match the user's request to a skill loop.
2. For weekly-report work, **read and follow** [`.cursor/skills/weekly-notes/SKILL.md`](.cursor/skills/weekly-notes/SKILL.md) immediately.
3. Load site constants from [`.cursor/skills/weekly-notes/config.yaml`](.cursor/skills/weekly-notes/config.yaml) before any Confluence or Jira call.

## MCP servers (required)

| Server | Purpose |
|--------|---------|
| `user-atlassian` | Confluence pages, Jira issues, search |
| `user-github` | Pull requests authored by you, commits, PR metadata |
| `user-gmail` | Email search, attachment download for handwritten notes |
| `user-google-drive` | Google Docs/Drive — Brag Doc, file search, note images |
| `user-google-calendar` | Calendar events — meeting context, participants, rooms, placeholders |
| `user-rippling-mcp` | Rippling onboarding/IT/HR tasks via `code` + `ask_ai` |
| `user-slack` | Your Slack messages and threads — search, read, summarize for daily notes |

Authenticate with `mcp_auth` when `STATUS.md` says the server needs it. Always read each tool's JSON schema under the MCP descriptors folder before calling.

## Skill routing

| User intent | Skill | Config |
|-------------|-------|--------|
| Update today's notes, add a day page, log tasks | `weekly-notes` | `config.yaml` |
| Meeting notes, 1:1 prep, meet-the-team pages | `weekly-notes` → child-page workflow | `config.yaml` |
| Handwritten note photo → text in Confluence | `weekly-notes` → image/OCR workflow | `config.yaml` |
| New week / new weekly report hub | `weekly-notes` → new-week workflow | `config.yaml` |
| Migrate oversized legacy week page | `weekly-notes` → migrate legacy monolithic week | `config.yaml` |
| Jira ticket context for a note | `weekly-notes` → Jira linking | `config.yaml` |
| Sync Jira ticket activity into daily notes | `weekly-notes` → Jira ticket sync | `config.yaml` |
| Sync GitHub PRs into daily notes + Jira comments | `weekly-notes` → GitHub PR sync | `config.yaml` |
| Enrich meeting notes from calendar (past meetings) | `weekly-notes` → calendar enrich | `config.yaml` |
| Placeholder child pages for upcoming meetings | `weekly-notes` → calendar placeholders | `config.yaml` |
| Sync Rippling completed tasks into daily notes | `weekly-notes` → Rippling task sync | `config.yaml` |
| Sync Slack threads and your channel messages into daily notes | `weekly-notes` → Slack thread sync | `config.yaml` |
| Brag Doc entry for a completed initiative | `weekly-notes` → brag-doc workflow | `config.yaml` |

Future skills extend [`.cursor/skills/manifest.yaml`](.cursor/skills/manifest.yaml); do not hardcode routing in agent prompts.

## Conventions

- **Fetch before write**: always `getConfluencePage` (HTML) and note `version.number` before `updateConfluencePage`.
- **One page per write**: update a single **day page** or child page — never the week hub with all days (avoids MCP payload limits).
- **Preserve structure**: keep existing `data-local-id` attributes when editing fetched HTML.
- **Table of contents**: every new Confluence page starts with a TOC at the top; add to legacy pages on next edit.
- **Page hierarchy**: week hub (index) → day pages (Completed/Outstanding) → child pages (meetings/tasks under the day).
- **New day**: create a day page under the hub; roll forward the previous day page's **Outstanding** (exclude struck-through items).
- **Child pages** for long meeting or task threads; parent = **day page**, not the hub.
- **Transcribe handwriting**: OCR text is required in the page body even when the image is also embedded.
- **Jira sync**: pull assignee/reporter ticket activity into Completed with inline links; remind about Brag Doc when initiatives hit Done/Resolved.
- **GitHub PR sync**: pull your PRs into Completed, link PRs and Jira keys, comment on associated tickets when `github.comment_on_linked_tickets` is true.
- **Calendar**: match events by date/title to enrich meeting pages with participants, duration, and Zoom/physical location; create placeholder pages for upcoming week meetings; skip commuting, vet, and DNS holds.
- **Rippling sync**: pull completed onboarding/IT/HR tasks via `ask_ai` into Completed; strike through matching Outstanding items.
- **Slack sync**: pull your channel messages and threads you participated in, summarize substantive discussions, and add to Completed with Slack permalinks; long threads → child pages under the day page.

## Cursor Cloud specific instructions

This repo is an **agent skill harness**, not a conventional app. There is no `package.json`, Docker stack, lint/test suite, or local server. “Running” means: load skills from `.cursor/skills/`, read `config.yaml`, and call authenticated MCP tools.

### What to run

| Goal | How |
|------|-----|
| Route work | `AGENTS.md` → `.cursor/skills/manifest.yaml` → matched `SKILL.md` |
| Site IDs | `.cursor/skills/weekly-notes/config.yaml` (`cloud_id`, hub/day `page_id`s) |
| Tool shapes | `.cursor/skills/weekly-notes/reference.md` |
| Lint / unit tests / `dev` server | N/A — none in-repo |
| Sanity check | Parse YAML (`manifest.yaml`, `config.yaml`); exercise MCP reads/writes below |

### MCP server IDs in Cloud Agents

`manifest.yaml` uses ids like `user-atlassian`. In Cursor Cloud the live server names are typically **`Atlassian`**, **`Github`**, **`Slack`**, **`Google Drive`** (no `user-` prefix). Discover with `GetMcpTools` before calling.

**Automations** must attach required MCP servers under Automation → Tools. The daily new-day cron needs **Atlassian** at minimum.

| Capability | Cloud status (as of env setup) |
|------------|--------------------------------|
| Atlassian Confluence **read** (`getConfluencePage`, CQL, descendants) | Works when MCP attached |
| Atlassian Confluence **write** (`updateConfluencePage`, `createConfluencePage`) | **Not exposed** on the Cloud Atlassian MCP binding — day-page creates/updates are blocked until those tools appear |
| Atlassian Jira read/write (search, comments, transitions) | Works (Jira comment write verified) |
| GitHub | Works |
| Slack | Works (`slack_search_public_and_private` for DMs/private; prefer public when enough) |
| Google Drive | Works (Brag Doc readable) |
| Gmail / Google Calendar / Rippling | Often **not** attached in Cloud — skip those skill loops or ask the user to enable the MCP servers |

### Non-obvious gotchas

- Always `getConfluencePage` with `contentFormat: html` and note `version.number` before any update; never dump the full week hub into an update payload.
- If `config.yaml` is missing a day that already exists under the hub, resolve via `getConfluencePageDescendants` and record `page_id` / `web_url` in `weekly_report.current.days`.
- No dependency install step is required on startup; the update script only asserts skill files are present.
- **New week Mondays**: when the calendar date is past the prior `week_label_date`, run the **new week workflow** (new hub + first day page) instead of only adding a day under the old hub.
