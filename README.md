# Merge-ready pull request reminder

[`merge-ready-reminder.yml`](merge-ready-reminder.yml) is a reference GitHub
Actions workflow for pull requests that have been idle while ready to merge,
blocked by failed checks, behind their base branch, or waiting for required
review. It runs once per day and uses pull request comments to notify the
people who can move each pull request forward through GitHub's existing
notification inbox, email, and mobile channels.

## Install it

1. Copy `merge-ready-reminder.yml` to
   `.github/workflows/merge-ready-reminder.yml` in the repository to monitor.
2. Merge the workflow into the repository's default branch. Scheduled workflows
   only run from the default branch.
3. No repository-level workflow-permission change is normally required. The
   **Read repository contents and packages permissions** setting controls the
   default permissions for workflows that do not declare their own. This
   workflow explicitly requests `contents: read`, `checks: read`,
   `statuses: read`, `issues: write`, and `pull-requests: write`, so its
   `GITHUB_TOKEN` receives those permissions for the run. Organization or
   enterprise policy can still restrict the permissions a workflow may request.
4. Open **Actions > Remind about merge-ready pull requests** and run
   **Run workflow** once. This tests the workflow without waiting for its
   schedule. The run verifies comment-write access only if it finds a pull
   request eligible for an initial or repeat reminder.

No personal access token, GitHub App, webhook, Teams channel, or external
service is required.

## How it classifies pull requests

For every open pull request, the workflow checks GitHub's current state in this
order:

1. Drafts, pull requests with auto-merge enabled, merge conflicts, and pull
   requests whose mergeability is still being calculated are non-actionable.
   They remain visible in the run summary but do not receive reminders.
2. `CHECKS_FAILED_IDLE` means at least one current check run or commit status
   has a terminal failure. The workflow examines only the latest result for
   each check identity or status context. It is implementation-agnostic: an
   AB# policy check is detected from its reported failure just like any other
   check, without hard-coding a workflow file, check name, or GitHub App.
3. `BASE_BRANCH_OUTDATED_IDLE` means checks are not currently failing, but the
   pull request head is one or more commits behind its current base branch.
4. `REVIEWERS_MISSING_IDLE` means there are no current failed checks, the branch
   is current, and GitHub reports that review is required. Visible requested
   users or teams are included in the reminder.
5. `MERGE_READY_IDLE` means there are no current failed checks, the branch is
   current, and no reviewers are requested, while GitHub reports
   `mergeable: true` and
   `mergeable_state: clean`. The `clean` state means required checks and
   reviews are satisfied and the branch has no conflicts.

Failed checks take precedence over an outdated base, which takes precedence
over requested reviewers, which takes precedence over merge-ready. This keeps
the reminder text unambiguous even when a pull request has more than one
condition.

The failed-check states are `failure`, `error`, `timed_out`, `cancelled`,
`action_required`, `startup_failure`, and `stale`. Pending or in-progress
checks do not qualify as failures.

Each actionable category uses `MIN_READY_HOURS`:

- Failed-check age starts at the newest current failure's update or completion
  time. A rerun resets that check's clock, and a successful rerun removes it
  from the current failures.
- Outdated-base age starts at the latest of the first base-only commit, pull
  request creation, and current head commit. Opening or updating a pull request
  therefore receives the full grace period even when its branch started from
  an older base.
- Requested-review age starts at the newest matching `review_requested`
  timeline event for the users and teams that are still requested. If GraphQL
  reports that review is required but identities are redacted, the workflow
  uses the newest available review-request event. If no request event is
  available, it reports that it cannot calculate the age and does not send a
  premature reminder.
- Ready time is approximated by taking the newest timestamp among the head
  commit, current commit statuses, current completed check runs, and approved
  reviews. GitHub does not provide a single "became merge-ready at" timestamp,
  so this is a conservative approximation.

Failed-check, outdated-base, and merge-ready reminders mention the pull request
author. Missing-reviewer reminders mention the author and the users or teams
whose reviews are currently requested.

Each comment contains a hidden marker with the category and head commit SHA.
The workflow uses matching marker comments as notification history. It sends
another reminder for an unchanged category and revision after
`REPEAT_REMINDER_HOURS`; until then, the result is `RECENTLY_NOTIFIED`. A
different blocker category or new head SHA has separate history and can receive
its first reminder after `MIN_READY_HOURS`. Existing SHA-only merge-ready
markers are also recognized during upgrades. Closing or merging the pull
request stops reminders.

The implementation combines GitHub REST and GraphQL API calls. REST retrieves
mergeability, commits, checks, statuses, reviews, comparisons, timeline events,
and comments. GraphQL supplies the authoritative `reviewDecision` and requested
reviewers because REST can omit requested teams when the workflow token cannot
read organization team details.

## Reading workflow output

Open a run under **Actions > Remind about merge-ready pull requests**. The
workflow posts a notice when the results table has been written; select the
run's **Summary** page to view it. The raw job log contains a collapsed group
for every open pull request with its URL, author, head SHA, draft state,
mergeability, mergeable state, auto-merge state, blocker, and blocker age when
applicable.

The run's **Summary** page contains one row per pull request. The **Category**
column distinguishes why it is stuck:

- `CHECKS_FAILED_IDLE`: current failed check names are shown in **Details**.
- `BASE_BRANCH_OUTDATED_IDLE`: the number of commits behind the base branch is
  shown in **Details**.
