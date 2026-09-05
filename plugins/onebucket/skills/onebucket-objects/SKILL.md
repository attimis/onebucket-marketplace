---
name: onebucket-objects
description: "Read, inspect, and process objects in OneBucket S3 storage — including large binary files (video, images, archives, datasets) that exceed inline size limits — and query the storage-event stream. Use whenever a task involves a stored object: reading a file from a bucket, analyzing a stored video or image, extracting metadata, searching a large dataset, writing results back to storage, or finding out what changed in a bucket and when. Covers the correct transport for objects of any size."
---

# Working with OneBucket objects

## The rule that governs everything

**Object bytes must never travel through the model's context window.**

A 95 MB object is roughly 25 million tokens. The size limits on `get` exist to protect
context, not bandwidth. For anything large, move the processing to the bytes rather than
the bytes to the model.

## Two connectors, one storage

| Connector | Tools | Purpose |
|---|---|---|
| `onebucket` | `list`, `get`, `put`, `head`, `presign`, `copy`, `move`, `delete`, `locate`, `migrate`, `prefetch`, `create` | Object operations |
| `onebucket-events` | `list_events`, `get_event`, `tail_events` | What happened in storage, and when |

Both are single global endpoints. DNS routes each request to the nearest region, so there
is no region to choose and no region argument on any tool. Do not ask the user which region
to use.

Each connector has its own authorization. If a connector's tools fail with an
authentication or authorization error, it is not enabled for this user: say so once,
continue with what works, and do not retry.

### `locate` and `migrate` are about backends, not regions

`locate` reports which *storage backends* currently hold an object; `migrate` copies it to
another backend. That is the real placement primitive and it operates inside the shared
storage. Neither has anything to do with the network path a request took.

---

## Choose a transport before transferring anything

Call `list` first. It returns every object with its exact size. Use that to pick a path.

```
size ≤ 700 KB  and  text / image / audio   →  get                    (path A)
size ≤ 10 KB   and  PDF / other binary     →  get                    (path A)
anything larger                            →  presign + sandbox pull (path B)
```

When uncertain, use path B. It is never wrong, only occasionally heavier.

---

## Path A — inline read

```
get(bucket, key)
```

Returns the object body directly in the response. Use for config files, notes, small
documents, and images you need to actually look at.

If the response contains metadata (`size`, `etag`, `content_type`) but no body, **stop
and report a connector fault**. Do not fall back to another transport. A silent empty
response is a bug, and working around it hides the bug from the person who can fix it.

---

## Path B — presign, pull to sandbox, process there

Three steps, in order. Do not improvise a different transport at any point.

### 1. Mint a download URL

```
presign(bucket, key)
```

Returns a short-lived GET URL, intended for **the execution sandbox to download via
shell**.

Do not pass this URL to the web fetch tool. Web fetch runs through Anthropic servers and
is limited to URLs found in the user prompt or web search results, so a connector-minted
URL will be refused regardless of any organization setting. This is by design and is not
a misconfiguration to route around.

### 2. Pull it into the sandbox

```bash
curl -sSfL -o /path/in/sandbox/<filename> "<presigned-url>"
```

Bytes travel storage → sandbox directly. Neither the model nor the MCP server sits on
that path, so object size is irrelevant. Confirm with `ls -lh` and compare against the
size `list` reported.

This requires sandbox network egress to the storage endpoint. If curl fails on DNS or
connection refused, that setting is the cause — say so plainly rather than searching for
another route.

### 3. Process locally, return only conclusions

Run tools against the file. Put **only their output** in the response.

```bash
# video → codec, duration, resolution, bitrate
ffprobe -v quiet -print_format json -show_format -show_streams FILE

# image → EXIF, dimensions, camera, GPS
exiftool -json FILE

# archive → manifest without extracting
unzip -l FILE
tar -tzf FILE

# PDF → page count, then text
pdfinfo FILE && pdftotext FILE - | head -100

# large dataset → aggregates only, never the rows
python3 -c "
import pandas as pd
df = pd.read_csv('FILE')
print(df.shape); print(df.dtypes); print(df.describe())
"

# large text → search, don't read
grep -n 'PATTERN' FILE | head -50
wc -l FILE
```

Extracting a still from a video, transcoding a preview, or checksumming for integrity are
all fine. Never `cat` a large file into the response.

---

## Writing results back

For small text results:

```
put(bucket, key, content)
```

For large files produced in the sandbox:

```
presign(bucket, key, method="PUT")
```
```bash
curl -sSfL -X PUT --upload-file /path/in/sandbox/result "<presigned-put-url>"
```

### `copy` and `move` are same-bucket only

Both operate **within a single bucket** — `dest_bucket` must equal the source bucket, so
neither can move an object to a different bucket. To place an object on a different storage
backend, use `migrate` with a `target_endpoint`. To get it into a *different bucket*, bridge
it: `presign` GET, pull to the sandbox, `presign` PUT to the destination, upload. Never
bridge by pulling the object into the response and writing it back out — that routes the
bytes through context, which the governing rule forbids.

All three are asynchronous and return once queued, so the destination may not be readable
immediately. Confirm with `head` before treating a copy as done.

---

## Storage events

The `onebucket-events` connector answers "what changed, and when" without listing buckets
and diffing. Events are scoped to the caller's organization and carry the bucket, key,
action (for example `s3:PutObject`), and timestamp.

- **`list_events`** — history, most recent first. Defaults to the last 24 hours; filter by
  `since`/`until` (RFC 3339), `bucket`, or `action`. Page with the returned `nextCursor`,
  keeping the filters identical across pages.
- **`get_event`** — one event by its `eventId`.
- **`tail_events`** — long-poll for new events, oldest first. Without a cursor it starts
  from *now*; use `list_events` for anything already in the past. Always pass the returned
  `nextCursor` to the next call. An empty result is normal — poll again.

Use cases: auditing who wrote to a bucket and when, confirming an upload from another
system has landed, or watching a prefix while a batch job runs. Use `tail_events` with a
sensible `waitSeconds` rather than polling `list` in a loop.

Events are a record of activity, not a source of object bytes. Once an event points at an
object of interest, go back to the `onebucket` connector and apply the transport rules
above.

---

## Anti-patterns

| Don't | Why |
|---|---|
| `cat` a large file, or open it with a file-reading tool | Defeats the pattern — bytes land in context |
| Pass a presigned URL to web fetch | Refused by design; no setting changes this |
| Split a large object into many inline `get` calls | Same context ceiling, reached more slowly |
| Ask the user to download and re-upload | The sandbox path exists precisely to avoid this |
| Work around an empty `get` response | Report it — it is a fault, not a size limit |
| Ask the user which region to use | There is one endpoint; DNS picks the region |
| Poll `list` in a loop to detect a change | `tail_events` blocks until something happens |
| Use `locate` to reason about regions | It reports storage backends, unrelated to routing |

## When a step fails

Name the step that failed and stop. Do not substitute a different transport to route
around an error or a refusal. An unexpected failure means something is broken or
misconfigured, and surfacing that is more valuable than obscuring it with a workaround.
