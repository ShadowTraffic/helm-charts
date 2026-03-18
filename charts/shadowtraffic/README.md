# ShadowTraffic Helm Chart

Runs [ShadowTraffic](https://shadowtraffic.io) as a Kubernetes `Job`. ShadowTraffic is a synthetic data generator that streams data to backends like Kafka, Postgres, S3, and webhooks.

> **Beta:** These charts are not yet published to a Helm repository. Install locally by cloning this repo and referencing the chart by path (e.g. `helm install my-release charts/shadowtraffic`).

## Prerequisites

- Kubernetes 1.21+
- Helm 3.0+
- A ShadowTraffic license ([free tier available](https://shadowtraffic.io/pricing.html))

## Installing the Chart

### 1. Create a values file with your config

```yaml
# my-values.yaml
config:
  files:
    config.json: |
      {
        "generators": [
          {
            "topic": "orders",
            "value": {
              "orderId": { "_gen": "uuid" },
              "amount":  { "_gen": "number", "min": 1, "max": 1000 }
            }
          }
        ],
        "connections": {
          "kafka": {
            "kind": "kafka",
            "producerConfigs": {
              "bootstrap.servers": "my-kafka:9092"
            }
          }
        }
      }
```

### 2. Store your license in a `.env` file (keep this out of source control)

```bash
# license.env
LICENSE_EDITION=xxx
LICENSE_ID=xxx
LICENSE_EMAIL=xxx
LICENSE_ORGANIZATION=xxx
LICENSE_EXPIRATION=xxx
LICENSE_SIGNATURE=xxx
```

### 3. Install

```bash
set -a && source license.env && set +a

helm install my-release charts/shadowtraffic -f my-values.yaml \
  --set license.inline.edition="$LICENSE_EDITION" \
  --set license.inline.id="$LICENSE_ID" \
  --set license.inline.email="$LICENSE_EMAIL" \
  --set license.inline.organization="$LICENSE_ORGANIZATION" \
  --set license.inline.expiration="$LICENSE_EXPIRATION" \
  --set license.inline.signature="$LICENSE_SIGNATURE"
```

## Configuration

### Config

The chart supports two ways to provide a ShadowTraffic configuration:

**Inline (chart manages the ConfigMap):**

```yaml
config:
  files:
    config.json: |
      { ... }
    # Additional files accessible via loadJsonFile:
    customers.json: |
      { ... }
  entrypoint: config.json   # file passed to --config
  mountPath: /home/files
  configFormat: json        # json or yaml
```

**From a directory of files (recommended for multiple files):**

If you have multiple config files (e.g. a root config that references others via `loadJsonFile`), load them into the cluster as a ConfigMap using `kubectl`, then point the chart at it:

```bash
# Upload all files in your configs directory to Kubernetes
kubectl create configmap st-config --from-file=configs/
```

Then install the chart referencing that ConfigMap:

```bash
set -a && source license.env && set +a

helm install my-release charts/shadowtraffic \
  --set config.existingConfigMap=st-config \
  --set config.entrypoint=a.json \
  --set config.mountPath=/home/mounted-configs \
  --set license.inline.edition="$LICENSE_EDITION" \
  --set license.inline.id="$LICENSE_ID" \
  --set license.inline.email="$LICENSE_EMAIL" \
  --set license.inline.organization="$LICENSE_ORGANIZATION" \
  --set license.inline.expiration="$LICENSE_EXPIRATION" \
  --set license.inline.signature="$LICENSE_SIGNATURE"
```

The files are mounted into the container at `mountPath`, so any `loadJsonFile` references in your config should use that path (e.g. `/home/mounted-configs/b.json`).

To update your config files and rerun:

```bash
kubectl delete configmap st-config
kubectl create configmap st-config --from-file=configs/
helm upgrade my-release charts/shadowtraffic ...
```

**Existing ConfigMap (if you already have one):**

```yaml
config:
  existingConfigMap: my-configmap
```

### License

**Inline (chart manages the Secret):**

```yaml
license:
  inline:
    edition: ""
    id: ""
    email: ""
    organization: ""
    expiration: ""
    signature: ""
```

**Existing Secret:**

```yaml
license:
  existingSecret: my-secret
```

The Secret must contain the keys: `LICENSE_EDITION`, `LICENSE_ID`, `LICENSE_EMAIL`, `LICENSE_ORGANIZATION`, `LICENSE_EXPIRATION`, `LICENSE_SIGNATURE`.

### Common CLI Flags

| Value | CLI flag | Default | Description |
|-------|----------|---------|-------------|
| `sample` | `--sample` | `""` | Generate N events then stop |
| `seed` | `--seed` | `""` | Fixed seed for repeatable runs |
| `watch` | `--watch` | `false` | Reload config on changes |
| `stdout` | `--stdout` | `false` | Forward all output to stdout instead of backends |
| `quiet` | `--quiet` | `false` | Suppress status text |

### Job Settings

```yaml
job:
  backoffLimit: 0              # Don't retry on crash — failures are deterministic
  ttlSecondsAfterFinished: ""  # Auto-delete Job after completion (e.g. 300)
```

### Studio

ShadowTraffic Studio is a browser-based UI for inspecting generated data. It requires `--watch` and `--sample`.

```yaml
studio:
  enabled: true
  port: 8080

watch: true
sample: 100
```

Access it via port-forward:

```bash
kubectl port-forward job/my-release-shadowtraffic 8080:8080
```

Then open `http://localhost:8080`.

### Metrics

Prometheus metrics are always exposed on port `9400` at `/metrics`. The chart automatically adds scrape annotations to the pod.

```yaml
metrics:
  port: 9400
```

### Escape Hatches

```yaml
extraArgs:
  - --report-benchmark    # any additional CLI flags

extraEnv:
  - name: GOOGLE_APPLICATION_CREDENTIALS
    value: /var/secrets/gcp/key.json

extraVolumes: []
extraVolumeMounts: []
```

## Quick Local Test

To smoke-test without a real backend, use `--stdout` to print generated data to logs instead of writing to any system:

```bash
helm install st charts/shadowtraffic -f my-values.yaml \
  --set license.inline.id="$LICENSE_ID" \
  ... \
  --set stdout=true \
  --set sample=10
```

View the output:

```bash
kubectl logs -l app.kubernetes.io/instance=st --tail=-1
```

## Uninstalling

```bash
helm uninstall my-release
```

## Values Reference

| Key | Default | Description |
|-----|---------|-------------|
| `image.repository` | `shadowtraffic/shadowtraffic` | Container image |
| `image.tag` | `""` (uses Chart appVersion) | Image tag |
| `image.pullPolicy` | `IfNotPresent` | Pull policy |
| `config.files` | `{config.json: ""}` | Map of filename → content |
| `config.entrypoint` | `config.json` | File passed to `--config` |
| `config.mountPath` | `/home/files` | Directory the ConfigMap is mounted at |
| `config.existingConfigMap` | `""` | Use a pre-existing ConfigMap |
| `config.configFormat` | `json` | `json` or `yaml` |
| `license.inline.*` | `""` | License values (chart creates Secret) |
| `license.existingSecret` | `""` | Use a pre-existing Secret |
| `sample` | `""` | Stop after N events |
| `seed` | `""` | Seed for repeatable runs |
| `watch` | `false` | Reload on config changes |
| `stdout` | `false` | Output to stdout instead of backends |
| `quiet` | `false` | Suppress status text |
| `studio.enabled` | `false` | Enable Studio UI on port 8080 |
| `metrics.port` | `9400` | Prometheus metrics port |
| `job.backoffLimit` | `0` | Job retry limit |
| `job.ttlSecondsAfterFinished` | `""` | Auto-cleanup delay after completion |
| `service.type` | `ClusterIP` | Kubernetes Service type |
| `resources` | `{}` | Container resource requests/limits |
| `nodeSelector` | `{}` | Node selector |
| `tolerations` | `[]` | Tolerations |
| `affinity` | `{}` | Affinity rules |
| `extraArgs` | `[]` | Extra CLI flags |
| `extraEnv` | `[]` | Extra environment variables |
| `extraVolumes` | `[]` | Extra volumes |
| `extraVolumeMounts` | `[]` | Extra volume mounts |
