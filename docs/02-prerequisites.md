# 2. Prerequisites

Elastic facts in this guide were read from the Elastic documentation on 2026-11-19 against
stack 9.5.x. Re-check each setting name against the documentation for your version.

## Low side (has internet)

| Need | Detail |
|---|---|
| Forward proxy | Squid or equivalent, with an allowlist per source network. See [03-low-side.md](03-low-side.md) |
| NiFi 2.x | Reaches the proxy and the one-way link. Native processors only |
| Distributed map cache | Redis or equivalent, used to remember which files were sent |
| Image copy tool | `skopeo` 1.13 or later, `curl`, `jq`, `python3` |

## High side (no internet)

| Need | Detail |
|---|---|
| Kubernetes | A CNI that enforces `NetworkPolicy` (Calico, Cilium) |
| cert-manager | Issues the mirror CA and serving certificate |
| Ingress controller | Only if agents outside the cluster read the file store |
| Internal image registry | Holds the EPR, store and `curl` images, pulled by digest |
| NiFi 2.x | Receives from the link and writes to the store over TLS |
| ECK operator | Already running Elasticsearch and Kibana at the stack version you mirror |
| Secret store | Holds the store admin key, the write-only key and the NiFi truststore password |

## The link

Any one-way transfer works if it can carry a FlowFile v3 package of up to about 1 GiB.
Attributes do not cross a link that moves plain bytes, so the low side packs each file
with its attributes and the high side unpacks it. If your link has a message size limit,
chunk on the way in and reassemble on the way out ([04-transfer.md](04-transfer.md)).

## Sizing

| Resource | Measured |
|---|---|
| Store volume | 50 GiB. One stack version of files is about 2.9 GiB |
| EPR `lite` image | 3.9 GiB compressed. The full image is 21 GiB compressed and 43.6 GiB extracted |
| NiFi content repository | 8 GiB per node held a 738 MiB file. See [04-transfer.md](04-transfer.md) |
| NiFi heap | 1 GiB per node handled the 738 MiB unpack |
| EPR memory | 512 MiB request, 2 GiB limit |

## Version pins

Pick one stack version `V` (for example `9.5.2`). It must equal your Kibana version. The
knowledge base uses the minor version (`9.5`). The ECK operator image is separate.
