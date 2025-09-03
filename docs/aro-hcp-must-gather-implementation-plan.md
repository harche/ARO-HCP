# ARO-HCP Must‑Gather Implementation Plan

## Problem Statement
When a customer’s ARO‑HCP cluster misbehaves (create/update/delete stuck, nodepools unhealthy, credentials issues, API unreachable, etc.), SREs need a fast, reliable, and privacy‑safe way to collect all relevant telemetry to triage issues. Direct access to service or management clusters is not allowed. Therefore, collection must flow through our centralized Azure Data Explorer (Kusto) telemetry using the Azure Kusto Go SDK.

## Goals
- Provide a repeatable, low‑friction collection flow for SREs and on‑call engineers.
- Capture the minimum yet sufficient data to debug common failure modes across RP (frontend/backend), Cluster Service, Maestro, Hypershift/HCP, and Azure infra.
- Use only approved, indirect access paths: query Kusto using `github.com/Azure/azure-kusto-go/azkustodata` with an AAD identity; do not access AKS clusters directly.
- Ensure safe handling/redaction of secrets, PII, tokens, and sensitive URLs.
- Produce a standard, versioned artifact structure and metadata manifest to enable automated post‑processing.

## Non‑Goals
- Replacing customer cluster must‑gather for OCP payload workloads.
- Persisting production data beyond short‑term debugging needs.
- Full DB dumps; only sanitized, minimal operational metadata is collected.

## Personas & Primary Use Cases
- SRE/On‑call: Urgent triage of a failing ARO‑HCP operation (create/update/delete, scaling, creds, reachability).
- Engineering: Deep‑dive analysis for regressions and systemic issues.
- Support: Guided data request flow with clear instructions for customers when needed.

## High‑Level Architecture
- CLI Wrapper: `hcpctl gather` (new) orchestrates collection via Azure ARM and Kusto.
- Kusto Collector: queries Azure Data Explorer using `azure-kusto-go` (azkustodata) to retrieve logs, events and metrics for the target time window and scope.
- Redactor: streaming filters; configurable policies.
- Packager: tarball; metadata manifest; checksum.

Execution Modes
- Local (connected): SRE runs `hcpctl gather` using current `az` login for AAD and a configured Kusto cluster/database.
- Automated Trigger (future): Backend/automation invokes the same Kusto queries using a managed identity.

Access model
- All resource state is gathered from telemetry exported to Kusto (container logs, kube events/inventory, and Prometheus metrics in Azure Monitor Logs).

## Scope of Data to Collect
1) Azure Control Plane (scoped to ARM resource / subscription / region)
- Resource details for the HcpOpenShiftClusters ARM resource and related child resources (nodepools, MI/role assignments, private endpoints, DNS, LBs, NSGs, RouteTables, VNets, Subnets, NICs, Public IPs, KeyVault refs).
- ARM operation history, activity logs, diagnostic settings, and relevant Insights/Monitor signals for a chosen time window.
- Relevant tags/identity assignments for tracing and access checks.

2) Service and Management Clusters via Kusto
- Container logs: RP frontend/backend, cluster‑service, maestro server/agent, secret‑sync, istio ingress, hypershift operator, and related controllers using `ContainerLogV2` (or your org’s container log table) filtered by namespaces and time.
- Kubernetes events and inventory: `KubeEvents`, `KubePodInventory`, `KubeNodeInventory`, `KubeServices`, `KubeReplicaSetInventory`, etc., for a time‑bounded window.
- Prometheus metrics (Azure Managed Prometheus written to Azure Monitor Workspaces): `PrometheusMetrics` table; filter by relevant namespaces (`aro-hcp`, `maestro`, hosted control plane namespaces like `ocm-.*`) and metric names.

