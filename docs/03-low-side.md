# 3. Low side

The low side downloads upstream bytes and sends them unchanged. It builds nothing.

## 3.1 Proxy allowlist

Allow the NiFi source network to reach five hosts and nothing else. An example Squid
configuration is in `examples/squid/elastic-allowlist.conf`.

| Host | Serves |
|---|---|
| `artifacts.elastic.co` | Agent, fleet-server, endpoint-security binaries and the upgrade key |
| `artifacts.security.elastic.co` | Defend manifest and artifacts |
| `geoip.elastic.co` | The GeoIP index |
| `ml-models.elastic.co` | ELSER, e5 and rerank models |
| `kibana-knowledge-base-artifacts.elastic.co` | Knowledge base zips and the bucket listing |

GeoIP needs a sixth entry. The index JSON returns presigned URLs on an object-storage host
whose name is account-specific. Read it from the JSON and allow it, or the proxy answers
HTTP 403 for every database:

    curl -fsS 'https://geoip.elastic.co/v1/database?elastic_geoip_service_tos=agree' \
      | jq -r '.[].url' | cut -d/ -f3 | sort -u

Those URLs answer GET only. A HEAD request fails, so the flow cannot check them cheaply
and dedupes on the hash in the index instead.

Pair the proxy rule with a low-side `NetworkPolicy` that lets the NiFi pods reach the proxy
on its port and nothing else. Then a request to any other host is refused by the proxy,
and a request that skips the proxy times out.

## 3.2 What to download

The list per stack version `V` (linux x86_64). `scripts/mirror-bundle.sh` computes the
same list:

    scripts/mirror-bundle.sh -v 9.5.2 -o ./bundle --dry-run     # sizes only, downloads nothing

| Part | Files | Size |
|---|---|---|
| Agent binaries | `elastic-agent`, `fleet-server`, `endpoint-security` tar.gz, each with `.sha512` and `.asc`, plus `GPG-KEY-elastic-agent` | 464 MiB |
| Defend artifacts | `artifacts-<V>.zip` manifest and the 49 files it lists | 45 MiB |
| GeoIP | `GeoLite2-City`, `-Country`, `-ASN` `.tgz` and a derived `overview.json` | 30 MiB |
| ELSER v2 | `elser_model_2` and `elser_model_2_linux-x86_64` (`.pt`, `.vocab.json`, `.metadata.json`) | 679 MiB |
| multilingual e5 small | Both variants, three files each | 850 MiB |
| Rerank v1 | One variant, three files | 741 MiB |
| Knowledge base | `kb-product-doc-{security,kibana,observability,elasticsearch,openapi}-<major.minor>.zip` and the newest `security-labs-*.zip` | 80 MiB |
| **Total** | | **about 2.9 GiB** |

Each extra agent platform adds roughly 240 to 480 MiB. Skip the models and knowledge base
if you use neither ELSER nor the AI Assistant. Skip GeoIP only if no pipeline uses the
`geoip` processor (the Elastic integrations do).

## 3.3 The files that are derived

Two files are not upstream bytes.

- `geoip/overview.json`: the index JSON with each `url` replaced by the database file name.
  Elasticsearch requests `<endpoint>?elastic_geoip_service_tos=agree` and follows each
  relative `url` against the endpoint, so no presigned link crosses the link. The `age`
  field is dropped so the file does not change daily.
- `elastic-kb/index.xml`: not needed with an S3-compatible store. Its bucket listing is the
  `ListBucketResult` document Kibana reads. A web server needs the file, filtered to the
  keys you mirrored.

## 3.4 The image

One image crosses once per stack version:

    docker.elastic.co/package-registry/distribution:lite-9.5.2

The `lite` tag holds a common subset (about 17 packages in this build, see
[09-gotchas.md](09-gotchas.md)). The full `distribution:<V>` tag holds every integration at
21 GiB compressed. See [04-transfer.md](04-transfer.md) for how it crosses.
