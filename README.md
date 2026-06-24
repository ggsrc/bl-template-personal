# bl-template

blcli template repository for one-click generation of GCP environment infrastructure configurations.

## Repository Purpose

This is a template repository for the `blcli` tool, used to quickly generate a personal-account-friendly GCP infrastructure baseline. The default model is one project + one cluster, and through templating it can quickly create:

- **Terraform Configurations**: Infrastructure as code configurations for GCP resources
- **Kubernetes Configurations**: Cluster platform components (Istio, monitoring, secrets, etc.)
- **GitOps Templates** (optional): Kubernetes manifest templates for apps; default stack does **not** install ArgoCD

**Personal developer defaults:** single GCP project + single GKE cluster, deploy apps with `kubectl` / `helm`. ArgoCD is intentionally omitted from the default Kubernetes stack.

## Design Architecture

### Core Design Principles

1. **Templating**: All configuration files use Go Template syntax with parameterization support
2. **Self-describing**: Parameter self-description through `config.yaml` and `args.yaml`
3. **Modular**: Components are categorized by function, supporting dependency management and independent deployment
4. **Convention over Configuration**: Follow the principle of convention over configuration to reduce explicit configuration

### Directory Structure

```
bl-template/
├── terraform/          # Terraform infrastructure configurations
│   ├── config.yaml     # Terraform component configuration definitions
│   ├── init/           # GCP Organization level initialization
│   ├── modules/         # Reusable Terraform modules
│   ├── project/        # Project-level Terraform configurations
│   └── projects/       # Project deployment configurations
├── kubernetes/         # Kubernetes configurations
│   ├── config.yaml     # Kubernetes component configuration definitions
│   ├── base/           # Base components (required)
│   └── optional/        # Optional components
├── gitops/             # Optional app manifest templates (disabled by default)
│   ├── config.yaml     # deployment/statefulset base templates
│   ├── args.yaml       # Parameter definitions for app manifests
│   ├── default.yaml    # Empty by default; add apps[] to enable generation
│   └── base-*.tmpl     # deployment/service/configmap 等基础模板
└── README.md           # 本文件
```

## Core Components

### 1. config.yaml

The `config.yaml` in each directory describes the template components provided by that directory and their purposes.

#### terraform/config.yaml

Defines three main sections:

- **init**: GCP initialization for a lightweight single-project setup
  - `projects`: Create one GCP project and enable required services
  - Keep `OrganizationID` as `"0"` to omit `org_id` in generated variables and resources (see [terraform/TERRAFORM_PROJECT.md](terraform/TERRAFORM_PROJECT.md)).
  
- **modules**: Reusable Terraform modules (only for template generation)
  - `gke`: GKE cluster module
  - `gke-node-pool`: GKE node pool module
  - `vm-server`: VM server module
  - `ssl-cert`: SSL certificate module
  - `tailscale-node`: Tailscale node module
  - `tailscale-exit-node`: Tailscale exit node module
  - `tailscale-subnet-router`: Tailscale subnet router module
  - `gke-sm-accessor-sa`: GKE Secret Manager accessor service account module
  - `security-policy-corp-ip-whitelist`: Security policy corporate IP whitelist module

- **projects**: Project-level deployment configurations (includes actual deployment dependency order)
  - `main`: Main configuration file
  - `modules`: Module usage examples
  - `outputs`: Output definitions
  - `gke`: GKE deployment configuration
  - `backend`: Terraform backend configuration
  - `variables`: Variable definitions
  - `provider`: Provider configuration

#### kubernetes/config.yaml

Default Kubernetes platform stack (personal / solo developer):

- `external-secrets-operator` + `external-secrets`: GCP Secret Manager integration
- `sealed-secret`: encrypted secrets in Git
- `istio`: service mesh
- `victoria-metrics-operator` + `victoria-metrics` + `grafana`: monitoring

**Not included by default:** ArgoCD (use `kubectl apply` / `helm` for apps).

#### gitops/config.yaml

Optional **application manifest** templates (Deployment, Service, HPA, Istio VirtualService, etc.).

- **Default:** `gitops/default.yaml` has no apps; `blcli init` skips GitOps output.
- **Enable:** add `apps[]` (and `argocd.project` if using blcli gitops generation) to `gitops/default.yaml` or your `args.yaml`.
- Generated manifests can be applied directly with `kubectl`; no ArgoCD required.

### 2. args.yaml

`args.yaml` is used to describe template parameters, implementing parameter self-description functionality.

#### Discovery Mechanism

1. **Convention First**: CLI automatically infers based on `path` in `config.yaml`
   - If `path` is a directory → look for `{path}/args.yaml`
   - If `path` is a file → look for `{dirname(path)}/args.yaml`

2. **Explicit Specification**: Optional `args` field in `config.yaml` to explicitly specify the path

3. **Hierarchical Lookup**: Supports upward lookup of parent directory's `args.yaml` for parameter inheritance

#### Structure Example

```yaml
version: 1.0.0

parameters:
  # Global parameters
  global:
    OrganizationID:
      type: string
      description: "GCP Organization ID"
      required: true
      example: "123456789012"
  
  # Component-level parameters
  components:
    gke:
      project_id:
        type: string
        description: "GCP Project ID"
        required: true
      cluster_name:
        type: string
        description: "Name of the GKE cluster"
        required: true
        pattern: "^[a-z0-9-]+$"
```

#### Parameter Validation

Parameters support **validation rules** that blcli enforces during `blcli init` (before writing any files):

- **validation** (list): Each rule is a map with `kind` and kind-specific params. Supported kinds: `required`, `stringLength`, `pattern`, `format`, `enum`, `numberRange`.
- **validation.unique** (top-level): Ensures uniqueness at a path (e.g. `terraform.projects[].name`).

