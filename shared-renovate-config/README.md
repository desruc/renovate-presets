# Shared Renovate config

Shared Renovate presets and workflows for Shared repositories. This holds the must-haves every repo
should get, plus the auto-merge approval flow. Anything specific to one repo stays in that repo's `renovate-config.js`.

Every repo references this at `main`, so a change to these presets or the Renovate workflows reaches all of them on their next Renovate run.

## Presets

| Preset                | Use for                                        |
| --------------------- | ---------------------------------------------- |
| `default.json5`       | Any repo. The base the other presets extend.   |
| `github-action.json5` | JavaScript GitHub Actions that ship a `dist/`. |
| `dotnet.json5`        | .NET repos.                                    |

The presets live in `shared-renovate-config/`. The reusable workflows live in the repo root's
`.github/workflows/`, because GitHub only loads reusable workflows from there.

Extend exactly one. Where a rule belongs:

- Every repo needs it: `default.json5`.
- Every repo of one type needs it: that ecosystem preset.
- Otherwise: the repo's own `renovate-config.js`.

## Using it in a repo

`renovate-config.js`:

```js
module.exports = {
  extends: ['github>desruc/renovate-presets//shared-renovate-config/dotnet.json5'],
  // Repo-specific rules go here.
};
```

`.github/workflows/renovate.yml`:

```yaml
name: Renovate
on:
  schedule:
    - cron: "0 3 * * *"
      timezone: Australia/Brisbane
  workflow_dispatch:
jobs:
  renovate:
    uses: desruc/renovate-presets/.github/workflows/renovate.yml@main
    secrets:
      RENOVATE_GITHUB_TOKEN: ${{ secrets.RENOVATE_GITHUB_TOKEN }}
```

`.github/workflows/renovate-approve.yml`:

```yaml
name: Renovate approve
on:
  pull_request:
    types: [labeled, synchronize]
  push:
    branches: [main]
jobs:
  renovate-approve:
    uses: desruc/renovate-presets/.github/workflows/renovate-approve.yml@main
    permissions:
      contents: read
      pull-requests: write
    secrets:
      RENOVATE_GITHUB_TOKEN: ${{ secrets.RENOVATE_GITHUB_TOKEN }}
```

`github-action.json5` runs a post-upgrade build, so repos using it must also allow that command in their own
`renovate-config.js` (`allowedCommands` and `allowShellExecutorForPostUpgradeCommands` can't be set from a preset).

## How auto-merge works

Auto-merge is opt-in per package. Renovate owns the merge; `renovate-approve` only supplies what Renovate can't do
for itself.

1. A package rule sets `automerge: true` and `addLabels: ['automerge']`.
2. Renovate opens the PR with the `automerge` label and an "Automerge: Enabled" note in the body, and turns on
   GitHub's auto-merge (`platformAutomerge`).
3. The label triggers `renovate-approve`. It approves the PR only if all of these hold:
   - Renovate's account opened or pushed it, and the branch starts with `renovate/`.
   - It has the `automerge` label.
   - The body says "Automerge: Enabled". Labels are combined across a group, but Renovate only enables
     auto-merge when every package in the group allows it, so this check catches mixed groups.
   - `RENOVATE_AUTOMERGE_PAUSED` isn't set.
4. Main requires branches to be up to date, and GitHub's auto-merge won't update a branch itself. On each push to
   main, `renovate-approve` updates the oldest approved automerge PR that's behind, skipping any whose required
   checks failed.
5. Once the PR is approved and its checks pass, it's squash-merged.

A PR Renovate hasn't opted in to never gets the label or the approval, so it waits for a human.

## Adding an auto-merge rule

- Only auto-merge what can't reach shipped code: test tooling, dev dependencies, CI actions. A green build should be
  all the confidence you need.
- Always pair `automerge: true` with `addLabels: ['automerge']`. The approve workflow needs both the label and
  Renovate's "Automerge: Enabled" note in the PR body.
- Watch out for grouping. `config:recommended` groups monorepo packages, and labels are combined across a group. Don't
  list a package that ships alongside production packages (e.g. `Microsoft.AspNetCore.Mvc.Testing`).
- Pin and digest updates skip `minimumReleaseAge`, so keep them on manual review.

## Pausing auto-merge

During a supply-chain incident:

1. Set the organisation variable `RENOVATE_AUTOMERGE_PAUSED` to `true`. This stops new approvals and branch updates.
2. Set `automerge: false` in `default.json5`. PRs that are already approved will still merge otherwise.

## Making changes

- `validate-renovate-config.yml` runs `renovate-config-validator --strict` on every PR that touches this config.
- The Renovate version is pinned in `renovate.yml` and `validate-renovate-config.yml`. Renovate raises one PR to
  update both.
- This repo runs Renovate on itself (config in the root `renovate-config.js`) to keep its pinned actions current. Nothing here auto-merges.
