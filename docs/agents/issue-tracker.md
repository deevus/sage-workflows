# Issue tracker: GitHub

Issues and PRDs for this repo live as GitHub issues in `deevus/sage-workflows`. Use the `gh` CLI for all issue operations.

## Conventions

Pass `--repo deevus/sage-workflows` explicitly so issue operations target the public GitHub repo even when the local `origin` remote points elsewhere.

- **Create an issue**: `gh issue create --repo deevus/sage-workflows --title "..." --body "..."`. Use a heredoc or temporary body file for multi-line bodies.
- **Read an issue**: `gh issue view <number> --repo deevus/sage-workflows --comments`, including labels where relevant.
- **List issues**: `gh issue list --repo deevus/sage-workflows --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'` with appropriate `--label` and `--state` filters.
- **Comment on an issue**: `gh issue comment <number> --repo deevus/sage-workflows --body "..."`
- **Apply / remove labels**: `gh issue edit <number> --repo deevus/sage-workflows --add-label "..."` / `--remove-label "..."`
- **Close**: `gh issue close <number> --repo deevus/sage-workflows --comment "..."`

## When a skill says "publish to the issue tracker"

Create a GitHub issue in `deevus/sage-workflows`.

## When a skill says "fetch the relevant ticket"

Run `gh issue view <number> --repo deevus/sage-workflows --comments`.
