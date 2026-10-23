# Fider GitOps — Amazon EKS

This repository manages the deployment of [Fider](https://github.com/getfider/fider), an open-source customer feedback application, on Amazon EKS using **Helm, Argo CD and External Secrets Operator**.

It separates application deployment from infrastructure provisioning and uses Git as the source of truth for Kubernetes configuration.

## What this repository does

- **Helm:** Defines the Kubernetes Deployment, Service and Ingress for Fider.
- **Argo CD:** Deploys Fider and manages changes from Git.
- **Multiple environments:** Uses separate `dev`, `test` and `prod` namespaces.
- **Image promotion:** Deploys SHA-tagged images from Amazon ECR, with manual promotion between environments.
- **External Secrets Operator:** Retrieves database credentials and application secrets from AWS Secrets Manager.
- **Monitoring:** Uses Prometheus, Grafana and Alertmanager to monitor Kubernetes resources and application availability.

## Deployment workflow

```text
GitHub Actions (fider-app)
          |
    Build Docker image
          |
      Amazon ECR
          |
Update image tag in fider-gitops
          |
        Argo CD
          |
     Helm Deployment
          |
       Amazon EKS
          |
    dev → test → prod
```

The `dev` environment uses automatic Argo CD synchronization and self-healing.

The `test` and `prod` environments use manual synchronisation, allowing the same tested container image to be promoted between environments.

## Repository structure

```text
fider-gitops/
├── argocd/
│   └── applications/        # Argo CD applications
├── environments/
│   ├── dev/                 # Dev values and secrets
│   ├── test/                # Test values and secrets
│   └── prod/                # Production values and secrets
├── helm/
│   └── fider/               # Fider Helm chart
├── monitoring/
│   └── prometheus/          # Monitoring and alert rules
├── docs/
│   └── RUNBOOK.md           # Deployment instructions
└── README.md
```

## Monitoring

The monitoring stack uses:

| Tool | Purpose |
|---|---|
| Prometheus | Collects Kubernetes metrics |
| Grafana | Visualises CPU, memory and Pod metrics |
| Node Exporter | Collects worker-node metrics |
| kube-state-metrics | Provides Kubernetes workload metrics |
| Alertmanager | Manages Prometheus alerts |
| AWS CloudWatch | Monitors AWS resources such as RDS |

A custom Prometheus alert detects when the development Fider Deployment has no available replicas.

## Deployment instructions

See the **[Deployment Runbook](docs/RUNBOOK.md)** for the full setup process, including Helm installations, AWS Secrets Manager configuration, Argo CD and Prometheus.

## Related repositories

- [fider-infra](https://github.com/aactive1/fider-infra) — AWS infrastructure provisioned using Terraform.
- [fider-app](https://github.com/aactive1/fider-app) — Fider application, Docker configuration and CI/CD pipeline.
