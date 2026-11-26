# Renovate discussion 48400: branch left at an autoclosed PR's old head when the reopen fails

Reproduced with the self-hosted Renovate CLI `44.263.5` on GitHub.com.

## Current behavior

When Renovate finds a recent `… - autoclosed` PR for a branch, `tryReuseAutoclosedPr` resets the branch to that PR's old head and then tries to reopen the PR.
If the old head is already merged into the base branch, GitHub refuses the reopen (`422: state cannot be changed. These commits are already merged.`).
Renovate then falls back to creating a new PR, which also fails (`422: No commits between main and renovate/actions-checkout-4.x`), because the branch is still at the old head.

The fresh commit that Renovate pushed seconds earlier is lost, no PR is created, and every run repeats it for 7 days (the reopen window):

```
05:50:08 branch_creation renovate/actions-checkout-4.x 0000000 -> bd69aea   (fresh v4.2.2 commit)
05:50:10 force_push      renovate/actions-checkout-4.x bd69aea -> 9e158f8   (reset to the autoclosed PR's head)
05:50:55 push            renovate/actions-checkout-4.x 9e158f8 -> 9fc3c48   (next run, fresh commit again)
05:50:56 force_push      renovate/actions-checkout-4.x 9fc3c48 -> 9e158f8   (reset again)
```

## Expected behavior

When the autoclosed PR cannot be reopened, the branch keeps (or gets back) the freshly pushed commit, so the fallback creates a new PR for v4.2.2.

## Steps

1. `allowedVersions: "<=4.2.1"`: Renovate opens #1 `Update actions/checkout action to v4.2.1` (head `9e158f8`).
2. `allowedVersions: "<=4.2.0"`: Renovate autocloses #1 (`… - autoclosed`) and deletes the branch.
3. Head `9e158f8` gets into `main` another way (here: a merge commit, `1e05d01`), so it is an ancestor of `main`.
4. `allowedVersions: "<=4.2.2"`: Renovate creates the branch with the v4.2.2 commit, resets it to `9e158f8` for the reuse, the reopen fails, the PR creation fails, and the branch stays at `9e158f8`.

Run with:

```sh
docker run --rm -e RENOVATE_TOKEN -e LOG_LEVEL=debug ghcr.io/renovatebot/renovate:44.263.5 \
  --platform=github --onboarding=false --require-config=optional \
  rldyourmnd/renovate-failed-reopen-reproduction
```

## With the proposed fix

Running Renovate from the fix branch on this repository (state after step 4) gives:

```
06:00:20 push       renovate/actions-checkout-4.x 9e158f8 -> 4a2b166   (fresh v4.2.2 commit)
06:00:21 force_push renovate/actions-checkout-4.x 4a2b166 -> 9e158f8   (reset for the reuse attempt)
06:00:24 push       renovate/actions-checkout-4.x 9e158f8 -> 4a2b166   (reopen failed, fresh commit restored)
```

and PR #2 for v4.2.2 is created.

## Link to the Renovate issue or Discussion

https://github.com/renovatebot/renovate/discussions/48400 (fix: https://github.com/renovatebot/renovate/pull/48401)
