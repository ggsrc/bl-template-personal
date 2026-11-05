# Optional Kubernetes Components (bl-template-one)

Uses the same optional component implementations (under `kubernetes/components/`) as [bl-template](../bl-template/kubernetes/OPTIONAL_COMPONENTS.md), **without** `navigation` (it is specific to the enterprise bl-template).

## Profiles and optional components

| Profile | Default Kubernetes components |
|---------|---------------------|
| `minimal` | sealed-secret |
| `full` | minimal + istio + victoria-metrics-operator + victoria-metrics + grafana |
| `optional` | Examples: cnpg, redis, loki, otel-collector (see `profiles/optional/kubernetes.yaml`) |

The `optional` profile can be layered on top of `minimal` / `full`. Alternatively, after generating args, add any component registered in `config.yaml` to `kubernetes.projects[].components[]` by hand.

```bash
blcli init-args -r ./bl-template-one --profile minimal --org my-dev -o workspace/config/args.yaml
# Edit args.yaml to add cnpg, loki, etc.
blcli init ./bl-template-one -a workspace/config/args.yaml -o workspace/output -w
```

## Available optional components

| Component | Official image / chart | Depends on |
|------|------------------|------|
| cnpg | `ghcr.io/cloudnative-pg/cloudnative-pg` | — |
| redis | `bitnami/redis` | external-secrets |
| paradedb | ParadeDB Chart | cnpg |
| kafka | `bitnami/kafka` | — |
| juicefs | `juicedata/mount` | external-secrets |
| loki | Grafana Community Chart | — |
| otel-collector | Official OpenTelemetry chart | — |
| uptrace | `uptrace/uptrace` | cnpg |
| n9e | `flashcatcloud/nightingale` | victoria-metrics-operator |
| bytebase | `bytebase/bytebase` | — |

For full details and enablement examples, see bl-template's [OPTIONAL_COMPONENTS.md](../bl-template/kubernetes/OPTIONAL_COMPONENTS.md).
