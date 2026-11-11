# Git Workflow

**Project**: CodeMie UI · **Stack**: TypeScript React · **Scope**: branch names, commit subjects, merge policy and their recovery steps; exact gate commands live in `.ai-run/guides/quality-gates.md`, the MR adapter in `.ai-run/guides/project.md`

## Branch Naming Convention

- base branch: main
- pattern: `EPMCDME-<n>_<short-kebab-description>`

### Rules
- R1. Create every branch from `main` and name it with the ticket key, an underscore and a short kebab-case description (`EPMCDME-15631_cost-center-filtering`), the form most recent remote branches use; no hook or CI job in this repository checks branch names, so check the name before pushing.

## Commit Message Format

```
EPMCDME-<n>: <Capitalised imperative sentence>
```

Examples: `EPMCDME-15960: Fix InputArray required asterisk size to match other form labels`, `EPMCDME-16851: Turn katas off by default`, `EPMCDME-14536: Decouple Budget Management admin tab from Enterprise`

### Rules
- R1. Write every commit subject as the ticket key with a non-zero number, a colon, a space and a capitalised imperative sentence, first line under 72 characters (`EPMCDME-123: Fix authentication redirect loop`).
- R2. Use `Generate release notes for version <x.y.z>` only for a release commit and `Revert "<original subject>"` only for a revert; both are maintainer formats.
- R3. Check each subject yourself before pushing: `.husky/pre-commit` runs no message check and `.gitlab-ci.yml` only includes an external guard template, so nothing in this repository rejects a malformed subject.

## Merge Strategy

### Rules
- R1. Squash-merge a branch into `main` so it lands as one commit whose subject follows the commit format (recorded user decision; `main` holds no merge commits).
- R2. Open every MR as Draft and mark it Ready only when the work is complete and the companion MRs of the same ticket (UI, SDK, backend) exist, because autoreview and the review pipeline skip Draft MRs; the default template `.gitlab/merge_request_templates/Default.md` ends with the `/draft` quick action, and `glab mr create` needs `--draft`.
- R3. Fill the MR body from `.gitlab/merge_request_templates/Default.md` and keep its `## Test harness` heading and code block, which the MR compliance bot checks; paste the `npm run test-harness` summary there and put screenshots under `## Verified locally`.

## MR State: Draft First

Open every MR as **Draft**. Ready is a decision: mark it Ready when the work is complete and the
companion MRs of the same ticket (UI, SDK, backend) exist, because that is the moment automatic review starts.
Autoreview and the review pipeline skip Draft MRs. With `glab` add `--draft`. In the GitLab UI the default
template `.gitlab/merge_request_templates/Default.md` ends with the `/draft` quick action, so an MR created from it
is Draft (GitLab runs the action on creation and removes it, so later description edits keep the state). Mark it
Ready with `glab mr update <id> --ready` or the "Mark as ready" button.

## Anti-Patterns

- Never commit directly to `main`; branch from it with the ticket key.
- Never bypass the pre-commit hook with `--no-verify`; fix what it reports and commit again.
- Never use a zero ticket number (`EPMCDME-0`); use the real Jira key of the work item.

## Troubleshooting

- When the pre-commit hook fails on licence headers, run `npm run license-headers:fix`, re-stage and commit again.
- When the secrets step fails with `No container engine found` or a stopped daemon, start Docker, Podman or Apple Containers and commit again; `scripts/validate-secrets.mjs` exits 1 in both cases, so the commit is blocked, not skipped.
- When the Sonar step fails before any scan, check `.sonarlint/connectedMode.json`, `sonar-project.properties` and the git HEAD: `scripts/sonar/run-local-sonar.mjs` reads all three before it looks at `SONAR_TOKEN`, so a missing one blocks the commit even without a token.
- Resolve a merge conflict by rebasing the feature branch on `origin/main`, then push it again.
