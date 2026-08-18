---
name: onebucket-objects
description: "Read, inspect, and process objects in OneBucket S3 storage — including large binary files (video, images, archives, datasets) that exceed inline size limits. Use whenever a task involves a stored object: reading a file from a bucket, analyzing a stored video or image, extracting metadata, searching a large dataset, or writing results back to storage. Covers choosing the right region (US East, US West) and the correct transport for objects of any size."
---

# Working with OneBucket objects

## The rule that governs everything

**Object bytes must never travel through the model's context window.**

A 95 MB object is roughly 25 million tokens. The size limits on `get` exist to protect
context, not bandwidth. For anything large, move the processing to the bytes rather than
the bytes to the model.

## Choose a region — a latency choice, not a data choice

Two endpoints are configured, each exposing its own copy of every tool:

| Region | Tool prefix | Site |
|---|---|---|
| US East | `onebucket-us-east` | Ashburn |
| US West | `onebucket-us-west` | San Jose |

**Both are points of presence onto the same logical storage** — the same buckets, the same
objects, the same backends, over shared metadata. Either region returns the same answer.
Calling the farther one costs time, never correctness.

Consequently:

- **A not-found is authoritative.** If an object isn't there through one region, it isn't
  there. Do not re-check the other region "to be sure" — it doubles the calls and tells you
  nothing new.
- **Prefer the region the user names**, or the one nearer to them if they've said where they
  are. With no signal either is correct — pick `onebucket-us-east` and proceed rather than
  asking which they want.
- **Stay on one region for a whole task.** Not for read correctness, but because `copy`,
  `move`, and `migrate` are asynchronous — they return once *queued*. Writing through one
  region and immediately reading through the other can observe the pre-write state. Keep any
  read-after-write sequence on a single region.
- **Never fan out the same query to both**, and never compare their answers hunting for a
  discrepancy. The data is shared; a genuine difference would be a fault to report, not a
  routing hint.

### If a region is not authorized

Each region is a separate connector with its own authorization. If a region's tools fail
with an authentication or authorization error, that region is not enabled for this user:
say so once, continue with the region that works, and do not retry.

### `locate` and `migrate` are about backends, not regions

`locate` reports which *storage backends* currently hold an object; `migrate` copies it to
another backend. That is the real placement primitive, it operates inside the shared Core,
and it is unrelated to which regional endpoint you called. Neither tool tells you anything
about `us-east` versus `us-west`.

---

## Choose a transport before transferring anything

Once the region is settled, call `list` first. It returns every object with its exact
size. Use that to pick a path.

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

Write through the same region you read through, so an async write and any follow-up read
stay on one endpoint.

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

## Anti-patterns

| Don't | Why |
|---|---|
| `cat` a large file, or open it with a file-reading tool | Defeats the pattern — bytes land in context |
| Pass a presigned URL to web fetch | Refused by design; no setting changes this |
| Split a large object into many inline `get` calls | Same context ceiling, reached more slowly |
| Ask the user to download and re-upload | The sandbox path exists precisely to avoid this |
| Work around an empty `get` response | Report it — it is a fault, not a size limit |
| Re-check the other region after a not-found | Same data both sides — a miss is authoritative |
| Fan out the same query to both regions | Same answer, twice the calls, no new information |
| Read through one region straight after an async write to the other | `copy`/`move`/`migrate` return when *queued* — you may see the pre-write state |
| Ask the user which region to use | It's a latency choice; pick one and proceed |
| Use `locate` to pick a region | It reports storage backends, which is unrelated to the endpoint you called |

## When a step fails

Name the step that failed and stop. Do not substitute a different transport to route
around an error or a refusal. An unexpected failure means something is broken or
misconfigured, and surfacing that is more valuable than obscuring it with a workaround.
