# Terraform Project Configuration Guide

## Required Components and When `main` Is Needed

- **No component is marked as "required" in config.yaml**; each project's `components` list in default.yaml determines what is included.
- **main**: provides `locals` (project_id, region, project_name, domain, vpc_name). Any project using **vpc** or **cert** should include **main**, because cert templates use `local.project_id`, and main defines `vpc_name = google_compute_network.main.name` for other resources to reference.
- **provider, variables**: typically required for any project that runs Terraform (any component producing .tf files), because they define variables such as project_id, region and zone, and the provider setup.

## Recommended Components for the Single-Project Model

| Project | Purpose | Recommended components |
|------|------|----------|
| **app** | All-in-one environment under a personal account (single cluster + workloads) | main, backend, config-gcs-tfbackend, provider, variables, vpc, ip, firewall, gke, dns, cert, outputs |

## Backend

- **Full backend block**: if a project directory should use the GCS backend, include the **backend** component. It renders `backend.tf` with a full `terraform { backend "gcs" { ... } }` block, so `terraform init` and `terraform apply` can run directly.
- **backend-config fragment**: the **config-gcs-tfbackend** component only generates the `bucket` and `prefix` lines, intended for `terraform init -backend-config=config.gcs.tfbackend`. In this mode you still need to provide a complete backend declaration in some other way (for example a separately maintained `backend.tf` or CLI options). If you want each project directory to be self-contained and ready for `terraform init`, use the **backend** component.

## Multi-subnet (subnetworks) Naming Convention

- When using the `subnetworks` array of the **vpc** component, each subnet `name` must be a valid Terraform/HCL resource identifier.
- Prefer lowercase letters, digits and underscores only (for example `us_west1`) and avoid hyphens. If you use hyphens, sanitize `name` in the template or at the call site (for example replace them with underscores) before using it as a Terraform resource name; otherwise parsing errors may occur.

## OrganizationID and org_id

- **OrganizationID** (`terraform.global.OrganizationID` / args): when set to `"0"` or `0`, init templates do not render any `org_id`-related content.
- **Effect**: `variable "gcp_common"` has no `org_id` field or default, and resources such as `google_project` have no `org_id` line. This suits setups without an Organization, or with a Billing Account only.
- **Parameters**: OrganizationID is optional in `terraform/args.yaml` and `terraform/init/args.yaml`, with a default of `"0"`.

## Avoid Duplicate Outputs and Locals

- **Gateway IP and similar outputs**: output only through the `output_addresses` of the **ip** component; the **outputs** component no longer includes `gateway_ips`, to avoid duplicating ip.tf.
- **vpc_name**: defined only in **main.tf** via `locals { vpc_name = google_compute_network.main.name }`; **vpc.tf** no longer defines locals and uses the `vpc_name` parameter directly for the network name.