3) RP State & Metadata (sanitized)
- From RP APIs/backends: operation state summaries, correlation IDs, timelines (without secrets), selected counts/IDs of objects tied to the cluster.
- If a DB exists, only pull minimal, whitelisted operational fields (IDs, status enums, timestamps) via read‑only queries or internal API; never raw secrets.

4) Networking & DNS
- Effective NSG rules for subnets/NICs, peering routes, private endpoints connection statuses.
- DNS zones/records used for API/ingress; resolution checks and traceroutes (if allowed).

5) Tracing & Correlation
- Correlation IDs from ARM requests and RP hops, request traces if available (OpenTelemetry/Azure Monitor exporting), and mapping to logs.

## Redaction & Privacy
- Default‑on redaction pipeline applied to logs and resource YAML/JSON:
  - Secrets data fields, token‑like strings (JWT, PAT), key material (PEM), connection strings, access keys, SAS tokens, internal hostnames/IPs when sensitive.
  - Configurable regex policies with allowlist for non‑sensitive fields.
- Strategy: redact → hash placeholder with stable prefix (e.g., `<redacted:sha1:...>`) for cross‑file correlation without exposing raw secrets.
- Redaction manifest: list of applied rules and counts per file.
- Tests: unit/regex fuzzing to avoid leaks; golden tests for typical inputs.

## CLI UX (`hcpctl gather`)
- Inputs
  - `--resource-id` or `--subscription/-s`, `--resource-group/-g`, `--name/-n` for the HcpOpenShiftClusters resource
  - `--since`, `--until`, or `--time-window` (e.g., `4h`)
  - `--kusto-endpoint`, `--kusto-db` (resolved from environment config by default)
  - `--skip-azure`, `--skip-kusto`
  - `--output` (path), `--upload-blob` (SAS URL) optional
  - `--max-rows`, `--concurrency`, `--timeout`
  - `--redact-policy` (default strong), `--no-redact` (requires confirmation)
- Behavior
  - Resolve mapping (resource → region → Kusto endpoint/database) via config.
  - Check `az account show` context; prompt if mismatched cloud/tenant.
  - Run Azure and Kusto collectors with bounded concurrency and per‑module timeouts.
  - Stream through redactor; write a normalized tarball plus `manifest.json` with versions, inputs, time window, and collection results.
- Output Structure
  - `manifest.json` (tool version, git commit, inputs, timings, redaction summary)
  - `azure/` (ARM resources, activity logs, role assignments)
  - `kusto/` (container logs, kube events/inventory, Prometheus metrics)
  - `timeline/` (operation summaries, correlation IDs)
  - `checks/` (basic validations: DNS resolves, endpoint health summaries when applicable)

 

## Implementation Plan (Phased)
1) Design & Scaffolding (Week 1)
- Create `tooling/must-gather/` with module layout (azure, kusto, redact, pack).
- Define artifact schema and `manifest.json` contract.
- Add container image `Dockerfile` and Makefile targets to build/push.

2) CLI Integration (Week 2)
- Add `hcpctl gather` command with flags and help.
- Implement mapping resolution using config (resource → region → Kusto endpoint/database).
- Wire module orchestration, concurrency, timeouts, and basic progress output.

3) Kusto Collector (Week 2–3)
- Build KQL queries for container logs, kube events/inventory, and Prometheus metrics.
- Add query parameterization by time window, namespaces, resource identifiers.
- Implement streaming export and pagination; cap rows and size.

4) Azure Collector (Week 3–4)
- Fetch ARM resource graph for target cluster and dependent resources (VNets, Subnets, NSGs, LBs, PEs, DNS, MI, KV refs, role assignments).
- Pull activity logs and ARM operation history for time window.
- Add network effective‑rules and connection status snapshots where API allows.

5) Redaction Engine (Week 4)
- Default policy and configs; stream processing for JSON/logs.
- Unit tests incl. fuzz; golden files for typical RP/service logs and events.

6) Packaging & Upload (Week 4)
- Create final tarball; compute checksum; optional upload to provided SAS.
- Include redaction manifest and module collection summaries (success/fail per step).

