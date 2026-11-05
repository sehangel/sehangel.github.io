---
name: qa-evidence
description: Post before-and-after proof (screenshots, GIFs, log excerpts) for an issue or PR, in any repository - upload it to the repository's add-only `qa-evidence` branch and embed it in the issue comment. Use whenever a QA check, a bug repro, a fix or a device test needs proof on GitHub. Covers the file layout, the add-only upload (contents API or a detached worktree), privacy (blur IDs, e-mails, IPs, names), sizes, and the link format. Triggers -- QA evidence, screenshot proof, before and after, attach a screenshot, post evidence, qa-evidence branch, embed image in issue.
---

<!-- synced from sehangel/claude-skills (issues-as-memory/sync/opt-in/qa-evidence/SKILL.md) -- edit there, not here -->

# QA evidence: before-and-after proof on the `qa-evidence` branch

Every repository keeps its proof (screenshots, GIFs, log excerpts) on one
orphan branch, **`qa-evidence`**, that is never merged, and links it from
its issues and PRs. A QA pass, a bug repro or a fix carries proof, not just
"works": the state before, the state after, and the check that shows it.

The branch is **add-only**: every issue's images live on it, so a commit
that drops or changes files breaks all of them (once, two hand-built
commits replaced the tree with two files and 938 images went 404). A
ruleset, `qa-evidence-add-only`, blocks force pushes and deleting the
branch. The sync (sehangel/claude-skills) creates the branch, its README
and the ruleset where they are missing.

## 1. Make the files

- **Layout:** `qa/<issue>/<YYYYMMDD>-<check-slug>-before.<ext>` and
  `…-after.<ext>`; a single shot is `…-<check-slug>.<ext>`.
- **Size:** at most 1080 px on the long side (`sips -Z 1080 <file>` on macOS,
  or Pillow); a screen recording becomes a short GIF (at most about 5 MB).
  Logs go in a text file or straight into the comment.
- **Privacy, before anything is uploaded:** nothing may show an account or
  device ID, a token, a public IP, an e-mail address, or a person's name or
  photo. Blur or crop it (Pillow `GaussianBlur` on the box), or leave the
  shot out. Scrub log excerpts the same way (`<redacted>`). In a public
  repository, upload nothing from a real account.

## 2. Upload: add files, never rewrite the branch

Either:
- **the contents API**, one new path per call (no checkout needed):

  ```bash
  R=<owner/repo>; f=<local file>; p=qa/<issue>/<name>
  gh api -X PUT "repos/$R/contents/$p" -f branch=qa-evidence \
    -f message="QA evidence: #<issue> <check>" -f content="$(base64 -i "$f")"
  ```

- **or a detached worktree** on `origin/qa-evidence`:
  `git fetch origin qa-evidence && git worktree add --detach <dir> origin/qa-evidence`,
  copy the new files in, `git add` them, commit, then check
  `git diff --name-status origin/qa-evidence HEAD` lists only `A` lines, and
  `git push origin HEAD:qa-evidence`.

Never force-push it, never merge it, never delete or overwrite a file on it,
and never build its commits by hand (`git mktree`, `commit-tree`, a new
orphan branch). A path already taken gets a new name (`…-2.png`).

## 3. Link it

Embed each file in the issue or PR comment that reports the check:

```markdown
| Before | After |
| --- | --- |
| ![<check> before](https://github.com/<owner>/<repo>/blob/qa-evidence/qa/<issue>/<file>-before.png?raw=true) | ![<check> after](https://github.com/<owner>/<repo>/blob/qa-evidence/qa/<issue>/<file>-after.png?raw=true) |
```

One pass/fail line per checked item goes with it. The link opens for
anyone who can read the repository.

## If the branch is missing

The sync creates it. If a repository has none yet, create it the same way
(an orphan commit holding only `README.md`, then the
`qa-evidence-add-only` ruleset: no deletion, no non-fast-forward), or ask
for a sync run.
