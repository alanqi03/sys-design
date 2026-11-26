---
title: "S3: Handling Large Blobs"
description: A plain-language guide to presigned URLs, direct S3 uploads and downloads, multipart uploads, and byte-range downloads.
---

# S3: Handling Large Blobs

A **blob** is a large file: an image, video, PDF, or backup. Store the bytes in
object storage like Amazon S3, and keep the searchable details (owner, status,
S3 key, size) in your database.

**Running example:** a video app where users upload 2 GB videos.

Don't stream that 2 GB through your app server. Your server should decide **who
may upload or download what**; S3 should move and store the bytes.

| Need | Approach |
| --- | --- |
| Small or medium file | One `PUT` to upload, one `GET` to download |
| Let a client reach a private bucket directly | A short-lived **presigned URL** |
| Large or flaky upload (over ~100 MB) | **Multipart upload**: send parts in parallel, retry only failed parts |
| Large, resumable, or parallel download | HTTP **`Range`** requests |

A single `PUT` can be at most **5 GB**, so larger files must use multipart.

## Presigned URLs

The bucket is private, and the user's browser has no AWS credentials. A
**presigned URL** is a temporary permission slip: your server signs "this one
operation, on this one file, until this time" using its own AWS identity, and
hands the URL to the client.

```text
https://videos.s3.us-east-1.amazonaws.com/user-42/uploads/7f3a.mp4
  ?X-Amz-Algorithm=AWS4-HMAC-SHA256
  &X-Amz-Credential=AKIA.../20261124/us-east-1/s3/aws4_request
  &X-Amz-Date=20261124T101500Z
  &X-Amz-Expires=900                   ← valid for 15 minutes
  &X-Amz-Signature=3b9c...             ← proves your server approved this
```

- S3 checks the signature itself, with no call back to your server.
- Changing anything signed (the file path, the expiry, `PUT` vs. `GET`) breaks
  the signature.
- The URL contains **no secret key**, but anyone holding it can use it until it
  expires. Keep expiries short and don't log full URLs.

```{mermaid}
sequenceDiagram
  participant C as Client
  participant A as App server
  participant S as S3
  C->>A: I want to upload cat.mp4 (2 GB)
  A->>A: Check user and quota, then choose key user-42/uploads/7f3a.mp4
  A-->>C: Presigned PUT URL (15 min)
  C->>S: PUT bytes directly
  S-->>C: 200 OK
```

## Simple upload

1. The client sends the file name, type, and size (not the bytes) to your
   server.
2. Your server checks permissions, picks the S3 key itself (never trust a
   client-chosen path), saves a `PENDING` row, and returns a presigned `PUT`
   URL.
3. The client uploads directly to S3.
4. Your server learns the upload finished (next section) and marks it `READY`.

If a single `PUT` fails at 95%, the whole file must be resent. That's fine for
small files, but painful for 2 GB.

## Confirming the upload with S3 events

Don't trust the client saying "done." It could crash right after uploading, or
lie. Instead, S3 emits an **Object Created** event when the file lands:

```{mermaid}
sequenceDiagram
  participant S as S3
  participant Q as SQS
  participant W as Worker
  participant D as Database
  S->>Q: Object Created (key, size, ETag)
  Q->>W: deliver event
  W->>D: find PENDING row for this key → mark UPLOADED
  W->>Q: ack
```

S3 events can arrive **twice** or **out of order**, so make the update
idempotent (setting `status = 'UPLOADED'` twice is harmless). For user uploads,
scan for malware and check the real file type before moving from `UPLOADED` to
`READY`.

## Simple download

1. The client asks your server for video 123.
2. Your server checks the user may watch it.
3. Your server returns a presigned `GET` URL, or a redirect to one.
4. The client downloads straight from S3, or through CloudFront as a CDN.

## Multipart upload

S3 accepts a big file as numbered **parts**, then stitches them together.

```{mermaid}
sequenceDiagram
  participant C as Client
  participant A as App server
  participant S as S3
  C->>A: Start upload (2 GB)
  A->>S: CreateMultipartUpload
  S-->>A: uploadId
  A-->>C: uploadId + presigned URLs for parts 1–200
  par in parallel
    C->>S: PUT part 1 → ETag 1
  and
    C->>S: PUT part 2 → ETag 2
  and
    C->>S: PUT part 3 → ETag 3
  end
  C->>A: Done: [(1, ETag 1), (2, ETag 2), ...]
  A->>S: CompleteMultipartUpload
```

**Example:** 2 GB in 10 MB parts is 200 parts. The client uploads 4 at a time.
If part 117 fails, it retries only part 117.

- Parts must be at least **5 MB** (except the last), with up to **10,000**
  parts.
- S3 returns an **ETag** per part; the client sends the list back to finish.
- To resume after a crash, save the `uploadId` and finished parts, get fresh
  URLs, and send only the missing parts.
- Unfinished uploads still cost money. Add a lifecycle rule to delete them after,
  say, 7 days.

## Byte-range downloads

There's no special "multipart download." The client just asks for byte ranges:

```http
GET /user-42/videos/7f3a.mp4?...signature...
Range: bytes=0-10485759          ← first 10 MB
```

S3 replies `206 Partial Content` with just those bytes. The client can fetch
several ranges in parallel, write each at its offset, retry only failed ranges,
and verify the checksum at the end. Video players use the same trick to jump to
the middle of a video without downloading the start.

To make sure every range comes from the same version of the file, pin the URL to
a version ID or send `If-Match: <ETag>`.

## Checklist

- Keep buckets private; the signing role can only reach the paths it needs.
- Generate S3 keys on the server.
- Use short expiries, and lock URLs to the method and content type.
- Configure bucket CORS for your web origins.
- Track ownership and status in the database; knowing an S3 key isn't
  permission.
- Scan untrusted uploads before serving them.
- Add lifecycle rules for abandoned multipart uploads.

## Further reading

- [S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [S3 multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [S3 Event Notification types and destinations](https://docs.aws.amazon.com/AmazonS3/latest/userguide/notification-how-to-event-types-and-destinations.html)
- [S3 performance guidance for byte-range fetches](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance-guidelines.html)