7) CI/CD & Release (Week 5)
- GitHub Actions to build and push the `aro-hcp-must-gather` image on merge.
- Version gating in `hcpctl gather` to ensure image/tooling compatibility.

8) Documentation & Runbooks (Week 5)
- `docs/must-gather.md`: usage, examples, output reference, redaction guarantees.
- On‑call runbook integration; troubleshooting matrix for common failures.

9) Pilot & Hardening (Week 6)
- Dry‑runs in `dev` environments; iterate on gaps.
- Load/size testing; ensure typical run < 10 minutes and artifact < 200MB by default window.

## Security, Compliance, and Access
- Azure: require least‑privileged role to read ARM resources and activity logs in target subscription/resource group.
- Kusto: require reader/query role on the ADX cluster/database (or Log Analytics workspace if queries federate to it). Use AAD (DefaultAzureCredential) or managed identity.
- No Kubernetes RBAC is required.
- Secrets: never export Secret data; redact all secret‑like values from logs/config.
- Data retention: artifacts are local unless explicitly uploaded; no backend persistence by default.

## Testing Strategy
- Unit tests for redaction and collectors’ serialization.
- Snapshot tests on sample query outputs for artifact structure.
- E2E in CI: run against a dev ADX with a narrow window, validate manifest and essential files exist.

## Risks & Mitigations
- Access fragmentation (dev vs public tenants): detect and print precise login instructions; support `--cloud` and tenant hints.
- Over‑collection and large artifacts: strict time windows, namespace allowlists, row caps, compression.
- Secret leakage: conservative redaction, tests, and optional `--no-redact` gated behind `--i-know-what-i-am-doing`.
- Mapping failures (resource → Kusto): allow manual `--kusto-endpoint/--kusto-db` overrides; log actionable guidance.

## Acceptance Criteria
- `hcpctl gather --resource-id <id> --time-window 4h --kusto-endpoint https://<cluster>.<region>.kusto.windows.net --kusto-db <db>` produces a tarball with `manifest.json`, `azure/`, `kusto/`, and `timeline/` directories.
- Redaction manifest present; no raw secrets in sampled golden tests.
- Documentation covers usage, scope, and privacy guarantees.

## Open Questions
- Which exact RP DB and tables are safe to query for operational metadata? Define a narrow whitelist with Sec/Privacy sign‑off.
- What is the authoritative mapping ARM resource → region → Kusto cluster/database and workspace table names? Document per‑cloud config.
- Preferred artifact upload target (internal blob, customer‑provided SAS, or ticket system attachment)?
- Are there environment‑specific restrictions (e.g., prod) requiring different identities or endpoints for Kusto?

---

# Kusto‑Based Collection Details

This section specifies how we gather signals via Kusto using `azure-kusto-go`.

## Collector Inputs (flags/env)
- `--kusto-endpoint`: Kusto cluster URI, e.g. `https://<cluster>.<region>.kusto.windows.net`.
- `--kusto-db`: Database name that stores ARO‑HCP telemetry.
- `--time-window`: Relative (e.g., `4h`) or absolute (`--start`, `--end`).
- `--subscription-id` and/or `--resource-id`: Used to scope queries (provide both where possible for precise filtering).
- `--namespaces`: Optional override for namespace allowlist (defaults cover `aro-hcp`, `maestro`, `aks-istio-ingress`, and `ocm-.*`).
- Authentication: DefaultAzureCredential by default; supports managed identity or service principal via standard Azure Identity environment variables.

## Client Initialization (Go)
Use the data client from `github.com/Azure/azure-kusto-go/azkustodata`.

