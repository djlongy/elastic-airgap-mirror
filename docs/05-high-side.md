# 5. High side

All manifests are in `examples/k8s/`. Replace `registry.example.internal`, the storage
class, the consumer namespaces and `elastic-mirror.example.internal` before applying.

## 5.1 Deploy

1. Create the secrets. Generate values into variables and keep the same values in your
   secret store, because NiFi needs the writer credentials.

       kubectl create namespace elastic-mirror
       kubectl -n elastic-mirror create secret generic minio-root \
         --from-literal=MINIO_ROOT_USER=mirror-admin \
         --from-literal=MINIO_ROOT_PASSWORD="$(openssl rand -base64 24)"
       WRITER_PASSWORD="$(openssl rand -base64 24)"
       kubectl -n elastic-mirror create secret generic minio-writer \
         --from-literal=WRITER_USER=transfer-writer \
         --from-literal=WRITER_PASSWORD="$WRITER_PASSWORD"
       kubectl -n elastic-mirror create secret docker-registry registry-pull \
         --docker-server=registry.example.internal \
         --docker-username=<puller> --docker-password="$PULL_TOKEN"

2. Apply the directory.

       kubectl apply -k examples/k8s

   The two bootstrap Jobs carry Argo CD `PostSync` hook annotations. Under Argo CD they run
   after every sync. With plain `kubectl apply` they run once, so delete a Job before
   re-applying it.

3. Wait for the registry. It indexes every package before it listens.

       kubectl -n elastic-mirror rollout status deploy/epr --timeout=20m

## 5.2 The EPR startup probe

The registry indexes its packages at start, about 7 minutes for the `lite` image. A liveness
probe that starts at once kills the container before it listens, and it restarts forever.
`epr.yaml` uses a `startupProbe` that allows 15 minutes (`periodSeconds: 10`,
`failureThreshold: 90`). Liveness and readiness start after it passes.

The image has no shell. Test it from another pod.

## 5.3 The store and its buckets

- Image: any S3-compatible store that returns the MD5 as the `ETag` for a single-part PUT
  and ships the `mc` client. Anonymous read is the point: Fleet, Endpoint and Elasticsearch
  send no credentials.
- `minio-bootstrap` creates buckets `elastic-mirror` and `elastic-kb` and a read policy
  (`s3:GetObject`, `s3:ListBucket`, `s3:GetBucketLocation`) for `*`. It answers 409 for a
  bucket that exists and replaces the policy with the same document, so it is safe to re-run.
- `minio-writer-bootstrap` creates the user `transfer-writer` with `PutObject`,
  `AbortMultipartUpload` and `ListMultipartUploadParts` on the two buckets and nothing
  else. It cannot read other keys, delete or administer.
- One ReadWriteOnce volume, so the Deployment uses `strategy: Recreate`.

## 5.4 TLS

`tls.yaml` makes a CA inside the namespace and one serving certificate for `minio` and
`epr` (names `<svc>`, `<svc>.elastic-mirror`, `.svc` and `.svc.cluster.local`). No external
CA is involved. The CA lives 10 years, the serving certificate 1 year with renewal 30 days
before expiry.

Consumers trust `ca.crt` of secret `elastic-mirror-tls`. Copy it into each consumer
namespace:

    kubectl -n elastic-mirror get secret elastic-mirror-tls -o jsonpath='{.data.ca\.crt}' \
      | base64 -d > mirror-ca.crt
    kubectl -n elastic-system create configmap elastic-mirror-ca --from-file=ca.crt=mirror-ca.crt

If the CA secret is ever recreated, copy the new `ca.crt` again. The ingress presents a
certificate from your own issuer and re-encrypts to the store, verified against the same
`ca.crt`.

## 5.5 Network isolation and its proof

`networkpolicy.yaml` denies all ingress and egress, then allows:

- Egress to cluster DNS and to pods of the same namespace.
- Ingress from the namespaces of your ECK stacks.
- Ingress on 9000 from the ingress controller, for agents outside the cluster.
- Ingress on 9000 from the high-side NiFi pods, for the writes.

Image pulls are made by the kubelet, so the registry needs no rule.

Prove that the pods cannot leave. `podSelector: {}` covers any pod in the namespace, so a
throwaway pod inherits the same denial. Run the probe in `examples/k8s/egress-probe.yaml`:

    kubectl -n elastic-mirror apply -f examples/k8s/egress-probe.yaml
    kubectl -n elastic-mirror wait --for=condition=complete job/egress-probe --timeout=120s
    kubectl -n elastic-mirror logs job/egress-probe
    kubectl -n elastic-mirror delete job/egress-probe

Expected: both external probes print `exit=28` (timeout) and the in-namespace probe prints
`200`. Also prove the allowed paths from a consumer pod, with the CA mounted:

    kubectl -n elastic-system exec deploy/kibana-kb -c kibana -- \
      curl -sS -m 10 --cacert /usr/share/kibana/mirror-ca/ca.crt \
      -o /dev/null -w '%{http_code}\n' https://epr.elastic-mirror.svc:8080/

A timeout means a policy or firewall rule is missing. A TLS error means trust.
