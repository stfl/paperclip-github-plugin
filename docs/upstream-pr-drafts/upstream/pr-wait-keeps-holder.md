# fix(sync): a PR that waits on someone else keeps its ticket holder

Branch: `upstream/pr-wait-keeps-holder` (three commits on `main`, one per case).

## Summary

**What breaks today.** For a directly linked pull request, sync treats three states as work for the PR author although the author has nothing to do. Each time, it takes the ticket from whoever holds it, assigns the PR's `followThroughAssigneeAgentId` and wakes them. The woken agent finds nothing to do and hands the ticket back. If the holder is an agent with a run in progress, Paperclip cancels that run as `issue_reassigned`.

| Case | GitHub state | What sync does today |
| --- | --- | --- |
| Required review | green CI, `MERGEABLE`, `BLOCKED`, `REVIEW_REQUIRED`, no threads | `BLOCKED` is an action-required merge state, so the gate becomes `action_required` and wakes the author |
| Healthy wait held by an agent | `in_review` wait (required review, or `CLEAN` without required reviews) | `shouldClearTransitionAssignee` unassigns the agent gating the ticket (for example a product owner after hand-back), because no execution policy names a reviewer |
| Answered change request | the author replied on every unresolved thread; `CHANGES_REQUESTED` stays until the reviewer re-reviews | the unresolved threads mean `review_actionable`, so the author's hand-back counts as local drift and is reverted on every pass |

**Scenario (live, third case).** A reviewer requests changes with two inline threads. The author pushes a fix, replies on both threads and hands the ticket back to the product owner in `in_review`. On every following sync (every 15 minutes) GitHub Sync moves the ticket back to the author with "unresolved review threads" and wakes them. The author replies that the threads are answered and hands back, and the next pass repeats it. This goes on until the reviewer approves.

**What changes.** A PR that waits on someone else becomes a `waiting` gate. The ticket keeps its holder and nobody is woken:

```diff
 getEffectiveDirectPullRequestGateCondition(pr)
   conflict | ci_failed                              → actionable (unchanged)
-  unresolved threads                                → review_actionable
+  unresolved threads the author has not answered    → review_actionable
+  pending CI behind BLOCKED/UNSTABLE                → waiting
+  BLOCKED, green CI, REVIEW_REQUIRED                → waiting
+  BLOCKED, green CI, threads answered by the author → waiting
   other BLOCKED/BEHIND/DRAFT/...                    → action_required

 in_review transition, gate waiting or ready
-  no execution-policy reviewer → clear agent assignee
+  agent already holds the open issue → keep it, no wake
```

1. `fix(sync): treat a PR waiting only on a required review as a wait`. `BLOCKED` with `REVIEW_REQUIRED`, green CI and no unresolved threads is a wait. A `null` review decision is excluded: GitHub returns it when no review is required, so `BLOCKED` comes from another rule and still wakes the executor. A summary-only "Request changes" review stays `action_required`.
2. `fix(sync): keep the agent holding a ticket while its PR waits`. On a `waiting` or `ready` gate an existing agent assignee stays assigned, the same way a human assignee already did. A `done` issue restored to `in_review` still has its stale agent cleared, and an execution policy's reviewer still wins when Paperclip exposes one.
3. `fix(sync): treat review threads the author answered as a wait on the reviewer`. For PRs with unresolved threads, sync makes one small extra GraphQL query (`author`, `reviews(last: 20)`, and the last comment of each thread). The threads wait on the reviewer when the PR author wrote the last submitted comment on every unresolved thread, and no one else submitted a `CHANGES_REQUESTED` or `APPROVED` review, or a review with a body, after the author's latest reply or review. A reply counts without a push, because an author may answer by disagreeing. While the threads wait, the author's own new comments are the answers, so the trusted-comment rule does not re-arm the work for that author; without this, the replies themselves bounce the ticket back once. Red CI and merge conflicts still wake the owner. A reviewer reply, a new change request, an approval (the author then resolves the threads and finishes) or a review with a body after the author's answer makes the threads actionable again, and the owner is woken once.

## Evidence

All tests are in `tests/plugin.spec.ts`.

- **Before:** `sync.runNow hands a direct PR to its follow-through owner only when the author has work to do`, scenario *green CI waiting only on a required review*: the ticket moves to `engineer-agent` with 1 wake.
  **After:** it stays with `po-agent` with 0 wakes. Summary-only change request, unresolved thread and red CI still go to `engineer-agent` with 1 wake.
- **Before:** `sync.runNow keeps a product owner on an in_review direct PR ticket while the PR waits` (`BLOCKED`/`REVIEW_REQUIRED` and `CLEAN`): `po-agent` is unassigned.
  **After:** `po-agent` stays, 0 wakes over two passes. Two existing tests that expected the unassignment now expect the agent to stay.
- **Before:** `sync.runNow leaves a change request the author answered with the ticket holder until the reviewer asks again`: the answered scenario moves the ticket to `engineer-agent` on pass 1.
  **After:**
  ```text
  answered threads                → in_review, po-agent, 0 wakes   (2 passes)
  answered, then reviewer review  → pass 1 wait, pass 2 engineer-agent, 1 wake
  one thread with reviewer last   → in_progress, engineer-agent, 1 wake
  author woken, replies (comment  → hand-back to po-agent sticks, no second wake
    count rises), hands back         (2 more passes)
  red CI with answered threads    → in_progress, engineer-agent, 1 wake
  ```
- Unit tests next to them: `a PR blocked only on a required review is a waiting gate, not follow-through work` and `a PR whose review threads the author answered waits on the reviewer until the reviewer asks again` cover the classifier (empty `COMMENTED` reviews, approvals, unsubmitted replies, unknown author, resolved threads), gate, status mapping, blocked preservation and the external-state hash.
- `pnpm typecheck` and `pnpm test` pass after each of the three commits.

## Merge Danger

**Door:** two-way

No persisted state changes shape. The external-state hash records the answered flag only when it is set, so hashes already stored for other PRs stay valid. Reverting the commits restores the old routing.

**Blast Radius:** direct-PR routing

This affects tickets with directly linked or issue-linked open PRs in the three wait states. Those tickets stop being reassigned and stop waking the follow-through owner. One case trades a missed wake for no loop: an author who answers a thread with a promise ("will fix") and hands back is not woken again until the reviewer replies. PRs with unresolved threads cost one more small GraphQL query per sync.
