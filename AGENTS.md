# note-bot — agent harness

This repo orients agentic harnesses (Cursor, Flue, Pi, CLI agents) around **weekly work notes** stored in Confluence (Atlassian Cloud). Jira, Confluence, GitHub, Gmail, Google Calendar, Google Drive, and Rippling are accessed **only via MCP** — never via browser automation unless the user explicitly asks.

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

Authenticate with `mcp_auth` when `STATUS.md` says the server needs it. Always read each tool's JSON schema under the MCP descriptors folder before calling.

## Skill routing

| User intent | Skill | Config |
|-------------|-------|--------|
| Update today's notes, add a day section, log tasks | `weekly-notes` | `config.yaml` |
| Meeting notes, 1:1 prep, meet-the-team pages | `weekly-notes` → child-page workflow | `config.yaml` |
| Handwritten note photo → text in Confluence | `weekly-notes` → image/OCR workflow | `config.yaml` |
| New week / new weekly report page | `weekly-notes` → new-week workflow | `config.yaml` |
| Jira ticket context for a note | `weekly-notes` → Jira linking | `config.yaml` |
| Sync Jira ticket activity into daily notes | `weekly-notes` → Jira ticket sync | `config.yaml` |
| Sync GitHub PRs into daily notes + Jira comments | `weekly-notes` → GitHub PR sync | `config.yaml` |
| Enrich meeting notes from calendar (past meetings) | `weekly-notes` → calendar enrich | `config.yaml` |
| Placeholder child pages for upcoming meetings | `weekly-notes` → calendar placeholders | `config.yaml` |
| Sync Rippling completed tasks into daily notes | `weekly-notes` → Rippling task sync | `config.yaml` |
| Brag Doc entry for a completed initiative | `weekly-notes` → brag-doc workflow | `config.yaml` |

Future skills extend [`.cursor/skills/manifest.yaml`](.cursor/skills/manifest.yaml); do not hardcode routing in agent prompts.

## Conventions

- **Fetch before write**: always `getConfluencePage` (HTML) and note `version.number` before `updateConfluencePage`.
- **Preserve structure**: keep existing `data-local-id` attributes when editing fetched HTML.
- **Table of contents**: every new Confluence page starts with a TOC at the top; add to legacy pages on next edit.
- **New day**: roll forward the previous day's **Outstanding** list (exclude struck-through items).
- **Child pages** for long meeting or task threads; keep the weekly report scannable with links.
- **Transcribe handwriting**: OCR text is required in the page body even when the image is also embedded.
- **Jira sync**: pull assignee/reporter ticket activity into Completed with inline links; remind about Brag Doc when initiatives hit Done/Resolved.
- **GitHub PR sync**: pull your PRs into Completed, link PRs and Jira keys, comment on associated tickets when `github.comment_on_linked_tickets` is true.
- **Calendar**: match events by date/title to enrich meeting pages with participants, duration, and Zoom/physical location; create placeholder pages for upcoming week meetings; skip commuting, vet, and DNS holds.
- **Rippling sync**: pull completed onboarding/IT/HR tasks via `ask_ai` into Completed; strike through matching Outstanding items.
