# Kubernetes 可选组件（bl-template-one）

与 [bl-template](../bl-template/kubernetes/OPTIONAL_COMPONENTS.md) 共享同一套可选组件实现（位于 `kubernetes/components/`），**不含** `web-ide`、`navigation`（企业模板 bl-template 专有）。

## Profile 与可选组件

| Profile | Kubernetes 默认组件 |
|---------|---------------------|
| `minimal` | sealed-secret |
| `full` | minimal + istio + victoria-metrics-operator + victoria-metrics + grafana |
| `optional` | 示例：cnpg、redis、loki、otel-collector（见 `profiles/optional/kubernetes.yaml`） |

`optional` profile 可与 `minimal` / `full` 叠加使用，或在生成 args 后手动向 `kubernetes.projects[].components[]` 添加任意 `config.yaml` 中已注册的组件。

```bash
blcli init-args -r ./bl-template-one --profile minimal --org my-dev -o workspace/config/args.yaml
# 编辑 args.yaml 增加 cnpg、loki 等
blcli init ./bl-template-one -a workspace/config/args.yaml -o workspace/output -w
```

## 可用可选组件

| 组件 | 官方镜像 / Chart | 依赖 |
|------|------------------|------|
| cnpg | `ghcr.io/cloudnative-pg/cloudnative-pg` | — |
| redis | `bitnami/redis` | external-secrets |
| paradedb | ParadeDB Chart | cnpg |
| kafka | `bitnami/kafka` | — |
| juicefs | `juicedata/mount` | external-secrets |
| loki | Grafana Community Chart | — |
| otel-collector | OpenTelemetry 官方 Chart | — |
| uptrace | `uptrace/uptrace` | cnpg |
| n9e | `flashcatcloud/nightingale` | victoria-metrics-operator |
| bytebase | `bytebase/bytebase` | — |

完整说明与启用示例见 bl-template 的 [OPTIONAL_COMPONENTS.md](../bl-template/kubernetes/OPTIONAL_COMPONENTS.md)。
