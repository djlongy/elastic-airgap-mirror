# 7. Daily update

## 7.1 Files

The low-side flow runs on its cron (`0 0 3 * * ?`).

1. It checks each upstream file with a HEAD request, or by hash for GeoIP, and sends only
   files whose id (`path|etag|length`) is new.
2. The high-side flow writes them under the same bucket and key. A changed file replaces
   the old object in place. Nothing restarts and the store serves the new file at once.
3. The record of sent files has a 30-day age-off. Flush its keys to force a full resend.

Measured in this build: run 1 crossed 83 files. Run 2, with no upstream change, sent 0
(33 skipped: the head-checked files and the GeoIP overview). Run 3, after deleting two
sent-records, sent exactly 2 and the store held the same `ETag` as before.

Prove it on your side: run the low flow twice and compare the counters
(`elastic.low.sent`, `elastic.low.unchanged`, `elastic.high.stored`). The second run must
send only changed files.

## 7.2 Registry image

A new registry image carries newer rules and integrations. It changes per stack version,
not daily.

1. Copy the new tag across ([04-transfer.md](04-transfer.md)).
2. Change the tag and digest in `epr.yaml` in a merge request. The pod rolls.
3. Wait for `kubectl -n elastic-mirror rollout status deploy/epr --timeout=20m`. The new pod
   indexes for about 7 minutes before it is ready, and the old pod serves until then.
4. Check `/categories` and `/search` ([08-verify.md](08-verify.md)).

The image tagged with the stack version is a snapshot. In this build the 9.5.2 image served
470 packages including `security_detection_engine` 9.5.4, while the public registry served
494 packages and 9.5.11 for the same Kibana. For rule freshness between stack upgrades, copy
a newer registry image that is compatible with your Kibana.

## 7.3 Cycles

| Cycle | How often | Action |
|---|---|---|
| Defend artifacts | Daily (Elastic publishes continuously) | The flow handles it. A new `manifest_version` appears after the next endpoint poll |
| GeoIP | Daily. Databases expire after 30 days without an update | The flow handles it. The download interval is 3 days |
| Rules and integrations | Monthly, or when a rule fix matters | New registry image, 7.2 |
| Stack upgrade | Your support window | Registry image, then Elasticsearch and Kibana, then change the stack version parameter so the new agent binaries cross, then Fleet Server, then agents |
| Knowledge base | On stack upgrade | Change the knowledge base version parameter |

Old agent versions stay in the store until you delete them. Remove them after every agent
has upgraded.

## 7.4 Rollback

- **Registry image:** revert the merge request that changed the tag.
- **Files:** send the previous files again through the same flow. Objects overwrite in
  place. Or delete the bad key.
- **Stack settings:** section 6.8.
