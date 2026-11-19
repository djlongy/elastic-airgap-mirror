# 8. Verify

Set the variables. Use a pod that holds the mirror CA (the Kibana pod) or a workstation
with a route and the CA file.

    V=9.5.2
    EPR=https://epr.elastic-mirror.svc:8080
    FILES=https://minio.elastic-mirror.svc:9000/elastic-mirror
    KB=https://minio.elastic-mirror.svc:9000/elastic-kb
    CA=/usr/share/kibana/mirror-ca/ca.crt

## 8.1 Registry

    curl -sS --cacert $CA "$EPR/categories?kibana.version=$V" | jq length
    curl -sS --cacert $CA "$EPR/search?kibana.version=$V&prerelease=false" | jq length
    curl -sS --cacert $CA "$EPR/search?package=security_detection_engine&kibana.version=$V" \
      | jq -r '.[0].version'

Both counts must be above 0.

## 8.2 Defend manifest and its `ETag`

    curl -sSI --cacert $CA "$FILES/downloads/endpoint/manifest/artifacts-$V.zip" | grep -i '^etag'
    curl -sS --cacert $CA -o m.zip "$FILES/downloads/endpoint/manifest/artifacts-$V.zip"
    md5sum m.zip                                    # macOS: md5 -q m.zip
    unzip -p m.zip manifest.json | jq -r .manifest_version

The `ETag` must be `"<32 hex characters>"` and equal the `md5sum`. A dash in it means a
multipart upload, and Endpoint will never refresh.

## 8.3 GeoIP, models, knowledge base

    curl -sS --cacert $CA "$FILES/geoip/overview.json" | jq '.[].name'
    curl -sSI --cacert $CA "$FILES/models/elser_model_2.metadata.json" | head -1
    curl -sS  --cacert $CA "$KB/" | head -c 400

The overview lists `GeoLite2-ASN.tgz`, `-City.tgz` and `-Country.tgz`. The knowledge base
listing starts with `<ListBucketResult`.

## 8.4 All of it at once

    scripts/verify-mirror.sh -v "$V" -e "$EPR" -f "$FILES" -b "$KB" -c ./mirror-ca.crt

Expect `ok` on every line and `N checks passed`. A `FAIL` stops the script. A `warn` names
something you did not mirror, such as a registry package absent from the `lite` image.

## 8.5 End to end, after the cutover

1. Fleet, Integrations lists the catalogue and an install of `System` succeeds with the
   internet blocked.
2. The proxy and firewall logs show no hits for `epr.elastic.co`, `artifacts.elastic.co`,
   `artifacts.security.elastic.co`, `geoip.elastic.co`, `ml-models.elastic.co` from the
   stack networks.
3. `GET _ingest/geoip/stats` shows databases downloaded and no failures.
4. In Discover, `metrics-*`: `Endpoint.policy.applied.artifacts.global.version` equals the
   `manifest_version` from 8.2.
5. Upgrade one test agent from Fleet to a version you mirrored. It downloads from the mirror
   and verifies the signature.
6. Security, Rules, Add Elastic rules, Install all installs `security_detection_engine`
   from your registry.
