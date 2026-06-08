# AWS S3 Resumable Uploads (Multipart Upload)

## Overview

AWS S3 supports resumable uploads through **Multipart Upload**.

Instead of uploading a large file in a single request, the file is split into multiple parts (chunks). Each part is uploaded independently, allowing failed uploads to resume without restarting the entire file upload.

---

## How Multipart Upload Works

### 1. Initiate Multipart Upload

The client starts a multipart upload request.

```text
CreateMultipartUpload
```

S3 returns an:

```text
UploadId
```

This `UploadId` uniquely identifies the multipart upload session.

---

### 2. Split the File into Parts

Break the file into chunks.

Example:

```text
Part 1 -> 5 MB
Part 2 -> 5 MB
Part 3 -> 5 MB
Part 4 -> 5 MB
```

Each part is assigned a unique part number.

---

### 3. Upload Parts Independently

Upload each chunk using:

* Bucket
* Object Key
* UploadId
* Part Number

```text
UploadPart
```

For every successful upload, S3 returns an:

```text
ETag
```

Example:

```text
Part 1 -> ETag abc123
Part 2 -> ETag def456
Part 3 -> ETag ghi789
```

Store these ETags because they are required when completing the upload.

---

### 4. Track Upload Progress

After each successful `UploadPart` call, store:

```text
UploadId
Part Number
ETag
```

This can be stored in:

* Memory
* Local Storage
* Database
* Redis
* Backend service

Example:

```json
{
  "uploadId": "xyz123",
  "uploadedParts": [
    { "partNumber": 1, "etag": "abc123" },
    { "partNumber": 2, "etag": "def456" }
  ]
}
```

---

### 5. Resume After Failure

Suppose:

```text
Part 1 -> Success
Part 2 -> Success
Part 3 -> Network Failure
Application Crash
```

When the application restarts, there are two options:

#### Option A: Use Stored State (Recommended)

If the application has already saved uploaded part information:

```text
Part 1 uploaded
Part 2 uploaded
Part 3 pending
```

Resume directly from Part 3.

#### Option B: Query S3

If local state was lost, call:

```text
ListParts
```

using the existing `UploadId`.

Example response:

```text
Part 1
Part 2
Part 4
```

This indicates:

```text
Part 1 -> Uploaded
Part 2 -> Uploaded
Part 3 -> Missing
Part 4 -> Uploaded
```

Only upload the missing parts.

---

## Do We Need to Poll S3 Continuously?

### No

In most implementations, continuous polling is unnecessary.

Normal flow:

```text
Upload Part
      |
      v
Receive Success Response
      |
      v
Store ETag and Part Number
```

Because every successful upload returns a response, the client already knows the current upload status.

---

## When Should ListParts Be Used?

Use `ListParts` only when:

* Application crashes
* Browser refreshes
* Client restarts
* Upload state is lost
* Multiple upload workers are involved
* Upload ownership changes between systems

`ListParts` acts as a recovery mechanism rather than a real-time progress API.

---

## Completing the Upload

After all parts are uploaded:

```text
CompleteMultipartUpload
```

Send S3:

```json
[
  { "PartNumber": 1, "ETag": "abc123" },
  { "PartNumber": 2, "ETag": "def456" },
  { "PartNumber": 3, "ETag": "ghi789" }
]
```

S3 assembles all parts into the final object.

---

## Re-uploading a Part

If a part is uploaded again with the same part number:

```text
Upload Part 2
Upload Part 2 Again
```

S3 replaces the older Part 2 with the newer version.

The latest upload for that part number is used when completing the multipart upload.

---

## Aborting an Upload

If the upload is cancelled:

```text
AbortMultipartUpload
```

This removes all uploaded parts associated with the UploadId and prevents storage costs from accumulating.

---

## Typical Architecture

```text
+-----------+
|  Client   |
+-----------+
      |
      | UploadPart
      v
+-----------+
|   AWS S3  |
+-----------+

Client stores:
- UploadId
- Part Numbers
- ETags
```

Recovery flow:

```text
Client Crash
      |
      v
Application Restart
      |
      v
ListParts(UploadId)
      |
      v
Resume Missing Parts
      |
      v
CompleteMultipartUpload
```

---

## Key Takeaways

* S3 resumable uploads use Multipart Upload.
* Each chunk is uploaded independently.
* Every uploaded part returns an ETag.
* Store UploadId, Part Numbers, and ETags locally.
* No continuous polling is required.
* Use `ListParts` only for recovery or synchronization.
* Complete the upload using all uploaded part numbers and ETags.
* Abort unfinished uploads when they are no longer needed.
