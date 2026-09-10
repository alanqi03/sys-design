---
title: "S3: Handling Large Blobs"
description: An ELI18 guide to presigned URLs, direct S3 uploads and downloads, multipart uploads, and byte-range downloads.
---

# S3: Handling Large Blobs

A **blob** is a large piece of unstructured data: an image, video, PDF, backup,
or model file. Put the bytes in object storage such as Amazon S3 and keep the
searchable business metadata—owner, status, S3 key, size, and checksum—in a
database.

Do not normally send a multi-gigabyte file through the application service. The
service should decide **who may access which object**; S3 should move and store
the bytes.

## ELI18 summary

| Need | Approach | Why |
| --- | --- | --- |
| Small or medium file | One HTTP `PUT` to upload or `GET` to download | Simplest flow; one request transfers the entire object |
| Private direct access | A short-lived presigned `PUT` or `GET` URL | The client talks directly to S3 without receiving AWS credentials |
| Large or unreliable upload | S3 multipart upload | Upload parts in parallel and retry only failed parts |
| Large, resumable, or parallel download | HTTP `Range` requests | Download selected byte ranges and retry only missing ranges |

An easy rule of thumb is to consider multipart upload around **100 MB**. A
single S3 `PUT` supports up to **5 GB**, so larger objects require multipart
upload. Measure on real client networks rather than treating 100 MB as a hard
boundary.

## Presigned URLs, ELI18

A private S3 object normally requires AWS authorization. A **presigned URL** is
a temporary, narrowly scoped permission slip. The application uses its IAM role
and the AWS SDK to sign a specific operation, bucket, object key, and expiration
time; it returns the signed URL, not its AWS credentials.

```{mermaid}
sequenceDiagram
  participant C as Client
  participant A as Application service
  participant S as Amazon S3

  C->>A: Request permission to upload or download
  A->>A: Authenticate user and authorize object
  A->>A: Choose bucket, key, method, and short expiry
  A->>A: AWS SDK signs request with service IAM role
  A-->>C: Presigned URL and required headers
  alt Upload
    C->>S: HTTP PUT bytes using presigned URL
    S-->>C: Upload accepted
  else Download
    C->>S: HTTP GET using presigned URL
    S-->>C: Object bytes
  end
```

S3 checks that the signature is valid, has not expired, matches the request,
and came from an identity allowed to perform that operation. Presigning happens
inside the application; generating the URL does not send the blob through the
application service.

Treat a presigned URL like a temporary bearer credential: anyone who obtains it
may be able to use it until it expires. Use short expirations, HTTPS,
least-privilege IAM permissions, and logs that do not record full URLs.

## Simple upload

For a single-request upload:

1. The client sends the filename, content type, and expected size to the
   application—not the file bytes.
2. The application authenticates the user, checks quota and file policy, and
   generates an opaque object key such as `tenant-42/uploads/uuid`.
3. The application records a `PENDING` upload and returns a presigned `PUT` URL
   plus any headers that were included in the signature.
4. The client sends one HTTP `PUT` directly to S3. The HTTP method and signed
   headers must match exactly.
5. The client reports completion. The application may use `HeadObject` or an S3
   event to verify the key, size, and checksum before marking it `READY`.

```http
PUT https://bucket.s3.region.amazonaws.com/tenant-42/uploads/uuid?...signature...
Content-Type: image/jpeg
x-amz-checksum-sha256: base64-checksum

<file bytes>
```

If the connection fails, the entire single `PUT` must be retried. That is fine
for modest files and stable networks, but wasteful for a large upload that fails
near the end.

## Simple download

For a private single-request download:

1. The client asks the application for an object.
2. The application checks that the caller may read it.
3. The application creates a short-lived presigned `GET` URL.
4. The client downloads directly from S3, or from CloudFront when a CDN belongs
   in front of the bucket.

The application can also return an HTTP redirect to the signed URL. Keep the S3
bucket private; the URL temporarily authorizes only the signed request.

## Chunked upload: S3 multipart upload

S3 calls chunked upload **multipart upload**. The client divides one file into
parts. S3 stores each part under an upload ID and creates the final object only
after receiving a completion request with the ordered list of parts.

```{mermaid}
sequenceDiagram
  participant C as Client
  participant A as Application service
  participant S as Amazon S3

  C->>A: Start upload with name, size, type, and checksum
  A->>A: Authenticate, authorize, and choose object key
  A->>S: CreateMultipartUpload
  S-->>A: Upload ID
  A->>A: Presign UploadPart requests
  A-->>C: Upload ID, part size, and part URLs
  par Upload part 1
    C->>S: PUT part 1
    S-->>C: ETag 1
  and Upload part 2
    C->>S: PUT part 2
    S-->>C: ETag 2
  and Upload part 3
    C->>S: PUT part 3
    S-->>C: ETag 3
  end
  C->>A: Complete with ordered part numbers and ETags
  A->>S: CompleteMultipartUpload
  S-->>A: Final object created
  A-->>C: Upload complete
```

