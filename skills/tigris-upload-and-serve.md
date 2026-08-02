---
name: Upload and serve objects on Tigris
description: Create a bucket and put, list, and get objects on Tigris using its S3-compatible API.
api: Tigris Object Storage (S3-compatible)
endpoint: https://t3.storage.dev
operations: [CreateBucket, PutObject, GetObject, ListObjectsV2, DeleteObject]
source: https://www.tigrisdata.com/docs/api/s3/
---

# Upload and serve objects on Tigris

Tigris is S3-compatible, so any AWS S3 SDK works by pointing it at the Tigris
global endpoint. This skill covers the create → upload → list → read → delete
happy path.

## Setup

Configure standard AWS credentials and endpoint (generate an access key in the
Tigris console at https://console.storage.dev):

```
AWS_ENDPOINT_URL=https://t3.storage.dev
AWS_ACCESS_KEY_ID=<your-key>
AWS_SECRET_ACCESS_KEY=<your-secret>
AWS_REGION=auto
```

## Steps

1. **CreateBucket** — create a globally distributed bucket. Bucket names are
   global; a clash returns `BucketAlreadyExists` (409) or, if you own it,
   `BucketAlreadyOwnedByYou`.
2. **PutObject** — upload an object under a key. Objects can be up to 5 TB.
   Custom metadata is supported via `x-amz-meta-*` headers. For a safe,
   idempotent create that will not overwrite an existing object, send
   `If-None-Match: "*"` (a conflicting object yields `PreconditionFailed`, 412).
3. **ListObjectsV2** — list objects with `prefix`, `delimiter`, and `max-keys`.
   Paginate with the `continuation-token` request param and the
   `NextContinuationToken` / `IsTruncated` response fields.
4. **GetObject** — read an object. Use conditional headers (`If-Match`,
   `If-Modified-Since`) to avoid re-transfers; an unmet condition returns
   `NotModified` (304) or `PreconditionFailed` (412).
5. **DeleteObject** — remove an object when done.

## Rules

- Authenticate every request with AWS Signature V4 (handled by the SDK).
- Errors come back as S3 XML (`<Error><Code>…</Code></Error>`); see
  `errors/tigris-error-codes.yml`. Retry `SlowDown` (503) and `InternalError`
  (500) with backoff.
- There are **no egress fees** — reads from any region are free.
