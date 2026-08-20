# Setup

## Prerequisites

- kudofools cluster with Flux, OpenBao, ESO, and Cloudflare tunnel (see [kudofools-infra](https://forgejo.kudofools.dev/izayoilv/kudofools-infra))
- OpenBao root token + unseal keys (`~/.bao-keys.json`)
- A reachable eturnal TURN server at `turn.kudofools.dev:3478`

## 1. Seed secrets into OpenBao

All secrets are managed by OpenBao + ESO. No secrets are committed to Git.

```bash
ROOT_TOKEN=$(jq -r '.root_token' ~/.bao-keys.json)

# Registration token — new users must provide this when registering
REGISTRATION_TOKEN=$(openssl rand -base64 24)
kubectl exec -n openbao openbao-0 -- env BAO_TOKEN=$ROOT_TOKEN bao kv put kv/matrix-conduit/secrets \
  registration_token=$REGISTRATION_TOKEN

# TURN shared secret — generated here, seeded to OpenBao, then copied to the eturnal server.
# Conduit derives ephemeral TURN credentials from it (HMAC-SHA1 over the secret).
TURN_SECRET=$(openssl rand -base64 32)
kubectl exec -n openbao openbao-0 -- env BAO_TOKEN=$ROOT_TOKEN bao kv patch kv/matrix-conduit/secrets \
  TURN_SECRET=$TURN_SECRET

echo "Copy this value into the eturnal server's secret: $TURN_SECRET"
```

## 2. Force ESO to sync the secrets into the cluster

```bash
kubectl annotate externalsecret -n matrix-conduit conduit-secrets force-sync=$(date +%s) --overwrite
```

## 3. Restart Conduit and verify

```bash
kubectl rollout restart deployment -n matrix-conduit conduit
kubectl get pods -n matrix-conduit
curl -s https://conduit.kudofools.dev/_matrix/client/versions
curl -s https://conduit.kudofools.dev/.well-known/matrix/server
```

## 4. Verify TURN is working

1. Make a voice/video call between two accounts in Element (https://element.kudofools.dev).
2. If both clients are on the same LAN, WebRTC uses a direct peer-to-peer path and never touches TURN — that's expected and correct.
3. To force the TURN relay path, disable P2P in the call (Element → call settings → force relay) or test from two different networks.

For a protocol-level check, an authenticated client can query the TURN config the homeserver advertises:

```bash
# From an authenticated session:
curl -s -H "Authorization: Bearer <access-token>" \
  https://conduit.kudofools.dev/_matrix/client/v3/voip/turnServer
# Expect "uris": ["turn:turn.kudofools.dev:3478?transport=udp", "...tcp"], username, password
```

## 5. Register an account

Open https://conduit.kudofools.dev with a token-capable client (e.g. the self-hosted [element-web](https://forgejo.kudofools.dev/izayoilv/element-web) or [Element](https://app.element.io) → "Edit homeserver") and register using the registration token from step 1.

## 6. Disable public registration (after onboarding)

Edit `clusters/default/configmap.yaml` (`allow_registration = false`), push, and restart:

```bash
kubectl rollout restart deployment -n matrix-conduit conduit
```

## Rotating the TURN secret

1. Generate a new secret at the rpi5 and patch `kv/matrix-conduit/secrets` in OpenBao.
2. Copy the new value into the eturnal server's `secret` and restart eturnal.
3. Force ESO sync (step 2) and restart Conduit (step 3).
