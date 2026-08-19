# matrix-conduit

Flux-managed deployment of [Conduit](https://conduit.rs) (Matrix homeserver) on the kudofools k3s cluster.

- Homeserver: `conduit.kudofools.dev` (Matrix IDs look like `@user:conduit.kudofools.dev`)
- Federation: enabled, delegated over port 443 via built-in `/.well-known/matrix/*` (Cloudflare Tunnel doesn't expose 8448)
- Storage: RocksDB on a 5Gi `local-path` PVC at `/var/lib/matrix-conduit`
- Secrets: registration token pulled from OpenBao (`kv/matrix-conduit/secrets`) via External Secrets Operator

## Layout

```
clusters/default/
├── namespace.yaml      # matrix-conduit namespace
├── pvc.yaml            # 5Gi local-path data volume
├── configmap.yaml      # conduit.toml (server_name, federation, well-known)
├── externalsecret.yaml # registration token from OpenBao
├── deployment.yaml     # matrixconduit/matrix-conduit:v0.10.12
├── service.yaml
├── ingress.yaml        # conduit.kudofools.dev -> conduit:6167
└── networkpolicy.yaml  # ingress from kube-system (Traefik) only
```

Applied by Flux from the parent repo's `clusters/default/matrix-conduit.yaml` (GitRepository + Kustomization).

## Setup

The cluster-side plumbing (DNS, tunnel route, Vault policy) lives in kudofools-infra (`opentofu/` and `clusters/default/matrix-conduit.yaml`). Once both repos are pushed, on the RPi5:

```bash
ROOT_TOKEN=$(jq -r '.root_token' ~/.bao-keys.json)
REGISTRATION_TOKEN=$(openssl rand -base64 24)

# 1. Store the registration token in OpenBao
kubectl exec -n openbao openbao-0 -- env BAO_TOKEN=$ROOT_TOKEN bao kv put kv/matrix-conduit/secrets \
  registration_token=$REGISTRATION_TOKEN

# 2. Force ESO to sync it into the cluster
kubectl annotate externalsecret -n matrix-conduit conduit-secrets force-sync=$(date +%s) --overwrite

# 3. Verify Conduit is up
kubectl get pods -n matrix-conduit
curl -s https://conduit.kudofools.dev/_matrix/client/versions
curl -s https://conduit.kudofools.dev/.well-known/matrix/server
```

Register an account at https://conduit.kudofools.dev with a token-capable client
(e.g. [Element](https://app.element.io) → "Edit homeserver" → `https://conduit.kudofools.dev`),
using the registration token above.

After creating your account(s), disable public registration by editing `clusters/default/configmap.yaml`
(`allow_registration = false`), push, and restart: `kubectl rollout restart deployment -n matrix-conduit conduit`.

## Operations

- **Reseal / pod restart**: unaffected by OpenBao resealing, but verify health with `kubectl logs -n matrix-conduit deploy/conduit`.
- **Rotate registration token**: patch `kv/matrix-conduit/secrets` in OpenBao, then force ESO sync (step 2 above) and restart the deployment.
- **Troubleshooting**: `kubectl logs -n matrix-conduit deploy/conduit`, `kubectl describe pod -n matrix-conduit`.
- **Federation test**: https://federationtester.matrix.org