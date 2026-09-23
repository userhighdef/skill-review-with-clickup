---
name: review-with-clickup
description: Review a branch or PR against its ClickUp card. Takes a PR number, a PR URL, a ClickUp task ID, or a ClickUp URL. Reads the card in full (task, subtasks, comments, attachments), grills the user until the acceptance criteria are settled, then runs mattpocock-skills:code-review (standards + spec) and the built-in code-review (bugs), and reports whether the work meets the requirement. Use when the user says "review ABC-123", "/review-with-clickup 1813", "/review-with-clickup <github PR url>", "review this against the ClickUp card", "/review-with-clickup", or pastes an app.clickup.com task URL and asks for a review.
---

# Review against a ClickUp card

A normal code review checks whether the code is good. This one also checks whether the
code does what the ticket asked for. To do that, the requirement has to be clear first,
and a ClickUp card is rarely clear on its own. So this skill reads the card, settles the
requirement with the user, and only then looks at the diff.

The order matters. The acceptance criteria are fixed **before** anyone reads the diff, so
the review measures the code against the requirement and not the other way round.

## Input

The user passes one of these. Nothing else is needed.

| Input | Example | Gives you |
|---|---|---|
| PR URL | `https://github.com/<owner>/<repo>/pull/1813` | repo + PR |
| PR number | `1813` | PR only — the repo is unknown |
| ClickUp task ID | `ABC-123` | task only |
| ClickUp URL | `https://app.clickup.com/t/86a1x2y3z` | task only |

For a PR number, find the candidate repos (see "Find the repo" in Step 2) and run
`gh pr view <number> --repo <owner>/<repo> --json number,title,url` in each. If only one
matches, use it. If more than one matches, show their titles and ask which one.

For a PR, find the task ID with
`gh pr view <number> --repo <owner>/<repo> --json headRefName,title,body,baseRefName,state`.
Look in this order: an `app.clickup.com/t/...` link anywhere in the body (it is often
written inline, like `Ref: ABC-123 [title](link)`), a custom task ID at the start of the
branch name (`ABC-123/...`), then one at the start of the title. If none of them has one,
ask the user for the ClickUp task. Say which task you found and where you found it.

## Step 0 — Ask where the result goes

Ask before doing any work, with a single `AskUserQuestion`:

- Show in chat only
- Show in chat and comment on the PR

Remember the answer. It decides what step 7 does.

## Step 1 — Read the ClickUp card in full

The real requirement is spread across subtasks, comment threads and attached files, not
only the description. Read all of it.

The task can be a custom task ID (`ABC-123`), a raw ClickUp ID, or a task URL. For a
custom task ID, pass `custom_task_ids: true` together with `team_id`. Get `team_id` at
runtime with `clickup_get_workspaces` rather than hardcoding it — it is the same value
`clickup_get_attachments` wants as `workspace_id`.

### Rename the session first

As soon as the first `clickup_get_task_details` call returns the custom task ID and name,
rename the session — before reading subtasks, comments or attachments. Call
`mcp__ccd_session_mgmt__set_session_title` with `session_id: "self"` and this title:

```
Review ABC-123 Disable carrier dropdown on invalid rows
```

`Review`, a space, the custom task ID, a space, then a short version of the task name —
about six words. The `Review` prefix keeps this session apart from any other session
working on the same card. Skip it if the tool is not available; it exists only in the
Claude Code desktop app.

### Read everything

Keep the raw task ID and the task `url` from the `clickup_get_task_details` response.
`clickup_get_attachments` has no `custom_task_ids` option, so its `entity_id` must be the
raw ID, and the report links to the card by its `url`.

| What | Tool |
|---|---|
| Main task | `clickup_get_task_details` with `include_markdown_description: true`, `include_subtasks: true` |
| Each subtask | `clickup_get_task_details` **and** `clickup_get_task_comments`, one call per subtask |
| Comments | `clickup_get_task_comments`, then `clickup_get_threaded_comments` for any comment with replies |
| Attachments | `clickup_get_attachments` with `entity_type: "task"`, then download and read each file |
| Custom fields | `clickup_get_task_custom_field_values` |
| Related work | `clickup_get_task_dependencies` |

Two traps:

- `include_subtasks: true` returns only a short summary of each subtask, without its
  description or comments. Fetch each subtask on its own.
- `clickup_get_task_comments` does not return replies, so a thread with five replies looks
  like one comment. Follow up with `clickup_get_threaded_comments` when a comment has them.

Attachments often define the acceptance criteria better than the description does — a
screenshot of the bug, a sheet of test cases, an API collection.

## Step 2 — Pick the repo and the fixed point

### Find the repo

If the input was a PR URL, the repo is in the URL. Otherwise:

1. If the current directory is a git repo, it is the only candidate.
2. If not, every git repo one level below the current directory is a candidate
   (`git -C <dir> rev-parse --is-inside-work-tree`).
