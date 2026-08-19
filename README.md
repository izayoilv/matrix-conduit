# matrix-conduit

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Conduit](https://img.shields.io/badge/homeserver-Conduit-5b5bd6.svg)](https://conduit.rs)
[![Matrix](https://img.shields.io/badge/domain-conduit.kudofools.dev-purple.svg)](https://conduit.kudofools.dev)
[![Database](https://img.shields.io/badge/database-RocksDB-6b9d4e.svg)](https://rocksdb.org)
[![Image](https://img.shields.io/badge/image-v0.10.12-2ea44f.svg)](https://hub.docker.com/r/matrixconduit/matrix-conduit)

Flux-managed [Conduit](https://conduit.rs) Matrix homeserver on the kudofools k3s cluster.

## Features

- Homeserver: `conduit.kudofools.dev` (Matrix IDs look like `@user:conduit.kudofools.dev`)
- Public federation delegated over port 443 via built-in `/.well-known/matrix/*` (Cloudflare Tunnel doesn't expose 8448)
- Token-gated registration; token injected from OpenBao via External Secrets Operator
- Runs unprivileged (UID 1000) with read-only root filesystem

## Structure

```
clusters/default/
├── namespace.yaml          # matrix-conduit namespace
├── pvc.yaml                # 5Gi local-path data volume
├── configmap.yaml          # conduit.toml
├── externalsecret.yaml     # registration token from OpenBao
├── deployment.yaml         # matrixconduit/matrix-conduit:v0.10.12
├── service.yaml
├── ingress.yaml            # conduit.kudofools.dev -> conduit:6167
└── networkpolicy.yaml      # ingress from kube-system (Traefik) only
```

## Configuration

| Setting | Value | Notes |
|---|---|---|
| `server_name` | `conduit.kudofools.dev` | Defines your Matrix ID suffix |
| `database_backend` / `database_path` | `rocksdb` / `/var/lib/matrix-conduit/` | Data on the PVC |
| `port` / `address` | `6167` / `0.0.0.0` | Container needs non-loopback bind |
| `max_request_size` | `20_000_000` | 20 MiB upload cap |
| `allow_federation` | `true` | Public federation |
| `allow_registration` | `true` | Token-gated; set `false` after onboarding |
| `trusted_servers` | `["matrix.org"]` | Key fetching |
| `[global.well_known]` | `conduit.kudofools.dev:443` | Delegates federation over 443 |

Deployment-level:

| Setting | Value |
|---|---|
| Image | `matrixconduit/matrix-conduit:v0.10.12` |
| Config path | `CONDUIT_CONFIG=/etc/conduit/conduit.toml` |
| Registration token | `CONDUIT_REGISTRATION_TOKEN` from `conduit-secrets` |
| Service links | disabled (`enableServiceLinks: false`) — avoids `CONDUIT_PORT` env collision |

## Prerequisites

- kudofools cluster with Flux, OpenBao, ESO, and Cloudflare tunnel (see [kudofools-infra](https://forgejo.kudofools.dev/izayoilv/kudofools-infra))
- OpenBao root token + unseal keys on the RPi5 (`~/.bao-keys.json`)

## Setup

The cluster-side plumbing (DNS, tunnel route, Vault policy, Flux resources) lives in kudofools-infra. On the RPi5:

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

Register an account at https://conduit.kudofools.dev with a token-capable client (e.g. the self-hosted [element-web](https://forgejo.kudofools.dev/izayoilv/element-web) or [Element](https://app.element.io) → "Edit homeserver").

After onboarding, disable public registration by editing `clusters/default/configmap.yaml` (`allow_registration = false`), push, and restart:

```bash
kubectl rollout restart deployment -n matrix-conduit conduit
```

## Operations

| Task | Command |
|---|---|
| Check health | `kubectl get pods -n matrix-conduit` |
| View logs | `kubectl logs -n matrix-conduit deploy/conduit` |
| Rotate registration token | patch `kv/matrix-conduit/secrets` in OpenBao → force ESO sync → rollout restart |
| Test federation | https://federationtester.matrix.org |

## License

[MIT](LICENSE) © 2026 IzayoiLv
