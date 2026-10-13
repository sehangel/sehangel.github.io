---
name: issues-as-memory
description: Turn every operator request into a GitHub issue and track it on a Kanban (status:* + priority:P0-P3 labels, ordered by priority and dependencies) so work survives agent context loss. Use at the start of every session, after every context summary, when starting/finishing/blocking any task, when opening a PR, and before ending a turn. Triggers -- what's left, what's next, backlog, kanban, update the ticket, log this, track this, resume work.
---

<!-- synced from sehangel/claude-skills -- edit there (issues-as-memory/SKILL.md), not here: the next sync overwrites this copy. -->

# GitHub Issues are the memory

Agents lose context; GitHub issues do not. In every repository, the
repository's own GitHub issue tracker is the single source of truth for what
is done, in progress, blocked and left. Roadmaps, `CURRENT_STATE` files and
chat are summaries that point there.

## First: read the repo's profile

This skill is the same in every repository. What is specific to one
repository lives in its **Kanban profile**, `.claude/kanban-profile.md` at the
repository root. Read it before using this skill. It says:

- what `priority:P0` and `status:qa` mean there, and any extra labels (areas,
  `epic`, legacy labels and how they map to the ones below);
- shared resources (production services, devices, deploy pipelines) and who
  may change them;
- deploy and verification rules, and the repo's lessons learned;
- stricter rules (a Kanban SOP, a triage runbook) that override this skill.

A repository may make these rules stricter, never looser. A profile adds to
the definitions here (e.g. more P0 conditions); it never narrows them. If the
profile is missing, use this skill as written and open an issue to add the
profile.

If the repository has its own `.claude/skills/issues-as-memory/SKILL.md`
without the "synced from sehangel/claude-skills" header, that is an older
repo-specific version: read it too, and follow whichever rule is stricter.

Set the repository once per session and use it in every command:

```bash
R=$(gh repo view --json nameWithOwner --jq .nameWithOwner)   # e.g. sehangel/ci-burst
```

### When GitHub is out of reach

The tracking rules never block the work itself:

- No `gh` login in the session (e.g. a cloud sandbox): use the GitHub MCP or
  connector tools for the same operations.
- No write access, or not even read access to the issue list (a read-only
  automated review or `@claude` job, an offline session): do the task, skip
  the steps you can't run (the session-start and end-of-turn sweeps
  included), and list the issue updates you could not make (new issues,
  label moves, comments) in the PR body or the final message, so the next
  session with access applies them. Automated PR-review jobs only review;
  they don't open or edit issues.

## Labels

Exactly one stage label per open issue:

| Label                | Meaning                                                                                           |
| -------------------- | ------------------------------------------------------------------------------------------------- |
| `status:backlog`     | Known, not prioritized                                                                            |
| `status:todo`        | Prioritized, ready to start                                                                       |
| `status:in-progress` | Someone is working on it now                                                                      |
| `status:review`      | PR open, waiting on review/CI                                                                     |
| `status:qa`          | Merged or deployed; needs a live, device or production check (the profile says what the check is) |
| `status:done`        | Verified; close the issue with evidence                                                           |
| `blocked`            | Add alongside a stage label; say why and who unblocks                                             |

Exactly one priority label per open issue:

| Label         | Meaning                                                                                                   | Rule                                                                                          |
| ------------- | --------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `priority:P0` | Urgent: production broken, users blocked, runaway spend, security or data risk (the profile may add more) | Drop everything. Pull the agent from lower work, fix, verify, close; then resume the pipeline |
| `priority:P1` | Release-blocking or next up                                                                               | Work before any P2/P3                                                                         |
| `priority:P2` | Normal planned work                                                                                       | Default for new tasks                                                                         |
| `priority:P3` | Backlog / nice to have                                                                                    | Only when nothing P0-P2 is actionable                                                         |

