# Grafana

## Prerequisites

- **helm** and **kubectl** must be in PATH. The install script checks for them before running.
- Optional: set `minHelmVersion` / `minKubectlVersion` in component args to enforce minimum versions.

## Install

After `blcli init`, run:

```bash
bash install
```

Uses `values.yaml` and rendered `values-override.yaml` (from `values-override.yaml.tmpl`).

Or manually:

```bash
helm dependency update
helm install grafana -n grafana . --values ./values.yaml --values ./values-override.yaml
```

`lokiDatasourceUrl` is optional; leave empty in args to skip the Loki datasource (template-one does not deploy Loki by default).
