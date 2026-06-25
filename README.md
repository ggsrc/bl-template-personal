<div align="center">

# bl-template-personal

**Personal GCP platform template for [blcli](https://github.com/ggsrc/blcli)** — one project, one cluster, minimal config, optional full observability stack.

*Default: deploy apps with `kubectl` / `helm` — ArgoCD is not required.*

[![GitHub stars](https://img.shields.io/github/stars/ggsrc/bl-template-personal?style=flat-square)](https://github.com/ggsrc/bl-template-personal/stargazers)
[![GitHub forks](https://img.shields.io/github/forks/ggsrc/bl-template-personal?style=flat-square)](https://github.com/ggsrc/bl-template-personal/network/members)
[![blcli](https://img.shields.io/badge/powered%20by-blcli-blue?style=flat-square)](https://github.com/ggsrc/blcli)
[![Enterprise template](https://img.shields.io/badge/enterprise-bl--template-orange?style=flat-square)](https://github.com/ggsrc/bl-template)

[Quick Start](#blcli-usage) · [中文说明](README_zh.md) · [PERSONAL_DEV.md](PERSONAL_DEV.md) · [Profiles](profiles/README.md)

</div>

<!-- ADOPTION:START -->
**Adoption snapshot:** tracked via GitHub stars and forks · [Star this repo](https://github.com/ggsrc/bl-template-personal) when you use it with blcli
<!-- ADOPTION:END -->

---

## What is bl-template-personal?

Lightweight **personal / solo-developer** template for [blcli](https://github.com/ggsrc/blcli). Generates a single GCP project + GKE cluster baseline with optional DNS, certs, Istio, and monitoring — without the multi-env Org complexity of [bl-template](https://github.com/ggsrc/bl-template).

```
bl-template-personal  +  args.yaml  +  blcli  →  one cluster  →  kubectl / helm for apps
```

Ideal for labs, AI-assisted infra iteration, and low-billing GCP experiments.

---

## Who is it for?

| Audience | Profile | Typical use |
|----------|---------|-------------|
| Solo developers | `minimal` (default) | Generate & check locally; smallest GCP bill |
| Developers with a real domain | `full` | DNS + managed certs + Istio + Victoria Metrics + Grafana |
| Enterprise multi-env teams | — | Use [bl-template](https://github.com/ggsrc/bl-template) instead |

See [PERSONAL_DEV.md](./PERSONAL_DEV.md) for AI agent workflows and required parameter changes.

---

## Why this template?

- **Low config burden** — `minimal` profile: core Terraform + sealed-secrets only; add complexity when you need it.
- **Progressive profiles** — `minimal` → `full` without changing blcli or the template protocol.
- **No ArgoCD by default** — platform via `blcli apply kubernetes`; apps via `kubectl` / `helm` (GitOps manifests optional).
- **AI-friendly** — self-describing `args.yaml`, `blcli explain`, and documented pitfalls in [PERSONAL_DEV.md](./PERSONAL_DEV.md).

| bl-template (enterprise) | bl-template-personal |
|---|---|
| Multi-project corp / stg / prd | Single project + cluster |
| ArgoCD GitOps included | ArgoCD omitted by default |
| Org-level GCP setup | Personal account (`OrganizationID: "0"`) |

---

## Who uses bl-template-personal?

We are collecting adopters. If you use this template in your lab or product, [open a PR](https://github.com/ggsrc/bl-template-personal/edit/main/README.md) to add your name or org below. The list is mirrored to [README_zh.md](README_zh.md) automatically.

<!-- ADOPTERS:START -->
<!-- Example:
- [Your Name](https://example.com) — solo GCP lab on blcli
-->
<!-- ADOPTERS:END -->

---

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

## blcli Usage

**Personal developers / AI agents:** see [PERSONAL_DEV.md](./PERSONAL_DEV.md) and [profiles/README.md](./profiles/README.md).

Use this repo as a blcli template via `-r` (local path or GitHub URL).

### Profiles (recommended)

| Profile | Command | Description |
|---------|---------|-------------|
| `minimal` | `blcli init-args -r github.com/ggsrc/bl-template-personal --org my-dev` (default) | No dns/cert; K8s sealed-secret only |
| `full` | `blcli init-args -r github.com/ggsrc/bl-template-personal --profile full --org my-dev` | Adds dns+cert and Istio/monitoring; **requires a real domain** |

### 1. Generate parameter file (`init-args`)

```bash
blcli init-args -r github.com/ggsrc/bl-template-personal -o args.yaml
blcli init-args -r /path/to/bl-template-personal -o args.yaml
```

Generated `args.yaml` includes `global`, `terraform`, `kubernetes`, etc. GitOps apps are omitted unless added in `gitops/default.yaml`.

### 2. Generate infrastructure configuration (`init`)

```bash
blcli init -r github.com/ggsrc/bl-template-personal -a args.yaml
blcli init terraform -r github.com/ggsrc/bl-template-personal -a args.yaml
blcli init kubernetes -r github.com/ggsrc/bl-template-personal -a args.yaml
blcli init -r /path/to/bl-template-personal -a args.yaml --output ./workspace/output -w
```

- **Terraform**: `{workspace}/terraform/`
- **Kubernetes**: `{workspace}/kubernetes/{project}/{component}/`
- **GitOps** (optional): manifests under `{workspace}/gitops/` when `gitops.apps[]` is set; deploy with `kubectl`

### 3. Deploy applications (personal workflow)

```bash
blcli apply kubernetes -d ./workspace/output/kubernetes
kubectl apply -f ./workspace/output/gitops/app/my-app/
```

### 4. Initialize and push repositories (`apply init-repos`)

```bash
blcli apply init-repos -o myorg -d ./workspace/output
```

Requires [gh](https://cli.github.com/) with `gh auth login`.

中文步骤摘要见 [README_zh.md](./README_zh.md)。

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
