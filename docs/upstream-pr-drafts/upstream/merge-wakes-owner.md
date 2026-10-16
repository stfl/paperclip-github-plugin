# fix(sync): wake a merged PR's follow-through owner instead of closing the issue

Branch: `upstream/merge-wakes-owner` (one commit on `main`; it does not depend on the PR-wait branch).

## Summary

**What breaks today.** When every directly linked PR is terminal, sync maps the Paperclip issue to `done` (any merged) or `cancelled` (all closed). `done` wakes nobody. A follow-through owner is set on the link precisely because that agent carries the PR to the end, but they never get a run after the merge. The post-merge work is left undone: closing out the work-product record, the merge handover to whoever asked for the change, follow-up tickets. For a PR closed without merge, nobody gets a run to decide whether to reopen, replace or drop the work, and the issue is silently `cancelled`.

**Scenario.** An engineer agent opens a PR through `create_pull_request`, which records them as `followThroughAssigneeAgentId`. They hand the ticket to the product owner in `in_review`. A maintainer merges. On the next sync the ticket goes to `done`, still assigned to the product owner, and no one is told. The engineer's handover note is never written, and the product owner learns of the merge only by reading the board.

**What changes.** This applies only when the PR that decides the outcome has a valid follow-through owner. That is a merged PR if any link merged, or else a closed one. Among several, the first in repository and number order that has an owner decides.

```diff
 all direct PR links terminal
-  any merged → done; else → cancelled
+  deciding PR has a valid followThroughAssigneeAgentId
+    first time sync sees this outcome, issue still open
+      → in_progress, assign owner, wake once:
+        "the linked pull request was merged. Finish the post-merge work…"
+        or "…was closed without merge. Decide how the issue continues."
+    outcome unchanged since the hand-off → leave status and assignee alone
+    issue already done/cancelled when first seen → leave it closed
+  no owner → done / cancelled (unchanged)
```

- The remote-action fingerprint marks the hand-off. While the outcome is unchanged, later passes do not touch status or assignee, so the owner's own close sticks.
- An issue that was already closed when sync first sees the outcome is never reopened. This includes tickets closed by the plain `done` mapping before this change.
- Issues linked to GitHub issues of mapped repositories keep the plain `done`/`cancelled` mapping.
- `reuseOrCreatePendingWake` also matches on the wake reason, so a pending status-transition wake is not reused with the wrong text.

**Why change a default users rely on.** The `merged → done` mapping stays for every link without a follow-through owner. The owner field is opt-in: `create_pull_request`, `link_github_item`, the metric endpoint and the hosted link action set it only when asked to, and it means "this agent sees the PR through". Once that agent exists, closing the ticket over their head is the surprising behaviour. The cost to a user who preferred the old mapping is one extra run in which the owner closes the ticket.

**Opt-out, if you want one.** A company-level advanced setting such as "Close the issue when its linked PR merges", on by default for existing installs, would skip the hand-off. It costs a boolean in the advanced settings, one check in the worker, and a toggle on the settings page, about 40 lines with a test. I can add it to this PR if you prefer the old default.

## Evidence

All in `tests/plugin.spec.ts`: `sync.runNow hands a merged or closed direct PR to its follow-through owner instead of closing the issue`.

```text
merged, owner engineer-agent, in_review   → in_progress, engineer-agent, 1 wake /was merged/
merged, no owner, in_review               → done, po-agent, 0 wakes                 (unchanged)
closed, owner engineer-agent, in_review   → in_progress, engineer-agent, 1 wake /closed without merge/
merged, owner, issue already done         → done, po-agent, 0 wakes
second pass, same outcome                 → no further wake
owner sets done, third pass               → stays done
```

- **Before:** the first and third scenarios end in `done` / `cancelled` with 0 wakes.
  **After:** as above.
- `pnpm typecheck` and `pnpm test` pass on the branch.

## Merge Danger

**Door:** two-way

Nothing is migrated. Reverting restores `merged → done` for every link. Issues handed to an owner before a revert stay `in_progress` until someone closes them.

**Blast Radius:** owned-PR closures

This affects only open issues whose deciding terminal PR link has a follow-through owner. Those issues get one `in_progress` run for the owner instead of an immediate `done`/`cancelled`. Dashboards that count `done` on merge see the close one agent run later.
