---
description: Create a pull request from the current branch against the default target branch (usually `dev`)
argument-hint: (no args)
allowed-tools:
  - Read
  - Bash
---

# /pr-template

Create a pull request from the current branch against `dev` (adjust if your project uses `main` as the default merge target).

## Steps

1. Read the PR template at `.github/pull_request_template.md` (fall back to no template if the file doesn't exist)
2. Run in parallel:
   - `git log dev..HEAD --oneline` — list commits on this branch
   - `git diff dev...HEAD` — see actual changes
   - `git rev-parse --abbrev-ref HEAD` — get the current branch name
3. Extract the ticket number from the branch name using the project's convention:
   - Feature branch: `feature/<TICKET_PREFIX>-*` → e.g. `feature/APP-123-add-profile`
   - Bugfix branch: `bugfix/<TICKET_PREFIX>-*` → e.g. `bugfix/APP-456-fix-crash`
   - The regex is `<TICKET_PREFIX>-\d+`. Adjust the placeholder to your project's actual ticket prefix (e.g. `APP-123`, `PROJ-456`, `JIRA-789`).
4. Fill in the template:
   - **Summary**: replace `<TICKET_PREFIX>-000` (or the equivalent placeholder in your template) with the extracted ticket number and its tracker link, then describe the change in 1–3 sentences
   - **Changes**: list what was added / modified / removed (bullets)
   - **Type of Change**: check the appropriate box (feature / bugfix / refactor / docs / chore)
   - **Testing / Checklist**: check items that apply based on the actual state — do not check boxes for things you didn't verify
5. Output the filled PR title and body so the user can review before creating

If the user says "go ahead", create the PR using `gh pr create --base dev` and assign it to the current git user.

## Notes

- PR titles follow `<TICKET_PREFIX>-XXX: short description` — same format as commit messages
- Never force-push. Never merge. Just create the draft.
- If the branch doesn't match the ticket regex, ask the user for the ticket number rather than proceeding without one
