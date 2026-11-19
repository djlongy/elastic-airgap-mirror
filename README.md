# elastic-airgap-mirror

## TL;DR

Offline mirror of Elastic Fleet, Defend, GeoIP, ML and docs content, served from Kubernetes with no egress: `scripts/mirror-bundle.sh -v 9.5.2 -o ./bundle --dry-run`.

## Requirements

- Low side: internet through an allowlisted forward proxy, NiFi 2.x, a distributed map cache (Redis), `skopeo` 1.13 or later, `curl`, `jq`, `python3` 3.9 or later.
- High side: Kubernetes with an enforcing `NetworkPolicy` CNI, cert-manager, an internal image registry, NiFi 2.x, ECK running Elastic Stack 9.5.x.
- A one-way transfer between the sides that carries files up to about 1 GiB.
- Disk: 50 GiB store volume. One stack version of files is about 2.9 GiB, the `lite` registry image 3.9 GiB.

## Contents

| Path | What it is |
|---|---|
| `docs/01-architecture.md` | The seven things an offline stack fetches, the design and the store layout |
| `docs/02-prerequisites.md` | Requirements and measured sizing |
| `docs/03-low-side.md` | Proxy allowlist, download list and sizes |
| `docs/04-transfer.md` | Image path, file path, large files, skipping resend copies |
| `docs/05-high-side.md` | Deploy, startup probe, buckets, TLS, egress-deny proof |
| `docs/06-point-eck-at-the-mirror.md` | Kibana, Elasticsearch, Fleet and Defend settings, CA trust, rollback |
| `docs/07-daily-update.md` | Daily file run, registry image update, rollback |
| `docs/08-verify.md` | The curl checks and the end-to-end test |
| `docs/09-gotchas.md` | Failures met while building it |
| `examples/k8s/` | Namespace, store, registry, TLS, network policy, ingress and egress probe |
| `examples/squid/` | Proxy allowlist |
| `examples/nifi/file-list.txt` | The file list that drives the low-side flow |
| `examples/eck/` | Kibana and Elasticsearch resource fragments |
| `scripts/mirror-bundle.sh` | Computes sizes, or exports every file and image into one bundle |
| `scripts/verify-mirror.sh` | Probes the registry and the store like Kibana, Fleet and Endpoint do |

## Usage

```sh
scripts/mirror-bundle.sh -v 9.5.2 -o ./bundle --dry-run     # sizes only, downloads nothing
kubectl apply -k examples/k8s                               # after the secrets in docs/05-high-side.md
scripts/verify-mirror.sh -v 9.5.2 -e https://epr.example.internal:8080 \
  -f https://elastic-mirror.example.internal/elastic-mirror \
  -b https://elastic-mirror.example.internal/elastic-kb -c ./mirror-ca.crt
```

## Preconditions

- Replace `registry.example.internal`, `elastic-mirror.example.internal`, the storage class and the consumer namespace in `examples/` before applying.
- The NiFi flows are described in `docs/04-transfer.md`, not shipped as JSON.

## Out of scope

- Licences, Elastic-hosted inference and telemetry cannot work offline.
- The one-way link itself. The guide assumes one exists.
- Pointing a live stack at the mirror is written from Elastic's documentation and has not been applied end to end. Sections 1 to 5 and 8 were run.

## Expected result

An `elastic-mirror` namespace with `epr` and `minio` pods that answer `/categories`, serve `downloads/endpoint/manifest/artifacts-9.5.2.zip` with a plain MD5 `ETag`, and cannot reach the internet. Check with `scripts/verify-mirror.sh`.
