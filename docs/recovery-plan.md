# Platform recovery plan

## Already inventoried

- [x] GKE cluster identity, region, network, subnet, IP ranges, private-cluster settings and enabled add-ons.
- [x] Node-pool names, machine types and disk sizes.
- [x] ECK chart and core Elasticsearch/Kibana configuration.
- [x] Fluent Operator chart, Fluent Bit image, source custom resource and selected input/filter/output inventory.
- [x] Kibana Gateway and HTTPRoute desired configuration.
- [x] Elasticsearch data stream, index template and ILM policy details.
- [x] Decision to retain the Fluent Bit `stdout-test` output.

## Still required before declaring recovery-ready

1. **GKE infrastructure as code**
   - Capture node-pool autoscaling, locations, labels, taints, service accounts and management settings.
   - Identify Terraform or other source of truth for VPC, subnet, secondary ranges, firewall rules, IAM, cluster and node pools.
   - Do not import or recreate existing resources until state ownership and drift are understood.

2. **IAM and identity**
   - Capture GCP custom role definitions and bindings for Fluent Bit and Kibana.
   - Capture Kubernetes ServiceAccounts, RBAC and Workload Identity bindings.
   - Capture Elasticsearch native security roles, users and role mappings for Fluent Bit and Kibana.
   - Store credentials outside Git in an appropriate secret manager.

3. **Elasticsearch recovery**
   - Configure a snapshot repository in storage outside the cluster.
   - Configure and test snapshot scheduling and restore.
   - Confirm storage bucket permissions, retention and encryption.
   - Preserve ILM policy and index template definitions independently of snapshots.

4. **Operator-managed configuration**
   - Capture clean source manifests for ClusterFluentBitConfig, ClusterInputs, ClusterFilters and ClusterOutputs.
   - Capture referenced Lua scripts and parser configuration after identifying their source and ownership.
   - Capture intentional Helm values rather than copying all computed defaults.
   - Keep generated ECK and Fluent Bit Secrets out of Git.

5. **GitOps**
   - Add reviewed, sanitized manifests and Helm chart references.
   - Install and configure Argo CD only after the initial desired state is reviewed.
   - Keep application source/CI in the separate `cicd` repository.
   - Define how application deployment manifests are sourced without merging application code into this platform repository.

6. **Recovery validation**
   - Test platform recreation in a disposable environment or controlled recovery environment.
   - Test Elasticsearch snapshot restore.
   - Verify Gateway, DNS/certificates, Fluent Bit ingestion and Kibana access.
   - Document recovery order, dependencies, RTO and RPO.

## Important boundaries

- Inventory documentation is not a deployable configuration.
- Do not apply reconstructed manifests to the live cluster until reviewed.
- Do not commit passwords, API keys, TLS private keys, kubeconfigs, or Kubernetes Secret data.
- Preserve existing resource ownership; avoid having Helm, Argo CD and operators fight over the same generated resource.
