# 4. Transfer

## 4.1 Image path

The registry image crosses once per stack version, not daily.

1. Low side: copy the image to a directory with its manifest digest preserved.

       skopeo copy --all --preserve-digests \
         docker://docker.elastic.co/package-registry/distribution:lite-9.5.2 \
         dir:./epr-lite-9.5.2

2. Carry the directory across the link with a checksum file.
3. High side: verify the checksums, then push it to the internal registry.

       skopeo copy --all --preserve-digests dir:./epr-lite-9.5.2 \
         docker://registry.example.internal/elastic/package-registry/distribution:lite-9.5.2
       skopeo inspect --format '{{.Digest}}' \
         docker://registry.example.internal/elastic/package-registry/distribution:lite-9.5.2

4. Put the printed digest in `examples/k8s/epr.yaml`. The Deployment pulls by digest.

A catalog-driven registry mirror automates this. It reads lines of `<source> <target>`
from a list file, pins each digest, and asks the low NiFi to carry what is new. A repository
allowlist in that flow must permit the target path, for example `elastic/*`.

`scripts/mirror-bundle.sh` writes `docker-archive` tars instead. That format regenerates
the manifest, so the digest after import differs from upstream. Pin the digest you read
from the high-side registry, not the upstream one.

## 4.2 File path

One scheduled low-side flow and one high-side flow. Both use native processors only.

### Low side flow

Parameters: stack version, knowledge base version, proxy host and port, pace
(`10 MB`), dedupe age (`30 days`), cache host, port and password.

| Step | Processor | Purpose |
|---|---|---|
| 1 | `GenerateFlowFile` on a cron (`0 0 3 * * ?`) | Content is the file list, `examples/nifi/file-list.txt` |
| 2 | `SplitText`, `ExtractText`, `UpdateAttribute` | One FlowFile per line, with `elastic.path`, `elastic.url`, `elastic.mode`, `elastic.expand` |
| 3 | `RouteOnAttribute` on mode | `head`, `nohead`, `always`, `inline` |
| 4 | `InvokeHTTP` HEAD through the proxy | Read `ETag` and `Content-Length` of the upstream file |
| 5 | `UpdateAttribute` | `elastic.id = path\|etag\|length` (path alone when the headers are empty) |
| 6 | `DetectDuplicate` (check only) | Drop files whose id was already sent |
| 7 | `InvokeHTTP` GET through the proxy | Fetch the file |
| 8 | `RouteOnAttribute` on expand | `endpoint`, `geoip`, `kb` branches |
| 9 | `ControlRate` | Pace the link, data rate |
| 10 | `UpdateAttribute` | Delete fetch and split attributes, keep `elastic.path` and `filename` |
| 11 | `PackageFlowFile` | FlowFile v3, so the path survives the link |
| 12 | `DetectDuplicate` (cache the id) | Record the file as sent, after it is packed |
| 13 | Output port to the link | |

Expand branches:

- `endpoint`: `UnpackContent` (zip) the manifest, `SplitJson` on
  `$.artifacts.*.relative_url`, then one GET per artifact at
  `https://artifacts.security.elastic.co<relative_url>`. The manifest zip itself is also
  sent. The 49 artifacts are not listed again while the manifest `ETag` is unchanged.
- `geoip`: `QueryRecord` keeps `GeoLite2-City`, `-Country` and `-ASN`. One FlowFile becomes
  `overview.json` (url set to the name) through `CryptographicHashContent` so a change in
  content changes the id. The others become one GET each, id `path|md5_hash`.
- `kb`: `EvaluateXPath` reads the bucket listing and selects the newest
  `security-labs-*.zip` that has no `--` in its name.

Record a file as sent only after step 11 succeeds. A file lost on the link is then
fetched again by the next run.

### High side flow

| Step | Processor | Purpose |
|---|---|---|
| 1 | Input port from the link | |
| 2 | `UnpackContent` (`flowfile-stream-v3`) | Restores `elastic.path` and the file bytes |
| 3 | `CryptographicHashContent` MD5 | Hash of the file |
| 4 | `UpdateAttribute` | `bucket = path before the first /`, `key = the rest` |
| 5 | `PutS3Object` | Path-style, one PUT, endpoint override to the store, truststore with the mirror CA |
| 6 | `RouteOnAttribute` | `s3.etag` (quotes removed) equals the MD5 |
| 7 | `LogMessage` at error | Mismatch, unpack failure and store failure each end in their own log sink |

Set `Multipart Threshold` and `Multipart Part Size` to the 5 GB maximum so every object is
one PUT. Then the store returns the MD5 as the `ETag`, with no dash. Use the write-only
store user, never the admin key.

## 4.3 Large files

The largest file is `rerank-v1.pt` at 738 MiB.

- **Back pressure.** Set the connection after the GET to an object threshold of 1 and a
  data size of 1 B. NiFi stops scheduling the GET while a file waits, so one large file
  is in flight at a time. Do not set this on loops (retry connections) because a full
  small limit on every connection in a loop deadlocks it.
- **Chunking.** If the link carries bounded messages, split the packed file into chunks and
  reassemble with `MergeContent` in `Defragment` mode. Defragment ignores `Maximum Number
  of Entries`, so a 738 MiB file in 2950 chunks of 256 KB merges although the limit is
  1000. A larger chunk size cuts the manifest rows if the run slows.
- **Content repository.** The original, the compressed copy, the encrypted copy, the
  chunks, the merged file and the decrypted file can all exist at once. With every file
  in flight the peak was 5.2 GiB of 7.8 GiB on one node, before the one-file back pressure.
  Size the repository to several times the largest file and prove it with that file.
- **Heap.** `MergeContent` of thousands of chunks and `UnpackContent` of a 738 MiB package
  ran in 1 GiB of heap.
- **Pace.** A `ControlRate` of 10 MB per second matched the throttled link in this build.
  Set yours to the link.

## 4.4 Skipping resend copies

Some links keep a resend copy of every batch for a retention period. For the file class the
copy is redundant, because every file is re-fetchable from upstream and the dedupe record
is written only after the file leaves. Route the file class around the copy step:

    resend copy? = RouteOnAttribute, retain = ${diode.class:equals('files'):not()}

In this build a 738 MiB file stalled at the copy step because the resend store, 10 GiB,
was 97 percent full of 30 days of copies. Keep copies for classes that cannot be
re-fetched.