3. With more than one candidate, pick the one that fits what the card describes, say why,
   and let the user confirm. Guessing wrong here wastes the whole run.

Get `<owner>/<repo>` with `git -C <repo> remote get-url origin`. The shell may go back to
its start directory after every call, so from here on use `git -C <repo>` for every git
command and `--repo <owner>/<repo>` for every `gh` command.

### Find the range

If there is no PR from the input yet, look for one with the custom task ID in its branch
name or title:

```
gh pr list --repo <owner>/<repo> --search "<customTaskId>" --state open --json number,headRefName,baseRefName,url
```

- If there is a PR, the fixed point is `origin/<baseRefName>` and the head is its branch.
- If there is none, the head is the current branch and the fixed point is the default
  branch: `origin/$(gh repo view <owner>/<repo> --json defaultBranchRef -q .defaultBranchRef.name)`.

Run `git -C <repo> fetch origin`. If the head branch is not checked out, run
`git -C <repo> status` first. If there are uncommitted files, name them and ask before
`git -C <repo> switch <branch>` — the user may be mid-work on something else.

If the PR is already merged, tell the user and ask whether to continue — the diff against
the base will be empty or misleading. Then check that
`git -C <repo> diff --stat <fixed-point>...HEAD` is not empty. Say which PR or branch and
which fixed point you picked, and let the user confirm. Reviewing the wrong range wastes
every step after it.

If the user chose "comment on the PR" but there is no PR, tell them now that the result
will stay in chat.

## Step 3 — Read the code the card touches, not the diff

Read the code as it is on the fixed point, with `git -C <repo> show <fixed-point>:<path>`:
entry points, the path a request or job takes, and prior art. Do **not** read the diff or
the working tree yet — the working tree is the head branch, so reading it leaks the diff
into the grill. Give the same rule to any sub-agent you send to find facts in Step 4.

Read enough that your questions in the next step are about decisions, not facts.

## Step 4 — Grill the requirement

Invoke `mattpocock-skills:grilling`. Do not invoke `grill-me` or `grill-with-docs` — both
set `disable-model-invocation: true`, so a skill cannot call them. `grilling` is what
`grill-me` runs.

Do not invoke `domain-modeling`. It writes `CONTEXT.md` and ADRs into the repo, and those
files would show up in the branch under review.

The design tree here is the requirement: what counts as done, edge cases, what is out of
scope, and anything where the card, the subtasks and the comments disagree. When the card
contradicts itself, the user decides which part wins.

When the user confirms shared understanding, write the result to a spec file in the
session's scratchpad directory (or a `mktemp -d` directory if there is none), named
`<customTaskId>-spec.md`:

```markdown
# <customTaskId> <task name>

<ClickUp task url>

## Acceptance criteria
- AC1: <one testable statement>
- AC2: ...

## Out of scope
- ...

## Decisions from grilling
- <question> → <answer>
```

Show the file path and the acceptance criteria to the user. This file is the spec for the
next step.

## Step 5 — Standards and spec review

Invoke `mattpocock-skills:code-review`. In its args, pass:

- The exact diff command, `git -C <repo> diff <fixed-point>...HEAD`, so its sub-agents
  run it in the right repo.
- The spec file path from Step 4 as the spec source (its step 2, option 2).
- An extra line for the Spec sub-agent brief: "Also list every acceptance criterion by ID
  (AC1, AC2, ...), including the ones that are met, with `file:line` evidence for each."
  Its default brief only reports problems, so without this a met AC is indistinguishable
  from one it never checked.

That skill says to stop and ask for `/setup-matt-pocock-skills` when
`docs/agents/issue-tracker.md` is missing. Ignore that check here: the tracker doc only
tells it how to fetch an issue, and this skill already passes the spec directly.

## Step 6 — Bug review

Invoke the built-in `code-review` skill. Pass the PR number as the target if there is a
PR, otherwise the head branch. Never pass `--comment`: it posts to the PR right away,
before the user has seen the findings. Step 7 posts one comment after the user approves.

Check that it reviewed the same commit range as Step 5. If it did not, say so in the
report instead of mixing results from two different diffs.

## Step 7 — Report

Show the result in chat in this order:

1. **Acceptance criteria** — one row per AC from the spec file: `✅ met`, `❌ not met`, or
   `⚠️ unclear`, with `file:line` evidence. Build this from the Spec findings in Step 5.
   An AC with no evidence is `⚠️ unclear`, never `✅ met`.
2. **Spec** and **Standards** — the two reports from Step 5, as that skill presents them.
   Do not merge or rerank them.
3. **Bugs** — the findings from Step 6.
4. One line: how many ACs are met, and the worst issue in each section.

If the user chose "comment on the PR" and a PR exists, draft one PR comment with all four
sections. Start it with a link to the card, `[<customTaskId>](<task url>)`. Show the
draft, and post it with `gh pr comment <number> --repo <owner>/<repo> --body-file <file>`
only after the user says yes.
