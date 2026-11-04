# bl-template-one Personal Developer Guide (with AI Agents)

This guide is for **a personal GCP account + AI-assisted development**. It covers template-one's default path, choosing a profile, the parameters you must change, and common pitfalls.

## 1. Design goals

| Goal | Approach |
|------|------|
| Low configuration burden | `minimal` profile by default; no ArgoCD, no Dex OAuth |
| Gradual complexity | Switch to `--profile full` when you need a domain, TLS or monitoring |
| AI-friendly | Self-describing `args.yaml` + `blcli explain` + `profiles/README.md` |
| Application deployment | **kubectl / helm** (no GitOps CD) |

## 2. Choosing a profile

```bash
# Default: personal experiments / generation checks / no real domain
blcli init-args -r ./bl-template-one --org my-dev -o args.yaml

# Equivalent to
blcli init-args -r ./bl-template-one --profile minimal --org my-dev -o args.yaml

# Cloud DNS + certificates + monitoring stack
blcli init-args -r ./bl-template-one --profile full --org my-dev -o args.yaml

# Examples of optional data/observability components (cnpg, redis, loki, otel)
blcli init-args -r ./bl-template-one --profile optional --org my-dev -o args.yaml
# Or, after generating args, add components to kubernetes.projects[].components[] by hand
```

### minimal (default)

**Terraform components:** main, backend, provider, variables, vpc, ip, firewall, gke, outputs  
**Not included:** dns, cert  
**Kubernetes components:** sealed-secret  
**Good for:** Phase 3–4 local generation and checks, a minimal GCP bill, AI iteration on infra code

### full

**Adds** on top of minimal (see `profiles/full/`):

**Terraform adds:** dns, cert, and extends outputs with certificate_maps  
**Kubernetes adds:** istio, victoria-metrics-operator, victoria-metrics, grafana  

**Good for:** you already own a real domain and need GCP-managed certificates and observability

### optional

Provides **examples of optional components** on top of minimal / full (see `profiles/optional/kubernetes.yaml`):

- cnpg, redis, loki, otel-collector, etc.
- Full list in [kubernetes/OPTIONAL_COMPONENTS.md](./kubernetes/OPTIONAL_COMPONENTS.md)

**Good for:** when you need a PostgreSQL operator, shared Redis, a logging stack, etc. You can also skip the profile and add any component registered in `config.yaml` directly to args

## 3. AI agent checklist: must change / do not change

### Must change after init-args

```yaml
global:
  GlobalName: my-dev          # Same as --org
  workspace: workspace/output/template-one
```

`.env`:

```env
BLCLI_TERRAFORM_BILLING_ACCOUNT_ID=<real Billing ID>
BLCLI_TERRAFORM_ORGANIZATION_ID=0   # Keep 0 for a personal account
```

### Placeholders you can keep (init/check only)

- `domain: app.example.com` — **only a locals placeholder in minimal**
- Grafana `root-url` — can stay as is if you use port-forward

### Must change before a real apply

| Scenario | Must change |
|------|----------|
| apply terraform | Billing ID |
| full + dns/cert | Real domain, DNS zone, certificate hostnames |
| full + public Grafana URL | `root-url`, `hosted-domain` (if Google OAuth is enabled) |

### Do not

- Do not use `/etc/hosts` instead of the DNS validation in the full profile
- Do not apply the cert component with placeholder domains
- Do not assume every component listed in `config.yaml` is in the default args (the generated `args.yaml` is the source of truth)

## 4. Recommended workflow

### Stage A: no cloud resources (AI code generation)

```bash
blcli init-args -r ./bl-template-one --org my-dev -o workspace/config/args.yaml
# Edit workspace + Billing placeholder
blcli init ./bl-template-one -a workspace/config/args.yaml -o workspace/output/template-one -w
blcli check kubernetes -d workspace/output/template-one/kubernetes -r ./bl-template-one
blcli check repo --args workspace/config/args.yaml --project app
```

### Stage B: minimal cloud footprint (minimal profile)

```bash
blcli apply init -d workspace/output/template-one/terraform/init --args workspace/config/args.yaml
blcli apply terraform -d workspace/output/template-one/terraform --args workspace/config/args.yaml --project app
gcloud container clusters get-credentials app-cluster --region us-west1 --project <project-id>
blcli apply kubernetes -d workspace/output/template-one/kubernetes --kubeconfig $KUBECONFIG
```

### Stage C: deploy workloads (no ArgoCD)

```bash
kubectl create namespace my-app
kubectl apply -f my-app/
# Or helm upgrade --install ...
```

Optional: add `apps[]` to `gitops/default.yaml`, run `blcli init gitops`, then `kubectl apply` the generated directory (still no ArgoCD).

## 5. full profile: dns + cert

`profiles/full/terraform.yaml` adds:

1. **dns** — Cloud DNS managed zone (`app.example.com.`)
2. **cert** — Certificate Manager certificate + map entries
3. **outputs** — outputs the certificate map for the Gateway/Ingress

**Prerequisites:**

1. You own the domain and can change its registrar / NS records
2. Replace every `app.example.com` with your domain
3. After `terraform apply`, add the DNS validation records shown in the outputs
4. Attach the certificate to the load balancer only after it finishes PROVISIONING

**minimal needs no domain at all** to apply vpc/gke (Billing is still required).

## 6. Kubernetes components

| Component | minimal | full | optional (opt-in) | Notes |
|------|---------|------|-------------------|------|
| sealed-secret | ✓ | ✓ | — | Encrypted Secrets stored in Git |
| external-secrets* | — | Common | — | Sync from GCP Secret Manager |
| istio | — | ✓ | — | Required by the full monitoring stack |
| victoria-metrics-* | — | ✓ | — | Metrics and alerting |
| grafana | — | ✓ | — | VM data source |
| cnpg / redis / paradedb / kafka | — | — | ✓ | Data layer, official images |
| juicefs / loki / otel-collector | — | — | ✓ | Storage and observability |
| uptrace / n9e / bytebase | — | — | ✓ | APM / monitoring / DB changes |

\* Registered in `config.yaml`; whether it is rendered depends on the `components` list in `args.yaml`.

See [kubernetes/OPTIONAL_COMPONENTS.md](./kubernetes/OPTIONAL_COMPONENTS.md) for how to enable optional components and their dependencies.

`kubernetes/init.sh` is a **no-op** (it no longer generates an ArgoCD SSH key).

## 7. Working with blcli v2 agents

```bash
blcli contract --format json
blcli explain -r ./bl-template-one -m terraform -l
blcli explain -r ./bl-template-one -m kubernetes -c gke
blcli init-args ... --profile minimal   # or full
# On failure
blcli diagnose --file apply.log --format json
blcli runs list --status failed --format json
```

## 8. config.yaml vs default.yaml

- **`config.yaml`**: every component the template repository **can** provide (including optional ones such as cnpg, redis, loki)
- **`default.yaml` + profile**: the subset of components **generated** into args by default (minimal / full / optional examples)
- **Optional components**: see [kubernetes/OPTIONAL_COMPONENTS.md](./kubernetes/OPTIONAL_COMPONENTS.md)
- AI agents should add or remove components only by editing the `components` list in the generated `args.yaml`

## 9. Cost notes

The default minimal GKE node pool is `e2-small` with 1–2 nodes. The full profile adds monitoring and Istio, which use more resources; for development, get minimal working first and then switch to full.

---

More profile details in [profiles/README.md](./profiles/README.md).