```go
import (
    "context"
    azkustodata "github.com/Azure/azure-kusto-go/azkustodata"
    "github.com/Azure/azure-kusto-go/azkustodata/kql"
)

func newKustoClient(endpoint string) (*azkustodata.Client, error) {
    kcsb := azkustodata.NewConnectionStringBuilder(endpoint).WithDefaultAzureCredential()
    return azkustodata.New(kcsb)
}

func runQuery(ctx context.Context, c *azkustodata.Client, db string, q string) (*azkustodata.Dataset, error) {
    return c.Query(ctx, db, kql.New(q))
}
```

Notes
- Prefer `IterativeQuery` for large result sets; always `Close()` datasets.
- For management commands (rare here) use `Mgmt()`.

## KQL Query Patterns
Time scoping
```
let start = datetime_add('minute', -240, now());
let end = now();
```

Container logs (service and management clusters)
```
ContainerLogV2
| where TimeGenerated between (start .. end)
| where KubernetesNamespace matches regex @"^(aro-hcp|maestro|aks-istio-ingress|ocm-.*)$"
| where ContainerName has_any ("frontend", "backend", "cluster-service", "maestro", "secret-sync", "istio")
| project TimeGenerated, KubernetesNamespace, PodName, ContainerName, LogMessage
| order by TimeGenerated desc
```

Kubernetes events
```
KubeEvents
| where TimeGenerated between (start .. end)
| where Namespace matches regex @"^(aro-hcp|maestro|ocm-.*)$"
| project TimeGenerated, Namespace, Reason, Message, InvolvedObjectKind, InvolvedObjectName, Source
| order by TimeGenerated desc
```

Pod inventory (to correlate rollouts / restarts)
```
KubePodInventory
| where TimeGenerated between (start .. end)
| where Namespace matches regex @"^(aro-hcp|maestro|ocm-.*)$"
| project TimeGenerated, Namespace, PodName, ContainerName, PodStatus, ContainerRestartCount
| order by TimeGenerated desc
```

Prometheus metrics (Azure Managed Prometheus → Azure Monitor Workspace)
```
PrometheusMetrics
| where TimeGenerated between (start .. end)
| where Labels["namespace"] matches regex @"^(aro-hcp|maestro|ocm-.*)$"
| where Name in~ ("up", "container_cpu_usage_seconds_total", "container_memory_working_set_bytes")
| summarize avg(value) by Name, tostring(Labels["namespace"]), bin(TimeGenerated, 5m)
| order by TimeGenerated
```

Cluster service errors seen by RP frontend (if exported into logs)
```
ContainerLogV2
| where TimeGenerated between (start .. end)
| where KubernetesNamespace == "aro-hcp" and ContainerName has "frontend"
| where LogMessage has "ClusterService" and LogLevel in ("error", "warn")
| project TimeGenerated, PodName, LogLevel, LogMessage
```

Adjust table/column names to match your environment if your ingestion schema differs (e.g., legacy `ContainerLog` vs `ContainerLogV2`).

## Output Layout
Queries are serialized as JSON (and optionally CSV) under `kusto/`:
- `kusto/container_logs.json`
- `kusto/kube_events.json`
- `kusto/pod_inventory.json`
- `kusto/prometheus_metrics.json`
- `kusto/queries.txt` (captured KQL and parameters for reproducibility)

## Authentication & Authorization
- Use `DefaultAzureCredential` (supports dev box, MI, SP). Ensure the identity has query rights on the ADX cluster/database (e.g., `Viewer`/`Database User`) or on the federated Log Analytics workspace.
- Region mapping (endpoint/database) is read from config; do not hardcode.

## Limitations
- Raw Kubernetes objects (Secret/ConfigMap/Deployment YAML) are not exported; rely on events/logs/metrics to infer state.
- For highly specific object snapshots, add RP‑side internal APIs to persist minimal state to Cosmos and surface via the gather tool if needed.

---

Next steps: approve the plan, then scaffold `tooling/must-gather/` and the `hcpctl gather` command with a minimal collector (manifest + contexts + a few core resources) to validate end‑to‑end flow before expanding coverage.