### Step by step

1. **Start.** The application creates a database upload record, calls
   `CreateMultipartUpload`, and stores the returned upload ID with the object
   key and owner.
2. **Plan.** Choose a fixed part size and number the parts from 1 upward. S3
   allows up to 10,000 parts; each part except the last must be at least 5 MiB.
3. **Sign.** The application generates presigned `UploadPart` URLs for the
   expected part numbers, either in a batch or as the client needs them.
4. **Upload.** The client uploads several parts directly to S3 with bounded
   parallelism. A failed part can be retried without resending successful parts.
5. **Remember.** S3 returns an `ETag` for every part. The client keeps each
   `(partNumber, ETag)` pair; an ETag is not always a full-file content hash.
6. **Complete.** The client sends the ordered pairs to the application. The
   application validates them and calls `CompleteMultipartUpload`; S3 joins the
   parts in part-number order.
7. **Verify.** Check the final size and a real object checksum before changing
   the business record from `UPLOADING` to `READY`.
8. **Clean up.** Call `AbortMultipartUpload` when the user cancels. Add an S3
   lifecycle rule to remove forgotten incomplete uploads because stored parts
   continue to cost money until completion or abort.

The client can resume after a crash by saving the upload ID and completed part
list, asking the service for fresh URLs, and sending only the missing parts.
Do not trust an upload ID from the client without checking that it belongs to
that user and object key.

## Chunked download: byte-range requests

S3 does not require a separate “multipart download” session. The client uses a
normal `GetObject` request with the HTTP `Range` header to request selected
bytes. S3 answers a valid partial request with `206 Partial Content` and a
`Content-Range` header.

```{mermaid}
flowchart LR
  start[Get presigned URL, size, and checksum] --> split[Split size into byte ranges]
  split --> r1[GET bytes 0 through 7 MiB]
  split --> r2[GET bytes 8 through 15 MiB]
  split --> r3[GET remaining bytes]
  r1 --> merge[Write each response at its byte offset]
  r2 --> merge
  r3 --> merge
  merge --> verify[Verify final size and checksum]
```

### Step by step

1. **Authorize.** The client asks the application for download access. The
   application checks ownership and returns a presigned `GET` URL, object size,
   object version or ETag, and a stored checksum when available. Pin the URL to
   a version ID when versioning is enabled; otherwise use `If-Match` with the
   ETag so separate ranges cannot come from different object versions.
2. **Split.** The client divides `[0, objectSize - 1]` into non-overlapping byte
   ranges. For example, an 18 MiB object could use 8 MiB, 8 MiB, and 2 MiB.
3. **Fetch.** Send a bounded number of concurrent requests using headers such as
   `Range: bytes=0-8388607`.
4. **Place.** Write each response at its correct byte offset rather than joining
   parts in completion order.
5. **Retry.** If one request fails, request only that range again. Save completed
   ranges to support pause and resume.
6. **Verify.** After all ranges arrive, verify the complete size and checksum.
   If the presigned URL expires during a later retry, request a fresh one.

Too much parallelism can exhaust browser connections, memory, client bandwidth,
or S3 request budget. Start with a small bounded concurrency and measure. For
video or audio playback, range requests also let a player seek without fetching
the entire object first.

## Production checklist

- Keep buckets private and give the signing role permission only for expected
  buckets, key prefixes, and operations.
- Generate object keys on the server; do not let a user overwrite an arbitrary
  key supplied by the client.
- Bind URLs to the intended HTTP method, short expiration, content type, and
  checksum where appropriate.
- Configure bucket CORS for the exact browser origins, methods, and headers that
  need direct access.
- Store ownership and upload state in a database; do not treat possession of an
  object key as authorization.
- Pin multi-request downloads to one object version, or require the same ETag
  on every range request.
- Scan or quarantine untrusted uploads before making them downloadable.
- Set lifecycle policies for abandoned multipart parts and expired objects.
- Monitor failed parts, incomplete uploads, transfer time, checksum failures,
  storage growth, request cost, and internet egress cost.

## Further reading

- [S3 presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [S3 multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [S3 multipart limits](https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html)
- [S3 performance guidance for byte-range fetches](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance-guidelines.html)
- [S3 object downloads](https://docs.aws.amazon.com/AmazonS3/latest/userguide/download-objects.html)
