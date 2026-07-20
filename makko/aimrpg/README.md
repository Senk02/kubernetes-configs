# aimrpg (makko)

Go aRPG game server. Image built by [`aimRPG/aRPG-server`](https://github.com/aimRPG/aRPG-server),
deployed by ArgoCD to namespace `aimrpg`. Public entry: `aimrpg.senk0.com` → `aimrpg-server:7350`
(TLS at the gateway). Postgres is internal.

## Deploys (automatic)

Bump `VERSION` in the app repo → CI publishes `ghcr.io/aimrpg/aimrpg-server:v<VERSION>` → Renovate
auto-merges the tag bump here → ArgoCD syncs. Rollback = revert the Renovate commit.

## Setup

- **Secrets**: fill the `REPLACE-UUID-*` fields in `postgres/external-secret.yaml` with your
  Bitwarden item UUIDs.
- **Private image**: pulled via `ghcr-secret` (shared GHCR PAT, `read:packages`). Give your
  Renovate app the same read token so it can bump tags.
- **Discord**: add `https://aimrpg.senk0.com/login/callback` to the app's OAuth2 redirects.

## Admin console (internal only)

```sh
kubectl -n aimrpg port-forward deploy/aimrpg-server 7351:7351
```
