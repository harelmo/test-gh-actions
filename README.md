# test-gh-actions

PRs that change **only** files under `docs/` are auto-approved by `github-actions[bot]`.
With branch protection requiring 1 approval, the PR author can then merge their own PR.

Approve logic: `.github/workflows/auto-approve.yml` (edit `WATCH_DIR` to change the folder).
