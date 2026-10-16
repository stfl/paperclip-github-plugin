# fix(sync): degrade instead of block without board access

Branch: `upstream/degrade-without-board-access` (one commit on `main`). Closes #225.

## Summary

**What breaks today.** On a Paperclip deployment that reports `deploymentMode: "authenticated"`, `performSync` refuses to run while any touched company lacks Paperclip board access and returns `missing_board_access`. The settings page, dashboard widget and toolbar lock the sync button for the same reason. This also blocks setups that only have `link_github_item` links and no repository mapping.

**Scenario.** An operator wants one thing from GitHub Sync: wake the assignee when review feedback or red CI lands on a linked PR. To get it, they must hand the plugin worker a board token, which carries all of a board user's rights. Without the token, nothing syncs.

**What changes.** Most of sync already has a plugin SDK path. Every `fetchPaperclipApi` call site is guarded by `if (paperclipApiBaseUrl)`, and #225 lists the fallbacks. For a company without board access, `performSync` now passes no worker Paperclip API URL, so that company syncs through `ctx.issues.*` only:

```diff
 performSync
   probe /api/health
-  authenticated && company lacks board access → fail: missing_board_access
+  authenticated && company lacks board access
+    paperclipApiBaseUrl = undefined for that company   # SDK paths only
+    syncState.warnings += "needs board access: create labels, assign users, tool-access check"
   for each mapping / link
     status, agent assignee, execution policy, description, comments, wakeups
-      REST with board token
+      REST with board token, or ctx.issues.* without it
```

- A GitHub label with no matching Paperclip label stays unmapped instead of failing the issue.
- The UI no longer locks sync. Settings shows board access as **Recommended** and renders the warning.
- An already persisted `missing_board_access` error state still normalizes, and it is shown and cleared as idle.
- Companies that have board access behave as before.
- The README documents a minimal PR follow-through setup: a read-only fine-grained GitHub token and no board token.

## Evidence

- **Before:** `worker blocks sync when the Paperclip deployment is authenticated and board access is missing` passes; sync returns `missing_board_access`.
  **After:** replaced by three tests that drive an authenticated deployment without board access:
  - `worker syncs a mapped repository through the SDK when an authenticated deployment has no board access`: the status transition, comment and wake go through the SDK, and no Paperclip REST call is made besides `/api/health`.
  - `scheduled sync follows a link-only third-party pull request through the SDK when an authenticated deployment has no board access`
  - `worker leaves GitHub labels unmapped without failing imports or status sync when an authenticated deployment has no board access`
- `pnpm typecheck`, `pnpm test`, and `pnpm test:e2e` (real Paperclip host, plugin install and UI mount) pass on the branch.

## Merge Danger

**Door:** two-way

Reverting restores the block. Nothing is migrated, and the `missing_board_access` type stays.

**Blast Radius:** authenticated-without-board-access

Only companies on authenticated deployments without board access change: they sync where they used to be refused. On those companies, label creation, user assignment and the tool-access check stay off, and the sync state warns about it.