Also `bug` for defects. Initiatives use a checklist of child issues (the
`epic` label where the repo has it). Keep other classification labels the
repo uses (areas, `documentation`, `enhancement`, provenance); the
one-label rule applies only to `status:*` and `priority:*`. If a label is
missing, create it: `gh label create "<label>" -R $R --force` (the sync
script creates the standard set in every repo).

Moving an issue means removing whatever label of that family (stage or
priority) it has now, not the one you expect it to have:

```bash
n=18; new=status:in-progress            # or priority:P1
prefix="${new%%:*}:"                     # status: or priority:
for l in $(gh issue view $n -R $R --json labels --jq ".labels[].name | select(startswith(\"$prefix\"))"); do
  gh issue edit $n -R $R --remove-label "$l"
done
gh issue edit $n -R $R --add-label "$new"
```

Move the stage label only when the issue crosses a stage boundary (started,
PR opened, merged, verified), not after every commit. When `blocked` no longer
applies, remove it. A reopened issue loses `status:done` and gets the stage
that matches its remaining work.

## Every operator request is a task (hard rule)

Everything the operator asks for or decides in a session is captured on
GitHub before (or while) acting, never only in chat:

- **Request** ("fix X", "add Y", "switch Z") -> search for an existing issue
  (`gh issue list -R $R --search "<words>" --state all --limit 200`); update
  it, reopening it if it was closed but the work is not done, or open a new
  one with a `status:*` and a `priority:*` label (plus any label the profile
  requires) in the same turn. Small requests finished in the same turn still
  get an issue (or a comment on the matching one) with the evidence.
- **Bug report** (even a vague one: "it freezes", "the button does nothing")
  -> `bug` issue with what the operator saw, then the evidence you collect
  (logs, screenshots, queries). Production-impacting or launch-blocking bugs
  are P0/P1.
- **Decision** ("ship to beta only", "use provider X", "don't mention Y") ->
  comment on every issue it affects, and update the issue title or checklist
  if the scope changed. A durable decision also goes where future sessions
  read it: the profile or the repo playbook the profile names, and agent
  memory where the session has it.
- **Question** -> answer it. If it is about an issue, add the question and
  answer there; if the answer exposes work, open an issue. A pure question
  with no issue and no work needs no new issue.
