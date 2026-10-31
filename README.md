# GKE Platform GitOps

GitOps and disaster-recovery configuration for `elk-gke-cluster`.

## Repository boundaries

- **Application repository:** [cicd](https://github.com/Sucheta-Gupta/cicd) — FastAPI source, tests, CI, image build and publishing.
- **Platform repository:** this repository — GKE/platform infrastructure, EFK stack, Argo CD, shared Kubernetes resources and recovery documentation.

Application deployment remains separate from platform components. CI publishes immutable image tags; GitOps manages deployment configuration and Argo CD reconciliation.

## Current state

- [Current platform inventory](docs/current-state.md)
- [Recovery plan and outstanding gaps](docs/recovery-plan.md)

## Safety rules

- Never commit Kubernetes Secret data, passwords, tokens, private keys, or kubeconfig files.
- Do not commit raw `kubectl get -o yaml` output without removing generated metadata and status.
- ECK-generated credentials and certificates should be recreated by ECK where appropriate.
- Back up Elasticsearch data independently using snapshots to storage outside the cluster.
- Treat this repository as desired state only after manifests have been reviewed and tested.

