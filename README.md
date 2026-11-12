# PR automation fixture

Test fixture for pull request automation.

## Installation

Nothing to install: the workflows run in this repository's GitHub Actions.

## Usage

The merge gate has three parts:

- `.github/workflows/review-verdict.yml` is reusable with `workflow_call` and
  can be run directly with `workflow_dispatch`. It posts the `review/verdict`
  commit status to the supplied head SHA: `MERGE` is `success`, and `CHANGES`
  is `failure`. The status description names the reviewer model.
- `.github/rulesets/main.json` declares the REST ruleset for the default
  branch. It requires `review/verdict` to be up to date, requires pull
  requests, and blocks force pushes and deletion. This fixture has no other CI
  check to require.
- `.github/workflows/keep-mergeable.yml` runs on pushes to `main` or by
  dispatch. It calls GitHub's update-branch API for open PRs whose
  `mergeStateStatus` is `BEHIND`, and adds or removes the `conflict` label
  according to whether the state is `DIRTY`.

### Prove the status follows a head SHA on PR #2

Run this after the workflows are on `main`. It posts a status to PR #2's
current head, then pushes an empty commit to move the PR to a new head and
confirms that the new SHA has no `review/verdict` status.

```sh
set -eu
REPO=JacobPEvans-ai/pr-automation-fixture
JQ_LAST='[.statuses[] | select(.context == "review/verdict")] | last'
JQ_COUNT='[.statuses[] | select(.context == "review/verdict")] | length'
gh repo clone "$REPO" merge-gate-status-proof
cd merge-gate-status-proof
PR2_SHA=$(gh api "repos/$REPO/pulls/2" --jq .head.sha)
gh workflow run review-verdict.yml --repo "$REPO" --ref main \
  -f pr_number=2 -f head_sha="$PR2_SHA" -f verdict=MERGE \
  -f reviewer_model=fixture-proof
RUN=$(gh run list --repo "$REPO" --workflow review-verdict.yml \
  --branch main --event workflow_dispatch --limit 1 \
  --json databaseId --jq '.[0].databaseId')
gh run watch "$RUN"
gh api "repos/$REPO/commits/$PR2_SHA/status" --jq "$JQ_LAST"
gh pr checkout 2 --repo "$REPO"
git commit --allow-empty -m "test: move PR 2 head for status proof"
git push
NEW_PR2_SHA=$(gh api "repos/$REPO/pulls/2" --jq .head.sha)
test "$PR2_SHA" != "$NEW_PR2_SHA"
gh api "repos/$REPO/commits/$NEW_PR2_SHA/status" --jq "$JQ_COUNT"
cd ..
```

The final command prints `0`, proving the status remains attached to the SHA
it was posted against.

### Prove conflict labeling and behind updates on PRs #2 and #3

Set PR #2 to `BEHIND` and PR #3 to `DIRTY`, then dispatch the workflow. The
run updates PR #2 by merging the base branch into its head and applies the
`conflict` label to PR #3.

```sh
set -eu
REPO=JacobPEvans-ai/pr-automation-fixture
state() {
  gh pr view "$1" --repo "$REPO" --json mergeStateStatus \
    --jq .mergeStateStatus
}
has_conflict() {
  gh pr view 3 --repo "$REPO" --json labels \
    --jq '[.labels[].name] | index("conflict") != null'
}
gh repo clone "$REPO" merge-gate-maintenance-proof
cd merge-gate-maintenance-proof
test "$(state 2)" = BEHIND
test "$(state 3)" = DIRTY
test "$(has_conflict)" = false
PR2_BEFORE=$(gh api "repos/$REPO/pulls/2" --jq .head.sha)
gh workflow run keep-mergeable.yml --repo "$REPO" --ref main
RUN=$(gh run list --repo "$REPO" --workflow keep-mergeable.yml \
  --branch main --event workflow_dispatch --limit 1 \
  --json databaseId --jq '.[0].databaseId')
gh run watch "$RUN"
PR2_AFTER=$PR2_BEFORE
attempt=0
while [ "$PR2_AFTER" = "$PR2_BEFORE" ] && [ "$attempt" -lt 30 ]; do
  sleep 2
  PR2_AFTER=$(gh api "repos/$REPO/pulls/2" --jq .head.sha)
  attempt=$((attempt + 1))
done
test "$PR2_BEFORE" != "$PR2_AFTER"
gh pr view 2 --repo "$REPO" --json mergeStateStatus,headRefOid
has_conflict
cd ..
```

The head SHA change on PR #2 confirms the update-branch request completed; the
final command prints `true` when PR #3 has the `conflict` label.