- **Operator-owned steps**: only what needs the operator's own login,
  identity or money, such as a dashboard or store click with no API or CLI we
  hold credentials for, a sign-in, a value only they hold, or new spend.
  These go in an issue with `blocked` naming the operator and the exact
  command or click path. Steps we can do with existing credentials (CLI, API,
  repo scripts), and secrets a repo script generates, are ours (see "Keep the
  operator's work minimal").

## Ordering the work (Kanban)

Pick work in this order, and say the order when reporting:

1. Any `priority:P0`, even if it means pausing an in-flight agent or other
   work (or taking back a shared device). Announce the interrupt on the
   paused issue.
2. A ready `priority:P1` before in-progress P2/P3 work: pause the lower one
   with a comment saying where it stands.
3. Otherwise `status:in-progress` before starting new work (limit WIP: finish
   or hand off before starting more than one task per worker or device).
4. `priority:P1`, then P2, then P3, within each `status:todo` before
   `status:backlog`.
5. Respect dependencies: write `Blocked by #n` / `Depends on #n` in the body;
   don't start a task whose dependency is open unless the work is independent.
   Example sequence: fix -> merge -> deploy -> live or device check -> close.
6. Re-prioritise when the operator changes goals (e.g. "release only to beta"):
   relabel affected issues in the same turn.

## Shared resources (one owner at a time)

Production services, deploy pipelines, test devices and store consoles are
shared. The profile lists them. Each shared resource has one place where its owner is
recorded (the profile names it: a pinned issue per device or service, or the
coordinator); claim it there, re-read it, and hand over explicitly ("device
yours" / "device free"). Never act on a resource someone else holds without a
hand-off. P0 work may preempt,
after a message to the owner.

## Loop

1. **Start of session / after a summary**: `gh issue list -R $R --state open --limit 1000`
   and read the epics or initiatives. Pick the next `status:in-progress` or
   `status:todo` item. Never start from memory alone.
2. **Before working**: search first (`gh issue list -R $R --search "<words>" --state all --limit 200`).
   If no issue covers the task, open one. Move it to `status:in-progress`.
3. **While working**: at every meaningful step (built, tested, deployed, found a
   blocker) add a comment: what changed, evidence (command + result,
   commit SHA, URL), and what is next. This is what a future agent will read.
4. **New work discovered**: open a new issue immediately (follow-ups, bugs,
   decisions the operator owes). Link it from the epic checklist.
5. **PR**: one branch per task in its own worktree (unless the repo's rules
   allow direct commits). Move the issues the PR delivers (`Closes`) to
   `status:review`; issues it only references (`Refs`) keep the status that
   matches their remaining work. The PR body references **every** issue the
   PR handles or touches, one keyword per line, so nothing is left open by
   accident:
   - `Closes #n` (the default) for each issue the PR delivers. GitHub
     closes it only when the PR merges into the **default branch**; a PR
     into another branch (a release branch) needs the issues closed by hand
     after the merge.
   - Still needs a live, device or production check after merge? By default,
     close the issue anyway and put the remaining check on a follow-up issue
     that the PR body links with `Refs #m`. Where the profile or a repo rule
     keeps the delivered issue open until it is verified (e.g. a crash fix
     that must be confirmed in the field), link it with `Refs #n` instead,
     move it to `status:qa` after the merge, and close it by hand with the
     proof.
     If other issues say `Blocked by #n` or `Depends on #n` because of that
     check, point them at the issue that carries the check, so closing `#n`
     doesn't unblock them early.
   - `Refs #n` only for issues the PR advances but cannot finish, because
     the rest is an operator or external action (a dashboard click, a store
     approval, a later PR). Say in the issue what remains.
   - Epics are never closed by a PR: use `Refs #epic` and tick its
     checklist. GitHub does not close an epic on its own: when its last
     child closes, move the epic to `status:done`, then close it
     (`gh issue close <epic> -R $R --reason completed`) with a one-line summary.
   - Each issue appears once in the body, with one keyword: don't write
     both `Refs #n` and `Closes #n`.
   - After opening or editing the PR, check the links:

     ```bash
     PR=$(gh pr view --json number --jq .number)  # PR of the current branch
     gh api graphql -F owner=${R%/*} -F name=${R#*/} -F pr=$PR -f query='query($owner:String!,$name:String!,$pr:Int!){repository(owner:$owner,name:$name){pullRequest(number:$pr){closingIssuesReferences(first:100){nodes{number repository{nameWithOwner}}}}}}' \
       --jq '[.data.repository.pullRequest.closingIssuesReferences.nodes[] | "\(.repository.nameWithOwner)#\(.number)"]'
     ```

     The list must contain every `Closes` issue, with the right repository.
     A PR into a non-default branch has no closing links at all (GitHub
     ignores the keywords there), so skip this check for it and close its
     issues by hand after the merge.
     `Refs` lines don't show up there, so also read them from the body:
     `gh pr view $PR -R $R --json body --jq .body | grep -oiE 'refs [a-z0-9/._-]*#[0-9]+'`.
     When you rewrite a PR body from a file, keep these lines.

6. **After merge / deploy**: confirm the `Closes` issues closed and move
   them from `status:review` to `status:done` (GitHub doesn't change labels).
   Move issues that wait on verification to `status:qa` until verified, then
   `status:done` and close with the evidence. Other `Refs` issues keep a
   stage label that matches what remains (e.g. `status:todo`), plus `blocked`
   when waiting on the operator or an external party, with a comment saying
   what is left.
7. **Blocked**: add `blocked`, comment the reason and who unblocks it.
8. **Before ending a turn**: every issue you touched has a current label and a
   comment saying where it stands.

## Checkpoints (hard rule)

Context can be lost at any moment, so progress lives in issues, not in the chat:

- **After every finished task or sub-task** (a commit, a deploy, a fix, a
  review, a decision from the operator) -- stop and update the issue(s) it
  touched before starting the next thing: a comment with what changed,
  evidence (commit SHA / deploy version / test result / URL) and what is next;
  move the `status:*` label if the issue crossed a stage boundary; tick the
  epic checklist.
- **Every 10 minutes during long sessions** -- a session reminder (`/loop 10m`
  or a session cron) prompts a checkpoint sweep: go through the issues you
  touched since the last checkpoint (keep their numbers in your notes; this
  repo's `status:in-progress` list helps but is not enough), make sure each
  has a current comment and label, and open issues for any new work or
  operator request not yet tracked.
- **When delegating to subagents** -- tell them which issue to comment on, and
  after they report, record their result on the issue yourself.
- **Operator decisions** (design choices, scope changes, "don't do X") are
  recorded on the relevant issue the moment they are made.

## Keep working -- the open issues are the to-do list (hard rule)

Finishing the task in front of you is not the end of the turn. The operator
expects the board to keep moving without having to say "keep working":

- **Never end a turn while authorized, actionable work is open.** Before
  stopping, run `gh issue list -R $R --state open --limit 1000` (check that
  the count is below the limit) and walk every issue that is not `blocked`
  and not waiting on the operator. For each one in scope (see below): work it
  now, delegate it to a background agent (own worktree), or record on the
  issue why it cannot move (with `blocked` + who unblocks it).
- **In scope** means: the session's request, issues you or the session own,
  and the queue the operator has authorized (standing instructions or the
  profile). An open issue on its own is not permission to start
  production-affecting or unrelated work: record it, and ask. `status:backlog`
  items the operator deferred stay deferred.
- **A finished task triggers the next one.** After each checkpoint, pick the
  next in-scope `status:in-progress`, then `status:todo`, then
  `status:backlog` item and start it in the same turn.
- **Waiting is not a reason to stop.** While a release, CI run or subagent is
  running, work another issue in parallel. Only stop when every in-scope
  issue is closed (done ones closed with their evidence),
  `status:review`/`status:qa` with its PR or check in flight, or blocked on
  the operator/an external party, and say which ones in the final message.
- **Self-reminder loop.** At the start of a long session, schedule a recurring
  work sweep (session cron every 10 min, off the :00/:30 marks, or
  `/loop 10m`) whose prompt is: "Sweep <owner/repo> open issues: checkpoint
  touched ones, then pick up any unblocked in-scope issue nobody is working
  on and continue it." The sweep both checkpoints and resumes work; a plain
  "checkpoint" reminder is not enough.
- **"Continue" / "keep working" from the operator** means: re-read every open
  issue and work all actionable ones, not only the last task discussed.
- **End-of-turn report** lists open issues by state: worked this turn,
  in review/QA, blocked (on whom), so nothing is silently dropped.

## Lessons from multi-agent sessions

Learned when one coordinator ran up to eight agents across several
repositories at once.

### Many agents at once

- **One owner per issue and PR.** The coordinator assigns issues before it
  dispatches agents. An agent that picks up an issue on its own comments
  "Owner: <agent>, worktree <path>", then re-reads the comments: if another
  "Owner:" comment came first after the latest "Released:" comment, that
  earlier claim wins and it backs off. When an
  owner stops or is stopped, it comments "Released: <state>": what's
  mid-flight, prod changes not yet restored, a device it holds. Never put two
  agents on one issue.
- **Taking over from a stale owner:** if an owner has posted nothing for
  2 hours and is no longer running (the coordinator checks its agent list),
  the coordinator may reassign. Post "Taken over from <agent>: <state found>".
  Restore any temporary prod change that agent left before continuing.
- **One production deploy at a time, granted by one coordinator.** This
  covers service deploys, rollouts and prod flag flips. A "check, then merge"
  by each agent races: two agents can both see an idle pipeline and merge.
  So the **coordinator** is the single lock:
  1. An agent asks the coordinator for the slot.
  2. The coordinator grants it to one agent at a time.
  3. The holder confirms the deploy pipeline is idle before merging, by
     checking the builds still in progress, not only what is already
     serving traffic (the profile says where to look).
  4. The holder merges, runs its post-deploy watch, and releases the slot
     back to the coordinator, also when the deploy failed.

  The slot is not permission to merge: merge only where the operator has
  authorized it (the profile says when agents may merge, e.g. never into a
  release branch without the operator's OK for that PR).

  Without a coordinator, use one pinned "deploy slot" issue as a queue.
  Comment "Deploy slot: claimed by <agent>", then re-read the comments: the
  oldest claim after the last "Deploy slot: released" comment holds the slot;
  everyone else waits. The holder comments "Deploy slot: released" after its
  watch, or on failure. If the repo can run deploys in a CI job, a
  `concurrency` group is the stronger lock.

- **Agents share one scratch folder:** give your helper scripts and outputs
  unique names (prefix them with your agent name). One agent overwrote
  another's watch script, and a deploy watch read the wrong output.
- **Priority beats queue order.** A ready P0/P1 fix takes the next deploy slot
  ahead of ready P3 PRs.
- **Pausing:** when the operator says pause, first let each agent reach a
  cleanup point: restore and read back any temporary production change
  (flags, caps, test servers, a borrowed device). Then stop it. If an agent
  has to be stopped mid-test, the coordinator restores that change itself.
  Then leave a "paused" comment with the state on each issue that was in
  flight.

### Keep the operator's work minimal

- **Before labelling `blocked` on the operator, check whether we can do it.**
  These are ours:
  - sysadmin: setup scripts, secrets generated by repo scripts, server
    config (through the infra repo), monitoring dashboards;
  - facts we already know: costs, what's retired, measured numbers;
  - sensible defaults.

  Only these are the operator's: their identity (passwords, 2FA, sign-ins
  with their accounts, their saved credentials), money (new spend, paid
  services), legal or counsel calls, and values only they hold (a third-party
  key they signed up for).

- **Production-wide changes still need the operator's per-change approval**:
  flags every user sees, live capacity limits, opening or closing a region,
  draining servers. The profile or a test skill may list more, and may name
  automations the operator approved in advance. An approval covers only the
  change it was given for; record it on the issue.
- **Decisions:** ask with concrete options and a recommendation. Record the
  answer on the issue at once. If the operator reverses it, record the
  correction on the same issue.
- **Identity actions** (sign-ins, account choosers, OAuth consent) need the
  operator's own approval in the session that acts. A subagent will rightly
  refuse an approval relayed by the coordinator: the coordinator does the step
  itself, or the operator does it.
- **Questions first:** answer an operator question at once, then go back to
  the work.

### Evidence, QA and production tests

- **Device or visual checks: no screenshot means no proof.** The proof (or a
  small GIF) is embedded in the comment that ticks the checklist item. Crop
  or blur personal data, tokens and account details first.
- **A fix needing live or device proof** follows the PR rule above: the
  check lives on a `status:qa` issue (the follow-up, the QA queue issue where
  the repo has one, or the delivered issue itself where the profile says so).
  Tick and close it only with a link to the proof.
- **Every prod config change made for a test:** record the state before,
  restore it after, read it back, and note all of it on the issue.
- **Measure, then label:** a QA failure becomes its own `bug` issue with
  numbers, linked to the check that found it, and goes to whoever owns that
  area.
- **Unit tests are not proof of a live fix.** Changes that touch startup,
  connect, auth, deploys or infrastructure get a live or device check before
  their QA issue is `status:done`.

### Incidents

- **Anything odd in production** (unexpected servers, alerts, a platform
  outage) gets an issue at once, P0 or P1, with a timeline and evidence.
  Check the vendor's status page and say on the issue whether the vendor has
  reported it.
- **A vendor support ticket:** draft its text in the issue (or a file), then
  file it with the operator's login.
- **Each incident becomes a test or simulator scenario,** so it can't come
  back unseen.

### Cross-repo work

- **Parent and children:** a parent issue in the product repo, and a child
  issue in each repo it touches, linked both ways.
- **Missing labels:** create any the target repo lacks before labelling,
  naming the repo: `gh label create "<label>" -R <owner>/<child-repo>
--force`.
- **Cross-repo references** use the full form, `owner/repo#n`, in issue
  bodies, PR bodies and comments. A bare `#n` links to an unrelated issue in
  the current repo.
- **Local checkouts of other repos go stale.** Read other repos' files with
  `gh api "repos/<owner>/<repo>/contents/<path>?ref=<branch-or-sha>" -H "Accept: application/vnd.github.raw"`;
  without `ref` you get the default branch, not an in-flight PR's version.

### Backlog hygiene

- **Triage sweep, at least daily:**
  - close what's done, with evidence;
  - close what's superseded or obsolete as not planned, naming the decision
    that ended it;
  - merge duplicates;
  - fix label drift: every open issue has exactly one `status:*` and one
    `priority:*` label, and issues filed by automations (templates, alert
    bots) get theirs too.
- **Postponed features** keep their design (prototype links, the agreed
  approach) in a `status:backlog` issue, so picking them up later redoes
  nothing.

## Comment style

GitHub timestamps every comment and label change, so don't start comments
with the date or time of writing. Start with the substance: what changed,
evidence, what's next. Dates that are evidence belong in the text: event
times in an incident timeline, a deadline and its source, a historical
start or finish date, an external timestamp (a log line), and any dated
header a repo workflow requires.

## Commands

```bash
R=$(gh repo view --json nameWithOwner --jq .nameWithOwner)
gh issue create -R $R --title "..." --label "status:todo,priority:P2" --body "..."
gh issue list -R $R --state open --label priority:P0 --limit 1000   # anything urgent?
gh issue comment 18 -R $R --body "Deployed abc123; /healthz 200. Next: ..."
gh issue close 18 -R $R --reason completed --comment "Verified: ..."
gh issue close 18 -R $R --reason "not planned" --comment "Why: ..."
# label drift: open issues without exactly one status and one priority label
gh issue list -R $R --state open --limit 1000 --json number,labels --jq \
  '.[] | select(([.labels[].name | select(startswith("status:"))] | length) != 1
            or ([.labels[].name | select(startswith("priority:"))] | length) != 1) | .number'
```

(Stage and priority moves: see "Labels" above.) If `gh issue edit` fails
with a "Projects (classic)" GraphQL error (older `gh`), use the REST API:
`gh api -X POST repos/$R/issues/18/labels -f 'labels[]=status:todo'` and
`gh api -X DELETE repos/$R/issues/18/labels/status:backlog`.

Never put secrets, tokens or user personal data in an issue. In a public
repository, also keep internal design, infrastructure details and client
data out of issues.

## Improving this skill

This skill is maintained in one place, `sehangel/claude-skills`
(`issues-as-memory/SKILL.md`), and mirrored into every repository by
`issues-as-memory/sync/sync.py`, which opens one PR per repository when the
canonical text changes. Never edit a repository's synced copy: it is
overwritten on the next sync. To refine the method:

1. Change `issues-as-memory/SKILL.md` (or the "Work tracking" block in
   `issues-as-memory/sync/work-tracking-block.md`) in a claude-skills PR.
2. After it merges, run `python3 issues-as-memory/sync/sync.py` from a
   claude-skills checkout. It skips repositories already in sync and updates
   the open sync PR instead of opening a second one.

A lesson that only applies to one repository goes in that repository's
`.claude/kanban-profile.md` instead.
