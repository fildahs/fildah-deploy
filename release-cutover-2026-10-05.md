# 2026-10-05 API and Care cutover

This release crosses non-compatible Django migrations. Use the `api` service in
`Deploy Production` only after the API and deploy `main` commits have been
reviewed and approved. The API `main` workflow publishes images but dispatches
no deployment while repository variable `API_AUTO_DEPLOY` is absent or not
`on`. The API container no longer migrates at startup.

## Before the window

1. Require green checks on the exact API, RxChat web, Auth, Fildah web and
   HealthScout commits. RxChat CI needs a credential with read access to the
   private API contract. Record all image SHAs and the current production image
   pins before merging.
2. Confirm the production database's applied migrations, free disk space for a
   full database backup and restore, Postgres health, and the current image
   tags. Verify the host `.env` has required credential **presence** without
   printing values. Keep its backup private.
3. Confirm Care's governed corpus and delivery baseline, and record the
   owner's decision to open Care before clinical sign-off. Do not call its
   draft alert levels clinician-approved.
4. Inventory weekly subscriptions (including pending cycle changes), pending
   weekly payments, and provider subscriptions attached to weekly plans. The
   workflow checks once before downtime and again after quiescence. It aborts
   before downtime if any exist at first check, or restarts the old services
   without migrating if an in-flight write appears at the second. Reconcile each affected
   customer and provider subscription before retrying; a migration alone does
   not change a live Paystack renewal.
   Checkout may reactivate an inactive Paystack plan mapping by owner decision;
   use the plan or provider checkout switch when sales must be closed.
5. Schedule a maintenance window. Allow in-flight requests and Celery tasks to
   finish before dispatch. The workflow stops Caddy and all API writers with
   a 180-second grace period, then backs up the quiesced database.

## Order

1. Merge the deploy repository's release PR. Its `main` push updates the VPS
   deploy checkout and Caddy configuration. Verify the deployment workflow.
2. Merge the API PR. Wait for both API images tagged with that exact commit SHA
   to publish. Verify `API_AUTO_DEPLOY` remains off.
3. Dispatch `Deploy Production` manually with `service=api`,
   `image_tag=<API SHA>`, `source_repo=fildahs/fildah-api`, and
   `commit_sha=<API SHA>`. The workflow saves previous Compose/Caddy/`.env`
   files before syncing new deploy config, checks weekly billing, stops Caddy
   and API writers, checks weekly billing again, saves a quiesced SQL backup,
   updates image pins, migrates, starts API/workers, checks migrations, sets up
   Qdrant and checks private API health. Only then does it reopen Caddy.
4. Check public API health, Console, Auth, RxChat, Fildah and HealthScout. Merge
   and deploy frontends only after their new API dependencies are available;
   put RxChat last and verify signed-in chat, Care records, quick check and the
   PWA. Record elapsed downtime and the live build SHAs.
5. Leave `API_AUTO_DEPLOY` off until the owner explicitly chooses to resume
   automatic production dispatch for later API merges.

## Failure and rollback

- A weekly preflight failure leaves the old site running. Resolve the listed
  counts before retrying.
- A backup or migration failure after quiescence leaves Caddy and API writers
  stopped. **Do not start the old API against a partly migrated database.**
  Preserve the backup path printed by the workflow and inspect the failure.
- Before public traffic reopens, a database rollback uses the saved SQL backup
  and previous `.env` image pins. Restore the SQL into a *new* database first,
  verify table counts and Django migrations, then swap database names while all
  API writers stay stopped. Keep the failed database intact for investigation.
  This production database swap needs explicit owner approval. Recreate the
  old containers only after the old schema and image pins match.
- The API-run config backup is the configuration present just before that run;
  the deploy repository has already been merged at this point. Keep the backup
  from the earlier deploy-main run as well for a full pre-release config rollback.
- Once public traffic has reopened, restoring the pre-cutover database would
  discard new customer writes. Resolve an incident with a forward fix or a
  write-aware reconciliation plan instead of blindly restoring that snapshot.

The migrations include transformations that cannot be reversed faithfully by
`migrate` alone: weekly billing states and prices, blank-country defaults,
retired settings, removed legacy plan columns, and Care entitlement changes.
The database backup is the rollback source. The local rehearsal in
`coordination/handoffs/release-execution-2026-10-05.md` records a successful
main-schema migration and a separate restore check.
