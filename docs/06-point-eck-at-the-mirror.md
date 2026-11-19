# 6. Pointing ECK at the mirror

Scope of what was run: sections 1 to 5 and 8 were run end to end. The settings here are the
ones prepared for the stack cutover, written from the Elastic documentation. The final
change to a live stack was not applied when this guide was drafted.

These are configuration changes to a live stack. A Kibana that cannot reach the registry
loses its integration catalogue, so prove reachability first.

## 6.1 Before you change anything

1. Elasticsearch is green by `_cluster/health`. Do not use the ECK resource status. It lags
   ([09-gotchas.md](09-gotchas.md)).
2. The mirror pods are `Ready`.
3. From a Kibana pod, with the CA mounted, `https://epr.elastic-mirror.svc:8080/categories?kibana.version=<V>`
   and an object URL on the store both answer 200. See [08-verify.md](08-verify.md).
4. Keep the current Kibana and Elasticsearch resources for rollback.

## 6.2 Trust the mirror CA

The mirror CA is private. Neither Kibana nor the Elasticsearch JVM trusts it by default.

- **Kibana.** Node trusts the public CAs only. Mount `ca.crt` and set
  `NODE_EXTRA_CA_CERTS` to it. See `examples/eck/kibana.yaml`.
- **Elasticsearch.** The GeoIP downloader and the model repository use the JVM trust store.
  `-Djavax.net.ssl.trustStore` replaces the JDK default, so an init container copies the
  JDK `cacerts`, adds the mirror CA, and the JVM option points at the copy. See
  `examples/eck/elasticsearch.yaml`. The store password only satisfies the keystore format
  because the file holds public certificates.

If you prefer not to touch the JVM trust store, serve `/geoip` and `/models` over plain
HTTP on a restricted network and use `http://` in the two Elasticsearch settings.

## 6.3 Kibana settings

    xpack.fleet.isAirGapped: true
    xpack.fleet.registryUrl: "https://epr.elastic-mirror.svc:8080"
    xpack.productDocBase.artifactRepositoryUrl: "https://minio.elastic-mirror.svc:9000/elastic-kb"

With `isAirGapped: true` Kibana also installs a small set of rules bundled in its build
with no registry. Newer rules need the registry image. Kibana restarts on the change.

## 6.4 Elasticsearch settings

    ingest.geoip.downloader.endpoint: "https://minio.elastic-mirror.svc:9000/elastic-mirror/geoip/overview.json"
    xpack.ml.model_repository: "https://minio.elastic-mirror.svc:9000/elastic-mirror/models"

Both are static and go on every node. ECK rolls the nodes one at a time. Optionally set
`ingest.geoip.downloader.eager.download: true` (a dynamic cluster setting) to fetch the
databases at once, instead of when the first pipeline needs them.

## 6.5 After the change, in this order

1. Wait for the Kibana rollout, the Elasticsearch pods `Ready` and `_cluster/health` green.
2. Fleet, Integrations lists packages served by the registry pod.
3. `GET _ingest/geoip/stats` shows databases downloaded and no failures.
4. Only once agents are enrolled, do 6.6 and 6.7.

## 6.6 Fleet agent binary source

Fleet, Settings, Agent binary download, add a source with host
`https://<mirror-host>/elastic-mirror/downloads/` and tick default. API form (confirm the
body for your version):

    curl -sS -X POST "https://<kibana-host>/api/fleet/agent_download_sources" \
      -H "kbn-xsrf: true" -H "Content-Type: application/json" \
      -H "Authorization: ApiKey ${KIBANA_API_KEY}" \
      -d '{"name":"airgap mirror","host":"https://<mirror-host>/elastic-mirror/downloads/","is_default":true}'

Agents outside the cluster reach the store through the ingress,
`https://elastic-mirror.example.internal/<bucket>/<key>`. Give them the CA at install
(`--certificate-authorities`).

Agent upgrades also need the upgrade key. Copy `downloads/GPG-KEY-elastic-agent` to
`<FLEETSERVER_BINARY_DIR>/elastic-agent-upgrade-keys/default.pgp` on each Fleet Server, or set
the Fleet Server integration option `server.pgp.upstream_url`. Without a key an agent cannot
verify an upgrade binary.

## 6.7 Defend artifact base URL

In Kibana, Security, Manage, Policies, open each Defend policy, Advanced settings, and set
one value per operating system the policy covers:

    linux.advanced.artifacts.global.base_url:   https://elastic-mirror.example.internal/elastic-mirror
    windows.advanced.artifacts.global.base_url: https://elastic-mirror.example.internal/elastic-mirror
    mac.advanced.artifacts.global.base_url:     https://elastic-mirror.example.internal/elastic-mirror

The base URL is the bucket, because Endpoint requests
`<base>/downloads/endpoint/manifest/artifacts-<version>.zip` and then each path the manifest
lists.

## 6.8 Rollback

Revert the Kibana and Elasticsearch resources and let ECK roll the pods. Elasticsearch
restarts on the old settings and Kibana returns to the public registry. Delete the Fleet
binary source and clear the three Defend `base_url` values if you set them. The mirror pods
and files stay as they are.

At a site that is air-gapped for real, rollback means the old state, which cannot install
anything.
