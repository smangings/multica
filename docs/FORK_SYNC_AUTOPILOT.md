# Wiring GitHub → Multica autopilot (HOME-62)

Autopilot is live. One human step remains on GitHub because this runtime has no
GitHub credentials.

## What exists now

- Autopilot **"Fix fork upstream sync failures"** (`504e910e-655b-4e30-ae7d-6c730b5ea067`),
  active, mode `create_issue`, assigned to **Mac Primary**, issue title template
  `Fork sync failure fix {{date}}`.
- Webhook trigger `github-sync-failure` (`66aca445-3ce8-4f00-81d2-5554f540f1d5`).
- Agent instructions cover: read failed logs, classify conflict vs token vs transient,
  minimal conflict resolution (no OIDC changes beyond resolution), fix branch +
  PR (never force-push except `--force-with-lease`), re-run workflow, report.

## Step you must do in GitHub (needs repo access)

1. Get the webhook URL (it is secret-bearing — do not paste it anywhere shared):
   ```
   multica autopilot get 504e910e-655b-4e30-ae7d-6c730b5ea067 --show-secrets --output json
   ```
   Copy `webhook_url`. If it ever leaks, rotate with
   `multica autopilot trigger-rotate-url <autopilot-id> 66aca445-3ce8-4f00-81d2-5554f540f1d5 --yes`.

2. Add it as a fork secret — repo **benjsnellings/multica** → Settings → Secrets
   and variables → Actions → New repository secret:
   - Name: `FORK_SYNC_MULTICA_WEBHOOK_URL`
   - Value: the URL from step 1.

3. Commit the provided workflow file to the fork as
   `.github/workflows/sync-failure-autopilot.yml`. It fires on
   `workflow_run` completion of "Sync fork with upstream" (and the rebase/tag
   workflow), and only acts when `conclusion == 'failure'`. Adjust the
   workflow name list if yours differ.

## Rotation documentation

The webhook token lives inside the URL path. To rotate:

```
multica autopilot trigger-rotate-url 504e910e-655b-4e30-ae7d-6c730b5ea067 66aca445-3ce8-4f00-81d2-5554f540f1d5 --yes
```

Then update the `FORK_SYNC_MULTICA_WEBHOOK_URL` secret with the new URL. The old
URL stops working immediately.

## Failure drill (acceptance test)

1. Trigger a deliberate failure: push a branch off `main`, add a commit that
   conflicts with `upstream/main` touching the same file, or temporarily break
   the sync step; run the sync workflow via `workflow_dispatch`.
2. On failure, `sync-failure-autopilot.yml` POSTs to the webhook with an
   `Idempotency-Key` keyed on the run id (GitHub retries reuse it safely).
3. Verify: `multica autopilot runs 504e910e-655b-4e30-ae7d-6c730b5ea067`,
   then check the created issue ("Fork sync failure fix <date>").
4. Webhook delivery statuses are visible via `multica autopilot get`; `queued`
   = worker pending, `failed` carries the worker error.

## Guardrails encoded

- Never force-push; `--force-with-lease` only with explicit justification.
- Sync-only fixes must not merge unrelated OIDC feature work.
- Token/permission failures are reported for a human, never guessed.
- Mac Primary's standing rule applies: no PR merges by agents.
