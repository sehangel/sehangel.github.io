# AGENTS.md

<!-- kanban-sync:begin (synced from sehangel/claude-skills -- edit there, not here) -->
## Work tracking (hard rule)

GitHub Issues in this repository are the project memory and Kanban board.
Follow the `issues-as-memory` skill (`.claude/skills/issues-as-memory/SKILL.md`,
synced from `sehangel/claude-skills`) and this repository's Kanban profile
(`.claude/kanban-profile.md`), which holds what is specific to this repo.

- Every operator message is a task: capture it as an issue (or a comment on
  the matching one) in the same turn.
- Every open issue has exactly one `status:*` label (`backlog`, `todo`,
  `in-progress`, `review`, `qa`, `done`; plus `blocked` with who unblocks it)
  and exactly one `priority:P0`-`P3` label. Work is picked P0 first.
- PRs reference every issue they handle: `Closes #n` for each issue they
  deliver, `Refs #n` for epics and for work an operator or a later PR
  finishes. Other repos' issues use the full `owner/repo#n` form.
- Labels and progress comments stay current as you build, test and ship.
  Never put dates in comments (GitHub timestamps them).
- Read the open issues at the start of every session and after every context
  summary.
- Never end a turn while an unblocked open issue is untouched: work it,
  delegate it, or record on the issue why it can't move.
- Rules in this file, the profile or a repo SOP may be stricter than the
  skill, never looser. Never put secrets or personal data in an issue.
<!-- kanban-sync:end -->
