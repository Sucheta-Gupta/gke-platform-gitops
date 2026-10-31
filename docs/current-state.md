# Current platform inventory

This document records facts gathered from the running environment. It is an inventory, not a set of deployable manifests. Values not captured must not be guessed.

## GKE cluster

| Setting | Observed value |
|---|---|
| Cluster | `elk-gke-cluster` |
| Region | `us-central1` |
| VPC | `elk-gke-vpc` |
| Subnet | `elk-gke-subnet` |
| Kubernetes version | `1.35.8-gke.1796000` |
| Release channel | `REGULAR` |
| Private nodes | Enabled |
| Private control-plane endpoint | Enabled |
| Private control-plane IP | `172.16.0.2` |
| Master IPv4 CIDR | `172.16.0.0/28` |
| Pod secondary range | `pod-range`, `10.20.0.0/16` |
| Service secondary range | `services-range`, `10.30.0.0/20` |
| Gateway API | Standard channel |
| Managed Prometheus | Enabled |
| GKE PD CSI driver | Enabled |
| DNS cache | Enabled |
| Shielded nodes | Enabled |
| Logging | System components and workloads |

Workload Identity was previously confirmed enabled. The exact pool value is not recorded here.

### Node pools

| Pool | Machine type | Disk | Node version |
|---|---|---:|---|
| `default-pool` | `e2-standard-2` | 100 GB | `1.35.8-gke.1796000` |
| `elasticsearch-pool` | `e2-standard-4` | 100 GB | `1.35.8-gke.1796000` |

Autoscaling bounds, node locations, service accounts, labels, taints and management settings still need to be captured.

## ECK / Elasticsearch / Kibana

| Component | Observed configuration |
|---|---|
| ECK Helm chart | `eck-operator-3.5.0` |
| ECK app version | `3.5.0` |
| ECK replicas | 1 |
| ECK resource requests | 100m CPU, 150Mi memory |
| ECK resource limits | 1 CPU, 1Gi memory |
| ECK CRDs | Installed by Helm |
| Elasticsearch resource | `elasticsearch/elk` |
| Elasticsearch version | `9.5.4` |
| Elasticsearch nodeSets | One nodeSet named `data`, count 3 |
| Elasticsearch storage | 100Gi per node, `standard-rwo`, RWO |
| Elasticsearch placement | `workloads=elasticsearch`; toleration `dedicated=elasticsearch:NoSchedule` |
| Elasticsearch zone awareness | `us-central1-a`, `us-central1-b`, `us-central1-c` |
| Elasticsearch PDB | `maxUnavailable: 1` |
| Elasticsearch HTTP TLS | Self-signed certificate enabled |
| Kibana resource | `elasticsearch/kibana` |
| Kibana version | `9.5.4` |
| Kibana replicas | 1 |
| Kibana Elasticsearch association | `elk` in namespace `elasticsearch` |

An Elasticsearch resource condition reported that CPU requests/limits were not set for the data NodeSet. The cluster was nevertheless observed healthy and green. Review this warning separately; do not silently change sizing during recovery work.

### Elasticsearch data lifecycle

- Data stream: `logs-kubernetes`
- Index template: `logs-kubernetes`
- ILM policy: `logs-kubernetes-7d`
- ILM configuration observed: rollover at 7 days or 50 GB primary shard size; warm phase at 2 days; delete phase at 7 days.
- Current backing index observed: `.ds-logs-kubernetes-2026.10.28-000001`.

No snapshot repository was returned by `spec.snapshotRepositories`. No Kubernetes CronJobs were found. External Elasticsearch snapshot/restore capability is an outstanding recovery requirement. ILM retention is not a substitute for backups.

## Fluent Operator / Fluent Bit

| Component | Observed value |
|---|---|
| Helm release | `fluent-operator` |
| Chart | `fluent-operator-4.3.0` |
| Operator app version | `3.10.0` |
| Fluent Bit image | `ghcr.io/fluent/fluent-operator/fluent-bit:5.1.0` |
| Fluent Bit replicas | DaemonSet; 6 pods observed, one per node |
| Fluentd | Disabled |
| Fluent Bit resources | Requests 10m CPU / 25Mi; limits 500m CPU / 200Mi |
| Operator resources | Requests 100m CPU / 20Mi; limits 100m CPU / 60Mi |
| Position database | HostPath `/var/lib/fluent-bit/` |
| Config source | `ClusterFluentBitConfig/fluent-bit-config` |
| Generated config Secret | `fluent-operator-system/fluent-bit-config` |

The generated Secret is owned by `ClusterFluentBitConfig/fluent-bit-config`. GitOps should manage the source custom resource and its selected Fluent Bit CRs, not the generated Secret.

### Fluent Bit selected resources

All listed resources match the `fluentbit.fluent.io/enabled=true` selectors.

- ClusterFluentBitConfig: `fluent-bit-config`
- ClusterInputs: `containerd`, `tail`
- ClusterFilters: `containerd`, `kubernetes`, `systemd`
- ClusterOutputs: `elasticsearch-output`, `stdout-test` (both intentionally retained)
- ClusterParsers: none observed
- ClusterMultilineParsers: none observed

The `tail` input reads `/var/log/containers/*.log`, uses the `cri` parser, excludes Fluent Bit's own container logs, and tags records `kube.*`.

The `containerd` systemd input reads `/var/log/journal`, filters for containerd and kubelet units, and tags records `service.*`.

Kubernetes metadata enrichment is enabled for `kube.*`; labels and annotations are disabled. The configuration lifts, modifies and re-nests Kubernetes metadata fields. Lua filters are referenced for containerd and systemd processing.

The Elasticsearch output targets `elk-es-http.elasticsearch.svc:9200`, uses TLS verification and CA file `/etc/fluent-bit/es-ca/ca.crt`, and writes to `logs-kubernetes`. Credentials must be supplied through a Kubernetes Secret, never embedded in this repository.

The stdout output matches `kube.*` and uses `json_lines`. It is intentionally retained.

**Security note:** A plaintext Elasticsearch password appeared in a pasted generated configuration during inventory. Treat that credential as exposed and rotate it. It is intentionally not recorded here.

## Kibana Gateway

- Gateway: `elasticsearch/kibana-gateway`
- GatewayClass: `gke-l7-rilb`
- Hostname: `kibana.elk.internal`
- Internal address observed: `10.0.0.23`
- Listeners: HTTP 80 and HTTPS 443
- HTTPS TLS termination option: `networking.gke.io/cert-manager-certs: external-cert`
- HTTPRoute: `elasticsearch/kibana-route`
- Backend: Service `kibana-kb-http`, port 5601
- Gateway and route were observed programmed/accepted and healthy.

Only desired `spec` fields should be committed. GKE-generated status and annotations must not be copied into Git.

## Namespace inventory

Relevant namespaces observed:

- `elastic-system`
- `elasticsearch`
- `fluent-operator-system`
- `demo-app`

GKE-managed namespaces and system namespaces are not application-owned GitOps resources.
