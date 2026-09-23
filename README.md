# review-with-clickup

A [Claude Code](https://claude.com/claude-code) skill that reviews a pull request against
its ClickUp card.

It reads the card in full (task, subtasks, comment threads, attachments), interviews you
until the acceptance criteria are clear, and only then reads the diff. Then it runs two
reviews and reports each acceptance criterion as met, not met, or unclear, with
`file:line` evidence.

## Usage

```
/review-with-clickup https://github.com/<owner>/<repo>/pull/1813
/review-with-clickup 1813
/review-with-clickup ABC-123
/review-with-clickup https://app.clickup.com/t/86a1x2y3z
```

A PR is enough. The skill finds the ClickUp card from the PR body, the branch name
(`ABC-123/...`) or the title.

## Flow

0. Ask where the result goes: chat only, or chat and a PR comment.
1. Read the ClickUp card in full, and rename the session to `Review ABC-123 <task name>`.
2. Find the repo, the PR and the diff range, and confirm them with you.
3. Read the existing code on the base branch — not the diff.
4. Grill you on the requirement (`mattpocock-skills:grilling`) and write the agreed
   acceptance criteria to a spec file.
5. Standards and spec review (`mattpocock-skills:code-review`).
6. Bug review (the built-in `code-review` skill).
7. Report. A PR comment is posted only after you approve the draft.

## Requirements

- Claude Code, with its built-in `code-review` skill.
- The [mattpocock-skills](https://github.com/mattpocock/skills) plugin, for `grilling` and
  `code-review`.
- A ClickUp MCP server that provides the `clickup_*` tools.
- The GitHub CLI (`gh`), logged in.

## Install

```bash
git clone https://github.com/userhighdef/skill-review-with-clickup ~/.claude/skills/review-with-clickup
```
