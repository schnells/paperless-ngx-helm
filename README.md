# paperless-ngx Helm chart

[![Lint and render](https://github.com/schnells/paperless-ngx-helm/actions/workflows/lint.yaml/badge.svg)](https://github.com/schnells/paperless-ngx-helm/actions/workflows/lint.yaml)

Source: <https://github.com/schnells/paperless-ngx-helm>

A small, community-maintained Helm chart for
[Paperless-ngx](https://github.com/paperless-ngx/paperless-ngx).

It deliberately **does not bundle** a database, Redis, Gotenberg or Tika:

- Paperless uses **SQLite** on the data PVC by default (set
  `PAPERLESS_DBENGINE`/`PAPERLESS_DBHOST` for PostgreSQL/MariaDB).
- Point `PAPERLESS_REDIS` at **Valkey or Redis** (see below). Redis is only
  the Celery broker/cache — documents and the database live elsewhere.

That keeps the chart small and avoids the maintenance trap of pinned
third-party subcharts (e.g. old Bitnami Redis images disappearing from Docker
Hub).

> This is **not an official Paperless-ngx chart**. The upstream project does
> not maintain a Helm chart; this is a minimal community alternative.

## Why this chart?

The most popular community charts either lag behind Paperless releases or pull
in a large stack (PostgreSQL operator, Redis, Gotenberg, Tika). This chart is
intentionally small and does one thing: run the Paperless-ngx container with
its PVCs and an optional Ingress. Database and Redis/Valkey are your choice.

It is easy to audit, has no subcharts, and can be updated by changing a single
`appVersion`.

## Requirements

- Kubernetes 1.25+
- Helm 3.10+
- A StorageClass (unless you point `persistence.*.existingClaim` at existing
  PVCs)
- A Redis-compatible broker (Valkey or Redis) reachable via `PAPERLESS_REDIS`

## What it creates

- `Deployment` (default strategy `Recreate`) and `Service`
- optional `Ingress`
- PVCs for `data`, `media`, `consume`, `export` — or `existingClaim` refs
- no CRDs, no operators, no subcharts

## Install

From a local clone:

```bash
git clone https://github.com/schnells/paperless-ngx-helm
cd paperless-ngx-helm
helm install paperless-ngx . \
  --namespace paperless --create-namespace \
  -f my-values.yaml
```

From OCI (after publishing, see below):

```bash
helm install paperless-ngx oci://ghcr.io/schnells/charts/paperless-ngx \
  --version 0.1.0 --namespace paperless
```

## Example values (SQLite + external Valkey + OIDC)

```yaml
env:
  TZ: Europe/Berlin
  PAPERLESS_URL: https://paperless.example.com
  PAPERLESS_CSRF_TRUSTED_ORIGINS: https://paperless.example.com
  PAPERLESS_SECRET_KEY:
    valueFrom:
      secretKeyRef:
        name: paperless-secret-key
        key: secret-key
  PAPERLESS_REDIS:
    valueFrom:
      secretKeyRef:
        name: paperless-redis
        key: url
  PAPERLESS_APPS: allauth.socialaccount.providers.openid_connect
  PAPERLESS_SOCIALACCOUNT_PROVIDERS:
    valueFrom:
      secretKeyRef:
        name: paperless-oidc
        key: providers.json

persistence:
  data:
    existingClaim: paperless-ngx-data
  media:
    existingClaim: paperless-ngx-media
  consume:
    existingClaim: paperless-ngx-consume
  export:
    existingClaim: paperless-ngx-export

ingress:
  enabled: true
  className: traefik
  # Optional: keep an existing Ingress object name (e.g. when adopting a
  # release installed from another chart).
  nameOverride: paperless
  hosts:
    - host: paperless.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: paperless-tls
      hosts:
        - paperless.example.com
```

If `existingClaim` is empty, the chart creates the PVC (`size`, `accessModes`,
`storageClass`) and keeps it on `helm uninstall`
(`helm.sh/resource-policy: keep`).

## Valkey / Redis

Paperless only needs a Redis-compatible broker/cache. A minimal Valkey:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: valkey
  namespace: paperless
spec:
  replicas: 1
  selector:
    matchLabels:
      app: valkey
  template:
    metadata:
      labels:
        app: valkey
    spec:
      containers:
        - name: valkey
          image: valkey/valkey:9.2-alpine
          args:
            - valkey-server
            - --requirepass
            - $(VALKEY_PASSWORD)
            - --appendonly
            - "no"
          env:
            - name: VALKEY_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: paperless-redis
                  key: password
---
apiVersion: v1
kind: Service
metadata:
  name: valkey
  namespace: paperless
spec:
  selector:
    app: valkey
  ports:
    - port: 6379
      targetPort: 6379
```

And the URL secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: paperless-redis
  namespace: paperless
stringData:
  url: redis://:<password>@valkey:6379/0
```

No persistence is required (broker/cache). A restart may lose queued tasks, not
documents. The official `redis` image works the same way.

## OIDC / social login

Set `PAPERLESS_APPS` to
`allauth.socialaccount.providers.openid_connect` and provide the provider JSON
through a Secret (as in the example above). The JSON format is documented at
<https://docs.paperless-ngx.com/configuration/#PAPERLESS_SOCIALACCOUNT_PROVIDERS>.

## Main values

| Value | Default | Description |
|---|---|---|
| `image.tag` | `Chart.appVersion` | Paperless image tag |
| `env` | `{}` | Env vars; values may be strings or `valueFrom` maps |
| `persistence.*.existingClaim` | `""` | Use an existing PVC instead of creating one |
| `persistence.*.size` | `10Gi`/`20Gi`/… | PVC size when the chart creates it |
| `ingress.enabled` | `false` | Create a `networking.k8s.io/v1` Ingress |
| `ingress.nameOverride` | `""` | Override the Ingress object name |
| `probes.startup.failureThreshold` | `40` (@30s) | 20-minute startup budget for migrations/indexing |
| `strategy.type` | `Recreate` | Required with RWO PVCs / single consumer |

## Updating and publishing

- **Renovate** (`renovate.json`) tracks upstream Paperless releases through
  `Chart.yaml` `appVersion`. `image.tag` defaults to it, so no second place to
  update.
- After merging an app update, bump `version` in `Chart.yaml` (chart SemVer:
  app update = minor, chart-only fix = patch), then tag `vX.Y.Z`.
- The included workflow publishes the packaged chart to
  `oci://ghcr.io/schnells/charts` on a `v*` tag. Alternative: GitHub Pages via
  [`chart-releaser-action`](https://github.com/helm/chart-releaser-action).
- CI (`.github/workflows/lint.yaml`) runs `helm lint` and `helm template` with
  defaults and `ci/values-test.yaml`.

The workflows assume the chart is at the repository root. To publish, copy this
directory into its own repository (or adjust the paths).

## Contributing

Issues and pull requests are welcome. Please keep the chart minimal — no
database/Redis/Gotenberg/Tika subcharts; those belong outside this chart.

Before opening a PR:

```bash
helm lint --strict .
helm template test . > /dev/null
helm template test . -f ci/values-test.yaml > /dev/null
```

Never put secrets into `values.yaml`; use `env` with `valueFrom`/`secretKeyRef`
or `existingClaim` references instead.

## Tested on

- k3s (arm64; CM3588) with Helm 3.x, Paperless-ngx 3.5.1
- The Paperless image is multi-arch, so amd64 works as well.

## License

MIT — see [LICENSE](LICENSE).
