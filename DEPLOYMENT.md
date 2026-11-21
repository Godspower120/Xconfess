# Deployment

This document covers how releases are cut, validated, and rolled back for the
application and its database schema. It also defines the **rollback evidence
archive** used to record application and schema rollback drills.

## Release compatibility gate

Before any release is promoted, run:

```sh
npm run check:release-compatibility && npm run secret-scan:artifacts
```

`check:release-compatibility` verifies that the current application build is
compatible with the schema version it will run against. `secret-scan:artifacts`
scans build and release artifacts for leaked credentials. Both must pass.

## Rollback drills

Two kinds of rollback drills are exercised before a release is considered
safe to promote:

- **Application rollback** — redeploy the previous application build against the
  current schema.
- **Schema rollback** — apply the down migration for the release's schema change
  while the previous application build is running.

Each drill produces an evidence record (see below). A release is not eligible
for promotion until both drill records exist for the release under test.

## Rollback evidence archive

The evidence archive lives at the owning release boundary (the release
pipeline), not in application runtime storage. Records are written by the
release tooling and are read-only to application code.

### Record shape

Each archived record contains:

| Field | Description |
| --- | --- |
| `id` | Stable record identifier. |
| `releaseId` | Release the drill was run against. |
| `drillType` | `application` or `schema`. |
| `startedAt` / `completedAt` | Drill timestamps (ISO 8601, UTC). |
| `durationMs` | Wall-clock duration of the drill. |
| `inputs` | **Sanitized** drill inputs only. |
| `decisionPoints` | Ordered list of `{ at, decision, rationale }` entries. |
| `outcome` | `success`, `failed`, or `aborted`. |
| `failureReason` | Present only when `outcome` is not `success`. |
| `retentionUntil` | Timestamp after which the record is eligible for purge. |

### Sanitization

Drill inputs are sanitized before they are persisted. Raw secrets, tokens,
connection strings, and unsanitized request/response payloads are **never**
written to the archive. Only the fields required to reproduce the decision
points are retained; everything else is redacted at capture time. Records that
fail sanitization are rejected rather than stored partially.

### Retention

- Records are retained for the configured retention window (default 180 days)
  and carry an explicit `retentionUntil` timestamp.
- Records past `retentionUntil` are purged by the release tooling; purging is
  idempotent and safe to retry.
- Records for a release that is still under active rollback investigation are
  pinned and excluded from purge until the investigation is closed.

### Access controls

- Writing records requires the release pipeline identity; application runtime
  credentials cannot write to the archive.
- Reading records requires an operator role with release-audit permission.
- Access is logged; reads of records containing `failureReason` are audited.

### Success, validation, and failure semantics

- **Success** — the drill completes and a sanitized record with
  `outcome: "success"` is archived.
- **Validation** — a record missing required fields, carrying an unknown
  `drillType`, or failing sanitization is rejected with a validation error and
  is not persisted.
- **Authorization** — a write or read from an identity without the required
  permission is rejected and audited; no partial record is written.
- **Failure / retry** — a drill that fails is archived with `outcome: "failed"`
  and a `failureReason`. Archive writes are retried with backoff on transient
  storage errors; retries are idempotent on `id`, so a repeated write does not
  create duplicate records. A drill may be re-run, producing a new record that
  supersedes the failed one for promotion purposes.

## Compatibility and existing data

The archive is additive: existing releases without drill records remain valid,
and the compatibility gate treats a missing record as "not yet drilled" rather
than as a failure of previously released data. No application schema change is
required to adopt the archive.
