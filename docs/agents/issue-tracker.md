# Issue tracker: GitHub

Issues and PRDs for this repo live as GitHub issues. Use the `gh` CLI for all operations.

Repo: `pepearayao/calendar-wiz` (public). **Because the repo is public, its issues are public — never paste secrets, tokens, or OAuth client secrets into an issue.**

## Conventions

- **Create an issue**: `gh issue create --title "..." --body "..."`. Use a heredoc for multi-line bodies.
- **Read an issue**: `gh issue view <number> --comments`.
- **List issues**: `gh issue list --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'`.
- **Comment on an issue**: `gh issue comment <number> --body "..."`
- **Apply / remove labels**: `gh issue edit <number> --add-label "..."` / `--remove-label "..."`
- **Close**: `gh issue close <number> --comment "..."`

Infer the repo from `git remote -v` — `gh` does this automatically when run inside a clone.

## When a skill says "publish to the issue tracker"

Create a GitHub issue.

## When a skill says "fetch the relevant ticket"

Run `gh issue view <number> --comments`.

## Wayfinding operations

Used by `/wayfinder`. The **map** is a single issue with **child** issues as tickets.

Both the sub-issue and issue-dependency endpoints were verified working on this repo (2026-08-22).

- **Map**: a single issue labelled `wayfinder:map`, holding the Destination / Notes / Decisions-so-far / Fog body. `gh issue create --label wayfinder:map`.
- **Child ticket**: an issue linked to the map as a GitHub **sub-issue**. Link with the child's numeric **database id** (`gh api repos/<owner>/<repo>/issues/<n> --jq .id`):
  `gh api --method POST repos/<owner>/<repo>/issues/<map>/sub_issues -F sub_issue_id=<child-db-id>`.
  Labels: `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`). Once claimed, the ticket is assigned to the driving dev.
- **Blocking**: GitHub **native issue dependencies** (UI-visible). Add an edge with
  `gh api --method POST repos/<owner>/<repo>/issues/<child>/dependencies/blocked_by -F issue_id=<blocker-db-id>`
  where `<blocker-db-id>` is the blocker's numeric **database id** (not `#number`, not `node_id`).
  Read the live gate with `gh api repos/<owner>/<repo>/issues/<n> --jq .issue_dependencies_summary.blocked_by` (counts open blockers). A brief replication lag on read-after-write is possible; re-read to confirm.
- **Frontier query**: the map's open children with no open blocker and no assignee; first in map (creation) order wins. Verified command:
  `gh api repos/<owner>/<repo>/issues/<map>/sub_issues --jq '[.[] | select(.state=="open" and .issue_dependencies_summary.blocked_by==0 and (.assignee==null)) | {number, title}] | .[]'`
- **Claim**: `gh issue edit <n> --add-assignee @me` — the session's first write.
- **Resolve**: `gh issue comment <n> --body "<answer>"`, then `gh issue close <n>`, then append a context pointer (gist + link) to the map's Decisions-so-far.
