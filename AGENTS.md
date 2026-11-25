# AGENTS.md

<!-- kanban-sync:begin (synced from sehangel/skills -- edit there, not here) -->

## Work tracking (hard rule)

GitHub Issues in this repository are the project memory and Kanban board.
Follow the `issues-as-memory` skill (laptop-wide at `~/.claude/skills/issues-as-memory/SKILL.md`;
elsewhere read it from `sehangel/skills`) and this repository's Kanban profile
(`.claude/kanban-profile.md`), which holds what is specific to this repo.

- Every operator request, bug report or decision is captured as an issue (or
  a comment on the matching one) in the same turn.
- An issue's stage is GitHub's native issue status (Backlog, Todo, In
  progress, In review; Done is a close as completed). GitHub moves a new
  issue and a `Closes` PR's issue itself, and agents never set those; what
  agents set is the skill's list ("Setting the stage"). QA and docs, which have no status, are the labels
  `status:qa` and `status:docs` on an issue In review. The old `status:*`
  stage labels are retired: never set them by hand. Every open issue has
  exactly one `priority:P0`-`P3` label, plus `blocked` with who unblocks it.
  Work is picked P0 first. Never claim by assignment.
- Completed work is closed as completed (the close sets Done) with no QA or
  docs label left and nothing unticked, or moved to the profile's next
  stage; checked right after every merge.
- PRs reference every issue they handle. An issue closes only after its
  proof, never because a PR merged: `Refs #n` for every issue that still owes
  a check (QA, docs, a post-merge proof), for epics and for work an operator
  or a later PR finishes; `Closes #n` only with a line `QA: none — <why>`
  (Golzak Bot reopens a `Closes` without it on `status:qa` where its
  `qaOwedReopen` is on). Other repos' issues use the full `owner/repo#n` form.
- Issues link natively: an issue blocked by another gets GitHub's "blocked
  by" link (the label bot sets and clears `blocked` from it), and an epic's
  children are its sub-issues.
- Work the task you were given first; with none, pick from the repo's
  `ready-queue` issue (within what the operator authorized) and
  `/claim <session name>` it; `/release` when you stop. A PR labelled `waiting:author` is yours to
  fix (conflicts included). Merge a PR only when the operator authorizes it
  in chat; there is no `/merge` command, and the bot never merges.
- Every issue is complete on its own: a Status block at the top of its body
  (stage, owner, next step, what is done, what was decided and why),
  rewritten at each step, one checklist in the body (a comment never
  carries a box; Golzak Bot reopens an issue closed with one unticked), and
  a comment feed of decisions with their why, before-and-after evidence,
  hand-overs, blockers and lessons; nothing conversational. Anything saved to agent memory is written on the issue too.
- A change that runs is proven before its PR opens (a fresh verifier that
  did not write it runs it and gives PASS or FAIL; the PR body carries the
  `Pre-PR verdict:` line) and again after the merge, on what shipped (a
  `Post-merge proof:` comment), before the issue leaves `status:qa`: the
  skill's "Prove the change". For an app on any platform, what shipped
  is a local release build of the merge commit on that platform's test
  device: never wait for a store release to prove a change (only a check
  that needs a store-signed build waits, on its own follow-up issue).
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
