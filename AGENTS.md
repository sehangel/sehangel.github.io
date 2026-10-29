# AGENTS.md

<!-- kanban-sync:begin (synced from sehangel/claude-skills -- edit there, not here) -->

## Work tracking (hard rule)

GitHub Issues in this repository are the project memory and Kanban board.
Follow the `issues-as-memory` skill (`.claude/skills/issues-as-memory/SKILL.md`,
synced from `sehangel/claude-skills`) and this repository's Kanban profile
(`.claude/kanban-profile.md`), which holds what is specific to this repo.

- Every operator request, bug report or decision is captured as an issue (or
  a comment on the matching one) in the same turn.
- Every open issue has exactly one `status:*` label (`backlog`, `todo`,
  `in-progress`, `review`, `qa`, `done`, and any stage the profile adds;
  plus `blocked` with who unblocks it) and exactly one `priority:P0`-`P3`
  label. Work is picked P0 first.
- Closing an issue moves its label too (GitHub never does): completed work
  ends on `status:done` (or the profile's next stage), checked right after
  every merge.
- PRs reference every issue they handle: `Closes #n` for each issue they
  deliver, `Refs #n` for epics and for work an operator or a later PR
  finishes. Other repos' issues use the full `owner/repo#n` form.
- Every issue is complete on its own: a Status block at the top of its body
  (stage, owner, next step, what is done, what was decided and why),
  rewritten at each step, and a comment feed of decisions with their why,
  before-and-after evidence, hand-overs, blockers and lessons; nothing
  conversational. Anything saved to agent memory is written on the issue too.
- Labels and progress comments stay current as you build, test and ship.
  Don't start comments with the date (GitHub timestamps them); dates that
  are evidence (incident times, deadlines) are fine.
- Read the open issues at the start of every session and after every context
  summary.
- Never end a turn while authorized, unblocked work is untouched: work it
  yourself, or record on the issue why it can't move; don't spawn sub-agents
  just to clear the board. An open issue alone is not permission to start
  unrelated or production-affecting work.
- No GitHub access in the session (a read-only review or `@claude` job, no
  login)? Do the work, skip the issue steps you can't run, and list the
  updates you couldn't make in the PR or final message.
- Rules in this file, the profile or a repo SOP may be stricter than the
  skill, never looser. Never put secrets or personal data in an issue.

<!-- kanban-sync:end -->
