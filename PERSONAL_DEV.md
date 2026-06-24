# bl-template-one 个人开发者指南（含 AI Agent）

本文档面向 **个人 GCP 账号 + AI 辅助开发**，说明 template-one 的默认路径、profile 选择、必改参数与常见误区。

## 1. 设计目标

| 目标 | 做法 |
|------|------|
| 低配置负担 | 默认 `minimal` profile；无 ArgoCD、无 Dex OAuth |
| 渐进复杂度 | 需要域名/TLS/监控时再 `--profile full` |
| AI 可推理 | 自描述 `args.yaml` + `blcli explain` + `profiles/README.md` |
| 应用部署 | **kubectl / helm**（不用 GitOps CD） |

## 2. Profile 怎么选

```bash
# 默认：个人实验 / 生成验证 / 无真实域名
blcli init-args -r ./bl-template-one --org my-dev -o args.yaml

# 等价于
blcli init-args -r ./bl-template-one --profile minimal --org my-dev -o args.yaml

# 需要 Cloud DNS + 证书 + 监控栈
blcli init-args -r ./bl-template-one --profile full --org my-dev -o args.yaml
```

### minimal（默认）

**Terraform 组件：** main, backend, provider, variables, vpc, ip, firewall, gke, outputs  
**不含：** dns, cert  
**Kubernetes 组件：** sealed-secret  
**适合：** Phase 3–4 本地生成与 check、最小 GCP 账单、AI 迭代 infra 代码

### full

在 minimal 基础上 **叠加**（见 `profiles/full/`）：

**Terraform 增加：** dns, cert，并扩展 outputs 的 certificate_maps  
**Kubernetes 增加：** istio, victoria-metrics-operator, victoria-metrics, grafana  

**适合：** 已有真实域名、需要 GCP 托管证书与可观测性

## 3. AI Agent 必改 / 勿改清单

### init-args 之后必改

```yaml
global:
  GlobalName: my-dev          # 与 --org 一致即可
  workspace: workspace/output/template-one
```

`.env`：

```env
BLCLI_TERRAFORM_BILLING_ACCOUNT_ID=<真实 Billing ID>
BLCLI_TERRAFORM_ORGANIZATION_ID=0   # 个人账号保持 0
```

### 占位符可保留（仅 init/check）

- `domain: app.example.com` — **minimal 下只是 locals 占位**
- Grafana `root-url` — 用 port-forward 时可不改

### 真实 apply 前必须改

| 场景 | 必须修改 |
|------|----------|
| apply terraform | Billing ID |
| full + dns/cert | 真实 domain、DNS zone、证书 hostname |
| full + grafana 对外 URL | `root-url`、`hosted-domain`（若启用 Google OAuth） |

### 勿做

- 不要用 `/etc/hosts` 代替 full profile 的 DNS 校验
- 不要对 placeholder 域名执行 cert 组件 apply
- 不要假设 `config.yaml` 里列出的所有组件都在默认 args 中（以生成的 `args.yaml` 为准）

## 4. 推荐工作流

### 阶段 A：零云资源（AI 代码生成）

```bash
blcli init-args -r ./bl-template-one --org my-dev -o workspace/config/args.yaml
# 编辑 workspace + Billing 占位
blcli init ./bl-template-one -a workspace/config/args.yaml -o workspace/output/template-one -w
blcli check kubernetes -d workspace/output/template-one/kubernetes -r ./bl-template-one
blcli check repo --args workspace/config/args.yaml --project app
```

### 阶段 B：最小上云（minimal profile）

```bash
blcli apply init -d workspace/output/template-one/terraform/init --args workspace/config/args.yaml
blcli apply terraform -d workspace/output/template-one/terraform --args workspace/config/args.yaml --project app
gcloud container clusters get-credentials app-cluster --region us-west1 --project <project-id>
blcli apply kubernetes -d workspace/output/template-one/kubernetes --kubeconfig $KUBECONFIG
```

### 阶段 C：部署业务（无 ArgoCD）

```bash
kubectl create namespace my-app
kubectl apply -f my-app/
# 或 helm upgrade --install ...
```

可选：在 `gitops/default.yaml` 增加 `apps[]` 后 `blcli init gitops`，对生成目录 `kubectl apply`（仍无 ArgoCD）。

## 5. full profile：dns + cert 说明

`profiles/full/terraform.yaml` 增加：

1. **dns** — Cloud DNS 托管区（`app.example.com.`）
2. **cert** — Certificate Manager 证书 + map entries
3. **outputs** — 输出 certificate map 供 Gateway/Ingress 使用

**前置条件：**

1. 你拥有该域名，并能修改 registrar / NS 记录
2. 将 `app.example.com` 全部替换为你的域名
3. `terraform apply` 后按 outputs 提示添加 DNS 验证记录
4. 证书 PROVISIONING 完成后再挂到负载均衡

**minimal 不需要任何域名**即可 apply vpc/gke（仍要 Billing）。

## 6. Kubernetes 组件说明

| 组件 | minimal | full | 说明 |
|------|---------|------|------|
| sealed-secret | ✓ | ✓ | Git 内存加密 Secret；个人 dev 默认密钥方案 |
| external-secrets | — | — | config 中可选，**不在默认 profile** |
| istio | — | ✓ | full 监控栈依赖 |
| victoria-metrics-* | — | ✓ | 指标与告警 |
| grafana | — | ✓ | VM 数据源；Loki 默认关闭 |

`kubernetes/init.sh` 为 **no-op**（不再生成 ArgoCD SSH key）。

## 7. 与 blcli v2 Agent 配合

```bash
blcli contract --format json
blcli explain -r ./bl-template-one -m terraform -l
blcli explain -r ./bl-template-one -m kubernetes -c gke
blcli init-args ... --profile minimal   # 或 full
# 失败时
blcli diagnose --file apply.log --format json
blcli runs list --status failed --format json
```

## 8. config.yaml vs default.yaml

- **`config.yaml`**：模板仓库**能**提供的全部组件（含 cdn、loki、atlantis 等）
- **`default.yaml` + profile**：**默认生成**进 args 的组件子集
- AI 应只修改生成后的 `args.yaml` 中的 `components` 列表来增删组件

## 9. 成本提示

minimal 默认 GKE node pool：`e2-small`，1–2 节点。full profile 叠加监控与 Istio 会增加资源占用；开发环境可先在 minimal 上跑通再切 full。

---

更多 profile 细节见 [profiles/README.md](./profiles/README.md)。