- `REVIEWERS_MISSING_IDLE`: currently requested users and teams are shown in
  **Details** when visible; otherwise the row states that required review is
  outstanding.
- `MERGE_READY_IDLE`: no observed check is currently failing, no reviewers are
  currently requested, and GitHub permits the merge.
- `NON_ACTIONABLE` or `NOT_READY`: the pull request is not eligible for these
  reminders; **Details** shows why.

The **Outcome** column reports `WAITING`, `RECENTLY_NOTIFIED`, `NOTIFIED`,
`RENOTIFIED`, or `SKIPPED`. `NOTIFIED` is the first reminder for that category
and head SHA; `RENOTIFIED` is a repeat after the configured cooldown. This
deliberately separates the reason a pull request is stuck from what the
reminder run did about it.

Near the end of the log, the workflow also reports the total number examined
and notified.

The workflow allows only one reminder run at a time. If a scheduled or manual
run starts while another run is active, one new run can wait without canceling
the active run. If another run enters the same concurrency group while one is
already waiting, GitHub cancels the older pending run and keeps the newest one.

## Settings to customize

### Daily schedule

Change this cron expression in the workflow:

```yaml
on:
  schedule:
    - cron: "17 14 * * *"
```

GitHub cron schedules use UTC. The sample runs every day at 14:17 UTC. Scheduled
runs can be delayed during periods of high GitHub Actions load.

Common alternatives:

```yaml
# Weekdays at 14:17 UTC
- cron: "17 14 * * 1-5"

# Every day at 09:23 UTC
- cron: "23 9 * * *"
```

### Minimum idle time

Change the job environment value:

```yaml
env:
  MIN_READY_HOURS: "24"
```

The same threshold applies to failed checks, outdated base branches, requested
reviews, and merge-ready pull requests. Because the workflow runs daily, a
24-hour setting normally sends a reminder between 24 and 48 hours after the
relevant state begins. Use `48` for a two-day grace period. Use `0` temporarily
when testing with **Run workflow**, then restore the intended value.

### Repeat reminder time

Change the repeat cooldown independently from the initial idle time:

```yaml
env:
  REPEAT_REMINDER_HOURS: "24"
```

The default permits one repeat reminder per day for an unchanged blocker and
head SHA. The workflow uses the newest matching reminder comment's creation
time, so each repeat starts a new cooldown. Because the workflow runs daily, a
24-hour value normally repeats between 24 and 48 hours after the previous
comment. Use a larger value such as `168` for weekly reminders, or `0` to
permit a repeat on every workflow run. There is no built-in maximum number of
repeats; reminders continue at the configured interval until the PR changes
state, changes head SHA, closes, or merges.

### Reminder text and recipient

Edit the relevant `body` array passed to
`github.rest.issues.createComment`. A GitHub mention creates an inbox
notification subject to that user's personal notification settings. The
failed-check, outdated-base, and merge-ready bodies mention the author. The
requested-review body also mentions currently requested users and teams when
GitHub makes their identities visible to the workflow token.

To notify a GitHub team too, add a team mention such as `@my-org/reviewers` to
the comment. The team must be visible to the repository, and its notification
settings determine who receives the mention.

### Include pull requests with auto-merge enabled

The sample skips those pull requests because GitHub will merge them when their
requirements allow it. To include them, remove the auto-merge branch from
the nested `nonActionableReason` expression and connect the preceding draft
case directly to the following merge-conflict case.

### Restrict labels or branches

The REST list response includes `pullRequest.labels` and `pullRequest.base.ref`.
Add a condition at the start of the loop, before
`getPullRequestWithMergeability`, to limit processing. For example:

```javascript
if (pullRequest.base.ref !== "main") {
  continue;
}

if (pullRequest.labels.some((label) => label.name === "no-merge-reminder")) {
  continue;
}
```

## Operational notes

- The workflow comments as `github-actions[bot]`; the author mention is what
  routes the reminder through GitHub notifications.
- GitHub can disable scheduled workflows after 60 days without repository
  activity in public repositories. Re-enable the workflow from the Actions tab.
- The repository's default **Workflow permissions** setting can remain
  read-only because this workflow has an explicit `permissions` block.
  Organization or enterprise rules can still prevent `GITHUB_TOKEN` from
  writing. A run will show a `Resource not accessible by integration` error in
  that case; investigate the governing Actions policy rather than adding a
  personal token where possible.
- `mergeable` can temporarily be `null`, and `mergeable_state` can be `unknown`,
  while GitHub computes mergeability. The workflow retries the pull request
  five times at two-second intervals. If the state is still unresolved, that
  run reports `NON_ACTIONABLE` and checks it again on the next schedule.
- Requested-review reminders use GraphQL's review decision even when requested
  team identities are not visible to the workflow token. In that case the
  author is notified without guessing or mentioning a reviewer. If no matching
  review-request timeline event is available, the workflow reports `WAITING`
  rather than sending a prematurely timed reminder.
- The readiness timestamp is an approximation because GitHub does not expose a
  single "became merge-ready at" timestamp. It is the latest head-commit,
  current-status, current completed-check-run, or approved-review timestamp,
  including events that might not be required for merging. The workflow
  deliberately uses current GitHub merge state as the authority and avoids
  using the pull request's general `updated_at`, which changes for unrelated
  activity.