Example:

```yaml
parameters:
  global:
    ProjectName:
      type: string
      required: true
      validation:
        - kind: required
        - kind: stringLength
          min: 6
          max: 30
          message: "GCP project ID: 6-30 characters"
        - kind: pattern
          value: "^[a-z][a-z0-9-]{4,28}[a-z0-9]$"

# Top-level: ensure project names are unique
validation:
  unique:
    - path: "terraform.projects[].name"
      message: "Project names must be unique"
```

See [ARGS_DESIGN.md](./ARGS_DESIGN.md) for full parameter and validation documentation.

For detailed design, please refer to [ARGS_DESIGN.md](./ARGS_DESIGN.md)

## blcli 用法说明

**个人开发者 / AI Agent：** 请先阅读 [PERSONAL_DEV.md](./PERSONAL_DEV.md) 与 [profiles/README.md](./profiles/README.md)。

本仓库作为 blcli 的模板仓库，通过 `-r` 指定本地路径或 GitHub 地址使用。

### Profile（推荐）

| Profile | 命令 | 说明 |
|---------|------|------|
| `minimal` | `blcli init-args -r . --org my-dev`（默认） | 无 dns/cert；K8s 仅 sealed-secret |
| `full` | `blcli init-args -r . --profile full --org my-dev` | 增加 dns+cert 与 istio/监控栈；**需真实域名** |

### 1. 生成参数文件（init-args）

从模板仓库收集各层 `args.yaml` 定义，生成一份可编辑的 `args.yaml`：

```bash
blcli init-args -r github.com/NFTGalaxy/bl-template -o args.yaml
# 或使用本地路径
blcli init-args -r /path/to/bl-template -o args.yaml
```

生成的 `args.yaml` 包含 `global`、`terraform`、`kubernetes` 等段；默认**不含** GitOps apps（除非在 `gitops/default.yaml` 中自行添加）。

### 2. 生成基础设施配置（init）

根据 `args.yaml` 和模板生成 Terraform、Kubernetes 配置（GitOps 仅在 args 含 `gitops.apps` 时生成）：

```bash
# 生成 terraform + kubernetes（默认）
blcli init -r /path/to/bl-template-one -a args.yaml

# 只生成 terraform
blcli init terraform -r /path/to/bl-template-one -a args.yaml

# 只生成 kubernetes
blcli init kubernetes -r /path/to/bl-template-one -a args.yaml

# 生成时指定输出目录与覆盖
blcli init -r /path/to/bl-template-one -a args.yaml --output ./workspace/output -w
```

- **Terraform**：输出到 `{workspace}/terraform/`（init、gcp 项目、modules 等）。
- **Kubernetes**：按 `kubernetes.projects[]` 与 `components` 输出到 `{workspace}/kubernetes/{project}/{component}/`。
- **GitOps**（可选）：在 `args.yaml` 配置 `gitops.apps[]` 后，输出 deployment/service 等 manifest 到 `{workspace}/gitops/{project}/{app}/`；用 `kubectl apply` 部署，无需 ArgoCD。

### 3. 部署应用（个人开发者推荐）

```bash
# 平台组件
blcli apply kubernetes -d ./workspace/output/kubernetes

# 业务应用：直接 kubectl / helm，或 apply 生成的 gitops manifest
kubectl apply -f ./workspace/output/gitops/app/my-app/
```

### 4. 初始化仓库并推送到 GitHub（apply init-repos）

对 `blcli init` 生成的 terraform、kubernetes、gitops 三个目录分别执行 git init、创建 GitHub 仓库、提交并推送（需在提示时输入 Y 确认）：

```bash
blcli apply init-repos -o myorg -d ./workspace/output
```

需要已安装并登录 [gh](https://cli.github.com/)（`gh auth login`）。

## Template Syntax

All template files use Go Template syntax, supporting:

- Variable substitution: `{{ .ProjectName }}`
- Conditional statements: `{{ if .Condition }}...{{ end }}`
- Loops: `{{ range .Items }}...{{ end }}`
- Function calls: `{{ .Function | format }}`

Example:

```hcl
# terraform/project/main.tf.tmpl
resource "google_compute_instance" "example" {
  name         = "{{ .ProjectName }}-instance"
  machine_type = "e2-medium"
  zone         = var.zone
}
```

## Dependency Management

The `dependencies` field in `config.yaml` defines dependency relationships between components:

```yaml
projects:
  - name: gke
    dependencies:
      - vpc  # gke depends on vpc
```

The CLI will automatically sort based on dependency relationships, ensuring dependent components are deployed before components that depend on them.

## Extension Guide

### Adding New Modules

1. Create module directory under `terraform/modules/`
2. Add module definition in the `modules` section of `terraform/config.yaml`
3. Create `args.yaml` to describe module parameters (optional)

### Adding New Components

1. Create component template file (`.tmpl`)
2. Add component definition in the corresponding `config.yaml`
3. Add parameter definitions in `args.yaml` (if needed)

### Modifying Parameter Definitions

Edit the `args.yaml` file in the corresponding directory to add or modify parameter definitions.

## Version Management

- Both `config.yaml` and `args.yaml` contain `version` fields for version control
- Template files use semantic versioning

## Contributing Guide

1. Follow existing directory structure and naming conventions
2. Add complete `args.yaml` definitions for new components
3. Update component list in `config.yaml`
4. Add necessary dependency relationships
5. Provide clear parameter descriptions and examples

## Related Documentation

- [args.yaml Design Document](./ARGS_DESIGN.md): Detailed parameter self-description design document
