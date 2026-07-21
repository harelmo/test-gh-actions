# Auto-approve + author self-merge — setup

PRs that change **only** files under `WATCH_DIR` (default `docs/`) are auto-approved
by `github-actions[bot]`. With branch protection requiring 1 approval, the PR author
can then merge their own PR. Any PR touching a file outside the folder gets no approval
and stays blocked.

Three parts must all be in place.

## 1. The workflow — `.github/workflows/auto-approve.yml`

```yaml
name: Auto approve
on:
  pull_request:
    types: [opened, synchronize, reopened]
permissions:
  contents: read
  pull-requests: write
env:
  WATCH_DIR: docs/          # only PRs touching solely this folder get approved
jobs:
  auto-approve:
    runs-on: ubuntu-latest
    steps:
      - name: Only approve if every changed file is under WATCH_DIR
        id: check
        env: { GH_TOKEN: "${{ github.token }}" }
        run: |
          files=$(gh api --paginate \
            "repos/${{ github.repository }}/pulls/${{ github.event.pull_request.number }}/files" \
            -q '.[].filename')
          outside=$(printf '%s\n' "$files" | grep -v "^${WATCH_DIR}" || true)
          if [ -z "$files" ] || [ -n "$outside" ]; then
            echo "approve=false" >> "$GITHUB_OUTPUT"
          else
            echo "approve=true" >> "$GITHUB_OUTPUT"
          fi
      - name: Approve
        if: steps.check.outputs.approve == 'true'
        uses: hmarr/auto-approve-action@v4
        with: { github-token: "${{ github.token }}" }
```

It lists the PR's changed files; if **all** are under `WATCH_DIR`, `github-actions[bot]` approves.

## 2. Repo setting (Settings → Actions → General)

**Allow GitHub Actions to create and approve pull requests** = ON

```sh
gh api -X PUT repos/OWNER/REPO/actions/permissions/workflow \
  -F default_workflow_permissions=write \
  -F can_approve_pull_request_reviews=true
```

Without this the bot's approval silently fails.

## 3. Branch protection on `main`

- **Require a pull request before merging** → **Require approvals: 1**
- *Require approval from someone other than the last pusher* = **OFF** — this is what lets the author self-merge
- `enforce_admins` — your call (ON = rules also apply to admins, no bypass)

```sh
cat > bp.json <<'JSON'
{
  "required_status_checks": null,
  "enforce_admins": true,
  "required_pull_request_reviews": {
    "required_approving_review_count": 1,
    "dismiss_stale_reviews": false,
    "require_code_owner_reviews": false
  },
  "restrictions": null
}
JSON
gh api -X PUT repos/OWNER/REPO/branches/main/protection --input bp.json
```

## How it works

The bot is a **separate identity** from the PR author, so there's no self-approval block —
it satisfies the 1 required approval. The author (write access) then merges their own
docs-only PR. Non-docs PRs get no approval and stay blocked.

## Notes / caveats

- `WATCH_DIR` match is a prefix, so keep the trailing slash (`docs/`) or a folder like
  `docsx/` would also pass.
- The approver is `github-actions[bot]`. A PR authored by that same bot **cannot** be
  auto-approved (GitHub blocks self-approval); human-authored PRs are fine.
- Turning *"Require approval from someone other than the last pusher"* ON would block the
  author from self-merging even with the bot's approval — leave it OFF for this flow.
