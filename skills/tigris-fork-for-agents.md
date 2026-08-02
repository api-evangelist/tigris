---
name: Fork a dataset per agent with copy-on-write buckets
description: Give each agent an isolated, instant copy-on-write clone of a shared dataset using Tigris bucket snapshots and forks.
api: Tigris storage-go / agent-kit (Tigris extensions over S3)
endpoint: https://t3.storage.dev
operations: [CreateSnapshotEnabledBucket, CreateBucketSnapshot, CreateBucketFork, HeadBucketForkOrSnapshot, ListBucketSnapshots, DeleteBucket]
source: https://www.tigrisdata.com/docs/buckets/forks/
---

# Fork a dataset per agent with copy-on-write buckets

Tigris bucket **forks** give each agent its own copy-on-write clone of a shared
dataset — instant at any size, zero duplication, isolated writes. This mirrors
the `@tigrisdata/agent-kit` `createForks` / `teardownForks` helpers and the
`storage-go` fork API.

## Preconditions

Forking requires the source bucket to have snapshots enabled.

## Steps

1. **CreateSnapshotEnabledBucket** — create (or ensure) the shared dataset
   bucket with snapshots enabled.
2. **CreateBucketSnapshot** — take a point-in-time snapshot of the dataset;
   capture the returned `SnapshotVersion`.
3. **CreateBucketFork** — for each agent, fork the bucket into its own isolated
   bucket (e.g. `experiment-run-42-1`). Each fork is a separate bucket with
   copy-on-write storage; writes in a fork never touch the source. Scope a
   per-fork access key to a role (e.g. `Editor`) so each agent only sees its
   fork.
4. **HeadBucketForkOrSnapshot** — inspect fork metadata (`SourceBucket`,
   `SourceBucketSnapshot`, `IsForkParent`) when reconciling.
5. **ListBucketSnapshots** — enumerate available snapshots for a bucket.
6. **DeleteBucket** (teardown) — when the run finishes, revoke the per-fork
   credentials and delete the fork buckets.

## Rules

- Never write to the source dataset from an agent — always operate inside the
  agent's fork.
- Forks are billed as copy-on-write: only diverged objects consume storage.
- Combine with `tigris-upload-and-serve` for the object-level operations inside
  each fork.
