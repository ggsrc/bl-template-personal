# Terraform Project Configuration Guide

## Required Components and When `main` Is Needed

- **No component is marked as "required" in config.yaml**; each project's `components` list in default.yaml determines what is included.
- **main**: provides `locals` (project_id, region, project_name, domain, vpc_name). Any project using **vpc** or **cert** should include **main**, because cert templates use `local.project_id`, and main defines `vpc_name = google_compute_network.main.name` for other resources to reference.
- **provider, variables**: typically required for any project that runs Terraform (any component producing .tf files), because they define variables such as project_id, region and zone, and the provider setup.

## Recommended Components for the Single-Project Model

| Project | Purpose | Recommended components |
|------|------|----------|
| **app** | All-in-one environment under a personal account (single cluster + workloads) | main, backend, config-gcs-tfbackend, provider, variables, vpc, ip, firewall, gke, dns, cert, outputs |

## GKE Baseline: NetworkPolicy, Control Plane and Secure Boot

- `datapathProvider` (default `ADVANCED_DATAPATH`) selects the datapath. Dataplane V2 enforces `NetworkPolicy` natively; `LEGACY_DATAPATH` keeps a pre-existing legacy datapath, where NetworkPolicy needs `networkPolicyEnabled: true` (Calico). Both `datapathProvider` and `enableSecureBoot` (default `true`) are ForceNew in google provider 5.x: the first replaces the cluster, the second replaces the node pools. This template's `default.yaml` sets `deletionProtection: false`, so a replacement plan deletes the cluster — set `datapathProvider: LEGACY_DATAPATH` or `deletionProtection: true` before applying to an existing install.
- `masterAuthorizedNetworks` (`[{cidrBlock, displayName}]`) restricts the public control-plane endpoint. Empty (default) renders no `master_authorized_networks_config` block, so the control plane is reachable from any IP. When set, the module renders the allowlist and disables Google public CIDR access; node IPs stay allowed. In-place change.
- `masterIpv4CidrBlock` is declared but not applied: the module has no `private_cluster_config`. It only becomes meaningful together with a private control-plane block, which would also recreate the cluster.

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

## DNSSEC for Public Zones

- The `dns` component turns DNSSEC on (`dnssec_config { state = "on" }`) for every zone with `visibility: public` that has no `dnssec_config`. Private zones get none. To opt out, set `dnssec_config.state: "off"` on the zone.
- Cloud DNS signs the zone right away, but resolvers only validate the signatures once a DS record for the zone exists in the parent zone. Until then the zone resolves as before, without DNSSEC protection.
- Publish the DS record after `terraform apply`:
  1. Find the key-signing key: `gcloud dns dns-keys list --zone=ZONE_NAME --project=PROJECT_ID` (type `keySigning`).
  2. Print its DS record: `gcloud dns dns-keys describe KEY_ID --zone=ZONE_NAME --project=PROJECT_ID --format='value(ds_record())'`.
  3. Add that DS record where the zone is delegated: at the domain registrar for a registered domain (for example `example.com.`), or as a `DS` record set in the parent zone for a delegated subdomain (for example `prd.example.com.` inside `example.com.`).
- To turn DNSSEC off later, first remove the DS record from the registrar or parent zone and wait until its TTL has expired, then set the state to `off`. Turning it off while the DS record is still published makes validating resolvers fail for the whole zone.

## TLS Policy for the CDN HTTPS Proxy

- The `cdn` component attaches an SSL policy to each HTTPS proxy: profile `MODERN`, minimum TLS 1.2. Without a policy Google Cloud load balancers accept TLS 1.0 (`COMPATIBLE` profile).
- `MODERN` keeps the ECDHE cipher suites used by current browsers and TLS 1.2 clients. Set `ssl_policy_profile: RESTRICTED` to also drop the CBC suites, or `ssl_min_tls_version: TLS_1_3` (requires `RESTRICTED`) if every client supports TLS 1.3.
