# bl-template-one Profiles

This directory defines **optional overlays** merged on top of module `default.yaml` files when you run:

```bash
blcli init-args -r /path/to/bl-template-one --profile <name> -o args.yaml
```

## Available profiles

| Profile | Command | Terraform | Kubernetes | When to use |
|---------|---------|-----------|------------|-------------|
| **minimal** | `--profile minimal` (default) | VPC + GKE + backend; **no dns/cert** | `sealed-secret` only | Personal dev, AI codegen, lowest cost, no real domain |
| **full** | `--profile full` | minimal + **dns + cert** | minimal + **istio + VM + grafana** | Production-like stack; **requires real domain** for TLS |

Overlays live under `profiles/<name>/`:

```
profiles/
  full/
    terraform.yaml   # adds dns, cert; extends outputs for certificate maps
    kubernetes.yaml  # adds istio, victoria-metrics-*, grafana
```

## AI agent checklist

1. **Always read** [PERSONAL_DEV.md](../PERSONAL_DEV.md) first.
2. Default to `--profile minimal` unless the user explicitly owns a domain and wants HTTPS on GCP.
3. After `init-args`, user must set in generated args:
   - `global.workspace`
   - `BLCLI_TERRAFORM_BILLING_ACCOUNT_ID` in `.env` (before terraform apply)
4. **Do not** enable `profiles/full` dns/cert with placeholder domains like `app.example.com` for real `terraform apply`.
5. Deploy applications with `kubectl` / `helm`, not ArgoCD (removed from template-one).
6. Use `blcli explain -r <repo> -m terraform -l` to list components; only components listed in generated `args.yaml` are rendered.

## dns + cert (full profile only)

See [profiles/full/terraform.yaml](./full/terraform.yaml) and [PERSONAL_DEV.md](../PERSONAL_DEV.md#full-profile-dns--cert).

Requirements for successful apply:

- A **real DNS zone** you control (or Cloud DNS + domain delegation)
- Update `terraform.projects[].global.domain` and cert/dns parameters to match
- Certificate Manager DNS authorization records applied at your registrar
- **Not** satisfiable with `/etc/hosts` alone

## GitOps

`gitops/default.yaml` is empty. Optional app manifests: add `apps[]` to args or `gitops/profiles/` in future; apply with `kubectl`, not ArgoCD.
