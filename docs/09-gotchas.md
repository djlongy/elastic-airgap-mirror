# 9. Gotchas

| # | Gotcha | What to do |
|---|---|---|
| G1 | The `lite` registry covers about 17 packages in this build. `apm` is absent, and an integration outside the subset is not offered in Fleet | Compare `/search` against the integrations you run before you choose `lite`. Use the full tag (21 GiB compressed) if you need more |
| G2 | A liveness probe kills the registry while it indexes (about 7 minutes) and it never starts | Use a `startupProbe` with a 15 minute budget. Liveness and readiness begin after it |
| G3 | GeoIP downloads fail with HTTP 403 at the proxy although `geoip.elastic.co` is allowed | The index returns presigned URLs on an account-specific object-storage host. Allow that host. Those URLs answer GET only, so HEAD fails |
| G4 | The one-way link's resend store fills with 30 days of copies and a large file stalls at the copy step | Skip the copy for re-fetchable classes ([04-transfer.md](04-transfer.md)). Size the store for the classes that keep copies |
| G5 | Configuration-management check mode cannot dry-run a NiFi flow that is not yet imported, because later tasks reference a group that does not exist | Prove new flows in an unversioned sandbox group. Use check mode only for changes to a flow that already exists |
| G6 | The ECK `Elasticsearch` status lags the cluster. It can still show a non-green health after the cluster API reports green | Gate on `GET _cluster/health`, not on the ECK resource |
| G7 | Endpoint never refreshes artifacts | The `ETag` has a dash. Write each object with one PUT (multipart threshold 5 GB) and confirm the `ETag` equals the MD5 |
| G8 | `docker-archive` image tars change the manifest digest | Pin the digest read from the high-side registry after import |
| G9 | A GitOps sync that waits on an unhealthy resource ignores later commits | Terminate the sync operation, then sync again |
| G10 | Kibana loses its integration catalogue if it cannot reach the registry | Prove DNS, network policy and CA trust from the Kibana pod before the change |
| G11 | The Fleet binary source and the Defend `base_url` do nothing for a cluster with no agents | Set them once agents are enrolled |

## What an air-gapped install still cannot do

- Elastic-hosted inference (Managed LLM, hosted ELSER) needs Elastic Cloud. Use a local
  model or an LLM connector you host.
- A trial or paid licence file must be obtained and applied while connected, or through your
  Elastic account team. Features gated on licence level stay off without it.
- Elastic Security Labs, blog and release-note links in the UI do not open.
- Telemetry and the latest-agent-versions lookup are skipped in air-gapped mode. The list
  of upgrade versions comes from the binaries you mirrored.
